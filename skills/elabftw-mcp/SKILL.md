---
name: elabftw-mcp
description: Operating manual for an AI agent working in an elabFTW ELN via the elabftw MCP server. The elabFTW data model (entities, templates vs. categories, metadata/extra_fields JSON, links, tags, compounds), the MCP tool surface and its write gates, what elabFTW does NOT do natively, the workaround patterns, and the hazards that silently lose data. Load whenever working with elabftw MCP tools. If a generated labschema-<lab> skill is present, load it alongside this — it is the binding contract.
version: "0.2"
---

# elabftw-mcp — agent operating manual

You are operating a real lab's notebook. Records here are legal/scientific evidence (see `references/data-integrity-alcoa.md`). Be precise, verify writes, never overwrite history.

**Composition.** This skill is generic and lab-agnostic. If a `labschema-<lab>` skill exists for the instance you are in, **it is the binding contract** — its required fields, vocabularies, tag namespaces, protocol conventions and link conventions override anything generic here. Always load both. If none exists, the lab has not been initialised — recommend `/elabftw-schema-init` (on Opus 5.5 or newer).

**Plugin paths.** `references/` sits at the plugin root, two levels above this skill's base directory.

---

## 1. The elabFTW data model (just enough to reason)

Everything is **team-scoped** — every entity carries a `team`; with several API keys configured, pass `team` on every MCP call. Four entity tables share an almost identical shape:

| Entity (`entityType`) | What it is |
|-----------------------|-----------|
| `experiments` | the lab-notebook entry — a dated activity: a run, an analysis, a note. Carries `category`, `status`. |
| `items` | resources — samples, batches, devices/specimens, chemicals, datasets, instruments, and reusable **recipes/procedures** and **protocol/SOP** methods. Created from a resource template; carries a resource category and status. |
| `items_types` | **resource templates** — the reusable body + extra-fields skeleton instances are created from. |
| `experiments_templates` | **experiment templates** — the same, for experiments. |

**Templates vs. categories (elabFTW ≥ 6.0).** A *template* (`items_types`, `experiments_templates`) carries a schema (body, extra fields, steps, tags, permissions). A *category* is a flat colored label (`elab_list_experiments_categories`, `elab_list_items_categories`). On 6.x resource categories are separate from resource templates — ids differ even when titles match. On 5.x the items_type doubled as the resource category. Check the version (`elab_info`) before reasoning about either.

Shared columns: `id`, `team`, `title`, `date`, `body` (+ `content_type`: 1=HTML, 2=Markdown), `category`, `status`, `custom_id`, `rating`, `metadata` (JSON — the structured store), permissions (`canread*/canwrite*`; on templates also `canread_target*/canwrite_target*` = permissions for entries created from them), integrity (`locked`, `timestamped`, `state` 1=normal / 2=archived / 3=deleted), `created_at/modified_at`.

Other model pieces: **status** = flat single-level labels. **tags** = flat strings, team-scoped, no namespacing. **links** = bare untyped `(id,id)` edges (to experiments, items, compounds). **compounds** = fixed-schema substance registry (name, formula, SMILES/InChI, CAS/PubChem/ChEBI, GHS hazards; *no* lot/qty/location). **storage_units** = the only native tree (physical storage) + `containers2*` carry `qty_stored`/`qty_unit`. **uploads** (content-addressed, immutable by default), **comments**, **steps** (flat checklist), **revisions** (body-only snapshots), **changelog** (textual audit — UI only, no MCP tool).

## 2. The `metadata` / `extra_fields` JSON (the only structured custom-data mechanism)

```json
{
  "extra_fields": {
    "Field Name": {            // the KEY is the field's identity — renaming = orphaning
      "type": "number",        // default "text"
      "value": "12",           // a string, a number id for link types, or an array if allow_multi_values
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

Field `type`s (15): `text, number, select, radio, checkbox, date, datetime-local, time, email, url`, and the **entity-link types** `experiments`, `items`, `users`, `compounds`, `uploads`. Setting an `experiments`/`items`/`compounds` field via `elab_set_extra_field` also creates the real link row (unless `autoLink:false`); `value` holds the target id. `select` is what the UI calls "Dropdown menu". Groups are **visual only**; field names must be unique per entity.

**Templates are copy-on-create, NOT a live binding.** Editing a template's schema does **not** propagate to existing instances — each carries its own frozen copy. Carry a `schema_version` field and reconcile (pattern P5).

## 3. The MCP tool surface

Tools may be deferred — load schemas with `ToolSearch` (`select:mcp__elabftw__elab_get,...`) before first use. What is exposed depends on the server's configuration, so a missing tool is a **configuration fact, not a capability gap**:

- **Read-only by default.** Create/update/tag/link/step/comment tools need `ELABFTW_ALLOW_WRITES=true`.
- **Destructive gate.** `elab_lock`, `elab_unlock`, `elab_timestamp`, `elab_bloxberg`, `elab_delete_entity`, `elab_delete_tag`, `elab_delete_comment`, `elab_delete_compound` additionally need `ELABFTW_ALLOW_DESTRUCTIVE=true`.
- **Toolsets.** `ELABFTW_TOOLSETS` can drop groups (`extra_fields`, `steps`, `comments`, `files`, `compounds`, `history`, `users`, `scheduler`, `integrity`; `core` is always on).
- **Single key.** With one API key the `team` parameter is absent from the schemas.

If a tool you need is absent, tell the user which flag or toolset enables it — don't work around it with a cruder tool (e.g. never replace `elab_set_extra_field` with a whole-blob `metadata` PATCH because the extra_fields toolset is off).

Grouped by job:

- **Orient:** `elab_me`, `elab_info` (version), `elab_configured_teams`, `elab_list_teams`, `elab_list_team_users` (team-admin key), `elab_list_templates`, `elab_list_items_types`, `elab_list_experiments_categories`, `elab_list_items_categories` (6.x), `elab_list_experiments_status`, `elab_list_items_status`, `elab_list_tags`, `elab_list_extra_field_names` (instance-wide field names + counts).
- **Find:** `elab_search` (per team; `extended` = the elabFTW search DSL: `tag:`, `category:`, `status:`, `rating:`, `date:`, `extrafield:`, boolean `and/or/not`, quote values with spaces — verify the exact `extrafield:` grammar against the instance before relying on it), `elab_search_all_teams`, `elab_search_compounds`, `elab_search_pubchem`, `elab_search_users`.
- **Read:** `elab_get` / `elab_get_bulk` (≤ 50; `include:["links","steps","comments","attachments"]` for one round-trip; `view:"native"` for round-trippable JSON; `format:"markdown"` for full bodies with tables and links). `elab_list_links`, `elab_list_extra_fields`, `elab_list_steps`, `elab_list_unfinished_steps`, `elab_list_comments`, `elab_list_attachments`, `elab_list_revisions`, `elab_get_revision`, `elab_list_events`.
- **Write entities:** `elab_create_entity` — **create from a template with `template=<id>`** (server-side instantiation: body, extra-field schema, steps, links, tags, permissions, category — exactly what the UI does); everything else you pass is applied on top, `metadata` merged per field. `elab_update_entity` (omitted fields stay; accepts `native`), `elab_duplicate_entity`, `elab_convert_body_to_html`, `elab_export` (`.eln`, PDF/A, JSON, CSV, …).
- **Bodies:** write markdown; the server renders it to HTML so lab members can keep editing in the rich-text editor.
- **Extra fields:** `elab_set_extra_field` (create-or-replace one field, typed + validated, auto-link), `elab_update_extra_field` (value only, fast path), `elab_remove_extra_field`, `elab_set_extra_field_groups`, `elab_clone_extra_fields_schema` (the UI's "Load fields").
- **Tags:** `elab_add_tag` (auto-creates the tag if absent), `elab_remove_tag` (by tag id), `elab_create_tag`; `elab_delete_tag` (destructive — detaches from every entity).
- **Links:** `elab_link_entities`, `elab_unlink_entities` (`targetKind` experiments / items / compounds).
- **Compounds:** `elab_create_compound`, `elab_create_compound_from_pubchem`, `elab_get_compound`, `elab_update_compound`.
- **Steps / comments / files:** `elab_add_step`, `elab_update_step`, `elab_toggle_step`, `elab_delete_step`, `elab_add_comment`, `elab_update_comment`, `elab_download_attachment`.
- **Integrity (destructive gate):** `elab_lock`, `elab_unlock`, `elab_timestamp` (RFC 3161, consumes the instance's timestamp balance), `elab_bloxberg`, `elab_delete_entity` (soft-delete, state=3, restorable in the UI). There is no signing tool — signatures need the user's passphrase and are done in the UI.

## 4. What elabFTW does NOT do — and what you must do instead

elabFTW **validates nothing**: `required` is cosmetic, metadata is an unchecked blob, links are untyped, tags are flat, there are no rollups/computed fields, schemas don't propagate, revisions cover only the body. **You are the schema, validation, provenance and aggregation layer.** The full gap catalogue and the workaround patterns (P1 structured-tag namespace, P2 artifact passport, P3 typed relations, P4 controlled vocab, P5 schema_version, P6 agent-as-validator, P7 lock+timestamp, P8 one-schema-per-team, P9 versioned protocols) and the resource-vs-experiment rule are in `references/eln-modeling-patterns.md` — read it before doing schema work. The lab's concrete instantiation is in `labschema-<lab>`.

## 5. Operating rules (non-negotiable)

1. **Verify every write.** Re-read with `elab_get(view:"native")` after create/update/tag/link. Trust per-entity reads; tag counts in `elab_list_tags` lag. `elab_create_entity` reports fields it could not apply — read that part of the response.
2. **Create from templates.** New entries of a known kind are created with `template=<id>`, never by hand-copying a schema into `metadata`. Then stamp `schema_version` if the template doesn't.
3. **Never rename an extra-field key.** The key *is* the field identity; renaming orphans searches/templates/integrations. Deprecate-and-add instead.
4. **Never write whole blobs you didn't just read.** `elab_update_entity(metadata=…)` replaces the entire extra-fields blob, and `native.tags` is applied as *exact match* (tags missing from your copy are removed). Use the per-field and per-tag tools; use `native` round-trips only on a copy read moments before, on an entity nobody else is editing.
5. **Tag hazard.** Removing a tag shortly after adding one on the same entity has wiped entire tag sets. Prefer pure additive tagging; never interleave add+remove on the same entity in one burst; when retiring a tag, verify the add with a native re-read first.
6. **Never overwrite history.** Locked or archived entries reject edits — that is a signal, not an obstacle: stop and ask. Corrections are additive with a reason (a comment, or a new entry linked to the old one). For defensible records: `elab_lock` **and** `elab_timestamp` (lock alone is reversible by its owner and admins; a timestamp alone doesn't freeze the row); metadata isn't in revisions, so snapshot integrity-critical metadata into the body before locking.
7. **Respect the contract.** When `labschema-<lab>` is loaded, enforce its required fields, vocabularies, namespaces, protocol conventions and typed-link conventions on every write; audit them on every read; report drift.
8. **Team scope always.** Pass `team` explicitly on multi-team instances; never assume the default.
9. **Soft-delete only, and only when asked** (FAIR-A2 tombstone): `elab_delete_entity` sets state=3 (restorable); archiving (`state:"archived"`) also locks. Never delete as a side effect of tidying.

## 6. Talking to the researcher

- **Irreversible or wide actions** (lock, timestamp, delete, bulk retag, anything touching > 5 entries) → show the exact list of what will change, then confirm with `AskUserQuestion` (question names the action and the count; options: do it / do a smaller test batch first / cancel).
- **Choices** with 2–4 options (which template, which batch, which of two conflicting values is right) → `AskUserQuestion` with your recommendation first, marked `(Recommended)`, grounded in what you read in the instance — not a prose menu.
- **Larger overviews** (everything of a batch, drift report across many entries, a proposed restructuring) → a decision sheet (`assets/decision-sheet.html`, see `references/decision-sheets.md`) rather than a long chat table.

When unsure whether an action is reversible on a real notebook, stop and ask the researcher.
