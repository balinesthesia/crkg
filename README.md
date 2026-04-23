# crkg

**Clinical Reasoning Knowledge Graph — Python library.**

A working clinical-reasoning knowledge graph built from openly-specified schemas, standard vocabularies, and pluggable adapters. Consumes [`crkg-schema`](https://github.com/balinesthesia/crkg-schema) as the data contract and drives Neo4j as the default graph backend.

---

## Status

- **Version:** v0.0 (scaffold — no graph code yet)
- **License:** Apache-2.0
- **Posture:** Public, open source
- **Stage:** M0 infrastructure-first. Graph content and ingestion begin in M1.
- **Schema dependency:** [`crkg-schema`](https://github.com/balinesthesia/crkg-schema) — separately versioned
- **First consumer:** a CDSS application (private) will consume `crkg` once M1 completes.

---

## What `crkg` is

`crkg` is a Python library that provides:

1. **Pydantic models** emitted from `crkg-schema` — typed entities, relationships, and identifiers.
2. **A graph store driver** — Neo4j Community Edition via the official `neo4j` Bolt driver.
3. **Vocabulary loaders** — ICD-11, SNOMED CT, LOINC, RxNorm, ATC ingestion into the graph.
4. **Context loaders** — regional epidemiology, national drug formularies.
5. **Vector embedding sync** — Qdrant integration for hybrid symbolic + dense retrieval.
6. **A `TraditionalMedicineAdapter` protocol** — pluggable interface for traditional-medicine corpora. A filesystem reference implementation is provided. Proprietary corpora implement this interface themselves.
7. **Hybrid retrieval** — graph traversal fused with vector similarity for evidence gathering.

Everything above is a library. `crkg` ships no application, no web UI, no CDSS product.

## What `crkg` is not

- Not a clinical decision support system. CDSS applications consume `crkg`; they are not `crkg`.
- Not a clinical terminology distributor. Upstream terminologies (SNOMED CT, ICD-11, etc.) remain the responsibility of each deployer. `crkg` parses the formats; it does not ship the content.
- Not a regulated medical device. When a CDSS product built on `crkg` needs SaMD clearance, that obligation is on the product, not on this library.
- Not a graph server. `crkg` is a client of Neo4j (or Memgraph in v0.2+). Users run their own graph store.
- Not a data lake or lakehouse. For large-scale analytical workloads on clinical or multi-omics data, see [`clinical-rs`](https://github.com/SHA888/clinical-rs) and [`multiomics-rs`](https://github.com/SHA888/multiomics-rs).

---

## How this differs from MannLabs/CKG

`crkg` is a deliberate redesign after studying the Mann Lab's original Clinical Knowledge Graph. Differences:

| | MannLabs/CKG | `crkg` |
|---|---|---|
| Schema source | Cypher DDL + YAML builder config | LinkML YAML (emits Cypher, JSON Schema, Pydantic, Mermaid) |
| Schema versioning | Implicit, coupled to code | Independent package (`crkg-schema`), SemVer |
| Source parsers | 25 bespoke Python parsers, if/elif dispatch | Library consumes parsers from sibling projects (`clinical-rs`, `multiomics-rs`); adapter pattern, not dispatch |
| Usada / traditional medicine | Direct PDF parsing coupled to the graph | `TraditionalMedicineAdapter` protocol, data sourcing decoupled |
| Backend | Neo4j 4.2 hard-pinned | Neo4j 5.x (Community), with `GraphStore` Protocol for future backends |
| Python version | 3.7.9 pinned | 3.13 |
| Scope | Monolith: ETL + UI + analytics + reporting | Library only; no UI, no analytics, no reporting |
| Licensing discipline | MIT code, implicit inheritance of upstream data restrictions | Apache-2.0 code; zero data redistribution; adapter boundary for licensed corpora |

The goal is not to clone CKG; the goal is to produce the clinical-reasoning graph layer that CKG's architecture implied but never cleanly isolated.

---

## Architecture at a glance

```
    ┌──────────────────────────────────────────────────────┐
    │                      crkg                            │
    │                                                      │
    │   ┌────────────────┐      ┌──────────────────────┐   │
    │   │  Pydantic      │      │  Retrieval           │   │
    │   │  models (from  │      │  (graph + vector)    │   │
    │   │  crkg-schema)  │      │                      │   │
    │   └───────┬────────┘      └──────────┬───────────┘   │
    │           │                          │               │
    │   ┌───────┴──────────────────────────┴───────────┐   │
    │   │         GraphStore Protocol                  │   │
    │   │    (Neo4j impl in v0.x; Memgraph in v0.2+)   │   │
    │   └───────────────────┬──────────────────────────┘   │
    │                       │                              │
    │   ┌───────────────────┴──────────────────────────┐   │
    │   │    Ingestion (vocabularies, context)         │   │
    │   │    ICD-11 · SNOMED CT · LOINC · RxNorm ·     │   │
    │   │    Region epidemiology · Formulary           │   │
    │   └───────────────────┬──────────────────────────┘   │
    │                       │                              │
    │   ┌───────────────────┴──────────────────────────┐   │
    │   │    TraditionalMedicineAdapter Protocol       │   │
    │   │    ┌────────────────┐  ┌─────────────────┐   │   │
    │   │    │ Filesystem     │  │ External        │   │   │
    │   │    │ fallback impl  │  │ adapter (priv.) │   │   │
    │   │    └────────────────┘  └─────────────────┘   │   │
    │   └──────────────────────────────────────────────┘   │
    └──────────────────────────────────────────────────────┘
                              │
                              │ uses Pydantic models from
                              ▼
                   ┌──────────────────────┐
                   │    crkg-schema       │   Apache-2.0
                   │  (separate package)  │   independent versioning
                   └──────────────────────┘
```

Full architecture in [`ARCHITECTURE.md`](ARCHITECTURE.md).

## Relationship to sibling projects

- **[`crkg-schema`](https://github.com/balinesthesia/crkg-schema)** — required runtime dependency. Provides the data contract.
- **[`clinical-rs`](https://github.com/SHA888/clinical-rs)** — sibling Rust workspace. Not a runtime dependency. Consumers may feed `crkg` with Arrow data produced by `clinical-rs::mimic-etl`; the translation happens in the consuming application, not in `crkg`.
- **[`multiomics-rs`](https://github.com/SHA888/multiomics-rs)** — same relationship as `clinical-rs`. Not a dependency; a complementary data source.
- **[`Zluidr/hl7-rs`](https://github.com/Zluidr/hl7-rs)** — Rust HL7 v2 / FHIR / SATUSEHAT crates. Not a dependency. Consumers may use `hl7-rs` at the hospital ingestion boundary (MLLP listener → Arrow → Python) and populate `crkg` with the resulting clinical events.

---

## Installation (once published)

```bash
# Python, via pip
pip install crkg

# Or with uv
uv add crkg

# Neo4j Community Edition must be installed separately
# Docker:  docker run -p 7474:7474 -p 7687:7687 -e NEO4J_AUTH=neo4j/yourpass neo4j:5
```

## Usage sketch (v0.1+, not yet available)

```python
from crkg import ClinicalKG, GraphStoreConfig
from crkg.adapters import FilesystemTraditionalMedicineAdapter

# Connect to Neo4j
kg = ClinicalKG(
    graph=GraphStoreConfig(uri="bolt://localhost:7687", user="neo4j", password="..."),
    traditional_medicine_adapter=FilesystemTraditionalMedicineAdapter(path="./usada/"),
)

# Bootstrap schema constraints (emitted from crkg-schema)
kg.bootstrap()

# Load vocabularies
kg.load_icd11(path="./icd11-release.json")
kg.load_snomed(path="./SnomedCT_International/")

# Query
diseases = kg.diseases_with_symptom("fever")
```

---

## Versioning and stability

- **v0.0.x** — scaffold only. No graph code. No Neo4j integration. Just infrastructure.
- **v0.1.0** — first end-to-end working release. Consumes `crkg-schema>=0.1,<0.2`. Pre-stable; breaking changes permitted between minor versions.
- **v0.x.y** — iteration. Breaking changes require a CHANGELOG entry and a migration note.
- **v1.0.0** — stability commitment. Issued only after six months of real consumer usage and a full end-to-end run in a downstream CDSS.

Schema compatibility matrix will be maintained in [`docs/COMPATIBILITY.md`](docs/COMPATIBILITY.md) once multiple schema versions exist.

---

## Origin and migration note

The loaders, Cypher schema, and Qdrant sync pipeline in this repository originate from the `/mkg/` folder of a proprietary CDSS project (DeepReasoner, private). That code has been extracted here, re-licensed under Apache-2.0, restructured around `crkg-schema`, and had its traditional-medicine loader removed in favor of the `TraditionalMedicineAdapter` protocol.

- The extraction is documented in [`TODO.md §Phase 8 — Migration`](TODO.md).
- The original `/mkg/` is deleted from its source project in the same migration cycle that `crkg` v0.1.0 releases — no long-term duplicate code.
- Contributions from the original `/mkg/` are credited in [`NOTICE`](NOTICE) and git history.

---

## Documents

- [`README.md`](README.md) — this file
- [`ARCHITECTURE.md`](ARCHITECTURE.md) — layers, adapter contracts, graph store strategy, migration origin, design decisions
- [`TODO.md`](TODO.md) — atomic M0 tasks, dependency-ordered, with kill-gates
- [`LICENSE`](LICENSE) — Apache-2.0
- [`NOTICE`](NOTICE) — third-party attributions and origin note

---

## Version log

- **v0.0 (2026-04-19):** Initial scaffold. Documentation-only. M0 infrastructure starting.
-
