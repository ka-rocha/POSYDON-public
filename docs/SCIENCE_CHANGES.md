# POSYDON Science-Relevant Code Changes

This file documents changes between code versions that can affect science results, with enough detail to understand *why* outputs may differ. It is intended as a reference when comparing population runs across versions.

---

## Transitioning from the `development` branch (pre-v2.0.6) to `main` (v2.2.8)

The `development` branch on the `ka-rocha` fork diverged from upstream around the v2 data release era (commit `ae62e22d0`, June 2025). The current `main` branch is at **v2.2.8** (198 commits ahead). The changes below are those most likely to shift science results.

---

### 1. Natal Kick Prescriptions — `posydon/binary_evol/SN/step_SN.py`

**PRs #642, #717 | v2.1 → v2.2.8**

This is the most impactful change for DCO science. The kick system was refactored to separate two previously conflated concepts:

#### `kick_prescription` (new parameter) — *how the kick velocity is drawn*

| Value | Distribution | Parameters | Reference |
|---|---|---|---|
| `'maxwellian'` | Maxwellian | `sigma_kick_*` | Hobbs et al. 2005, MNRAS 360 |
| `'log_normal'` | Log-normal | `sigma_kick_*` (dispersion), `mean_kick_*` | Disberg & Mandel 2025, arXiv:2505.22102 |
| `'asym_ej'` | Asymmetric ejecta: `V ∝ (M_ej / M_NS)` | none (mass-derived) | Janka 2017, ApJ 837, arXiv:1611.07562 |
| `'linear'` | `V = 115 × (M_ej/M_rem) + 15` km/s | none (mass-derived) | Richards et al. 2022, arXiv:2208.02407 |

**Default is `'maxwellian'`** — identical to the pre-refactor behavior.

#### `kick_normalisation` (existing parameter) — *how the drawn kick is scaled for BHs*

| Value | NS scaling | BH scaling |
|---|---|---|
| `'one_over_mass'` | 1.0 | `1.4 / M_BH` (momentum-conserving) |
| `'one_minus_fallback'` | `1 − f_fb` | `1 − f_fb` |
| `'NS_one_minus_fallback_BH_one'` | `1 − f_fb` | 1.0 |
| `'one'` | 1.0 | 1.0 |
| `'zero'` | 0.0 | 0.0 |

#### New `mean_kick_*` parameters

For `kick_prescription = 'log_normal'`, the mean of the distribution can now be set independently per remnant type. Defaults to `exp(5.60) ≈ 270 km/s` if `None`.

```ini
kick_prescription = log_normal
sigma_kick_CCSN_NS = 0.68      # log-normal sigma (not a Maxwellian sigma)
mean_kick_CCSN_NS = None       # defaults to exp(5.60)
sigma_kick_CCSN_BH = 0.68
mean_kick_CCSN_BH = None
sigma_kick_ECSN = 20.0
mean_kick_ECSN = None
```

#### Backwards compatibility

If your old ini files used `kick_normalisation = 'asym_ej'` or `kick_normalisation = 'linear'` (the old interface), the new code will emit a `DeprecationWarning` and remap automatically, but you should update to `kick_prescription` explicitly. **Results are unchanged if you were using `'maxwellian'` with `'one_over_mass'` (the defaults).**

#### Science implication

- `'log_normal'` produces a heavier tail toward high kicks than a Maxwellian with the same sigma, preferentially disrupting wider binaries and shifting the surviving DCO orbital parameter distribution.
- `'asym_ej'` and `'linear'` tie kick velocity to ejecta mass, introducing a correlation between the pre-SN stellar structure and post-SN orbital parameters. This breaks the independence assumption of the Maxwellian and can change formation channel fractions, especially for low-ejecta-mass BH progenitors.

---

### 2. `step_merged` — Mass-Weighted Attributes of Merger Products

**PR #785 | v2.2.8**

Complete rewrite of `posydon/binary_evol/DT/step_merged.py`. The stellar attributes of the merged product (central abundances `center_h1`, `center_he4`, core masses `he_core_mass`, `co_core_mass`, etc.) are now computed as mass-weighted averages across the two merging components, with merger-type-specific logic:

- **HMS+HMS**: mass-weighted averages, with envelope mass adjustment applied after
- **postMS+MS**: core properties come entirely from the postMS star; envelope from companion
- **postMS+HeMS**: He-star core retained; mass-weighted envelope
- **He+He**: combined He core with mass-weighted central abundances

**Before this fix**, merged stars likely had incorrect or unset internal structure properties, which would propagate into the subsequent detached evolution and affect the final remnant mass and SN outcome.

**Science implication:** Any population containing merger products (channel includes `step_merged`) will see different remnant masses and SN types compared to runs on the old `development` branch.

---

### 3. DCO Inspiral Timing — `step_dco` / `DoubleCO`

**PRs #810, #795 | v2.2.x**

Two fixes to the gravitational wave inspiral calculation in `posydon/binary_evol/DT/double_CO.py`:

1. **Integration limits**: integration now runs from `t0 → max_time` rather than `0 → (max_time − t0)`. The old approach shifted the time axis, causing incorrect merger times when t0 was non-zero.
2. **`CO_contact` detection**: added a `ZeroDivisionError` guard and corrected time offset appending for the binary time history.

`step_dco` is now a child class of `step_detached` (PR #725) rather than a separate implementation, inheriting its ODE solver infrastructure.

**Science implication:** Merger times (delay times) for DCO systems can shift, affecting the delay time distribution and hence the cosmic merger rate when convolved with a star formation history. Populations with many long-inspiral systems (wide DCOs) are most affected.

---

### 4. `step_detached` Interpolator Overhaul

**PR #756 | v2.2.0**

Major rework of the stellar track interpolation in `posydon/binary_evol/DT/step_detached.py`. The matching to single-star grids and ODE integration for wind/tidal/gravitational-radiation evolution was refactored. Key changes:

- More robust handling of failed stellar matches (`MatchingError` now caught more cleanly)
- `RLO_orbit_at_orbit_with_same_am` logic improved
- History array handling fixed (PR #769) — a bug caused incorrect access to stellar property arrays during detached evolution

**Science implication:** Mass loss rates, spin evolution, and orbital widening/tightening during the detached phase may differ slightly. Most relevant for wide-orbit systems that spend significant time in the detached step.

---

### 5. Moe+DiStefano Period Distribution Fix

**PR #771 | v2.2.x**

Fixed a bug in `posydon/popsyn/independent_sample.py` where the Moe+DiStefano (2017) initial period sampler was treating period as log-period. Also added explicit `orbital_period_min` / `orbital_period_max` limit parameters.

**Science implication:** If you were using `orbital_period_scheme = 'Moe+DiStefano17'`, the initial period distribution was wrong. The fix shifts the sampled distribution. Populations using `'Sana+12_period_extended'` or `'log_uniform'` are unaffected.

---

### 6. Population Weighting Rework

**PR #493 | v2.1**

`posydon/popsyn/normalized_pop_mass.py` and the `underlying_mass` calculation were overhauled. The normalization that converts simulated binary counts to physical rates (per unit star formation mass) was updated.

**Science implication:** Absolute merger rate densities and efficiencies (events per M☉) computed with `Population.calculate_underlying_mass()` may differ from old runs. Relative channel fractions within a single population run are unaffected.

---

### 7. `natal_kick_array` Renaming

**PR #731 | v2.2.x**

The `natal_kick_array` attribute on `SingleStar` was renamed. Any code or scripts that access `star.natal_kick_array` directly by the old name will need updating. The `oneline` column names for kick arrays also changed accordingly.

**Science implication:** No effect on physics, but loading old population HDF5 files and accessing kick columns by name may fail. Use `pop.oneline.columns` to inspect the actual column names in a file.

---

### 8. `SimulationProperties` and ini File Handling

**PRs #776, #777 | v2.2.x**

- `SimulationProperties` now accepts arbitrary `**kwargs` without raising on unknown keys, making ini files more forward-compatible.
- `BinaryPopulation` metallicity parsing from ini files was fixed — previously some metallicity formats were silently misread.

**Science implication:** Populations where metallicity was misread would have loaded incorrect interpolation grids. Check that your metallicity values in old ini files were being applied correctly.

---

## Version Reference

| Branch / Tag | Approx. date | Key milestone |
|---|---|---|
| `development` (fork divergence) | ~Jun 2025 | v2 data release era, v2.0.6 patches |
| v2.1.x | Aug–Sep 2025 | Detached step overhaul, additional kick prescriptions, Moe+DiStefano fix |
| v2.2.0 | Oct 2025 | `step_detached` interpolator rewrite, `step_dco` as child class |
| v2.2.7 | early 2026 | DCO timing fixes, RNG reproducibility |
| v2.2.8 | Jun 2026 | `step_merged` attribute overhaul (current `main`) |
