# Changelog

The protocol line is **S2UI/1**. Repository releases are numbered
0.1.x while S2UI/1 is a draft with a single reference
implementation; the line will not be called 1.0 until a second,
independent implementation exists. Published schema files do not
change in place — a change is a new version beside the old one.

## 0.2.0 — 2026-11-23

- **A real waiting-run example.** `examples/waiting-approval.rundigest.json`
  is a RunDigest captured from a live run while it was waiting on
  the human: an unresolved approval attention item, its prepared
  action, and the gated work unit — the fixture the 0.1.x examples
  lacked (every earlier digest had empty attention). Paired with
  its composed surface. The digest is anonymised the same way as
  the other real-run examples (subject and watch URL sanitised;
  the attention id shortened to an opaque form).
- **The command set is implemented.** §2's status table and the
  `answer-attention.command.json` example said no S2UI command
  tool ships yet; that stopped being true when the reference
  Beth layer shipped its command tools (Phase 4 of the Beth
  plan): start, answer, approve, stop and resume over an audited
  command layer with per-run authorisation, fresh-state checks
  and idempotency keys. The command example now targets the
  waiting-approval digest for real. The envelope schema remains
  DRAFT.

## 0.1.3 — 2026-11-23

References to the private reference implementation removed.

- Releases 0.1.0–0.1.2 named the reference implementation (a
  private codebase) and cited its commit hashes. A public
  specification cannot lean on a source its readers cannot see,
  so throughout the spec, schemas, README, INSTALL, examples
  and this changelog it is now described generically — "the
  reference implementation / supervisor / composer" — and the
  commit hashes are gone. (Earlier entries below were edited
  in the same pass; their substance is unchanged.) The example
  task text no longer names it either.
- No contract change: the only schema edits are description
  prose and one illustrative `extensions` key name.

## 0.1.2 — 2026-11-23

Documentation correction: the reference implementation adopted
the public core.

- The reference implementation adopted the published S2UI names
  the same day it was published (citing release 0.1.0): its
  RunDigest boundary now emits the public core
  (`digestVersion`, `subject`, `workUnits`, `finalizing`,
  `interrupted`, …) and its repository vendors the published
  schema file. Spec §4.3, the README and the RunDigest schema's
  `description` no longer describe the old naming as current —
  the mapping table is retained as the record of the earlier
  internal names, for comparing pre-adoption output.
- No contract change: the only schema edit is that one
  `description` sentence. (The reference implementation's
  vendored copy differs by that sentence until its next sync.)

## 0.1.1 — 2026-11-23

Correction to the surface examples and the surface schema,
flagged by the surface implementer within hours of 0.1.0.

- **Surface examples regenerated from real composer output.**
  The 0.1.0 examples were draft compositions written in an
  A2UI **0.9.1-style** envelope (`version: "v1.0"`, separate
  `createSurface` / `updateComponents` / `updateDataModel`
  messages, `{"path": ...}` bindings, a draft Beth catalogId).
  S2UI/1 is a profile of A2UI **1.0**, and the shipped reference
  composer uses the 1.0 form:
  `version: "1.0"`, a single `createSurface` carrying components
  and the full data model inline, `{"@path": ...}` bindings, and
  the A2UI Basic Catalog URL as `catalogId`. Both
  `*.surface.json` examples are now that real output, extracted
  from the reference supervisor's MCP tools; the only edit is
  the sanitised watch URL. The composer's text labels use the
  reference implementation's internal naming ("Packages") —
  see the mapping in spec §4.3.
- **Surface schema aligned** (`s2ui-surface-v1-draft.schema.json`,
  still DRAFT): now describes the A2UI 1.0 message list as
  actually emitted.
- **Spec §5/§6 aligned:** the snapshot form is defined as the
  single-`createSurface` payload; §5 records that surfaces carry
  the Basic Catalog URL as `catalogId` until a Beth catalog file
  ships with a stable id of its own.

The RunDigest schema — the implemented contract — is unchanged.

## 0.1.0 — 2026-11-23

Initial public draft.

- **Spec:** `spec/S2UI-1.md` — original prose covering the four
  parts of S2UI/1 (RunDigest, Beth catalog, command set, surface
  lifecycle rules), the MCP embedding (surfaces inside MCP tool
  results, with a `ui://` MCP App fallback carrying the identical
  payload), versioning, and conformance. Status is stated per
  part: RunDigest and the surface composer are implemented in the
  reference supervisor (its
  state-reporting MCP tools return composed
  `application/a2ui+json` surfaces beside the digest); the
  catalog file and the command set are DRAFT.
- **Schema:** `schema/run-digest-v1.schema.json` with the stable
  `$id` `https://yakov.khalinsky.com/s2ui/schema/run-digest-v1.schema.json`,
  plus three DRAFT schemas (surface envelope, Beth catalog,
  command envelope), each with a stable `$id` under the same base.
- **Naming pass applied.** Relative to the reference supervisor's
  internal RunDigest v1 (the reference implementation's), the public core
  renames `repo` → `subject`, `packages` → `workUnits`,
  `agents[].currentPackageId` → `currentWorkUnitId`,
  `packages.items[].branch` → `ref` (opaque locator),
  `attention[].packageId` → `workUnitId`; replaces the state
  `merging` with `finalizing` and adds `interrupted`; carries the
  in-band version as `digestVersion: "1"` (the internal schema's
  integer `version`); requires only the minimal core set
  (`digestVersion`, `runId`, `supervisorId`, `state`, `startedAt`,
  `updatedAt`, `seq`) with everything else optional-but-defined;
  makes `links` an open object; and adds an `extensions` object
  for implementation-profile data. The full mapping is spec §4.3.
  The reference implementation has **not** been renamed — the
  divergence is known and recorded, to be synced later.
- **Examples:** two real run digests from the reference
  supervisor (sanitised for publication: local paths and private
  watch URLs replaced), each paired with a clearly labelled DRAFT
  surface composition, plus a DRAFT catalog example and a DRAFT
  command example.
- **Validator:** `tools/validate` (Node, no dependencies) checks
  every example against its schema; all 6 pass.
- **Site:** the package is served at
  https://yakov.khalinsky.com/s2ui/ with schema `$id` URLs
  resolving to the published files.

Known gaps carried forward: no real waiting-on-a-human (attention)
digest fixture exists yet; the surface examples in this release
are draft compositions, to be replaced in 0.2.0 by output
extracted from the real composer (which landed after this
package's examples were composed).
