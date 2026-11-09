# Polarized foregrounds in regions of the QUIJOTE-MFI wide survey

Analysis code of the paper *Spatial variations of polarized synchrotron and
dust parameters from power-spectrum analysis* (de Ferra et al., in preparation).

The EE and BB auto- and cross-spectra of 14 channels (QUIJOTE-MFI 11 and 17 GHz,
WMAP K to W, Planck PR4 30 to 353 GHz) are measured with NaMaster in seven masks,
three Galactic cuts and four regions. A synchrotron plus thermal dust model is
then fitted to them with a Wishart likelihood, whose noise term comes from
half-mission difference maps (HMDMs). A Gaussian likelihood with uncertainties
from simulations gives the comparison of Appendix A, and both likelihoods are
tested on PySM simulations.

Only the analysis is here. The scripts that draw the figures and write the LaTeX
tables of the paper are not included.

## Layout

```
data/                 inputs kept in the repository
  beams/              beam transfer functions of the 14 bands (wmap_da/: WMAP DA beams)
  color_corrections/  colour-correction polynomials and the bandpass-shifted HFI dust table
  cmb/                Planck 2018 best-fitting CMB spectrum
  masks/              QUIJOTE-MFI footprint and the seven apodized masks
  maps/               (not in git) sky maps, HMDMs, PySM maps, noise simulations
results/              (not in git) written by the scripts
python/
  config.py           paths, bands, binnings and masks: every script reads its settings here
  data.py             the 14 channels: frequency, beam and input files
  functions.py        maps, NaMaster spectra, spectra files, beams, unit conversions,
                      CMB, colour corrections, dust SED
  make_masks_galcut.py, make_masks_regions.py
  compute_spectra.py
  inputs/             builders of the derived inputs (HMDMs, WMAP band beams, colour
                      corrections, PySM maps, noise simulations)
  wishart/            Wishart likelihood, its fits and the binning checks
  gaussian/           Gaussian likelihood: simulations, corrected spectra and fits
  sim_tests/          both likelihoods on the PySM sky
```

## Requirements

Python 3.11 with the packages in `requirements.txt`, the versions used for the
paper: healpy, NaMaster (`pymaster` 1.5), emcee, numpy, scipy and astropy, plus
pysm3 and requests for the input builders only.

## Input maps

The maps are not in the repository. Put them, or links to them, in `data/maps/`,
or set the environment variable `SPECIFIC_REGIONS_MAPS` to their directory, with
this layout (the file names are in `python/data.py`):

| Directory | Contents | Source |
|---|---|---|
| `QUIJOTE/` | MFI DR1 maps at 11 and 17 GHz | LAMBDA |
| `QUIJOTE/hmdm/` | DR1 half-survey maps (`_half1`, `_half2`) | LAMBDA |
| `WMAP/` | 9-year band maps, and 9-year DA maps for the beam weights | LAMBDA |
| `WMAP/yearly/` | yearly maps of each DA, for the half-mission maps | LAMBDA |
| `Planck/` | PR4 (NPIPE) BPassCorrected LFI and HFI maps | PLA |
| `Planck/hmdm/` | PR4 half-ring maps | PLA |
| `*/hmdm/*_hmdm.fits` | half-mission difference maps | `inputs/make_hmdm.py` |
| `PYSM/` | PySM sky maps | `inputs/make_pysm_maps.py` |
| `WMAP/noise_simulations/` | white-noise realizations | `inputs/make_wmap_noise_sims.py` |
| `Planck/noise_simulations/<band>_pr4/` | PR4 npipe6v20, realizations 00200-00299 | `inputs/download_planck_noise_sims.py` |
| `QUIJOTE/noise_simulations/` | MFI noise simulations | QUIJOTE collaboration |

The Wishart fits to the data need only the sky maps and the HMDMs. The PySM maps
and the noise simulations are used by the Gaussian likelihood and the simulation
tests.

## Running the analysis

All commands are run from `python/` or from the subfolder shown. Steps already
done in the repository are marked *(done)*.

1. **Derived inputs** (`inputs/`): `make_hmdm.py`, and *(done, in `data/`)*
   `make_wmap_beams.py` and `make_cc_polynoms.py`. For the Gaussian likelihood and
   the simulation tests, also `make_pysm_maps.py`, `make_wmap_noise_sims.py` and
   `download_planck_noise_sims.py`.
2. **Masks** *(done, in `data/masks/`)*: `make_masks_galcut.py` and
   `make_masks_regions.py`.
3. **Spectra**: `compute_spectra.py --hmdm`, the data and HMDM spectra for the
   three binnings at once.
4. **Wishart fits** (`wishart/`): `run_mcmc_power_law.py --binning dl20` and
   `run_mcmc_bin_to_bin.py --binning dl20`. With the same scripts for `dl30` and
   `eqmode`, `check_binning.py` gives the binning tests of Sect. 3.2.
5. **Gaussian likelihood** (`gaussian/`): `run_simulations.py noise` and
   `run_simulations.py skyplusnoise`, 100 realizations per mask, which is the
   expensive step. They can be split into pieces (`--first 1 --last 50`,
   `--first 51 --last 100`) and joined by `combine_simulations.py`. Then
   `correct_spectra.py`, `run_mcmc_power_law.py` and `run_mcmc_bin_to_bin.py`.
6. **Simulation tests** (`sim_tests/`): `reference.py` for the noiseless reference,
   `spectra.py` for the PySM sky plus noise simulation k, `fit.py` for both
   likelihoods, power law or bin to bin, and `combine_fits.py`. Their noise term is
   the mean of the noise simulations, whatever `SPECIFIC_REGIONS_NOISE` says.

The spectra, simulation, correction and MCMC scripts accept
`--masks galcut10,fan,...` to run only some masks (all by default), and the MCMC
scripts accept `--nproc` for the worker processes.

The noise term of the paper, the HMDM spectra, is the default. With
`SPECIFIC_REGIONS_NOISE=sims` both likelihoods take it from the mean of the noise
simulations instead, and write to `results/wishart/` and `results/gaussian/`.

## Where each part of the paper is

| Paper | Code |
|---|---|
| Sect. 2.4, noise estimates | `inputs/make_hmdm.py`, `gaussian/run_simulations.py` |
| Sect. 3.1, sky masks | `make_masks_galcut.py`, `make_masks_regions.py` |
| Sect. 3.2, power spectra and binning | `compute_spectra.py`, `wishart/check_binning.py` |
| Sect. 3.3, foreground model and colour corrections | `functions.py`, `inputs/make_cc_polynoms.py` |
| Sect. 3.4, data model and Wishart likelihood | `wishart/wishart_likelihood.py` |
| Sect. 3.5, 4.1 and 4.2, parameter inference and results | `wishart/run_mcmc_power_law.py`, `wishart/run_mcmc_bin_to_bin.py` |
| Sect. 3.6 and Appendix A.2, tests on simulations | `sim_tests/` |
| Appendix A, Gaussian likelihood | `gaussian/` |

## Reproducibility

The MCMC scripts use fixed random seeds, so a rerun is reproducible. The chains of
the paper were sampled before the seeds were fixed, and a rerun agrees with them
within the Monte Carlo noise. The spectra agree with those of the paper to about
10⁻¹² (relative), because the spherical-harmonic transforms are not
bit-reproducible across machines and thread counts.

## License

The code and the masks are released under the MIT License (see `LICENSE`). The
beam transfer functions, the CMB spectrum and the colour-correction tables in
`data/` come from the Planck Legacy Archive, LAMBDA, the QUIJOTE-MFI DR1 and the
fastcc code, and remain under the terms of their providers.
