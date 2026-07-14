# Kairos Architecture Notes

Org-level architecture documentation that doesn't belong to any single repo.
This file is the first entry: a preventive audit of a coupling risk between
`.swift` (the iOS app) and `.website` (the web app + Supabase backend) that
does not exist yet, but will exist the moment someone wires networking into
the iOS client.

---

## 1. Schema-Contract Audit: Swift/Postgres Drift Risk

**Status:** Preventive — no bug exists today. This documents a risk to design
around *before* the sync layer is built, not a fix for something broken.

### 1.1 Current state (as of this writing)

`.swift` is a fully local, offline SwiftUI app. It has **zero networking
code and zero backend coupling** (no `URLSession`, no API client, no
Supabase SDK usage anywhere in the app target). All persistence goes
through `UserDefaults`, and all domain state lives in plain Swift
structs/enums defined in the app target:

| Swift type | File | Role |
|---|---|---|
| `SchedulePattern` | `FireScheduleEngine.swift` | A named, repeating shift cycle (`name`, `description`, `cycle: [ShiftSegment]`) — the schedule pattern concept |
| `ShiftSegment` | `FireScheduleEngine.swift` | One on/off block within a cycle (`isOn: Bool`, `hours: Int`) — the shift segment/cycle concept |
| `OverrideType` | `FireScheduleEngine.swift` | Enum: `.vacation` ("off"), `.extraShift` ("on"), `.onCall` ("on_call") — the override concept |
| `JobRole` | `FireScheduleState.swift.swift` | Enum of job-role categories (firefighter, medical, law enforcement, industrial, transportation, hospitality) — the job role concept |
| `SharedSchedule` | `FireScheduleState.swift.swift` | A snapshot of another user's schedule (`userName`, `selectedPatternIndex`, `startDate`, `overrides`) held locally after being "claimed" — the shared schedule concept |
| `MockAccessCodeServer` | `FireScheduleState.swift.swift` | `UserDefaults`-backed stand-in for a real access-code service; generates/claims 6-character codes — the access code concept, currently mocked entirely client-side |
| `RecoveryData` / `RecoveryManager` | `RecoveryManager.swift` | AES-GCM–encrypted, Base64-encoded blob of the user's full local state, used as the app's backup/restore mechanism — the recovery key concept |

None of these types are generated from, validated against, or even aware of
a schema. They are hand-written and evolve however the app's SwiftUI views
need them to.

Concurrently, `.website` is standing up a Supabase Postgres schema (see
`supabase/migrations/0001_init.sql`, the schema-foundation migration, plus
`0002_rls.sql` and `0003_realtime.sql` layered on top of it in that repo)
covering the same domain from the server side: `users`, `schedule_patterns`,
`shift_segments`, `overrides`, `shared_schedules`, `access_codes`. Same
nouns, same domain — two independent, hand-maintained representations.

`0001_init.sql`'s own header comments confirm the mirroring is intentional
but informal: "Where a table mirrors a concept from the existing iOS app
(`FireScheduleEngine.swift` / `RecoveryManager.swift`), a comment notes the
Swift-side name. The Swift sources are referenced for naming consistency
only — they are not read in depth here and are not modified by this
migration." That is precisely the gap this section is about: naming
consistency at authoring time is not the same as an enforced contract, and
nothing stops the two sides from diverging on the very next migration —
see 1.2 for a case where they already have.

**Today these two representations never touch.** `.swift` has no way to
call `.website`'s API or Supabase project, so today's version skew is
inert — it's a latent risk, not an active one.

### 1.2 The risk: invisible coupling once sync lands

The risk activates the moment `.swift` gains a networking layer that reads
from or writes to the `.website`/Supabase backend (e.g. to support the same
shared-schedule / access-code flow the web app has via
`supabase/migrations/0003_realtime.sql`'s realtime channels, described in
`.website/lib/supabase/realtime-channels.md`).

At that point, `.swift`'s structs and Postgres's tables become **two
descriptions of the same data that are never checked against each other**:

- Swift and SQL are compiled/executed by entirely separate toolchains. There
  is no step in either repo's build where a Swift struct's shape is
  compared to a Postgres table's shape.
- A migration in `.website` (e.g. renaming a column, changing a type,
  adding a `NOT NULL` constraint, splitting `shift_segments.hours` into
  `starts_at`/`ends_at` timestamps) can land, pass `.website`'s own
  TypeScript type checks, and merge — while `.swift`'s `ShiftSegment` or
  `OverrideType` silently continues encoding/decoding the *old* shape.
- The failure mode is not a compile error on either side. It's a runtime
  failure: a `JSONDecoder` throw, a Postgres constraint violation, a
  `PostgrestError`, or — worse — a *successful* write of a value the new
  schema no longer expects, silently corrupting data or dropping fields
  with no error at all.
- This is classic "invisible coupling": two systems that behave as if they
  share a contract, without any artifact that *is* that contract. Nothing
  fails until a real device, with a real user's real recovery-key-derived
  data, hits the mismatch in production.

Concretely, the vocabulary most exposed to this risk, mapped both ways:

| Domain concept | Swift representation | Postgres representation (`0001_init.sql` unless noted) |
|---|---|---|
| Schedule pattern | `SchedulePattern` struct | `schedule_patterns` table (`id`, `owner_user_id`, `name`, `description`, `job_role`) |
| Shift segment/cycle | `ShiftSegment` struct, `[ShiftSegment]` cycle array | `shift_segments` table: `is_on boolean`, `hours integer`, `sequence_index`, FK'd to `schedule_patterns` via `schedule_pattern_id` |
| Override (vacation / extra_shift / on_call) | `OverrideType` enum (`"off"` / `"on"` / `"on_call"` raw values — note these do **not** textually match the Postgres `override_type` enum values `vacation` / `extra_shift` / `on_call`) | `overrides` table, `override_type` column, a Postgres enum constrained to `vacation \| extra_shift \| on_call` |
| Job role | `JobRole` enum (6 cases: firefighter, medical, lawEnforcement, industrial, transportation, hospitality) | `job_role` Postgres enum with the **same 6 raw values**, used on both `users.job_role` and `schedule_patterns.job_role` — this one is already in sync; called out in 1.4 as the pattern to replicate elsewhere, not as a gap |
| Shared schedule | `SharedSchedule` struct | `shared_schedules` table (`owner_user_id`, `schedule_pattern_id`, `cycle_start_date`, `is_active`) |
| Access code | `MockAccessCodeServer.CodeEntry` (mocked; has an `isUsed: Bool` field) | `access_codes` table, single-use, gated by `is_used boolean` (see below) plus `expires_at` |
| Recovery key | `RecoveryData` / `RecoveryManager` (local AES-GCM blob, no server involvement) | `users.recovery_key_hash` — a one-way hash only, per `0001_init.sql`'s explicit design note that no `recovery_data` table exists server-side; the decryptable blob never leaves the device |

Two concrete drift examples worth naming, because they illustrate the risk
in data that already exists on disk today — before any sync code has been
written:

- **`OverrideType` raw-value mismatch.** Swift's `OverrideType` enum uses
  raw values `"off"` / `"on"` / `"on_call"` (`FireScheduleEngine.swift`),
  while the Postgres `override_type` enum (and the RLS/realtime docs built
  on top of it) uses `vacation` / `extra_shift` / `on_call`. This is
  exactly the kind of drift this section warns about, and it costs nothing
  to fix now (rename the Swift raw values, or plan an explicit mapping
  layer) versus a production incident to find later if it ships silently
  into a sync payload.
- **`access_codes` "used" column mismatch *between the migrations
  themselves*.** `0001_init.sql` (the migration that actually creates the
  table) defines `is_used boolean not null default false`. But
  `0002_rls.sql`, written against an *assumed* shape of 0001 rather than
  the real file, documents and writes RLS policies against a `used_at
  timestamptz` column instead (see its header: "a `used_at timestamptz`
  column (assumed from 0001)..." and its `access_codes_mark_used` policy,
  which reads/writes `used_at`). One of these is wrong relative to the
  other — this is the exact failure mode described above, except it has
  already happened *within `.website` itself*, between two SQL migrations
  written by different people at different times, without needing a Swift
  client in the picture at all. It is flagged here as evidence that "the
  same repo, same language, same PR reviewers" is not sufficient protection
  either — only a checked, generated contract is sized appropriately
  against this failure mode. Whoever reconciles 0001 vs. 0002 should treat
  that reconciliation as validation that the mitigation in 1.4 would have
  caught it automatically.

### 1.3 Why this is worth solving before sync work starts, not after

- **No compiler catches this.** Xcode's build has no visibility into
  `.website`'s Postgres schema; `tsc`/`next build` has no visibility into
  `.swift`'s structs. Each repo's CI is green in complete ignorance of the
  other — and, per the `is_used`/`used_at` example above, two migrations in
  the *same* repo can already disagree without either one's review process
  catching it.
- **The two repos are on independent release cadences.** A Supabase
  migration can ship to production the moment it's merged; the iOS app
  ships on App Store review timelines. A schema change and an app update
  are never atomic — some population of installed iOS clients will always
  be running against a schema version newer or older than the one they
  were built against. Without a versioned contract, "newer or older" is
  undetectable at runtime.
- **Kairos's privacy model raises the stakes.** Because identity is a
  recovery key rather than an account (per `0001_init.sql`'s design and
  `0002_rls.sql`'s "Identity Strategy" section), there is no admin/support
  path to "look up the user's account and fix their data" the way an
  email-based system might. A silent decode failure or malformed write
  on-device is much harder to triage after the fact than in an
  email/password system with a backing admin console.
- **It's cheap now, expensive later.** Today there are exactly two "client"
  representations of the schema to reconcile: `.swift`'s structs and
  `.website`'s TypeScript/SQL. Once a sync layer exists and ships to real
  users, reconciling drift means shipping a migration, a Swift app update,
  *and* a data-backfill or compatibility shim simultaneously — three moving
  parts instead of a documentation/tooling change.

### 1.4 Mitigation: a versioned schema-contract artifact

**Proposal:** introduce a single generated file, `schema-contract.json`,
that is the source of truth both `.swift` and `.website` code-generate or
validate against. Concretely:

1. **Location and ownership.** Generate `schema-contract.json` in
   `.website` (it is the repo that owns the Supabase migrations, i.e. the
   actual source of truth for table/column shape), at
   `.website/supabase/schema-contract.json`, committed to the repo — not
   generated only at build time and discarded. This makes it diffable in
   PRs the same way a migration file is, and would have surfaced the
   `is_used`/`used_at` mismatch (1.2) the moment `0002_rls.sql` was
   proposed, instead of leaving it for a manual reconciliation later.

2. **Generation.** Derive it mechanically from the applied migrations
   rather than hand-writing it (hand-writing it defeats the purpose — a
   hand-maintained contract file can drift from the migrations exactly
   like the Swift structs can, or exactly like `0002_rls.sql` drifted from
   `0001_init.sql`). Two concrete options, either acceptable:
   - Run `supabase gen types typescript` (or the equivalent
     `--lang json-schema`-style introspection) against a local/shadow
     Postgres instance with all migrations applied, then transform that
     output into a flat, language-agnostic shape (table name, column name,
     Postgres type, nullability, enum values, FK target). This avoids
     hand-maintaining a second description of the schema.
   - Alternatively, a lightweight OpenAPI-style document hand-authored
     *once* per migration as part of the migration PR itself (i.e. every
     PR that touches `supabase/migrations/*.sql` must also update
     `schema-contract.json` in the same commit) — acceptable if
     introspection tooling isn't set up yet, but strictly worse long-term
     than (a) because it re-introduces a manual sync point. Prefer (a)
     once feasible; use (b) only as a bootstrap.

3. **Shape of the contract file.** Minimal, flat, and easy for both a
   Swift code generator and a TypeScript type generator to consume — not
   a full OpenAPI spec unless one is already in use elsewhere. Example
   shape, modeled on the real `access_codes` table in `0001_init.sql`:

   ```json
   {
     "contractVersion": 3,
     "generatedFromMigration": "0001_init.sql",
     "tables": {
       "access_codes": {
         "columns": {
           "id": { "type": "uuid", "nullable": false, "primaryKey": true },
           "code": { "type": "text", "nullable": false },
           "shared_schedule_id": { "type": "uuid", "nullable": false, "references": "shared_schedules.id" },
           "is_used": { "type": "boolean", "nullable": false, "default": false },
           "expires_at": { "type": "timestamptz", "nullable": false }
         }
       }
     }
   }
   ```

   `contractVersion` is a plain monotonically increasing integer, bumped
   any time the generated content changes — this is what both sides pin
   against (see 1.4.5) and what a stale-contract CI check (see 1.5)
   compares.

4. **Consumption:**
   - **TypeScript side (`.website`):** generate/refresh
     `lib/supabase/database.types.ts` (or equivalent) from the same
     migration state that produced `schema-contract.json`, so both files
     are provably generated from one another's source of truth in the
     same CI run.
   - **Swift side (`.swift`):** once networking work begins, add a build
     step (Swift Package Manager plugin or a pre-build script phase in the
     Xcode project) that reads `schema-contract.json` (vendored into
     `.swift` via a lightweight sync script, or fetched from `.website` at
     build time from a pinned commit/tag) and either (a) code-generates
     `Codable` structs from it, replacing the hand-written
     `SchedulePattern` / `ShiftSegment` / `OverrideType` / `RecoveryData`
     definitions with generated ones, or (b) — if full codegen is too
     heavy a lift for the first pass — generates a small validation test
     that fails the Swift build if the hand-written structs' `CodingKeys`
     and enum raw values no longer match the contract. Option (a) is the
     stronger long-term fix; option (b) is an acceptable incremental step
     that still turns drift into a compile-time/test-time failure instead
     of a runtime one.
   - Either way, **no hand-maintained Swift struct should describe
     server-synced data without something checking it against
     `schema-contract.json`.** Purely local-only concepts that never
     round-trip through Supabase (if any ever exist) are exempt by
     definition.

5. **Pinning.** `.swift` should pin to a specific `contractVersion` (or a
   specific `.website` commit/tag that produced a given
   `schema-contract.json`), not "latest," the same way any client pins a
   dependency version. Bumping the pin is a deliberate, reviewed change in
   `.swift`, not something that happens implicitly.

### 1.5 Process: CI check for contract staleness

Add a CI job in `.website` (the repo that owns migrations) that fails a PR
if `schema-contract.json` is stale relative to the migrations it's
supposed to describe:

1. On every PR touching `supabase/migrations/**`, re-run the generation
   step from 1.4.2 against a fresh/shadow database with all migrations
   (including the PR's new one) applied.
2. Diff the freshly generated contract against the committed
   `schema-contract.json`.
3. If they differ, fail the check with a message pointing at "regenerate
   `schema-contract.json` and commit it as part of this PR" — the same
   pattern as a lockfile-drift check (e.g. `package-lock.json` vs.
   `package.json`), which this repo's contributors will already recognize
   from normal `npm`/`next` workflows.
4. Do **not** attempt to auto-fix and auto-commit the regenerated file in
   CI — require a human (or the PR author's agent) to regenerate locally
   and commit it, so the diff is visible in code review rather than
   silently appended by a bot.
5. This check would have caught the `is_used`/`used_at` mismatch between
   `0001_init.sql` and `0002_rls.sql` (1.2) at PR time for `0002_rls.sql`:
   regenerating the contract from migrations-so-far would either fail to
   apply `0002_rls.sql`'s policies (wrong column name against the real
   table) or produce a contract that obviously didn't match what
   `0002_rls.sql`'s own comments assumed, either of which is a much
   cheaper place to catch it than a runtime `PostgrestError`.

A second, lower-priority check once `.swift` has adopted the contract
(post-networking-layer): a scheduled or manually-triggered cross-repo check
that compares `.swift`'s pinned `contractVersion` against `.website`'s
current `contractVersion` and opens/updates a tracking issue (not a hard
CI failure, since `.swift`'s release cadence is independent — see 1.3) when
they diverge, so the drift is visible before it becomes a shipped-app
problem rather than after.

### 1.6 Summary for whoever builds the sync layer

If you are the developer (or agent) implementing `.swift`'s networking
layer:

- Do not hand-write `Codable` structs against your best guess of
  `.website`'s schema. Generate or validate them against
  `schema-contract.json` (1.4).
- Check `.website/supabase/migrations/` for the latest applied migration
  number and confirm `schema-contract.json`'s `generatedFromMigration`
  matches it before you start — if `.website`'s CI check (1.5) is in place,
  a matching, committed contract file is guaranteed to exist for every
  merged migration.
- Resolve the `is_used` vs. `used_at` disagreement between
  `0001_init.sql` and `0002_rls.sql` (1.2) — confirm which column actually
  exists in the live schema and update whichever side (migration or RLS
  policy) is wrong — before treating either name as ground truth.
- Fix the `OverrideType` raw-value mismatch called out in 1.2
  (`"off"`/`"on"`/`"on_call"` vs. `vacation`/`extra_shift`/`on_call`)
  as part of this work if it hasn't been fixed already — either align the
  Swift raw values to the Postgres/TypeScript strings directly, or add an
  explicit, tested mapping function at the serialization boundary. Do not
  let it round-trip through JSON silently.
- `JobRole` is a good-news case, not a gap: `0001_init.sql`'s `job_role`
  Postgres enum already matches Swift's six `JobRole` raw values exactly
  (`firefighter`, `medical`, `lawEnforcement`, `industrial`,
  `transportation`, `hospitality`), on both `users.job_role` and
  `schedule_patterns.job_role`. Treat this as the template for what "in
  sync" looks like, and make sure the contract-generation tooling in 1.4
  captures Postgres enum values (not just column names/types) so this
  stays true automatically rather than by coincidence.
