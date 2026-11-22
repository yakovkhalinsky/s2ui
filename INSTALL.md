# Installing S2UI — by consumer

**Status: public DRAFT, release 0.1.0.** Repository:
https://github.com/yakovkhalinsky/s2ui — site:
https://yakov.khalinsky.com/s2ui/. "Installing" means vendoring
these files into your project, or pointing a tool at them. There
is no package registry entry. Licence: MIT (see `LICENSE`).

S2UI has three kinds of consumer. Find yours.

---

## 1. An MCP host / LLM agent

You run (or configure) an assistant that talks to an S2UI-speaking
supervisor over MCP, and you want the model to understand the
surfaces it receives. There is no code to install.

**Option A — catalogue instructions (the A2UI pattern).**
Point your agent SDK at `examples/beth.catalog.json` (draft) or
your supervisor's catalogue file. Its `instructions` field is
Markdown written to be injected into the model's system prompt by
the SDK.

**Option B — paste this into your agent's instructions:**

```markdown
## S2UI run surfaces

You may receive S2UI surfaces and RunDigest snapshots describing
supervised runs (a supervisor running a team of agents on a task).

- A RunDigest is a snapshot, not a delta. Replace the previous
  digest for a run with each new one. Never merge digests.
- Never state run facts (state, verdict, work units, tokens,
  attention) that are not in the latest digest for that run.
- An attention item needs a human only while it is listed in the
  digest. Its id is stable; commands must name that id.
- A digest with no verdict has no verdict yet. Absent token cost
  means "not reported", never $0.
- Surfaces are snapshots in assistant hosts: quote their
  updatedAt when freshness matters, and refresh rather than
  guessing newer state.
- You do not authorise actions. Commands are authorised by the
  supervisor, per run, server-side.
```

**Connecting:** your MCP client connects to the supervisor's MCP
server as usual. On the reference server, the relevant tools are
`list_run_digests` (all runs) and `supervisor_status` (one run);
both return RunDigest v1 as structured content **and** the
composed S2UI surface as an `application/a2ui+json` embedded
resource (implemented). Whether your host *renders* the
surface is host-dependent — in testing, the Muse host rendered
none of the A2UI/MCP Apps dialects natively for connector tool
results, and the payload degraded gracefully to the digest. A host
that ignores the surface resource loses nothing.

---

## 2. A supervisor implementer

You run a system that supervises agent teams and you want it to
speak S2UI. Three obligations:

1. **Emit RunDigest v1.** Fold your run record + events into the
   snapshot defined by `schema/run-digest-v1.schema.json` — the
   public core names (`subject`, `workUnits`, `digestVersion`…).
   Every renderable state must be in the digest; attention items
   appear only while unresolved, with stable ids.
2. **Expose the digests, with their surfaces.** All runs at once
   and per run, and embed the `application/a2ui+json` surface
   resource per spec §8, composed deterministically from the
   digest (Basic Catalog only in v1; durations as text).
3. **Accept the commands (DRAFT).** `answer_attention`,
   `approve_run`, `stop_run`, `check_run_state` — each
   idempotency-keyed, each authorised server-side per run, each
   re-checking current state immediately before acting, each
   audit-logged. See spec §9 and
   `schema/s2ui-command-v1-draft.schema.json`.

**Prove it:** run your own emitted digests through the validator:

```bash
node tools/validate --schema schema/run-digest-v1.schema.json your-digest.json
```

Your digests must validate for **every** run shape you can
produce — running, waiting on a human, failed, done — not just
the happy path.

---

## 3. A UI host / renderer

You render surfaces for a human.

- Support **A2UI 1.0** and the Beth catalogue id
  (`beth-v1-draft` in this package). Surfaces use Basic Catalog
  components only; durations are text.
- Treat digests as the state: rebuild surfaces from digests alone
  on reconnect (snapshot-replay, spec §7). Never require an event
  replay to render correctly.
- Label snapshot surfaces with their `updatedAt` and provide
  Refresh. Never present a snapshot as live (spec §3, principle 5).
- For hosts with MCP Apps but no native A2UI renderer, the
  fallback is an MCP App (`ui://`) resource consuming the
  **identical** S2UI payload (spec §8). DRAFT: no bundled renderer
  ships in this package.

---

## Provenance of the files

- `schema/run-digest-v1.schema.json` is the **public core
  contract**. It descends from the RunDigest v1 schema in the
  reference implementation with the
  generic-core naming pass applied (spec §4.3 gives the
  field-by-field mapping between the public names and the
  reference implementation's earlier internal names; the
  reference implementation adopted the public names on
  23 Nov 2026). Its `$id`
  is stable: `https://yakov.khalinsky.com/s2ui/schema/run-digest-v1.schema.json`.
- The other three schemas are **drafts written for this package**
  from the specification prose. Each says so in its
  `description`, and each carries a stable `$id` under
  https://yakov.khalinsky.com/s2ui/schema/.
