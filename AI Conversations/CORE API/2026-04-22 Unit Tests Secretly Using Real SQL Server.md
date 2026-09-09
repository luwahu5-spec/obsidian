---
date: 2026-04-22
source: Codex (VS Code)
project: core-web-api
tags: [debugging, unit-tests, testdatabasefixture, sql-server, db-coupling, ci]
---

# Why "unit" tests created a real `UnitTest` database in SQL Server

## The mystery
A new endpoint's unit tests ("EF in-memory, of course") created an actual database named `UnitTest` on local SQL Server.

## Answer
`TestDatabaseFixture` is the single source of truth and it uses **real SQL Server**, not in-memory:
- `UseSqlServer(ConnectionString)` (TestDatabaseFixture.cs ~59)
- connection string contains `database=UnitTest` (~15)
- runs `EnsureDeleted()` + `EnsureCreated()` each fixture init.
Every controller test using `Fixture.CreateContext()` hits real SQL Server.

## CI behavior
SQL-backed controller tests are marked `[IgnoreOnLinuxFact]`; the Bitbucket pipeline runs on Linux (`mcr.microsoft.com/dotnet/sdk:8.0`) → they're **skipped in CI** and only run locally on Windows. So removing `TrustServerCertificate=True` from the fixture connection string doesn't matter for CI — locally it may break the TLS handshake if the server cert isn't trusted.

## Concept: "proper level of DB coupling (none!)"
The lead's comment decoded: in-memory EF **still counts as DB coupling** — the test exercises code through `DbContext`/EF, so it isn't an isolated unit test:
- `DbContext` in prod = coupling; EF in-memory provider in tests = still coupling; mocked service/repository with no EF = none.
- Such tests are being commented out first, to be rewritten so business logic tests need no DB at all.
