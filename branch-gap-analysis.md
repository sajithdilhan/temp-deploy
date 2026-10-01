# RFC-009 vs `feature/rbac-permissions-redesign` — gap analysis

## Context

RFC-009 (the business-facing "Role and scope access assignments" proposal, in business review) must be
compared with what another developer built on this branch, and we must know how that build would land
on UAT (~200 users, ~150 projects). This file is the analysis itself; the "Next steps" at the end are
the only actions proposed.

Method: three Sonnet explorers (backend model; API/frontend/tests/docs; migration/UAT), plus direct
verification of the load-bearing claims (`ValidateApprovers`, `AllowedTransitions`, `GrantPolicy.MayGrant`,
`RoleAssignmentService.RevokeAsync`, requesting-unit derivation).

---

## 0. The headline finding — RFC-009 and the branch describe two different "current states"

- The branch (589 commits ahead of `main`) implements **RFC-008 (ACCEPTED 2026-09-27 by the product
  owner) → ADR-025 → PLN-018 (status DONE 2026-09-28, phases A–G)**. The role ladder is already retired:
  authority comes only from `role_assignments` + record relationships + 7 named positional rules.
- RFC-009's "Current behaviour (verified 2026-09-30)" describes **`main`** — `HasAtLeast`,
  `[Authorize(Roles)]`, `ROLE_RULES`, `roleGuard`. None of that exists on this branch any more.
  RFC-009 never mentions RFC-008, ADR-025 or PLN-018, yet says it "would supersede" ADR-010…025.
- **Consequence:** RFC-009 is not a greenfield proposal against this branch. The branch already has
  RFC-009's *architecture* (catalogue, evaluator, scoped assignments, audit, migration CLI) — and goes
  further (custom roles, expiry, suspension, delegation, project scope). What differs is the **role
  vocabulary and several business rules**, some of which directly **contradict decisions the product
  owner recorded in RFC-008 §21.1 three days earlier**. That governance conflict is the first thing to
  resolve (see §4, C1).

---

## 1. Role-by-role mapping (RFC-009 matrix → what the branch has)

| RFC-009 role | Branch equivalent | Fit |
|---|---|---|
| External User | **None.** RFC-008 Q7 removed external users; self-registration creates active **Staff, unplaced** (Q12). `EXTERNAL_USER` enum value is unused. | ✗ conflicts with D8 |
| Contributor (scoped) | **Staff** — baseline, **GLOBAL-only**, eligibility only (`project.lead/contributor_eligible`, `incident.maintainer_eligible`, `work_request.create_project`) + positional rules (own team, own active unit) | ~ concept matches, but not scoped |
| Approver (scoped) | **Approver** — baseline, **GLOBAL-only** `work_request.approver_eligible`; *where* is decided by the approver's **home seat** being in the requester's reporting line (ADR-016) | ~ same intent, different mechanism (D1) |
| Dashboard Viewer | **Portfolio Viewer** (SUBTREE default): `dashboard.management.view`, `dashboard.portfolio.view` **plus** `project.view`, `work_request.view` | ~ answers Q4 as "both dashboards + read-only detail" |
| Pain Point Moderator | **Reference Data Steward** (`pain_point.manage`, UNIT default) | ✓ rename only |
| User Administrator | **No such role.** Bundled inside Organisation Administrator. **Cannot be built as a custom role** — `CustomRoleRules` forbids `role.assign`/`role.manage` in custom roles | ✗ needs a new system role |
| Organisation Administrator | **Organisation Administrator** (SUBTREE): `organisation.manage` **+ `user.view/manage` + `role.assign` + `work_request.delete`** | ~ must be split from User Admin |
| System Administrator | **Administrator** (GLOBAL): operations, admin, scoring, audit; **no** approve-on-behalf, curation, FX, identity edit, role management | ✓ matches the matrix's boundary text (RFC-009 Q2) |
| Super Administrator | **Super Administrator** (GLOBAL), every action permission; only a Super Admin grants it; last-permanent-holder guard | ✓ |
| *(not in matrix)* | **Delivery Manager** (UNIT): all project delivery, initialise, choose delivery, backlog, incidents, change-request approval, dashboards | answers RFC-009 **Q1** ("new role") |
| *(not in matrix)* | **Project Manager (own projects)** (`member_manager`, baseline): manage any project you're a member of — preserves legacy MANAGER behaviour | extra |
| *(not in matrix)* | **Custom roles** composed by Super Admin (action permissions only) | extra |

---

## 2. Aligned (RFC-009 asks → branch already delivers)

**Model & architecture**
- Role + Scope assignments, effective access = union, no rank, no designation/seniority authority — `role_assignments`, `AccessContext` (`Application/Access/AccessContext.cs`).
- Scope inherited downward through `OrganisationHierarchy` (SUBTREE expands to active descendants); System/Super Admin unscoped (GLOBAL).
- Capability catalogue as data in Application — `PermissionCatalogue.cs` (47 permissions, action vs eligibility, allowed scopes, privileged/override flags).
- Evaluator answering RFC-009's three questions: `Decide` (P at U), `ReachOf` (units where P), and SQL-translatable `ViewableBy`/`ChangeableBy` expressions per resource policy.
- Assignments loaded per request (scoped `AccessContextProvider`, one indexed query), token format unchanged.
- `[Authorize(Roles=…)]` gone → `[RequirePermission]` + service-level checks; rank helpers quarantined in `Access/Legacy/` and banned by architecture tests (RFC-009 says "deleted" — kept only as a frozen parity oracle).
- One active assignment per user/role/scope — partial unique index `role_assignments_live_identity_key`.
- Indexed for the visibility predicates (`user_id`, `organisation_unit_id`, `role_id`, `project_id`, all `WHERE revoked_at IS NULL`) — covers RFC-009's `(user_id, revoked_at)` risk.
- Enum risk avoided: text + CHECK constraints instead of a new Postgres enum.

**Decisions**
- **D1 (partly)** HOD → GM with named approvers kept; approver must hold Approver eligibility.
- **D2** Automatic mapping from legacy role + unit (`--convert-legacy-assignments`, dry-run, idempotent); legacy `users.role` kept (frozen; drop needs a separately approved migration).
- **D3** Own records + own department: requester/named-approver relationships + positional rules (own team's projects, own active unit) preserve ADR-025.
- **D5 (partly)** Organisation Administrator can grant only within its subtree and cannot grant Org Admin / Administrator / Super Admin; administrative roles are central (Administrator/Super Admin).
- **Q2** answered the way the matrix's boundary text wants: Administrator does **not** approve; only Super Admin may act on a named approver's behalf, audited as `PRIVILEGED_ACTION`.

**Audit & logging**
- Every grant/revoke/suspend/resume/extend writes an `AuditTrail` row in the same `SaveChanges` with before/after snapshot (incl. reason) + `LogInformation`; role edits audited (`ROLE_CREATED`, `ROLE_PERMISSION_ADDED`, …). Historical audit rows untouched.

**Frontend**
- `hasAtLeast`, `ROLE_RANK`, `roleGuard`, `ROLE_RULES` gone (ESLint-banned); 15 `capabilityGuard` routes; ~26 `authz.can` sites render server flags only.
- People screen: responsibility column/filter; person page with responsibilities table; add-responsibility sheet with preview.

**Tests**: ~12.7k-case legacy parity harness, `GrantPolicyTests`, Phase G access matrix, escalation suite, JWT-level endpoint suites, audit assertions (`StaffGrantAudit`, `LocalPostgreSqlRoleAssignmentTests`).

---

## 3. Gaps (RFC-009 requires, branch lacks or contradicts)

Severity: **H** = contradicts a D-decision / blocks acceptance; **M** = missing feature; **L** = naming/cosmetic.

| # | RFC-009 | Branch today | Sev | Effort to close |
|---|---|---|---|---|
| G1 | **D4** strict self-approval for everyone | Requester **may be their own HOD** (documented intent in `ValidateApprovers`, `WorkRequestRuntimeService.cs:2255-2271`); Super Admin with `approve_on_behalf` can take **both** stages on **their own** request (`WorkRequestAccessPolicy.AllowedTransitions:217-233` has no requester exclusion) | H | S — two checks |
| G2 | **D1** Approver = assignment whose **scope covers the requesting unit** | GLOBAL eligibility + approver's **home seat** in requester's reporting line (ancestor-or-self). At act time only the global permission is re-checked (reporting line re-checked for delegates only) | H | M — eligibility perms are GLOBAL-only by catalogue design; needs a scoped approver permission + `HoldersOf` change + picker |
| G3 | **D6** no home unit; requesting unit chosen from assignments | Home seat (`users.organisation_unit_id`, `team_id`) is still load-bearing: requesting unit (client choice **rejected with 400**, ADR-012), approver reporting line, project-lead eligibility (must sit in delivery unit), dashboard default context (403 without a seat), pain-point selection (exact own unit), admin visibility of a person, team-based project visibility, Staff positional rules, unit-move resolution | H | **L** — the largest change; touches ~10 services and ADR-012/016 |
| G4 | **D8** External User; first grant confirms the account; "pending accounts" list for User Admins | Self-registration = **active Staff immediately** (can log in, raise incidents/change requests); no pending state, no list; unplaced people visible only to **GLOBAL** `user.manage` — a scoped Org Admin cannot even see them to place them | H | M |
| G5 | **D5** User Admin grants exactly Contributor, Approver, Dashboard Viewer, PPM | Org Admin grantable list = **Delivery Manager**, Steward, Approver, Portfolio Viewer, Staff. Delivery Manager (full project/incident/initialise powers) is grantable by a local admin | H | S — edit `SystemRoles` grantable list + `--sync-access` |
| G6 | Separate **User Administrator** and **Organisation Administrator** | One combined role; a User Admin cannot be modelled as a custom role | M | S–M — new system role + split permissions + migration of existing holders |
| G7 | **D7** reason mandatory on **every** grant and revoke | Reason required only for **temporary or privileged** grants (`GrantPolicy.cs:158`); revoke/suspend reason optional (UI says "Optional") | M | S |
| G8 | **D7** access review report (CSV) | Not present (no endpoint, no UI) | M | S–M |
| G9 | **D7** conflict warnings (e.g. User Admin + Approver overlapping; anything + Super Admin) | Not present (no SoD logic at all) | M | M |
| G10 | **D7** start/expiry dates **out of scope** | Branch **has** `valid_from/valid_until`, suspend/resume/extend, expiry sweep (hosted service off by default) | — over-delivery | decide: keep (hide in UI) or remove |
| G11 | `/api/users/me` returns assignments + effective permissions **per scope**; Angular `can(permission, unitId?)` | Returns 17 coarse navigation booleans **by design** (RFC-008 §9: client never evaluates scoped permissions); assignments only via admin endpoints | M | S–M (but is a deliberate security stance — clarify) |
| G12 | Server permission flags on **lists and dashboards** | Detail only (project, work request, incident; managed-user list has them). Dashboards use nav capabilities only | M | M |
| G13 | People screen shows **inherited** access | Not shown; "Who has authority here" (F6) and "Explain access" (F5) deferred | M | M |
| G14 | **Q5** team possibly on the Contributor assignment | No team on assignments; team stays on the user | M | depends on Q5 |
| G15 | Audit names `USER_ROLE_ASSIGNMENT_GRANTED/REVOKED`; migration audit names the system actor | `ROLE_ASSIGNED` / `ROLE_REVOKED` / `ROLE_ASSIGNMENT_SUSPENDED…`; conversion writes **one** `LEGACY_ASSIGNMENTS_CONVERTED` row per user (null actor), **not** one per derived grant | L | S |
| G16 | Shadow-mode evaluator phase logging mismatches | Existed (Phase C `--sync-access --check`), **retired in Phase G**; a UAT DB coming straight from `main` gets no shadow period — parity rests on the 12.7k-case differential test | M | M if wanted |
| G17 | Pre-check SQL, per-user **diff report**, mapping-override file | Dry-run prints **counts only** (per-user lines are Debug logs); no override file; RFC-009's `uat-access-precheck.sql` is **not in the repo** | H for UAT | M |
| G18 | Re-route in-flight requests before strict rules switch on | Nothing re-validates or re-routes named HOD/GM on in-flight requests (`ValidateApprovers` runs only on create/edit/submit) | M | S–M |
| G19 | Drop legacy columns after one stable release | Frozen indefinitely; drop requires separate approval | L | — |
| G20 | Playwright journeys per role | 7 e2e tests total; one per-role landing-shell check; **no** grant/revoke/roles/person-page journeys; PLN-018 still lists admin e2e as "not run" | M | M |

---

## 4. Needs clarification

**Governance (must be settled first)**
- **C1. Which RFC is authoritative?** RFC-008/ADR-025 were ACCEPTED on 2026-09-27; RFC-009 contradicts them on:
  - external users (Q7 "remove entirely" vs D8 "External User + pending");
  - self-registration (Q12 "active Staff" vs D8);
  - self-approval (requester-as-HOD allowed vs D4);
  - approver eligibility (ADR-016 seat-based vs D1 scope-based);
  - requesting unit (ADR-012 derived vs D6 chosen);
  - expiry dates (Q-delivered vs D7 out of scope).

  Does RFC-009 supersede RFC-008, or should it be rewritten as an **amendment on top of** the branch?
- **C2.** Do the RFC-009 authors and the business know this branch exists and is "DONE / ready for review"? RFC-009's impact estimate (rewrite ~30 frontend checks, 13+15 test suites by `Role`) is largely already done.

**Questions the branch already answered (propose these to the business as defaults)**

| RFC-009 Q | Branch answer |
|---|---|
| Q1 delivery rights | New **Delivery Manager** role (unit/subtree) + **Project Lead** relationship manages own project in full (RFC-008 Q10) + Project Manager (own projects) baseline for legacy managers |
| Q2 System Admin approve? | **No** — boundary text wins; only Super Admin acts on behalf, audited |
| Q3 moderators | All functions at their subtree: Delivery Manager + Steward + Org Admin @SUBTREE, plus Staff + Approver |
| Q4 Dashboard Viewer | Both dashboards + read-only project and request detail; personal dashboard open to all |
| Q5 own department | Exact home unit (+ own team's projects); team on the user; requesting unit = home unit |
| Teams | Org Admin (`organisation.manage`) within subtree |
| Audit log | Administrator + Super Admin |
| FX / scoring | FX: Super Admin only; scoring: Administrator + Super Admin |
| Recommend / not recommend | Not produced by the staged HOD/GM flow (only Approve / Reject→Changes requested / Request changes); survive only for change requests and legacy un-staged requests |
| Super/System Admin scoped? | Default GLOBAL; the schema allows scoping any non-baseline role |
| External User | No assignments = unplaced Staff (no such role) |

**Branch-internal questions for the other developer**
- C3. Org Admin can grant **Delivery Manager** — intended given D5?
- C4. Org Admin is granted at SUBTREE, so one anchored at a BU/BA can restructure BUs/BAs. RFC-009 says BA/BU are "centrally authorised".
- C5. Pain-point tags at BA/Group units are selectable only by requesters seated exactly there — intended?
- C6. Docs say a unit move offers "keep / **move with them** / end"; code offers only **Keep / Suspend / Revoke** (`admin-move-resolution-dialog.component.ts:26-28`).
- C7. "Seat-only shape rules" still marked *awaiting product-owner confirmation* (PLN-018).
- C8. `--migrate-external-users` is missing from `docs/deployment/production.md`.
- C9. Accepted MEDIUM risk: whoever manages a project's members can change an Administrator's/Super Admin's membership of that project.
- C10. Conversion ignores `users.team_id` and unit type. A legacy MANAGER with a team, or seated on a non-DEPARTMENT unit, **gains** initialise/assign-delivery at their seat; legacy denied this (`LegacyRules.DirectlyManagedDepartmentIds`). Is this an unannounced widening?
- C11. Migration seeds the role name "Member Manager"; code calls it "Project Manager (own projects)". Does `--sync-access` rename it?

---

## 5. How this implementation would handle UAT (~200 users, ~150 projects)

### 5.1 Cut-over path (as built)

Documented in `docs/deployment/production.md` §"Retiring the role ladder", plus one missing step:

1. **Snapshot the DB**. This is the real rollback path.
2. `pnpm db:migrate`: additive. Creates `permissions`, `roles`, `role_permissions`, `role_grantable_roles`, `role_assignments` and `approval_delegations`, plus two columns on `work_requests`. It seeds 9 system roles and writes **zero** assignments. `users.role`, `organisation_unit_id` and `team_id` are untouched.
3. `pnpm db:bootstrap`: reference seed + `--sync-access`. Skip `db:bootstrap:admin`; it refuses (exit 2) because admins already exist.
4. Pause admin edits to people. Run `pnpm run db:convert-access -- --dry-run`, which **always exits 1 on first run by design**. It prints counts only.
5. `pnpm run db:convert-access`: one advisory lock, one transaction per user (`FOR UPDATE`). Derived EXPLICIT grants are written by "System" with the reason "PLN-018 Phase G conversion (derived from legacy role)". It is idempotent and never re-grants what an admin later revoked.
6. **Not in the runbook:** `--migrate-external-users [--dry-run]`. Without it, existing `EXTERNAL_USER` accounts end up with **no grants at all**.
7. Run the dry-run again and expect exit 0. Then start the API.

⚠ **If the API starts after step 2 but before step 5, every user has zero authority.** A DB that never ran a Phase C–F build has no LEGACY rows, so not even the startup error log fires.

### 5.2 Mapping applied to each UAT user (`LegacyAssignmentSource.Derive`)

| Legacy | Gets |
|---|---|
| EXTERNAL_USER | nothing; becomes Staff, unplaced, only via `--migrate-external-users`, which also clears unit and team |
| USER | Staff |
| MANAGER | Staff, Approver, Project Manager (own projects) @GLOBAL; Delivery Manager + Steward @UNIT(seat) |
| MODERATOR | Staff, Approver @GLOBAL; Delivery Manager + Steward + Org Admin @SUBTREE(seat) |
| ADMIN | Staff, Approver, Administrator @GLOBAL; Steward @SUBTREE(seat) |
| SUPER_ADMIN | Super Administrator, Staff, Approver @GLOBAL |

At ~200 users this is roughly 600–1,000 `role_assignments` rows. Runtime is trivial.

**Edge cases to check in UAT data:**
- **Unplaced MANAGER/MODERATOR** (no `organisation_unit_id`): unit-scoped grants are skipped, so they **lose** all delivery and curation reach.
- **Seat on an inactive unit**: the grant is written but inert, so effectively lost.
- **Seat on a GROUP or BUSINESS_AREA unit**: a MANAGER's Delivery Manager @UNIT covers only that one unit, not the departments below.
- **MANAGER with a `team_id`, or on a non-DEPARTMENT unit**: **gains** initialise, assign-delivery and backlog (C10).
- **Inactive users** are converted too. That is harmless; they cannot log in.

### 5.3 Behaviour changes UAT users will notice

These are decided in RFC-008 §21.1 and announced in `docs/product/access-changes.md`.

**Losses:**
- A MANAGER can no longer change the status of **any** project company-wide, or administer **all** incidents, or delete others' drafts.
- An ADMIN can no longer act as any HOD/GM. Only a Super Administrator can, and it is audited.
- Unit-scoped managers lose descendant drill-down on the management and portfolio dashboards.
- Change requests are now approved by the delivering side and the project lead, not by "any MANAGER+".

**Gains:**
- Project leads manage their project in full.
- Completed and archived projects are readable by all staff.
- The requesting unit can view the resulting project.
- Named approvers can delegate their slot.

**Process change:** unit-scoped grants **no longer follow** a person who changes unit. An administrator must choose Keep, Suspend or Revoke.

### 5.4 Existing data

- **In-flight approvals.** Named approvers who were MANAGER+ become Approver, so their requests keep flowing. Nothing is re-routed. A request whose named approver no longer qualifies stalls until the requester edits and resubmits, or a Super Administrator acts on their behalf. Requests where the requester is their own HOD remain valid, which is exactly the D4 exposure.
- **~150 projects.** These stay reachable through relationships (lead, member, requester, own team) plus Delivery Manager reach at seats, and through Administrator/Super Admin globally. Projects whose owning, delivery and team units sit outside every manager's seat become editable only by global admins and the project's own lead/members. Measure this with the pre-check below.
- **Audit.** One `LEGACY_ASSIGNMENTS_CONVERTED` row per user, with no per-grant `ROLE_ASSIGNED` rows. Old audit rows keep their role names.

### 5.5 Performance at this scale

Fine. List filtering is SQL, with a unit-set `Contains` and an `EXISTS` on members. Each request loads all organisation units, which is fine for hundreds. Known non-blocking hot spots:
- Both dashboards still load all projects and filter in memory. This predates the branch.
- `ValidateApprovers` and the approver search reload all units each call.
- The assignment loader includes `role_permissions` unnecessarily for system roles.

### 5.6 Rollback

- **Schema.** Additive and reversible (Down drops the new tables), and the legacy columns are intact.
- **App rollback to `main` after cut-over.** Unsafe once anyone edits people: `users.role` is frozen and new users are written as USER.
- **Practical rollback.** Restore the snapshot.

### 5.7 What's missing for a safe UAT cut-over

These are RFC-009's own asks:
- Per-user diff report (legacy access vs derived assignments) for business sign-off.
- Pre-check queries, at minimum:
  - users per role and unit type;
  - unplaced MANAGER/MODERATOR users;
  - managers seated on inactive or GROUP/BA units;
  - managers with a team;
  - EXTERNAL_USER count;
  - in-flight requests with requester = HOD;
  - in-flight requests with an ADMIN as named approver;
  - projects with no Delivery Manager reach after mapping.
- A mapping-override mechanism.
- The runbook fix for `--migrate-external-users`.

---

## 6. Recommendation

- **Don't treat RFC-009 as a rewrite.** Re-base it as an amendment to RFC-008/ADR-025 on this branch. Most of RFC-009 becomes role composition plus a handful of rule changes:
  - **Small:** G1, G5, G7, G15.
  - **Medium:** G4, G6, G8, G9, G12, G17, G18.
  - **Large:** G3 (no home unit) is the only one. G2 depends on G3.
- **Resolve C1 with the product owner before anything else.** Two accepted-vs-proposed decision sets cannot both stand.
- **Run the UAT pre-check on a UAT copy before the business signs RFC-009.** The data shows how many people the edge cases in §5.2 actually hit.

## Next steps (proposed, on approval)

1. Publish this analysis as a private Artifact page, or save it as `docs/rfcs/RFC-009-assets/branch-gap-analysis.md`, so it can go to the RFC-009 reviewers.
2. Optionally write a read-only `uat-access-precheck.sql` covering the §5.7 queries. Validate it against the local `project_repo_dotnet` DB now, and against a UAT copy later.

## Verification of this analysis

Every claim above is cited to code by the explorers. Load-bearing ones were re-read directly:
- `WorkRequestRuntimeService.cs:2255-2308` (approvers, self-HOD);
- `:218-240` (requesting unit derived);
- `WorkRequestAccessPolicy.cs:217-245` (on-behalf without requester exclusion);
- `GrantPolicy.cs:150-162` (reason rule);
- `RoleAssignmentService.cs:432-449` (optional revoke reason);
- `docs/deployment/production.md:340-406` (cut-over);
- PLN-018 Phases C and G.
