# CLAUDE.md

Guidance for Claude Code when working **on** this repository: the `elabftw-rdm` Claude Code plugin. (Labs that *use* the plugin don't need this file — the workshop adds a small marked block to their own CLAUDE.md.)

## What is here

| Path | What |
|------|------|
| `skills/elabftw-mcp/SKILL.md` | Generic operating manual for an agent working in elabFTW through the elabftw MCP server |
| `skills/elabftw-schema-init/SKILL.md` | The design workshop (`/elabftw-schema-init`) |
| `skills/labschema/SKILL.md` | Template + worked example of the per-lab contract the workshop generates |
| `references/` | Normative backbone (FAIR, ALCOA/DCC/PROV), the gap → pattern catalogue (`eln-modeling-patterns.md`), and how proposals are presented (`decision-sheets.md`) |
| `assets/decision-sheet.html` | Self-contained renderer for decision sheets |
| `.claude-plugin/` | `plugin.json` + `marketplace.json` |

No build, no code to run — the deliverable is the text the agent reads. There are no automated tests.

## Rules when editing

- **Tool names must exist.** Every `elab_*` tool, parameter and env flag named in a skill must exist in the MCP server ([`letrplB/elabftw-mcp`](https://github.com/letrplB/elabftw-mcp), its README tool tables). When the server renames, removes or re-gates a tool, update `skills/elabftw-mcp/SKILL.md` and the Phase 0 list in `elabftw-schema-init`. Say which server version the skills are written against (README *Status*).
- **elabFTW claims need a source.** Behaviour claims (what elabFTW stores, enforces or offers in team settings) come from the elabFTW source/docs or the MCP server's tested behaviour — note anything unverified as "confirm per instance".
- **Pattern and gap ids are stable.** P1–P9 and G1–G16 are cited across all skills and in generated lab contracts. Add new ones at the end; never renumber.
- **Decision sheet template:** labs copy `assets/decision-sheet.html` and replace only the JSON block. Keep the renderer self-contained (no network, no external fonts or libraries) and backward-compatible with the data shape in `references/decision-sheets.md`; document any new section type there. Check a change by opening the file in a browser in light and dark mode and pressing *Accept all recommendations* → *Copy my answers*.
- **Researcher-facing wording** (sheet examples, the workshop's interaction rules) uses plain language — no pattern codes, acronyms or elabFTW internals.
- **Model requirement:** the workshop states Opus 5.5 or newer; keep README, the workshop's model check and the CLAUDE.md block in Phase 7 consistent when this changes.
- **Never** put lab data (dossiers, generated `labschema-<lab>` skills) into this repo.

## Release

- Bump `version` in `.claude-plugin/plugin.json` and in both places in `.claude-plugin/marketplace.json`, and the `version` frontmatter of any changed skill.
- Add a `CHANGELOG.md` entry (Added / Changed / Fixed).
- Users install from the default branch (`/plugin marketplace add letrplB/elabftw-rdm`), so a release is what is merged to `main`.
- Commit style: Conventional Commits — `feat: …`, `fix: …`, `docs: …`.
