# elabftw-rdm

A research-data-management plugin for **elabFTW**, for labs in any domain — chemistry, physics, engineering, life sciences, computational. It helps a lab design a data schema that actually works — and gives the AI working the notebook the operating manual to keep it that way.

elabFTW enforces nothing: `required` is cosmetic, metadata is an unvalidated blob, links are untyped, tags are flat strings, schemas don't propagate, revisions cover only the body. An unstructured instance rots. This plugin makes the *agent* the schema, validation, provenance and aggregation layer, and makes the lab's structure an explicit, FAIR/ALCOA-grounded contract.

## Before you start

| You need | Why |
|----------|-----|
| **Claude Code** (CLI or desktop app) | the skills, `AskUserQuestion` and the decision sheets run there |
| **Opus 5.5 or newer** for the workshop | Designing a lab schema means holding the whole workflow, the live instance, a pattern catalogue and FAIR/ALCOA in one coherent design across many turns. Smaller models (Sonnet-, Haiku-class) lose the thread and produce contracts that don't resolve — don't use them for `/elabftw-schema-init`. Opus is also recommended for day-to-day work under the finished contract. Switch with `/model` or the desktop model picker. |
| **elabFTW 6.x** (5.6 mostly works) and an **API key** per team | 6.x separates resource templates from resource categories; the skills handle both |
| **`@sura_ai/elabftw` ≥ 0.7** MCP server | the tool surface the skills are written against (see *MCP servers* below) |
| A **browser** | the workshop shows its proposals as interactive HTML decision sheets |

## Quick start for a new lab

1. Install the plugin (below) and configure the `elabftw` MCP server — read-only is enough for the whole design phase.
2. In a folder for your lab's ELN work (it will hold the design dossier), start Claude Code with Opus and run `/elabftw-schema-init --lab=<short-name>`.
3. Answer the interview in your own words. Proposals come back as **decision sheets** that open in your browser: a diagram of the proposed structure, a "what a normal week looks like" storyboard, and a handful of decisions with the assistant's recommendation pre-marked. Answer there, press **Copy my answers**, paste into the chat. Small choices come as clickable questions in the chat.
4. Plan on 2–4 sessions; the dossier (`./elabftw-rdm-<lab>/`) makes every session resumable.
5. At the end you get the binding contract (`.claude/skills/labschema-<lab>/`), a human blueprint, and — only after your explicit approval, ideally on a test team first — the templates, categories and tags created in elabFTW.

## How the workshop keeps it digestible

Most researchers avoid ELN structuring because of the cognitive load. The workshop is built around that:

- **Picture first** — every proposed structure arrives as a diagram in a self-contained HTML sheet (`assets/decision-sheet.html`; no network, works offline, printable), never as a wall of tables.
- **Pre-answered decisions** — each decision carries a recommendation grounded in what the assistant learned about *your* lab, so "accept the recommendations" is always a fast path; disagreeing is treated as information.
- **Clickable questions** — discrete choices use Claude Code's `AskUserQuestion`, with the context in the question and the recommendation first.
- **Your words** — pattern codes, FAIR/PROV acronyms and elabFTW internals stay in the dossier; you see "batch page", "recipe version", "project tag".
- **Nothing changes silently** — every write to elabFTW or to your files is shown first and needs a yes.

## The three skills

| Skill | Role |
|-------|------|
| **`elabftw-mcp`** | Generic, lab-agnostic operating manual: the elabFTW data model, the metadata/extra_fields JSON, the MCP tool surface, what elabFTW does **not** do, and the workaround patterns. Loaded whenever an agent touches the elabftw MCP. |
| **`elabftw-schema-init`** | Interactive workshop. Surveys the live instance, interviews the team, maps their workflow to PROV/ISA, designs item types + extra-field schemas + tag namespaces + typed-link conventions, runs a FAIR/ALCOA pass, and **emits a `labschema-<lab>` skill** + a human blueprint. |
| **`labschema`** | Template for the per-lab structural **contract**. `elabftw-schema-init` instantiates it as `labschema-<lab>`. The working agent loads that alongside `elabftw-mcp` and enforces it on every read/write. Ships with a fully worked example. |

How they compose: `elabftw-mcp` (always) + `labschema-<lab>` (the binding contract for a specific instance). `elabftw-schema-init` is run once to create the contract, and again to refine it.

## References (normative backbone)

`references/` holds the principles `elabftw-schema-init` cites when justifying design choices:

- `fair-principles.md` — the 15 FAIR sub-principles → concrete elabFTW practice (Wilkinson 2016).
- `data-integrity-alcoa.md` — ALCOA/ALCOA+ (MHRA 2018), the DCC lifecycle, W3C PROV.
- `eln-modeling-patterns.md` — the elabFTW gap catalogue, the resource-vs-experiment rule, and the workaround patterns (P1–P9: structured tags for projects/methods, artifact passports, typed links, controlled vocabularies, schema versioning, agent-side validation, lock+timestamp, per-team replication, versioned protocols). The design core.
- `decision-sheets.md` — how proposals are presented: the sheet loop, its data shape, the writing rules, which sheet per phase.

## Install

Add the marketplace and install the plugin:

```
/plugin marketplace add letrplB/elabftw-rdm
/plugin install elabftw-rdm
```

(For local development, point the marketplace at a working copy instead: `/plugin marketplace add path/to/elabftw-rdm`.)

Then, in a lab's elabFTW: run `/elabftw-schema-init` and work through the workshop. Its final phase offers to wire your `CLAUDE.md` (see below) so every agent session that touches the notebook loads `elabftw-mcp` + the generated `labschema-<lab>`.

## MCP servers

The skills here drive MCP servers, not the elabFTW web UI. The binding one is **`elabftw`** — `elabftw-mcp` and any generated `labschema-<lab>` issue every read and write through its `mcp__elabftw__*` tools. Two further first-party servers complement the RDM work. All three are authored by the same maintainer as this plugin (the [`@sura_ai`](https://www.npmjs.com/org/sura_ai) npm scope; [`letrplB`](https://github.com/letrplB) on GitHub).

A ready-to-fill template ships as [`.mcp.json.example`](./.mcp.json.example):

```
cp .mcp.json.example .mcp.json   # then fill in your keys
```

The real `.mcp.json` is git-ignored, so your keys never get committed.

| Server | npm package | Role | Keys |
|--------|-------------|------|------|
| **`elabftw`** | [`@sura_ai/elabftw`](https://www.npmjs.com/package/@sura_ai/elabftw) | the notebook itself (`mcp__elabftw__*`) — the skills bind to this | `ELABFTW_BASE_URL` + an API key (**required**) |
| **`papers`** | [`@sura_ai/papers`](https://www.npmjs.com/package/@sura_ai/papers) | literature & DOI/citation resolution (CrossRef, OpenAlex, Semantic Scholar, CORE) — references and provenance context (R1.2) | all optional; runs keyless on CrossRef |
| **`chemistry`** | [`@sura_ai/chemistry`](https://www.npmjs.com/package/@sura_ai/chemistry) | PubChem compound metadata (CID / CAS / InChIKey) — optional, for substance-handling labs; one instance of the general P4 controlled-vocab pattern (FAIR-I2), which a non-chemistry lab fills with its own registry | none (public PubChem) |

**elabFTW keys are team-scoped.** A key is bound to the team you were viewing when you minted it (Settings → API keys). Single-team instances set `ELABFTW_API_KEY`; multi-team instances set one `ELABFTW_KEY_<teamId>` per team plus `ELABFTW_DEFAULT_TEAM` — every tool then takes an optional `team` parameter, matching the team-scope rule in `elabftw-mcp`. The example shows the multi-team shape.

**Write gating.** The server is read-only by default. `ELABFTW_ALLOW_WRITES=true` lets the agent create/update entities, fields, tags and links (the example enables it). The integrity primitives — `lock`, `timestamp`, sign, delete — are additionally gated behind `ELABFTW_ALLOW_DESTRUCTIVE=true` (off in the example); the **lock + timestamp** policy of pattern P7 needs it, so enable it on instances where the agent must produce defensible records.

## Wiring `CLAUDE.md` (additive, opt-in)

The plugin never edits your `CLAUDE.md` without consent. `/elabftw-schema-init`'s last phase offers to append a small marker-fenced block; it only ever writes between `<!-- BEGIN elabftw-rdm -->` and `<!-- END elabftw-rdm -->`, so re-runs update in place rather than duplicating, and content outside the markers is untouched. If no `CLAUDE.md` exists, it offers to create a minimal one containing just the block — it won't seed your file with anything else.

If you'd rather wire it by hand (or you're using `elabftw-mcp` read-only without running the workshop yet), paste this anywhere in your `CLAUDE.md`:

```markdown
<!-- BEGIN elabftw-rdm -->
## elabFTW research-data management — `elabftw-rdm` plugin

When working in an elabFTW instance via the elabftw MCP, load the **`elabftw-mcp`** skill (data model, MCP tool surface, gap catalogue, operating rules). If a `labschema-<lab>` skill has been generated for the instance, load that alongside — it is the binding contract and overrides generic guidance.

To design or refine a lab's schema with the researchers, run **`/elabftw-schema-init`**. Normative refs (FAIR, ALCOA+, DCC, PROV, the elabFTW gap→pattern catalogue) live under the plugin's `references/`.
<!-- END elabftw-rdm -->
```

## Status

v0.2 — usable. The tool surface matches `@sura_ai/elabftw` 0.7; the data-model claims are grounded in the elabFTW source (`src/sql/structure.sql`, API v2 OpenAPI) and the MCP server's tested behaviour. Entity-link extra fields take the target id as `value` (the server's `elab_set_extra_field` validates a positive integer id). One item to confirm per instance: the exact `extrafield:` grammar of the search DSL. See `CHANGELOG.md`.

License: MIT.
