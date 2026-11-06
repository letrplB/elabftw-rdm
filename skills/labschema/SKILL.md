---
name: labschema
description: TEMPLATE for the per-lab structural contract. elabftw-schema-init instantiates this as a labschema-<lab> skill. When a generated labschema-<lab> exists for the instance you are working in, that skill (not this template) is the binding contract — load it alongside elabftw-mcp and enforce it on every read and write. This file is the template + a fully worked example; do not enforce the template itself.
version: "0.2"
---

# labschema — the per-lab structural contract (TEMPLATE)

> **This file is a template, not an active contract.** `elabftw-schema-init` copies it to the lab's project skills (`./.claude/skills/labschema-<lab>/SKILL.md`, or `~/.claude/skills/labschema-<lab>/` for a single user — never into the plugin's install directory, which plugin updates replace) and replaces every `{{placeholder}}` with the lab's approved design. The working agent loads the *generated* `labschema-<lab>` (alongside `elabftw-mcp`) and treats it as binding: elabFTW enforces nothing, so this contract is the only schema. A fully worked example (a fictional heterogeneous-catalysis lab) follows the template so the shape is concrete.

---

## TEMPLATE

```
---
name: labschema-{{lab_slug}}
description: Binding elabFTW data contract for {{lab_name}} (team {{team_id}}). Load alongside elabftw-mcp whenever working in this instance; enforce on every read and write.
version: "{{schema_version}}"
---

# labschema-{{lab_slug}} — binding contract

Authority: this overrides anything generic in elabftw-mcp. Patterns referenced (P1–P9) are defined in the elabftw-rdm plugin's references/eln-modeling-patterns.md.

## 1. Lab identity & scope
- Lab / group: {{lab_name}}
- elabFTW team id(s): {{team_id}}  — pass `team` on every MCP call
- schema_version: {{schema_version}}  — every entity created carries an extra-field `schema_version` = this string (P5)
- Design dossier: {{dossier_path}} (e.g. `./elabftw-rdm-{{lab_slug}}/`)  — the workshop record: described workflow, decisions, rationale, history. Consult for context; the contract below is what you enforce.

## 2. Entity model
| Role | elabFTW kind | template (id) / category | Reusable? | PROV / ISA |
|------|--------------|--------------------------|-----------|------------|
{{for each: role | experiments|items | template name (id) / category name | reusable/one-time/physical | PROV term / ISA level}}
Always create entries with `elab_create_entity(template=<id>)` using the ids above.

## 3. Extra-field schema per type
For each items_type / template, the exact metadata.extra_fields JSON (ready to PATCH), grouped, with type/unit/options/description, required flag, and schema_version. Field keys are FROZEN — never rename (G13/P6).
{{per type: the JSON block + the list of REQUIRED keys}}

## 3a. Protocols & repeatable procedures (P9)
- Protocol families: {{family id → current active version (resource id) → experiment template that follows it}}
- A change = new protocol resource, `supersedes` → previous; previous status → superseded; update the template's `follows_protocol` default. Never edit an active protocol in place.
- Every run: `follows_protocol` = exact version; `deviations` filled ("none" allowed).
{{If the lab has no repeatable procedures, state so and mark N/A.}}

## 4. Tag-namespace registry (controlled vocabulary, P1)
| Namespace | Meaning | Allowed values | Cardinality | Rule |
|-----------|---------|----------------|-------------|------|
{{branch:/project:/topic:/method:/canonical:/batch:/... with allowed values}}
Deduplication rule: {{when a structured tag covers a concept, the freeform duplicate is removed}}
canonical binding: {{how canonical:<batch> + method:<m> bind an analysis; supersession = canonical:superseded}}

## 5. Typed-link & provenance conventions (P3 → PROV)
| Relation (PROV) | Encoding | From → To |
|-----------------|----------|-----------|
{{wasGeneratedBy/used/wasDerivedFrom/wasAttributedTo: named extra-field link | rel: tag | edge items_type}}

## 6. Canonical / passport pattern for this lab (P2)
{{what the central-artifact passport must show — a Sample/Batch, a device/specimen, or a dataset/model record; which canonical_<method>_analysis pointer fields; the cache-vs-source-of-truth rule. If the lab has no passport-style central artifact, state so and mark this section N/A.}}

## 6a. Everyday recipes (how the common tasks are done here)
{{3–8 recipes in the lab's words, each as numbered tool-level steps, e.g. "Start a synthesis: 1. elab_create_entity(experiments, template=<id>) … 2. link recipe version … 3. tags … 4. create the batch passport from template <id> … 5. set produced_by_run … 6. verify native."}}

## 7. Validation rules the agent MUST enforce (P6 — elabFTW enforces none)
- Required fields present & non-empty per type before an entry is "complete": {{...}}
- Controlled-vocab fields only take allowed values: {{...}}
- Numeric fields carry the declared unit: {{...}}
- Every {{producing run}} links its {{products}}; every {{analysis}} links its {{input artifact (sample / dataset / run)}} + sets the reciprocal pointer.
- Never rename a field key; deprecate-and-add. Keep names globally unique per type.
- Re-validate metadata links on read (dangling ids, G12).

## 8. Integrity policy (ALCOA, P7)
- Defensible records: {{which}} → lock AND timestamp; snapshot integrity-critical metadata into the body at lock time.
- No overwrite — corrections additive with a reason. Personal logins only.
- Retention: {{period}}. Unlock authority: {{who}}. Tombstone (soft-delete, never hard-delete metadata) — FAIR-A2.

## 9. FAIR / ALCOA conformance notes
{{per principle that drove a required field or rule, one line: F2→…, I2→…, I3→…, R1.2→…, R1.3→…, ALCOA-O→…}}
{{Known residual gaps stated honestly.}}
```

---

## WORKED EXAMPLE — `labschema-acme-cat` (fictional heterogeneous-catalysis lab)

A fully-resolved labschema for a fictional catalysis lab — invented for illustration. Shows the template's shape on a common chemistry archetype (reusable recipes & methods, one-time synthesis runs, physical sample passports, canonical characterisation analyses). Substitute your lab's real types, fields, vocabulary, and team id on instantiation. The template shape is domain-general — a physics device passport, an engineering specimen-and-test, a biology cell-line-and-assay, or a computational dataset-and-run all instantiate the same structure; only the entities, activities, methods and vocabulary change.

**1. Identity.** Lab: Acme Catalysis Group. Team id: `<team_id>`. `schema_version: "acme-cat/v1"`.

**2. Entity model.**

| Role | Kind | Type / category | Reusable? | PROV / ISA |
|------|------|-----------------|-----------|------------|
| Recipe | `items` | template `Recipe` (versioned, P9) | reusable | the plan a run `used` (ISA: protocol) |
| Method | `items` | template `Protocol` (versioned, P9) | reusable | measurement SOP |
| Run | `experiments` | category `Synthesis` | one-time | Activity (ISA: Study) — `used` recipe+reagents, `wasGeneratedBy` ← samples |
| Sample/batch | `items` | `Sample` | physical | Entity passport (ISA: material node) |
| Analysis | `experiments` | category `Characterisation` | one-time | Activity — `used` sample, produces result |
| Instrument | `items` | `Instrument` | reusable | Agent/Entity for attribution |

**3. Extra-field schema (excerpt — `Sample` passport).**

```json
{ "extra_fields": {
  "schema_version":      {"type":"text","value":"acme-cat/v1","group_id":1,"readonly":true},
  "synthesis_route":     {"type":"select","options":["sol_gel","co_precipitation","impregnation","hydrothermal","other"],"group_id":1},
  "composition":         {"type":"text","group_id":1,"description":"nominal composition, e.g. 5 wt% Pt/Al2O3"},
  "synthesis_date":      {"type":"date","group_id":2},
  "synthesis_operator":  {"type":"users","group_id":2,"description":"operator (ALCOA-Attributable) — a users link, not free text"},
  "produced_by_run":     {"type":"experiments","group_id":2,"description":"PROV wasGeneratedBy → the synthesis run"},
  "canonical_xrd_analysis": {"type":"experiments","group_id":3,"description":"source of truth for crystalline_phase, crystallite_size_nm"},
  "canonical_bet_analysis": {"type":"experiments","group_id":3,"description":"source of truth for BET_m2_g, pore_*"},
  "BET_m2_g":            {"type":"number","unit":"m²/g","units":["m²/g"],"group_id":4,"description":"CACHE — mirrored from canonical BET analysis"},
  "crystallite_size_nm": {"type":"number","unit":"nm","units":["nm"],"group_id":4,"description":"CACHE — mirrored from canonical XRD analysis"},
  "notes":               {"type":"text","group_id":5}
},
  "elabftw": {"extra_fields_groups":[
    {"id":1,"name":"Identity"},{"id":2,"name":"Provenance"},
    {"id":3,"name":"Canonical analyses (pointers)"},
    {"id":4,"name":"Characterisation cache"},{"id":5,"name":"Notes"}]}}
```
Required on a *complete* sample: `schema_version, synthesis_route, composition, synthesis_date, synthesis_operator, produced_by_run`.

**3a. Protocols.** Families `RCP-PtAl` (Recipe) and `SOP-XRD`, `SOP-BET` (Protocol), one resource per version, `supersedes` chain, status draft/active/superseded. Experiment templates `Synthesis`, `XRD`, `BET` carry `follows_protocol` (type `items`) defaulting to the active version and a required `deviations` field.

**4. Tag-namespace registry.**

| Namespace | Meaning | Allowed values | Card. | Rule |
|-----------|---------|----------------|-------|------|
| `branch:` | program | `catalysis` | 1 | always present |
| `project:` | project | `screening`, `aging`, … | 1 | replaces freeform project codes |
| `topic:` | theme | `support-effect`, `metal-loading`, … | 1+ | |
| `method:` | technique | `xrd`, `bet`, `tem`, `tpd`, `xps` | 1+ on analyses | drop freeform `XRD`/`BET` |
| `canonical:` | batch binding | `<batch-id>` or `superseded` | 1 on analyses | re-run ⟹ old → `canonical:superseded` |
| `batch:` | material axis | `<batch-id>` (e.g. `LOT-2026-042`) | 1 on samples | |

Dedup rule: a structured tag retires its freeform synonym. Canonical binding: an analysis tagged `canonical:<batch>` + `method:<m>`; the sample's `canonical_<m>_analysis` pointer + `batch:<id>` close the loop (P2).

**5. Typed links.** `wasGeneratedBy`: sample.`produced_by_run` (extra-field link). `used`: run links recipe+reagent compounds. `wasDerivedFrom`: `derived_from_sample` link. `wasAttributedTo`: `synthesis_operator` / `users` link. Generic untyped links are insufficient — always use the named field (P3).

**6. Passport pattern.** A `Sample` shows in one hop: identity, provenance (run + operator + date), canonical-analysis pointers, and the characterisation **cache** (headline scalars, units, explicitly "mirrored from the canonical analysis — source of truth is that experiment"). G1 means there is no native rollup; the passport is the workaround.

**6a. Everyday recipe — start a synthesis.** 1. `elab_create_entity(experiments, template=<Synthesis id>, title=…)`. 2. `elab_set_extra_field(follows_protocol = <active Recipe id>)`. 3. tags `branch:catalysis`, `project:<p>`. 4. `elab_create_entity(items, template=<Sample id>)`, `custom_id` = batch id, tag `batch:<id>`. 5. on the sample: `produced_by_run` = the run, `synthesis_operator`, `synthesis_date`, `schema_version`. 6. re-read both natively and check §7.

**7. Enforced validation.** Required fields per type before "complete"; `select` fields reject out-of-vocab values; numeric fields carry units; every Synthesis run links its Sample(s); every Analysis links its Sample and the Sample's `canonical_<m>_analysis` is set; field keys never renamed; metadata links re-checked on read.

**8. Integrity.** Published/defensible analyses: lock **and** RFC3161 timestamp; serialize the analysis JSON into the body at lock time (revisions are body-only, G11). No overwrite; personal logins. Retention per institute policy; unlock = team admin only. Soft-delete only (A2 tombstone).

**9. Conformance.** F1→`elabid`+`custom_id`(batch-id); F2→required extra-fields; I2→`compounds`+ORCID+`units`; I3→typed `produced_by_run`/`canonical_*` links; R1.2→Provenance group; R1.3→required set = lab minimum-information; ALCOA-O→lock+timestamp+no-overwrite. Residual gap: cross-batch rollups still require agent-side aggregation (G1) — accepted.
