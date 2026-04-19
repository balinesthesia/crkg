# crkg — TODO

**Document version:** v0.0 (2026-04-19)
**SemVer target:** v0.0.1 → v0.0.x → v0.1.0 → v1.0.0
**Current phase:** M0 — Infrastructure-first
**Discipline:** SDLC-aligned, hierarchical task/subtask. Stable IDs. Kill-gates per phase.

---

## How this file works

- **Task IDs** are stable (`M0-T01`). Never renumbered. Removed tasks tombstoned in [Track Z](#track-z--tombstones).
- **Status**: `TODO`, `IN PROGRESS`, `DONE`, `BLOCKED`, `PARKED`.
- **Dependency**: task IDs that must complete first.
- **Architecture links** `[A§X.Y]` point into `ARCHITECTURE.md`.
- **SemVer bands:**
  - `v0.0.x` — infrastructure, no graph code
  - `v0.1.0` — first end-to-end working release
  - `v0.x.y` — iteration, breaking changes allowed
  - `v1.0.0` — stability commitment (post-M0)

---

## M0 — Infrastructure-first

**Goal:** A contributor can clone, run `make check`, see CI pass. Schema dependency wired. Graph store abstraction defined. Adapter protocol defined. No actual ingestion or graph writes in M0 — those begin in M1.

**Target completion:** v0.0.1 on TestPyPI with an importable Pydantic re-export, defined `GraphStore` Protocol, defined `TraditionalMedicineAdapter` Protocol, working CI, and documented migration plan.

---

## Phase 1 — Legal & Governance (target v0.0.1)

### M0-T01 — Commit `LICENSE` (Apache-2.0)
- [ ] **Status:** TODO
- **Dependency:** —
- **Notes:** Full Apache-2.0 text. Copyright line: `Copyright 2026 Kresna Sucandra and crkg contributors`.

### M0-T02 — Commit `NOTICE` with origin note
- [ ] **Status:** TODO
- **Dependency:** M0-T01
- **Notes:** Attribution for `neo4j` driver, Qdrant client, Pydantic, LinkML runtime (via crkg-schema), PyYAML. **Origin note**: code extracted from a proprietary CDSS's `/mkg/` folder, re-licensed under Apache-2.0 with the same copyright holder's consent. See [A§6].

### M0-T03 — Commit `SECURITY.md`
- [ ] **Status:** TODO
- **Dependency:** —
- **Notes:** Private disclosure process. Email placeholder.

### M0-T04 — Commit `CODE_OF_CONDUCT.md`
- [ ] **Status:** TODO
- **Dependency:** —

### M0-T05 — Add `.github/CODEOWNERS`
- [ ] **Status:** TODO
- **Dependency:** —
- **Notes:** `* @SHA888` for bootstrap.

### M0-T06 — Reserve `crkg` on PyPI
- [ ] **Status:** TODO
- **Dependency:** M0-T01
- **Notes:** Publish empty v0.0.0 placeholder. Also reserve on TestPyPI.

---

## Phase 2 — Python toolchain (target v0.0.1)

### M0-T10 — Create `pyproject.toml`
- [ ] **Status:** TODO
- **Dependency:** M0-T01
- **Notes:** Build backend `hatchling`. Python `>=3.13`. Core deps: `crkg-schema>=0.1,<0.2` (placeholder until schema publishes), `pydantic>=2.0`, `neo4j>=5.0`, `qdrant-client`. Dev deps: `pytest`, `pytest-cov`, `testcontainers`, `ruff`, `mypy`.

#### M0-T10.1 — Configure hatchling for `src/` layout
- [ ] Source dir `src/crkg`
- [ ] Package data: none — no bundled terminologies

#### M0-T10.2 — Configure ruff and mypy strict
- [ ] Strict mypy, no implicit Any
- [ ] Ruff: E, F, I, N, UP, B, SIM, ANN

### M0-T11 — Pin Python in `.python-version`
- [ ] **Status:** TODO
- **Dependency:** M0-T10
- **Notes:** `3.13`.

### M0-T12 — Commit `uv.lock`
- [ ] **Status:** TODO
- **Dependency:** M0-T10

### M0-T13 — Verify `uv sync` on cold clone
- [ ] **Status:** TODO
- **Dependency:** M0-T12
- **Notes:** Linux, macOS tested; Windows best-effort.

### M0-T14 — `.pre-commit-config.yaml`
- [ ] **Status:** TODO
- **Dependency:** M0-T10
- **Notes:** ruff, mypy, check-merge-conflict, trailing-whitespace, end-of-file-fixer, check-yaml, detect-private-key, gitleaks.

---

## Phase 3 — Schema dependency wiring (target v0.0.1)

### M0-T20 — Depend on `crkg-schema>=0.0.1`
- [ ] **Status:** BLOCKED
- **Dependency:** `crkg-schema` M0-T70 (crkg-schema v0.0.1 on TestPyPI)
- **Notes:** Pin via `[tool.uv.sources]` to TestPyPI for M0. Production PyPI pin lands when crkg-schema v0.1.0 ships.

### M0-T21 — `src/crkg/__init__.py`
- [ ] **Status:** TODO
- **Dependency:** M0-T10
- **Notes:** Exports `__version__`, re-exports nothing yet.

### M0-T22 — `src/crkg/models.py` — re-export from `crkg-schema`
- [ ] **Status:** BLOCKED
- **Dependency:** M0-T20
- **Notes:** `from crkg_schema.models import *` plus explicit `__all__`. Fails gracefully with a clear error if `crkg-schema` is not installed or incompatible.

### M0-T23 — Compatibility matrix doc
- [ ] **Status:** TODO
- **Dependency:** M0-T22
- **Notes:** `docs/COMPATIBILITY.md` lists `crkg` version ↔ `crkg-schema` version pairs. Matrix has one row at v0.0: `crkg 0.0.1` ↔ `crkg-schema >=0.0.1`.

---

## Phase 4 — GraphStore Protocol definition (target v0.0.1)

### M0-T30 — Define `crkg.graph.GraphStore` Protocol
- [ ] **Status:** TODO
- **Dependency:** M0-T22
- **Notes:** `typing.Protocol` with `bootstrap`, `upsert_node`, `upsert_edge`, `find_node`, `traverse`, `raw_cypher`. No implementation yet. Full docstrings.

#### M0-T30.1 — Protocol method signatures
- [ ] All methods take/return Pydantic models from `crkg.models`, never raw dicts

#### M0-T30.2 — Document thread-safety and transaction expectations
- [ ] Implementations must be safe to share across coroutines in the same event loop
- [ ] `bootstrap` is idempotent; running twice is a no-op

### M0-T31 — Stub `crkg.graph.neo4j.Neo4jGraphStore`
- [ ] **Status:** TODO
- **Dependency:** M0-T30
- **Notes:** Class shell implementing the Protocol. All methods raise `NotImplementedError`. Connection config accepted in `__init__`. Tests confirm the class satisfies the Protocol statically.

#### M0-T31.1 — Verify Protocol conformance via `mypy --strict`
- [ ] Structural check: `Neo4jGraphStore` is assignable to `GraphStore`

---

## Phase 5 — TraditionalMedicineAdapter Protocol (target v0.0.1)

### M0-T40 — Define `crkg.adapters.TraditionalMedicineAdapter` Protocol
- [ ] **Status:** TODO
- **Dependency:** M0-T22
- **Notes:** Methods `list_entities()`, `get_entity(id)`, `provenance()`. Full docstrings. Returns `crkg_schema.models.EthnobotanyEntity` (once schema lands).

### M0-T41 — Implement `NullTraditionalMedicineAdapter`
- [ ] **Status:** TODO
- **Dependency:** M0-T40
- **Notes:** No-op adapter. `list_entities()` yields nothing. `get_entity(id)` returns `None`. Default wiring. Useful as a test double and as the zero-config option.

### M0-T42 — Stub `FilesystemTraditionalMedicineAdapter`
- [ ] **Status:** TODO
- **Dependency:** M0-T40
- **Notes:** Class shell. Accepts a directory path. All methods raise `NotImplementedError`. Full implementation deferred to M1.

### M0-T43 — `docs/ADAPTERS.md`
- [ ] **Status:** TODO
- **Dependency:** M0-T40
- **Notes:** How to write a `TraditionalMedicineAdapter`. Example: a proprietary corpus might wrap its own API; a public corpus might wrap a GitHub release. No named corpus in this document — keep it generic.

---

## Phase 6 — CI/CD (target v0.0.1)

### M0-T50 — `.github/workflows/ci.yml`
- [ ] **Status:** TODO
- **Dependency:** M0-T10, M0-T14, M0-T30, M0-T40
- **Notes:** Jobs: `lint` (ruff, mypy), `test-unit` (pytest, no external services), `test-protocol-conformance` (Protocol vs stub impl check), `build` (uv build). All required before merge.

### M0-T51 — `.github/workflows/release.yml`
- [ ] **Status:** TODO
- **Dependency:** M0-T50
- **Notes:** Triggered on tag `v*`. Full CI, sdist + wheel, GitHub Release, publish to PyPI via trusted publisher.

### M0-T52 — Branch protection on `main`
- [ ] **Status:** TODO
- **Dependency:** M0-T50
- **Notes:** Require CI green, signed commits, PR review.

### M0-T53 — `cargo install cargo-skill` note in dev docs
- [ ] **Status:** PARKED
- **Dependency:** —
- **Notes:** Not applicable — `crkg` is Python-only. If a future Rust companion crate materializes, re-open.

---

## Phase 7 — Documentation (target v0.0.1)

### M0-T60 — `CONTRIBUTING.md`
- [ ] **Status:** TODO
- **Dependency:** M0-T50

### M0-T61 — `CHANGELOG.md`
- [ ] **Status:** TODO
- **Dependency:** —
- **Notes:** Keep-a-changelog format.

### M0-T62 — `docs/DEV_SETUP.md`
- [ ] **Status:** TODO
- **Dependency:** M0-T13
- **Notes:** Cold-start to green CI on first PR within a working day.

### M0-T63 — `docs/MIGRATION_FROM_MKG.md`
- [ ] **Status:** TODO
- **Dependency:** —
- **Notes:** Historical note. Documents origin, what moved, what was removed, who to contact about the originating project. Non-technical section of the migration narrative; the technical steps live in Phase 8.

### M0-T64 — PR and Issue templates
- [ ] **Status:** TODO
- **Dependency:** —

---

## Phase 8 — Migration from `/mkg/` (target v0.1.0, NOT v0.0.1)

> **Critical**: migration is the transition from M0 to M1. It does NOT happen in the scaffold release (v0.0.1). It happens when the infrastructure is ready to receive real code. Six ordered steps, each independently testable and reversible.

### M0-T70 — Step 1: Audit existing `/mkg/` code in origin project
- [ ] **Status:** BLOCKED
- **Dependency:** M0-T63
- **Notes:** Not a task for this repo — a task for the origin project. See `CROSS-REPO-TASKS.md` DeepReasoner row. Output: a `MKG_INVENTORY.md` in the origin repo listing every file, its test coverage, and its readiness for extraction.

### M0-T71 — Step 2: Strengthen `/mkg/` test suite in origin
- [ ] **Status:** BLOCKED
- **Dependency:** M0-T70
- **Notes:** Not a task for this repo. Coverage target: ≥ 70% on each loader before extraction. This is the expensive step. Do not skip.

### M0-T72 — Step 3: Copy code into `crkg` and re-test in situ
- [ ] **Status:** BLOCKED
- **Dependency:** M0-T71
- **Notes:**
  - Copy `/mkg/schema/*.cypher` → `crkg-schema/schema/` (converted to LinkML)
  - Copy `/mkg/ingestion/icd11_loader.py` → `crkg/src/crkg/ingestion/icd11.py`
  - Copy `/mkg/ingestion/snomed_loader.py` → `crkg/src/crkg/ingestion/snomed.py`
  - Copy `/mkg/ingestion/kemenkes_endemic.py` → `crkg/src/crkg/ingestion/region_epi.py`
  - Copy `/mkg/ingestion/fornas_loader.py` → `crkg/src/crkg/ingestion/formulary.py`
  - **Do NOT copy** `/mkg/ingestion/usada_loader.py` — replaced by adapter
  - Copy `/mkg/vector/embedding_pipeline.py` → `crkg/src/crkg/vector/embedding.py`
  - Copy `/mkg/vector/qdrant_sync.py` → `crkg/src/crkg/vector/qdrant.py`
  - Re-run origin test suite against copied code; must pass before merging.

### M0-T73 — Step 4: Publish `crkg` v0.1.0 with migrated code
- [ ] **Status:** BLOCKED
- **Dependency:** M0-T72

### M0-T74 — Step 5: Origin depends on `crkg>=0.1,<0.2` and deletes `/mkg/`
- [ ] **Status:** BLOCKED
- **Dependency:** M0-T73
- **Notes:** Single PR in origin repo. Add `crkg` dependency, rewrite imports, delete `/mkg/`. Origin test suite must pass with imports now coming from `crkg`. This is the riskiest PR — guarded by the strengthened test suite from Step 2.

### M0-T75 — Step 6: Origin wires to `FilesystemTraditionalMedicineAdapter`
- [ ] **Status:** BLOCKED
- **Dependency:** M0-T74, M1 completion of `FilesystemTraditionalMedicineAdapter` (M1 task TBD)
- **Notes:** Origin's previous `usada_loader.py` path is replaced by a filesystem adapter reading from the same files. When the external proprietary adapter ships, origin swaps adapter implementation via config. No further origin code change.

---

## Phase 9 — Release v0.0.1 (target: M0 kill-gate)

### M0-T80 — Tag and release v0.0.1 to TestPyPI
- [ ] **Status:** TODO
- **Dependency:** all Phases 1–7
- **Notes:** First tagged release. TestPyPI only (not production PyPI). Scaffold content: Protocols defined, stubs raising NotImplementedError, no functional graph code. Verifies the package is importable and CI is green end-to-end.

### M0-T81 — Verify `pip install -i https://test.pypi.org/simple/ crkg` works
- [ ] **Status:** TODO
- **Dependency:** M0-T80
- **Notes:** Fresh venv; `import crkg; from crkg.graph import GraphStore; from crkg.adapters import TraditionalMedicineAdapter` all succeed.

---

## M0 Kill-criteria

All of the following must hold before M1 begins:

1. `LICENSE` (Apache-2.0) and `NOTICE` (with origin note) committed.
2. `pyproject.toml`, `uv.lock`, `.python-version` pinning 3.13 committed.
3. `crkg-schema>=0.0.1` dependency wired; `import crkg.models` succeeds.
4. `GraphStore` Protocol defined with full docstrings; `Neo4jGraphStore` stub passes Protocol conformance check.
5. `TraditionalMedicineAdapter` Protocol defined; `NullTraditionalMedicineAdapter` implemented; `FilesystemTraditionalMedicineAdapter` stubbed.
6. CI green on `main` for all jobs.
7. `crkg` reserved on PyPI (empty v0.0.0) and v0.0.1 published to TestPyPI.
8. `docs/DEV_SETUP.md` validated by a cold-start contributor reaching green CI on their first PR within a working day.
9. `CROSS-REPO-TASKS.md` (in this conversation's artifacts) reviewed and signed off by stakeholders of every affected repo.

> **Critical:** Phase 8 (migration) tasks M0-T70..M0-T75 are NOT required for M0 kill-gate. They are sequenced to execute across the M0 → M1 boundary. The scaffold releases (v0.0.1) with stubs only; real migration happens before v0.1.0.

---

## M1 — First end-to-end release (deferred detail)

Not in scope for this document. Sketch:

- Phase 1: Neo4j driver full implementation — target v0.1.0-alpha
- Phase 2: ICD-11 loader — migrated from origin, tested against origin fixtures
- Phase 3: SNOMED CT loader — same pattern
- Phase 4: LOINC, RxNorm, ATC loaders — new, not in origin
- Phase 5: Regional epidemiology and formulary loaders — migrated
- Phase 6: Vector embedding + Qdrant sync — migrated
- Phase 7: Hybrid retrieval — migrated
- Phase 8: `FilesystemTraditionalMedicineAdapter` full implementation
- Phase 9: Integration test suite with testcontainers
- Phase 10: Migration steps M0-T70..M0-T75 execution
- Phase 11: Release v0.1.0 to production PyPI

**Gate to M1:** M0 kill-criteria met AND `crkg-schema` v0.1.0 released with enough schema content to type the loaders.

---

## M2 — Optional backends and polish (deferred)

- Memgraph `GraphStore` implementation (v0.2.0)
- Performance optimization against non-functional targets
- Compatibility matrix coverage beyond v0.1
- Additional loaders as consuming applications need them

---

## Track Z — Tombstones

Reserved.

| ID | Original Task | Reason for Removal | Date Removed |
|---|---|---|---|
| — | — | — | — |

---

## Parked ideas

| Idea | Why parked | Likely milestone |
|---|---|---|
| Kùzu backend | Project abandoned October 2025 | Dropped from scope; re-open only if `bighorn` or `Ladybug` fork reaches v1.0 |
| Rust companion crate | No consumer demand yet | When a Rust consumer materializes |
| RxNorm loader expansion (RRF + MED-RT) | Not needed by v0.1 consumers | M1 Phase 4 depending on consumer needs |
| HL7 v2 / FHIR parsers | Out of scope (D-14) | Never — consuming apps handle hospital ingress |
| Audit logging framework | Belongs to consuming CDSS | Never |
| LLM prompt templates | Belongs to consuming CDSS | Never |

---

## Version log

| Version | Date | Description |
|---|---|---|
| v0.0 | 2026-04-19 | Initial TODO. M0 phases 1–9 defined. Migration (phase 8) cross-references `CROSS-REPO-TASKS.md`. M1/M2 sketched. |
