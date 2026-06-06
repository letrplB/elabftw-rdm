---
name: elabftw-schema-init
description: Interactive workshop that designs a research lab's elabFTW data schema together with the researchers, then emits it as a labschema-<lab> skill plus a human-facing blueprint. Surveys the live instance, interviews the team, maps workflows to PROV/ISA, designs item types + extra-field schemas + tag namespaces + typed-link conventions that mitigate elabFTW's structural gaps, and runs a FAIR/ALCOA conformance pass. Use when a lab is adopting elabFTW, has an unstructured instance, or says "/elabftw-schema-init", "set up our ELN schema", "design our lab data structure".
version: "0.1"
user-invocable: true
argument-hint: "[--lab=<slug>] [--team=<id>] [--dossier=<path>] [--seed] — lab slug for the generated skill; team to scope to; dossier dir for the workshop record (default ./elabftw-rdm-<lab>/); --seed also creates the items_types/categories/tags in elabFTW after approval."
allowed-tools: Read, Write, Edit, Glob, AskUserQuestion, ToolSearch, mcp__elabftw__*
---

# /elabftw-schema-init — design the lab's data schema

**Intent.** A research team rarely knows how to model their work in an ELN, and elabFTW enforces nothing, so unstructured instances rot. This is a guided workshop: you and the researchers co-design a schema that captures their real workflow as a queryable, FAIR, integrity-sound structure *despite* elabFTW's limits — then you emit it as a binding contract the working agent obeys.

This is **interactive and consultative**. Do not invent the lab's science. Ask, reflect back, converge. Never seed the live instance without explicit approval.

## Preconditions

1. The elabftw MCP server is configured. Load `elabftw-mcp` (the data model + tool surface + gap catalogue).
2. Read all three references: `references/eln-modeling-patterns.md` (the patterns you will apply), `references/fair-principles.md`, `references/data-integrity-alcoa.md`. These are the normative backbone — cite them when justifying design choices to the team.

## The workshop dossier (persistent, additive — survives across sessions)

This workshop is **stateful across conversations**. It maintains a per-lab **dossier**: a directory in the lab's workspace recording the described workflow and every design decision, so any later run — a refine pass, or a different agent — resumes with full context instead of starting cold. Keep the two roles distinct: the generated `labschema-<lab>` is the **contract** (what to enforce); the dossier is the **rationale, the described workflow, and the history**.

**Location.** `./elabftw-rdm-<lab>/` in the lab's working directory (alongside their `CLAUDE.md`), or the `--dossier` path. **Never** write it into this plugin's own repo — it is the lab's private data; the lab decides whether to version it.

**Files** — one intermediary artifact per phase, plus the accumulating spine:

| File | Written / updated by | Content |
|------|----------------------|---------|
| `README.md` | every phase | Manifest: lab, team(s), `schema_version`, a phase-status table (phase · status · date), open questions, and where the `labschema-<lab>` lives. The cheap "where we left off" load. |
| `00-instance-survey.md` | Phase 0 | Dated state-of-instance snapshot. |
| `01-workflow.md` | Phase 1 | The lab's **described workflow** in their own words + the PROV/ISA mapping. Grows and matures across runs. |
| `02-entity-model.md` | Phase 2 | The approved entity-model table. |
| `03-field-schema.md` | Phase 3 | Per-type `extra_fields` JSON + required sets. |
| `04-vocabulary-links.md` | Phase 4 | Tag namespaces + typed-link conventions. |
| `05-conformance.md` | Phase 5 | FAIR/ALCOA pass + residual gaps. |
| `decisions.md` | every phase | **Dated, append-only** log: what was chosen, what was rejected, and why. The spine that lets the dossier mature. |
| `blueprint.md` | Phase 6 | The human-facing blueprint. |

**Conventions.**
- **Load first.** On every run, *before* Phase 0's survey, locate the dossier; if it exists, load `README.md` + the relevant files and open with a short "**where we left off** — decided / open" recap. If none exists, this is an initial run: create the directory and say so.
- **Additive, never overwrite.** Append dated entries; supersede with a new entry that references the prior one (the same no-overwrite / deprecate-don't-rename discipline the schema itself follows, G13/P6). Re-date snapshots (`00-…`) rather than clobbering them.
- **Write as you go.** At the close of each phase (and on each approval) update that phase's file, append a `decisions.md` entry, and refresh the manifest's phase-status table — so the process is trackable mid-flight and resumable later.
- **Cheap to load.** The manifest is the index; deep files are read on demand.

## The workshop

### Phase 0 — Survey the live instance (read-only)

Inventory current reality before proposing anything. Via MCP: `elab_configured_teams`, `elab_list_teams`, `elab_list_items_types`, `elab_list_experiments_categories`, `elab_list_experiments_status`, `elab_list_items_status`, `elab_list_tags`, and a sample of recent `experiments`/`items` (`elab_get_bulk`, native view, with links). Produce a short **state-of-the-instance** summary: what types/categories/tags exist, adoption counts, naming inconsistencies, whether structured tags are already in use. Show it to the team — it grounds the conversation.

**Start of run:** first load any existing dossier and recap where the design left off (see *The workshop dossier*). **→ Dossier:** write the dated snapshot to `00-instance-survey.md`; create or refresh the manifest `README.md`.

### Phase 1 — Elicit the workflow (interview)

Use `AskUserQuestion` and open dialogue. Map answers onto **PROV** (Entity / Activity / Agent) and **ISA** (Investigation / Study / Assay). Question bank:

- *Programs & projects:* What research lines/branches exist? How should work sort by program vs. project vs. topic? (→ `branch:`/`project:`/`topic:` namespaces, P1)
- *The core loop:* Walk me through one experiment end to end — from idea to result. What is reused across runs (recipes, methods, SOPs, pipelines)? What is produced (samples, batches, devices, datasets, results)? (→ reusable procedure/method = `items_types`; run = `experiments`; product = passport `items`, P2)
- *Primary artifacts:* Is your work physical, computational, or a mix? What is the central thing it tracks — a sample/batch, a device/specimen, a dataset, a model or code version, an observation target? What must that thing's at-a-glance "passport" show, and what runs/analyses are performed against it? (→ P2 passport applied to your central artifact, `canonical:`/`batch:`/`<id>:`, `method:`)
- *Instruments & configuration:* Which instruments — and which settings (temperature, field, wavelength, gain, solver/version) — must be captured per run? (→ an `Instrument` items_type linked per run, P3)
- *Provenance:* For any result, which questions must be answerable in two clicks? (who/when/from-what/with-what — R1.2, PROV chain → P3 typed links)
- *Reuse standard:* What's the minimum a stranger needs to reproduce a run? (→ required fields, MIBBI/ISA, R1.3)
- *Integrity:* Which records are legal/defensible? Retention period? Who may unlock? (→ lock+timestamp policy, ALCOA)
- *Sharing:* Do you publish datasets? To which repository? License? (→ FAIR R1.1/I1, lifecycle handoff)
- *Vocabulary:* Where do free-text fields cause pain today? Which fields have a fixed set of values? (→ P4 controlled vocab — bind to the relevant registry: compounds/PubChem for chemicals, RRID/Addgene/GenBank for biological resources, ASTM/ISO for test methods, plus units and ORCID)

Reflect each answer back as a model fragment and confirm before moving on.

**→ Dossier:** capture the lab's described workflow (in their own words) + the PROV/ISA mapping in `01-workflow.md`; on a refine, append so it matures rather than overwriting.

### Phase 2 — Derive the entity model

From the workflow, propose (deriving the actual names from Phase 1 — do not assume synthesis): the **items_types** — a *reusable-procedure* type (a synthesis route / fab recipe / assay protocol / analysis pipeline), a *Method/SOP* type, a *central-artifact passport* (a Sample/Batch, a device/specimen, or a dataset/model), an *Instrument* type, optionally a *Study/Campaign* edge type for aggregation per G9; the **experiment categories** named for the lab's own activity verbs (e.g. Synthesis/Characterisation for chemistry; Fabrication/Measurement/Calibration for physics; Specimen-prep/Test/Simulation for engineering; Culture/Assay/Imaging for biology; Run/Processing/Analysis for computational) and the **status** lifecycle; which role is reusable (item) vs. one-time (experiment) vs. physical/central (passport). Present as a table with the PROV/ISA mapping. Get approval.

**→ Dossier:** record the approved model in `02-entity-model.md`; log the choice and any rejected alternatives in `decisions.md`.

### Phase 3 — Design the extra-field schema per type

For each items_type/template, design the `metadata.extra_fields` JSON: grouped fields (Identity / Provenance / Canonical pointers / Characterisation cache / Notes per P2), correct `type`s (entity-link types for provenance pointers per P3; `select` bound to controlled vocab per P4), `units`, `description`s, a `schema_version` field (P5), and which fields are **required** (you enforce this — P6; choose them from the lab's minimum-information standard, R1.3). Output ready-to-PATCH JSON. Confirm field-by-group with the team.

**→ Dossier:** save the per-type JSON + required-field sets to `03-field-schema.md`.

### Phase 4 — Design tag namespaces & typed-link conventions

Define the lab's controlled tag vocabulary (P1) — the namespace set, allowed values, the `canonical:`/`batch:` binding rule, the freeform-deduplication rule. Define the typed-link conventions (P3): which relation is a named extra-field link vs. a `rel:` tag vs. an edge items_type, each mapped to a PROV term. Document the canonical/passport pattern concretely for *this* lab.

**→ Dossier:** save the namespaces + typed-link conventions to `04-vocabulary-links.md`.

### Phase 5 — FAIR / ALCOA conformance pass

Walk the proposed schema against the 15 sub-principles and ALCOA+: every required-field choice cites a principle; confirm the no-overwrite + lock+timestamp policy, attribution (personal logins), license/export field, tombstone (soft-delete) rule, and repository-handoff step. Surface any residual gap explicitly — honesty over completeness.

**→ Dossier:** record the conformance pass + residual gaps in `05-conformance.md`.

### Phase 6 — Emit outputs (on approval)

1. **Generate `labschema-<lab>`** — instantiate the template at `skills/labschema/SKILL.md` (replace every `{{ }}` placeholder) into a new sibling skill directory `skills/labschema-<lab>/SKILL.md` (or the user's chosen location). This is the binding contract the working agent loads alongside `elabftw-mcp`. Include a **Design dossier** pointer (the `./elabftw-rdm-<lab>/` path) so the working agent can consult the rationale and described workflow for context.
2. **Generate the human blueprint** — a prose document for the lab (mirror the generalised blueprint structure), written to the dossier as `blueprint.md`, for the researchers.
3. **`--seed` only:** after explicit approval, create the items_types / categories / status / tag vocabulary in the live instance from the approved schema (additive only; never delete; verify each write native). Stamp `schema_version`.
4. **Finalise the dossier:** mark every phase complete in `README.md`, stamp `schema_version`, and append a `decisions.md` entry summarising this run.

### Phase 7 — Wire CLAUDE.md (offer; additive only)

Plugins don't auto-edit the user's CLAUDE.md, and the working agent needs to know to load `elabftw-mcp` + `labschema-<lab>` whenever the elabftw MCP is touched. Offer — never enforce — to wire this in. **Never modify content outside the markers.** Always show the diff first; the user owns their CLAUDE.md.

1. Locate the target file. Prefer `CLAUDE.md` in the cwd. If absent, ask: in a parent dir? a workspace root? a global `~/.claude/CLAUDE.md`? Don't guess.
2. If the file exists, search for the marker pair `<!-- BEGIN elabftw-rdm -->` / `<!-- END elabftw-rdm -->`:
   - **Present** → use `Edit` to replace the block between the markers (typically only the `labschema-<lab>` slug changes on a refine pass). Show the diff first.
   - **Absent** → use `Edit` to append the block to the end of the file. Show it first.
3. If the file does not exist, ask whether to create a minimal one **containing only this block** (do not seed unrelated content — the user populates their CLAUDE.md, not you).
4. If the user declines at any step, paste the snippet into chat so they can apply it themselves. Do not retry without consent.

**The block to insert** (substitute `{{lab_slug}}`; keep the markers verbatim — they make future runs idempotent):

````markdown
<!-- BEGIN elabftw-rdm -->
## elabFTW research-data management — `elabftw-rdm` plugin

When working in an elabFTW instance via the elabftw MCP, load the **`elabftw-mcp`** skill (data model, MCP tool surface, gap catalogue, operating rules) alongside the lab's binding contract — **`labschema-{{lab_slug}}`** for this instance. The labschema overrides anything generic in `elabftw-mcp`.

To refine the schema with the researchers, run **`/elabftw-schema-init`**. Normative refs (FAIR, ALCOA+, DCC, PROV, the elabFTW gap→pattern catalogue) live under the plugin's `references/`.
<!-- END elabftw-rdm -->
````

## Output contract

The generated `labschema-<lab>` MUST contain, fully resolved (no placeholders): lab identity + team mapping; the entity-model table with PROV/ISA mapping; per-type extra-field JSON; the tag-namespace registry (controlled vocabulary + rules); the typed-link/provenance conventions; the canonical/passport pattern for this lab's central artifact (or an explicit note that the lab has none); the validation rules the agent must enforce; the integrity policy; and a FAIR/ALCOA conformance note. It must be specific enough that a fresh agent, loading only `elabftw-mcp` + `labschema-<lab>`, works the instance correctly with no further questions.

The run also maintains the **workshop dossier** in the lab's workspace (`./elabftw-rdm-<lab>/`): the per-phase records, an append-only `decisions.md`, and the manifest — so later runs resume with full context and the design's history is auditable. The dossier is the lab's private data; never write it into this plugin's repo.

## Refinement mode

Re-running on an initialised lab is a **refine** pass: **load the dossier first** (`README.md` + `01-workflow.md` + `decisions.md`) and the existing `labschema-<lab>`, recap where the design left off, survey drift in the live instance (Phase 0), then propose a *diff* (new types/fields/namespaces, deprecations — never rename, per P6/G13; bump `schema_version`). Apply on approval, and **append** the changes to the dossier (new `decisions.md` entries; grow `01-workflow.md`) rather than overwriting — the dossier is the longitudinal record of how the schema matured.
