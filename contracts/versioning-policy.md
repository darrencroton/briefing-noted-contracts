# Contracts Versioning Policy (v1.0)

This repository is the single source of truth for the cross-component contracts between `briefing` and `noted`. Everything that is written here is a promise to both consumers; everything that changes here has to be rolled out deliberately on both sides.

## What is versioned

Three kinds of version move together on each tag:

1. **The repository tag** — semver on the `briefing-noted-contracts` root repo. This is the anchor consumers pin against.
2. **Schema files** — JSON Schemas live at `schemas/<name>.v<major>.json`. The `<major>` is encoded in the filename so that a v2 schema can sit alongside v1 during a transition. Each schema file pins its own `schema_version` as `<major>.<minor>` inside the document, identical for all files in a given repo tag.
3. **Payload `schema_version` field** — every manifest and completion payload carries `schema_version: "<major>.<minor>"`. Readers use this to decide compatibility.

## Compatibility rule (§8.4)

- **Major match required.** A consumer pinned to schema major N accepts only payloads with the same major N.
- **Forward-tolerate minor.** A v1.0 reader accepts a v1.1 payload: unknown fields are ignored.
- **Backward-strict minor.** A v1.1 reader does **not** accept a v1.0 payload that is missing a v1.1-required field; validation fails.
- **Unknown major → fail.** `noted validate-manifest` returns exit 2.

### How the schemas enforce this

- `schema_version` in `schemas/manifest.v1.json` and `schemas/completion.v1.json` is a pattern matching `^1\.[0-9]+$` — **not** a const of `"1.0"`. Any 1.x payload therefore passes the major check; minor drift is handled by JSON Schema's default `additionalProperties: true`, which ignores unknown fields.
- Enums (stop reasons, terminal statuses, mode types, audio strategies, ASR backends, runtime statuses, runtime phases) are closed in the schemas. Adding a value to any of them is breaking, and therefore a major bump. A reader cannot simultaneously be strict about unknown enum values and forward-tolerant about them; the contract chooses strictness, which is why enum additions are major.

## When to bump

### Patch (1.0.0 → 1.0.1)

- Documentation or comment changes.
- Example fixtures added or corrected.
- Clarifications that do not change any schema, enum value, file layout, CLI exit code, or vocabulary string.

No schema field changes. Payload `schema_version` stays at `1.0`.

### Minor (1.0 → 1.1)

Additive, backward-compatible changes **only**. Consumers on 1.0 must continue to work against 1.1 payloads, and consumers on 1.1 must still accept 1.0 payloads that omit fields introduced in 1.1.

Examples:

- Adding a new optional field to a schema.
- Adding a new optional CLI command or optional flag that does not change existing exit codes.
- Adding a new optional file to the session directory.

Not examples (these are **major**, see below):

- Adding a new **required** field.
- Adding a value to any locked vocabulary (stop reasons, terminal statuses, runtime statuses, runtime phases, mode types, audio strategies, ASR backends, transcript filenames). Enums in the schemas are closed, and readers reject unknown values by design (`contracts/vocabulary.md`). An added value is therefore breaking for any older reader that receives it.
- Changing the meaning or type of an existing field.
- Removing or renaming any field, enum value, CLI command, exit code, or filename.
- Tightening a regex, enum, or range that previously accepted a value.

On a minor bump:

- Bump `schema_version` in the relevant schemas to `1.1`.
- Write a `CHANGELOG.md` entry listing the added fields / enum values.
- Release as `v1.1.0` on the root repo.

### Major (1.x → 2.0)

Breaking changes. This includes any of the "not examples" above — notably, any change to a locked vocabulary (added stop reasons, terminal statuses, runtime statuses, runtime phases, mode types, audio strategies, ASR backends, transcript filenames).

Rules:

- Create new schema files at `schemas/<name>.v2.json`. The v1 files remain in the repo until both consumers have migrated.
- Update `cli-contract.md`, `session-directory.md`, and `vocabulary.md` to reflect v2 shape; keep v1 notes visible where a behaviour diverges.
- Release as `v2.0.0` on the root repo after both `briefing` and `noted` have opened draft implementations against v2 on branches.
- Consumers migrate by bumping the contracts pin and updating payload emission in lockstep.

## Change proposal process

1. **Open a PR** against `main` of the contracts repo with:
   - The schema / spec changes.
   - A diff summary in the PR description.
   - A short migration note: what consumers have to do.
   - A `CHANGELOG.md` entry for the forthcoming version.
2. **Classification.** Mark the PR `patch`, `minor`, or `major`. Reviewer verifies the classification against the rules above.
3. **Approval.** The repository owner (currently Darren) approves. For `major` bumps, approval requires a rollout plan that names the `briefing` and `noted` branches that will adopt v2.
4. **Merge.**
5. **Tag** `vX.Y.Z` on the root repo **immediately** after merge. Untagged contracts commits are not considered released — consumers must pin to tags, not to commits.
6. **Consumer adoption** happens in each sibling repo as a separate PR that bumps the contracts pin.

## Why tags, not commits

Tagging is what makes a schema version real. Two consequences:

- **Never edit a released tag.** If a release is wrong, cut a new patch release. Re-pointing a tag silently breaks consumers who have already pinned.
- **Never ship code that emits a `schema_version` that is not tagged.** If `schema_version: "1.1"` is on `main` but not tagged yet, no producer should be writing `1.1` payloads — the receiver has no stable reference for the shape.

## Authorisation

| Change type | Approver                                   |
|-------------|--------------------------------------------|
| Patch       | Repo owner                                 |
| Minor       | Repo owner + one of the consumer maintainers |
| Major       | Repo owner + both consumer maintainers, with a rollout plan |

In the solo-developer phase, all three roles collapse into one person. The requirement then is that a major bump has an explicit rollout note written down in the PR before the tag is cut — so that the decision is deliberate rather than quiet.

## What lives outside the version bump

- Consumer code (`briefing`, `noted`) ships on its own release cadence. Consumer versions are independent of the contracts tag, but a release note in each should record which contracts tag it is pinned to.
- Fixtures in `fixtures/` (populated in Step 5 of the action plan) may be refreshed without a version bump as long as they continue to satisfy the current schemas. Additions, removals, or replacements that change fixture semantics warrant a patch bump for traceability.
