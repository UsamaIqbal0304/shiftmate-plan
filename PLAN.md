# ShiftMate — Core Architecture Plan

## 1. Context

Usama runs a chain of retail stores and currently has no software for staff
scheduling. He wants a web app where admins/managers manage employees, build
weekly schedules, and employees self-serve (view shifts, set availability,
request swaps) — with scheduling conflicts **prevented outright**, and when
that's not possible, **resolved immediately by whoever hit the conflict, in
that same interaction** — never left dangling for someone else to sort out
later. It has a real ownership hierarchy (a super admin, plus admins who
each own a subset of locations, plus single-location managers) and a
per-location activity log. This is a **brand-new project** — nothing exists
at `/home/sun` for it yet. Hosting/deployment is intentionally deprioritized
(see Flags below) so this plan can focus on getting the core structure right
first.

## 2. Guiding principles

These are the rules every design decision below follows. When in doubt,
come back to these rather than special-casing a page.

1. **Proactive first.** The system should make it hard to even attempt a
   conflicting assignment — pickers show, at a glance, who's free and who
   isn't for a given slot. Most of the time, no conflict is ever triggered.
2. **No dead ends.** If a conflict *is* triggered anyway (the person picked
   a flagged option, or approved something that turned out to conflict),
   the system never blocks with "go ask an admin" or "fix it somewhere
   else." It stops the commit, explains the conflict, and offers the small,
   fixed set of valid resolutions right there — the same person who hit the
   conflict resolves it, in the same interaction, immediately.
3. **One mechanism, everywhere.** There is exactly one conflict-checking
   primitive, one resolution-options policy table, and one modal component.
   Every place a conflict can occur (schedule builder, swap approval,
   Publish, drift on an already-published week) reuses all three rather
   than growing its own logic.
4. **Overrides are logged, not blocked.** Since nothing is left for an admin
   to manually gatekeep, every override/replace decision is written to the
   Activity Log with who decided it and why — the audit trail is the safety
   net, not a blocking approval step.
5. **Overtime is a different kind of thing.** Working >40h/week is often a
   legitimate, wanted outcome — it needs sign-off, not prevention. It's
   handled by separate logic (`lib/overtime.ts`) and must never be merged
   into the conflict mechanism above.
6. **Reuse over invention.** One `UserLocation` table expresses employee
   work-locations, manager scope, and admin scope by role instead of three
   separate tables; one `LocationDashboard` component serves both a
   Manager's home view and an Admin's drill-down; one `/team` page replaces
   what would otherwise be several admin pages. Simple, not sprawling.
7. **Config lives at the layer it actually varies at.** A setting is
   org-wide if it's policy/legal and must stay identical for everyone (e.g.
   the overtime threshold — it can't differ by store when hours are
   aggregated per employee across stores, see §3.7). It's location-level if
   it's genuinely operational per store (opening hours, display color).
   It's user×location-level if it's about one person's relationship to one
   store (pay rate, role scope). See §3.8 for the concrete list — new
   settings should be placed by asking "at what level does this actually
   differ?" rather than defaulting to global or bolting on a new table.

## 3. Core architecture

### 3.1 Tech stack

**Next.js 14+ (App Router) + TypeScript, PostgreSQL, Prisma ORM, Auth.js
(NextAuth v5, Credentials provider), Tailwind CSS + shadcn/ui.** One
codebase for frontend + API + server actions; type-safe DB access; clean
components out of the box.

**⚠️ Flag 1 — hosting (deferred):** default target Vercel + Neon
(~$20–40/mo); a self-hosted VPS is cheaper but more ops burden. Not decided
yet, not blocking.

**⚠️ Flag 2 — auth scope for v1 (deferred):** no self-serve signup, no
automated email-based password reset — admin sets/resets passwords
manually. Real email-based reset is a clean fast-follow.

### 3.2 Role hierarchy

```
SUPER_ADMIN — the owner. Sees/manages everything, all locations. Only
              SUPER_ADMIN creates Locations and grants ADMINs their
              location scopes.
ADMIN        — owns/oversees a SUBSET of locations (e.g. a regional
               partner across 2 of N stores) — "basically an owner" of
               that group, not of the whole business.
MANAGER      — scoped to exactly ONE location.
EMPLOYEE     — scoped to self; can work at one or more locations they're
               assigned to.
```

One join table, reused, no separate grants table: `UserLocation` rows mean
different things depending on the row-owner's role — EMPLOYEE row = "can be
scheduled at this location," MANAGER row = "manages this one location"
(exactly one row), ADMIN rows = "has admin-tier access to this location"
(their subset). SUPER_ADMIN needs no `UserLocation` rows — access is
implicit/global.

### 3.3 Data model

Core Prisma models: `User` (role, active — no pay rate here, see below),
`Location` (name, address, active, plus per-store config fields — see
§3.8: `openTime`/`closeTime`, `colorHex`), `UserLocation` (see
above; for EMPLOYEE rows also carries `payRateCents` — pay is per employee
**per location**, not a single flat rate), `OrgSettings` (singleton row —
see §3.8), `Availability` (recurring `dayOfWeek` rows +
one-off `date` exception rows, exceptions win), `Shift` (location, date,
`startMinute`/`endMinute`, assigned user, status
DRAFT/PUBLISHED/CANCELLED — see "Overnight shifts" below),
`ShiftSwapRequest` (shift, requester, target, status
PENDING/APPROVED/REJECTED/CANCELLED), `TimeOffRequest` (see below),
`OvertimeApproval` (one row per employee per week, carrying the
`approvedMinutes` it was granted for — see §3.7 on stale approvals), and:

```prisma
model ActivityLog {
  id         String          @id @default(cuid())
  action     ActivityAction
  actorId    String
  actor      User            @relation(fields: [actorId], references: [id])
  locationId String?         // null = org-level event (e.g. Location created)
  location   Location?       @relation(fields: [locationId], references: [id])
  targetType String?         // "Shift" | "User" | "ShiftSwapRequest" | ...
  targetId   String?
  metadata   Json?           // denormalized display fields, incl. conflict
                              // type + overridden shift id when relevant.
                              // Never the time-off `reason` text (§3.10).
  createdAt  DateTime        @default(now())
  @@index([locationId, createdAt])
  @@index([actorId, createdAt])
}

enum ActivityAction {
  SHIFT_CREATED SHIFT_UPDATED SHIFT_REASSIGNED SHIFT_CANCELLED
  WEEK_PUBLISHED SWAP_REQUESTED SWAP_APPROVED SWAP_REJECTED
  SWAP_CANCELLED CONFLICT_FLAGGED CONFLICT_OVERRIDDEN
  OVERTIME_APPROVED AVAILABILITY_CHANGED
  EMPLOYEE_ADDED EMPLOYEE_REMOVED LOCATION_CREATED GRANT_CHANGED
  TIME_OFF_REQUESTED TIME_OFF_APPROVED TIME_OFF_DENIED TIME_OFF_CANCELLED
}
```

**Overnight shifts — store minutes, not `"HH:mm"` strings.** A retail
closing shift of 22:00–06:00 has `end < start`, so any string or
same-day comparison silently reports "no overlap" — a correctness bug
sitting inside the one primitive everything in §3.4 depends on. `Shift`
therefore stores `startMinute` (minutes from midnight on `date`, 0–1439)
and `endMinute` (minutes from the *same* midnight, so an overnight shift
is simply `1320 → 1800`, i.e. it may exceed 1440 rather than wrapping).
Overlap is then plain integer math on absolute minute ranges, correct
across midnight with no special case. `"HH:mm"` remains the *display*
format only, converted at the UI edge. Consequences:
- `checkAssignmentConflict` compares `[startMinute, endMinute)` ranges,
  including against the previous day's shifts (an overnight shift
  ending 06:00 conflicts with a 05:00 start the next calendar day).
- Hours worked for overtime and labor cost is `endMinute - startMinute`
  — no date arithmetic, no DST edge case.
- `Location.openTime`/`closeTime` (§3.8) stay `"HH:mm"`, since they are
  a cosmetic pre-fill hint and never take part in overlap math.

**Time off — a request with an approval, not a self-marked flag.**
`Availability` lets an employee say "I'd rather not work Tuesdays"; it
is unilateral and §3.4 lets an admin override it with one click. That is
deliberately too weak for a booked vacation, so time off is its own
model rather than an availability exception:
```prisma
model TimeOffRequest {
  id         String             @id @default(cuid())
  userId     String
  user       User               @relation(fields: [userId], references: [id])
  startDate  DateTime
  endDate    DateTime           // inclusive; single day = same as startDate
  reason     String?
  status     TimeOffStatus      // PENDING | APPROVED | DENIED | CANCELLED
  decidedById String?
  decidedAt  DateTime?
  createdAt  DateTime           @default(now())
  @@index([userId, startDate])
  @@index([status, startDate])
}
```
It reuses the swap request's shape and lifecycle wholesale (§2.6):
request → the other party approves/denies → logged. Decided by
admin-tier or the employee's Manager; the requester can cancel their own
`PENDING` request, exactly as with swaps (§3.7). An **`APPROVED`**
request makes the employee `UNAVAILABLE` for those dates through the
*existing* `checkAssignmentConflict` primitive — no second conflict type,
no parallel mechanism, so scheduling over approved leave surfaces the
same `<ConflictModal>` with the same override-and-log behavior (§2.3).
`PENDING` and `DENIED` requests have no scheduling effect at all.

Conflicts are computed at query/commit time, never persisted as their own
records (they either get resolved immediately, per §3.4, or never occur).
`OvertimeApproval` is the one thing that *is* persisted, since it's an
auditable sign-off. No email/SMS delivery in v1 (in-app only).

**One timezone for the whole business.** All shift times are naive local
times — no `timezone` field on `Location`, no UTC conversion. Three stores
in one chain share a timezone, and a stored-but-unused timezone field
reads as supported when the overlap math would in fact ignore it, which
is worse than not having it. If stores ever do span timezones, that is a
deliberate migration: `checkAssignmentConflict` would compare each shift
via its location's zone. Until then this assumption is written down here
rather than left implicit.

**Pay rate — per employee, per location.** `payRateCents` lives on
`UserLocation`, not `User` — an employee working at two stores can have two
different rates (covers different cities/minimum-wage zones without a
migration later). Consequences to build consistently from milestone 1:
- Labor cost for a shift always uses the *shift's location's* rate for the
  assigned employee (`UserLocation` row for that user+location), never a
  single number off the `User`.
- The `/team` add/edit form sets a location-scoped rate per location grant,
  not one field on the person.
- Adding an employee's second `UserLocation` row (a new store) must require
  setting a rate for it — no location grant is allowed to exist without one,
  otherwise labor cost silently undercounts hours at that store.

**Pay rate visibility — viewing is broader than editing.** Viewing: any
admin-tier user (their accessible locations), a Manager (their one
location's employees), and an employee themselves (their own rate only,
never a coworker's) can all see pay rate. Editing stays admin-tier only —
changing pay is a more deliberate action than seeing it, and nothing in
this request asked to widen who can change it. Enforced by one new helper,
`canViewPayRate(viewer, targetUserId, locationId)`, alongside the existing
`requireAdminTier` (which now gates *editing* only, not viewing).

### 3.4 The unified conflict mechanism

**One backend primitive.** `checkAssignmentConflict(userId, date,
startMinute, endMinute, excludeShiftId?)` in `lib/conflicts.ts` → `null`,
or `{ type: 'DOUBLE_BOOKING' | 'UNAVAILABLE', ...details }`. This is the
single source of truth, called synchronously by every action that would
assign a user to a shift. It works in absolute minute ranges so overnight
shifts compare correctly (§3.3), and it recognizes exactly two sources of
`UNAVAILABLE` — an `Availability` row and an **`APPROVED`
`TimeOffRequest`** covering the date — which the returned details
distinguish so the modal can say *why* ("marked unavailable" vs
"approved time off"), without either becoming a separate conflict type.

**One server contract.** Every such action — create-and-assign a shift,
reassign a shift, approve a swap — attempts the write. If
`checkAssignmentConflict` is clear, it commits immediately (the common,
friction-free path). If it finds a conflict, the action does **not** error
and does **not** silently commit — it returns a `CONFLICT` result carrying
the conflict details. The caller re-submits the same action with an
explicit `resolution` field once the user has decided; the server
re-validates and applies it atomically (nothing is ever left half-done).

**One frontend component.** `<ConflictModal>` renders whatever conflict
details + resolution options it's given — no page builds its own conflict
UI. The valid options are looked up from one fixed policy table:

| Where | Who's deciding | Conflict type | Options offered |
|---|---|---|---|
| Schedule builder assign/reassign | Admin/Manager | Double-booking, other shift **in scope** | Replace the other shift with this one / Cancel this assignment |
| Schedule builder assign/reassign | Admin/Manager | Double-booking, other shift **out of scope** | Cancel this assignment (only — see "Scope bounds the options" below) |
| Schedule builder assign/reassign | Admin/Manager | Marked unavailable / approved time off | Assign anyway (overrides it) / Cancel this assignment |
| Swap approval | Approving employee | Double-booking | Replace my conflicting shift with this one / Reject this swap |
| Swap approval | Approving employee | Marked unavailable / approved time off | Approve anyway (overrides my own unavailability) / Reject this swap |
| Publish Week, and drift on an already-published week (§3.7) | Admin/Manager | Either, per flagged shift | The matching schedule-builder row above, scope-filtered the same way, resolved inline per shift |

Whichever option is chosen executes immediately server-side and is logged
to `ActivityLog` as `CONFLICT_OVERRIDDEN` (or the more specific
`SHIFT_REASSIGNED`/`SWAP_APPROVED` when it's a straightforward replace) with
both shifts involved — this is what makes "resolve it yourself, right now"
safe: there's always a record of who decided what.

**Scope bounds the options — a decider can never act on a shift outside
their own locations.** A MANAGER is location-scoped exactly like an
EMPLOYEE: one store, and nothing outside it. So when the *conflicting*
shift sits at a location the decider can't access, the "Replace the other
shift" option is not offered at all — it isn't disabled-with-explanation,
it simply isn't in the list, because offering it would let a Manager
cancel another store's shift. What they get instead is the redacted fact
plus the one action they're entitled to take:

> *Sana is already scheduled at another location for part of this time.*
> **Cancel this assignment** · *(or assign anyway — see below)*

The other store is never named, nor its times shown — the same redaction
§3.7 already applies to cross-location overtime, now applied consistently
here rather than contradicting it. This is a `lib/authz.ts` decision, not
a UI one: `getResolutionOptions(viewer, conflict)` filters the policy
table by `getAccessibleLocationIds(viewer)`, so the server rejects an
out-of-scope `resolution` even if a client sends one. Admin-tier users
whose scope *does* cover both locations see the unredacted detail and the
full option set — same table, same function, different scope.

The swap-approval rows are the one place an EMPLOYEE may act on a shift
at another location, and that is correct rather than an exception: the
conflicting shift there is *their own*, so "Replace my conflicting shift"
touches only their own assignment. Scope filtering asks whether the
decider may act on that shift, not which location it sits at.

**Picker behavior stays proactive, not a hard filter.** In the schedule
builder's employee picker, candidates who would conflict for the exact slot
(unavailable *or* already double-booked) are visually flagged (badge +
reason, redacted to "booked at another location" when that shift is
outside the viewer's scope) so most of the time the admin just picks
someone else — zero
friction, conflict never triggered. But nothing is disabled/unselectable:
picking a flagged candidate simply guarantees `<ConflictModal>` appears on
commit, per the table above. Both conflict types behave identically:
neither is a hard block, and neither has a privileged escape hatch.

**Publish Week** runs a safety-net sweep, `getWeekPublishSafetyIssues`
(the same primitive across the week), catching only drift — e.g. an
employee's availability changed *after* their shift was assigned. It
should rarely fire. When it does, Publish shows the same
`<ConflictModal>` per flagged shift, resolved inline in the Publish flow
itself — never a dead-end block requiring the admin to go elsewhere. The
same sweep also re-runs after publication whenever its inputs change; see
§3.7, "Drift on an already-published week," for that half.

**Overtime stays separate.** `lib/overtime.ts` computes per-employee weekly
hours and gates Publish behind an `OvertimeApproval` sign-off. It shares no
code path with `checkAssignmentConflict` — OT is a sign-off, not a
conflict.

### 3.5 Dashboard architecture

Single `/dashboard` route, one server component branching on role, three
reusable pieces:

- **SUPER_ADMIN / ADMIN** → `AdminDashboard` — defaults to a consolidated
  view across all accessible locations (labor cost, OT flags, headcount),
  with a location filter. Selecting one location swaps in
  `LocationDashboard` for it — the *same* manage-oriented component a
  Location Manager sees, not a separate read-only summary.
- **MANAGER** → `LocationDashboard` directly, no filter chrome. Now that
  pay rate is Manager-visible, this view includes a per-employee cost
  breakdown for the store, not just the aggregate total.
- **EMPLOYEE** → `PersonalDashboard` — own upcoming hours, next shift,
  pending swaps, weekly hours vs. the configured threshold (§3.8 — never
  a hardcoded 40), and an **estimated earnings**
  figure for the week (scheduled hours × their own per-location rate) since
  employees can see their own pay rate. No management capability, no
  visibility into coworkers' rates.

### 3.6 Activity log

`/activity`, gated to admin-tier only (`SUPER_ADMIN`/`ADMIN`) — Managers and
Employees never see it, even for their own location. Same location-filter
pattern as the dashboard: SUPER_ADMIN gets the full feed (incl. org-level
`locationId = null` events) plus a per-location filter; ADMIN's feed is
constrained to their accessible locations. Every conflict override is
visible here with full detail — since no conflict waits on an admin to
gatekeep it (§2.4), this log is the accountability mechanism in place of
that approval step.

### 3.7 Closing the loop: state transitions and edge cases

Every entity with a status/active flag needs every transition defined —
otherwise something ends up stuck with no path forward, which is exactly
what §3.4 exists to avoid for conflicts. Applying the same principle
elsewhere:

**Shifts and publishing**

- **Editing a `PUBLISHED` shift.** Publish is not a one-way lock — admins/
  managers can still edit or reassign a published shift directly (no
  "unpublish" step). Every such edit re-runs `checkAssignmentConflict` and
  goes through `<ConflictModal>` exactly like a DRAFT assignment; the
  employee just sees the updated shift. **A published week stays live:**
  any later change to it — edit, reassign, cancel, or an availability or
  time-off change that now collides with it — updates the published
  schedule in place and writes an `ActivityLog` entry, rather than the
  week being frozen at Publish. There is no second "republish" step and
  no silent divergence between what was published and what is true.
- **Drift on an already-published week.** `getWeekPublishSafetyIssues` is
  no longer only a Publish-time sweep: the same primitive runs whenever
  something that *feeds* it changes (an `Availability` edit, a
  `TimeOffRequest` approval) for any week that is already published. A
  new collision doesn't block anything and doesn't pop a modal at the
  employee who caused it — it is logged and surfaced as a flag on the
  affected shift in the schedule grid, plus a `<NavBadge>` count for that
  location's Manager (§3.9), who resolves it with the ordinary
  `<ConflictModal>` when they get to it. Without this, the sweep only
  ever protected DRAFT weeks and published ones drifted unnoticed. Logged
  as `CONFLICT_FLAGGED`.
- **`CANCELLED` shifts.** Terminal — a cancelled shift is never revived;
  scheduling that slot again means creating a new shift. It is excluded
  from conflict checks (otherwise a cancelled shift would keep blocking
  its own slot), from weekly overtime hours, and from labor cost. It
  stays queryable for the Activity Log and history. Cancelling is
  therefore always safe and never traps anything.
- **Past shifts are read-only.** Shifts whose date has passed can't be
  edited, reassigned or cancelled — they're the basis of labor-cost
  history, and conflict checks don't run against the past either. Fixing
  a genuine mistake in history is deliberately out of scope for v1.
- **Abandoned `DRAFT` weeks.** `DRAFT` shifts in past weeks are hidden
  from the schedule grid by default (a "show unpublished history" toggle
  reveals them) so a week someone started and abandoned doesn't clutter
  the grid forever or get mistaken for real coverage.

**Swaps**

- **Swaps are employee-to-employee.** A swap needs no manager approval:
  the target employee accepting *is* the approval, and the reassignment
  commits immediately. `SWAP_REQUESTED`/`SWAP_APPROVED`/`SWAP_REJECTED`
  in the Activity Log is how managers and admins stay aware of it, rather
  than a gate they have to clear.
- **Swap request cancellation.** A requester can cancel their own `PENDING`
  swap before the target responds (`ShiftSwapRequest.status → CANCELLED`).
  Without this, a requester who changes their mind has no way out except
  waiting on the target.
- **A `PENDING` swap whose shift moved underneath it.** If the shift is
  reassigned away from the requester or cancelled while a swap request is
  open, the request is automatically set to `CANCELLED` with a logged
  reason, and both parties see it disappear from `/swaps` with a one-line
  note. The alternative — asking a user to adjudicate a swap for a shift
  that is no longer theirs — is exactly the dead end §2.2 rules out.

**Time off**

- **Who approves time off.** The Manager of the location the request
  concerns, plus admin-tier for their accessible locations. There is no
  escalation for employees who work at two stores — each Manager decides
  for their own store, consistent with Managers being location-scoped
  everywhere else (§3.4).
- **Approving time off that collides with existing shifts.** The approver
  is warned, not blocked: `approveTimeOff` first runs the same sweep over
  the requested dates and, if the employee already has shifts there,
  shows them inline — *"Approving this leaves 2 shifts unassigned:
  Tue 9–5, Thu 12–8."* The approver confirms, and the time off is
  approved regardless; the now-colliding shifts are flagged per the entry
  above for the Manager to reassign or cancel when convenient. This is
  deliberately a warning rather than a hard prerequisite: whether someone
  gets their leave shouldn't depend on the schedule being tidied first,
  and the decision of what to do with the shifts belongs to the
  manager/admin, after the fact. An employee may cancel their own
  `PENDING` request (`TIME_OFF_CANCELLED`); a `DENIED` or `CANCELLED`
  request has no scheduling effect.

**People and access**

- **Changing someone's role.** Because a MANAGER is defined as holding
  exactly one `UserLocation` row (§3.2), the role form reconciles grants
  as part of the same transaction: promoting an EMPLOYEE to MANAGER
  requires picking which single location they manage; demoting a MANAGER
  requires saying which locations they remain schedulable at (possibly
  none). A role change is never saved in a state that violates its own
  tier's grant invariant.
- **Revoking a single location grant.** Same rule as deactivating a user:
  a `UserLocation` row can't be removed while that user still
  has future `DRAFT`/`PUBLISHED` shifts *at that location*; the form
  lists them to reassign or cancel first. Without this, revoking a grant
  is a quiet back door around the deactivation guard.
- **Never locking the business out.** The last active SUPER_ADMIN cannot
  be demoted or deactivated, and no user can change their own role — with
  no self-serve signup and no email-based reset (Flag 2), either action
  would strand the account permanently with no recovery path through the
  UI. Enforced in `lib/authz.ts`, not just hidden in the form.
- **Reactivation.** Setting a `User` or `Location` back to `active` is
  always allowed and restores nothing implicitly — no past shift,
  cancelled shift, or removed grant comes back. Reactivation is a fresh
  start, which keeps it a safe, un-surprising action.
- **Deactivating an employee with future shifts.** Setting a `User` inactive
  is blocked until their future `DRAFT`/`PUBLISHED` shifts are each
  reassigned or cancelled — the deactivation flow surfaces that list
  up front rather than silently orphaning shifts. Same "resolve it here,
  now" principle as conflicts.
- **Deactivating a location.** Same rule: a `Location` can't be deactivated
  while it still has future `DRAFT`/`PUBLISHED` shifts. Past shifts and
  `UserLocation` history are kept for reporting either way (soft
  delete via `active`, never a hard delete).
- **Bootstrapping the first account.** Since there's no self-serve signup
  (Flag 2) and only SUPER_ADMIN can grant roles, the very first SUPER_ADMIN
  has nothing to be invited by — it's created by the `prisma/seed.ts` /
  a one-time setup script, not through the UI. Every account after that
  is created by someone already in the system.

**Overtime**

- **Stale overtime approvals.** `OvertimeApproval` stores the
  `approvedMinutes` it was granted for, not just "approved." If the
  employee's recomputed weekly total later exceeds that figure, the
  approval no longer covers it and Publish asks for a fresh sign-off
  showing both numbers. Hours *dropping* back under the threshold simply
  makes the approval moot rather than requiring anything. Otherwise a
  sign-off for 41h silently authorizes 55h.
- **Overtime, cross-location visibility.** Overtime must be computed
  **org-wide across all of an employee's locations for the week**, not
  per-location — an employee working 25h at Store A and 25h at Store B is
  over 40 total even though neither location alone shows it, and this is
  the same cross-location logic already used for double-booking (§3.4).
  Practical effect: a location-scoped Publish action can be blocked by
  hours the Manager can't otherwise see, so a Manager sees only "this
  employee is over 40h total and needs admin approval," never the other
  location's shift detail; the Admin/Super Admin who approves OT
  (already admin-tier-only) sees the full cross-location breakdown. This
  aggregation is what makes OT catch-up correctly regardless of which
  location's week gets published first or last for a given employee.

**Concurrency**

- **Atomicity of conflict resolution.** Every check-then-commit sequence in
  §3.4 (assign, reassign, swap-approve, publish-resolve) runs inside a
  single Prisma transaction — the conflict check and the write happen
  atomically, so two admins acting at the same moment can't both "win" and
  recreate the exact conflict the modal was meant to prevent.

### 3.8 Where configuration lives (per-store customization)

Three layers, each editable by whoever already manages at that layer —
nothing new needed beyond fields on models that already exist, except one
small singleton table for the org-wide layer:

**Org-wide** (`OrgSettings`, one row, SUPER_ADMIN-only, edited at
`/admin/settings`):
```prisma
model OrgSettings {
  id                       String  @id @default(cuid())
  overtimeWeeklyThresholdMinutes Int  @default(2400) // 40h
  overtimeMultiplier       Float?  // null = not applied to labor cost in v1
  currency                 String  @default("USD")
  weekStartDay             Int     @default(0) // 0=Sunday..6=Saturday
}
```
The 40h rule and the "don't auto-multiply OT by 1.5x" decision are
settings, not assumptions baked into code — the owner can change either
without a release. Note the v1 consequence of `overtimeMultiplier` being
null: labor cost counts overtime hours at the base rate, so it
understates true cost until a multiplier is set.

`weekStartDay` must be read from `OrgSettings` by every week-scoped query
— schedule grid, Publish, OT calculation, dashboard rollup — rather than
each picking its own Sunday-or-Monday convention. One definition,
referenced everywhere.

All four stay org-wide rather than per-location: OT is computed from an
employee's *total* hours across all their locations (§3.7), so a
per-location threshold or week boundary would be incoherent — there is
only ever one combined week to compare against one threshold.

**Per-location** (fields directly on `Location`, edited by SUPER_ADMIN at
`/admin/locations`, visible to whoever can already see that location):
- `openTime` / `closeTime` (nullable `"HH:mm"`) — a soft default only: the
  schedule builder pre-fills new shifts within this range and shows a
  visual hint if a shift falls outside it. Deliberately **not** part of the
  §3.4 conflict mechanism (it's a store-hours hint, not an
  employee-scheduling conflict) — no modal, no blocking.
- `colorHex` — a small accent color per store, used as a chip/legend color
  in the schedule grid, consolidated dashboard, and activity feed so a
  multi-location view is easy to scan at a glance. Purely cosmetic, cheap,
  fits "clean and modern."

**Per employee-at-location** (`UserLocation` fields): `payRateCents` (§3.3)
— the one thing that's genuinely about *this person at this store*, not
the store itself or the business as a whole.

### 3.9 "Notify me about conflicts" — what that means here

The original ask was to *be notified* when a conflict happens. The
architecture above answers most of that by removing the thing worth
notifying about: §3.4 makes a conflict impossible to leave lying around,
since whoever hits it resolves it in that same interaction. What's left
is the genuinely useful half — knowing that someone *overrode* one, and
knowing something is waiting on you. That is deliberately **not** an
email/SMS/push system (no delivery infrastructure in v1, §3.3), and not
a `Notification` table either. It's one server-side count function,
`getPendingCounts(user)` in `lib/activity.ts`, returning small numbers
rendered by `<NavBadge>`:

- **Employee** — swap requests awaiting my approval, plus my own
  time-off requests that got decided since I last looked. Without this,
  a swap request sits in `/swaps` until the target happens to check,
  which is the one flow in the app that genuinely stalls without a nudge.
- **Manager** — time-off requests awaiting my decision at my location,
  plus shifts at my location flagged by post-publication drift (§3.7).
- **Admin-tier** — `CONFLICT_OVERRIDDEN` and `CONFLICT_FLAGGED` entries
  at my locations since I last opened `/activity`, badged there. This is
  the direct answer to "I should get notified if there's a conflict":
  the owner sees a count the next time they're in the app rather than
  having to remember to go read the log.

"Since I last looked" needs one field, `User.activitySeenAt`, stamped
when `/activity` is opened; the swap/time-off counts need no new state
at all since `PENDING` already means "awaiting someone." Deliberately
excluded: read/unread per row, notification preferences, digests.

### 3.10 Who sees what

One table, one place, enforced by `lib/authz.ts` — every other section
defers to this one rather than restating its own visibility rule:

| Thing | SUPER_ADMIN | ADMIN (their locations) | MANAGER (their one store) | EMPLOYEE |
|---|---|---|---|---|
| Team schedule (published shifts: who, when) | all | their locations | their store | **their store(s)** — read-only |
| Draft shifts / schedule builder | ✓ | ✓ | ✓ | ✗ |
| Pay rate — view | all | their locations | their store | own only |
| Pay rate — edit | ✓ | ✓ | ✗ | ✗ |
| Labor cost | org + per store | their locations | their store only | own estimated earnings |
| Employees' availability / time-off dates | all | their locations | their store | own only |
| Time-off `reason` text | ✓ | their locations | their store's requests | own only |
| Another person's cross-location shift detail | ✓ | if both in scope | redacted (§3.4) | ✗ |
| Overtime breakdown | full | full | "needs approval" only | own hours |
| Activity log | all + org-level | their locations | ✗ | ✗ |
| `/admin/locations`, `/admin/settings` | ✓ | ✗ | ✗ | ✗ |

Three consequences worth stating outright:

- **Employees see their store's published week.** Own-shifts-only made
  swaps unusable — you can't ask to trade with someone whose schedule you
  can't see. `/shifts` gets a "My shifts / Team week" toggle rendering the
  same grid read-only: names, dates, times, nothing else. No draft shifts
  (an unpublished schedule isn't a promise), no pay, no labor cost, and
  **no availability or time-off markers** — who is working is roster
  information their coworkers need; why someone isn't is not.
- **Time-off `reason` stays free text, written by the employee**, and is
  readable only by that employee and whoever can approve it (their
  store's Manager, admin-tier above). It is never shown on the team
  schedule or in a coworker-visible surface, and the `ActivityLog`
  `metadata` for `TIME_OFF_*` stores dates and status but not the reason
  text — the log is admin-tier-readable and the reason may be medical or
  personal.
- **The swap target list is bounded, not the whole company.** When
  requesting a swap, candidates are active employees holding a
  `UserLocation` grant for that shift's location — anyone else either
  can't legally work it or shouldn't be visible. This is the same
  `getAccessibleLocationIds` logic read from the shift's side.

## 4. Backend plan

**`lib/` modules** (the business logic layer, framework-agnostic where
possible):
- `lib/authz.ts` — `getAccessibleLocationIds(user)`, `assertLocationAccess(user, locationId)`,
  `requireAdminTier(user)`, `requireRole(user, [...])`,
  `getGrantableRoles(viewer)`, `getGrantableLocationIds(viewer)`,
  `canViewPayRate(viewer, targetUserId, locationId)`,
  `getResolutionOptions(viewer, conflict)` (§3.4 — scope-filtered),
  `canApproveTimeOff(viewer, request)`, and the two lockout guards from
  §3.7 (`assertNotLastSuperAdmin`, `assertNotSelfRoleChange`). Small, composable,
  called from every server action — not one bespoke check per route.
- `lib/conflicts.ts` — `checkAssignmentConflict`, `getEmployeePickerConflicts`
  (batch version for annotating a whole picker in one query),
  `getWeekPublishSafetyIssues`, and the resolution-options policy table
  from §3.4.
- `lib/overtime.ts` — weekly-hours calculation (org-wide per employee),
  `OvertimeApproval` gate incl. the `approvedMinutes` staleness rule.
- `lib/activity.ts` — `logActivity()`, called from every mutating action,
  and `getPendingCounts(user)` for the nav badges (§3.9).

**Server actions per feature area** (colocated with their routes):
locations/team (create location incl. its per-store config fields,
invite/edit user + role + location grants + per-location pay rate,
change role, revoke grant, deactivate/reactivate user or location — each
enforcing its §3.7 guard), schedule (create/edit/delete/assign/reassign
shift, resolve conflict, publish week), availability (set recurring +
exceptions), swaps (request, approve/reject, cancel own pending, resolve
conflict), time off (request, approve/deny, cancel own pending),
overtime (approve), settings (edit the single `OrgSettings` row —
SUPER_ADMIN only).

**`middleware.ts`**: unauthenticated → `/login`; `/admin/locations` and
`/admin/settings` are SUPER_ADMIN-only; `/activity` requires admin tier;
role-appropriate default landing page.

## 5. Frontend plan

**Routes:**
- `/login`
- `/dashboard` — role-branching (§3.5), `?location=` filter for admin-tier
- `/schedule` — grid builder, admin-tier + Manager
- `/shifts` — employee's own, plus a read-only "Team week" toggle for
  their store (published only, §3.10)
- `/availability` · `/time-off` (mine + to-decide) · `/swaps` (mine +
  for-me) — employee self-service
- `/team` · `/team/[id]` — unified user management, tier-conditional
- `/admin/locations` — SUPER_ADMIN only; per-store hours and color
- `/admin/settings` — SUPER_ADMIN only; org-wide OT threshold,
  multiplier, currency, `weekStartDay`
- `/activity` — admin-tier only

**Key shared components** (built once, reused everywhere they apply):
- `<ConflictModal>` — the single conflict-resolution UI (§3.4).
- `<EmployeePicker>` — used in the schedule builder; renders availability/
  double-booking badges from `getEmployeePickerConflicts`.
- `<AdminDashboard>` / `<LocationDashboard>` / `<PersonalDashboard>` — role
  dashboard pieces (§3.5), `LocationDashboard` shared between Manager home
  and Admin drill-down.
- `<ActivityFeed>` — used by `/activity`, location-filter aware.
- `<NavBadge>` — the whole of "notify me" (see below): a count rendered
  next to a nav item, nothing more.
- Unified team form — role `<Select>` + location multi-select, both bounded
  by `getGrantableRoles`/`getGrantableLocationIds` for the current viewer;
  each granted location shows its own pay-rate field, *visible* to
  admin-tier, the location's Manager, and the employee themselves
  (`canViewPayRate`), but *editable* only by admin-tier — Managers and
  employees see the number, only Admin/Super Admin can change it.

**Forms & validation:** React Hook Form + Zod schemas shared between client
validation and server action input parsing (one schema per entity, not
duplicated).

**Styling:** Tailwind + shadcn/ui, clean/modern by default; no custom
design system needed.

## 6. Engineering standards & checks

Small project, but not a throwaway one — this section is the one place
both layers' conventions and "done" criteria live, so nothing gets
special-cased per feature the way §2 already rules out for conflict logic.

### 6.1 Structure

- **Backend:** `lib/<module>.ts` per §4 (framework-agnostic business logic)
  + colocated `app/<feature>/actions.ts` server actions calling into it.
  Components never touch Prisma directly — every read/write goes through a
  server action or a `lib/` query function, never an inline `prisma.*` call
  in a page/component.
- **Frontend:** `app/<route>/page.tsx` per §5's route list; shared pieces
  (§5's "Key shared components") live in `components/`, shadcn primitives
  in `components/ui/` untouched from their generated form; anything used
  by exactly one route stays colocated in that route's folder instead of
  moving to `components/` pre-emptively (reuse over invention, §2.6, cuts
  both ways — don't extract until a second caller actually exists).
- **Shared:** one Zod schema per entity in `lib/validation/<entity>.ts`,
  imported by both the React Hook Form on the client and the server action
  doing input parsing. It is the one artifact both layers depend on, so it
  must be a single named file per entity from milestone 1 onward —
  duplicate a schema and client and server drift apart silently.

### 6.2 Coding standards

- TypeScript `strict: true` from `create-next-app` init, kept on — no
  loosening it later to unblock a feature.
- ESLint (`next/core-web-vitals` + `@typescript-eslint`) + Prettier, both
  configured in milestone 1, not bolted on later once violations have
  accumulated.
- Every exported function in `lib/` gets its behavior driven by its
  signature (explicit param/return types) rather than inferred `any` —
  this is the layer §3.4's single conflict primitive and §4's authz
  helpers live in, so its types are load-bearing for correctness, not
  decoration.
- Authorization is never duplicated ad hoc in a route: every server action
  starts by calling the relevant `lib/authz.ts` helper (§4) rather than
  re-checking `session.user.role === ...` inline — one enforcement point
  per rule, matching the "one mechanism" principle in §2.3 applied to
  access control instead of conflicts.

### 6.3 Checks — definition of done, per milestone

Each milestone in §7 is only "done" when all of these pass, not just when
the feature visually works:

1. `tsc --noEmit` — zero type errors.
2. `eslint .` — zero errors (warnings triaged, not silently ignored).
3. `next build` — production build succeeds (catches server/client
   boundary mistakes that dev mode hides).
4. Relevant Vitest unit tests pass — every `lib/` module ships with tests
   in `lib/__tests__/<module>.test.ts` covering its non-trivial branches.
   `authz.ts` is included, not just `conflicts.ts` and `overtime.ts`: a
   scoping bug silently leaks data across locations rather than failing
   loudly, so it needs the same coverage.
5. The milestone's own scenario from §9/Verification walked through
   manually (or via the Playwright smoke test where one exists).

### 6.4 CI

One GitHub Actions workflow, added in milestone 1 alongside scaffolding
(not deferred to polish): install → typecheck → lint → unit tests → build,
on every push. Not deploy-gated, since hosting is deferred (Flag 1) — this
exists purely to make §6.3's checks automatic instead of relying on manual
discipline before every milestone is called done.

### 6.5 Repo layout & server action naming (concrete, bare minimum)

```
shiftmate/
  app/
    (auth)/login/page.tsx
    schedule/{page.tsx, actions.ts}
    shifts/page.tsx
    availability/{page.tsx, actions.ts}
    swaps/{page.tsx, actions.ts}
    time-off/{page.tsx, actions.ts}
    team/{page.tsx, [id]/page.tsx, actions.ts}
    admin/locations/{page.tsx, actions.ts}
    admin/settings/{page.tsx, actions.ts}
    dashboard/page.tsx
    activity/page.tsx
    layout.tsx
  components/
    ui/                  # shadcn-generated, left as-is
    conflict-modal.tsx · employee-picker.tsx · nav-badge.tsx
    admin-dashboard.tsx · location-dashboard.tsx · personal-dashboard.tsx
    activity-feed.tsx
  lib/
    authz.ts · conflicts.ts · overtime.ts · activity.ts
    validation/{shift,user,availability,swap,time-off,location,org-settings}.ts
    __tests__/{conflicts,overtime,authz}.test.ts
  prisma/{schema.prisma, seed.ts}
  middleware.ts
  next.config.ts
  .env.example
```

One `actions.ts` per feature folder (matches §4's "colocated with their
routes"), not a single global `actions.ts` — keeps each file scoped to one
feature's server actions only.

**`next.config.ts`** — two flags, nothing else, and they're what actually
make §6.3's checks 1–2 bite at build time rather than only in CI:
```ts
import type { NextConfig } from "next";
export default {
  typescript: { ignoreBuildErrors: false },
  eslint: { ignoreDuringBuilds: false },
} satisfies NextConfig;
```

**Server action naming:** `verbNoun`, e.g. `createShift`, `assignShift`,
`reassignShift`, `cancelShift`, `resolveShiftConflict`, `publishWeek`,
`requestSwap`, `approveSwap`, `rejectSwap`, `cancelSwap`, `setAvailability`,
`requestTimeOff`, `approveTimeOff`, `denyTimeOff`, `cancelTimeOff`,
`approveOvertime`, `updateOrgSettings`, `createLocation`, `updateLocation`,
`inviteUser`, `updateUserGrant`, `changeUserRole`, `revokeUserGrant`,
`deactivateUser`, `reactivateUser`, `deactivateLocation`. Every action:
- starts with `'use server'` once at the top of the file, not per-function;
- calls its `lib/authz.ts` guard first, before any DB access (§6.2);
- returns one shared shape — `{ status: 'OK', data } | { status: 'CONFLICT', conflict } | { status: 'ERROR', message }`
  — the same three-status contract §3.4 already defines for
  conflict-capable actions, used even for actions that can never conflict
  (they just never return `CONFLICT`), so the frontend has one result
  type to handle everywhere instead of a bespoke shape per action.

## 7. Build milestones (each independently demoable)

1. Scaffolding + auth + data model — Next.js/TS/Tailwind/shadcn init, full
   Prisma schema (4-tier `Role`, `ActivityLog`, `OrgSettings`, per-store
   `Location` config fields, per-location `payRateCents` on `UserLocation`,
   `TimeOffRequest`, `Shift.startMinute`/`endMinute`, `User.activitySeenAt`)
   up front, Auth.js credentials, `middleware.ts`, seed script (1
   SUPER_ADMIN + 2 ADMINs with different location subsets, one default
   `OrgSettings` row), protected shell.
2. Location + unified Team CRUD + org settings — `/admin/locations`
   (SUPER_ADMIN-only, incl. hours/color), `/admin/settings`
   (SUPER_ADMIN-only, OT threshold/multiplier/currency), `/team`
   (tier-conditional form with view-vs-edit pay-rate split via
   `canViewPayRate`), the full `lib/authz.ts` helper set — everything
   downstream depends on this.
3. Schedule builder core — `Shift` model, `/schedule` grid (add/edit/delete
   modal), week/location selector built on `OrgSettings.weekStartDay` (not
   a hardcoded Sunday/Monday), `DRAFT` only, logs `SHIFT_CREATED`/
   `SHIFT_UPDATED`. No conflict logic yet.
4. Availability + time off — `Availability` model, `/availability` CRUD,
   logs `AVAILABILITY_CHANGED`, visual indicator in the schedule grid;
   `/time-off` request → approve/deny/cancel reusing the swap lifecycle,
   logs `TIME_OFF_*`. Both land here because both feed `UNAVAILABLE` in
   milestone 5.
5. Unified conflict mechanism + Publish — `lib/conflicts.ts`,
   `<ConflictModal>` (incl. `getResolutionOptions` scope filtering and the
   redacted out-of-scope case, §3.4), `<EmployeePicker>` badges, Publish
   safety-net sweep plus its re-run on published weeks (§3.7),
   `CONFLICT_OVERRIDDEN`/`SHIFT_REASSIGNED`/`WEEK_PUBLISHED` logging,
   `PUBLISHED` gating what employees see on `/shifts`, including the
   read-only "Team week" toggle (§3.10).
6. Shift swaps — `ShiftSwapRequest`, `/swaps`, employee-to-employee
   approve/reject (no manager gate) reusing `<ConflictModal>` and the same
   primitive, bounded target list (§3.10), auto-cancel when the underlying
   shift moves (§3.7), `SWAP_*` logging.
7. Overtime + role-adaptive dashboard — `lib/overtime.ts` reading its
   threshold/multiplier from `OrgSettings` (not hardcoded), `OvertimeApproval`
   gating publish, `AdminDashboard`/`LocationDashboard`/`PersonalDashboard`
   (incl. Manager's per-employee cost view and the employee's estimated
   earnings figure), `OVERTIME_APPROVED` logging.
8. Activity log UI + nav badges — `/activity` with admin-tier gating and
   the location-filter pattern, surfacing everything logged in milestones
   1–7; `getPendingCounts` + `<NavBadge>` (§3.9), including stamping
   `activitySeenAt` on open.
9. Polish — mobile responsiveness, empty/loading/error states, optional
   drag-and-drop upgrade, accessibility pass.

## 8. Critical files

- `prisma/schema.prisma` — entire data model (`Role`, overloaded
  `UserLocation`, `ActivityLog`).
- `prisma/seed.ts` — seed data covering all 4 roles + the verification
  cases below.
- `lib/authz.ts`, `lib/conflicts.ts`, `lib/overtime.ts`, `lib/activity.ts` —
  the whole business-logic layer described in §4.
- `components/conflict-modal.tsx` — the single conflict-resolution UI.
- `app/schedule/page.tsx` + assign/reassign server actions — the highest
  complexity surface, exercises the picker + modal together.
- `app/dashboard/page.tsx` — role branching into the three dashboard
  pieces.
- `app/team/page.tsx` — tier-conditional role/location-grant form.

## 9. Verification

- **Seed script**: 3+ locations, 1 SUPER_ADMIN, 2 ADMINs (different
  location subsets), a MANAGER per location, ~12–15 employees, realistic
  pay rates, a full upcoming week of shifts.
- **Proactive-path check**: in the picker, confirm flagged candidates show
  a visible badge/reason before selection.
- **Resolve-in-place check**: select a double-booked candidate — confirm
  `<ConflictModal>` appears with Replace/Cancel, and that Replace correctly
  cancels the old shift and logs `CONFLICT_OVERRIDDEN`. Select an
  unavailable candidate — confirm the same modal appears with Assign
  anyway/Cancel, never a hard block. Repeat both cases in swap approval
  (Replace/Reject and Approve anyway/Reject). Repeat once more via the
  Publish safety-net sweep with a drifted-availability case.
- **Role/scoping walkthrough**: log in as each of SUPER_ADMIN, ADMIN
  (subset), MANAGER, EMPLOYEE — confirm dashboard defaults, the Admin
  location-filter drill-down rendering `LocationDashboard`, `/activity`
  reachable only for admin-tier and correctly scoped/consolidated,
  `/admin/locations` SUPER_ADMIN-only.
- **Overtime check**: one seeded employee working across two locations
  scheduled 25h + 25h — confirm Publish for whichever location goes second
  blocks on the org-wide 40h total, that the Manager view shows only
  "needs admin approval" without the other store's shift detail, and that
  an ADMIN/SUPER_ADMIN sees the full breakdown and can approve it.
- **Scope-redaction check**: as a MANAGER, assign an employee who is
  already booked at another store — confirm the picker badge says only
  "booked at another location," the modal offers *only* Cancel (no
  Replace), and that POSTing an out-of-scope `resolution` by hand is
  rejected server-side. Repeat as SUPER_ADMIN and confirm the full detail
  and both options appear.
- **Published-week drift check**: publish a week, then have an employee
  change availability (and separately, have a Manager approve time off)
  over an existing shift — confirm the shift is flagged in the grid, the
  Manager's `<NavBadge>` increments, the change is logged, and nothing
  blocked the employee or the approver at the time.
- **Lifecycle dead-end checks**: reassign a shift that has a `PENDING`
  swap against it and confirm the swap auto-cancels for both parties with
  a note; confirm a `CANCELLED` shift no longer blocks its slot, no
  longer counts toward hours or labor cost, and still appears in the log;
  try to revoke a location grant from an employee with future shifts
  there and confirm it's blocked with the list; try to demote the only
  SUPER_ADMIN and to change your own role, and confirm both are refused;
  approve 41h of overtime, then schedule the same employee to 55h and
  confirm Publish asks for a fresh sign-off showing both numbers.
- **Employee visibility check**: as an EMPLOYEE, open `/shifts` → Team
  week — confirm coworker names/times for their store appear, and that
  draft shifts, pay, labor cost, and any availability or time-off marker
  do not. Confirm a coworker's time-off `reason` is never visible, and
  that the swap target list contains only active employees granted at
  that shift's location.
- **Closed-loop checks**: cancel a pending swap as the requester before the
  target responds; edit a `PUBLISHED` shift directly and confirm it
  re-triggers `<ConflictModal>` when applicable; attempt to deactivate an
  employee/location that still has future shifts and confirm it's blocked
  with the list of shifts to resolve first.
- **Overnight-shift check**: schedule a 22:00–06:00 closing shift, then
  try to assign the same employee a 05:00 start the next calendar day —
  confirm `checkAssignmentConflict` reports `DOUBLE_BOOKING` (the case a
  `"HH:mm"` comparison would have missed), that the grid renders it
  spanning both days, and that it counts as 8h toward weekly overtime.
- **Time-off check**: request time off as an employee, confirm it shows
  `PENDING` with no scheduling effect; approve it as that location's
  Manager, then try to schedule the employee on one of those dates —
  confirm the same `<ConflictModal>` appears with reason "approved time
  off" (not a new modal or a hard block), and that overriding it logs
  `CONFLICT_OVERRIDDEN`. Confirm the requester can cancel a `PENDING`
  request, and that a `DENIED` one leaves them schedulable.
- **Nav badge check**: as an employee with a swap awaiting them, confirm
  the `/swaps` badge shows the count and clears on visit; as the owner,
  override a conflict as a Manager at one store and confirm the
  `/activity` badge increments for the admin-tier user and resets once
  `/activity` is opened.
- **Pay-rate visibility check**: log in as a Manager — confirm they can see
  (not edit) pay rates for employees at their one location, and cannot see
  rates at other locations. Log in as an Employee — confirm they see only
  their own rate and an estimated-earnings figure on `PersonalDashboard`,
  never a coworker's rate.
- **Per-store config check**: as SUPER_ADMIN, set two seeded locations to
  different hours/`colorHex` at `/admin/locations`, confirm the
  color shows up in the schedule grid and consolidated dashboard; adjust the
  overtime threshold at `/admin/settings` and confirm `lib/overtime.ts`
  picks up the new value on the next Publish check rather than the old
  hardcoded 40h; change `weekStartDay` and confirm the schedule grid,
  Publish, OT calc, and dashboard rollup all shift their week boundary
  together, not just one of them.
- **Automated tests**: Vitest unit tests for `checkAssignmentConflict` and
  the OT-hours calculator, plus one Playwright smoke test (login → trigger
  a double-booking → resolve via `<ConflictModal>` → publish). Kept
  intentionally light beyond that.

## 10. Execution plan

§7 says *what* each milestone contains. This section says *how* each one
gets built, proven and landed: build locally, verify locally, one PR per
milestone, merge, move on. Nothing is deployed until everything is built
and tested (§10.5).

### 10.1 Verified environment (checked on mercury, 2026-09-23)

| Tool | Status |
|---|---|
| Node | v22.23.2 — fine for Next 14/15 |
| npm / pnpm | 10.9.8 / 12.5.1 (pnpm via corepack). **Use pnpm**, commit `pnpm-lock.yaml` |
| PostgreSQL | 16.15, running locally on `:5432` — no Docker needed |
| gh CLI | 2.45.0, authenticated as **`UsamaIqbal0304`**, git protocol ssh, scopes `repo` + `workflow` (enough to push Actions) |
| Docker | present, unused |

Two things to settle before the first commit:

1. **`git config user.name` and `user.email` are unset on this machine.**
   Commits will fail or be attributed wrongly. Set them *in the repo*
   (`git config user.name … && git config user.email …`) as part of
   §10.2, rather than globally.
2. **A second, broken gh account (`usama-git-io`) is configured as the
   default and its token is invalid.** `UsamaIqbal0304` is the active
   one, so `gh` works — but if a command ever errors with a bad token,
   it picked the wrong account; `gh auth switch --user UsamaIqbal0304`
   fixes it. Worth confirming the repo lands on the intended account.

### 10.2 PR 0 — repository and local environment

Not a feature PR; this is the foundation everything else branches from.

```bash
mkdir ~/shiftmate && cd ~/shiftmate
git init -b main
git config user.name  "Usama Iqbal"
git config user.email "<the email you want on these commits>"

createdb shiftmate_dev            # local Postgres 16 is already running
createdb shiftmate_test           # separate DB so tests never touch dev data
```

`.env.local` (git-ignored) and a committed `.env.example` with the same
keys and dummy values:
```
DATABASE_URL="postgresql://localhost:5432/shiftmate_dev"
AUTH_SECRET="<openssl rand -base64 32>"
AUTH_URL="http://localhost:3000"
```

Then scaffold, and push:
```bash
pnpm dlx create-next-app@latest . --ts --tailwind --app --eslint --src-dir=false
gh repo create shiftmate --private --source=. --remote=origin
git add -A && git commit -m "chore: scaffold Next.js app" && git push -u origin main
```

**Branch protection on `main`** — do this now, while there is nothing to
lose: require the CI check to pass before merge. This is what makes the
per-PR loop below meaningful rather than ceremonial.
```bash
gh api -X PUT repos/UsamaIqbal0304/shiftmate/branches/main/protection \
  -F required_status_checks[strict]=true \
  -F 'required_status_checks[contexts][]=ci' \
  -F enforce_admins=false \
  -F required_pull_request_reviews=null \
  -F restrictions=null
```
Self-review is the review step here — there is no second engineer, so
`required_pull_request_reviews` stays null rather than blocking you on an
approval you'd have to grant yourself.

Also lands in PR 0: Prettier + ESLint config, the `next.config.ts` from
§6.5, Vitest + Playwright installed and wired to `pnpm test` /
`pnpm test:e2e`, and the §6.4 CI workflow. **CI must be green on PR 0**,
even with one trivial test — a pipeline added later is a pipeline that
never runs.

`.github/workflows/ci.yml` — job name `ci` to match the protection rule
above; it needs its own Postgres service since GitHub runners have none:
```yaml
name: ci
on: [push, pull_request]
jobs:
  ci:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_PASSWORD: postgres }
        options: >-
          --health-cmd pg_isready --health-interval 10s
          --health-timeout 5s --health-retries 5
        ports: ['5432:5432']
    env:
      DATABASE_URL: postgresql://postgres:postgres@localhost:5432/shiftmate_test
      AUTH_SECRET: ci-secret-not-real
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm prisma migrate deploy
      - run: pnpm exec tsc --noEmit
      - run: pnpm lint
      - run: pnpm test
      - run: pnpm build
```

### 10.3 The per-PR loop

One milestone from §7 = one branch = one PR. Never two milestones in a
branch: the point of the sequence is that each merge leaves `main` in a
demoable state.

```bash
git switch -c feat/03-schedule-builder     # feat/<NN>-<slug>, NN = milestone
# … build …
pnpm exec tsc --noEmit && pnpm lint && pnpm test && pnpm build   # §6.3 gate
pnpm dev            # walk the milestone's acceptance list by hand
git push -u origin feat/03-schedule-builder
gh pr create --fill-first        # then edit the body to the template below
gh pr checks --watch             # CI must be green
gh pr merge --squash --delete-branch
git switch main && git pull
```

PR body template — short, but the acceptance list is not optional, since
it is the record that the milestone was actually exercised:
```markdown
## What
<one paragraph: which §7 milestone, what a user can now do>

## Plan refs
§3.4 (conflict primitive) · §3.10 (visibility) — sections this implements

## Checks
- [ ] tsc --noEmit · lint · test · build all pass locally and in CI

## Acceptance (walked manually)
- [ ] <the §9 verification bullets that belong to this milestone>

## Notes / deviations from plan
<anything built differently, and why — or "none">
```

A deviation noted in a PR body is a signal the plan needs an edit. Update
the plan file in the same PR rather than letting the two drift.

### 10.4 PR sequence, with what proves each one

PRs 1–9 are §7's milestones 1–9 unchanged. What follows is the *test and
demo* obligation for each — the §9 bullets are the master list, sliced
here so each PR carries only its own.

| PR | Milestone | Automated tests added | Walked manually before merge |
|---|---|---|---|
| 1 | Scaffolding, auth, full schema, seed | — (schema compiles; seed runs) | Log in as each of the 4 seeded roles; `middleware.ts` redirects each to its landing page; unauthenticated → `/login` |
| 2 | Locations, team, org settings | `authz.test.ts`: `getAccessibleLocationIds`, `canViewPayRate`, `getGrantableRoles` | §9 role/scoping walkthrough; pay-rate visibility check (Manager sees-not-edits; employee sees own only); `/admin/*` SUPER_ADMIN-only |
| 3 | Schedule builder, DRAFT only | `weekStartDay` boundary test | Create/edit/delete shifts; change `weekStartDay` and confirm the grid boundary moves |
| 4 | Availability + time off | `conflicts.test.ts` begins: availability + approved-leave → `UNAVAILABLE` | Request → approve → deny → cancel a time-off request; confirm `reason` invisible to coworkers (§3.10) |
| 5 | **Conflict mechanism + Publish** | `conflicts.test.ts` in full: double-booking, cross-midnight overlap, `getResolutionOptions` scope filtering, `CANCELLED` excluded | §9 proactive-path, resolve-in-place, scope-redaction and overnight-shift checks. **The riskiest PR — budget for it.** |
| 6 | Shift swaps | swap auto-cancel when shift moves; target list bounded to granted employees | Request → approve → reject → cancel; approve one that conflicts and resolve via the modal |
| 7 | Overtime + dashboards | `overtime.test.ts`: cross-location 25h+25h, threshold from `OrgSettings`, `approvedMinutes` staleness | §9 overtime check incl. Manager-sees-no-detail redaction; all three dashboards |
| 8 | Activity log + nav badges | — | §9 published-week drift check; badge increments and clears |
| 9 | Polish: mobile, empty/error states, a11y | Playwright smoke test (login → double-booking → resolve → publish) | Every page at 375px wide; keyboard-only pass over the schedule builder |

Two ordering notes worth respecting: PR 4 must precede PR 5 because
approved time off is an input to the conflict primitive, and PR 5 must
precede PR 6 because swap approval reuses `<ConflictModal>` wholesale.

### 10.5 Local verification before any hosting

When PR 9 merges, run the whole of §9 against a fresh database — not the
one that has been accumulating test data for weeks:

```bash
dropdb shiftmate_dev && createdb shiftmate_dev
pnpm prisma migrate reset --force    # runs prisma/seed.ts
pnpm build && pnpm start             # production build, not pnpm dev
```

Testing against `pnpm dev` alone hides server/client boundary errors and
real-world performance, so the final pass runs the production build.
Walk every §9 bullet, with a browser, as each of the four roles. Fix
anything found as its own small PR, with the same loop as §10.3.

### 10.6 Deployment — only after §10.5 passes

Flag 1 in §3.1 is still open, so the hosting decision belongs here rather
than earlier. The default recommendation stands: **Vercel + Neon**, since
the app is already Next.js and neither needs ops work. A VPS is cheaper
and more work; either way the code doesn't change.

Deployment lands as its own final PR (`chore/deploy`):
- Neon project created; `DATABASE_URL` set as a Vercel env var.
- `pnpm prisma migrate deploy` against the production database, then the
  seed script run **once** to create the first SUPER_ADMIN (§3.7's
  bootstrapping rule — there is no signup, so this is the only way in).
- Vercel project linked to the GitHub repo; preview deploys on PRs,
  production on `main`.
- Flag 2 closed by decision: admin-set passwords only, no email reset.
  Revisit if that becomes painful in real use.
- Smoke-test the production URL as all four roles before handing it to
  anyone at the stores.

Nothing here is destructive to the local setup, but the production
database is the one place where a `migrate reset` would be catastrophic —
never run it against `DATABASE_URL` once it points at Neon.

## 11. Side plan — Rails + React + GraphQL + Postgres + JWT

An alternative to §3.1's stack for the same product. Sections 1–3 are
deliberately framework-agnostic, so this is a substitution at the
implementation layer, not a redesign.

### 11.1 What carries over untouched

Everything that makes this project non-trivial survives the stack change,
because none of it is Next.js-specific:

- The whole data model (§3.3) — same tables, same columns, same
  `startMinute`/`endMinute` decision, same enums.
- The conflict mechanism (§3.4) — one primitive, one policy table, one
  modal. `checkAssignmentConflict` becomes a Ruby service object with an
  identical signature and identical semantics.
- Every lifecycle rule in §3.7 and the entire visibility matrix in §3.10.
- The guiding principles (§2) and the verification list (§9).

If anything here needs rewriting for the new stack, that is a sign the
architecture leaked framework assumptions — it shouldn't.

### 11.2 What replaces what

| §3.1 stack | This stack |
|---|---|
| Next.js App Router (UI + API in one) | Rails 7.1 API-only backend + separate React SPA (Vite) |
| Server actions | GraphQL mutations (`graphql-ruby`) |
| Prisma schema + migrate | ActiveRecord migrations; `schema.rb` is the source of truth |
| Auth.js credentials + session cookie | JWT — `bcrypt` for password hashing, `jwt` gem for tokens |
| `lib/authz.ts` | Pundit policies + a `CurrentUser` context object |
| `lib/conflicts.ts`, `overtime.ts`, `activity.ts` | `app/services/conflicts.rb`, `overtime.rb`, `activity.rb` — plain Ruby objects, no Rails magic |
| Zod schema shared client+server | **Nothing shares.** See §11.3 |
| React Hook Form + Zod | React Hook Form + Yup/Zod on the client only |
| Vitest + Playwright | RSpec + factory_bot (backend), Vitest + RTL (frontend), Playwright (e2e, unchanged) |
| One `pnpm build` | Two build pipelines, two dependency managers, two deploy targets |

Tailwind + shadcn/ui carry over to the SPA unchanged.

### 11.3 Where this stack genuinely changes the design

These are the parts that need a decision rather than a translation.

**The shared validation schema is gone.** §6.1 made one Zod schema per
entity the single artifact both layers depend on. Ruby and TypeScript
can't share it. The replacement is the **GraphQL schema as the contract**,
with `graphql-codegen` generating TypeScript types for the frontend from
the Rails schema — so types stay in sync automatically, but *validation
rules* (min/max, format, required-if) must be written twice: once in Rails
model validations, once in the client form. Accept the duplication and
keep the Rails side authoritative; never trust the client copy.

**The `CONFLICT` result contract becomes a GraphQL payload type.** §3.4's
three-status return maps cleanly onto the standard mutation-payload
pattern — not onto GraphQL `errors`, which are for exceptional failures,
whereas a conflict is an expected outcome:
```graphql
type AssignShiftPayload {
  shift: Shift
  conflict: Conflict      # null unless status == CONFLICT
  errors: [UserError!]!
}
```
The client checks `conflict` first, exactly as it checks `status` today.

**§3.10 must be enforced field-by-field, and that is the main new risk.**
REST returns what an endpoint chose to return; GraphQL lets a client ask
for any field it can reach. `payRateCents` and `TimeOffRequest.reason` are
both reachable from several directions — via `Location.employees`, via
`Shift.user`, via `User.timeOffRequests`. So authorization cannot live at
the query root: each sensitive field needs its own guard, and Pundit
policies must be invoked in the field resolvers rather than once per
request. Budget real time for this; it is the one place this stack is
harder to get right than §3.1's, where a server action simply doesn't
return what it doesn't select.

**N+1 queries will bite the schedule grid.** A week × 15 employees ×
nested location and user lookups is the textbook GraphQL N+1. Use
`graphql-batch` (or `dataloader`) from the first resolver, not as a later
optimization — and add `bullet` in development to fail loudly on N+1s.
This is the JACE-constrained-resources instinct applied to a different
problem: it's a small app, but the grid is the one genuinely hot query.

**JWT needs three decisions §3.1 didn't:**
- *Storage.* Put the token in an **httpOnly, SameSite=Lax cookie**, not
  `localStorage` — localStorage is readable by any injected script, which
  turns one XSS into full account takeover. Cookies mean CSRF protection
  must be on, which Rails gives you.
- *Lifetime and refresh.* Short-lived access token (~15 min) plus a
  refresh token, or a single longer-lived token (~24h) with no refresh.
  For an app whose users log in at the start of a shift, the simpler
  second option is defensible; pick one deliberately.
- *Revocation.* A plain JWT stays valid until it expires — which
  contradicts §3.7's deactivation rules, since a deactivated employee
  would keep working access until their token lapses. Fix with a
  `jti` denylist table checked on each request, or a `token_version`
  column on `User` bumped on deactivate/password-change. The second is
  simpler and enough here.

**Two runtimes means CORS, two dev servers, and a split deploy.** Rails on
`:3000`, Vite on `:5173`, CORS configured via `rack-cors` for local dev
and the production origin. Deploy is two targets (e.g. Fly.io/Render for
Rails + Neon, Vercel/Netlify for the SPA) rather than one.

### 11.4 Repo layout

One repo, two apps — keeps the §10.3 one-PR-per-milestone loop intact,
since most milestones touch both sides:

```
shiftmate/
  api/                      # Rails 7.1 --api
    app/
      graphql/
        types/              # shift_type.rb, user_type.rb, …
        mutations/          # assign_shift.rb, approve_swap.rb, …
        shiftmate_schema.rb
      models/               # ActiveRecord: user, location, shift, …
      policies/             # Pundit: shift_policy.rb, user_policy.rb, …
      services/             # conflicts.rb, overtime.rb, activity.rb
    db/{migrate/, seeds.rb, schema.rb}
    spec/{models/, services/, graphql/, policies/}
  web/                      # Vite + React + TS
    src/
      components/           # conflict-modal.tsx, employee-picker.tsx, …
      pages/                # schedule.tsx, shifts.tsx, team.tsx, …
      graphql/              # .graphql documents + codegen output
  .github/workflows/ci.yml  # both suites in one workflow
```

### 11.5 Milestone mapping

The §7 sequence and its ordering constraints hold — PR 4 before PR 5,
PR 5 before PR 6, for the same reasons. Differences worth planning for:

- **PR 1 grows.** It now covers Rails API scaffold, the React SPA
  scaffold, GraphQL schema wiring, codegen, CORS, *and* the whole JWT
  flow including `token_version`. Expect it to be roughly twice the §10.4
  PR 1. Consider splitting it: `1a` backend + auth, `1b` SPA + login.
- **PR 2 carries the field-level authorization work** from §11.3, not
  just the CRUD — Pundit policies wired into resolvers, with specs
  asserting that a Manager querying another store's `payRateCents` gets
  null rather than a value.
- **PR 5 is still the riskiest**, and slightly riskier here: the conflict
  service is straightforward Ruby, but the mutation payload plumbing and
  the modal's typed client are new surface.
- **PR 9 adds an N+1 audit** with `bullet` enabled, over the schedule grid
  and the consolidated dashboard.

Testing split: service objects get RSpec unit specs (the §9 conflict and
overtime cases translate directly), GraphQL gets request specs asserting
both data *and* redaction, the SPA gets Vitest + RTL for the modal and
picker, and Playwright covers the same single smoke path end to end.

### 11.6 Prerequisites on mercury

Unlike §10.1, this stack is **not installable-and-ready on this machine**:

- **No Ruby, no Rails, no bundler, and no version manager** (`rbenv`,
  `rvm`, `asdf`, `mise` are all absent). Install a version manager first,
  then Ruby 3.3.x — do not use a system-wide Ruby.
- `libpq` headers *are* present (`pg_config` resolves), so the `pg` gem
  will build without extra packages.
- PostgreSQL 16.15, Node 22, pnpm and gh are all shared with §10.1 and
  need nothing new.

So this path starts with a toolchain install that §3.1's path doesn't.

### 11.7 Honest comparison

**Pick Rails + React + GraphQL if** the goal includes practising that
stack, or if a mobile client or third-party integration is coming and a
typed GraphQL API is genuinely wanted.

**Otherwise §3.1 remains the recommendation for this product.** For a
3-store scheduling app the second stack is strictly more moving parts —
two runtimes, two dependency managers, two test suites, two deploys, CORS,
JWT lifecycle management, N+1 defense, field-level authorization, and the
loss of the shared validation schema — in exchange for an API surface
nothing currently consumes. §2's "reuse over invention" and the brief's
"doesn't need to be fancy, just needs to work well" both point the same
way. This section exists so the choice is informed, not to relitigate it.
