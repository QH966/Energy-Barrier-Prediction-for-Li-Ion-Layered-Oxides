# SQS Generation and Li Migration Barrier Prediction

This project provides a two-step workflow: generate SQS structures, relax them with VASP, and predict Li migration barriers from the relaxed structures.

## 1. Generate SQS Structures

**Requirements:** Linux, Python >= 3.10 (3.11 recommended), `numpy`, `ase`, `pymatgen`, `icet` 3.2.x, `spglib`, and `PyYAML`.

Activate an environment containing these dependencies, then extract the portable package:

```bash
unzip li_layered_sqs_portable.zip
cd li_layered_sqs_server
chmod +x run.sh scripts/*.sh
```

No installation of the SQS program is required.

Edit `SQSINCAR` to specify compositions and the number of structures:

```ini
FORMULA = LiNi0.83Mn0.13Mg0.01Ca0.01Ti0.01Zr0.01O2
FORMULA = LiNi0.8Mn0.1Co0.1O2
N_STRUCTURES = 5
```

Check the configuration and run:

```bash
./run.sh -i SQSINCAR --dry-run
./run.sh -i SQSINCAR --workers 16 --quality high
```

Quality options: `standard`, `high` (default), and `ultra`.

To resume an interrupted run:

```bash
./run.sh -i SQSINCAR --workers 16 --quality high --resume
```

For HPC submission, use one of the included scripts:

```bash
sbatch scripts/submit_slurm.sh
qsub scripts/submit_pbs.sh
bsub < scripts/submit_lsf.sh
```

The program uses single-node multiprocessing, not MPI.

Each composition has its own output directory containing `1/POSCAR`, `2/POSCAR`, etc., along with `ranking.csv` and `run_summary.json`.

## 2. Relax the Structures with VASP

Select the generated `POSCAR` structures and perform VASP structural relaxation yourself. Save the relaxed results as files named **`CONTCAR`**.

For example:

```text
relaxed_structures/
├── sample_1/CONTCAR
├── sample_2/CONTCAR
└── sample_3/CONTCAR
```

## 3. Predict Li Migration Barriers

Use the separate predictor directory containing `predict.py`, `requirements.txt`, and the supplied `model/` files.

**Requirements:** Python 3.12 recommended.

Install dependencies (Linux/macOS):

```bash
cd contcar-predict-github
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
```

Run predictions for all `CONTCAR` files under a directory:

```bash
python predict.py "path/to/relaxed_structures" -o results.csv
```

Or export Excel results:

```bash
python predict.py "path/to/relaxed_structures" -o results.xlsx
```

For a single structure:

```bash
python predict.py "path/to/CONTCAR" -o single.csv
```

The program automatically finds `CONTCAR` files in subdirectories, extracts eight structural descriptors, and predicts in-plane Li migration barriers using the supplied CatBoost model. Results are ranked by predicted barrier.

**Note:** The predicted barriers are screening estimates, not NEB results. Important candidates should be validated with further calculations.
