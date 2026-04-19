# crkg — Architecture

**Document version:** v0.0 (2026-04-19)
**Scope:** design, not implementation. Describes what `crkg` will be at v1.0.

---

## 1. Design goals

1. **Library, not application.** `crkg` is consumed by CDSS applications and research tools. It is not a CDSS itself.
2. **Schema-driven.** Every entity, relationship, and constraint flows from `crkg-schema`. No parallel schema lives in this repo.
3. **Backend-agnostic by interface.** Graph store access goes through a `GraphStore` Protocol. Neo4j is the only implementation in v0.x; Memgraph is planned for v0.2+. The Protocol exists from v0.1 so consumers do not code against driver specifics.
4. **Adapter-driven for licensed data.** Any data source under non-redistributable terms (traditional-medicine corpora, licensed terminologies beyond their URL references) enters through an adapter interface defined here. `crkg` ships reference filesystem adapters only.
5. **No redistribution of licensed data.** The graph store is populated by the deployer using their own licensed sources. `crkg` parses formats; it does not ship content.
6. **Honest about boundaries.** Where `crkg` stops and where consuming applications begin is defined, written down, and enforced by the package layout.

---

## 2. Layers

### 2.1 Model layer (from `crkg-schema`)

Imports Pydantic v2 models emitted from `crkg-schema`. No modifications. No shadowing classes. If a model needs extension, the extension lives in `crkg-schema`, not here.

Entrypoint: `crkg.models` (re-exports `crkg_schema.models.*`).

### 2.2 Graph layer

Typed access to a property-graph store.

**`GraphStore` Protocol** — defined once, implemented per backend. Methods:

- `bootstrap()` — apply constraints and indexes from `crkg-schema`'s Cypher DDL emission
- `upsert_node(node)` — idempotent upsert based on identifier
- `upsert_edge(from_id, to_id, edge_type, properties)` — idempotent edge write
- `find_node(node_type, filters)` — typed query
- `traverse(start, pattern)` — pattern-based traversal
- `raw_cypher(query, parameters)` — escape hatch for complex queries not yet abstracted

**Neo4j implementation** — `crkg.graph.neo4j.Neo4jGraphStore`. Uses the official `neo4j` Python driver over Bolt. v0.x default and only implementation.

**Memgraph implementation** — deferred to v0.2+. Bolt-compatible, so the implementation is mostly a driver swap; documented in TODO.

### 2.3 Ingestion layer

One module per upstream terminology / data source. Each module:

- Takes a path to the source material (never bundled)
- Parses into `crkg-schema` Pydantic models
- Writes via `GraphStore.upsert_node` / `upsert_edge`

Planned M1 modules:

- `crkg.ingestion.icd11` — WHO ICD-11 API / JSON dumps
- `crkg.ingestion.snomed` — SNOMED CT RF2 files (International edition, Indonesian edition)
- `crkg.ingestion.loinc` — LOINC CSV
- `crkg.ingestion.rxnorm` — RxNorm RRF files
- `crkg.ingestion.atc` — WHO ATC/DDD tables
- `crkg.ingestion.region_epi` — regional epidemiology (pluggable; reference impl reads a JSON file)
- `crkg.ingestion.formulary` — national formularies (pluggable; reference impl reads a JSON file)

Each loader is independent. Each loader can be run standalone. No loader depends on another at runtime.

### 2.4 Adapter layer

Pluggable contracts for data sources that cannot be embedded.

**`TraditionalMedicineAdapter` Protocol** — defined in `crkg.adapters.traditional_medicine`. Methods:

- `list_entities() -> Iterable[EthnobotanyEntity]` — stream every entry the adapter exposes
- `get_entity(entity_id: str) -> EthnobotanyEntity | None` — single lookup
- `provenance() -> ProvenanceDescriptor` — describes what this adapter is (corpus name, license, source URL or reference, version)

Reference implementations:

- `FilesystemTraditionalMedicineAdapter` — reads a directory of JSON/YAML files matching the `EthnobotanyEntity` schema. Zero external dependencies. Suitable for public, small corpora and for testing.
- `NullTraditionalMedicineAdapter` — no-op adapter. Returns nothing. Default when consumers do not configure a real one.

External implementations live in external repos (public or proprietary) and are injected by the consumer.

### 2.5 Retrieval layer

Hybrid symbolic + dense retrieval.

- **Symbolic**: graph traversal via `GraphStore`. Given extracted entities, walks `HAS_SYMPTOM`, `TREATED_BY`, `CAUSED_BY` relationships.
- **Dense**: vector similarity against Qdrant. Node descriptions are embedded at ingestion time.
- **Fusion**: reciprocal rank fusion (default) or weighted score (configurable). Returns a ranked list of evidence.

Retrieval is deliberately plain. No LLM calls. No reasoning. No DDx generation. Those belong in consuming applications.

---

## 3. Package layout

```
crkg/
├── README.md
├── ARCHITECTURE.md
├── TODO.md
├── LICENSE                       Apache-2.0
├── NOTICE                        third-party attributions + origin note
├── CONTRIBUTING.md               (Phase 7 of M0)
├── SECURITY.md                   (Phase 7 of M0)
├── CHANGELOG.md
├── pyproject.toml
├── uv.lock
├── .python-version               3.13
├── .pre-commit-config.yaml
├── .github/
│   ├── workflows/
│   │   ├── ci.yml
│   │   └── release.yml
│   └── CODEOWNERS
│
├── src/crkg/
│   ├── __init__.py               version, public API re-exports
│   ├── models.py                 re-exports crkg_schema.models
│   │
│   ├── graph/
│   │   ├── __init__.py           GraphStore Protocol
│   │   ├── neo4j.py              Neo4jGraphStore
│   │   └── memgraph.py           (v0.2+)
│   │
│   ├── ingestion/
│   │   ├── __init__.py
│   │   ├── icd11.py
│   │   ├── snomed.py
│   │   ├── loinc.py
│   │   ├── rxnorm.py
│   │   ├── atc.py
│   │   ├── region_epi.py
│   │   └── formulary.py
│   │
│   ├── adapters/
│   │   ├── __init__.py
│   │   └── traditional_medicine.py   Protocol + FilesystemImpl + NullImpl
│   │
│   ├── vector/
│   │   ├── __init__.py
│   │   ├── embedding.py          embedding pipeline
│   │   └── qdrant.py             Qdrant sync
│   │
│   ├── retrieval/
│   │   ├── __init__.py
│   │   ├── graph.py              graph traversal
│   │   ├── vector.py             vector similarity
│   │   └── fusion.py             RRF + weighted
│   │
│   └── cli.py                    thin operator CLI (status, bootstrap, load)
│
├── tests/
│   ├── unit/                     per-module tests with Neo4j + Qdrant mocked
│   ├── integration/              against real Neo4j + Qdrant via testcontainers
│   └── fixtures/                 small synthetic terminology slices
│
└── docs/
    ├── DEV_SETUP.md
    ├── COMPATIBILITY.md          crkg ↔ crkg-schema version matrix
    ├── ADAPTERS.md               how to write a TraditionalMedicineAdapter
    └── MIGRATION_FROM_MKG.md     origin note + migration steps (historical)
```

---

## 4. Decision record

- **D-01** (Schema dependency): consume `crkg-schema`, do not parallel-model — Locked 2026-04-19
- **D-02** (Graph backend v0.x): Neo4j Community Edition via Bolt driver — Locked 2026-04-19
- **D-03** (Graph backend v0.2+): Memgraph Community Edition optional via `GraphStore` Protocol — Planned
- **D-04** (Graph backend Kùzu): dropped from scope after October 2025 project archival — Locked 2026-04-19
- **D-05** (License): Apache-2.0 — Locked 2026-04-19
- **D-06** (Python floor): `>=3.13` — Locked 2026-04-19
- **D-07** (Package manager): `uv` exclusively — Locked 2026-04-19
- **D-08** (Vector store): Qdrant — Locked 2026-04-19 (matches the `/mkg/` origin; reconsidered at v1.0)
- **D-09** (Ingestion philosophy): one module per source, no shared dispatcher, no if/elif tree — Locked 2026-04-19
- **D-10** (Traditional medicine): adapter protocol, not direct loader — Locked 2026-04-19
- **D-11** (CI platform): GitHub Actions — Locked 2026-04-19
- **D-12** (Test infrastructure): `pytest` + `testcontainers` for integration — Locked 2026-04-19
- **D-13** (Public API surface): everything under `crkg.*` is stable from v1.0; everything under `crkg._internal.*` is free to change — Locked 2026-04-19
- **D-14** (HL7 ingestion): `crkg` does NOT include HL7 v2 / FHIR parsers. Hospital ingestion is a consuming-application concern. Consumers using the H3 hybrid pattern (Rust `hl7-rs` → Arrow → Python) populate `crkg` with the resulting structured events. — Locked 2026-04-19

---

## 5. Licensing posture and the Neo4j GPLv3 question

### 5.1 `crkg` itself

Apache-2.0. All original code in this repository. See `LICENSE`.

### 5.2 Neo4j Community Edition (GPLv3)

`crkg` is a client of Neo4j, connecting via the Bolt protocol through the official `neo4j` Python driver. The Neo4j Python driver is Apache-2.0. The Neo4j server itself is GPLv3 (Community) or commercial (Enterprise).

The mainstream legal interpretation — consistent with how thousands of Apache/MIT-licensed projects use Neo4j today — is that a client connecting to a Neo4j server over Bolt does not become a GPL derivative work. The client-server boundary separates the two. `crkg` therefore remains Apache-2.0; deployers of `crkg` using Neo4j Community remain subject to GPLv3 for the server itself, which is unaffected by `crkg`.

`crkg` does **not**:
- Bundle Neo4j binaries
- Vendor Neo4j source
- Embed Neo4j in-process
- Link against Neo4j's GPL-licensed libraries in any form

`crkg` **does**:
- Document Neo4j Community as the default backend (operationally, not legally)
- Depend on the Apache-2.0 `neo4j` Python driver
- Ship Bolt connection parameters in configuration only

### 5.3 Memgraph Community Edition (BSL 1.1)

When the Memgraph backend lands in v0.2+:

- Memgraph Community is source-available under the Business Source License 1.1, not OSI open source. It converts to Apache-2.0 four years after each release.
- The Additional Use Grant permits internal business use but restricts redistribution of Memgraph as a standalone service.
- `crkg` will connect to Memgraph via Bolt (same driver model). The client-server boundary applies here too.
- Documentation will make the BSL-not-OSI status explicit so consumers in OSI-strict environments can avoid Memgraph and stay on Neo4j.

### 5.4 Upstream terminology licenses

`crkg` ingests ICD-11, SNOMED CT, LOINC, RxNorm, ATC, and regional data. All of these have their own licenses. `crkg` ships parsers only. The deployer is responsible for obtaining and complying with each.

---

## 6. Relationship to the MKG in the origin project

This section is historical. It documents where the code in `src/crkg/ingestion/`, `src/crkg/graph/neo4j.py`, and `src/crkg/vector/` originates and how the migration preserved integrity.

### 6.1 Origin

The `crkg` package originates as an extraction of the `/mkg/` folder from a proprietary CDSS codebase. The originating project (a clinical decision support system for pre-hospital and pre-ICU settings) had developed:

- A Neo4j schema for clinical entities
- Loaders for ICD-11, SNOMED CT, Kemenkes regional epidemiology, FORNAS formulary
- A Usada (Balinese traditional medicine) loader
- An embedding pipeline and Qdrant sync
- Hybrid retrieval over graph and vector stores

### 6.2 What changed in extraction

- **License**: from proprietary → Apache-2.0. Only code originally authored by the same copyright holder was extracted; no third-party code needed relicensing.
- **Schema source**: from Cypher DDL files → `crkg-schema` LinkML source. Cypher DDL is now emitted from the schema, not authored by hand.
- **Usada loader**: **removed**. Replaced by the `TraditionalMedicineAdapter` protocol. Usada data now flows through an external adapter (the traditional-medicine corpus project named in `NOTICE`), never through `crkg` directly.
- **Dispatch pattern**: no change needed — the origin already used one module per source. No if/elif trees to refactor.
- **Backend coupling**: the `GraphStore` Protocol is new. The origin was Neo4j-only without abstraction.
- **Tests**: strengthened. The origin's test suite had coverage gaps. Extraction re-tests every loader against synthetic terminology fixtures before acceptance.

### 6.3 What the migration preserves

- Node/edge semantics
- Loader behavior for ICD-11, SNOMED CT, formulary, regional epidemiology
- Vector embedding model choice (pending review at v1.0)
- Hybrid retrieval fusion logic

### 6.4 What the migration does not preserve

- Usada direct ingestion (moved to adapter)
- Direct Cypher schema files (now emitted)
- Configuration-file formats (refactored to match `crkg-schema` conventions)

The migration is coordinated with the originating project via the cross-repo task list (`CROSS-REPO-TASKS.md` in this conversation's artifacts). The origin deletes its local `/mkg/` in the same release cycle that `crkg` publishes v0.1.0.

---

## 7. Non-functional targets (v0.1 aspirational, v1.0 committed)

- Cold ingest: 100K-entity ICD-11 in **< 10 min** against local Neo4j
- Cold ingest: full SNOMED CT International in **< 60 min**
- Query latency (typed find_node): **< 50 ms p95** on a warm graph of 1M nodes
- Package sdist size: **< 5 MB** (no bundled data)
- Public API docstring coverage: **100%**

Aspirational until measured. v0.x publishes measurements; v1.0 commits to the numbers that held in v0.x.

---

## 8. What this document does not cover

- The `crkg-schema` modeling approach (see its `ARCHITECTURE.md`)
- Clinical validation of the reasoning this graph enables (belongs to consuming CDSS)
- SaMD regulatory posture (belongs to consuming CDSS)
- Specific prompts, LLM calls, or DDx algorithms (belongs to consuming CDSS)
- Hospital integration (HL7/FHIR/MLLP) — out of scope for `crkg`; see `crkg-schema` Layer 1 consumers and sibling project `hl7-rs`
- Traditional-medicine corpus authoring, provenance, or digital signing (belongs to each adapter implementer)

---

## Version log

| Version | Date | Description |
|---|---|---|
| v0.0 | 2026-04-19 | Initial architecture. Layers, GraphStore Protocol, adapter contract, migration origin, decision record D-01..D-14, Neo4j GPLv3 licensing analysis. No code yet. |
