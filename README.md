# VAE Studio (VAE) vs. Monolix (SAEM) and NONMEM (FOCEI): benchmark runs

This repository contains every run of the paper

> Rohleff J, Krause A, Koch G, Schropp J. *Benchmarking the Variational Autoencoder against SAEM in Monolix and FOCEI in
> NONMEM: Combining Accuracy and Speed in Nonlinear Mixed Effects Modeling.* [LINK]

Each run is a complete project of its tool: data, model, settings and results. It can be opened and repeated
without any other file. VAE Studio is available at [Link].

## Structure

There is one folder per data set. Inside, every tool has its own subfolder, named software_algorithm:
`Monolix_SAEM`, `NONMEM_FOCEI` and `VAEStudio_VAE`. Data sets with known true parameters also contain
`true_parameters.json` (population parameters) and `true_individual_parameters.csv`.

| Folder | Content | Paper |
|---|---|---|
| `01_Schoemaker_benchmark/rich/runN001` … `runN036` | 36 rich data sets of Schoemaker et al. (2019) | Figures 2–3, Supplement S4 |
| `01_Schoemaker_benchmark/sparse/bs_001` … `bs_500` | 500 sparse bootstrap data sets of Schoemaker et al. (2019) | Figures 2–3, Supplement S4 |
| `02_problem_classes/<class>/<model>/rep01_single_dose` … `rep20_multiple_dose` | 16 models, 20 data sets each, 100 individuals; rep01–rep10 single dose, rep11–rep20 multiple doses | Figures 2–4 |
| `03_small_data_sets/<class>/<model>/N020_rep01_single_dose` … | first 20 or 50 individuals of rep01–rep05 and rep11–rep15 | Figures 2–3, Table S1 |
| `04_population_size/{1cmt_oral,indirect_response}/N00050_rep01` … `N10000_rep03` | 50 to 10 000 individuals, three nested data sets per size | Figures 2, 3 and 5 |
| `05_neural_network/weights_010` … `weights_160` | indirect response model with a neural network in the production, 10 to 160 weights | Figure 6 |
| `06_covariate_selection/candidates_000` … `candidates_512` | covariate selection with 0 to 512 candidate covariates | Figure 7 |
| `07_stiff_models_tolerance/rtol_1e-3` … `rtol_1e-7` | stiff models at different solver tolerances (SAEM, VAE) | Supplement S6 |
| `08_FOCEI_rtol_1e-3/` | problem classes with FOCEI at rtol 1e-3, and the repeats with NSIG=2 (`NSIG2_repeats/`) | Supplement S6 |

Problem classes (`<class>`): `closed_form` (1cmt_oral, 2cmt_oral, 2cmt_infusion, 3cmt_iv_bolus), `nonstiff_ODE`
(1cmt_Michaelis-Menten, 2cmt_Michaelis-Menten, transit_absorption, effect_compartment, parent_metabolite,
indirect_response, TMDD_QSS, myelosuppression, tumour_growth_inhibition, daridorexant) and `stiff_ODE` (TMDD_full,
indirect_response_binding). Model equations, true parameters and starting values are given in Supplement S2.

Neural network study: `weights_XXX/start_weights.npz` holds the common starting weights of all three tools,
`simulation/` the simulated data with the true production function, and `true_parameters/VAEStudio_VAE` the likelihood
at the true parameters.

Covariate selection: `candidates_000` holds the model without covariates (Monolix_SAEM, VAEStudio_VAE). From two
candidates on, the folders contain the VAE run with integrated selection (`VAEStudio_VAE`) and the Monolix model
building with covSAMBA, COSSAC and SCM (`Monolix_covSAMBA`, `Monolix_COSSAC`, `Monolix_SCM`; SCM up to 128
candidates). The candidate models evaluated during the Monolix model building are not included (about 50 GB); the
selected model and the history of every search are in `out/ModelBuilding/modelBuilding.txt` and `result.json`.

Every run folder contains `result.json` with the numbers used in the paper (read from the tool's own output files).

## Software

All runs were computed on one machine (Apple M4 Pro, 8 performance and 4 efficiency cores, 24 GB, macOS 27) with
eight threads or processes.

### Monolix 2024R1 (`Monolix_SAEM`, `Monolix_covSAMBA`, `Monolix_COSSAC`, `Monolix_SCM`)

A run folder contains `project.mlxtran`, the data file, `model.txt` where the model is not taken from the Monolix
library, and the results in `out/`.

* Open: Monolix → *Open project* → `project.mlxtran`. The results in `out/` are loaded with it.
* Re-run: *Run* in the GUI, or from inside the folder `monolix --no-gui -p project.mlxtran --thread 8`.
* Covariate selection: the project is the base model without covariates, the model-building settings are in
  `mb.ini` (strategy, criterion BICc, likelihood of every candidate by importance sampling, tested parameters and
  covariates). Command line: `monolix --no-gui -p project.mlxtran -t modelBuilding --config mb.ini --thread 8`.

### NONMEM 7.6.0 (`NONMEM_FOCEI`)

A run folder contains the control stream `run.ctl`, the data file `data.csv`, for ODE models the tolerance routine
`tolswitch.f90`, the NONMEM output (`run.lst`, `run.ext`, `run.phi`, …) and `params.json` (which THETA, OMEGA and
SIGMA belongs to which parameter).

* Estimation: `$EST METHOD=COND INTERACTION FAST NSIG=2 SIGL=6 MAXEVAL=9999`. For the neural network models the
  limit of function evaluations is higher (see `run.ctl`).
* Solver: ADVAN1–4/11 for closed-form models, ADVAN13 for non-stiff and ADVAN14 for stiff ODE models, estimation
  with rtol/atol 1e-6 (daridorexant 1e-9/1e-12). `tolswitch.f90` switches to rtol 1e-3/atol 1e-6 for the
  importance sampling, the tolerance of Monolix and VAE Studio.
* −2LL: two `$EST METHOD=IMP EONLY=1` steps after FOCEI, 5 iterations with 1 000 samples and 1 iteration with
  10 000 samples (`DF=5`). The reported value is `OBJECTIVE FUNCTION VALUE WITH CONSTANT` of the last step.
* Re-run: from inside the folder `nmfe76 run.ctl run.lst -maxlim=2 -parafile=fpilinux8.pnm "[nodes]=8"`
  (`fpilinux8.pnm` from NONMEM's `run/` folder; without it the run is serial, then remove the `PARAFILE=` options
  from `$EST`). Run in a path without spaces.

### VAE Studio (`VAEStudio_VAE`)

A run folder is a VAE Studio project: the project file `<run>.vae`, the data file, `model.txt` (structural model
in plain text), `summary.txt` and `results/` (readable results) and `engine/` (everything the engine needs,
including the project specification `engine/project.json`).

* Open: VAE Studio → *Open project* → `<run>.vae`. Up to VAE Studio 0.9.5: *Open project* → select the run folder.
* Re-run: Structural Model → *Apply model*, then Run → *Run*. Settings, starting values and the seed are stored
  in the project. Closed-form models reproduce bit for bit, ODE models up to the order of the threaded summation
  (−2LL typically within a few hundredths).

On macOS 27 with Xcode 16, Monolix and NONMEM need `SDKROOT` set to the Xcode SDK to compile models
(`export SDKROOT=/Applications/Xcode.app/Contents/Developer/Platforms/MacOSX.platform/Developer/SDKs/MacOSX.sdk`).

## Data

* Schoemaker benchmark: supplementary data of Schoemaker R, Fidler M, Laveille C, Wilkins JJ, Hooijmaijers R,
  Post TM, Trame MN, Xiong Y, Wang W. Performance of the SAEM and FOCEI algorithms in the open-source, nonlinear mixed
  effect modeling tool nlmixr. CPT Pharmacometrics Syst Pharmacol. 2019;8(12):923–930. doi: 10.1002/psp4.12471.
* Problem classes, small data sets, population size and neural network: simulated for this study (Supplement S3).
* Covariate selection: candidate covariates derived from retinal OCT images of an earlier study; only the file
  layout was changed (Supplement S2).

All three tools read the same data in every run; the NONMEM files only add the columns `CMT`, `MDV` and `EVID`.
