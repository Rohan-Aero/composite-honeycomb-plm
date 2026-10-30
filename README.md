# composite-honeycomb-plm

PLM-inspired engineering data management for the composite honeycomb sandwich panel FEA study. The tool loads the study into a
relational SQLite database, keeps every value traceable to its source file (path, archive member, line or PDF page, SHA-256),
checks data quality, and answers questions about the three face-sheet configurations with documented SQL.

The study itself (ANSYS files, README, report, images, and the `data/` inputs) lives in the
[FEA repository](https://github.com/Rohan-Aero/composite-honeycomb-structural-optimization-FEA). This repository holds only the tool.
It reads the study from a checkout of that repository and never modifies its source files.

> **What is real and what is not.** Simulation inputs, solver outputs and the documented results come from the FEA repository.
> The BOM, part numbers (`DEMO-...`), design revisions and the engineering change request are a **demonstration** workflow created
> by `python -m plm demo`. They are not records of a real PLM system and nothing was approved by any organisation. Derived values
> (percent changes, stiffness proxy) are calculated by queries and labelled as such. Values that are not available are stored as *missing*.

## Requirements

- Python 3.9+ with its standard library only (SQLite 3.25+ for window functions).
- A checkout of the FEA repository. Point `PLM_SOURCE_ROOT` at it:

```bash
git clone https://github.com/Rohan-Aero/composite-honeycomb-structural-optimization-FEA ../composite-honeycomb-structural-optimization-FEA
export PLM_SOURCE_ROOT=../composite-honeycomb-structural-optimization-FEA
```

- Optional: `pdftotext` (poppler). It enables the check that each transcribed report value really appears on its cited PDF page.

## Quick start

```bash
python -m plm all                    # init (rebuild) -> import -> demo -> validate -> report
python -m unittest discover -v       # 80 tests, all pass on the committed data
```

Or step by step. `--db FILE` and `--root DIR` are accepted by every command; `--root` defaults to `$PLM_SOURCE_ROOT`.

```bash
python -m plm init [--force]         # create build/honeycomb_plm.sqlite from plm/schema.sql
python -m plm import                 # ANSYS archives + README + report values -> database (sources are read-only)
python -m plm demo                   # DEMONSTRATION BOM, revisions A/B, change request DEMO-ECR-0001
python -m plm validate               # data-quality checks; exit status 1 if any error
python -m plm report [--out DIR]     # Markdown, CSV and SVG charts
python -m plm query sql/01_compare_thickness_configs.sql
```

- `init` never overwrites an existing database without `--force`. The database lives in `build/` and is git-ignored.
- `import` refuses to run twice and rejects a malformed input file as a whole instead of repairing it.
- Commands that read sources exit with status 2 when no source repository is given.
- `report` writes by default to `<FEA checkout>/generated_reports/`, which overwrites the sample outputs committed there. Pass `--out` to write elsewhere.

## What is imported, and from where

All paths are relative to the FEA checkout (`PLM_SOURCE_ROOT`).

| Data | Source file | Notes |
|---|---|---|
| Configurations 0.85 / 1.2 / 1.6 mm | `Simulation Files (ANSYS)/<t>mm/` | thickness also confirmed from solver mass |
| Solver, run time, mesh, mass, warnings | `solve.out` in each `.wbpz` | ANSYS MAPDL 2025 R1, build 25.1 |
| Materials, load, fixed support | `ds.dat` + `MatML.xml` in each `.wbpz` | PLA core, ePA-CF face sheets; load = pressure x computed area |
| Max displacement component | `file.mntr` | cross-check on the documented deformation |
| Deformation, stress, strain | `README.md` table and report Table 6 | report values are page-referenced in `data/report_values.csv` |
| Demonstration BOM | `data/demo_bom.csv` | demonstration data only |
| Every file (98) | SHA-256 + size, including all members of the `.wbpz` archives | `source_file` table |

Each result, input and material property stores its source file, archive member, line or table locator, the **value and unit as
printed**, and the canonical value (e.g. report stress `2989600 Pa` becomes `2.9896 MPa`). The raw binary results (`file.rst`,
`.mechdb`) are not parsed, so directional deformation, strain energy and volume (available only as contour images) are recorded as **missing**.

## Schema

21 tables and one view, foreign keys enforced, a uniqueness constraint on every natural key, and indexes on the main join and
provenance columns. The full description is in the FEA repository at `docs/schema.md`.

```
project -> product -> configuration -> simulation_case -> simulation_input / result_metric / material_property
                          |                  |                              \-> source_file (file, member, line, SHA-256)
                          |                  \-> solver
                          +-> bom_line <- part                               (demonstration BOM)
                          +-> design_revision <- change_request -> ecr_item  (demonstration revisions / ECR, linked to cases)
quantity_def (canonical units)   artifact_link (file roles)   validation_run -> validation_issue
```

## Results from the available data

Documented ANSYS results with derived percent changes versus 0.85 mm (`sql/01_compare_thickness_configs.sql`):

| Config | Thickness (mm) | Deformation (mm) | Stress (MPa) | Strain | Deformation vs 0.85 mm | Stress vs 0.85 mm |
|---|---|---|---|---|---|---|
| T085 | 0.85 | 0.0070107 | 2.9896 | 0.00089581 | - | - |
| T120 | 1.2 | 0.006009 | 2.9189 | 0.00087613 | -14.29 % | -2.36 % |
| T160 | 1.6 | 0.0055584 | 2.8536 | 0.00085723 | -20.72 % | -4.55 % |

Ranking (`sql/02_rank_configurations.sql`): 1.6 mm is best for deformation, stress and strain; 0.85 mm is the lightest; 1.2 mm is best
for the derived specific stiffness F/(deformation x mass). Indicative only: it assumes equal load and uses the maximum deformation.

## Data-quality checks

`python -m plm validate` runs 42 check codes (missing values, duplicate IDs, wrong types, units, invalid parameters, broken links,
comparability, cross-source consistency, BOM/ECR integrity, source-file hashes). Each finding is an **error**, **warning** or
**info**, with the affected record and a suggested correction. Source data is never modified.

On the committed FEA data the current run gives **0 errors, 11 warnings, 14 info**.

| Finding (warning) | Detail |
|---|---|
| Duplicate file content | the loose `0.85mm/Main file/WORKBENCH N MECH.wbpj` is byte-identical to the `.wbpj` inside the **1.6 mm** archive |
| Report vs solver material data | report Table 4 (flexural modulus, density) differs from the solver deck (E, density) |
| Report Table 5 labels | core shown as "PLA -CF" and face sheet as "PLA"; README and solver deck have a PLA core and ePA-CF face sheets |
| Unit label | report Table 6 labels equivalent elastic strain with "(mm)"; strain is dimensionless |
| README claim | "improved stiffness-to-weight ratio significantly" is not supported by F/(deformation x mass): -2.3 % from 0.85 to 1.6 mm |

These checks *confirm* consistency; they do not prove the results are physically valid. The FEA repository has no experimental data
and no mesh-convergence study.

## Example SQL

```sql
-- Documented total deformation per configuration, best first (v_result_primary: one value per case/metric)
SELECT c.case_code, r.metric_name, r.value, r.unit, r.origin
FROM v_result_primary r JOIN simulation_case c USING (case_id)
WHERE r.metric_name = 'total_deformation' ORDER BY r.value;

-- Trace a value to its source file, archive member, line and hash
SELECT r.metric_name, r.value, r.unit, sf.rel_path, sf.archive_member, r.locator, substr(sf.sha256, 1, 12) AS sha256
FROM result_metric r JOIN source_file sf USING (source_file_id) JOIN simulation_case c USING (case_id)
WHERE c.case_code = 'SIM-T085' AND r.metric_name = 'total_mass';

-- Missing data (never estimated)
SELECT c.case_code, r.metric_name, r.missing_reason
FROM result_metric r JOIN simulation_case c USING (case_id) WHERE r.status = 'missing' ORDER BY c.case_code, r.metric_name;
```

## SQL files

| File | Purpose |
|---|---|
| `01_compare_thickness_configs.sql` | deformation / stress / strain by thickness, % change vs baseline |
| `02_rank_configurations.sql` | ranks by deformation, stress, strain, mass, specific stiffness |
| `03_summary_by_solver_material.sql` | available / missing results by solver and material |
| `04_missing_data.sql`, `05_quality_issues.sql` | missing results; findings of the latest validation run |
| `06_revision_tracking.sql` | revisions -> configuration -> ECR -> simulation case -> results (demonstration) |
| `07_cross_solver_comparison.sql` | cross-solver comparison, only where corresponding results exist (none with one solver) |
| `08_traceability.sql` | configuration -> inputs / outputs -> source files, locators, SHA-256 |
| `09_bom_by_configuration.sql`, `10_change_requests.sql`, `11_change_request_history.sql` | demonstration BOM, ECR with verification results, status history |

Every file is documented in its header and runs with `python -m plm query <file>` or any SQLite client.

## Tests

`python -m unittest discover -v` runs 80 tests covering the database, import, checks with injected errors, the workflow, the queries,
the reports and the CLI pipeline. The tests use the real FEA data, so `PLM_SOURCE_ROOT` must be set. Without it, the test package
stops at import with a message saying so.

## Known issues

- **Report source.** The importer reads the report that `data/report_values.csv` cites, named by `REPORT_PDF` in `plm/checks.py` (currently `Report/MAJOR PROJECT REPORT (1).pdf`). The two Appendix A PDFs in `Report/` are kept and registered as source files, but are not transcribed.
- Only **one solver** (ANSYS MAPDL 2025 R1) and one run per configuration exist, so cross-solver comparison returns no rows. The query
  is tested with clearly labelled synthetic data only.
- Raw nodal results (`file.rst`, `SYS.mechdb`) are not parsed. Stress, strain and deformation are the documented values, cross-checked
  against solver outputs, not re-extracted.
- The report is a PDF. Report Tables 4–6 are transcribed once into `data/report_values.csv` with page numbers. `pdftotext` checks only
  that each value appears on its cited page, not the layout.
- The mesh is identical in all cases and there is no mesh-convergence study, so mesh sensitivity cannot be assessed.
- Face-sheet thickness comes from folder names and is confirmed through solver mass, which assumes a solid plate of the loaded-face area.
  The SpaceClaim geometry files are not parsed.
- BOM, revisions and ECR are demonstration data with a deliberately small status model: no approval authority, no users, no permissions.

## Layout

```
plm/            schema.sql, db.py, ingest.py, checks.py, workflow.py, report.py, cli.py, units.py
sql/            11 documented SQL queries
tests/          unittest suite (run against the FEA checkout in PLM_SOURCE_ROOT)
build/          generated database (git-ignored)
```

Origin: extracted from the FEA repository (`plm/`, `sql/`, `tests/`) at commit `fdc4bf0`. The history of those files stays in the FEA repository.
