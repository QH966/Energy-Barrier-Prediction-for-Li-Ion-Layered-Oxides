# Layered Lithium Oxide: SQS Generation and Migration Barrier Prediction

An end-to-end, command-line workflow for **generating special quasirandom structures (SQS)** of layered lithium-containing oxides and **screening in-plane lithium migration barriers** after structural relaxation.

The workflow consists of two supplied Python tools connected by a **user-performed VASP structural relaxation** step. SQS generation does not predict migration barriers, and the prediction tool does not perform VASP relaxation.

```text
Chemical formula(s)
       |
       v
[1] SQS structure generation (Linux / HPC)
    SQSINCAR -> run.sh -> ranked POSCAR structures
       |
       v
[2] VASP structural relaxation (performed by the user)
    Selected POSCAR -> VASP relaxation -> relaxed CONTCAR
       |
       v
[3] Migration barrier prediction (local CLI)
    CONTCAR files -> 8 descriptors -> trained CatBoost model
       |
       v
    Ranked CSV / Excel predictions
```

**Scientific scope:** The predictor estimates *in-plane Li migration barriers* for candidate paths. Predictions are screening estimates, **not** nudged elastic band (NEB) calculations, direct VASP migration-barrier calculations, or experimental measurements.

## Workflow at a Glance

| Stage | Input | Operation | Output |
| --- | --- | --- | --- |
| 1. Generate SQS | Chemical formulas in `SQSINCAR` | Sample and rank SQS candidates | `POSCAR` files and SQS rankings |
| 2. Relax structures | Selected generated `POSCAR` files | User runs VASP structural optimization | Relaxed `CONTCAR` files |
| 3. Predict barriers | Relaxed `CONTCAR` files | Enumerate in-plane Li paths, extract eight features, predict with CatBoost | Ranked `.csv` or `.xlsx` results |

The two Python components have **different runtime requirements**. Prepare each environment according to its section below; do not assume that the predictor's `requirements.txt` contains all SQS dependencies.

## Stage 1 — SQS Structure Generation

### 1.1 Requirements

The portable SQS generator is pure Python and does **not** require `pip install -e .`. It requires an existing environment with:

- Python >= 3.10 (3.11 recommended)
- NumPy
- ASE
- pymatgen
- icet 3.2.x
- spglib
- PyYAML

Activate an already configured Conda, venv, or module environment:

```bash
conda activate li-sqs
python -c "import numpy, ase, pymatgen.core, icet, spglib, yaml; print('Environment OK')"
```

The bundled `portable_runner.py` prepends the package's `src` directory to `sys.path`. The runner and scheduler scripts use the bundled source and `resources/POSCAR_LiNiO2`.

### 1.2 Extract and Prepare

```bash
unzip li_layered_sqs_portable.zip
cd li_layered_sqs_server
chmod +x run.sh scripts/*.sh
```

No installation of the SQS package itself is required.

### 1.3 Specify Chemical Formulas

Edit `SQSINCAR`. Repeat `FORMULA` lines to process multiple compositions with a shared `N_STRUCTURES` value:

```ini
FORMULA = LiNi0.83Mn0.13Mg0.01Ca0.01Ti0.01Zr0.01O2
FORMULA = LiNi0.8Mn0.1Co0.1O2
N_STRUCTURES = 5
```

Alternatively, put comma-separated formulas on one line:

```ini
FORMULA = LiNi0.83Mn0.13Mg0.01Ca0.01Ti0.01Zr0.01O2, LiNi0.8Mn0.1Co0.1O2
```

Duplicate formulas after normalization are rejected. Each formula receives its own normalized-formula output directory and formula-specific `SQSINCAR.snapshot`.

Inspect the planned workload before running:

```bash
./run.sh -i SQSINCAR --dry-run --workers 2
```

Dry-run information includes `formula_count`, `jobs`, and each formula's `quality`, `n_steps`, `target_unique_pool`, `max_sqs_runs`, and `seed`.

### 1.4 Quality Presets

The default preset is `high`.

| Preset | Steps | Relative cost | Suggested use |
| --- | ---: | --- | --- |
| `standard` | 60,000 | Lower | Initial exploration |
| `high` | 200,000 | Higher | Recommended default |
| `ultra` | 500,000 | Highest | More intensive sampling |

Candidate-pool targets are adaptive:

```text
max(N_STRUCTURES * oversample_factor, min_unique_pool)
```

The pool is also limited by `max_sqs_runs`; requesting five final structures does not automatically trigger 500 candidate runs. Computational cost depends on composition and duplicate rejection.

All presets retain the same scientific definition: **fixed supercell, pair-only correlations through 3NN, and no triplet clusters**.

```bash
./run.sh -i SQSINCAR --dry-run --quality standard
./run.sh -i SQSINCAR --dry-run --quality high
./run.sh -i SQSINCAR --dry-run --quality ultra
```

`--n-steps` is an explicit override intended for testing or specialized runs. Overrides are included in the checkpoint parameter hash.

### 1.5 Run, Resume, or Restart

Generate SQS structures using 16 CPU worker processes:

```bash
./run.sh -i SQSINCAR --workers 16 --quality high
```

Specify a base random seed:

```bash
./run.sh -i SQSINCAR --workers 16 --seed 12345
```

Multiple formulas receive stable formula-index seed offsets to prevent collisions. The seeds are recorded in dry-run information, summaries, and checkpoint parameters. Formulas execute in input order; the first failure stops the run by default, with the formula index and name reported.

Run in the background:

```bash
nohup ./run.sh -i SQSINCAR --workers 16 --quality high > li_sqs.log 2>&1 &
tail -f li_sqs.log
```

Resume from compatible checkpoints:

```bash
./run.sh -i SQSINCAR --workers 16 --quality high --resume
```

Formulas with compatible existing checkpoints resume, while later formulas without output directories are started normally.

To clear matching old results and regenerate:

```bash
./run.sh -i SQSINCAR --workers 16 --quality high --overwrite
```

**Warning:** `--resume` and `--overwrite` are mutually exclusive. Changing quality, step count, candidate-pool settings, seed, or other scientific/engineering parameters invalidates checkpoint compatibility.

To use a nondefault Python interpreter:

```bash
PYTHON_BIN=/path/to/python ./run.sh -i SQSINCAR --workers 16
```

### 1.6 HPC Scheduler Submission

Bundled scripts support single-node, multiprocess execution on SLURM, PBS, and LSF:

```bash
sbatch scripts/submit_slurm.sh
qsub scripts/submit_pbs.sh
bsub < scripts/submit_lsf.sh
```

The default quality is `high`; it can also be added explicitly to the script arguments. **This program is not MPI-based: request one node.**

### 1.7 SQS Outputs

Each normalized-formula directory contains:

```text
<formula-directory>/
├── 1/
│   ├── POSCAR
│   └── metadata.json
├── 2/
│   ├── POSCAR
│   └── metadata.json
├── ...
├── ranking.csv
├── run_summary.json
├── generation.log
├── SQSINCAR.snapshot
└── .work/
```

- `1/POSCAR`, `2/POSCAR`, etc.: generated candidate structures.
- `ranking.csv`: candidates ranked by ascending SQS Q.
- `run_summary.json`: formula-specific status and statistics.
- `generation.log`: formula-specific log.
- `SQSINCAR.snapshot`: input snapshot containing that formula.
- `*/metadata.json`: per-structure composition and parameters.
- `.work/`: candidate and checkpoint/resume files.

The CLI also reports overall `status`, `formula_count`, and a summary for each formula.

**Interpretation:** Lower SQS Q means cluster correlations more closely approach the target random solid solution. **SQS Q is not VASP energy**, and a low Q does not establish structural or thermodynamic stability.

## Stage 2 — Relax Generated Structures with VASP

**This is a required handoff performed by the user, not an automated step of either Python tool.**

1. Select one or more generated `POSCAR` candidates from Stage 1, using `ranking.csv` and any additional scientific criteria.
2. Use each selected `POSCAR` as the starting structure for an appropriate VASP structural relaxation.
3. Run and assess the relaxation using your own validated VASP settings and convergence criteria.
4. Retain the resulting relaxed **`CONTCAR`** for Stage 3.

For example, organize relaxed outputs as follows:

```text
relaxed_structures/
├── composition_A/
│   ├── candidate_1/
│   │   └── CONTCAR
│   └── candidate_2/
│       └── CONTCAR
└── composition_B/
    └── candidate_1/
        └── CONTCAR
```

The predictor discovers files named exactly `CONTCAR`, including files in nested directories. **Do not pass unrelaxed SQS `POSCAR` files as though they were relaxed `CONTCAR` results.** The supplied documentation does not prescribe a universal VASP INCAR/KPOINTS/POTCAR recipe or automatically transfer files between stages; those choices and file organization remain the user's responsibility.

## Stage 3 — Predict In-Plane Li Migration Barriers

The second tool recursively finds relaxed `CONTCAR` structures, enumerates in-plane Li–Li candidate paths, reduces symmetry-equivalent paths, groups similar local environments, extracts eight descriptors, and applies the **supplied trained CatBoost model**.

It requires no notebook, server, account, GPU, or retraining.

### 3.1 Predictor Files

The supplied predictor project is documented with the following layout:

```text
contcar-predict-github/
├── predict.py
├── feature_extractor.py
├── requirements.txt
├── README.md
├── .gitignore
├── model/
│   ├── catboost_best_model.cbm
│   └── model_metadata.json
└── examples/
    └── sample/
        └── CONTCAR
```

`feature_extractor.py` and the model are preserved from the original scientific workflow. The metadata retains original feature names, order, model parameters, and recorded metrics; a private local training-data path was removed. Some original extractor comments and diagnostics remain in Chinese. **Run `predict.py`, not the legacy standalone entry point in `feature_extractor.py`.**

### 3.2 Predictor Environment

Python **3.12** is recommended to reproduce the documented validated environment. Other Python versions and operating systems have not been validated for this predictor package.

The supplied predictor `requirements.txt` contains:

```text
catboost==1.2.8
numpy>=2.0,<3.0
pandas>=2.2,<3.0
pymatgen>=2025.1
openpyxl>=3.1,<4.0
```

From the predictor directory, install dependencies in a dedicated virtual environment.

**Linux / macOS**

```bash
python3 -m venv .venv
.venv/bin/python -m pip install --upgrade pip
.venv/bin/python -m pip install -r requirements.txt
.venv/bin/python predict.py examples -o results.csv
```

**Windows PowerShell**

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install --upgrade pip
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe predict.py examples -o results.csv
```

The remaining examples assume `python` refers to the activated predictor environment.

### 3.3 Predict Relaxed Structures

From the predictor project directory, process a directory recursively:

```bash
python predict.py "path/to/relaxed_structures" -o results.csv
```

Export an Excel workbook:

```bash
python predict.py "path/to/relaxed_structures" -o results.xlsx
```

Predict one relaxed structure:

```bash
python predict.py "path/to/relaxed_structures/composition_A/candidate_1/CONTCAR" -o single.csv
```

Run the included example:

```bash
python predict.py examples -o example_predictions.csv
```

Show CLI help:

```bash
python predict.py --help
```

When `-o` is omitted, a timestamped file named `predictions_YYYYMMDD_HHMMSS_microseconds.csv` is created in the current working directory. Model and metadata paths are resolved relative to `predict.py`.

**Existing output files are never overwritten.** Use a new output filename for each run.

The only optional scientific parameter exposed by the wrapper is the maximum Li–Li candidate distance:

```bash
python predict.py "path/to/relaxed_structures" --max-distance 4.0 -o results.csv
```

The default is **4.0 Å**. Change it only with physical justification and revalidate the results. Changing the distance does not retrain the model.

### 3.4 Input Discovery

Only files named exactly `CONTCAR` (case-sensitive) are discovered; the wrapper does not automatically accept `POSCAR`, CIF, compressed structures, or arbitrary extensions.

```text
relaxed_structures/
├── CONTCAR
├── family_A/
│   └── relaxed/
│       └── CONTCAR
└── family_B/
    └── relaxed/
        └── CONTCAR
```

All three structures are processed. For directory input, identifiers are relative directory paths (`.`, `family_A/relaxed`, `family_B/relaxed`), preventing collisions between identically named leaf folders. For a single file, the parent directory name is used.

Hidden directories and `__pycache__`, `venv`, `env`, `node_modules`, and `catboost_info` are skipped; child symbolic links are not followed. Identical structures stored at different paths are separate inputs.

The wrapper copies structures into an automatically cleaned temporary directory. It does not modify original `CONTCAR` files or save intermediate features/logs beside them.

### 3.5 Prediction Outputs

CSV contains all predicted path groups, encoded as UTF-8 with BOM. Excel contains:

- `predictions`: all predicted path groups.
- `lowest_barriers`: the first-ranked (lowest predicted barrier) group for each structure.

| Column | Description |
| --- | --- |
| `structure` | Relative directory identifier, or parent name for single-file input |
| `path` | Structure identifier plus `_pathN` |
| `path_id` | Original grouped path number within a structure |
| `rank_in_structure` | Ascending predicted-barrier rank, starting at 1; ties use `path_id` |
| `predicted_barrier_eV` | Predicted migration barrier, in eV according to the source workflow |
| `source_file` | Absolute path to the original `CONTCAR` |
| Eight model feature columns | Original model input values and Unicode feature names |

**The original feature names and their order are preserved exactly.** The wrapper checks that the metadata, mapping, and loaded model agree.

### 3.6 The Eight Model Descriptors

| Model feature | Original extraction field | Meaning |
| --- | --- | --- |
| `A_ab` | `a_times_b` | Lattice parameter product a × b (Å²), not a × b × sin(γ) |
| `R_Li/TM` | `Li_to_transition_metal_ratio` | Li-to-transition-metal count ratio |
| `d_OO` | `d_OO_bottleneck_opposite` | Oxygen bottleneck distance (Å) |
| `∆d_O` | `O_bottleneck_asymmetry` | Absolute difference between midpoint-to-gate-oxygen distances (Å) |
| `M_cat` | `wmean_cation_mendeleev_3p5` | Inverse-distance-weighted local cation Mendeleev number |
| `σ_LiO,end` | `distortion_LiO_end` | Standard deviation / mean of endpoint Li–O distances for up to four nearest O sites |
| `d̄_MO,6` | `mean_M_O_6nearest` | Mean midpoint-to-O distance for up to six nearest O sites (Å) |
| `d_LiLi` | `d_LiLi_inplane` | In-plane Li–Li path length (Å) |

### 3.7 Preserved Path-Extraction Workflow

The original extractor enumerates Li-site pairs with periodic boundary conditions and a default **0.30–4.00 Å** distance window. Candidate in-plane paths must satisfy all of the following:

- Absolute c-axis projection ≤ 0.80 Å.
- c-axis projection / path distance ≤ 0.25.
- Fractional c-coordinate displacement ≤ 0.08.

Space-group symmetry reduction is enabled with `symprec=0.001` and an angle tolerance of 5°. Opposite directions are not separately generated by default.

For each path, a virtual initial state removes the endpoint Li, and a virtual final state moves the same-index starting Li to that endpoint. The original endpoint descriptor routines are applied. Similar paths are grouped using the original five local-feature tolerances, and grouped features are averaged.

The original feature formulas, grouping thresholds, missing-value fallbacks, and fitted model parameters have not been rewritten.

## Interpretation, Validation, and Limitations

### Distinguish SQS Quality from Predicted Barriers

- **SQS Q** (Stage 1) evaluates how closely generated cluster correlations match a target random solid solution. It is **not** an energy.
- **Predicted barrier in eV** (Stage 3) is a trained-model estimate for an in-plane Li migration path in a relaxed structure. It is **not** an independently calculated NEB barrier.

For consequential scientific conclusions, use appropriate structural and energetic checks, and independently validate important migration candidates through NEB calculations or experiments.

### Predictor Limitations

- At least two Li sites and a suitable layered lithium-oxide environment are expected.
- Cross-layer diffusion, non-Li mechanisms, strong disorder, and out-of-distribution chemistries have not been validated.
- The original extractor may replace missing/nonfinite descriptors with zero, fall back to a nearby oxygen pair when an opposite-side pair is absent, or use identity symmetry operations. These preserved behaviors do **not** guarantee physical validity.
- A malformed structure, extraction failure, or a structure with no valid paths causes the batch to fail **without a partial prediction table**.
- Missing or empty inputs, unsupported output extensions, invalid distances, and existing output filenames are rejected.
- Successful execution returns status `0`; runtime validation errors return nonzero (typically `1`), and argparse can return `2` for invalid syntax.
- Output `source_file` values are absolute local paths. Remove the column before sharing if paths are sensitive. XLSX identifiers are stored as text; CSV text beginning with spreadsheet formula-trigger characters is prefixed with an apostrophe.
- Do not load untrusted model files. Confirm redistribution rights for the model, source code, and examples before publishing.

### Recorded Model Performance

The supplied predictor documentation records a dataset of **125 samples** (100 training / 25 test) and the following historical metrics:

| Metric | Recorded value |
| --- | ---: |
| Test R² | 0.8926 |
| Test RMSE | 0.0542 |
| Test MAE | 0.0444 |

These are **historical recorded metrics**, not a new accuracy measurement. The test set was also used for early stopping/model selection, so this is not a fully independent external assessment. The eV output label follows the source workflow; the original metadata does not independently specify a target-unit field.

### Documented Predictor Validation

The predictor was validated locally on **Windows / Python 3.12.10**, with CatBoost 1.2.8, pymatgen 2026.10.24, NumPy 2.5.4, pandas 2.3.3, and openpyxl 3.1.5.

Documented checks include:

- Five original example structures in nested folders, yielding 115, 127, 123, 115, and 120 grouped paths (**600 total**).
- All eight features and predictions compared against the original extractor, conversion script, and model: agreement within `1e-12`; maximum serialized prediction difference `1.11e-16`.
- Single-file, CSV, XLSX, lowest-barrier worksheet, ranking, and source-mapping checks.
- Root/nested discovery and excluded-directory handling.
- Rejection of invalid inputs and existing output files.
- Failed malformed/no-path batches without partial result tables.
- Syntax and spreadsheet formula-injection checks.
- SHA-256 verification that the packaged model and extractor match the originals.

These are functional and numerical-regression checks, **not** a new assessment of predictive accuracy. Dependency versions are bounded, not a complete transitive lock; future fresh installations may resolve different versions.

## Quick Start: Complete Workflow

**1. Generate SQS candidates on Linux / HPC:**

```bash
cd li_layered_sqs_server
./run.sh -i SQSINCAR --dry-run --quality high
./run.sh -i SQSINCAR --workers 16 --quality high
```

**2. Relax selected generated `POSCAR` structures with VASP.**

After verifying the relaxation results, collect the resulting `CONTCAR` files in a dedicated directory such as `relaxed_structures/`.

**3. Run the predictor from its own environment:**

```bash
cd contcar-predict-github
python predict.py "path/to/relaxed_structures" -o migration_barriers.xlsx
```

**4. Review** the `predictions` and `lowest_barriers` worksheets, and validate important candidates with higher-fidelity calculations.

---

**Scope of automation:** SQS generation and post-relaxation CatBoost prediction are provided. VASP relaxation, the selection of structures to relax, and subsequent high-fidelity migration-barrier validation are performed by the user.
