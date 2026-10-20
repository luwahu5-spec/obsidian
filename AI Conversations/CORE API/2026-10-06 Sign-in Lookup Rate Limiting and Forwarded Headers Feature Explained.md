---
date: 2026-10-06
updated: 2026-10-06
source: Claude Code
project: core-web-api
tags: [domain-knowledge, sso, rate-limiting, forwarded-headers, aws, security]
---

# Sign-in lookup rate limiting & forwarded headers — what the feature actually does

> **Takeaway:** The anonymous `POST /SignIn/Lookup` gets a fixed allowance per caller IP per minute.
> Behind a proxy or load balancer the API only sees the proxy's IP, so the real caller has to come
> from `X-Forwarded-For`. That header can be forged by anyone, so the API believes it **only from hops
> you list**: `KnownProxies` (single, exact IPs) and `KnownNetworks` (CIDR ranges).
> The non-obvious part: **without the right list for the environment, everyone (attacker and core-web)
> can collapse into one bucket.** Then one attacker can block every user's sign-in.
> **Our setup:** the API is on EC2 with no load balancer, so neither list is configured, only the built-in loopback trust.
>
> ⚠️ **2026-10-07: the in-app limiter was removed** (decision: do rate limiting in AWS). See *Why not AWS throttling instead* for what AWS needs before it can replace it.

## The real-world situation being modelled

A request can pass through several machines before reaching the API:

```
caller (203.0.113.7) ──► AWS ALB (10.0.1.15) ──► [nginx on instance (127.0.0.1)] ──► API
```

- The API's `RemoteIpAddress` is only **the last hop**, the machine that opened the TCP connection to it.
- Each well-behaved proxy **appends** the address it received the request from to `X-Forwarded-For` (XFF). After the ALB: `XFF: 203.0.113.7`.
- The caller can also send their own XFF. The ALB appends to it rather than replacing it, so the header becomes `XFF: <whatever the caller typed>, 203.0.113.7`.
- **The right-most entries are written by your own infrastructure. Everything further left is unverified.**
- So "trust" means: *I believe the entry this hop added.*

## Plain-language picture (caller ID vs sticky note)

- **`RemoteIpAddress` = caller ID.** The network records it, and it can't usefully be faked, because the reply goes back to that address.
- **`X-Forwarded-For` = a sticky note** the sender writes. Anyone can write anything on it.
- It exists because a receptionist (proxy/load balancer) makes the caller ID show the receptionist's number. The receptionist writes the note saying who really called.
- **The rule:** believe the note only if the call came from a receptionist on our trusted list (`KnownProxies`/`KnownNetworks`). Otherwise throw the note away and use the caller ID.
- **Default list = loopback only** (`127.0.0.1`, `::1`), meaning "this same computer". Nobody on the internet can connect from it.
- **Example:** an attacker at 203.0.113.7 sends notes 1.1.1.1, 2.2.2.2, 3.3.3.3, ….
  - Every note is discarded, so every request counts against 203.0.113.7.
  - Request 121 gets 429.
  - If notes were believed, each request would get a new counter and the limit would never trigger.

## KnownProxies vs KnownNetworks

Both answer the same question, *"is this hop one of my proxies?"*, at different granularity.

| | `KnownProxies` | `KnownNetworks` |
|---|---|---|
| What it holds | Individual IP addresses | CIDR ranges (`address/prefix`) |
| Example | `"10.0.1.15"` | `"10.0.0.0/16"` = 10.0.0.0–10.0.255.255 |
| Use when | The proxy has a **fixed** IP (one nginx box, a fixed reverse proxy) | Proxy IPs **change**. AWS ALB nodes take addresses from their subnets and change over time, so list the subnet/VPC CIDR |
| Built-in default (.NET 8) | `::1` (IPv6 loopback) | `127.0.0.0/8` (IPv4 loopback) |

- A hop matching **either** list counts as trusted.
- `ForwardLimit` (.NET default 1) caps how many XFF entries are consumed from the right. Set it to **the number of proxies in front of the API**.
- **We don't configure either list (removed 2026-10-06).** The API runs on EC2 with no load balancer, so only the built-in loopback trust applies.
  - The `ForwardedHeaders:*` settings existed briefly and were removed because nothing in our setup needed them.
  - If a load balancer is ever added, re-add its CIDR to `KnownNetworks`. The comment at [SignInRateLimiting.cs:36-41](C:/Users/CULI/source/repos/core-web-api/Minnovare.Core.WebApi/Services/SignInRateLimiting.cs) says so.

## The configuration chain

| Setting | Default | Read at |
|---|---|---|
| `SignInLookup:PermitLimitPerMinute` | 120 | `SignInRateLimiting.cs:30-34` (validated > 0 at startup) |

- `UseForwardedHeaders` must still be switched on for `XForwardedFor` ([SignInRateLimiting.cs:40-41](C:/Users/CULI/source/repos/core-web-api/Minnovare.Core.WebApi/Services/SignInRateLimiting.cs)).
  - The .NET default is to process **no** forwarded headers.
  - If the platform runs nginx on the box, every request then arrives from `127.0.0.1`. Without this switch, all callers would share one bucket.
  - Scheme (`X-Forwarded-Proto`) and host are untouched.

### Sizing decision (2026-10-06): 500/min, set in `appsettings.json` → `SignInLookup:PermitLimitPerMinute`

Basis is a count of web users in the local prod-copy database. CORE stores no sign-in history, so this is a capacity bound, not measured traffic.

| Measure | Value |
|---|---|
| All accounts | 1,942 |
| Users with `site_id` (web access) | 896 |
| Web users on **active** sites | 526 |
| Largest group sharing one shift-change time: Perth (`W. Australia Standard Time` 271 + `Australia/Perth` 146) | ≤ 417 |
| Next largest: Alaska (`Alaskan Standard Time` + `America/Anchorage`) | ≤ 59 |

- **Rule:** the limit must not block the largest same-timezone group even if all of them signed in within the same minute (worst case). Take ~420, round up with headroom for mistyped emails and retries → **500**.
- **Cost:** one direct caller is capped at ~8 requests/second.
- **Revisit** if web users grow substantially, or once real peak data exists (Keycloak LOGIN events).
- The code fallback `DefaultPermitLimitPerMinute` was changed to match (SignInRateLimiting.cs:20-27).

### Background: the original 120/min
- **Not an industry standard.** It's a judgement value chosen when the endpoint was built. OWASP says to rate-limit account-enumeration endpoints but gives no number.
- **Two jobs for one number.** It is:
  - a per-IP cap for anyone calling the API directly, and
  - effectively a **global cap on all web sign-ins**, because every user's lookup comes from core-web's single address.
- **The second job sets the floor.** It must sit comfortably above peak real sign-ins per minute (shift changes across sites). Too low, and everyone gets locked out at peak.
- **How to set it properly.**
  - Measure peak lookups per minute, e.g. from Keycloak login events or API logs around 06:00/18:00 site-local time.
  - Set the limit to 3–5× that peak.
  - Change it per environment via `SignInLookup:PermitLimitPerMinute`, with no code change.
- **Limits of a per-IP cap.** It slows one machine (120/min ≈ 170k guesses/day per IP). It does little against a distributed attacker.
- **Per-person protection** on core-web's `/Signin` is covered by the AWS rate-limiting task (decided 2026-10-14), not by code. It needs an ALB or CloudFront in front of core-web before a WAF rate-based rule can apply.

### Why not AWS throttling instead (2026-10-07)
- **"API throttling" is an API Gateway feature, not EC2.** Our API is on a plain EC2 instance with nothing in front of it, so there is nothing to configure in the console today.
- **API Gateway throttling counts all callers together** (per account/stage/route), or per API key with usage plans. It is **not per IP**.
  - A route limit would put one attacker and core-web in one shared bucket. That's the sign-in DoS we designed against.
- **AWS WAF rate-based rules are per IP** (with a scope-down to `/SignIn/Lookup`).
  - WAF attaches only to ALB, CloudFront, API Gateway, AppSync, Cognito, App Runner and Verified Access, **never directly to EC2**.
  - Using it means adding one of those in front of the API. That's a DNS/TLS/cost change, and it would bring back the need for forwarded-headers config (`KnownNetworks`).
- ~~**Decision:** keep the in-app limiter.~~ **Superseded 2026-10-07:** the in-app limiter was **removed**.
  - The team's position is that rate limiting belongs in AWS (WAF), not application code.
  - Until WAF is in front of the API, `/SignIn/Lookup` has **no rate limit**. Email enumeration is possible at any speed.
  - The WAF rule needs an ALB or CloudFront first. Track it as an infrastructure task.
  - `SignInRateLimiting.cs`, its tests, the `SignInLookup:*` settings and `UseForwardedHeaders` are gone. The rest of this note describes the removed design, kept for when WAF is set up.

### Why not reuse Keycloak's brute-force numbers
Keycloak realm → Security defenses → Brute force detection. Our values: max login failures 30, wait increment 1 min, max wait 15 min, failure reset 12 h, quick login check 1000 ms → min quick wait 1 min.

| | Keycloak brute force | Lookup rate limit |
|---|---|---|
| Protects against | Password guessing on **one account** | Checking **which emails exist**, across all accounts |
| Counted per | User account | Calling IP (and core-web is one IP for everyone) |
| Counts | Failed passwords only | Every request, including successful ones |
| When triggered | That one user waits 1–15 min | Every caller in that bucket gets 429 for the rest of the minute |

- The numbers aren't interchangeable. A limit of 30/min on the lookup would cap **all web sign-ins** at 30 a minute.
- Keycloak never sees the lookup (it's CORE-only), so its settings don't protect it.
- Keycloak brute force also doesn't apply to SSO users' passwords. Those are typed at Microsoft and protected by Entra smart lockout.
- The two layers complement each other: the lookup limit for enumeration, Keycloak for local passwords, Entra for SSO passwords.

## Where it takes effect

1. `app.UseForwardedHeaders()` ([Startup.cs:456](C:/Users/CULI/source/repos/core-web-api/Minnovare.Core.WebApi/Startup.cs)) rewrites `Connection.RemoteIpAddress` for **every request in the app**, not just the lookup.
2. `app.UseRateLimiter()` (Startup.cs:459) applies policy `signin-lookup` to `SignInController.LookupAsync`.
3. The partition key is `GetCallerAddress` ([SignInRateLimiting.cs:89-94](C:/Users/CULI/source/repos/core-web-api/Minnovare.Core.WebApi/Services/SignInRateLimiting.cs)).
   - It reads **only** the resolved `RemoteIpAddress`, never the raw header.
   - IPv4-mapped IPv6 (`::ffff:1.2.3.4`) is folded into IPv4.
   - A missing address goes to one shared `"unknown"` bucket.

## How the middleware resolves the caller (algorithm)

```
current = RemoteIpAddress
repeat up to ForwardLimit times:
    if current is NOT in KnownProxies/KnownNetworks → stop          (don't believe what it wrote)
    take the right-most unread XFF entry
    if it isn't a valid IP → stop, keep the proxy's address
    current = that entry
RemoteIpAddress = current
```

## Worked examples (same caller 203.0.113.7, PermitLimit 120)

| Topology | What the API sees | Config needed | Resulting bucket |
|---|---|---|---|
| **A. Instance exposed directly** (IIS in-process / Kestrel) | peer `203.0.113.7` | none | `203.0.113.7` ✅. A forged XFF is ignored, because the peer is not a trusted proxy |
| **B. nginx on the same instance, no ALB** (EB Linux single instance) | peer `127.0.0.1`, XFF `203.0.113.7` | none (loopback trusted by default) | `203.0.113.7` ✅ |
| **C. ALB → API** (no proxy on the box) | peer `10.0.1.15`, XFF `203.0.113.7` | `KnownNetworks: ["<VPC CIDR>"]`, ForwardLimit 1 | `203.0.113.7` ✅ |
| C. **without** config | same | none | `10.0.1.15` ❌. Everyone shares the ALB's bucket |
| **D. ALB → nginx → API** | peer `127.0.0.1`, XFF `203.0.113.7, 10.0.1.15` | `KnownNetworks: ["<VPC CIDR>"]`, **ForwardLimit 2** | `203.0.113.7` ✅ |
| D. with ForwardLimit 1 | same | | `10.0.1.15` ❌. One hop short, so it lands on the ALB |
| **Forgery via ALB** (setup C, configured) | XFF `1.2.3.4, 203.0.113.7` | | `203.0.113.7` ✅. Only one entry is consumed from the right, so the forged `1.2.3.4` is never read |

All of these are covered by tests in `Minnovare.Core.WebApi.Tests/Services/SignInRateLimitingTests.cs`:

| Test | Example covered |
|---|---|
| `UntrustedSender_CannotResetAllowanceWithForgedForwardedHeaders` | Forged XFF from an untrusted sender |
| `TrustedProxy_ClientsHaveSeparateAllowances_AndForgedLeftEntryIsIgnored` | Trusted proxy; forged left-hand entry |
| `TwoTrustedProxyHops_ResolveClientRatherThanSharedBalancer` | Two trusted hops |
| `MalformedForwardedAddress_DoesNotChooseAnotherBucket` | Malformed XFF entry |

## Design subtleties & edge cases

- **Why the ❌ rows are dangerous, not just inaccurate.** core-web calls the lookup server-side for *every* user's sign-in.
  - If the attacker and core-web share one bucket, 120 junk requests a minute from one attacker → every real sign-in gets 429.
  - Correct trust config gives core-web its own bucket, separate from the attacker's.
- **Why not "trust any private IP"?** That was the earlier hand-written version (right-most *public* entry).
  - On a directly exposed instance (setup A), the caller writes XFF themselves.
  - They could send a new fake public IP on every request, get a fresh bucket each time, and bypass the limit entirely.
  - Trust must come from config, never from what an address looks like.
- **Trusting a whole VPC CIDR** means any machine inside the VPC could forge XFF. That's acceptable, because those machines are already ours. Keep the range as tight as practical (e.g. just the ALB subnets).
- **App-wide side effect.** Anything else that reads `RemoteIpAddress` (logs, the exception handler) also sees the resolved client.
  - `UseForwardedHeaders` currently runs after `UseExceptionHandler`/`UseHttpsRedirection` (Startup.cs:419-424 vs 456).
  - Because of that, the exception handler's own logging still sees the proxy address. This is low impact.
- **Never list the API's own server IP.** The lists are for machines that sit *in front of* the API and forward requests to it. The server itself is never one of those hops.
  - Listing it adds nothing; the only thing it can do is open a hole.
  - Anything that connects *from* that address would have its `X-Forwarded-For` believed, and could pick a fresh bucket on every request.
  - A non-Elastic EC2 public IP returns to AWS's pool on stop/start and can be reassigned to someone else's machine. That stranger would then be trusted.
- **Our actual topology (confirmed 2026-10-06):** the API runs on an EC2 instance with no ALB, exposed directly. That is setup A, or B if the platform runs nginx on the box.
  - Leave `KnownProxies`/`KnownNetworks` empty and keep the default `ForwardLimit` of 1.
  - Revisit this if an ALB is ever added in front.
  - The instances have an **Elastic IP**. That is the same address as the "Public IPv4" the EC2 console shows, and it stays with our account until it's released. So the IP-reuse risk doesn't apply here.
  - Listing the IP is still pointless, and still lets anything connecting from it forge XFF.
- **Choosing values for an environment** (questions for DevOps):
  1. Is there an ALB/ELB in front of the API?
  2. Is there a reverse proxy on the instance (EB Linux = nginx; EB Windows = IIS)?
  3. What are the CIDRs of the VPC/ALB subnets?

  Then: `ForwardLimit` = number of proxies, and `KnownNetworks` = the ALB subnet/VPC CIDR.

## Questions asked
- 2026-10-06 — "KnownProxies or KnownNetworks — explain these two in detail" → Same trust question; exact IPs vs CIDR ranges. ALBs need `KnownNetworks` because their IPs change. See *KnownProxies vs KnownNetworks* and *Worked examples*.
- 2026-10-06 — "API is on EC2, no ALB, exposed directly — is adding the EC2 IP to appsettings a security risk?" → Yes, and it's pointless: add nothing. The server is never a hop in front of itself; trusting its IP only lets traffic from that IP forge XFF (plus the risk of public-IP reuse). See *Design subtleties*.
- 2026-10-06 — "Each instance has a public IP and an Elastic IP" → With an EIP associated they're the same address and it's ours until released, so the reuse risk is gone. The answer is unchanged: still add nothing.
- 2026-10-06 — "Is forged XFF already handled by our rate limiting?" → Yes, with empty lists. Internet callers' XFF is ignored (test `UntrustedSender_CannotResetAllowanceWithForgedForwardedHeaders`). The protection is exactly *not* listing addresses. Only built-in loopback is trusted, so just a process on the same box could forge XFF, and at that point the box is already compromised.
- 2026-10-06 — "Explain the loopback / untrusted-caller point explicitly, no prior knowledge" → Caller-ID vs sticky-note analogy with a request-by-request table. See *Plain-language picture*.
- 2026-10-06 — "If KnownProxies/KnownNetworks don't apply, remove them? Where does 120 come from, is it best practice?" → Removed the settings, but kept `XForwardedFor` switched on with loopback trust. 120 is a judgement value, not a standard; it's effectively a global cap on web sign-ins, so set it from measured peak. See *Configuration chain → Where 120/min comes from*.
- 2026-10-06 — "So `Configure<ForwardedHeadersOptions>(XForwardedFor)` is the part that stays?" → Yes, as a pair with `app.UseForwardedHeaders()` (Startup.cs:456): one says *which* header, the other *runs* it, and neither works alone. Needed if the platform has nginx on the box; harmless no-op with IIS.
- 2026-10-06 — "Can PermitLimitPerMinute refer to Keycloak's brute-force settings?" → No. Different threat (per-account password guessing vs per-IP enumeration), different unit (failures per user vs all requests per IP). 30/min would cap all web sign-ins. See *Why not reuse Keycloak's brute-force numbers*.
- 2026-10-06 — "Put the limit in configuration with a defensible number" → Sized from local user counts: largest shift-change group ≈ 417 Perth web users → 500/min in appsettings.json. See *Sizing decision*.
- 2026-10-07 — "AWS provides API throttling, configure that instead of our own limiter?" → Not on EC2. Throttling is API Gateway (not per IP), and WAF rate rules are per IP but need ALB/CloudFront/API Gateway in front. Keep the in-app limiter. See *Why not AWS throttling instead*.

## Related
- [[2026-08-27 SSO Enterprise Auth Ticket Breakdown]] — open question 1 (anonymous lookup + rate limit)
- [[2026-07-14 Authentication Authorization and SSO IdP Brokering Feature Explained]]
