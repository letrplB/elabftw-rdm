# ELN Modeling Patterns — elabFTW gaps and the workarounds

The design core. `elabftw-schema-init` leans on this to turn a lab's workflow into a schema that *works despite* elabFTW's structural limits. `elabftw-mcp` points here for the "why". Grounded in `fair-principles.md`, `data-integrity-alcoa.md`, and the elabFTW source (`src/sql/structure.sql`, API v2 OpenAPI).

---

## 1. What elabFTW gives you (the expressive surface)

The agent has exactly these levers — nothing else:

- **`items_types` (resource templates) + `experiments_templates`** as schema carriers (a body + a `metadata` JSON skeleton + steps, tags, links, permissions), instantiated server-side on create.
- **`metadata.extra_fields`** JSON on every entity — including the entity-link field types (`experiments`, `items`, `users`, `compounds`) that auto-create real link rows when set.
- **categories** and **status** — one flat, single-level, colored label axis each, per entity kind, per team (elabFTW ≥ 6.0: resource categories are separate from resource templates).
- **tags** — flat strings, team-scoped, no namespacing.
- **untyped links** — bare `(id, id)` edges, no relation type/role/direction/metadata.
- **compounds** — a fixed-schema substance registry (identifiers + GHS booleans; no lot/quantity/location).
- **storage_units** — the only native tree (physical storage), with `containers2*` carrying `qty_stored` + `qty_unit`.
- **integrity primitives** — soft-delete (`state`), lock, body-only revisions, changelog, RFC3161/Bloxberg timestamp, digital signature.

## 2. What elabFTW does NOT do — the gap catalogue

| # | Gap | Consequence for schema design |
|---|-----|-------------------------------|
| G1 | **No cross-entity rollups / aggregation / views** | No "all analyses of this batch", no "sum of masses". Must be synthesized by the agent via search/API. → *pattern P1, P2* |
| G2 | **No computed / derived / formula fields** | Extra fields are inert. No auto-population, no inter-field validation. → agent computes & writes |
| G3 | **No typed/semantic relations** | Links are unlabeled edges. No "derived-from"/"consumed-by". `created_from_id` records *duplication* only, not derivation. → *pattern P3* |
| G4 | **No schema enforcement on bodies** | `body` is free HTML/Markdown; `steps` a flat checklist. → structure lives in extra-fields + body conventions |
| G5 | **`required` is not enforced; metadata is an unvalidated blob** | elabFTW accepts empty "required" fields, missing links, invalid shapes. → **the agent is the validation layer** (*pattern P6*) |
| G6 | **items_type schema is copy-on-create, no propagation** | Editing a type does not update existing instances; each carries a frozen divergent copy. → *pattern P5 (schema_version)* |
| G7 | **Tags are unstructured strings** | No native key:value, hierarchy, vocabulary control. → *pattern P4* |
| G8 | **No native material/batch/sample passport** | `compounds` is substance-level (no lot/qty/location). → *pattern P2* |
| G9 | **No entity containers / logical aggregation** | Only `storage_units` form a tree. No "study contains experiments" aggregate. → model as an edge/index entity (*P3*) |
| G10 | **Coarse permissions** | 5-level base + allow-lists; no field-level ACL. → don't rely on field-level secrecy |
| G11 | **Revisions snapshot `body` only** | metadata/status/links/tags changes are logged but **not restorable**. → *pattern P7* |
| G12 | **No referential integrity on metadata links** | Deleting a target leaves dangling ids in another entity's metadata. → agent re-validates links |
| G13 | **Field identity = the JSON object key (the display name)** | Renaming an extra field silently orphans `extrafield:` searches, templates, integrations. → **never rename a field key** (*P6*) |
| G14 | **One flat extra-field namespace per entity** | Groups are visual only; two fields can't share a name across groups. → globally unique field names per type |
| G15 | **Everything is per-team; no org-wide schema** | items_types/templates/categories/status/tags duplicated per team, manually kept in sync. → *pattern P8* |
| G16 | **No protocol versioning** | An edited procedure keeps only its body text in revisions; runs that followed the old version now point at a procedure that no longer says what they did. → *pattern P9* |

## 3. The workaround patterns

### 3a — Resource or experiment? (the first modelling decision)

Every thing the lab names in the interview becomes exactly one of: a **resource** (`items`, from a resource template), an **experiment** (`experiments`, from an experiment template), a **compound**, an **attachment**, a **field**, or a **tag**. Getting this wrong is the most expensive mistake — it cannot be fixed later without re-creating entries.

- **Resource** if any holds: it persists beyond the activity that made it; more than one experiment will refer to it; it has an identity people ask about (lot number, serial, dataset DOI); it is the subject of "show me everything about X"; it is stored, located, used up.
- **Experiment** if it is an activity with a date, an operator and an outcome — repeating it would be a *new* entry.
- Two quick tests: *"Will another entry ever link **to** it?"* → resource. *"Did it **happen** on a day?"* → experiment.

| Thing | Usually becomes | Why |
|-------|-----------------|-----|
| Recipe / route / fab recipe / pipeline | resource, versioned (P9) | reused by many runs |
| SOP / measurement method | resource, versioned (P9) | cited by runs |
| Synthesis / fabrication / culture / simulation run | experiment | dated activity |
| Measurement / analysis | experiment, linked to what it measured | dated activity |
| Sample / batch / device / specimen | resource — the passport (P2) | persists, measured many times |
| Chemical substance | compound | registry with identifiers + hazards |
| A specific bottle / lot of a reagent | resource ("Reagent lot") only if lot traceability matters | compounds carry no lot/qty/location |
| Instrument | resource | linked per run for attribution |
| Calibration / maintenance | experiment linked to the instrument | dated activity |
| Raw data file | attachment on the experiment that produced it | stays with its origin (ALCOA-Original) |
| Processed dataset / trained model | resource if reused or published; else attachment | needs its own identity only when cited |
| A single result value | field on the analysis; cached on the passport (P2) | the analysis is the source of truth |
| Project / campaign | `project:` tag; additionally a resource only if it needs its own page (aims, members, member links — P3 option 3) | tags list, pages describe |
| Idea / meeting note | experiment (category `Note`) — or not in the ELN | no lineage value |

Anti-patterns: one experiment per sample (the sample becomes unlinkable); one resource per measurement (floods the inventory); spending the single category axis on projects (it is needed for activity type — projects go to tags).

### P1 — Structured-tag pseudo-namespace (the project/material axis elabFTW lacks)

elabFTW has no project/program dimension and no aggregation (G1, G7). Encode it as **literal `key:value` tag strings**, parsed client-side, applied as a controlled vocabulary the agent enforces (G5).

Recommended base namespaces (a lab adapts the values):

| Namespace | Purpose | Example |
|-----------|---------|---------|
| `branch:` | research program / group | `branch:catalysis` |
| `project:` | project axis (replaces freeform project codes) | `project:screening` |
| `topic:` | cross-cutting research theme | `topic:support-effect` |
| `method:` | analytical / synthetic / measurement method | `method:xrd`, `method:hplc`, `method:rna-seq`, `method:tensile` |
| `canonical:` | binds an analysis to *the* current result for a batch; flip to `canonical:superseded` on re-run | `canonical:LOT-2026-042` |
| `batch:` | the material-batch axis | `batch:LOT-2026-042` |
| `status:` | lifecycle beyond the single native status, when needed | `status:archived-raw` |
| `contract:` | marks the lab's data-contract resource (one per schema version) | `contract:acme-cat` |
| `protocol:` | protocol family, to list all runs of a procedure across its versions (P9) | `protocol:sop-dls` |

Why tags not extra-fields for these axes: tags are queryable lab-wide (`tag:"project:screening"`), reversible without schema migration, and visible without following links. Rules: lowercase-hyphenated values; one value per namespace per entity unless the axis is genuinely multi-valued; **drop the freeform duplicate** when a structured tag covers the concept (`method:xrd` ⟹ remove bare `XRD`). Where work is run- or dataset-centric rather than material-centric, the `canonical:`/`batch:` axis can bind to a device, dataset, or campaign id instead of a material lot.

> ⚠️ **Operational hazard.** Some elabFTW MCP `remove_tag` paths do a read-modify-write of the whole tag array and can wipe an entity's entire tag set if a tag *add* on the same entity is still settling. Prefer pure additive tagging; never interleave add+remove on the same entity in one burst; verify with a native-view re-read. (See `elabftw-mcp` operating rules.)

### P2 — The artifact passport (compensates G1, G8)

Any central artifact whose downstream results need aggregating (a physical sample/batch, a device/specimen, a dataset, a trained model) gets its **own `items` record** — a passport — to compensate for the missing rollup view (G1):

- **Identity** group: what the artifact *is* — e.g. (chemistry) route, surface chemistry, form; (materials) grade/designation, geometry; (data) source, schema, record count.
- **Provenance** group: creation date, operator/agent, link to the producing run (PROV `wasGeneratedBy`) — e.g. `synthesis_date`/`synthesis_operator`, or `acquired_date`/`generated_by_run`.
- **Canonical-analysis pointers** group: `canonical_<method>_analysis` fields of extra-field type `experiments`, each pointing at the source-of-truth analysis.
- **Characterisation cache** group: headline scalars **with units**, explicitly documented as a *cache* mirrored from the canonical analysis (the analysis is the source of truth; the cache exists only because G1 means there is no rollup view) — e.g. `BET_m2_g`, `d_H_nm` (catalysis); `yield_strength_MPa`, `hardness_HV` (materials); `n_records`, `auc` (data).
- Tag `batch:<id>` (or a `device:`/`dataset:` id); analyses tagged `canonical:<id>` + `method:<m>`.

This yields a one-hop view of the artifact elabFTW does not provide natively.

### P3 — Typed relations via convention (compensates G3, G9)

A link is an unlabeled edge. Encode the PROV relation out-of-band, in priority order:

1. **Directional extra-field link** with a name that *is* the relation: `produced_by_run` (type `experiments`), `used_reagent` (type `items`/`compounds`), `derived_from_sample` (type `items`). The field name carries the semantics; setting it also creates the real link row.
2. **Tag the relation** when a field is impractical: `rel:aliquot-of:<id>`.
3. **Edge / index items_type** for aggregation (the "study contains experiments" case G9): a thin `Study`/`Campaign` items_type whose extra-fields are typed link arrays to members.

Map every modelled relation to a PROV term (`wasGeneratedBy`, `used`, `wasDerivedFrom`, `wasAttributedTo`) so lineage is explicit and queryable — satisfies FAIR-I3 / R1.2.

### P4 — Controlled vocabularies bound to fields (FAIR-I2)

Prefer `select`/`radio` extra-fields with an explicit `options` list over free `text`. Bind to community vocabularies where one exists: units → the field `units`/`unit` mechanism with a QUDT/OM-aligned list, persons → `users` link type / ORCID, instruments → an `Instrument` items_type linked rather than typed by name, and domain registries — chemicals → elabFTW **compounds** (PubChem CID / ChEBI / CAS already carried); biological resources → an items_type carrying RRID / Addgene / GenBank ids (the `compounds` table is chemical-only); test methods / materials → ASTM/ISO designations or grade codes. Where elabFTW has no native registry, a controlled `select` the agent enforces.

### P5 — `schema_version` field (compensates G6)

Every items_type metadata template carries a `schema_version` extra-field (e.g. `"sample/v3"`). Because edits don't propagate, the agent: (a) stamps the version on create, (b) on encountering an instance, reads `schema_version` and reconciles against the current type spec from `labschema`, (c) records drift and offers a migration patch. There is no native migration — this is the only handle.

### P6 — The agent is the validation & enforcement layer (compensates G5, G12–G14)

Since elabFTW validates nothing: the per-lab `labschema` declares required fields, allowed vocab, units, link cardinality, and naming rules; the agent enforces them on every write and audits on every read. **Never rename an extra-field key** (G13) — names are the only stable identifier; deprecate-and-add instead. Keep field names globally unique per type (G14). Re-validate metadata links on read (G12).

### P7 — Integrity = lock **and** timestamp; snapshot metadata externally (compensates G11)

For records that must be defensible: `elab_lock` **and** `elab_timestamp` (RFC3161; both behind the MCP server's `ELABFTW_ALLOW_DESTRUCTIVE` gate) — lock alone is reversible, timestamp alone doesn't freeze the live row. Because revisions cover only the body (G11), any integrity-critical metadata must also be serialized into the body or an external snapshot at lock time, or it is not restorable. No-overwrite is the backbone (see `data-integrity-alcoa.md`).

### P8 — One canonical schema, replicated per team (compensates G15)

`labschema` is the single source of truth for the schema, published twice: as the agent's skill and as a locked `Lab data contract` resource in the instance (tag `contract:<lab>`), so people and non-Claude-Code clients read the same contract. Where elabFTW can enforce part of it natively, use that too: team admins switch off user tag creation and user template creation, and templates lock the permissions of derived entries. Per-team elabFTW objects (items_types, categories, status, tags) are *projections* the agent creates and keeps in sync from `labschema`, with a `schema_version` to detect divergence (P5).

### P9 — Versioned protocols (compensates G16, G6, G11)

A repeatable procedure has three possible homes in elabFTW; use the first two together and never the third:

1. **Protocol resource** — the citable procedure itself, one resource per *version*. Fields: `protocol_family` (stable id, e.g. `SOP-DLS`), `version` (`v3`), `valid_from`, `owner` (`users`), `supersedes` (`items` link → previous version). Status: `draft → active → superseded`. A change is a **new resource** linked `supersedes` → old, and the old one's status flips to `superseded` — never an in-place edit of an active protocol (revisions keep body text only, so changed parameter fields would be lost, G11; and past runs would point at a procedure that no longer says what they followed).
2. **Experiment template per protocol family** — pre-fills the run's steps, fields and `method:` tag, and carries a `follows_protocol` field (type `items`) defaulting to the active version. When a new version becomes active, update the template's default; existing runs keep their link to the version they followed — exactly right, since templates don't propagate (G6).
3. ~~Copying the procedure text into every run's body~~ — unversioned, unlinkable, diverges silently.

Each run links the exact version via `follows_protocol` (PROV `used`, the plan) and records **deviations** in a `deviations` field (required; "none" is a valid value) — deviations never go back into the protocol. Optional tag `protocol:<family>` lists all runs of a family across versions. When systematic parameter variation is the point (DoE, screening), the parameters are fields of the run, not new protocol versions.

## 4. ISA framing (structuring the model)

Map the lab's work onto **Investigation → Study → Assay**: Investigation ≈ project/space (`project:` tag / a `Campaign` edge type), Study ≈ the experiment (synthesis/run), Assay ≈ the linked measurement record. ISA's material→process→data DAG is the same chain as PROV and as the passport pattern. Use the domain's minimum-information checklist (MIBBI-style) to decide which extra-fields are **required** (FAIR-R1.3).

## 5. The 15 practical rules (fused FAIR + ALCOA + PROV + RDM)

Each: *rule — why — how it shows up*.

1. **One entity, one persistent ID, forever.** Findability+attribution (F1). → `elabid`+`custom_id`; never reuse.
2. **Record at the bench, at the time.** Contemporaneity can't be retrofitted (ALCOA-C). → create before/while doing; server timestamps; lock when done.
3. **Never overwrite — correct visibly.** Originality + audit trail (ALCOA-O). → lock + revision; corrections tracked with a reason.
4. **Attribute to a real authenticated person.** Accountability (ALCOA-A, R1.2). → personal logins; instrument data attributed to the instrument.
5. **Structured fields beat prose.** Searchable, comparable, machine-actionable (F2/I1). → fill the extra-fields schema, not just the body.
6. **Use the community vocabulary.** Interoperability (I2/R1.3). → `select`/compounds/ORCID/units, not free text.
7. **Link entities with typed relations.** Lineage must be queryable (I3, PROV). → P3 conventions.
8. **Capture full provenance: who/when/from-what/with-what.** Reuse needs it (R1.2). → reagent lots, instrument+settings, operator, parent links on every record.
9. **Keep raw data raw and dynamic.** A PDF is not the original (ALCOA-O). → attach native files + checksums alongside any rendering; never delete raw.
10. **Keep everything, including failures and excluded data.** Completeness (ALCOA-Complete). → failed runs/excluded points stay, with a reason.
11. **Record enough to reproduce without asking you.** Reusability test (R1). → conditions/params/units/calibration as fields.
12. **Templates encode the minimum-information checklist.** Standards must be structural (R1.3). → required fields per type; "complete" gated on them.
13. **Attach a license and an export when you share.** Reuse blocked by ambiguity (R1.1/I1). → `license` field + `.eln`/ISA/RO-Crate export on publication.
14. **The ELN is the start of the lifecycle, not the end.** Preservation belongs to a repository (DCC). → routine repository handoff.
15. **Metadata outlives the data.** Tombstones preserve findability (A2). → never hard-delete metadata with its data; retain what-it-was + why-withdrawn.
