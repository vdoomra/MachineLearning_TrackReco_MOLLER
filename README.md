# Optics Reconstruction for the MOLLER Experiment

This repository contains the simulation and analysis framework I developed for optics reconstruction in the MOLLER experiment at Jefferson Lab. Starting from simulated hits in the GEM tracking detectors, it reconstructs the scattering angles of each electron at the target using regression models calibrated with a sieve and carbon foil targets.

## Background

MOLLER will measure the parity-violating asymmetry in electron–electron (Møller) scattering, which gives a precise determination of the weak mixing angle. To interpret the measurement, we need to know the kinematics of each scattered electron at the target, in particular the polar angle θ and the azimuthal angle φ.

Those angles are not measured directly. Electrons travel through the spectrometer's magnetic fields before reaching the GEM trackers, which record their position and direction. **Optics reconstruction** is the mapping from the GEM measurements back to the target kinematics.

The mapping is calibrated with a standard technique:

- A **sieve**, a plate with a pattern of 21 holes, is placed in the beam path, so only electrons passing through known holes reach the detectors.
- Thin **carbon (C12) foils** at several positions along the target chamber provide scattering at known locations.
- Data are taken at **four beam energies** (2.2, 4.4, 6.6 and 8.8 GeV, labelled passes p1–p4) to cover a wide range of kinematics.

## Method

**1. Simulation.** Elastic electron–carbon scattering is simulated with remoll, the Geant4-based MOLLER simulation, including the full spectrometer geometry and magnetic field maps, with the sieve in place.

**2. Event selection.** For each sieve hole, the image at the GEM plane is characterised by its covariance matrix in r–φ, r′–φ and φ′–φ space. An elliptical cut around each image keeps well-reconstructed electrons and removes those that radiated or scattered on the way. The selected events are written to one CSV file per sieve hole, foil position and beam pass.

**3. Regression.** Polynomial regression (degree 2) maps the GEM quantities to the target angles:

| Target quantity | Input features |
|---|---|
| θ (polar angle) | GEM r, r′ |
| φ (azimuthal angle) | GEM r, r′, φ, φ′ |

Here r and φ are the hit position at the GEM plane, and r′ and φ′ are the track slopes. The data are split into training and test sets. To avoid biasing the fit toward any one target position, events are resampled so that each foil contributes similar statistics.

**4. Uncertainties.** Uncertainties on the fit parameters are estimated with 500 bootstrap resamples of the training data.

**5. Target-position correction.** The mean θ residual depends on where along the target the electron scattered. This dependence is fitted with a quadratic in the foil position and applied as an iterative correction.

**Performance** is evaluated from the residual distributions (true − reconstructed angle) on the test set, fitted with a Gaussian to extract the mean and width.

<!-- Add a results figure here, e.g. the θ residual distribution or residual mean vs foil position:
<p align="center">
  <img src="figures/theta_residuals.png" width="650" alt="Reconstructed θ residuals">
</p>
-->

## Workflow

**Step 1: Run the simulation.** Submit remoll jobs to a SLURM cluster:

```bash
python sBatchSubmit.py         # C12 optics runs: sieve in, four beam passes
python sBatchSubmit_moller.py  # Møller/elastic ep runs at 11 GeV
```

**Step 2: Slim the output.** Keep only the GEM and truth information from the relevant virtual detector planes, using the `SlimGeneral.C` macro. `sBatchSubmit_Slim.py` runs this as batch jobs, which is optional but faster.

```bash
python sBatchSubmit_Slim.py
```

**Step 3: Check the GEM distributions.** Use the macros in `GEM Distributions/` to plot hit distributions at the GEM planes and check the simulation output. A sample is shown below.

<p align="center">
  <img src="GEM%20Distributions/sample_GEM_distributions.png" width="650" alt="Sample GEM hit distributions">
</p>

**Step 4: Select events and write the CSV files.** Apply the elliptical sieve-hole selection and produce one CSV file per hole, foil position and pass:

```bash
root -l -b -q 'generating_csv_files.C("<slim_file>.root")'
```

**Step 5: Fit the optics.** Run the regression, bootstrap uncertainties and target-position correction:

```bash
python optics_fitting_codes/optics_fitting_Moller.py
```

The script expects the CSV files in `csv_output/<target>_<pass>/`.

## Data

Pre-produced slim outputs are too large for GitHub and are available on [Google Drive](https://drive.google.com/drive/folders/1Aaf2tjZkWhYhN6xnJbsI_ZxcN90_1zpB?usp=sharing).

File names use the following tags:

| Tag | Meaning |
|---|---|
| Generator: `C12`, `MOLLER` | Elastic electron–carbon or Møller scattering |
| Target: `US` | C12 foil at the upstream end of the target chamber |
| Target: `UM` | Between upstream and middle |
| Target: `MS` | Middle of the target chamber |
| Target: `MD` | Between middle and downstream |
| Target: `DS` | Downstream end of the target chamber |
| Pass: `p1`–`p4` | Beam energy 2.2, 4.4, 6.6, 8.8 GeV |

## Repository structure

```
.
├── sBatchSubmit.py                   # Step 1: submit C12 optics simulations (SLURM)
├── sBatchSubmit_moller.py            # Step 1: submit Møller/ep simulations
├── SlimGeneral.C                     # Step 2: slim remoll output to GEM + truth information
├── sBatchSubmit_Slim.py              # Step 2: run slimming as batch jobs
├── GEM Distributions/                # Step 3: plotting macros and sample figure
├── generating_csv_files.C            # Step 4: sieve-hole selection, CSV output
├── optics_fitting_codes/
│   └── optics_fitting_Moller.py      # Step 5: regression, bootstrap, target-position correction
├── slim_output/                      # Notes on pre-produced slim files
├── C12 Form Factors in Beam generator simulations.pdf
└── Generators in Remoll.pdf
```

## Requirements

- [remoll](https://github.com/JeffersonLab/remoll) (Geant4) for the simulation
- ROOT for the slimming and selection macros
- Python with numpy, pandas, scipy, scikit-learn, matplotlib and colorama
- A SLURM cluster for the batch scripts (paths in the scripts are set for the JLab farm and need adjusting)

## Supporting studies

- **C12 Form Factors in Beam generator simulations.pdf** — An early validation study showing that the beam generator in remoll incorporates the correct C12 form factors, by comparing it with the dedicated C12 generator.
- **Generators in Remoll.pdf** — How events are sampled from the different targets in remoll. At the time of this study there were two carbon foils in a single chamber; this is no longer the case.
