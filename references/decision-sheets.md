# Decision sheets & visuals — making the design digestible

How `elabftw-schema-init` puts large decisions and proposed structures in front of researchers. The premise: most researchers avoid ELN structuring because the cognitive load is high and the payoff is abstract. Every design step must therefore arrive as **a picture, a few pre-answered decisions, and a concrete "what changes for you" example** — never as a wall of tables and pattern codes.

The renderer is `assets/decision-sheet.html` (self-contained, no network, works from `file://`, light/dark, printable). You never write HTML or CSS for a sheet: copy the template and replace **only** the JSON inside `<script type="application/json" id="sheet-data">`.

---

## 1. Which channel for which moment

| Situation | Channel |
|-----------|---------|
| 1–4 discrete, independent choices that are understandable from one sentence each | **`AskUserQuestion`** |
| More than 4 decisions, decisions that depend on each other, or anything that needs a picture to be understood (entity model, link structure, tag map, field schema, seed plan) | **Decision sheet** |
| The lab's story — "walk me through one experiment" | **Open dialogue** — never force narrative into options |
| A yes/no go-ahead for an irreversible step (seeding, migrating, CLAUDE.md edit) | **`AskUserQuestion`**, after the plan was shown (sheet or diff) |

A sheet and `AskUserQuestion` combine well: open the sheet for the picture and the details, then — for a small sheet — ask the 1–3 headline decisions with `AskUserQuestion` so the researcher can answer without leaving the chat. For a large sheet, let them answer in the sheet and paste the export.

## 2. The sheet loop

1. **Write** `sheets/NN-<topic>.html` into the lab's dossier (`NN` = phase number, e.g. `02-entity-model.html`, `03-fields-sample.html`). Never into the plugin directory.
2. **Open** it for the researcher: `open <path>` (macOS) / `xdg-open <path>` (Linux); if that is unavailable, give the absolute path. In the Claude desktop app, a file inside the session folder can also be shown in the file pane.
3. **Say in one or two chat lines** what the sheet asks and how long it takes ("3 decisions, ~5 min; my recommendations are pre-marked — *Accept all recommendations* then *Copy my answers* is a valid answer").
4. **Wait** for the pasted export. It looks like:
   ```
   DECISION SHEET 02-entity-model — Acme Catalysis Group — answered 2026-11-06
   D1 Batch page: A "Yes — one passport per batch" (recommended)
   D2 Recipes: B "Edit the recipe in place" (NOT the recommendation)
      note: we never change recipes after the first month
   D3 Methods: OPEN — not decided
   ```
5. **Act on it:** every answer goes to `decisions.md` (dated, with the option text). A pick marked *NOT the recommendation* or a note means you learned something — reflect it back in one line and update your model of the lab. `OPEN` items are carried to the next sheet or asked via `AskUserQuestion`; never silently defaulted.
6. **Re-issue, don't edit in place:** when a sheet's content changes after feedback, write the next sheet (`02-entity-model-v2.html`) so the dossier keeps the history.

Answers live in the researcher's browser (`localStorage`, keyed by lab + sheet id), so they can close the sheet and come back. Two people answering the same sheet produce two exports — collect both and surface disagreements as a decision of their own.

## 3. Data shape

```jsonc
{
  "lab": "Acme Catalysis Group",          // shown in the header; part of the storage key
  "sheet": "02-entity-model",             // stable id — part of the storage key and the export
  "title": "How your work maps into elabFTW",   // a plain-language claim or question, no jargon
  "date": "2026-11-06",
  "phase": { "current": 3, "names": ["Survey", "Your workflow", "What becomes what", "Fields", "Tags & links", "Checks", "Hand-over"] },   // current = 1-based position in names (sheet 02 → step 3)
  "intro": "2–3 sentences: what this sheet is, how many decisions, how to answer.",   // **bold**, `code`, *italic* allowed
  "learned": ["What you understood about the lab — the basis of every recommendation."],
  "sections": [ /* any order; rendered top to bottom */ ]
}
```

Section types:

| `type` | Fields | Use for |
|--------|--------|---------|
| `diagram` | `title`, `caption`, `lanes: [{id,label}]`, `nodes: [{id,lane,label,sub?,note?,tone?:"accent"}]`, `edges: [{from,to,label,dashed?}]` | The structure picture. Lanes are columns (left → right reads as the workflow). `tone:"accent"` marks the central page (the passport). `note` = hover text. `dashed` = a pointer (e.g. "current result") rather than a production link. Parallel edges are fanned automatically. |
| `decision` | `id` (`D1`…), `header` (≤ 12 chars), `title` (the question), `why` (why it matters to *them*), `options: [{key,label,detail?,consequence?,recommended?}]`, `multi?`, `more?` (collapsed background) | Every choice. |
| `storyboard` | `title`, `steps: [{who, do, elab?}]` | "What a normal week looks like" — the single most effective device for making a structure concrete. Use real member names (from Phase 1) and their real activities. |
| `table` | `title`, `columns`, `rows`, `highlight?: [rowIdx]`, `caption?` | Field lists, tag registries, seed plans, conformance scorecards. Keep ≤ 12 rows; split otherwise. |
| `list` | `title`, `items` | Short enumerations. |
| `note` | `tone: info|warn|good|bad`, `title`, `body` | "Nothing is changed yet", risks, residual gaps. |

Diagram layout is deterministic: nodes stack inside their lane in the order given. 2–4 lanes and ≤ 5 nodes per lane stay legible; split bigger models into one overview diagram plus one diagram per area.

## 4. Writing rules (these matter more than the renderer)

- **Their words, not ours.** Use the lab's own names for things ("batch", "Anna's DLS runs"), learned in Phase 1. Pattern codes (P2, G13), PROV/ISA/FAIR acronyms and elabFTW internals (`items_types`, `extra_fields`) stay in the dossier and the contract. In a sheet, say "resource template", "experiment template", "field", "tag", "link" — and explain a term once, in a `note` or `more`, only where it is unavoidable.
- **Picture first.** Every sheet that proposes structure opens with a `diagram`; decisions come after it.
- **Few decisions per sheet.** 3–7. More than 7 → split by topic (one sheet per resource type in Phase 3).
- **Every decision is pre-answered.** Exactly one `recommended` option for single-choice (the recommended set for `multi`). The recommendation must follow from something in `learned` — if nothing you learned supports a recommendation, mark none and say in `why` that this one is genuinely theirs to call.
- **Consequences, not features.** Each option's `consequence` says what it costs or gains in daily work or later ("**Daily cost:** …", "**Later cost:** …", "**Risk:** …"), not how elabFTW implements it.
- **Show the week.** Each structural sheet carries a `storyboard` with 3–5 steps of real work under the proposal.
- **State what will be created** (a `table`) and that **nothing is changed yet** (a `note`) whenever the sheet precedes a write.
- **Honest gaps.** If elabFTW cannot do something the lab wants, say so plainly in a `warn` note with the workaround — no silent omissions.

## 5. Sheets per phase

| Phase | Sheet | Core sections |
|-------|-------|---------------|
| 0 Survey | `00-instance-survey.html` | `diagram` of what exists today (types, categories, how they are actually linked); `table` of top freeform tags/fields with counts; `note`s on inconsistencies found. No decisions, or only "is this picture right?". |
| 1 Workflow | `01-workflow.html` (optional) | `diagram` of the workflow as you understood it (inputs → activity → products → measurements) + `storyboard` in their words; one decision: "is this right?" with a notes field. Recommended after the interview, before Phase 2 — misunderstandings are cheapest here. |
| 2 Entities | `02-entity-model.html` | Overview `diagram` (resources / experiments / products), decisions on what becomes a resource vs. an experiment, passports, protocol versioning, aggregation; `storyboard`; `table` of what gets created. |
| 3 Fields | `03-fields-<type>.html` (one per type) | `table` of proposed fields (name, kind, unit, required?, why); decisions on required sets and controlled lists; `note` on fields that can never be renamed later. |
| 4 Tags & links | `04-tags-links.html` | `table` tag registry (namespace, meaning, allowed values, example); `diagram` of the link conventions; decisions on namespaces and the dedup of existing freeform tags. |
| 5 Checks | `05-conformance.html` | `table` scorecard (principle → how it is met → status); `note`s for residual gaps; decisions only where a gap needs a lab policy (retention, who may unlock, licence). |
| 6 Seed / migrate | `06-seed-plan.html` | `table` of every object to be created (kind, name, team), `table` of migration mappings (old tag → new tag, count); `note` "nothing changes until you approve"; one decision: go / go for a sandbox team first / not yet. |
| Final | `blueprint.html` | The human blueprint as a sheet without decisions: overview diagram, storyboard per role, tag registry, "how to ask the assistant for X". |
