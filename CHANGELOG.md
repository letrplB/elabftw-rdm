# Changelog

## 0.2.0 — 2026-11-06

### Added
- **Decision sheets** (`assets/decision-sheet.html`, `references/decision-sheets.md`): a self-contained, offline HTML renderer for proposals — lane diagram of the proposed structure, pre-answered decisions with the recommendation marked, "a normal week" storyboards, tables, notes, "Accept all recommendations", and a copy-paste export of the answers. The agent writes only the JSON data block; one sheet per phase in the lab's dossier.
- **Digestibility contract** in `elabftw-schema-init`: picture before prose, `AskUserQuestion` for discrete choices (context in the question, recommendation first and grounded in what was learned), the lab's own words instead of pattern codes, storyboards, one decision batch per turn, nothing changes without a shown plan and a yes.
- **Model requirement**: the workshop checks its model and recommends Opus 5.5 or newer before starting; README lists it under prerequisites.
- **Resource-or-experiment rule** (`eln-modeling-patterns.md` §3a) with a table of common borderline things.
- **P9 — versioned protocols** (and gap G16): protocol resource per version with a `supersedes` chain, experiment template per protocol family with `follows_protocol`, required `deviations` field.
- `labschema` template: protocols section, everyday-recipes section (tool-level steps for the lab's common tasks).
- Workshop: seed-plan sheet, sandbox-team-first recommendation, `06-seed-log.md` undo map, optional migration of existing entries (retag, backfill `schema_version`).
- README: prerequisites, quick start for a new lab.

### Changed
- `elabftw-mcp` aligned with `@sura_ai/elabftw` 0.7: templates vs. categories (6.x), create-from-template (`template=<id>`), write/destructive gates and toolsets (a missing tool is configuration, not capability), actual integrity tools (`elab_lock`, `elab_timestamp`, `elab_bloxberg`, `elab_delete_entity`), the `uploads` field type, markdown bodies stored as rich text, and the whole-blob hazards of `metadata` updates and `native.tags` round-trips.
- Phase 0 survey uses `elab_info`, `elab_list_templates`, `elab_list_items_categories` and `elab_list_extra_field_names`.
- The generated `labschema-<lab>` goes to the lab's project skills (`./.claude/skills/`) or `~/.claude/skills/`, no longer next to the template inside the plugin, where plugin updates would delete it.
- The human blueprint's structure is defined in the skill (it referenced a document colleagues don't have).
- Worked example: operator is a `users` link, not free text.

### Fixed
- Removed references to tools that don't exist (changelog listing, `action:lock`/`action:timestamp`) and the "set `state`" soft-delete instruction (`state` only archives; soft-delete is `elab_delete_entity`).
- README pointed to a dossier note in `eln-modeling-patterns.md` that didn't exist.

## 0.1.0

Initial release: `elabftw-mcp`, `elabftw-schema-init`, `labschema` skills; FAIR, ALCOA+/DCC/PROV and ELN-modelling references.
