---
tags: [meta, playbook, prompts, skills]
updated: 2026-07-06
---

# Claude Workflow Playbook — skills, templates, and prompts

Built from analysis of ~55 sessions (Claude Code + Codex, Mar–Jul 2026). Companion to [[Knowledge Priorities and Correlations]].

## 1. Your five real workflows → five skills (auto-trigger)

These live in `~/.claude/skills/` and trigger automatically from normal phrasing — you never need to type a slash command:

| Skill | Triggers when you say… | What it forces |
|---|---|---|
| **debug-root-cause** | paste an error log, "why is this happening", "help me recreate" | diagnosis-only mode, reproduce-first, env-gap analysis, no fixes until asked |
| **trace-data** | "where does this data come from", "which API/table", "which code decides" | full UI→API→DB chain with file:line, gatekeepers, defaults, ready SQL |
| **sql-testdata** | paste a SELECT scaffold + "generate update/insert sql", "check if X appears" | Id-constrained statements, pre/post-check SELECTs, prod-DB warning |
| **branch-ops** | "create a branch on top of…", "stash", "pop", "merge A to B" | base-branch confirmation, `-u` for untracked, no auto-staging, status report |
| **ui-fix** | "change this color/border/spacing", "did not take effect", paste a mockup | full spec upfront, 5-cause checklist (artifact/specificity/scoped/BS5/cache), blast-radius check on shared components |
| **orchestrate** | "plan and delegate", "break this down, let sonnet/codex execute", give a goal + "save tokens" | expensive model plans + writes task contracts to a plan file; Sonnet subagents or Codex execute; architect verifies by diff, escalates after 2 failures |

Also: `~/.claude/CLAUDE.md` (new) now loads your global working rules into **every** session — that's the file that makes Sonnet behave like Fable.

## 2. Best-practice prompt templates (copy, fill, paste)

The pattern behind every template: **front-load every fact you already have** (branch names, hex codes, exact log lines, file paths, what you tried). Each missing fact costs one full round trip — that's the real "price" of a prompt.

### Prod/bug investigation
```
[paste exact error log / stack trace]
Environment: <prod/staging/local> on <branch/version>. Works in <other env>: <yes/no>.
What I already tried: <postman replay / payload / result>.
MODE: diagnosis only — find the root cause and how to reproduce it locally. Do not propose fixes yet.
```

### UI style change
```
Component/page: <name or how to reach it>. [attach mockup + current screenshot]
States and exact colors:
- <state>: bg <#hex>, border <#hex>, text <#hex>
Check which CSS artifact the browser loads and which rule currently wins BEFORE editing.
If the component is shared, list all usages first and parameterize instead of restyling.
```

### Data trace
```
On <page>, the value/element <X> — trace it: razor → API endpoint (with exact params and defaults)
→ repository → SQL table. List every condition that can hide/zero it. Give me a SELECT to inspect the raw data.
```

### SQL generation
```
[paste your SELECT TOP (1000) scaffold — it tells the model the exact columns and DB name]
Generate <insert/update/check> SQL that <intent>. Constrain by Id. Include pre/post-check SELECTs.
```

### Branch/stash operation
```
Base branch: <exact name>. New branch: <exact name>. Carry uncommitted changes: <yes/no>.
Do not stage anything; report git status + stash list when done.
```

### API doc for an external consumer (Deswik-style)
```
Generate consumer documentation for <endpoints>. Format per endpoint:
Relative URL / Request type / Body / Response: JSON + one example.
Replace all real values with dummy data (names, GUIDs, coordinates, API keys). Query params: list every
enum option, not just the default.
```

### PR comment / reviewer message
```
[paste the PR comment verbatim + the code it points at]
Explain what the reviewer means, whether they're right, and the minimal change that addresses it. Draft a one-line reply.
```

### Explain to lead / tester / PM
```
Audience: <my lead / testers / PM — non-coding>. Topic: <the change/design>.
Write it so they can act on it: what changed, what they need to do/decide, what to test and how to set it up.
No code unless essential.
```

## 3. Model strategy (cost vs quality)

Use **Sonnet** (default) for: UI fixes, SQL generation, data traces, branch ops, doc generation, unit tests from existing patterns, dataset generation. With the global rules + skills, these are procedure-following tasks — Sonnet handles them well.

Escalate to **Fable/Opus** for: prod incident root-causing (accumulated-state bugs), architecture planning (2.23-refactor-class decisions), API integration design reviews, anything where the answer isn't findable by procedure. These are exactly the P1/P5 categories in [[Knowledge Priorities and Correlations]] — the sessions that produced your most valuable knowledge.

Cost hygiene regardless of model:
- **One problem per session.** The June-16 marathon (spinner + tabs + dashboard + SQL in one chat) burned context and forced compaction summaries. New problem → new session.
- Long session about to compact? Ask for a handover note into the vault instead, and start fresh.
- Paste artifacts, don't describe them. A pasted log line is cheaper than three clarification rounds.

## 4. Methodology corrections (what went wrong before, and the rule that prevents it)

1. **Fix-ping-pong on UI** ("still white… still small… ×6") → caused by editing before knowing which artifact/rule wins. Rule: the ui-fix 5-cause checklist runs BEFORE the first edit.
2. **AI proposing fixes while you wanted reproduction** (Hole.Id saga frustration) → state MODE explicitly; the debug-root-cause skill + global rules now hard-code diagnosis-first.
3. **Base-branch near-misses** (branch almost created off stale master) → never let the AI assume a base; branch-ops now requires confirmation.
4. **Timezone bugs written twice** (shift EndTime filter; UTC+8 test machine) → decide UTC vs site-timezone before writing any date filter; it's now a global rule.
5. **"Unit tests" silently hitting real SQL Server** → known fixture behavior, documented; don't rediscover it.
6. **Verification gap** — AI said "done", you found it wasn't (translations, spinner) → global rule: AI must say "changed X, verify by Y", never "fixed".
7. **Knowledge rot** — notes were chat walkthroughs → save-knowledge now enforces takeaway-first (see [[Knowledge Priorities and Correlations]] curation rules).

## 5. Skill housekeeping (found during audit — action needed)

- **Duplicated skills with DIFFERENT content**: `check-quality`, `implement-feature`, `plan-feature`, `suggest-optimizations` exist in both `~/.claude/commands/*.md` and `~/.claude/skills/*/SKILL.md` with different text (edited at different times). Pick one home (recommend `skills/`), diff each pair, keep the better version, delete the other.
- **Exact duplicates**: `plan-update` and `scan-problems` (commands vs skills copies are identical) — delete the `commands/` copies.
- **`scan-problem.md` vs `scan-problems.md`** in commands — near-duplicates, keep one.
- **Usage reality**: transcripts show you almost never invoke skills by slash command (only `codex:rescue` 4×) — you phrase requests naturally. That's why the five new skills use auto-trigger descriptions instead of relying on you remembering names. Consider deleting `chorus` if you're not actively using it.
