# Li Layered SQS Generator

Portable, installation-free usage guide for generating special quasirandom structures (SQS) for layered Li-based high-nickel materials on Linux and HPC clusters.

## Requirements

The program is distributed as pure Python source code and **does not require package installation** (`pip install -e .` is unnecessary). An existing Python environment must provide:

- Python >= 3.10 (3.11 recommended)
- NumPy
- ASE
- pymatgen
- icet 3.2.x
- spglib
- PyYAML

The bundled `portable_runner.py` places the archive's `src` directory at the beginning of `sys.path`. Both `run.sh` and the scheduler scripts use the bundled source code and `resources/POSCAR_LiNiO2`.

## 1. Prepare the HPC Environment

Activate a preconfigured Conda, virtualenv, or module environment, for example:

```bash
conda activate li-sqs
```

Verify the required imports:

```bash
python -c "import numpy, ase, pymatgen.core, icet, spglib, yaml; print('Environment OK')"
```

## 2. Extract the Portable Package

```bash
unzip li_layered_sqs_portable.zip
cd li_layered_sqs_server
chmod +x run.sh scripts/*.sh
```

No `pip install` step is needed for the program itself.

## 3. Configure Multiple Formulas

Edit `SQSINCAR`. The recommended approach is to repeat `FORMULA` entries. All formulas share the same `N_STRUCTURES` setting:

```ini
FORMULA = LiNi0.83Mn0.13Mg0.01Ca0.01Ti0.01Zr0.01O2
FORMULA = LiNi0.8Mn0.1Co0.1O2
N_STRUCTURES = 5
```

Alternatively, separate formulas with commas on one line:

```ini
FORMULA = LiNi0.83Mn0.13Mg0.01Ca0.01Ti0.01Zr0.01O2, LiNi0.8Mn0.1Co0.1O2
```

Duplicate formulas after normalization are strictly rejected. Each formula gets its own normalized-formula output directory and a formula-specific `SQSINCAR.snapshot`.

Check the configuration and estimated workload before starting:

```bash
./run.sh -i SQSINCAR --dry-run --workers 2
```

The dry-run output includes `formula_count`, `jobs`, and each formula's `quality`, `n_steps`, `target_unique_pool`, `max_sqs_runs`, and `seed`.

## 4. Quality Presets and Computational Cost

The default preset is `--quality high`.

| Preset | Steps | Candidate pool | Intended use |
| --- | ---: | --- | --- |
| `standard` | 60,000 | Smaller | Quick exploration |
| `high` | 200,000 | Larger | Recommended default |
| `ultra` | 500,000 | Larger | Highest computational cost |

The candidate-pool target adapts according to:

```text
max(N_STRUCTURES * oversample_factor, min_unique_pool)
```

It is bounded by `max_sqs_runs`. Requesting five final structures does **not** automatically mean running 500 candidates. Actual cost also depends on duplicate rejection and composition.

The scientific definition remains the same across presets: a fixed supercell, pair-only clusters through the third-nearest-neighbor shell (3NN), and no triplet clusters.

Compare preset workloads before running:

```bash
./run.sh -i SQSINCAR --dry-run --quality standard
./run.sh -i SQSINCAR --dry-run --quality high
./run.sh -i SQSINCAR --dry-run --quality ultra
```

Use `--n-steps` only for an explicit override or testing. Its value is included in the checkpoint parameter hash.

## 5. Run SQS Generation

Run with 16 CPU worker processes:

```bash
./run.sh -i SQSINCAR --workers 16 --quality high
```

Set a base random seed:

```bash
./run.sh -i SQSINCAR --workers 16 --seed 12345
```

For multiple formulas, stable formula-index offsets are applied to avoid seed collisions. Seeds are recorded in dry-run output, summaries, and checkpoint parameters.

Formulas run in input order. By default, a failure stops execution immediately and reports the failing formula's index and name.

### Run in the Background

```bash
nohup ./run.sh -i SQSINCAR --workers 16 --quality high > li_sqs.log 2>&1 &
```

Follow the log:

```bash
tail -f li_sqs.log
```

### Resume an Interrupted Run

```bash
./run.sh -i SQSINCAR --workers 16 --quality high --resume
```

Existing formula directories resume from compatible checkpoints. Later formulas that have not started and have no output directory are created and run automatically.

### Start Over

If you intend to clear existing results for the matching formula directories and rerun:

```bash
./run.sh -i SQSINCAR --workers 16 --quality high --overwrite
```

> **Important:** `--resume` and `--overwrite` cannot be combined. Changes to quality, step count, candidate-pool settings, seed, or other scientific/engineering parameters make previous checkpoints incompatible. Use `--overwrite` only when you are prepared to replace old results.

### Use a Specific Python Interpreter

```bash
PYTHON_BIN=/path/to/python ./run.sh -i SQSINCAR --workers 16
```

## 6. Submit Jobs to an HPC Scheduler

The bundled SLURM, PBS, and LSF submission scripts use the same portable, installation-free execution method:

```bash
# SLURM
sbatch scripts/submit_slurm.sh

# PBS
qsub scripts/submit_pbs.sh

# LSF
bsub < scripts/submit_lsf.sh
```

The scripts can run unchanged with the default `high` preset, or you can explicitly add `--quality high` to their command arguments.

**Parallelization:** The application uses multiple processes on a **single node**, not MPI. Keep the requested node count at **1**.

## 7. Output Files

Each formula produces its own normalized-formula results directory:

```text
<formula-output-directory>/
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

| Path | Description |
| --- | --- |
| `1/POSCAR`, `2/POSCAR`, ... | Final generated structures |
| `1/metadata.json`, etc. | Per-structure composition and parameters |
| `ranking.csv` | Structures ranked by ascending SQS Q value |
| `run_summary.json` | Run status and statistics for the formula |
| `generation.log` | Formula-specific execution log |
| `SQSINCAR.snapshot` | Input snapshot containing only that formula |
| `.work/` | Candidate-generation and resume/checkpoint files |

After a normal CLI run, the program also reports the overall `status`, `formula_count`, and a summary for each formula.

### Interpreting SQS Q

A lower **SQS Q** indicates that the cluster correlations are closer to those of the target random solid solution. **Q is not a VASP energy** and does not establish thermodynamic stability. Final stability must be evaluated through VASP structural relaxation and energy calculations.

## Quick Reference

```bash
# Check inputs and workload
./run.sh -i SQSINCAR --dry-run --workers 2

# Run with the recommended quality preset
./run.sh -i SQSINCAR --workers 16 --quality high

# Resume a compatible interrupted run
./run.sh -i SQSINCAR --workers 16 --quality high --resume

# Replace old results and start again
./run.sh -i SQSINCAR --workers 16 --quality high --overwrite
```
