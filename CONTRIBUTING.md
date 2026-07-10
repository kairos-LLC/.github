# Contributing to Kairos

Kairos is proprietary, closed-source software owned by Kairos LLC (see
[LICENSE](https://github.com/Kairos-LLC/.website/blob/main/LICENSE) in the
`.website` and `.swift` repositories: "All Rights Reserved"). This document
is **not** an invitation for public or open-source contributions — it is an
internal workflow guide for authorized Kairos LLC contributors (employees,
contractors, and agents) working across the Kairos polyrepo. Access to these
repositories is itself restricted; if you can read this file, you are
expected to already be an authorized contributor.

If you're looking for the product overview, see the
[organization profile README](https://github.com/Kairos-LLC/.github/blob/main/profile/README.md).
For system design, see `ARCHITECTURE.md` (in `.website`).

---

## Repository map: where does my change go?

Kairos is split into three repositories. Pick the repository based on
*what* you're changing, not where you happen to be looking:

| Repository | Contains | Contribute here for... |
| :--- | :--- | :--- |
| [`.website`](https://github.com/Kairos-LLC/.website) | Next.js web app, marketing site, Supabase schema/migrations, server-side logic | Web client changes, marketing/landing page content, database schema changes, RLS policies, API routes, web-facing auth/session logic |
| [`.swift`](https://github.com/Kairos-LLC/.swift) | Native iOS application (Swift) | iOS UI, on-device schedule engine (`FireScheduleEngine.swift`), on-device recovery key handling (`RecoveryManager.swift`), App Store release concerns |
| [`.github`](https://github.com/Kairos-LLC/.github) | Org-level governance, community health files, workflow templates, org profile | This file, `CODE_OF_CONDUCT.md`, `req.txt`, shared GitHub Actions workflow templates, the org profile README |

A rule of thumb: if the change affects what ships to a browser or the
Supabase backend, it belongs in `.website`. If it affects what ships in the
iOS app bundle, it belongs in `.swift`. If it affects how the org or its
repos are governed rather than the product itself, it belongs here in
`.github`.

Some features (e.g. schedule pattern semantics) are mirrored conceptually
across `.website` and `.swift` for naming consistency — see `req.txt` for
the shared vocabulary. A schema or naming change on one side does not
automatically require an edit on the other side, but it should be called
out in the PR description so the sibling repo's maintainers are aware.

**Do not cross repository boundaries in a single PR.** Each repository has
its own history, review owners, and release cadence. If a change genuinely
requires coordinated edits in more than one repo (e.g. a schema rename that
both `.website` and `.swift` mirror), open a PR in each repository
separately and cross-link them in the PR descriptions.

---

## Branch naming

Branch names should be descriptive and prefixed by type:

- `feature/<short-description>` — new functionality
- `fix/<short-description>` — bug fixes
- `chore/<short-description>` — tooling, dependency bumps, non-functional cleanup
- `docs/<short-description>` — documentation-only changes
- `work/<unit-name>` — scoped units of work assigned to a specific
  contributor or agent task (e.g. `work/unit-14-governance-docs`)

Use lowercase, hyphen-separated words for the description
(`fix/access-code-expiry-check`, not `fix/AccessCodeExpiryCheck`). Keep
branch names short enough to be legible in a PR list.

---

## Commit messages

- Write commit messages in the imperative mood ("Add access code expiry
  check", not "Added" or "Adds").
- Keep the first line under ~70 characters; use the body for the "why" when
  it isn't obvious from the diff alone.
- Prefer several small, reviewable commits over one large commit, but don't
  split a single logical change across commits just to pad the count.
- Do not commit secrets, credentials, `.env` files, or any material that
  could reconstruct a user's recovery key. See `req.txt` for the privacy
  model this project is built around — treat it as a hard constraint on
  what may ever appear in a commit.

---

## Pull request conventions

- **One repository, one concern per PR.** Avoid bundling unrelated changes.
- **Title**: short, descriptive, in the imperative mood (e.g. "Add RLS
  policy for shared_schedules revocation").
- **Description** should cover:
  - What changed and why.
  - Which repository/repositories are affected (and links to companion PRs
    in sibling repos, if any).
  - How it was verified (tests run, manual verification steps, screenshots
    for UI changes).
  - Any follow-up work intentionally left out of scope.
- Link related issues or tracking items where they exist.
- Keep PRs small enough to review in one sitting where practical. Large,
  unavoidable changes (e.g. a schema migration touching many tables) should
  say so up front in the description so reviewers can budget time.

---

## Code review expectations

- All changes to `.website` and `.swift` require at least one review
  approval before merging, except for trivial fixes explicitly exempted by
  a repository maintainer.
- Changes to `.github` (this repository), including governance documents
  like this one, should also go through PR review rather than being pushed
  directly to `main` — governance docs are low-frequency but high-impact
  when wrong.
- Reviewers should check for:
  - Correctness and test coverage appropriate to the change.
  - Consistency with the domain vocabulary and privacy model in `req.txt`
    (in particular: no new server-side storage of recoverable secret
    material, no new account fields like email/password that contradict the
    recovery-key identity model).
  - Consistency with existing naming conventions, especially where
    `.website` schema mirrors `.swift` model names (see `req.txt`).
  - Security implications of any change touching access codes, recovery
    keys, or Row Level Security (RLS) policies.
- Address review feedback with follow-up commits on the same branch rather
  than force-pushing over history mid-review, unless the reviewer asks for
  a squash/cleanup pass before merge.
- Squash-merge is the default merge strategy unless a repository's own
  README/CONTRIBUTING notes otherwise.

---

## Questions

For questions about which repository a change belongs in, or about the
contribution process itself, open a discussion or issue in this
(`.github`) repository, or reach out to a Kairos LLC maintainer directly.
