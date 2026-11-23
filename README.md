# S2UI — Supervisor-to-UI

**Status: public DRAFT, release 0.1.3 (23 Nov 2026).** The protocol
line is **S2UI/1**; the repository releases are numbered 0.1.x until
a second, independent implementation exists.

- Site: https://yakov.khalinsky.com/s2ui/
- Spec: [`spec/S2UI-1.md`](spec/S2UI-1.md) (also rendered at
  https://yakov.khalinsky.com/s2ui/spec.html)
- Schemas: [`schema/`](schema/) — each carries a stable `$id` under
  https://yakov.khalinsky.com/s2ui/schema/

S2UI/1 is a profile of A2UI 1.0 — not a fork — that standardises
how a supervisor of agent teams publishes run state to a user
interface, and how that interface asks the supervisor to act.
Its reference implementation — a supervisor, and the Beth
layer above it — is private and is not published; this
repository is the protocol, extracted so anyone can implement
it without it.

One honest summary of what exists: **RunDigest v1 is running
code**, and **the surface composer is running code** (the
reference supervisor's state-reporting MCP tools return
deterministically composed A2UI surfaces beside the digest).
The catalogue *file* and the command set are **specified here as
drafts** and are not implemented yet. The spec says which is
which, everywhere it matters. There is today exactly one producer
and no built consumer; the status table in spec §2 is the truth,
not the marketing.

## 10-minute quickstart

1. **Read the shape of the data** (2 min) — open
   `examples/dogfood-persistence.rundigest.json`: one real run of
   a supervisor fixing its own persistence bugs, as a snapshot:
   state, stage, agents, work units, verdict, tokens, a watch link,
   and a monotonic `seq`.
2. **See its surface** (2 min) — open the paired
   `examples/dogfood-persistence.surface.json`: the A2UI 1.0
   messages a UI renders for that digest — real output of the
   reference composer's implementation, extracted from
   its MCP tools (the watch URL is sanitised; the composer's
   text labels say "Packages" for work units — a display label,
   not a field name).
3. **See a run waiting on its human** (1 min) — open
   `examples/waiting-approval.rundigest.json`: a real run
   captured mid-flight with an unresolved approval in its
   attention, the prepared action it is gated on, and the work
   unit it blocks. `examples/answer-attention.command.json`
   is the command that answers it.
4. **Validate everything** (1 min):
   ```bash
   node tools/validate
   ```
   Expect 6 PASS lines. Then break a copy of a digest (change its
   `state`) and watch it fail with the exact reasons.
4. **Skim the spec** (5 min) — `spec/S2UI-1.md`, especially §4
   (RunDigest rules and the field mapping), §7 (surface
   lifecycle) and §9 (commands and authorisation).
5. **Pick your install path** — `INSTALL.md`: MCP host / LLM
   agent, supervisor implementer, or UI host.

## Layout

```
s2ui/
  README.md            this file
  CHANGELOG.md         release history
  LICENSE              MIT
  INSTALL.md           per-consumer installation
  spec/
    S2UI-1.md          the spec (original prose, RFC 2119 keywords)
  schema/
    run-digest-v1.schema.json        the public core contract
    s2ui-surface-v1-draft.schema.json    DRAFT
    beth-catalog-v1-draft.schema.json    DRAFT
    s2ui-command-v1-draft.schema.json    DRAFT
  examples/
    dogfood-persistence.rundigest.json   real run (reference supervisor on itself)
    dogfood-persistence.surface.json     real composer output, paired
    parallel-packages.rundigest.json     real run (parallel work units)
    parallel-packages.surface.json       real composer output, paired
    waiting-approval.rundigest.json      real run, captured while waiting on the human
    waiting-approval.surface.json        real composer output, paired
    beth.catalog.json                    DRAFT catalogue example
    answer-attention.command.json        DRAFT command example
  tools/
    validate           Node, no dependencies; validates examples or your files
    build-site         regenerates site/ from the canonical files
  site/                the static site served at yakov.khalinsky.com/s2ui/
```

## The public core vs the reference implementation's names

The RunDigest schema in this repository is the **public core**:
implementation-neutral field names (`subject`, `workUnits`, `ref`,
`digestVersion`, a `finalizing` state, an `interrupted` state, an
`extensions` object for profile data). The reference supervisor
originally emitted an earlier, implementation-specific naming
internally (`repo`, `packages`, `branch`, `version`, `merging`)
— and **adopted the public names on 23 Nov 2026**, citing
release 0.1.0 of this package. The mapping
table in spec §4.3 records the earlier names, for readers
comparing pre-adoption output. Neither naming is renamed in
place once published — a change is a new version beside the
old one.

## Examples provenance

Both digests were taken live from the reference supervisor's
`list_run_digests` on 23 Nov 2026 and transformed to the public
core names by the mapping in spec §4.3. The paired surfaces are
the reference composer's real output for the same runs,
taken from `supervisor_status` the same day. For publication, local
filesystem paths and private watch URLs were sanitised (the
`subject` fields are descriptive, and `links.watchUrl` uses
`example.invalid`). Both kept runs happen to be finished with a
pass verdict — the kept set contains no running or
waiting-on-a-human run, so this package has **no real example of
an attention item yet** (see spec §11).

## Licence

MIT — see [`LICENSE`](LICENSE).

## Naming note

No published protocol named S2UI or "Supervisor-to-UI" was found
(GitHub, spec/RFC and registry searches, 23 Nov 2026). Unrelated
uses exist (s2ui.com is an expo/media company; `s-ui`/`2s-ui` are
sing-box admin panels). This repository is the protocol work; do
not confuse it with those.
