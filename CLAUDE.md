# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Environment Requirements

POSYDON requires **Python 3.11** (strictly pinned). Three environment variables must be set before importing `posydon`:

```bash
export PATH_TO_POSYDON=/path/to/POSYDON-public/
export PATH_TO_POSYDON_DATA=/path/to/data/parent/   # must contain POSYDON_data/ subdir
export MESA_DIR=/path/to/mesa/                       # only needed for grid creation
```

These can also be placed in a `.env` file at the repo root (loaded automatically via `python-dotenv`).

## Install

```bash
pip install .                    # base install
pip install ".[hpc]"             # adds mpi4py for HPC parallel population runs
pip install ".[ml]"              # adds tensorflow for profile interpolation
pip install ".[doc]"             # adds sphinx etc. for building docs
pip install ".[vis]"             # adds PyQt5 for visualization
```

## Running Tests

```bash
# Unit tests (CI-enforced, requires 100% coverage on targeted modules)
export PATH_TO_POSYDON=./
export PATH_TO_POSYDON_DATA=./posydon/unit_tests/_data/
export MESA_DIR=./
python -m pytest posydon/unit_tests/

# Run a single unit test file
pytest posydon/unit_tests/utils/test_posydonerror.py

# With coverage (mirroring CI)
python -m pytest posydon/unit_tests/ \
    --cov=posydon.utils --cov=posydon.config \
    --cov=posydon.grids --cov=posydon.popsyn.star_formation_history \
    --cov-branch --cov-report term-missing --cov-fail-under=100

# Integration tests (require real data)
pytest posydon/tests/
```

## CLI Entry Points (installed to `bin/`)

```bash
get-posydon-data                        # download POSYDON_data from Zenodo
posydon-popsyn <ini_file>               # set up multi-metallicity HPC population run
posydon-run-grid / posydon-setup-grid   # MESA grid management
posydon-run-pipeline / posydon-setup-pipeline
compress-mesa                           # compress raw MESA output
```

## Architecture Overview

### Data Flow

```
MESA runs (folders) → PSyGrid (HDF5) → IFInterpolator (.pkl) → BinaryPopulation / Population
```

### `posydon/grids/` — Grid ingestion

`psygrid.py::PSyGrid` reads raw MESA simulation folders and packages them into a single HDF5 file. It handles: termination flags (4-flag system in `termination_flags.py`), history downsampling, profile downsampling, scrubbing, and initial/final value extraction. The HDF5 grid is the source of truth for all ML training.

### `posydon/interpolation/` — Machine learning layer

`IF_interpolation.py::IFInterpolator` wraps scikit-learn classifiers and regressors to map initial binary parameters → final state/values without running MESA. Supports `linear`, `1NN`, and `linear3c_kNN` methods. Trained interpolators are serialized as `.pkl` files and loaded by evolution steps at runtime. `interpolation.py` provides track interpolation (`psyTrackInterp`) for time-resolved evolution within a step.

### `posydon/binary_evol/` — Binary evolution engine

The core loop state machine:

- **`BinaryStar` / `SingleStar`** (`binarystar.py`, `singlestar.py`): stateful objects. Every parameter (mass, state, orbital period, etc.) has a `*_history` list. `BINARYPROPERTIES` and `STARPROPERTIES` lists control what gets saved to history.

- **`flow_chart.py`**: the flow chart is a `dict` mapping `(star1_state, star2_state, binary_state, binary_event)` 4-tuples to step name strings. `flow_chart()` builds the default dict; pass `CHANGE_FLOW_CHART` to override specific entries. See `posydon/user_modules/my_flow_chart_example.py` for a custom flow example.

- **`SimulationProperties`** (`simulationproperties.py`): holds the flow chart function and all instantiated step objects (`step_HMS_HMS`, `step_CE`, `step_SN`, etc.), plus global physics parameters (mass transfer efficiency, CE efficiency). Hooks (`extra_pre_step`, `extra_post_step`, etc.) can be injected for custom logic.

- **Evolution steps** — called by the flow chart at each stage:
  - `MESA/step_mesa.py`: grid-interpolated step for HMS-HMS, CO-HeMS, CO-HMS-RLO, CO-HeMS-RLO grids
  - `CE/step_CEE.py`: common envelope ejection (analytic)
  - `SN/step_SN.py`: supernova + natal kick
  - `DT/step_detached.py`: detached evolution (winds, tides, gravitational waves)
  - `DT/step_disrupted.py`, `step_isolated.py`, `step_merged.py`, `step_initially_single.py`: terminal/special steps
  - `step_end.py`: terminates the evolution loop

### `posydon/popsyn/` — Population synthesis

- **`BinaryPopulation`** (`binarypopulation.py`): samples initial conditions (from `.ini` file or kwargs), iterates binaries through `SimulationProperties`, writes results to HDF5 with `history` and `oneline` dataframes. Memory-efficient: evolves and saves one binary at a time. MPI-parallel via `mpi4py`.

- **Population configuration**: `.ini` files (ConfigParser syntax) declare the flow, each step's class and parameters, IMF/period/mass-ratio distributions, star formation history, and metallicity. Default template: `posydon/popsyn/population_params_default.ini`.

- **`synthetic_population.py`**: post-processing classes — `Population`, `History`, `Oneline`, `TransientPopulation`, `Rates`. `Rates` computes cosmic merger rates using star formation history convolution.

- **`star_formation_history.py`**: SFR as a function of redshift and metallicity.

### `posydon/utils/`

- `posydonerror.py`: hierarchy — `POSYDONError` → `FlowError`, `GridError`, `ClassificationError`, `ModelError`, `NumericalError`, `MatchingError`
- `posydonwarning.py`: `Pwarn` and `Catch_POSYDON_Warnings` for custom warning categories
- `common_functions.py`: shared orbital mechanics, state inference, star-flipping utilities
- `constants.py`: physical constants and unit conversions
- `data_download.py`: Zenodo dataset downloads

## Version and Science Change Notes

**[`docs/SCIENCE_CHANGES.md`](docs/SCIENCE_CHANGES.md)** documents code changes between versions that can shift science results, with enough detail to understand *why* outputs may differ. Covers: natal kick prescription refactor, `step_merged` attribute overhaul, DCO inspiral timing fixes, detached step interpolator rewrite, Moe+DiStefano period distribution fix, and population weighting changes.

## Tutorial Notebooks

Working code examples are in `docs/_source/tutorials-examples/`. See **[`docs/NOTEBOOKS.md`](docs/NOTEBOOKS.md)** for a detailed description of each notebook, its science case, and the key code patterns it demonstrates.

Population synthesis notebooks (in order of complexity):

| Notebook | Science case |
|---|---|
| `population-synthesis/10_binaries_pop_syn.ipynb` | Minimal local run; `Population` class API |
| `population-synthesis/evolve_single_binaries.ipynb` | Debug/re-evolve individual binaries; arbitrary initial states |
| `population-synthesis/pop_syn.ipynb` | Multi-metallicity HPC Slurm setup |
| `population-synthesis/custom_step_and_flow.ipynb` | Custom steps and flow charts; full step kwargs reference |
| `population-synthesis/bbh_analysis.ipynb` | BBH selection, `TransientPopulation`, `Rates`, GW detectability |
| `population-synthesis/lgrb_pop_syn.ipynb` | LGRB population from collapsar disk formation |
| `population-synthesis/one_met_pop_syn.ipynb` | Single-metallicity rates with SFH metallicity window |

## Unit Test Conventions

- Mirror source structure: `posydon/utils/foo.py` → `posydon/unit_tests/utils/test_foo.py`
- Import the module under test as `totest` (e.g. `import posydon.utils.foo as totest`)
- Access dependencies through the tested module (`totest.np`) rather than re-importing independently
- CI enforces **100% branch coverage** on `posydon.utils`, `posydon.config`, `posydon.grids`, and `posydon.popsyn.star_formation_history`
- Mark legitimately unreachable code with `# pragma: no cover`
- Template for new tests: `posydon/unit_tests/test_template.py`
