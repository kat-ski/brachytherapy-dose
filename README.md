# brachytherapy-dose

A dose calculator for iodine-125 brachytherapy seeds (model 6711), written in
Python using the TG-43 formalism. It gives the dose rate at any point around a
single seed or an arrangement of several, and maps the result. Calculated dose
rates were compared against Carleton University's Monte Carlo reference tables
for this seed across 143 points: 97.2% agree to within 1%, with a maximum
discrepancy of 2.1%.

Carleton page to find data for the 6711 seed available here: https://physics.carleton.ca/clrp/node/27

AAPM consensus report: https://irochouston.mdanderson.org/rpc/BrachySeeds/TG-43-Update-1.pdf

<img width="652" height="498" alt="Screenshot 2026-10-07 at 17 34 01" src="https://github.com/user-attachments/assets/09840fc8-d3a9-415a-9145-261ea49c7719" />

## Background

Brachytherapy treats cancer by placing a radioactive source inside or beside the
tumour, rather than directing a beam at it from outside. In permanent seed
implants, small sealed sources are placed in the tissue and left there, giving
out dose as the isotope decays. Iodine-125 is commonly used for prostate
implants: it emits low-energy photons, so the dose falls away steeply with
distance and nearby healthy tissue is largely spared.

Planning a treatment therefore requires knowing the dose rate at every point
around a seed, and around a group of seeds placed together. Simulating photon
transport for each plan would be impractical in a clinic, so the field uses a
standard parameterised model instead.

## The TG-43 formalism

TG-43 is the dose calculation protocol published by the American Association of
Physicists in Medicine, and it is what clinical planning systems use. Rather
than simulating the physics each time, it expresses the dose rate as a product
of factors, each describing one effect:

**D(r, θ) = S_K · Λ · [G_L(r, θ) / G_L(r₀, θ₀)] · g_L(r) · F(r, θ)**

- **S_K**, the air-kerma strength, describes how radioactive the seed is. It is
  set to 1 U throughout this code; typical prostate seeds are 0.4–0.6 U.
- **Λ**, the dose-rate constant, converts that strength into a dose rate at the
  reference point: 1 cm from the seed, on the perpendicular bisector of its long
  axis. For the 6711 seed, Carleton give Λ = 0.932 cGy h⁻¹ U⁻¹.
- **G_L(r, θ)**, the geometry function, accounts for the radioactivity being
  spread along a line inside the seed rather than concentrated at a point, so
  that the falloff is not quite inverse-square. The active length is taken as
  L = 0.28 cm.
- **g_L(r)**, the radial dose function, accounts for attenuation and scattering
  in tissue as distance increases.
- **F(r, θ)**, the anisotropy function, accounts for the seed not being
  spherically symmetric: the titanium capsule and the silver core absorb more
  radiation along the seed's length than across it.

g_L and F are published as tables for each seed model, so the code has to
interpolate between tabulated values. The parameter values used here are
Carleton's, which are derived from Monte Carlo simulation, because the
validation data is also theirs and the comparison is therefore internally
consistent. Clinics use the AAPM consensus values instead, and a clinical
version of this code should do the same.

## What the code does

**`Single_Seed_Dose_6711.ipynb`** implements the formalism for one seed and
validates it.

- `g_L(r)` interpolates the radial dose function. Because g_L falls roughly
  exponentially, the interpolation is linear in log space and then exponentiated
  back, which is more accurate than interpolating the raw values. The table runs
  from 0.05 cm to 10 cm.
- `geometry_function(r, theta_deg)` implements the line-source geometry
  function. On the seed's axis, at θ = 0° or 180°, the general formula divides by
  sin θ and so breaks down; the code detects this case and uses the on-axis
  limit 1/(r² − L²/4) instead.
- `anisotropy_function(r, theta_deg)` interpolates F bilinearly over a 32 × 12
  grid of angles and distances using `RegularGridInterpolator`. The table only
  covers 0° to 90°, so angles beyond 90° are folded back by symmetry.
- `dose_rate(r, theta_deg, S_K)` assembles the five factors.

**`Seed_Dose_6711.ipynb`** extends this to several seeds and produces a map.

- `total_dose_rate(point, seeds, S_K)` sums the contribution of each seed at a
  given point, since dose superposes. Seeds are all taken as aligned along the
  z-axis. Points more than 10 cm from a seed are skipped, as that is beyond the
  tables and the contribution is negligible.
- The dose map takes a slice through the arrangement and evaluates the total
  dose rate on an 81 × 81 grid, plotted on a logarithmic colour scale because
  the dose varies over several orders of magnitude across a few centimetres.

## Validation

The calculated dose rates were compared against Carleton's "along and away"
table, which gives Monte Carlo dose rates on a grid of distances along and
perpendicular to the seed axis. Points inside the seed itself have no reference
value and were skipped, as were points beyond 10 cm, leaving 143 points.

97.2% of points agree to within 1%. The largest disagreement is 2.09%.

The disagreements are not spread evenly: they cluster at small θ, close to the
seed's long axis. The three worst points are all at θ ≤ 15°, and the single
worst is directly on the axis at 1.5 cm. This is where the anisotropy function
changes most steeply with angle, and where the published table is most sparsely
sampled, so linear interpolation between tabulated values has the furthest to
travel. The discrepancy is therefore an artefact of interpolating a coarse
table, not an error in the implementation of the formalism. Using a finer
interpolation scheme near the axis would reduce it.

## A bug found while testing

Testing the multi-seed function at deliberately awkward points revealed that the
on-axis geometry formula, 1/(r² − L²/4), becomes negative when r < L/2, which is
to say for any point inside the seed's own capsule. The code was returning
negative dose rates there.

Such points are physically meaningless, but a dose calculator should not return
a negative number for any input. Points within 0.25 cm of a seed are now clamped
to r = 0.25 cm, which is also the innermost distance in the anisotropy table, so
the same bound serves both purposes.

## Running it

Both notebooks run in Google Colab, which is how they were written. They need
`numpy`, `pandas`, `matplotlib` and `scipy`, all of which Colab provides. They
expect `I125_OncoSeed-6711.xlsx`, downloaded from Carleton's seed database, in a
`Brachytherapy_Project` folder in Google Drive. The path is set at the top of
each notebook and can be changed to run them locally.

## Next steps

The intention is to train a model to outline the prostate in publicly available
MRI scans, and then compare the dose calculated from those outlines against the
dose calculated from outlines drawn by clinicians, using clinical metrics such
as D90 and V100.

The question underneath this is what "ground truth" means here at all. Expert
contours are themselves inconsistent between clinicians, so a segmentation model
is being measured against a standard that varies. What matters clinically is not
how closely a predicted outline matches one expert's, but how much the dose
changes when it does not.

## References

- Carleton Laboratory for Radiotherapy Physics, seed database:
  https://physics.carleton.ca/clrp/node/27
- Rivard et al. (2004), *Update of AAPM Task Group No. 43 Report*:
  https://irochouston.mdanderson.org/rpc/BrachySeeds/TG-43-Update-1.pdf
