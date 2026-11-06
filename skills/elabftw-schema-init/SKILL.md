---
name: elabftw-schema-init
description: Interactive workshop that designs a research lab's elabFTW data schema together with the researchers, then emits it as a labschema-<lab> skill plus a human-facing blueprint. Surveys the live instance, interviews the team, decides what becomes a resource vs. an experiment, how repeatable protocols are versioned, which project/method tags structure the work, designs templates + extra-field schemas + typed-link conventions that mitigate elabFTW's structural gaps, and runs a FAIR/ALCOA conformance pass — presenting every larger decision as a visual, pre-answered HTML decision sheet. Use when a lab is adopting elabFTW, has an unstructured instance, or says "/elabftw-schema-init", "set up our ELN schema", "design our lab data structure", "organise our elabFTW".
version: "0.2"
user-invocable: true
argument-hint: "[--lab=<slug>] [--team=<id>] [--dossier=<path>] [--seed] — lab slug for the generated skill; team to scope to; dossier dir for the workshop record (default ./elabftw-rdm-<lab>/); --seed also creates the templates/categories/tags in elabFTW after approval."
allowed-tools: Read, Write, Edit, Glob, Grep, AskUserQuestion, ToolSearch, Bash(open:*), Bash(xdg-open:*), Bash(cp:*), Bash(mkdir:*), mcp__elabftw__*
---

# /elabftw-schema-init — design the lab's data schema

**Intent.** A research team rarely knows how to model their work in an ELN, and elabFTW enforces nothing, so unstructured instances rot. This is a guided workshop: you and the researchers co-design a schema that captures their real workflow as a queryable, FAIR, integrity-sound structure *despite* elabFTW's limits — then you emit it as a binding contract the working agent obeys.

This is **interactive and consultative**. Do not invent the lab's science. Ask, reflect back, converge. Never seed the live instance without explicit approval.

**Plugin paths.** This skill's base directory is `<plugin>/skills/elabftw-schema-init/`. `references/` and `assets/` sit at the plugin root, two levels up (`<plugin>/references/…`, `<plugin>/assets/…`).

## Before you start — model check

This workshop holds a lab's whole workflow, the live instance's state, a pattern catalogue and three normative frameworks in one coherent design over many turns. It needs the most capable model available: **Opus 5.5 or newer**. Smaller models (Sonnet-, Haiku-class) lose the thread across phases, contradict earlier decisions and produce contracts that do not resolve — do not run the workshop on them.

At the start of every run, check which model you are. If you are not Opus 5.5 (or a newer Opus-class model), say so in your first message and recommend switching before continuing (`/model` in Claude Code, or the model picker in the desktop app). Proceed only if the user explicitly chooses to continue anyway — then record that in `decisions.md`.

## Talking to the researchers — the digestibility contract

Most researchers avoid this work because of the cognitive load. Your job is to make every step cheap to digest. These rules bind every phase:

1. **Picture before prose.** Any proposed structure (entity model, link conventions, tag map, seed plan) is shown as a **decision sheet** with a diagram — `assets/decision-sheet.html`, data-only, see `references/decision-sheets.md` for the loop, the data shape and the writing rules. Never describe a structure only in chat tables.
2. **`AskUserQuestion` for discrete choices.** Whenever the next step depends on a choice among 2–4 options, use the tool instead of writing the options as prose. Make each question carry its own context:
   - **Question text** says what is being decided *and* the fact you learned that makes it relevant: "You split every batch for XRD, BET and TEM — should each batch get its own overview page?" — not "Passport pattern?".
   - **Header** (≤ 12 chars) names the topic: `Batch page`, `Recipes`, `Projects`.
   - **Options:** concise label (1–5 words); the description states the consequence in daily-work terms ("one click from a batch to all its results; created automatically with the run").
   - **Your recommendation first**, labelled `(Recommended)`, and only when something you learned about this lab supports it — say what in the description. No basis → no recommendation; say it is their call.
   - Use `multiSelect` for non-exclusive choices (which methods, which instruments). Use option `preview`s to show the concrete artefact being compared (a tag list, a field set, a short JSON excerpt).
   - Up to 4 questions per call; batch related ones. More than 4, or interdependent ones → a sheet.
   - Do **not** use it for the lab's narrative (open dialogue) or for questions whose answer you can find in the instance yourself.
   - If `AskUserQuestion` is unavailable on this surface, fall back to a short numbered list with the recommendation marked — still one decision per line.
3. **Their words.** Use the lab's own names for things. Pattern codes (P1–P9, G1–G15), PROV/ISA/FAIR acronyms and elabFTW internals stay in the dossier and the contract; in conversation say "resource template", "experiment template", "field", "tag", "link" and explain a term once when it is unavoidable.
4. **Pre-answered decisions.** Every decision arrives with a recommended default, so "accept the recommendations" is always a valid, fast path. Disagreement is information: when the researcher overrides you, reflect back what you learned in one line.
5. **Show the week.** For every structural proposal, a 3–5 step storyboard of real members doing real work under it ("Anna starts a synthesis from Recipe v2 → …").
6. **Small bites.** At most one sheet or one `AskUserQuestion` batch per turn. End each phase with a three-line summary: decided / still open / next. A phase per session is fine — the dossier makes it resumable.
7. **Nothing changes silently.** Before any write to elabFTW or to the user's files, show what will change (sheet `table` or diff) and get a yes via `AskUserQuestion`.

## Preconditions

1. The elabftw MCP server is configured (≥ 0.7, `@sura_ai/elabftw`). Load the `elabftw-mcp` skill (data model, tool surface, gap catalogue). The workshop itself is read-only until Phase 6 and works with writes disabled.
2. Read the references: `references/eln-modeling-patterns.md` (the patterns you will apply — including the resource-vs-experiment rule and the protocol pattern P9), `references/decision-sheets.md` (how to present), `references/fair-principles.md`, `references/data-integrity-alcoa.md`. The first two are working manuals; the last two are the normative backbone — cite them in the dossier when justifying design choices, not at the researchers.

## The workshop dossier (persistent, additive — survives across sessions)

This workshop is **stateful across conversations**. It maintains a per-lab **dossier**: a directory in the lab's workspace recording the described workflow and every design decision, so any later run — a refine pass, or a different agent — resumes with full context instead of starting cold. Keep the two roles distinct: the generated `labschema-<lab>` is the **contract** (what to enforce); the dossier is the **rationale, the described workflow, and the history**.

**Location.** `./elabftw-rdm-<lab>/` in the lab's working directory (alongside their `CLAUDE.md`), or the `--dossier` path. **Never** write it into this plugin's own directory — it is the lab's private data; the lab decides whether to version it.

**Files** — one intermediary artifact per phase, plus the accumulating spine:

| File | Written / updated by | Content |
|------|----------------------|---------|
| `README.md` | every phase | Manifest: lab, team(s), `schema_version`, a phase-status table (phase · status · date), open questions, where the `labschema-<lab>` lives, and the model used. The cheap "where we left off" load. |
| `00-instance-survey.md` | Phase 0 | Dated state-of-instance snapshot. |
| `01-workflow.md` | Phase 1 | The lab's **described workflow** in their own words + the PROV/ISA mapping. Grows and matures across runs. |
| `02-entity-model.md` | Phase 2 | The approved entity-model table (incl. resource-vs-experiment and protocol decisions). |
| `03-field-schema.md` | Phase 3 | Per-type `extra_fields` JSON + required sets. |
| `04-vocabulary-links.md` | Phase 4 | Tag namespaces + typed-link conventions. |
| `05-conformance.md` | Phase 5 | FAIR/ALCOA pass + residual gaps. |
| `06-seed-log.md` | Phase 6 | What was created/migrated in elabFTW: kind, id, team, date — the undo map. |
| `decisions.md` | every phase | **Dated, append-only** log: what was chosen (option text), what was rejected, who decided, and why. Pasted sheet exports go here verbatim. |
| `sheets/` | every phase | The decision sheets (`NN-<topic>.html`), re-issued as `-v2`, `-v3` rather than edited. |
| `blueprint.md` + `sheets/blueprint.html` | Phase 6 | The human-facing blueprint (prose + visual). |

**Conventions.**
- **Load first.** On every run, *before* Phase 0's survey, locate the dossier; if it exists, load `README.md` + the relevant files and open with a short "**where we left off** — decided / open" recap. If none exists, this is an initial run: create the directory and say so.
- **Additive, never overwrite.** Append dated entries; supersede with a new entry that references the prior one (the same no-overwrite / deprecate-don't-rename discipline the schema itself follows, G13/P6). Re-date snapshots (`00-…`) rather than clobbering them.
- **Write as you go.** At the close of each phase (and on each approval) update that phase's file, append a `decisions.md` entry, and refresh the manifest's phase-status table — so the process is trackable mid-flight and resumable later.
- **Cheap to load.** The manifest is the index; deep files are read on demand.

## The workshop

### Phase 0 — Survey the live instance (read-only)

Inventory current reality before proposing anything. Via MCP: `elab_me`, `elab_info` (elabFTW version — 6.x separates resource *templates* from resource *categories*), `elab_configured_teams`, `elab_list_teams`, `elab_list_team_users`, `elab_list_items_types`, `elab_list_items_categories` (6.x), `elab_list_templates`, `elab_list_experiments_categories`, `elab_list_experiments_status`, `elab_list_items_status`, `elab_list_tags`, `elab_list_extra_field_names` (instance-wide field names + frequency — the fastest read of what structure people already use), and a sample of recent `experiments`/`items` (`elab_search` / `elab_get_bulk` with `include:["links"]`).

Produce a short **state-of-the-instance** picture: what exists, adoption counts, naming inconsistencies, freeform tags that should become structured, fields used under several spellings, whether structured tags are already in use. Show it as `sheets/00-instance-survey.html` (diagram of what exists today + top tags/fields + inconsistencies) — it grounds the conversation and often earns the researchers' trust ("that is exactly our mess").

**Start of run:** first load any existing dossier and recap where the design left off (see *The workshop dossier*). **→ Dossier:** write the dated snapshot to `00-instance-survey.md`; create or refresh the manifest `README.md`.

### Phase 1 — Elicit the workflow (interview)

Open dialogue first — ask them to walk you through one real piece of work end to end, and listen. Then close gaps with `AskUserQuestion` (discrete facts: which methods, how often, who). Map answers onto **PROV** (Entity / Activity / Agent) and **ISA** (Investigation / Study / Assay) in the dossier, not at them. Question bank:

- *Programs & projects:* What research lines exist? How should work sort — by program, project, topic, funding? Who needs to see "everything of project X"? (→ `branch:`/`project:`/`topic:` namespaces, P1)
- *The core loop:* Walk me through one experiment end to end — from idea to result. What is reused across runs (recipes, methods, SOPs, pipelines)? What is produced (samples, batches, devices, datasets, results)? (→ resource vs. experiment rule; passport, P2)
- *Repeatable protocols:* Which procedures do you repeat? How often do they change, and do you need to know which version a run followed? Do people deviate from them, and where is the deviation written down? (→ protocol pattern P9)
- *Primary artifacts:* Is your work physical, computational, or a mix? What is the central thing it tracks — a sample/batch, a device/specimen, a dataset, a model or code version? What must its at-a-glance page show, and which runs/analyses are performed against it? (→ P2 passport, `canonical:`/`batch:`, `method:`)
- *Instruments & configuration:* Which instruments — and which settings (temperature, field, wavelength, gain, solver/version) — must be captured per run? (→ an `Instrument` resource linked per run, P3)
- *Provenance:* For any result, which questions must be answerable in two clicks? (who/when/from-what/with-what — R1.2, PROV chain → P3 typed links)
- *Reuse standard:* What's the minimum a stranger needs to reproduce a run? (→ required fields, MIBBI/ISA, R1.3)
- *Integrity:* Which records are legal/defensible? Retention period? Who may unlock? (→ lock+timestamp policy, ALCOA)
- *Sharing:* Do you publish datasets? To which repository? License? (→ FAIR R1.1/I1, lifecycle handoff)
- *Vocabulary:* Where do free-text fields cause pain today? Which fields have a fixed set of values? (→ P4 controlled vocab — bind to the relevant registry: compounds/PubChem for chemicals, RRID/Addgene/GenBank for biological resources, ASTM/ISO for test methods, plus units and ORCID)
- *People:* Who will enter data, who reviews, who administers elabFTW? (→ storyboards, permissions, unlock authority)

Reflect each answer back as a model fragment and confirm before moving on. Close the phase with `sheets/01-workflow.html` — the workflow as you understood it, as a diagram + storyboard, with one "is this right?" decision. Misunderstandings are cheapest to fix here.

**→ Dossier:** capture the lab's described workflow (in their own words) + the PROV/ISA mapping in `01-workflow.md`; on a refine, append so it matures rather than overwriting.

### Phase 2 — Derive the entity model

From the workflow, decide for each thing the lab named whether it becomes a **resource** (item, from a resource template) or an **experiment** (from an experiment template) — apply the resource-vs-experiment rule in `references/eln-modeling-patterns.md` §3a and show the reasoning per thing. Then propose (names from Phase 1 — do not assume synthesis): resource templates for the *reusable procedure* and *Method/SOP* (versioned per P9), the *central-artifact passport* (a Sample/Batch, a device/specimen, or a dataset/model), *Instrument*, optionally a *Study/Campaign* aggregation resource (G9); experiment templates and **experiment categories** named for the lab's own activity verbs (e.g. Synthesis/Characterisation; Fabrication/Measurement/Calibration; Specimen-prep/Test/Simulation; Culture/Assay/Imaging; Run/Processing/Analysis); resource categories on 6.x; and the **status** lifecycle.

Present it as `sheets/02-entity-model.html`: overview diagram, decisions (passport yes/no, protocol versioning, aggregation, borderline resource-vs-experiment cases), storyboard, table of what would be created. Get approval.

**→ Dossier:** record the approved model in `02-entity-model.md`; log the choice and any rejected alternatives in `decisions.md`.

### Phase 3 — Design the extra-field schema per type

For each template, design the `metadata.extra_fields` JSON: grouped fields (Identity / Provenance / Canonical pointers / Characterisation cache / Notes per P2), correct `type`s (entity-link types for provenance pointers per P3; `users` for people; `select` bound to controlled vocab per P4), `units`, `description`s, a `schema_version` field (P5), and which fields are **required** (you enforce this — P6; choose them from the lab's minimum-information standard, R1.3). Reuse field names that already exist in the instance (`elab_list_extra_field_names`) where they mean the same thing — renaming later is not possible (G13).

Present one sheet per type (`sheets/03-fields-<type>.html`): field table (name, kind, unit, required?, why), decisions on the required set and controlled lists, a note that field names are permanent. Keep the ready-to-PATCH JSON in the dossier, not in the sheet.

**→ Dossier:** save the per-type JSON + required-field sets to `03-field-schema.md`.

### Phase 4 — Design tag namespaces & typed-link conventions

Define the lab's controlled tag vocabulary (P1) — the namespace set, allowed values, the `canonical:`/`batch:` binding rule, the freeform-deduplication rule (map every existing freeform tag from Phase 0 to keep / rename-to-namespaced / retire). Define the typed-link conventions (P3): which relation is a named extra-field link vs. a `rel:` tag vs. an aggregation resource, each mapped to a PROV term. Document the canonical/passport pattern concretely for *this* lab.

Present as `sheets/04-tags-links.html`: tag registry table with examples, link diagram, decisions on namespaces and on the dedup mapping.

**→ Dossier:** save the namespaces + typed-link conventions to `04-vocabulary-links.md`.

### Phase 5 — FAIR / ALCOA conformance pass

Walk the proposed schema against the 15 sub-principles and ALCOA+: every required-field choice cites a principle; confirm the no-overwrite + lock+timestamp policy, attribution (personal logins), license/export field, tombstone (soft-delete) rule, template target permissions (who can read/write entries created from each template), and repository-handoff step. Surface any residual gap explicitly — honesty over completeness.

Present as `sheets/05-conformance.html`: scorecard table in plain language, residual gaps as notes, decisions only where a lab policy is needed (retention, unlock authority, licence).

**→ Dossier:** record the conformance pass + residual gaps in `05-conformance.md`.

### Phase 6 — Emit outputs (on approval)

1. **Generate `labschema-<lab>`** — instantiate the template at `<plugin>/skills/labschema/SKILL.md` (replace every `{{ }}` placeholder) into the **lab's project skills**: `./.claude/skills/labschema-<lab>/SKILL.md` in the lab's working directory (shared with colleagues if that directory is a shared repo), or `~/.claude/skills/labschema-<lab>/` for a single user — ask which with `AskUserQuestion`. **Never** write it into the plugin's install directory: plugin updates replace that directory and the contract would be lost. Include the **Design dossier** pointer so the working agent can consult the rationale.
2. **Generate the human blueprint** — `blueprint.md` (prose) and `sheets/blueprint.html` (visual, no decisions) in the dossier, for the researchers. Structure: (1) what we set up and why — one paragraph; (2) overview diagram; (3) "a normal week" storyboard per role; (4) the tag registry with examples; (5) per template: what it is for, its required fields; (6) rules of thumb (never rename a field, correct don't overwrite, lock + timestamp for defensible records); (7) what elabFTW cannot do and how the assistant compensates; (8) how to ask the assistant for common tasks ("start a synthesis from recipe v2", "show everything of batch 42").
3. **`--seed` only:** present `sheets/06-seed-plan.html` — every object to be created (kind, name, team) and every migration mapping — and ask go / sandbox team first / not yet with `AskUserQuestion`. Recommend a sandbox/test team first when the instance has real data. After explicit approval, create the templates / categories / status / tag vocabulary from the approved schema (additive only; never delete; verify each write native). Stamp `schema_version`. Log every created object with its id in `06-seed-log.md`.
4. **Migrate existing entries (optional, separate approval):** retag existing entries per the Phase 4 dedup mapping and backfill `schema_version` — strictly additive (add the structured tag; retire the freeform one only after the add is verified, never in the same burst — see the tag hazard in `elabftw-mcp`), in small batches, each batch logged in `06-seed-log.md`.
5. **Finalise the dossier:** mark every phase complete in `README.md`, stamp `schema_version`, and append a `decisions.md` entry summarising this run.

### Phase 7 — Wire CLAUDE.md (offer; additive only)

Plugins don't auto-edit the user's CLAUDE.md, and the working agent needs to know to load `elabftw-mcp` + `labschema-<lab>` whenever the elabftw MCP is touched. Offer — never enforce — to wire this in. **Never modify content outside the markers.** Always show the diff first; the user owns their CLAUDE.md.

1. Locate the target file. Prefer `CLAUDE.md` in the cwd. If absent, ask (`AskUserQuestion`): in a parent dir? a workspace root? a global `~/.claude/CLAUDE.md`? Don't guess.
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

To refine the schema with the researchers, run **`/elabftw-schema-init`** (use Opus 5.5 or newer). Normative refs (FAIR, ALCOA+, DCC, PROV, the elabFTW gap→pattern catalogue) live under the plugin's `references/`.
<!-- END elabftw-rdm -->
````

## Output contract

The generated `labschema-<lab>` MUST contain, fully resolved (no placeholders): lab identity + team mapping; the entity-model table with PROV/ISA mapping; per-type extra-field JSON; the protocol/versioning convention; the tag-namespace registry (controlled vocabulary + rules); the typed-link/provenance conventions; the canonical/passport pattern for this lab's central artifact (or an explicit note that the lab has none); the everyday recipes; the validation rules the agent must enforce; the integrity policy; and a FAIR/ALCOA conformance note. It must be specific enough that a fresh agent, loading only `elabftw-mcp` + `labschema-<lab>`, works the instance correctly with no further questions.

The run also maintains the **workshop dossier** in the lab's workspace (`./elabftw-rdm-<lab>/`): the per-phase records, the sheets, an append-only `decisions.md`, the seed log, and the manifest — so later runs resume with full context and the design's history is auditable. The dossier is the lab's private data; never write it into this plugin's directory.

## Refinement mode

Re-running on an initialised lab is a **refine** pass: **load the dossier first** (`README.md` + `01-workflow.md` + `decisions.md`) and the existing `labschema-<lab>`, recap where the design left off, survey drift in the live instance (Phase 0 — entries without `schema_version`, out-of-vocabulary tags, fields under new spellings), then propose a *diff* as a sheet (new types/fields/namespaces, deprecations — never rename, per P6/G13; bump `schema_version`). Apply on approval, and **append** the changes to the dossier (new `decisions.md` entries; grow `01-workflow.md`) rather than overwriting — the dossier is the longitudinal record of how the schema matured.
