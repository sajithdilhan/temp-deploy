# RFC-009 — Role and scope access assignments

**Status:** PROPOSED — blocked on the business questions in [Open questions](#open-questions)
**Created:** 2026-09-30
**Backlog:** —
**Supersedes / superseded by:** would supersede parts of ADR-010, ADR-013, ADR-014, ADR-016, ADR-017 and ADR-025 once accepted
**Plan:** [PLN-019](../plans/PLN-019-rbac-role-scope-revamp.md)
**Source:** *CAPTAIN Proposed Access Matrix* (business document, 3 pages)

## Summary

Replace the single-rank role ladder with **Role + Scope assignments**. A user holds any number
of assignments, each granting one functional role at one organisation unit and the units
beneath it. Effective access is the union of the user's active assignments. Job designation
and seniority grant nothing.

UAT has more than 100 projects and 200 users. This RFC therefore also defines how existing
users are mapped, what changes for them, and how the cut-over is checked. It does not
authorise implementation. Five business questions must be answered first.

## Current behaviour

Verified in code on 2026-09-30.

### Role and scope

- **One role per user.** `users.role` is the Postgres enum `"UserRole"`
  (`Domain/Users/UserRole.cs`). The ladder is
  `EXTERNAL_USER < USER < MANAGER < MODERATOR < ADMIN < SUPER_ADMIN`.
- **Rank comparison.** `RoleHierarchy.HasAtLeast`
  (`Application/OrganisationAdministration/OrganisationAdministrationContracts.cs:176`) does
  the comparison. A second copy of the rank lives in `IncidentRuntimeContracts.cs:126`.
- **Scope is implied.** It comes from the user's single `organisation_unit_id` and optional
  `team_id`. Subtrees come from `OrganisationHierarchy`.
- **Role and position are independent** (ADR-014). A moderator administers its own subtree
  (ADR-017).
- **The role is read fresh on every request.** It is re-read from the database into the
  claims (`Api/Authentication/AuthenticationApiExtensions.cs:45-96`), so a role change takes
  effect immediately.
- **The actor abstraction.** `ICurrentActor` exposes `UserId`, `Email`, `Role`,
  `OrganisationUnitId` and `TeamId`.

### Where authorization rules live

Rules sit in three layers:

| Layer | Where |
|---|---|
| Controller attributes | `[Authorize(Roles="…")]` strings on `AdministrationReadsController`, `OrganisationUnitsController`, `TeamsController`, `ProjectsController`, `ProjectDocumentsController`, `ProjectBackfillsController` and `CurrentUserController` |
| Application policies | `ProjectReadVisibilityPolicy`, `ProjectMutationAuthorizationPolicy` (24 actions), `WorkRequestWorkflow` (`VisibleTo`, `Allowed`, `CanInitialize`, `CanCurateReferenceData`), `AdministrationAuthorization`, `UserMutationAuthorization`, `OrganisationReadAuthorization`, and the incident rules |
| Service checks | `WorkRequestRuntimeService.Allowed` (approval stages), `DashboardRuntimeService`, `PortfolioDashboardService`, `ProjectBackfillService`, `ExchangeRateService`, `ScoringConfigurationService` and `OrganisationAccessPolicy` |

### What each role can do today

| Role | Capabilities |
|---|---|
| EXTERNAL_USER | Nothing. Self-registration creates this role immediately, with no confirmation step. |
| USER | Submit requests. Read their own records and their own unit's non-draft work (ADR-025). Work on project tasks they own or claim, as a member. |
| MANAGER | Named HOD or GM approver. Management and portfolio dashboards for their subtree. Initialise projects for their own department. Edit projects, timelines, tasks, members, roadblocks, documents and status within scope. Administer incidents. Curate their own department's pain points. Backfill. |
| MODERATOR | Everything MANAGER can do, across their subtree. Also administers users (can assign USER or MANAGER) and the units and teams in that subtree (ADR-017). |
| ADMIN | Global operational reach and audit log. Scoring configuration. Curates reference data in their own subtree (ADR-013). |
| SUPER_ADMIN | Everything. Exchange rates. Edits other users' identity fields. |

### Approval

- Every project request names two approvers, an HOD and a GM, who act in that order
  (`WorkRequestRuntimeService.Allowed`, `ValidateApprovers`).
- Each approver must be active, MANAGER or above, and sit in the requester's unit or one of
  its ancestors (ADR-016).
- A requester may be their own HOD, but never their own GM.
- ADMIN and above may act at any stage, including on a request they raised themselves.

### Frontend

- `core/auth/roles.ts` provides `hasAtLeast`. `roleGuard(min)` guards about 8 routes, and
  about 30 component checks use the rank.
- The people screen offers a single role `<select>` and applies `ROLE_RULES`.
- Project detail already renders server-computed permission flags
  (`ProjectDetailPermissionsDto`).

### Audit

- `USER_ROLE_CHANGED`, `USER_ASSIGNMENT_CHANGED` and related actions are written through
  `AdministrationAudit.Create`.
- The snapshot stored with them embeds `Role`.

## Problem

- The business wants access to be **role × scope**, not a rank.
  - Today a manager is automatically an approver, a dashboard viewer and a pain-point curator.
  - A moderator is all of those plus a user and org administrator.
  - Nobody can be given one of those functions without the rest.
  - Nobody can hold different functions in different parts of the organisation.
- **Governance gaps:**
  - Self-approval is allowed at the HOD stage, and for admins at any stage.
  - A role change records no reason.
  - No review report shows who holds privileged access.

## Proposal

### Decisions already taken

Taken by the product side on 2026-09-30.

| # | Topic | Decision |
|---|---|---|
| D1 | Approval flow | **Keep HOD → GM with named approvers.** A named approver must hold an active **Approver** assignment whose scope covers the request's requesting unit. |
| D2 | UAT migration | **Map automatically** from the legacy role and unit. The business reviews and signs off a diff report before cut-over. The legacy column is kept for one release. |
| D3 | Contributor visibility | **Own records plus own department, always.** This preserves ADR-025. |
| D4 | Self-approval | **Strict for everyone.** Nobody approves their own request at either stage, admins included. |
| D5 | What a User Administrator may grant | **Contributor, Approver, Dashboard Viewer and Pain Point Moderator only**, and only at units inside their own scope. Administrative roles are granted centrally. |
| D6 | Home unit | **None.** A user's position comes from their assignments. The requesting unit is chosen from those assignments when a request is submitted. |
| D7 | Governance | **Required:** a mandatory reason on every grant and revoke, an access review report, and conflict warnings. **Out of scope:** assignment start and expiry dates. |
| D8 | External User | **Granting the first assignment confirms the account.** User Administrators get a "pending accounts" list. There is no separate account status. |

### Roles and scope

The roles are the ones in the matrix:

- External User
- Contributor
- Approver
- Dashboard Viewer
- Pain Point Moderator
- User Administrator
- Organisation Administrator
- System Administrator
- Super Administrator

Scope rules:

- Every assignment has a scope unit, except System and Super Administrator, which may be
  unscoped ("All").
- **Scope is inherited downwards** through the existing `OrganisationHierarchy`: Group,
  then Business Area, then Business Unit, then Department.

### Data model (sketch)

A new table, `user_role_assignments`:

| Column | Notes |
|---|---|
| `id` | |
| `user_id` | |
| `role` | New enum `"AccessRole"` |
| `organisation_unit_id` | Null means All. Allowed only for System and Super Administrator. |
| `team_id` | Nullable. Depends on Q5. |
| `reason` | Required |
| `assigned_by_user_id`, `assigned_at` | |
| `revoked_at`, `revoked_by_user_id`, `revoke_reason` | |

- Only one active assignment may exist per user, role and unit.
- The legacy `users.role` and `users.organisation_unit_id` columns become read-only for one
  release, then they are dropped. `"UserRole"` is dropped with them.
- Historical audit rows keep the old role names, and the audit viewer must keep rendering
  them.

### Authorization architecture

- **Capability catalogue.** A single Application-layer table maps each role to permissions,
  for example `Request.Submit`, `Request.Approve`, `Dashboard.View`, `PainPoint.Manage`,
  `User.Manage`, `Org.Manage`, plus the delivery permissions that Q1 settles. The matrix
  becomes data instead of scattered conditions.
- **`IAccessEvaluator`** in Application answers three things:
  - whether the actor holds permission P at unit U;
  - which units the actor holds P for;
  - a query-translatable predicate for list filtering.
- **Assignments on the actor.** `ICurrentActor` carries the active assignments. They are
  loaded in `OnTokenValidated` exactly as the role is today, so the token format does not
  change.
- **Replacements:**
  - Every rule listed under Current behaviour is rewritten against the evaluator.
  - `[Authorize(Roles=…)]` becomes permission-based policies.
  - Both rank helpers are deleted.
- **Rules that change with this RFC:**
  - Approver eligibility changes (D1).
  - Self-approval is refused both when approvers are named (`ValidateApprovers`) and at the
    status transition (`Allowed`) (D4).
  - User Administrators are limited to the grants in D5.

### API and frontend

- `GET /api/users/me` returns the user's assignments and their effective permissions per
  scope.
- New admin endpoints:
  - grant and revoke an assignment, reason required;
  - list pending accounts;
  - an access review report as CSV;
  - conflict warnings.
- **Audit.** Each grant or revoke writes an `AuditTrail` row in the same `SaveChanges`:
  `USER_ROLE_ASSIGNMENT_GRANTED` or `USER_ROLE_ASSIGNMENT_REVOKED`. The row records the
  assigner, the reason and the previous value. A `LogInformation` line is also written.
- **Angular:**
  - `can(permission, unitId?)` replaces `hasAtLeast`, and guards become permission-based.
  - The people screen gets an assignments panel that shows inherited access (governance
    note 3).
  - Server permission flags are extended from project detail to lists and dashboards.

### Migration and UAT cut-over

1. **Pre-check.** Run the read-only
   [`uat-access-precheck.sql`](./RFC-009-assets/uat-access-precheck.sql) on a copy of UAT
   now, and again before cut-over. It reports:
   - users per role and unit type, and users the mapping cannot place;
   - pending external accounts;
   - each moderator's subtree;
   - in-flight approvals, and whether each named approver is eligible;
   - self-approval exposure;
   - projects each user reaches only through their position;
   - pain-point tags by unit type;
   - open incidents by assignee role.
2. **Mapping migration.** It derives assignments from `role` and `organisation_unit_id` using
   the mapping table agreed under Q1 and Q3.
   - The reason reads `Migrated from legacy role <ROLE>`, and audit rows name the system
     actor.
   - Ambiguous data stops the migration with a `DO $$ … RAISE EXCEPTION` guard, following the
     precedent of `ScopeOrganisationAndTeamCodesToParent`.
3. **Diff report.** It lists each user's legacy access against their derived assignments. The
   business reviews and signs it off before release. Corrections go in through a
   mapping-override file before cut-over, or through the admin screen after it.
4. **Re-route in-flight requests.** Any request the pre-check flags (queries 5 and 6) must be
   re-routed before strict self-approval and the new eligibility rule are switched on.
5. **Release.**
   - The evaluator and the new rules ship in the same release as the backfill.
   - Take a database backup first. Rollback restores the application, not the database.
   - After release, re-run the report and do a smoke test with one UAT tester per role.
   - After one stable release, drop the legacy columns.

## Alternatives considered

### Alternative A — Keep the ladder and add a secondary "scope" field

This means one role plus a list of units. It is cheaper, but it cannot express different
functions in different units, which is the matrix's central example ("Contributor for
Systems BU, Dashboard Viewer for Commercial Aerospace"). Rejected.

### Alternative B — Reset everyone and reassign by hand

Every user would become Contributor or External User, and administrators would reassign.
It is simple, but 200+ users would lose access and in-flight approvals would be stranded.
Rejected in favour of D2.

### Alternative C — A generic permission-per-user grant table (no roles)

This is the most flexible option, but it is unmanageable for administrators and does not
match how the business describes access. Rejected. Roles stay the unit of assignment, and
permissions stay an internal catalogue.

## Benefits

- Functions can be granted independently, for example a Dashboard Viewer who cannot approve,
  or an Approver who cannot administer users.
- One user can hold different functions in different parts of the organisation.
- There is one authorization evaluator instead of three layers of rank checks.
- Self-approval is closed.
- Every access change carries a reason, and privileged access becomes reviewable.

## Risks

- **Access regressions in UAT.** Users may lose visibility or delivery rights that are not
  mapped. This is mitigated by the pre-check, the diff report, parity tests for mapped users,
  and a shadow-mode evaluator phase that logs mismatches without enforcing them.
- **Stranded approvals.** Mitigated by pre-check queries 5 and 6 and by re-routing before the
  switch.
- **Big-bang rewrite.** The rewrite touches almost every service and component. It is
  mitigated by phasing (PLN-019) and by keeping the legacy column until parity is proven.
- **Query performance.** Visibility predicates become unions over assignment scopes, so they
  need an index on `(user_id, revoked_at)` and a check of the dashboard queries.
- **Enum migration.** This is the repository's first `ALTER TYPE`. A new enum type is created
  instead, so the legacy one can be dropped cleanly.

## Impact

**Frontend:**
- Auth model: `AuthUser.role` becomes assignments plus permissions.
- Guards and the navigation.
- Dashboard default scope.
- Project view, incidents and backfill flags.
- Admin screens: people (role select, `ROLE_RULES`, role filter), overview (`usersByRole`),
  structure (scoped logic) and pain points.
- `ROLES.md`, and about 30 component checks with their specs.

**Backend:**
- Every authorization site listed in Current behaviour.
- `ICurrentActor`, the `OnTokenValidated` claims, approver eligibility and pickers,
  project-lead eligibility, and the member-candidate rank rule.
- `OrganisationAccessPolicy` assignment validation.
- `ProjectMutationContextLoader` ("directly managed department").
- `DemoDataSeeder`, `AdminBootstrapper` and the e2e seed (9 role users).

**Database:**
- A new table and a new enum.
- A backfill migration.
- Later, dropping `users.role`, `users.organisation_unit_id`, `users.team_id` (depending on
  Q5) and `"UserRole"`.

**Migration:** see [Migration and UAT cut-over](#migration-and-uat-cut-over).

**Security and permissions:**
- The whole model changes.
- Self-approval closes.
- Privilege escalation is bounded by D5.
- The backend stays authoritative, and the browser only renders server-computed permissions.

**Operational:**
- UAT testers need a briefing and a per-role retest script.
- The access review report becomes a periodic task for the platform owner.

**Tests:**
- About 13 unit and 15 integration suites build actors by `Role`, and all of them are
  rewritten.
- New tests: evaluator tests per role × scope × inheritance, JWT-level endpoint tests,
  grant and revoke audit tests, and Playwright journeys per role.

## Open questions

These need a business answer before this RFC can be accepted.

1. **Q1 — Project delivery rights.**
   - The matrix has no column for editing a project, its timeline, tasks, status, members,
     roadblocks, documents and assets.
   - Nor does it cover project initialisation or choosing the delivery team (today the HOD
     chooses, per ADR-015), the approved list or backlog, backfill, or incident
     administration.
   - Today MANAGER and above hold these within scope.
   - The options are a new role (for example "Project Manager"), folding them into Approver,
     or granting them only through a relationship to the project (team lead or member).
2. **Q2 — System Administrator.** The matrix gives it Approve = YES and Dashboard = All. Its
   boundary text says technical administration should be kept apart from business approval
   and management visibility. Which one wins?
3. **Q3 — Mapping existing moderators.** The three options:
   - all five functions at their unit (Approver, Dashboard Viewer, Pain Point Moderator, User
     Administrator and Organisation Administrator);
   - the business roles only;
   - a decision per person. Pre-check query 4 lists them.
4. **Q4 — What Dashboard Viewer covers.** Is it the portfolio dashboard, the management
   dashboard (prioritisation and delivery), or both? Does it include read-only project
   detail? The personal dashboard is presumably open to every non-external user.
5. **Q5 — Which department is a user's "own".** With no home unit (D6):
   - Do the exact units of a user's Contributor assignments count, or those units and their
     subtrees?
   - Where does a user's **Team** live, for example on the Contributor assignment?
   - When a user holds several Contributor assignments, how is the requesting unit chosen?

Further gaps in the matrix to raise with the business:

- **Teams.** Who manages them: the Organisation Administrator?
- **Audit log.** Who reads it: today ADMIN and above.
- **Exchange rates and scoring configuration.** Today SUPER_ADMIN and ADMIN.
- **Approval status names.** The matrix says Approver can "approve, return, reject", but
  today's statuses include *recommend* and *not recommend*. How do these map?
- **Scope of System and Super Administrator.** "Yes does not remove scope controls" sits
  beside "All". Are these two roles ever scoped?
- **Organisation Administrator.** It maintains Departments, while Business Areas and Business
  Units stay "centrally authorised". Who is central: System Administrator only?
- **Pain points at higher levels.** Tags exist only on Business Units and Departments
  (ADR-013). What does a Business Area or Group Pain Point Moderator assignment manage?
- **Conflict warnings (D7).** Which combinations are flagged, and do they warn or block?
  Suggested: User Administrator plus Approver in overlapping scope, and any role held
  together with Super Administrator.
- **External User.** Is it an assignment, or simply "no assignments"?

## Decision

Pending. Decisions D1–D8 above are agreed product direction. Acceptance needs answers to
Q1–Q5, and a signed-off mapping table from each legacy role to its assignments.
