# Changelog

The protocol line is **S2UI/1**. Repository releases are numbered
0.1.x while S2UI/1 is a draft with a single reference
implementation; the line will not be called 1.0 until a second,
independent implementation exists. Published schema files do not
change in place — a change is a new version beside the old one.

## 0.1.1 — 2026-11-23

Correction to the surface examples and the surface schema,
flagged by the surface implementer within hours of 0.1.0.

- **Surface examples regenerated from real composer output.**
  The 0.1.0 examples were draft compositions written in an
  A2UI **0.9.1-style** envelope (`version: "v1.0"`, separate
  `createSurface` / `updateComponents` / `updateDataModel`
  messages, `{"path": ...}` bindings, a draft Beth catalogId).
  S2UI/1 is a profile of A2UI **1.0**, and the shipped the reference implementation
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
  internal RunDigest v1, the public core
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
extracted from the reference composer's real composer (which landed after this
package's examples were composed).
