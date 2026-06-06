---
name: elabftw-mcp
description: Operating manual for an AI agent working in an elabFTW ELN via the elabftw MCP server. The elabFTW data model (entities, metadata/extra_fields JSON, links, tags, compounds), the MCP tool surface, what elabFTW does NOT do natively, and the workaround patterns. Load whenever working with elabftw MCP tools. If a generated labschema-<lab> skill is present, load it alongside this — it is the binding contract.
version: "0.1"
---

# elabftw-mcp — agent operating manual

You are operating a real lab's notebook. Records here are legal/scientific evidence (see `references/data-integrity-alcoa.md`). Be precise, verify writes, never overwrite history.

**Composition.** This skill is generic and lab-agnostic. If a `labschema-<lab>` skill exists for the instance you are in, **it is the binding contract** — its required fields, vocabularies, tag namespaces and link conventions override anything generic here. Always load both. If none exists, the lab has not been initialised — recommend `/elabftw-schema-init`.

---

## 1. The elabFTW data model (just enough to reason)

Everything is **team-scoped** — every entity carries a `team`; pass `team` on every MCP call when the instance has more than one team. Four entity tables share an almost identical shape:

| Entity (`entityType`) | What it is |
|-----------------------|-----------|
| `experiments` | the lab-notebook entry — a run, an analysis, a note. Carries `category`, `status`. |
| `items` | typed inventory / resources — samples, batches, devices/specimens, chemicals, datasets, instruments, and reusable **recipes/procedures** (a synthesis route, a fab recipe, an assay protocol, a pipeline) and **protocol/SOP** methods. Categorised by an **items_type**. |
| `items_types` | the resource *category that owns a metadata schema* — the reusable extra-fields skeleton instances inherit. |
| `experiments_templates` | the same, for experiments. |

Shared columns: `id`, `team`, `title`, `date`, `body` (+ `content_type`: 1=HTML, 2=Markdown), `category`, `status`, `custom_id`, `rating`, `metadata` (JSON — the structured store), permissions (`canread*/canwrite*`), integrity (`locked`, `timestamped`, `state` soft-delete 1=normal/2=archived/3=deleted), `created_at/modified_at`.

Other model pieces: **categories** & **status** = flat single-level colored labels (no hierarchy, no metadata). **tags** = flat strings, team-scoped, no namespacing. **links** = bare untyped `(id,id)` edges. **compounds** = fixed-schema substance registry (name, formula, SMILES/InChI, CAS/PubChem/ChEBI, ~30 GHS booleans; *no* lot/qty/location). **storage_units** = the only native tree (physical storage) + `containers2*` carry `qty_stored`/`qty_unit`. **uploads** (content-addressed, `immutable=1` default), **comments**, **steps** (flat checklist), **revisions** (body-only snapshot), **changelog** (textual audit).

## 2. The `metadata` / `extra_fields` JSON (the only structured custom-data mechanism)

```json
{
  "extra_fields": {
    "Field Name": {            // the KEY is the field's identity — renaming = orphaning
      "type": "number",        // default "text"
      "value": "12",           // always a string, or array if allow_multi_values
      "unit": "nm", "units": ["nm","µm"],
      "description": "...","position": 3,"group_id": 2,
      "options": ["a","b"],    // select/radio
      "required": true,        // VISUAL ONLY — not enforced by elabFTW
      "readonly": false, "blank_value_on_duplicate": true
    }
  },
  "elabftw": {
    "display_main_text": true,
    "extra_fields_groups": [ { "id": 1, "name": "Identity" }, { "id": 2, "name": "Provenance" } ]
  }
}
```

Field `type`s: `text, number, select, radio, checkbox, date, datetime-local, time, email, url`, and the **entity-link types** `experiments`, `items`, `users`, `compounds` — setting one of these auto-creates a real link row; `value` holds the target id(s). Groups are **visual only** (a `group_id` → a label); field names must be globally unique per entity.

**items_types/templates are copy-on-create, NOT a live binding.** Editing a type's schema does **not** propagate to existing instances — each carries its own frozen copy. Carry a `schema_version` field and reconcile (pattern P5).

## 3. The MCP tool surface

Tools are deferred — load schemas with `ToolSearch` (`select:mcp__elabftw__elab_get,...`) before first use. Group by job:

- **Orient:** `elab_me`, `elab_info`, `elab_configured_teams`, `elab_list_teams`, `elab_list_team_users`, `elab_list_experiments_categories`, `elab_list_experiments_status`, `elab_list_items_types`, `elab_list_items_status`, `elab_list_tags`.
- **Find:** `elab_search` (per team; `extended` = the DSL: `tag:`, `category:`, `status:`, `rating:`, `date:`, `extrafield:`, boolean `and/or/not`, quote spaces), `elab_search_all_teams`, `elab_search_compounds`, `elab_search_pubchem`.
- **Read:** `elab_get` / `elab_get_bulk` (use `include:["links","steps","comments","attachments"]` to do one round-trip; `view:"native"` for round-trippable JSON, `format:"markdown"` for tables/links). `elab_list_links`, `elab_list_extra_fields`, `elab_get_revision`, `elab_list_revisions`, `elab_list_changelog` equivalents.
- **Write entities:** `elab_create_entity`, `elab_update_entity` (accepts `native:` round-trip shape), `elab_duplicate_entity`, `elab_export`.
- **Extra fields:** `elab_set_extra_field`, `elab_update_extra_field`, `elab_remove_extra_field`, `elab_set_extra_field_groups`, `elab_clone_extra_fields_schema`.
- **Tags:** `elab_add_tag` (auto-creates the tag globally if absent), `elab_remove_tag` (by tag id), `elab_create_tag`.
- **Links:** `elab_link_entities`, `elab_unlink_entities`.
- **Compounds:** `elab_create_compound`, `elab_create_compound_from_pubchem`, `elab_get_compound`, `elab_update_compound`.
- **Steps/comments/attachments:** `elab_add_step`, `elab_toggle_step`, `elab_add_comment`, `elab_download_attachment`, etc.

## 4. What elabFTW does NOT do — and what you must do instead

elabFTW **validates nothing**: `required` is cosmetic, metadata is an unchecked blob, links are untyped, tags are flat, there are no rollups/computed fields, schemas don't propagate, revisions cover only the body. **You are the schema, validation, provenance and aggregation layer.** The full gap catalogue and the workaround patterns (P1 structured-tag namespace, P2 artifact passport, P3 typed relations, P4 controlled vocab, P5 schema_version, P6 agent-as-validator, P7 lock+timestamp, P8 one-schema-per-team) are in `references/eln-modeling-patterns.md` — read it before doing schema work. The lab's concrete instantiation is in `labschema-<lab>`.

## 5. Operating rules (non-negotiable)

1. **Verify every write.** Re-read with `elab_get(view:"native")` after create/update/tag. MCP success messages have lied (see rule 4). Tag counts in `elab_list_tags` lag — trust per-entity reads.
2. **Never rename an extra-field key.** The key *is* the field identity; renaming orphans searches/templates/integrations. Deprecate-and-add instead.
3. **Never overwrite history.** No destructive edits to locked/timestamped records. Corrections are additive with a reason. For defensible records: `lock` **and** `timestamp` (lock alone reverses; timestamp alone doesn't freeze the row); metadata isn't in revisions, so snapshot integrity-critical metadata into the body too.
4. **Tag hazard:** some `elab_remove_tag` paths read-modify-write the whole tag array and can wipe an entity's entire tag set if a recent `elab_add_tag` on the same entity is still settling. Prefer pure additive tagging; never interleave add+remove on the same entity in one burst; re-read native to confirm.
5. **Respect the contract.** When `labschema-<lab>` is loaded, enforce its required fields, vocabularies, namespaces and typed-link conventions on every write; audit them on every read; report drift.
6. **Team scope always.** Pass `team` explicitly on multi-team instances; never assume the default.
7. **Soft-delete, never hard-delete metadata** (FAIR-A2 tombstone): set `state` rather than destroying the record.

When unsure whether an action is reversible on a real notebook, stop and ask the researcher.
