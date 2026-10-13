# Ground-truth acquisition protocol for aslscan and TRXScan

Date: 2026-10-12. Status: proposal. Codex adversarial review applied 2026-10-13 (24 findings).

This is a scan protocol for one human subject. Its purpose is to measure every map that
aslscan and TRXScan read as input, so that both simulators can run on one real anatomy. It also
lists data worth acquiring for extensions that do not exist yet, and data that is useful only for
validating the simulators.

Scan at the field strength you intend to simulate. aslscan refuses a phantom whose
`MagneticFieldStrength` differs from the protocol's. The values below assume 3 T. Scan times are
rough estimates and depend on the scanner.

**Code versions.** Statements about the simulators were checked against aslscan 3faa1be,
mrsim-acq 5b4dd60, TRXScan `main` 02ca507 and the `resync-mrsim-acq` branch 5c08edf
(PennLINC/TRXScan#13). TRXScan `main` has its own acquisition stage and does not depend on
mrsim-acq; the re-sync branch does, pinned to mrsim-acq e64caff. Statements about a shared
acquisition stage apply to TRXScan only once #13 merges.

## 1. What the "ground truth" will and will not be

- **Perfusion and arrival time are conditioned on a model, not physiologically exact.** They come
  from a kinetic-model fit to real ASL data, and aslscan simulates with a kinetic model of the
  same family. The result is a sound reference for testing reconstruction, acquisition effects
  and processing pipelines. It does not show that a method measures true perfusion. Simulating a
  different ASL protocol from the one used to measure CBF is a cross-protocol check; it does not
  remove the shared-model circularity.
- **"Same model" has to be made true.** aslscan uses tissue T1, a flow-dependent apparent T1 and
  `M0b = M0 / lambda` (`aslscan/src/kinetic.rs`). The fit must use matching choices; see
  section 5, step 3.
- **Measured maps carry their own noise and partial volume.** Noise in a measured map becomes
  fixed texture in the phantom. Light denoising or per-tissue regularization helps.
- **The scan-resolution reference is the simulator's own output.** aslscan computes the signal at
  phantom resolution and averages magnetization down to the scan grid. Evaluations should compare
  against its `ground-truth/desc-*_gt` files, not against the phantom maps.

## 2. Acquisitions

### 2.1 Targets

Derivatives we want and which tool uses each.

1. CBF, ATT, aBAT, aBV, M0 scan (aslscan)
2. Tractography (TRXScan)
3. T1 (aslscan, TRXScan)
4. T2 (aslscan)
5. T2* (aslscan)
6. Relative proton density map/M0 map/S0 map (aslscan, TRXScan)
7. Coil sensitivity map (aslscan, TRXScan)
8. Gradient coefficients (aslscan, TRXScan)
9. Tissue segmentations (aslscan, TRXScan)
10. Labeling efficiency (aslscan)
11. Noise level (aslscan, TRXScan)
12. Physiological traces (aslscan, TRXScan)
13. Motion estimates (aslscan, TRXScan)
14. ihMTSat (TRXScan, optionally replaced with T1)
15. B0 field map (aslscan, TRXScan)


### 2.2 Core: required for inputs that exist today

| # | Acquisition | Key settings | Time | Yields |
|---|---|---|---|---|
| 1 | MP2RAGE | 1 mm isotropic; keep INV1 and INV2 as well as UNI | ~8 min | T1 map; UNI image for segmentation |
| 2 | B1+ map | vendor method (e.g. turbo-FLASH or AFI) | ~1 min | corrects the MP2RAGE T1 and the M0 map |
| 3 | Multi-echo 3D GRE | 1 mm, 6-8 echoes to ~40 ms, low flip angle, magnitude and phase | ~7 min | T2\* map; B0 fieldmap (Hz); M0 map |
| 4 | T2 mapping | 3D-QALAS if available, otherwise a multi-echo spin echo; keep the source images | ~6 min | T2 map |
| 5 | Multi-PLD PCASL | 3D segmented GRASE or stack-of-spirals, background suppression, ~6 PLDs from 0.5 to 3.0 s, highest resolution the SNR allows | 12-15 min | CBF and arterial transit time |
| 6 | ASL M0 calibration | same readout as #5, long TR, no labeling or background suppression | ~1 min | CBF calibration |
| 7 | Coil calibration GRE | fast low-resolution 3D GRE (2-4 mm), uncombined channels with magnitude and phase, plus a matched repeat with body-coil receive | ~2 min | receive-bias field for M0 now; coil sensitivities later (section 4) |
| 8 | CS-DSI, AP and PA | full Fourier, magnitude and phase saved | per current protocol | tractogram, SIFT2 weights, eddy and motion estimates |

Notes:

- **#1-#4, relaxometry in general.** A map is only as good as its fit, and the fit needs more
  than the reconstructed map. Keep the source images (MP2RAGE inversion images, QALAS source
  volumes, individual spin-echo echoes) and the sequence parameters (TR, TI, flip angles,
  preparation timings, RF pulse and slice-profile information where available). Name the fitting
  implementation and the usable TE range before scanning, not after.
- **#3, M0.** The spoiled-GRE signal is
  `S = C_rx * M0 * sin(a) * (1 - E1) / (1 - E1 * cos(a)) * exp(-TE / T2*)`, with `a` the actual
  flip angle. Recovering M0 needs three corrections:
  - T2\* decay: extrapolate to TE = 0.
  - T1 weighting: invert the steady-state term using the T1 map and the B1+-corrected flip
    angle. Record TR, nominal flip angle and the spoiling scheme.
  - Receive bias `C_rx`: estimate it as the smoothed ratio of the array image to the matched
    body-coil image from #7, then divide the M0 estimate by that field. Do not divide by the
    body-coil image itself; it contains anatomy and would remove the contrast being measured.
    The body coil's own receive profile is not perfectly uniform at 3 T and remains as a residual.
- **#3, fieldmap.** Exported phase is not a fieldmap. It needs the vendor's phase scaling,
  unwrapping, `f = dphi / (2 * pi * dTE)`, and a sign convention. Bipolar readouts add an
  odd/even echo offset that must be corrected or avoided (use same-polarity echoes). Fat in the
  scalp breaks a single-species fit; mask it or separate water and fat. Check the resulting
  field by confirming that it predicts the distortion direction seen in the AP and PA diffusion
  images.
- **#4, T2.** Per-voxel precision matters little for today's inputs: TRXScan applies T2 per
  compartment, and aslscan's exact `class` mode uses one T2 per tissue label. It matters once
  TRXScan takes voxel-wise maps (section 4, item 3) and for aslscan's `voxel` mode, so choose the
  sequence for map quality, not only tissue means. A multi-echo spin-echo fit must model
  stimulated echoes, which needs the B1+ map and the refocusing slice profile, not B1+ alone.
  QALAS can need B1+ and inversion-efficiency correction as well. CSF T2 is too long for these
  sequences to measure well; take it from the literature.
- **#5, PCASL.** A 3D readout has little distortion but not none: GRASE has an EPI direction and
  T2 blurring through-plane, and spirals blur with off-resonance. A segmented readout keeps both
  low.
- **#5, arterial blood volume and arterial transit time.** These need a design of their own;
  removing flow crushing is not enough. In aslscan's arterial model the signal exists only while
  `aATT <= t < aATT + tau` (`aslscan/src/kinetic.rs`). With a first PLD of 0.5 s, any arterial
  component whose transit time is shorter than that has already passed and cannot be recovered.
  Add PLDs well below 0.5 s with enough repeats, consider matched crushed and uncrushed
  acquisitions, and check identifiability before scanning. The fitted arterial amplitude also
  has to be converted to aslscan's `abv` (a volume fraction in [0, 1]); the two are not
  interchangeable as output.
- **#7, coil maps.** Uncombined complex images contain object phase as well as coil phase. The
  matched complex body-coil image cancels the object phase only if geometry, timing and
  phase/gain conventions are identical, and the result is a sensitivity relative to that
  reference. A vendor-combined phase image is not a substitute. Record channel order and gain or
  scaling metadata, and keep them consistent with the noise scan (#10).
- **#8, CS-DSI.** Full Fourier lengthens TE and lowers SNR; that is accepted here.

### 2.3 Calibration extras: cheap, and usable today

| # | Acquisition | Time | Yields |
|---|---|---|---|
| 9 | Phase-contrast flow in the brain-feeding arteries | 1-2 min | an effective global CBF calibration |
| 10 | Noise-only scan (no RF), or exported noise prescan | <1 min | noise level now; coil noise covariance later |
| 11 | Pulse and respiration recording during ASL and DWI | none | cardiac and respiratory frequencies and variability now; full traces later |
| 12 | Hematocrit (blood draw), if possible | none | subject-specific arterial blood T1 |
| 13 | Scanner gradient coefficient file (`.grad` on Siemens) | none | gradient nonlinearity in TRXScan |

Notes:

- **#9, phase contrast.** It measures volumetric inflow, not labeling efficiency. An efficiency
  estimate comes from comparing total inflow per unit brain mass with the calibrated ASL result,
  so errors in M0, background-suppression losses and kinetic assumptions are absorbed into it.
  Treat it as an effective global calibration. Measure both internal carotids and both
  vertebrals above the carotid bifurcation (a common-carotid measurement includes external
  carotid flow), set VENC and resolution for those vessels, correct background phase, and
  average over the cardiac cycle. Separate labeling efficiency from background-suppression
  efficiency before setting `label_efficiency`.
- **#10, noise.** A raw prescan variance is not the number either simulator takes. See the
  conversions in section 3.

### 2.4 Optional

| # | Acquisition | Time | Yields |
|---|---|---|---|
| 14 | T2w 3D SPACE/CUBE, 1 mm | ~5 min | better segmentation, especially the pial and CSF boundary |
| 15 | Myelin-sensitive acquisition: ihMT, or MT saturation | 5-15 min | source for TRXScan's `--myelin` map, after conversion |
| 16 | Time-of-flight angiogram | ~5 min | labeling-plane placement only; neither simulator reads it |

Notes on #15:

- **The two are different acquisitions.** ihMTsat (the target in 2.1) needs single-offset and
  dual-offset saturation images. An ordinary MT-weighted GRE does not measure it. MT saturation
  needs PD-weighted, T1-weighted and MT-weighted GRE volumes in the conventional fit; replacing
  the T1-weighted volume with the MP2RAGE T1 needs an explicitly implemented alternative fit.
  Choose one and specify its acquisition and fitting model.
- **`--myelin` is not a physical unit.** It is a dimensionless coefficient in [0, 1] that blends
  the white-matter parameters from the chosen base preset (0) to the adult preset (1)
  (`TRXScan/src/compartments.rs`, `src/mixture.rs`). Raw ihMTsat, MTsat or R1 values cannot be
  passed in. A conversion to [0, 1] has to be defined and validated.
- **The base preset must not be `adult`.** With an adult base, blending toward the adult
  endpoint changes nothing. A non-adult base is not a usable workaround for an adult subject
  either: its fibre T2 applies everywhere and its low end is infant white matter. Using the map
  for an adult needs configurable endpoints (section 4, item 8).
- **R1 as a stand-in** is subject to the same conversion requirement.

### 2.5 Validation scans: not simulator input

| # | Acquisition | Purpose |
|---|---|---|
| 17 | Standard partial-Fourier DWI (the protocol normally run) | compare simulated against real images |
| 18 | Routine ASL (e.g. single-PLD, or 2D EPI) | compare simulated against real images; a cross-protocol check on the perfusion maps |

Save phase images for #17 and #18 as well, to validate the complex output and phase models.

### 2.6 Settings that apply to every scan

- Save images with vendor gradient-distortion correction **off**, or save both versions, so the
  correction is applied deliberately and once (section 5, step 1).
- Record the table position and the scanner-to-image transform for each series, so the
  isocentre is known in image coordinates. Gradient nonlinearity is defined relative to it.
- Save magnitude and phase wherever the scanner allows.
- Export raw data for #7 and #10 if possible (TWIX, ScanArchive, or .raw/.lab).

## 3. What each simulator can use today

### 3.1 aslscan

| Source | Input |
|---|---|
| #1 + #2 | `T1map.nii.gz`; UNI → `dseg.nii.gz` |
| #3 | `T2starmap.nii.gz`, `fieldmap.nii.gz` (Hz), `M0map.nii.gz` |
| #4 | `T2map.nii.gz` |
| #5 + #6 | `perfusion.nii.gz`, `att.nii.gz`; with a suitable design, `abv.nii.gz`, `aatt.nii.gz` |
| #7 (array / body-coil ratio) | receive-bias correction of `M0map` |
| #9 | `label_efficiency` in the overlay, as an effective value |
| #10 | `noise_variance` (one scalar, after conversion to the simulator's signal units) |
| #11 | `[physio] cardiac_frequency`, `respiratory_frequency`, `cardiac_cv`, `respiratory_cv` |
| #12 | `t1_arterial_blood` (overlay) or `T1ArterialBlood` (`phantom.json`) |
| motion estimates from any real time series | `[motion] trajectory` TSV |

### 3.2 TRXScan

| Source | Input |
|---|---|
| #8 | tractogram and SIFT2 weights; motion → `--motion` |
| #8, after a fitting step | `--eddy`, `--eddy-quad`, `--eddy-phase`; `--diff-scale` |
| #1 (+ #14) | WM/GM/CSF tissue-fraction maps |
| #3 (phase) | fieldmap: `--sim-fmap` on the simulation grid, or `--fmap` with `--oversample 1` |
| #13 | `--gnl <file>`, with `--isocenter` |
| #15, after conversion to [0, 1] | `--myelin` map; has an effect only with a non-adult base preset (section 4, item 8) |
| #10, after conversion | `--noise` (a variance), or a per-voxel `--noise-map` (a standard deviation) |
| M0 map and T1 map (per-tissue means) | `--tissue-s0` |

Conversions these rows depend on:

- **Eddy currents.** The eddy options are scalar strengths of a synthetic model; they do not
  read FSL `eddy` parameter files. `--eddy-phase` describes a phase ramp on the object. A
  fitting procedure from the real data to these scalars has to be defined.
- **Noise.** `--noise` is the variance of each of the real and imaginary components of the
  reconstructed image at full sampling, single coil, before combination. `--noise-map` is a
  standard deviation. A prescan variance has to be carried through gain, bandwidth,
  reconstruction normalization and coil combination into the simulator's signal units. The same
  applies to aslscan's `noise_variance`.
- **`--tissue-s0`.** It scales the compartment images before relaxation; the acquisition stage
  then applies T2 decay. Set it from proton density times T1 saturation at the diffusion TR,
  using the M0 and T1 maps. Do not copy measured b=0 intensities: they already contain TE
  decay and receive weighting, which would be applied again.
- **`--diff-scale`.** It changes diffusivities, which b=0 images cannot identify. Estimate it
  from the attenuation across the non-zero b-values.

Physiological traces (target 12 in 2.1) have no consumer in TRXScan today.

## 4. Extensions worth building

Items 1-6 are in order of value; items 7 and 8 are stated requirements. None of these exists
today.

1. **Measured coil sensitivities and noise covariance (both simulators).** mrsim-acq generates
   sensitivities analytically (`coil_sensitivity`, `src/kspace.rs`): real-valued Gaussian
   falloffs from points on a ring, with no phase. A single ring has no variation along z; spiral
   readouts can use several rings with Gaussian z profiles (`coil_rows`). TRXScan `main` carries
   its own copy of the single-ring model. Measured complex maps and correlated per-coil noise
   would be the largest realism gain for parallel imaging and spatially varying noise. The coil
   combine should then use sensitivities estimated from the simulated calibration data, not the
   true maps. The input must define the phase reference, channel order and whitening convention
   (section 2.2, note on #7). Data: #7 and #10.
2. **Gradient nonlinearity in aslscan.** TRXScan already models it; this is mostly a move into the
   shared crate. Data: #13.
3. **Voxel-wise T2 and T2\* maps in TRXScan** (PennLINC/TRXScan#16). The goal is voxel-wise map
   inputs, not per-tissue overrides. Today the presets give one T2 each for fibre, grey matter
   and CSF. There is no compartment T2\*: the re-sync adapter passes each T2 as a uniform value
   and leaves the per-compartment T2' unset, so one acquisition-level T2' applies everywhere
   (`TRXScan-resync/src/kspace.rs`); `main` likewise uses a scalar. mrsim-acq can take a
   per-voxel map for each compartment's T2 and T2' (`T2Volume::Map`), but TRXScan's adapter does
   not expose that. The work is therefore map loading and validation, conversion of measured T2
   and T2\* to T2', and changes to the adapter and its inputs. Decisions:
   - **Mixed voxels.** A fitted voxel T2 is not a compartment T2: a sum of exponentials is not
     one exponential. Assigning the voxel value to the tissue compartments while CSF keeps its
     own long T2 counts the CSF contribution twice wherever CSF is present. Prefer
     tissue-conditioned estimates, from sufficiently pure voxels or from a constrained
     compartment fit that uses the tissue fractions. If a shared-map approximation is kept,
     quantify its error across CSF fractions and TE; it has no general bound.
   - **T2\* versus fieldmap dephasing.** See section 7.

   Map inputs (`--s0-map`, `--t2-map`) existed before the re-sync and were removed. Data: #3
   and #4.
4. **Recorded physiological traces in aslscan.** `[physio]` already produces variable cycles:
   it takes amplitudes, two frequencies and their coefficients of variation (`cardiac_cv`,
   `respiratory_cv`), plus stochastic drift. What it cannot do is replay a measured recording.
   The extension is importing measured traces, synchronized to the acquisition, with their real
   waveform shapes. Data: #11.
5. **B1+ in the simulations.** A measured flip-angle map would scale excitation, refocusing and
   background-suppression pulses per voxel. It matters most for aslscan's background suppression
   and Look-Locker readouts. How much of the pulse modelling can take a per-voxel scale has not
   been checked. Data: #2.
6. **A voxel-wise S0 map in TRXScan.** Today `--tissue-s0` takes one amplitude per compartment
   (proton density x T1 saturation), which can be set from the per-tissue means of the M0 map and
   the T1 map. A map input would add within-tissue variation. It is a small change: TRXScan keeps
   one image per compartment through the signal stage, and `apply_s0` (`src/compartments.rs`)
   already multiplies each by a scalar; the acquisition stage is untouched. An earlier
   `--s0-map` (commit 57858a5, paired with `--t2-map`) is a reference. It ranks last because
   proton density varies little within a tissue, the tissue-fraction maps already carry most of
   the contrast, and the larger intensity variation in real images is coil shading (item 1).
   Data: #3, with #1 and #2 for the corrections. Decisions to make:
   - **Double counting.** A measured map already contains the WM/GM/CSF differences. When a map is
     given, it carries proton density and `--tissue-s0` carries only T1 saturation.
   - **One map or one per compartment.** A measured map describes the whole voxel. Multiplying
     every compartment by the voxel average keeps the total equilibrium amplitude but changes
     the compartments' relative contributions compared with compartment-specific proton
     densities, most where CSF is present. As for T2, tissue-conditioned maps are the better
     input, and the error of a shared map should be quantified, not assumed small.
   - **Grid.** The map must be on the signal grid, as `--myelin` must when `--oversample` > 1.
   - **Motion.** `--myelin` is refused with `--motion`; whether an S0 map can be applied on that
     path has not been checked. It would have to move with the head.
   - **Ground truth.** Like `--tissue-s0`, it scales the simulated signal only; the
     microstructure truth maps are normalized by S0 and do not change.
   - **Python bindings** need the same option.
7. **`--myelin` together with `--motion` in TRXScan.** This is a requirement for the planned
   simulations, not a ranked option. Today the combination is refused. With a motion trace,
   TRXScan switches to the per-segment signal stage: for each volume it moves the streamlines and
   tissue maps by that volume's pose and re-simulates. That path has no orientation mixture, and
   the myelin lerp (WM parameters blended per voxel toward the adult endpoint) lives on the
   mixture (`src/compartments.rs`, `src/mixture.rs`). The same switch also ignores SIFT2 weights
   and the Watson kappa, and refuses `--gnl` and `--truth-peaks`, so a motion run currently uses
   a different signal model from the matching no-motion run. Two routes, of which the broad one
   is recommended:
   - **Broad (recommended):** make the motion path use the orientation mixture, rotated and
     resampled per pose. This restores myelin, SIFT2 weights, kappa and truth peaks, and makes
     motion and no-motion runs share one signal model. The planned simulations need SIFT2
     weights and gradient nonlinearity with motion as well as myelin, so the narrow route would
     leave most of the gap open. Neither route has been sized against the code. Its scope is
     more than transporting the mixture:
     - **Gradient nonlinearity is fixed in the scanner frame.** Moving the mixture does not by
       itself give correct motion with GNL. The moved anatomy has to be evaluated against the
       field at its new location, for both the encoding deviation and the spatial warp. Today's
       GNL signal evaluation uses a field on the mixture grid (`src/compartments.rs`).
     - **Scalar maps** (myelin, and the S0 and relaxation maps of items 3 and 6) have to be
       transported with the head.
     - **Truth frame.** Decide whether truth peaks and microstructure maps are written per
       volume or in the reference pose.
     - **Tests.** Identity poses reproduce the no-motion output; translated anatomy in a
       non-uniform GNL field; rotated anisotropic tissue.
   - **Narrow (fallback):** add the myelin lerp to the per-segment path, resampling the myelin
     map by each volume's pose as the tissue maps are. Weights, kappa, GNL and truth peaks stay
     unavailable with motion.

   Data: #15 after conversion for the map; motion estimates from #8 or #17 for the trace.
8. **Configurable endpoints for `--myelin` in TRXScan** (PennLINC/TRXScan#19). Also a
   requirement: without it the myelin map from #15 cannot be used for an adult subject. The
   blend runs from the base preset (map value 0) to the adult preset (1), and the adult end is
   hard-coded (`src/compartments.rs`, `src/microstructure.rs`). The feature was designed for
   developing brains, with a neonatal or infant base. For an adult:
   - with `--params adult` both ends are the same parameters, so the map does nothing, and
     nothing warns about it;
   - with an infant or neonatal base, the low end is unmyelinated infant white matter, which is
     not what a less-myelinated adult tract looks like;
   - T2 is not blended. The blend covers the intra-axonal fraction and the intra- and
     extra-axonal diffusivities. Fibre T2 comes from the base preset in every voxel, so an
     infant base gives 180 ms everywhere, against 68 ms for adult, and distorts the b=0
     contrast.

   The extension lets the caller choose both ends (for example adult and a "low-myelin adult"
   parameter set), warns or refuses when the ends are identical, applies the same endpoints in
   the ground-truth tool, and either blends fibre T2 too or leaves T2 to the maps of item 3.
   The link from myelin content to diffusion parameters is a modelling assumption, so the
   simulated tract-to-tract variation is plausible, not measured. Data: #15.

## 5. Processing into an aslscan phantom

1. **Geometry first.** The phantom must describe undistorted physical anatomy, because the
   simulators add distortion themselves. Registering an uncorrected image to the MP2RAGE does
   not remove its distortion, and an already warped map passed through a simulated warp is
   warped twice.
   - Correct gradient nonlinearity in every anatomical and quantitative map (3D correction,
     applied once, from the uncorrected images and the coefficient file).
   - Correct the ASL data for whatever susceptibility distortion its readout has.
   - For the diffusion data, correct motion, eddy currents, susceptibility distortion and
     gradient nonlinearity (geometry and encoding) before tractography.
   - Keep the transform from scanner coordinates to the phantom grid, and the isocentre in
     phantom coordinates, so a simulated `--gnl` is applied in the intended frame.
2. **One grid.** Register every corrected map to MP2RAGE space and resample to a 1 mm grid with
   an axis-aligned affine. aslscan's resampler refuses a rotated affine, and all maps must share
   one grid.
   - Resample the segmentation with nearest-neighbour or another label-preserving method;
     aslscan refuses fractional labels.
   - aslscan refuses non-finite values anywhere, including background. Fitted maps commonly
     contain NaNs and failed voxels: mask them explicitly and document how each is imputed or
     excluded.
   - After resampling, clip perfusion and arrival time to non-negative values and re-apply the
     background mask so M0 is exactly 0 outside the foreground.
3. **CBF and arrival time.** Fit with BASIL (`oxford_asl`) using spatial priors and
   partial-volume correction (PVC).
   - **Match the model.** Document the BASIL model and every parameter: labeling duration, the
     PLD and time conventions, blood and tissue T1, labeling efficiency, lambda and the
     calibration method. These must agree with what aslscan will be given. Prove it with a
     round trip: simulate a noiseless dataset from known maps with aslscan, fit it with the same
     BASIL settings, and recover the maps.
   - **Use the calibrated outputs.** BASIL's plain `perfusion` output is in scanner units; the
     calibrated outputs (`_calib`) are in ml/100g/min. PVC writes separate grey- and
     white-matter estimates. Build the phantom map from the calibrated PVC estimates and save
     the result as `perfusion.nii.gz`.
   - **PVC needs tissue fractions, not labels.** Give it probabilistic grey- and white-matter
     fractions transformed through the ASL geometry and resolution, not the hard segmentation.
   - **What the result is.** PVC does not measure perfusion at 1 mm. Assigning the
     tissue-specific estimates to high-resolution labels imposes the anatomical structure of the
     segmentation. Call it a model-conditioned phantom, keep the fit's uncertainty maps, and
     decide in advance how to handle poorly constrained estimates, white-matter arrival time
     above all.
   - Set perfusion and arrival time to 0 in CSF.
   - Arterial maps need the dedicated design in section 2.2 and a conversion to a fraction.
4. **Relaxation maps against aslscan's limits.**
   - Measured T2 and T2\* vary voxel to voxel. Either run `--t2-mode voxel`, or replace them with
     per-label medians to use `class` mode. `auto` picks `class` only when the maps allow it.
   - With a 3D echo train refocused below 180 degrees, `class` mode also requires T1 to be
     constant within each label, because the stimulated echoes depend on T1.
   - The 3D gradient-echo EPI train limits the number of (compartment, T1) groups, not simply
     the number of distinct T1 values. The limit is `[readout] max_t1_groups`, default 16, and
     can be raised. Quantize T1 with the group count in mind, or use per-label values.
   - Foreground T1, T2, T2\* and M0 must be positive.
   - Check that T2\* < T2 in every foreground voxel. Where it is not, aslscan silently treats
     T2' as infinite (`t2prime_ms_of`, `src/phantom.rs`).
5. **Units and sidecars.** Each map needs a JSON sidecar with `Units`. aslscan compares the
   string literally and does not convert, so the numbers must already be in these units:

   | Map | `Units` |
   |---|---|
   | `perfusion` | `ml/100g/min` |
   | `att`, `aatt`, `T1map`, `T2map`, `T2starmap` | `s` |
   | `M0map` | `arbitrary` |
   | `dseg` | `label indices` |
   | `fieldmap` | `Hz` |
   | `abv` | `fraction` |

   Convert milliseconds and percentages numerically. `phantom.json` carries
   `MagneticFieldStrength`, `LambdaBloodBrain` and `T1ArterialBlood`.
6. **Checks.** Run numerical checks before the simulator (finite values, integer labels, units,
   T2\* < T2, zero background M0). Then: aslscan loads the phantom; a noiseless run's
   ground-truth outputs look as expected; bids-validator passes on the output.

**TRXScan inputs.** The reference kit (`data/trxscan_truth_data/PROVENANCE.md`) describes the
ingredients: tissue probability maps, an ACT + SIFT2 tractogram converted to TRX, and a
fieldmap. Its commands use an older positional syntax and should not be copied. The current CLI
uses named options and oversamples by default, which needs tissue maps and a fieldmap
(`--sim-fmap`) on the simulation grid as well as the acquisition grid, with consistent affines.
An acquisition-grid fieldmap (`--fmap`) is used only with `--oversample 1`.

## 6. Session plan

The full list is well over two hours, so plan two sessions.

- B0 fieldmaps and coil sensitivities depend on head position and shim. Acquire #3 and #7 in the
  same session as the data they pair with; repeat #7 in each session.
- Repeat a fast anatomical scan in each session for registration.
- A natural split: session A holds the relaxometry and ASL (#1-#7, #9, #18); session B holds the
  diffusion (#8, #17) with its own fieldmap and coil calibration.

## 7. Open questions

- **T2\* and fieldmap dephasing overlap.** The mechanism is now known; the right correction is
  not. aslscan derives `R2' = R2* - R2` from the two maps and applies it as an exponential. It
  also applies the fieldmap as a complex phase on the simulation grid, including the phase
  accrued by TE for gradient echoes (`mrsim-acq/src/kspace.rs`). Summing simulation cells with
  different phases produces signal cancellation on top of the supplied decay, and nothing
  separates the two. Measured T2\* should therefore be understood as the decay not resolved by
  the simulation-grid field. How much the two overlap depends on the measurement and simulation
  resolutions and has to be characterized for the actual grids. Removing all macroscopic
  dephasing from the T2\* map would be wrong, and a with/without-fieldmap comparison does not
  by itself prove the absence of double counting.
- **Choice and conversion of the myelin measure.** ihMTsat or MTsat, the fitting model, the
  mapping to `--myelin`'s [0, 1], and the two endpoint parameter sets (section 2.4; section 4,
  item 8).
- **Arterial compartment design.** Early PLDs, repeats, and crushed versus uncrushed pairs
  (section 2.2).
- **Runtime at fine phantom resolution.** Not measured for a full-brain 1 mm measured phantom in
  `voxel` T2 mode with a fieldmap.
- **Sequence availability.** QALAS, uncombined reconstruction and raw-data export depend on the
  scanner and its licences.

Resolved since the first draft: aslscan applies the fieldmap to spin-echo stack-of-spiral
readouts as off-resonance during the readout (`mrsim-acq/src/spiral.rs`); this is not a claim
about arbitrary non-EPI sequences.
