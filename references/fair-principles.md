# FAIR Data Principles — reference

Normative reference for `elabftw-schema-init`. Consulted on demand; not loaded every session.

**Canonical source.** Wilkinson, M. D., Dumontier, M., Aalbersberg, Ij. J. *et al.* "The FAIR Guiding Principles for scientific data management and stewardship." *Scientific Data* **3**, 160018 (2016). DOI: [10.1038/sdata.2016.18](https://doi.org/10.1038/sdata.2016.18). Operationalization: GO FAIR, <https://www.go-fair.org/fair-principles/>.

Two load-bearing framing points from the paper:

- The principles apply to **`(meta)data`** — metadata and data are addressed *separately*, so metadata can stay FAIR even when the data are restricted, embargoed, or gone. This is what drives the **A2 tombstone rule**.
- The end goal is **machine-actionability**: a machine should find, access, interoperate with, and reuse a digital object with minimal human intervention. Free text defeats this; typed fields + vocabularies enable it.

---

## The 15 sub-principles → concrete elabFTW practice

### Findable

| ID | Verbatim | elabFTW practice |
|----|----------|------------------|
| **F1** | "(Meta)data are assigned a globally unique and persistent identifier" | Every entity carries the system `elabid` (UUID) + `id`, and a meaningful user `custom_id`. IDs are never reused or repurposed. Mirror/mint DOIs for published datasets. |
| **F2** | "Data are described with rich metadata (defined by R1 below)" | Mandatory **extra-fields schema** per items_type / template — not just the free-text body. Title, type, status, project, instrument, operator, date as structured fields. |
| **F3** | "Metadata clearly and explicitly include the identifier of the data they describe" | The metadata record carries its own id and the ids/filenames+checksums of the raw files it describes. Metadata is never orphaned from its data. |
| **F4** | "(Meta)data are registered or indexed in a searchable resource" | All entities indexed and reachable via the elabFTW search/API; on publication, also registered in an external indexed repository (Zenodo, an institutional or domain repository). |

### Accessible

| ID | Verbatim | elabFTW practice |
|----|----------|------------------|
| **A1** | "(Meta)data are retrievable by their identifier using a standardised communications protocol" | Retrievable by id over the HTTPS REST API v2, not only the GUI. |
| **A1.1** | "The protocol is open, free, and universally implementable" | HTTP/REST/JSON; no proprietary-only access path. |
| **A1.2** | "The protocol allows for an authentication and authorisation procedure, where necessary" | API keys / SSO with per-team, per-user scopes. Access-controlled, not access-blocked. |
| **A2** | "Metadata are accessible, even when the data are no longer available" | **Tombstone**: on delete/embargo/loss, retain the metadata record (what it was, who, when, why withdrawn). Never hard-delete metadata with its data. elabFTW soft-delete (`state=3`) supports this. |

### Interoperable

| ID | Verbatim | elabFTW practice |
|----|----------|------------------|
| **I1** | "(Meta)data use a formal, accessible, shared, and broadly applicable language for knowledge representation" | Export in standard structured formats — the elabFTW `.eln` consortium format, JSON, ISA, RO-Crate — not only PDF/screenshots. |
| **I2** | "(Meta)data use vocabularies that follow FAIR principles" | Metadata fields bound to community vocabularies with resolvable URIs: units (QUDT/OM), persons (ORCID — `users.orcid`), instruments (OBI/CHMO or a vendor-model registry), and domain registries where one exists — chemicals (ChEBI / PubChem CID — elabFTW compounds carry these), biological resources (RRID / Addgene / GenBank), test methods & materials (ASTM/ISO designations, UNS/EN grades). `select` extra-fields backed by the controlled list. |
| **I3** | "(Meta)data include qualified references to other (meta)data" | **Qualified** links, not generic "related": sample *wasGeneratedBy* synthesis; dataset *wasDerivedFrom* a run; a measurement record *isMeasurementOf* its sample/specimen. elabFTW links are untyped — the relation is encoded out-of-band (see `eln-modeling-patterns.md` §typed-links). |

### Reusable

| ID | Verbatim | elabFTW practice |
|----|----------|------------------|
| **R1** | "(Meta)data are richly described with a plurality of accurate and relevant attributes" | Conditions, parameters, units, instrument settings, calibration state captured as fields — enough to reproduce without asking the author. |
| **R1.1** | "(Meta)data are released with a clear and accessible data usage license" | A `license` field on shared/exported datasets (CC-BY-4.0, CC0). Default-deny is not FAIR-reusable. |
| **R1.2** | "(Meta)data are associated with detailed provenance" | Who, when, from what, with what, derived-from-which — captured via the audit trail + explicit typed entity links. See `data-integrity-alcoa.md` and PROV mapping. |
| **R1.3** | "(Meta)data meet domain-relevant community standards" | Templates structured around the domain's minimum-information checklist (MIBBI-style) / ISA model. Compliance is structural (required fields), not discretionary. |

---

## GO FAIR FAIRification process (operational recipe)

1. retrieve non-FAIR data → 2. analyse → 3. **define a semantic model of entities/relations** → 4. make data linkable (URIs) → 5. assign a license → 6. define rich metadata → 7. deploy as an indexed FAIR resource.

For schema design this means: **design the schema as a semantic model first** (entities, typed relations — PROV/ISA), bind fields to vocabularies, attach a license, then make repository handoff a first-class operation.

---

## Source register

| # | Source | Identifier |
|---|--------|-----------|
| 1 | Wilkinson et al., *Scientific Data* 3:160018 (2016) | DOI [10.1038/sdata.2016.18](https://doi.org/10.1038/sdata.2016.18) |
| 2 | GO FAIR — FAIR Principles & FAIRification | <https://www.go-fair.org/fair-principles/> |
| 3 | ISA abstract model | <https://isa-specs.readthedocs.io/> |
| 4 | FAIRsharing (MIBBI / standards registry) | <https://fairsharing.org/> |
