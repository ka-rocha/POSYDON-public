# POSYDON Tutorial Notebooks

All notebooks live under `docs/_source/tutorials-examples/`. This file describes the population synthesis notebooks that show how to implement and run POSYDON popsyn code, along with the science cases each one covers.

---

## Population Synthesis Notebooks

Path: `docs/_source/tutorials-examples/population-synthesis/`

---

### `10_binaries_pop_syn.ipynb` — Your First Binary Simulation (start here)

**Science case:** Minimal working example for running and inspecting a local population.

**What it covers:**
- Copying `population_params_default.ini` and creating a `PopulationRunner`
- Running 10 binaries with `PopulationRunner.evolve()`
- Reading output with the `Population` class: `pop.history`, `pop.oneline`, `pop.mass_per_metallicity`
- Memory-safe access patterns: `pop.history[idx]`, `pop.history.select(where='state == RLO2', columns=[...])`
- Computing and caching formation channels with `pop.calculate_formation_channels(mt_history=True)`
- Exporting sub-selections to new HDF5 files with `pop.export_selection(indices, outfile, append=True)`
- Optional: local MPI runs with `mpiexec` + manual merge via `PopulationRunner.merge_parallel_runs()`

---

### `evolve_single_binaries.ipynb` — Evolving Individual Binaries

**Science case:** Debugging, parameter exploration, and re-evolving specific systems from an existing population.

**What it covers:**
- Loading `SimulationProperties` from an ini file via `simprop_kwargs_from_ini`; manually setting `metallicity` on each step
- Constructing `SingleStar` and `BinaryStar` objects from scratch and calling `binary.evolve()`
- Inspecting the full evolution history with `BinaryStar.to_df(extra_columns={'step_names':'string'})`
- Restoring a binary to its initial state (`BINARY.restore()`) or to a specific timestep (`BINARY.restore(row_number)`)
- Controlling natal kicks via `star.natal_kick_array = [velocity, azimuthal, polar, phase]` to reproduce or alter outcomes
- Loading a binary from an existing population HDF5 file with `BinaryStar.from_df(pop.history[idx])` and re-evolving it
- Starting evolution from an arbitrary state (e.g. NS + H-rich star in RLO, or a detached binary needing internal structure columns like `center_h1`, `he_core_mass`, `separation`)

**Key patterns for starting from arbitrary states:**
```python
# RLO state (no internal structure needed)
binary = BinaryStar(star_1=SingleStar(state='NS', mass=1.1, spin=0.),
                    star_2=SingleStar(state='H-rich_Core_H_burning', mass=2.5),
                    **{'time':0., 'state':'RLO2', 'event':'oRLO2',
                       'orbital_period':1., 'eccentricity':0.},
                    properties=sim_prop)

# Detached state (needs internal structure for single-star grid matching)
star = SingleStar(mass=17.8, state='H-rich_Core_H_burning',
                  metallicity=Z, center_h1=X, center_he4=Y,
                  log_R=np.nan, he_core_mass=0.0)
```

---

### `pop_syn.ipynb` — Large-Scale Multi-Metallicity HPC Run

**Science case:** Setting up production population synthesis runs at 8 metallicities on a Slurm cluster.

**What it covers:**
- Multi-metallicity ini file configuration (`metallicity = [2., 1., 0.45, 0.2, 0.1, 0.01, 0.001, 0.0001]`)
- Using `posydon-popsyn setup population_params.ini --job_array=N --walltime=... --partition=... --account=...` to generate Slurm scripts
- Submitting all metallicities at once with `sh slurm_submit.sh`
- Verifying and rescuing failed runs: `posydon-popsyn check {folder}`, `posydon-popsyn rescue {folder}`
- Rule of thumb: ~1–2 seconds per binary; load time adds overhead; 5 GB RAM default fits `dump_rate=2000`
- Output: one `{MET}_Zsun_population.h5` per metallicity

---

### `custom_step_and_flow.ipynb` — Custom Steps and Flow Chart

**Science case:** Replacing physics prescriptions (e.g., CE, SN) with user-defined models, or redirecting the evolution flow.

**What it covers:**

**Via ini file:**
- Replacing the built-in flow with a custom one using `absolute_import = ['my_file.py', 'my_flow_chart']`
- Replacing a step (e.g., `step_CE`) with a custom class via `absolute_import`
- Placing custom modules in `posydon/user_modules/` to use the `import` key instead

**In-notebook (full kwargs example):**
- Building `SimulationProperties` from scratch with explicit step classes, parameters, and `extra_hooks = [(StepNamesHooks, {})]`
- Complete reference for all step kwargs: `MESA_STEP`, `DETACHED_STEP`, `CE_STEP`, `SN_STEP`, `DCO_STEP`
- CE step parameters: `prescription`, `common_envelope_efficiency`, `common_envelope_option_for_lambda`, `core_definition_H_fraction`, `common_envelope_option_for_HG_star`
- SN step parameters: `mechanism` (e.g., `'Patton&Sukhbold20-engine'`), `engine`, `PISN`, `ECSN`, `kick`, `kick_normalisation`, `sigma_kick_CCSN_NS/BH`
- Full column selection lists for `only_select_columns` (binary and star history) and `scalar_names`
- Swapping `step_HMS_HMS` to `detached_step` instead of the MESA grid step
- Writing a minimal custom step class (only needs `__init__` and `__call__(self, binary)`)

**Custom step template:**
```python
class my_CE_step:
    def __init__(self, verbose=False):
        self.verbose = verbose
    def __call__(self, binary):
        # donor/companion determined from binary.event: oCE1/oDoubleCE1 → star_1 is donor
        binary.orbital_period /= 2.
        donor.mass = donor.he_core_mass
        binary.state = 'detached'
        binary.event = None
```

---

### `bbh_analysis.ipynb` — Binary Black Hole Transient Population and Cosmic Rates

**Science case:** Selecting merging BBHs across metallicities, computing cosmic merger rates, and applying GW detector selection effects.

**What it covers:**
- Selecting BBH mergers from history: `S1_state == 'BH'`, `S2_state == 'BH'`, `event == 'CO_contact'`
- Looping over 8 metallicity files and appending selections to a combined `BBH_contact.h5`
- Calculating underlying mass normalization with `Population.calculate_underlying_mass(f_bin=0.7)`
- Writing a `selection_function(history_chunk, oneline_chunk, formation_channels_chunk)` that returns a DataFrame with required `time` (Myr) and `metallicity` columns
- Adding derived quantities: `chirp_mass`, `mass_ratio`, `chi_eff`, `t_inspiral`, spin-orbit tilts
- Using built-in `BBH_selection_function` from `posydon.popsyn.transient_select_funcs`
- Creating `TransientPopulation` with `Population.create_transient_population(func, 'BBH')`
- Efficiency per metallicity: `TransientPopulation.get_efficiency_over_metallicity()`, `.plot_efficiency_over_metallicity(channels=True)`
- Delay time distributions: `BBH_mergers.plot_delay_time_distribution(bins=...)`
- Applying SFH models with `TransientPopulation.calculate_cosmic_weights('IllustrisTNG', MODEL_in={...})` — supported models: `'IllustrisTNG'`, `'Neijssel2019'`, `'Madau+Fragos2017'`
- `Rates` class: `calculate_intrinsic_rate_density(channels=True)`, `plot_intrinsic_rate()`
- GW detectability: wrapping `DCO_detectability(sensitivity, ...)` from `transient_select_funcs`; `rates.calculate_observable_population(wrapper, 'design_H1L1V1')`
- Plotting intrinsic vs. observable mass distributions: `rates.plot_hist_properties('S1_mass', intrinsice=True, observable='design_H1L1V1', ...)`

---

### `lgrb_pop_syn.ipynb` — Long-Duration Gamma-Ray Burst Population

**Science case:** Extracting LGRB events from a BBH population based on collapsar disk formation.

**What it covers:**
- Uses `BBH_contact.h5` from the BBH analysis notebook as input
- Using `GRB_selection(history, oneline, formation_channels, S1_S2='S1')` from `posydon.popsyn.transient_select_funcs`; wrapping it to handle both stars
- `m_disk_radiated` in oneline as the LGRB trigger criterion (non-zero → potential LGRB)
- Creating a `TransientPopulation` for LGRBs alongside the existing BBH transient in the same file
- Metallicity bias function and plotting LGRB rate vs. redshift alongside the BBH rate
- Post-SN spin properties: `S1_spin_postSN`, `S2_spin_postSN`
- Further GRB properties available via `posydon.popsyn.GRB.get_GRB_properties` (energy, beaming)

---

### `one_met_pop_syn.ipynb` — Single-Metallicity DCO Rates

**Science case:** Rate calculation when only one metallicity is available (e.g., reprocessed v1 data), integrating SFH over a metallicity window.

**What it covers:**
- Using `select_one_met=True` and `dlogZ=[log10(Z_low/Zsun), log10(Z_high/Zsun)]` in `MODEL_in` to restrict to a metallicity band (e.g., `[0.5Zsun, 2Zsun]`)
- Comparing SFH models: IllustrisTNG vs. Madau+Fragos2017 vs. Neijssel2019
- Observable population with LVK design sensitivity at single metallicity
- Normalizing intrinsic and observable histograms with `normalise=True`

---

## Other Tutorial Notebooks

**`docs/_source/tutorials-examples/generating-datasets/`** — Grid creation and post-processing (PSyGrid HDF5 creation, termination flag pipeline, grid visualization)

**`docs/_source/tutorials-examples/MESA-grids/`** — Running MESA simulations directly (single HMS, grid runs, laptop-scale tests)
