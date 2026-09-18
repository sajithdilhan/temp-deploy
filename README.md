# UAT release runbook — observability, logging and moderator scope

**Target environment:** UAT on EC2 / IIS (`https://<host>/site`)
**Release date:** 2026-09-18
**Currently deployed:** `e5e9fae` ("moderator role update", 2026-09-14)
**Deploying:** `6298b94` plus the committed WF-021 pain-point scope fix

This is a release-specific runbook. The evergreen hosting reference is
[`production.md`](./production.md) — read its
[Blocking prerequisites](./production.md#blocking-prerequisites) if this is your first
deployment of Webapp. This document covers only what is different about *this* release.

---

## What is in this release

Five commits over three days, plus the WF-021 fix.

| Area | Change |
| --- | --- |
| API | Serilog structured logging, rolling daily log files, correlation IDs |
| API | New anonymous endpoint `POST /api/client-logs` for browser error reports |
| API | Catch-all unhandled-exception handler returning ProblemDetails with a `traceId` |
| API | Role claim now read from the database per request, not from the access token |
| API | Log statements added throughout the service layer (no behaviour change) |
| Web | Global error handler, HTTP observability interceptor, client log transport |
| Web | Moderator organisation selector on Manage Form Fields (WF-021) |
| Web | Work request detail shows requester **Name** instead of Employee ID |

**There are no EF Core migrations in this release.** Do not run `pnpm db:migrate`.
Rollback is a code-only redeploy with nothing to undo in the database.

---

## Risk summary

Read this before you start — it tells you what to watch for.

**The failure mode has changed.** Previous releases in this environment could only break a
feature. This one touches `Program.cs` startup: Serilog initialisation, four new NuGet
packages, three new middleware, and the exception-handler registration. A startup throw under
IIS is a site-wide `500.30`, not a degraded feature. Everything else in this release is
comparatively low risk.

**Two known verification gaps at the time of writing:**

- 223 PostgreSQL integration tests were skipped for lack of a local database — including
  `LocalPostgreSqlModeratorScopedAdministrationTests`, which covers the moderator scope change
  that testers will actually exercise.
- The Angular unit suite could not be executed (all 59 spec files fail at
  `angular:test-bed-init` with "Vitest failed to find the runner"). This is a pre-existing
  tooling problem — no test configuration changed since `e5e9fae` — but it means the changed
  web files ship without unit verification.

**What was verified:** 466/466 backend unit tests pass, 108/108 runnable integration tests
pass, and `pnpm nx run web:build:production` succeeds, so the whole frontend type-checks.

---

## 1. Pre-deploy checklist

- [ ] The WF-021 working-tree changes are committed. Do not deploy from a dirty tree — you
      will not be able to reproduce what you shipped.
- [ ] The current publish folder on the server is backed up. This is your rollback.
- [ ] You have RDP/Administrator access to the EC2 instance and know the application pool name.
- [ ] Testers have been told the deployment window, and that they must sign out and back in
      afterwards (see [§6](#6-tell-the-testers)).

---

## 2. Configuration changes

Nothing in this release is *strictly* required: the API starts with no new configuration,
because `LogFiles` defaults to `<publish>\logs` and the `ClientLogs` rate limits have code
defaults. The changes below are what make the release behave correctly rather than merely boot.

### 2.1 `appsettings.Production.json` — required in practice

This file lives on the server only. It is not in the repository, so `dotnet publish` does not
overwrite it and clearing the site directory on redeploy does not destroy it.

Merge these keys into the existing file:

```json
{
  "LogFiles": {
    "Directory": "D:\\Logs\\captain"
  },
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft.AspNetCore": "Warning",
        "Microsoft.EntityFrameworkCore.Database.Command": "Warning",
        "Microsoft.EntityFrameworkCore.Infrastructure": "Warning"
      }
    }
  }
}
```
#### Add missing Allowed domains. ####

**Do not skip the EF Core override.** `appsettings.json` ships
`Serilog:MinimumLevel:Default: Information` with an override for `Microsoft.AspNetCore` only.
`appsettings.Development.json` has an EF override, but that file is not read in UAT. EF Core
logs every executed SQL statement at Information, so without this override **every database
query is written to disk**. This is the difference between a log file measured in megabytes
per day and one measured in gigabytes.

Parameter values are not at risk — `EnableSensitiveDataLogging` is not enabled anywhere in the
codebase, so values render as `@p0='?'`. The concern is SQL text and volume, not data exposure.

Absolute paths in `LogFiles:Directory` work as expected: the value is passed through
`Path.Combine` with the content root, and a rooted path wins. A relative value resolves
against the content root.

Two things to avoid in this file:

- **`Logging:LogLevel` is now inert.** `builder.Host.UseSerilog(...)` replaces the logging
  factory, so the `Logging:LogLevel` section — and any `Logging__LogLevel__*` environment
  variable currently set on this server — no longer controls anything. `Serilog:MinimumLevel`
  is the only level knob now. Re-point any existing overrides.
- **Do not add a `Serilog:WriteTo` section.** Sinks are configured in code, and
  `ReadFrom.Configuration` is also called, so a `WriteTo` entry in JSON *adds* sinks on top of
  the code-configured console and file sinks rather than replacing them.

### 2.2 `web.config` — one temporary change

Use the `web.config` that `dotnet publish` emits. Do **not** use
[`deploy/iis/web.config`](../../deploy/iis/web.config) — that file is for the split
frontend/API proxy arrangement and is not used by this single-application layout.

For this deployment only, enable stdout logging:

```xml
<aspNetCore processPath="dotnet"
            arguments=".\ProjectRepository.Api.dll"
            stdoutLogEnabled="true"
            stdoutLogFile=".\logs\stdout"
            hostingModel="inprocess" />
```

This matters because of a specific failure mode in this release. If the Serilog log directory
cannot be created, the API **keeps starting** and writes
`File logging disabled: cannot create log directory '<path>': <reason>` to the console — which
IIS discards. Enabling stdout logging is the only way that warning, or a startup crash from the
new middleware, becomes visible. Turn it back off once the deployment is confirmed healthy
([§5](#5-post-deploy-verification)).

Note that the ASP.NET Core Module's default `stdoutLogFile` is `.\logs\stdout`, the same `logs`
folder Serilog would use by default. There is no filename collision (`stdout_*.log` versus
`api-*.log`), but it is a further reason to move Serilog's output to `D:\Logs\captain`.

**Do not set `LogFiles__Directory` in `web.config`.** The next publish replaces that file.
Use `appsettings.Production.json` instead.

### 2.3 Environment variables

**No new environment variables are required.** These remain unchanged:

| Variable | Value |
| --- | --- |
| `ASPNETCORE_ENVIRONMENT` | `Production` |
| `ConnectionStrings__DefaultConnection` | *(unchanged)* |
| `Authentication__Jwt__SigningKey` | *(unchanged)* |
| `Authentication__RefreshToken__SecureCookie` | `true` |
| `PathBase` | `/captain` |

Two notes:

- **The `.env` file is not read in production.** `RepositoryEnvironment.FindRootDotEnv` only
  loads a `.env` that sits beside an `nx.json`, which a publish output never contains. The
  `PathBase=/captain` → `PathBase=` change in `.env.example` in this release is therefore a
  local-development fix only and **cannot affect UAT**. Ignore it.
- The optional new settings, if you prefer environment variables to JSON, are
  `LogFiles__Enabled`, `LogFiles__Directory`,
  `ClientLogs__AuthenticatedPermitsPerMinute` and `ClientLogs__AnonymousPermitsPerMinute`.
  On IIS these live in `<environmentVariables>` under `<aspNetCore>`, which the next publish
  wipes — prefer `appsettings.Production.json`.

### 2.4 IIS-level changes

| What | Action | Why |
| --- | --- | --- |
| `D:\Logs\captain` | Create the directory; grant `IIS AppPool\<pool name>` **Modify** | The pool identity normally has read-only access to the site folder, so the default `logs` directory cannot be created and file logging silently disables itself |
| App pool → Advanced → **Maximum Worker Processes** | Confirm it is **1** | The file sink is configured without `shared: true`, so two worker processes would contend for `api-YYYYMMDD.log` |
| App pool → **Start Mode** | Confirm `AlwaysRunning` | Pre-existing requirement for `ProjectBackfillWorker` — verify it survived |
| App pool → **Idle Time-out (minutes)** | Confirm `0` | Same |
| Site → **Preload Enabled** | Confirm `True` | Same |

No new IIS modules are needed. No URL Rewrite or ARR changes. No request-filtering changes: the
new `X-Correlation-Id` request header passes through by default, and the 64 KB
`[RequestSizeLimit]` on `/api/client-logs` is well under IIS's 30 MB default
`maxAllowedContentLength`.

### 2.5 What does *not* need changing

Listed explicitly so the deployment is not over-configured.

- **Database.** No migrations. Do not run `pnpm db:migrate`.
- **CORS.** The `WithExposedHeaders(X-Correlation-Id)` addition only applies when
  `Cors:AllowedOrigins` is non-empty. In the single-application IIS layout it is empty,
  `UseCors` is skipped entirely, and same-origin JavaScript can read the header without CORS
  exposure. This change is a no-op here.
- **Rate limits.** The defaults — 12 reports/minute per signed-in user, 30/minute shared across
  all anonymous callers — are appropriate for UAT.
- **`PathBase`, Angular `baseHref`, proxy rules.** Unchanged.

---

## 3. Lower-risk alternative for the first deploy

If you would rather remove the file-sink hazard from the deployment entirely, set:

```json
{ "LogFiles": { "Enabled": false } }
```

Deploy, confirm the site is healthy, then enable file logging in a second pass. This costs one
extra restart and eliminates the only startup-path risk that configuration can cause.

---

## 4. Deploy sequence

```bash
# On the build machine
pnpm run build:deploy
# → nx reset → web:build:production → dotnet publish -c Release -o ./publish
# → node scripts/stage-web.mjs publish
```

Then on the server:

1. Back up the current publish folder.
2. Create `D:\Logs\captain` and grant the application pool identity **Modify**.
3. Write `appsettings.Production.json` ([§2.1](#21-appsettingsproductionjson--required-in-practice)).
4. Stop the site.
5. Copy the publish output into the site's physical path.
6. Edit the published `web.config` to set `stdoutLogEnabled="true"`
   ([§2.2](#22-webconfig--one-temporary-change)).
7. Start the site.

---

## 5. Post-deploy verification

Work through all six. Step 1 alone is not sufficient — `/api/health` is anonymous and returns
200 even on a completely misconfigured deployment.

- [ ] `GET /captain/api/health` returns 200.
- [ ] Sign in, open a work request, and load the dashboard. All succeed.
- [ ] A dated log file exists: `D:\Logs\captain\api-20260918.log`.
- [ ] `curl -i` against any API endpoint returns an `X-Correlation-Id` response header.
- [ ] `logs\stdout*` does **not** contain `File logging disabled`. If it does, the ACL in
      step 2 did not take effect.
- [ ] The log file is **not** full of `Executed DbCommand` lines. If it is, the EF Core
      override in `appsettings.Production.json` is not being read — check that
      `ASPNETCORE_ENVIRONMENT=Production` and that the file is in the site root.

Then:

- [ ] Set `stdoutLogEnabled` back to `false` and recycle the pool.

---

## 6. Tell the testers

The role claim is now stripped from the access token and re-read from the database on **every
request**. Combined with the moderator subtree changes (ADR-017) and the WF-021 selector, this
means:

- **Testers must sign out and sign back in** after the deployment.
- A user whose role is changed while signed in now has their permissions change immediately,
  rather than at token expiry. This is intended, but it will be reported as a bug if nobody is
  warned.
- Moderators will see a new organisation-unit selector on Manage Form Fields, offering Business
  Unit and Department targets within their own subtree.
- The work request detail panel now shows the requester's **Name** where it previously showed
  their Employee ID.

A burst of `chunk-load` entries in the browser error log immediately after the deployment is
expected — it means users still had the previous bundle open. See
[`production.md`](./production.md#browser-error-logs-and-correlation).

---

## 7. Rollback

No database changes were made, so rollback is a straight code redeploy:

1. Stop the site.
2. Restore the backed-up publish folder from step 1 of [§4](#4-deploy-sequence).
3. Start the site.

`appsettings.Production.json` can be left in place — the `LogFiles` and `Serilog` keys are
simply ignored by the older build.

---

## 8. Watch items after go-live

| Item | Why it matters | What to do |
| --- | --- | --- |
| Log volume on the EC2 volume | The file sink retains 30 daily files. Each is capped at 1 GB and, because `rollOnFileSizeLimit` is not set, writing **stops silently** for the rest of the day at that cap | Check `D:\Logs\captain` size after 24 hours and confirm the EF override is in place |
| Emails in log files | `AuthenticationService` logs `{Email}` on login success and failure, registration, refresh and logout. That is personal data at rest for 30 days | Decide whether this is acceptable for UAT; it is a code change if not |
| Application pool recycles | The file sink takes an exclusive lock. During an overlapped recycle two processes may contend for the same dated file; whether Serilog degrades or throws here was not verified | Watch for gaps in the log around recycle times |
| Manager access to pain-point curation | A known, deliberately open gap: the `pain-point-tags` endpoints carry no role attribute, so a `MANAGER` can reach curation through a direct API call. Documented in `work-request-form.component.ts` and WF-021 | Confirm this is an accepted decision for the UAT window |

---

## Appendix — change inventory

```
e5e9fae..HEAD + working tree
  89 files changed, 4270 insertions(+), 216 deletions(-)

  41  apps/api-dotnet             (mostly additive log statements)
  26  apps/web                    (features, shared UI)
  10  docs
   9  apps/web/.../observability  (new)
   3  .gitignore, .env.example, .claude/settings.json
```

Verification status at the time of writing:

| Check | Result |
| --- | --- |
| `dotnet test` — unit | 466 passed, 0 failed |
| `dotnet test` — integration | 108 passed, 0 failed, **223 skipped** (no local PostgreSQL) |
| `pnpm nx run web:build:production` | Succeeds |
| `pnpm nx run web:test` | **Cannot run** — 59/59 spec files fail at `angular:test-bed-init`. Pre-existing; no test tooling changed since `e5e9fae` |
| EF Core migrations | None |
| `e5e9fae` is an ancestor of `HEAD` | Yes — clean fast-forward |
