# Data Integrity (ALCOA / ALCOA+) & the Data Lifecycle — reference

Normative reference for `elabftw-schema-init`. Consulted on demand.

---

## 1. ALCOA / ALCOA+

**Canonical source.** MHRA, *'GXP' Data Integrity Guidance and Definitions*, Revision 1, March 2018. <https://www.gov.uk/government/publications/guidance-on-gxp-data-integrity>. Origin: FDA/GxP good-practice regulation; "ALCOA" coined by Stan Woollen (FDA, 1990s); the "+" attributes added by EMA/MHRA.

> MHRA uses "ALCOA" and states there is **no difference in expectation** vs "ALCOA+". Quote this to preempt the common misconception that ALCOA+ is a stricter standard.

**MHRA definition of data integrity (§6.4, verbatim):** *"Data integrity is the degree to which data are complete, consistent, accurate, trustworthy, reliable and that these characteristics of the data are maintained throughout the data life cycle. The data should be collected and maintained in a secure manner, so that they are attributable, legible, contemporaneously recorded, original (or a true copy) and accurate."*

| Attribute | MHRA wording | ELN entry discipline |
|-----------|--------------|----------------------|
| **Attributable** | "attributable to the person generating the data" | Every create/modify bound to an authenticated user — **no shared logins**. Instrument data attributed to its source instrument. |
| **Legible** | "legible and permanent" | Human-readable, durable; entries not erasable; corrections strike-through-visible, never overwritten. |
| **Contemporaneous** | recorded at the time of the activity | Server-side, synchronised, timezone-aware timestamps at the point of action. No "write it up later." (MHRA §5.1: controlled/synchronised clocks.) |
| **Original** | "original record (or certified true copy)" | First capture preserved; dynamic data (spectra, DAQ traces, raw datasets) kept dynamic, not flattened to PDF. A *true copy* retains content, context, structure, metadata, audit trail. |
| **Accurate** | "accurate" | Correct & truthful; supported by calibration, review/approval, verified transcription. |
| **Complete** | "the data must be whole; a complete set" | All data — including repeats and **excluded** data — retained with the original set and reason recorded. No selective deletion. |
| **Consistent** | "self-consistent" | Consistent sequencing, dating, cross-record agreement across the lifecycle. |
| **Enduring** | "durable; lasting throughout the data lifecycle" | Retained for the full retention period in a controlled archive; survives system decommissioning. |
| **Available** | "readily available for review or inspection" | Retrievable on demand throughout retention. |

**Audit trail (MHRA §6.13, verbatim):** *"The audit trail is a form of metadata containing information associated with actions that relate to the creation, modification or deletion of GXP records … without obscuring or overwriting the original record … facilitates the reconstruction of the history of such events … including the 'who, what, when and why' of the action."*

ELN requirement: audit trail **on, non-disableable by users, captures old+new value + timestamp + actor + reason**, reviewable. This is why an ELN must enforce **lock + immutable revision history**, not in-place overwrite.

> **Strongest cross-framework convergence:** *no-overwrite / immutable revision* is simultaneously ALCOA-Original, the MHRA audit-trail definition, and FAIR-R1.2 provenance. It is the **non-negotiable backbone** of any ELN schema.

### elabFTW caveat (read with `eln-modeling-patterns.md`)

elabFTW **revisions snapshot the `body` only** — not `metadata` (extra fields), status, links, or tags. The changelog logs metadata changes textually but they are **not restorable**. Locking is a gate, not cryptographic immutability; an admin can unlock. Trusted timestamping (RFC3161 / Bloxberg) and digital signatures produce an **external immutable archive snapshot** but do **not** freeze the live row. For a hard freeze you must **lock *and* timestamp**, and capture metadata-critical state in the body or an external snapshot if it must be restorable.

---

## 2. The data lifecycle — the ELN is the start, not the end

**Canonical source.** *The DCC Curation Lifecycle Model*, Higgins, S. (2008), *International Journal of Digital Curation* 3(1). DOI: [10.2218/ijdc.v3i1.48](https://doi.org/10.2218/ijdc.v3i1.48). <https://www.dcc.ac.uk/guidance/curation-lifecycle-model>.

- **Full-lifecycle actions** (continuous): Description & Representation Information; Preservation Planning; Community Watch; Curate & Preserve.
- **Sequential actions**: Conceptualise → Create/Receive → Appraise & Select → Ingest → Preservation Action → Store → Access/Use/Reuse → Transform.
- **Occasional**: Dispose, Reappraise, Migrate.

Simpler generic lifecycle (UK Data Service): **Plan → Collect → Process → Analyse → Preserve → Share → Reuse**.

**Role of the ELN.** It is the system of record for **Create/Receive → Process → Analyse** and the *origin point* for Description/Representation metadata (captured at creation). It is **not** the long-term preservation store — Preservation Action / Store / Dispose belong to a repository the ELN feeds.

Design implications:
- Make **export / repository handoff** (FAIR formats: `.eln`, ISA, RO-Crate, JSON-LD) a first-class, routine operation.
- Apply **Appraise-and-Select** discipline: decide what raw data is worth keeping, and record the decision.
- Capture preservation metadata **at creation time**, not retrospectively.

---

## 3. Provenance — W3C PROV (the lineage backbone)

**Canonical source.** *PROV-DM: The PROV Data Model*, W3C Recommendation, 30 April 2013. <https://www.w3.org/TR/2013/REC-prov-dm-20130430/>.

Three core types: **Entity** (a sample, a specimen, a dataset, a spectrum or DAQ trace, an input lot), **Activity** (a synthesis, a fabrication, a measurement, a test, a simulation or processing run), **Agent** (the researcher, the instrument or solver software, the lab).

Core relations (the queryable lineage spine):

| PROV relation | Direction | Lab instance |
|---------------|-----------|--------------|
| **wasGeneratedBy** | entity ← activity | sample *wasGeneratedBy* synthesis; dataset *wasGeneratedBy* a run |
| **used** | activity → entity | synthesis *used* reagents; analysis *used* its sample/dataset |
| **wasDerivedFrom** | entity ← entity | product *wasDerivedFrom* starting material; processed data *wasDerivedFrom* raw |
| **wasAttributedTo** | entity ← agent | spectrum/result *wasAttributedTo* operator |
| **wasAssociatedWith** | activity ← agent | measurement *wasAssociatedWith* analyst |
| **wasInformedBy** | activity ← activity | analysis *wasInformedBy* the producing run |

The generic lineage spine `inputs → [activity] → entity → [analysis] → result` **is** PROV — instantiate it per domain: chemistry `reagents → [synthesis] → sample → [analysis] → spectrum`; physics/materials `recipe/stock → [fabrication] → device/specimen → [measurement/test] → dataset`; biology `cells/plasmids → [culture/assay] → sample → [sequencing/flow] → result`; computational `code + config → [run] → dataset → [analysis] → result`. (The producing activity may be absent — a purchased or standard sample — in which case provenance starts at acquisition.) This is exactly what FAIR-I3 ("qualified references") and FAIR-R1.2 ("detailed provenance") demand and what ALCOA Attributable/Original enforces at entry level. Because elabFTW links are untyped, the relation must be encoded out-of-band — see `eln-modeling-patterns.md`.

---

## Source register

| # | Source | Identifier |
|---|--------|-----------|
| 1 | MHRA, *'GXP' Data Integrity Guidance & Definitions*, Rev 1, Mar 2018 | <https://www.gov.uk/government/publications/guidance-on-gxp-data-integrity> |
| 2 | DCC Curation Lifecycle Model (Higgins 2008), *IJDC* 3(1) | DOI [10.2218/ijdc.v3i1.48](https://doi.org/10.2218/ijdc.v3i1.48) |
| 3 | UK Data Service — research data lifecycle | <https://www.ukdataservice.ac.uk/manage-data/lifecycle/> |
| 4 | W3C PROV-DM, W3C Recommendation 2013 | <https://www.w3.org/TR/2013/REC-prov-dm-20130430/> |
