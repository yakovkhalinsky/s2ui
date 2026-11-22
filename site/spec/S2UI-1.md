# S2UI/1 — Supervisor-to-UI

**Status: public DRAFT, release 0.1.3 (23 Nov 2026).**
This document is an original specification draft, published at
https://yakov.khalinsky.com/s2ui/ with its source in the public
`s2ui` repository. Implementation status is stated honestly in
§2: parts of S2UI/1 are running code (RunDigest v1 and the
surface composer, in the reference supervisor) and parts are
still design only (the catalogue file, the command tools, and
the Beth layer itself).

The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are to be
read as in RFC 2119.

---

## 1. What S2UI is

S2UI (Supervisor-to-UI) is a **profile of A2UI 1.0**, not a new
wire format and not a fork of A2UI. A2UI standardises how an agent
describes a user interface. S2UI standardises one narrower thing:
how a **supervisor** — a system that runs a team of agents on a
task, in stages, in work units, under a verdict, sometimes waiting
on a human — publishes the state of its runs so that a user
interface can render them, and how that interface asks the
supervisor to act.

An implementer who has never seen the reference implementation
should be able to implement S2UI from this document and the
schemas beside it. The reference implementation is private; this
specification is written to stand without it.

S2UI is named in parallel with A2UI (Agent-to-UI): A2UI covers
agent → screen; S2UI covers supervisor → screen.

### 1.1 Differentiation

- **Against raw event streams (e.g. AG-UI-style streaming):** a
  stream tells a UI what just happened in a live session. S2UI's
  core object is a **snapshot digest** of a run — its state,
  stage, work units, verdict and outstanding human attention —
  which a UI can render correctly after reconnecting, restarting,
  or arriving late, with no event replay.
- **Against product-specific supervisor contracts:** S2UI is a
  standalone, implementation-independent profile with a published
  schema. A supervisor keeps its own lifecycle, roles and runtime;
  S2UI only constrains what it publishes and the commands it
  accepts.

---

## 2. The four parts, and their status

S2UI/1 bundles four versioned parts:

| # | Part | Schema | Status (23 Nov 2026) |
|---|------|--------|----------------------|
| 1 | RunDigest v1 | `schema/run-digest-v1.schema.json` | **Implemented.** Shipping in the reference implementation (its `list_run_digests` tool, and `supervisor_status` → `structuredContent.digest`), validated against real runs. |
| 2 | Beth catalog v1 | `schema/beth-catalog-v1-draft.schema.json` | **DRAFT.** Defined in prose in the reference implementation's decision record; no catalogue file ships yet. The reference composer uses Basic Catalog components only (Text, Card, Column, List). |
| 3 | Command set | `schema/s2ui-command-v1-draft.schema.json` | **DRAFT.** Defined in prose; no S2UI command tool ships yet. |
| 4 | Surface lifecycle rules | `schema/s2ui-surface-v1-draft.schema.json` + §7–§8 | **Implemented in the reference supervisor.** Its state-reporting MCP tools return deterministically composed surfaces beside the digest. The surface *examples* in this package are real composer output (release 0.1.1; in 0.1.0 they were 0.9.1-style draft compositions — see the changelog). |

Anything in this spec that describes parts 2–3 describes the
**design contract**, not running behaviour. Conforming
implementations MUST NOT claim those parts are live in the
reference implementation until they are. Part 4's composer is
live in the reference supervisor; whether any given *host*
renders its surfaces is host-dependent (see §8).

---

## 3. Principles

1. **Deterministic composition only.** Surfaces are composed from
   RunDigest snapshots by code and templates. An LLM MUST NOT
   compose UI in the observability path.
2. **The supervisor is the source of truth.** A UI layer (Beth or
   any other) is a view and a command proxy. Killing the UI layer
   MUST never orphan, stop or lose a run. Supervisors MUST be
   re-discoverable from their own registry, not from the UI's
   cache.
3. **A detailed drill-in stays outside S2UI.** Every digest links
   to a per-run detailed view (`links.watchUrl`). S2UI
   surfaces summarise; they do not replace event feeds or logs.
4. **Actions are authorised by the supervisor, never by the
   envelope.** S2UI standardises how a command is expressed. It
   does not grant permission. See §9.
5. **Honest liveness.** A live surface MAY present itself as live.
   A snapshot surface MUST carry its `updatedAt` and offer
   refresh, and MUST NOT present itself as live.

---

## 4. RunDigest v1 (normative)

A RunDigest is a **snapshot of one supervised run**, folded from
the run's record and events. It is not a delta.

- A producer MUST emit digests that validate against
  `run-digest-v1.schema.json`.
- A consumer MUST replace the previous digest for a run with each
  new digest for that run. Consumers MUST NOT merge digests.
- `seq` is the number of the last event folded into the digest and
  is monotonic per run. A consumer that sees a lower `seq` than
  the one it holds SHOULD treat the newer-held digest as current.
- Every state a surface can render MUST be present in the digest.
  A surface MUST NOT invent run state (for example an attention
  item, a verdict, or a token total) that is not in the digest.
- **Attention items exist only while unresolved.** An attention
  item (kind `question`, `approval` or `blocked`) is present in
  `attention[]` only while it needs a human. Resolving it removes
  it from the next digest. Attention `id`s are stable for the life
  of the item, so a command can name one unambiguously.
- `verdict` is absent until a verdict has been posted. `tokens`
  is absent when no usage was reported; `tokens.costUsd` is
  present only when a provider reported a non-zero cost — a
  consumer MUST NOT render an absent cost as $0.
- Required top-level fields — the minimal core set is
  `digestVersion`, `runId`, `supervisorId`, `state`, `startedAt`,
  `updatedAt` and `seq`; everything else in the schema is
  optional-but-defined, so a producer omits a field it cannot
  populate honestly rather than fabricating it. States
  (`queued`, `running`, `waiting_human`, `verifying`,
  `finalizing`, `interrupted`, `done`, `failed`, `stopped`),
  stage, agent, work unit and attention shapes are defined by
  the schema, which is the normative source for this part.
  Timestamps are ISO 8601.

### 4.1 Known v1 limits (recorded, not hidden)

From the reference implementation's own mapping notes:

- In the reference implementation, no run maps to `queued`
  today; a run is `running` from the moment it is recorded. The
  enum keeps `queued` for supervisors that do queue.
- The core state enum includes `interrupted`, so a resumable
  run is distinguishable from a terminally stopped one. (Before
  adopting the public core, the reference implementation's
  internal enum mapped interrupted runs to `stopped` — see
  §4.3.)
- Per-work-unit `verdict` is a bare outcome string derived from
  the work unit's state only (`verified` → `pass`, `failed` →
  `fail`); it is not the run verdict object. Verdicts are
  recorded per run, not per work unit.
- `attention[].workUnitId` is in the schema but is not populated
  by the reference implementation today, because its pending
  authorisations are not recorded against a work unit.
- Stage `role` is the nominal owner of the stage, not a claim
  about which agent is active at that moment.

### 4.2 Generic-core naming (applied in this package)

The schema in this package is the **public core**: field names
are implementation-neutral, and implementation-specific data
belongs in the optional top-level `extensions` object, keyed by
profile name — never in new core fields. The naming pass that
produced the core was applied *before* first publication, when
renaming cost nothing; from publication on, the core follows
§10: no in-place renames, a change is a new version beside the
old one.

### 4.3 Mapping from the reference implementation's earlier naming

The reference implementation originally emitted an earlier,
implementation-specific naming internally. On 23 Nov 2026 it
**adopted this public core** (citing release 0.1.0 of this
package): its RunDigest boundary now emits the public names,
and its repository vendors this schema file itself. The table
below records the mapping from the earlier internal naming —
it is the transformation this package's digest examples were
published under, and it is what a reader needs when comparing
pre-adoption output from that implementation:

| Public core (this package) | Reference implementation's internal v1 (pre-adoption) |
|---|---|
| `digestVersion: "1"` (string) | `version: 1` (integer) |
| `subject` | `repo` (a local working-directory path) |
| `workUnits` | `packages` |
| `agents[].currentWorkUnitId` | `agents[].currentPackageId` |
| `workUnits.items[].ref` (opaque locator) | `packages.items[].branch` |
| `attention[].workUnitId` | `attention[].packageId` |
| state `finalizing` | state `merging` |
| state `interrupted` | mapped to `stopped` (no such state) |
| required: the minimal core set (§4) | all top-level fields required (several nullable) |
| `links` open object, `watchUrl` optional | `links.watchUrl` required |
| `extensions` object | — (not emitted) |

Consumers should expect current reference-implementation
output in public-core names; output produced before the
adoption commit uses the right-hand names. Producers
implementing S2UI from this spec MUST emit the public names.

---

## 5. The Beth catalog v1 (DRAFT)

A catalogue is an A2UI catalogue document: a stable `catalogId`,
the component names surfaces may use, named surface templates,
and an `instructions` field — Markdown that an agent SDK MAY
inject into an LLM's system prompt. That field is the LLM
installation mechanism; there is no code install for a host that
only needs to understand the catalogue.

For v1:

- Surfaces MUST use only the **A2UI Basic Catalog** components.
  Custom components (bars, meters, lanes, charts) are out of v1.
- Durations and elapsed times are rendered as **text**.
- The v1 templates are: `run-card` (one run's glance card),
  `attention-card` (one unresolved attention item), and
  `portfolio` (the cross-run surface, §7).
- Until a Beth catalog file ships with a stable `catalogId` of
  its own, surfaces carry the **A2UI Basic Catalog URL** as
  their `catalogId` — that is what the reference composer emits
  today. Adopting a future Beth `catalogId` is a one-constant
  change in a composer, not a protocol change.

---

## 6. Surfaces (DRAFT)

An S2UI surface payload is a list of A2UI 1.0 messages for one
surface. The **snapshot form** — the form the reference composer
emits today, and the form of this package's examples — is a
single `createSurface` message (version `"1.0"`) carrying the
components and the **full data model inline**. Component text
binds to the data model with `{"@path": ...}` references. Live
surfaces MAY follow with `updateComponents` and
`updateDataModel` messages, and `deleteSurface` when the
surface's lifecycle ends (§7).

- A run surface is composed from exactly one RunDigest. A
  portfolio surface is composed from a list of RunDigests.
- Composition MUST be deterministic: the same digest(s) in, the
  same surface out. In the snapshot form the whole data model
  travels with `createSurface`; in a live surface, changing data
  SHOULD travel in `updateDataModel`, with component structure
  changing only when the template itself changes.
- A run's glance content is, in order: a title, the task
  summary, state, stage (n of m, name), elapsed time as text,
  the work-unit line (done/total), verdict if present, the team
  rows, the work-unit rows, the drill-in link, and a snapshot
  line carrying the digest's `updatedAt` and `seq`. (The
  reference composer's labels use its internal naming —
  "Packages" for work units — a display label, not a field
  name; see §4.3.) An attention card
  exists for each unresolved attention item in the digest, and
  for nothing else.

---

## 7. Surface lifecycle rules (DRAFT)

- There is **one long-lived `portfolio` surface** and **one
  `run_<runId>` surface per active run**.
- A run surface is created when the run launches and deleted
  when the run is **archived** — not when it finishes. A finished
  run's card remains visible, showing its verdict.
- Ordinary run churn (a run starting, progressing, finishing) is
  **data-only**: `updateDataModel` against templates defined
  once. It MUST NOT create new component structure per event.
- **Snapshot-replay is the reconnect story.** A2UI provides no
  resync. On (re)connect, the composer rebuilds every surface
  from current digests. A consumer MUST be able to discard its
  surfaces and rebuild from digests alone.
- **Coalescing:** a composer SHOULD emit at most one data-model
  update per run per 1–2 seconds, except that state transitions
  and new attention items SHOULD be emitted immediately.

---

## 8. The surface inside MCP tools (the §3A embedding)

The S2UI surface ships **inside** MCP tool results, not beside
them. Every MCP tool that reports run state also returns the
S2UI surface for that state.

**Supervisor tools (reference names):**

- `supervisor_status` returns the run's RunDigest **and** an
  `application/a2ui+json` resource carrying that run's surface
  (glance card + attention cards). Existing text/JSON content
  stays, so hosts without A2UI degrade gracefully.
  *Status today: implemented — digest in
  `structuredContent.digest`, surface as an embedded resource
  and in `structuredContent.surface`.*
- `list_run_digests` returns all digests **and** the portfolio
  surface composed from them.
  *Status today: implemented, as above.*
- **Host rendering is host-dependent.** In the reference
  implementation's dialect
  test, the Muse host rendered none of the A2UI / MCP Apps
  dialects natively for connector tool results; the payload
  ships in canonical A2UI 1.0 form and degrades gracefully to
  the digest. A producer MUST NOT treat "the host rendered it"
  as part of conformance.
- Raw event tools (e.g. `supervisor_events`) are unchanged. Raw
  events stay out of S2UI envelopes.

**Layer tools (reference names):**

- Model-visible: `beth_portfolio`, `beth_run`, and (later)
  `beth_start_run` — each returns digests plus the surface.
- App-only (`visibility: ["app"]`, so they do not pollute a
  model's tool list): `beth_answer_attention`,
  `beth_approve_run`, `beth_stop_run`, `beth_refresh` — the
  callbacks a surface's buttons invoke. Each returns a lone
  `updateDataModel` or a refreshed surface, so the card updates
  in place.

**Host fallback:** for hosts with MCP Apps but no native A2UI
renderer, the same tools also carry an MCP App (`ui://`)
resource bundling a renderer that consumes the **identical**
S2UI payload. One payload, two render paths — never two payloads
to keep in sync.

---

## 9. Commands (DRAFT)

The command set is: `answer_attention`, `approve_run`,
`stop_run`, and `check_run_state`. The request envelope is
defined by `s2ui-command-v1-draft.schema.json`.

- Every command MUST carry an `idempotencyKey`. Replaying a key
  MUST NOT perform the action twice.
- Every command MUST be **authorised server-side, per run**,
  before it acts. The envelope is never the authority, and a
  caller-supplied identity field is audit data, not proof.
- Before a consequential command (`approve_run`, `stop_run`,
  and `answer_attention` where it releases work), the server
  MUST re-check current state (`check_run_state`) immediately
  before acting — including comparing the caller's `expectedSeq`,
  when supplied, against the run's current `seq`. A stale card
  MUST produce a refusal the UI can show, not a guess.
- The server MUST keep an audit log of commands: who, action,
  runId, attentionId where relevant, outcome, time.
- Consequential commands SHOULD require a human confirmation in
  the UI (a modal for stop and merge-approve).
- The first action to be wired in any implementation SHOULD be
  `answer_attention`: it is the smallest command that proves the
  whole loop (card → command → digest changes → card clears).

---

## 10. Versioning

- **S2UI and RunDigest version independently** of each other, of
  any implementation, and of A2UI.
- This package is a **draft of S2UI/1**. A public release of this
  package SHOULD be numbered 0.1.x while the protocol line stays
  S2UI/1, and MUST NOT be called 1.0 until a second, independent
  implementation exists.
- Schemas carry a stable `$id`. A published schema file MUST NOT
  change in place; a change is a new version beside the old one.
- **Dialects:** a host that speaks A2UI 0.9.1 receives the 0.9.1
  dialect of the same S2UI payloads. Dialect choice is per target
  host and MUST be decided by testing that host, not assumed.

---

## 11. Conformance

An implementation is S2UI-conforming for a part when:

- **Producer (RunDigest):** its digests validate against
  `run-digest-v1.schema.json` for every run shape it can produce,
  including a run waiting on a human and a failed run.
- **Composer (surfaces, DRAFT):** the same digests produce the
  same surfaces on repeat composition, and a rebuild from
  digests alone reproduces the live surfaces (snapshot-replay).
- **Commander (DRAFT):** replayed idempotency keys do not repeat
  actions, unauthorised commands are refused, and a stale
  `expectedSeq` is refused.

`tools/validate` in this package checks example and producer
files against the schemas. Passing it is necessary, not
sufficient, for conformance.

---

## 12. Non-goals

- Replacing a supervisor's detailed per-run UI (event feeds,
  logs) — S2UI links to it instead.
- LLM-composed UI anywhere in the observability path.
- Scheduling, repository management, or log viewing.
- Agent-to-agent transport. S2UI is supervisor-to-UI only.

---

## Provenance

Original prose written for this package, 23 Nov 2026, from the
reference implementation's private decision record, its
RunDigest v1 implementation and its surface composer. First
published 23 Nov 2026 as release 0.1.0; release 0.1.1 the same
day replaced the draft surface examples with real composer
output in the A2UI 1.0 envelope form, 0.1.2 recorded the
reference implementation's adoption of the public core (§4.3),
and 0.1.3 removed references to that private implementation by
name throughout the package. No text is adapted from A2UI's
documents.
