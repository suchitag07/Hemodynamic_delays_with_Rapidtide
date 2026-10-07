## Rapidtide Troubleshooting (v3.1.10)

- **Background**: rapidtide is a software package that applies lag‑correlation based modelling to fMRI time‑series data to estimate when blood‑borne low‑frequency oscillations (sLFOs) arrive in each voxel. It does this by extracting each voxel’s sLFO, cross‑correlating it with a reference sLFO (eg from the superior sagittal sinus), and estimating the time delay that maximizes the correlation. This eventually produces a whole‑brain map of 'hemodynamic delay' estimates (ie an indirect 'vascular latency' map).
- This repo documents a small bug I identified and fixed in rapidtide's delay-fitting routine.
- **Link to the release version with fix : [rapidtide version 3.1.11](https://github.com/bbfrederick/rapidtide/releases/tag/v3.1.11)**

### Summary of the Bug and Fix

- **Data**: The primary goal for this rsfMRI dataset was to examine how vascular risk impacts global delay patterns. To do this, we attempted to extract whole-brain lag maps using rapidtide. We have been working with 3T resting-state scans (5 min, TR = 0.46 s, MB factor = 8). The cohort spans ages 20–70+, mostly young and healthy, with a small subset of subjects exhibiting some vascular pathology (PVS/WMH/stroke etc).

### Failure Modes
- When running rapidtide on minimally preprocessed rsfMRI data (via fMRIPrep), we encountered two major failure modes:
  1) Program errors out (>90% of voxels fail) before the run completes. 
  2) Program runs to completion, but produces largely empty maps with strange lag distributions and error messages (`initlaghigh/fitlaghigh` for eligible delays well within the search range). 

- **Initial Troubleshooting Attempts**: We attempted to troubleshoot this behavior by adjusting a number of external parameters, including (but not limited to) changing the reference regressor (SSS, GM, cerebellum), search range limits,  motion regression confounds, and smoothing levels. None of these resolved the issue. 

- **Clue**: Given that a large chunk of fit failures were for lags exceeding the active search window: (`initlaglow/high`, or `fitlaglow/high`), I took that as a hint and went into the source code to inspect how the lagmin/max thresholds were being applied to pass and fail voxels.

### Culprit
- After examining the source code, I identified the following behavior:
  - When rapidtide performs an initial correlation fit routine (`--passes`), it attempts to refit outlier delay estimates using a despeckling step (`--despecklepasses`) within each major pass.
  - During this despeckling step, the program resets the local search window (`lagmin`/`lagmax`) to find the "true/correct" delay for that outlier voxel.
  - This voxel-wise search window is intentionally conservative to avoid selecting delays near spurious/'sidelobe'peaks (which arise through 'autocorrelation' in our reference sLFO). 
  - However, after despeckling (refitting) completes, this modified search window persists and effectively overwrites the global search range (eg -5s to 30s --> drifts down to -4s to 0.5s) severely restricting the set of allowable delays. This leads to widespread fit failures (`initlaghigh`, `fitlaghigh`) and, ultimately, empty maps.
  - ***In essence, this is an object-mutation bug: the local `lagmin/lagmax` values used during despeckling persist and overwrite the global search range. Resetting these parameters after the inner despeckling passes breaks this state leak and restores the intended global search window***

### Relevant Modules/Functions/Calls
```
1) `rapidtide.py` (initializes `theFitter` object which holds user defined parameters)

2) `simFuncClasses.py` (class `SimilarityFunctionFitter`):
      -----> defines `setrange` (which sets lag search window)
      -----> defines `fit` (which screens and assigns pass/fail reasons)
      
3) `simfuncfit.py`: 
      -----> defines `fitcorr`
      -----> `fitcorr` calls `_procOneVoxelFitcorr`, `onesimfuncfit`
      -----> `onesimfuncfit` calls `fit` (during outer `pass`) and `setrange` (during inner `despecklepasses`)

4) `fitSimFuncMap.py` calls `fitcorr` with `theFitter`

```
  - `theFitter` is an object initialized inside `rapidtide_main` that holds user defined parameters for the fitting process (eg `optiondict[lagmax]`). 
  - During fitting, this object is passed into `fitcorr`, then to `_procOneVoxelFitcorr`, and finally `onesimfuncfit`. 
  - During despeckling, `onesimfuncfit` calls on `setrange` via `thefitter.setrange(initiallag - despeckle_thresh / 2, initiallag + despeckle_thresh / 2)`; creating a **conservative voxelwise search window during despeckling/refitting** (to ensure we don't select lags near the sidelobe peak). `setrange` assigns the first argument to `lagmin`, and the second to `lagmax`.
  - Once this window is updated, the ***new*** `lagmin` and `lagmax` persist, essentially mutating the global window (`theFitter`) that is used across passes.
- This causes the global range to drift from the starting/user defined `[-5, 30]` to about `[-4, 0.5]` (in our example case). Pass 2 then starts with this narrowed window, true long-lag voxels get systematically flagged as `FITLAGHIGH` and clipped/no longer eligible for despeckling (fall below that threshold).


### Solution/Outcome

- Important note: The observed failure pattern was driven by how `lagmin`/`lagmax` drifted during voxel-wise despeckling (which I tracked). In some cases, this resulted in a ~20% fit failure rate (manageable), but in many cases it rose to ~70% (problematic).
- ***To fix this, I reset the search window to the original user-defined values immediately after the despeckling routine is executed in the code `fitSimFuncMap.py line 970-973 theFitter.setrange(optiondict["lagmin"], optiondict["lagmax"])`. This resoved the issue!***

***[Link to detailed debugging log, print-statement traces, and pre/post-fix outputs](https://github.com/suchitag07/Hemodynamic_delays_with_Rapidtide/blob/main/Debugging_Log.md)*** 

***Example Rapidtide Call***

- We specified an input `--searchrange` of `-5 to 30 seconds`.

```
rapidtide \
	/path_to_data/fmriprep/sub-${subjID}/ses-01/func/sub-${subjID}_ses-01_task-rest_desc-preproc_bold.nii.gz \
	/path_to_data/latest/Rapidtideout/Bug_test_SG/sub-${subjID}/sub-${subjID}
	--numnull 10000 \
	--filterband lfo \
	--preppass \
	--sharpenregressor \
	--searchrange -5 30 \
	--passes 3 \
	--nofitfilt \
	--simcalcrange 130 -1 \
	--corrmask /path_to_data/fmriprep/sub-${subjID}/ses-01/func/sub-${subjID}_ses-01_task-rest_desc-brain_mask.nii.gz \
	--globalmeaninclude /path_to_data/fmriprep/sub-${subjID}/ses-01/fMRIPrep_parc_bold_space/sub-${subjID}_ses-01_task-rest_space-bold_desc-aparcaseg.nii.gz:8,47 \
	--refineinclude /path_to_data/fmriprep/sub-${subjID}/ses-01/fMRIPrep_parc_bold_space/sub-${subjID}_ses-01_task-rest_space-bold_desc-aparcaseg.nii.gz:8,47 \
	--whitemattermask /path_to_data/fmriprep/sub-${subjID}/ses-01/fMRIPrep_parc_bold_space/sub-${subjID}_ses-01_task-rest_space-bold_desc-WM_probseg_bin.nii.gz \
	--csfmask /path_to_data/fmriprep/sub-${subjID}/ses-01/fMRIPrep_parc_bold_space/sub-${subjID}_ses-01_task-rest_space-bold_desc-CSF_probseg_bin.nii.gz \
	--motionfile /path_to_data/fmriprep/sub-${subjID}/ses-01/func/sub-${subjID}_ses-01_task-rest_desc-confounds-trimmed_timeseries_revised_final.tsv \
	--motpowers 2 \
	--mklthreads 1 \
	--nprocs 1 \
	--numskip 0 \
	--outputlevel max
```

### Test Example Run of Pre/Post-Fix

- **In the original run**: This participant had a frontal-lobe stroke. Rapidtide failed to map delays in roughly 70% of voxels, with a large proportion of fit failures flagged as `highlagfails`. By the end of the run, the internal `lagmin/lagmax` parameters had been truncated from a user defined input of `-5 to 30 s` down to `-4.407391 and 0.592608`, clipping and rejecting delays outside this range.
- Note: I tracked the mutation of the fitting object parameters by inserting print statements throughout the source code (you would not see these in the standard terminal output/run). I've described an example of my tracing steps in this section of my `Debugging_Log` : See [Step-by-Step Tracing](https://github.com/suchitag07/Hemodynamic_delays_with_Rapidtide/blob/main/Debugging_Log.md#step-by-step-tracing)

![](https://github.com/user-attachments/assets/5307a29f-5f87-41b1-8ac4-409b759ea77d)

- **Post-patch run**: After patching the parameter state leak, rapidtide internally retained the user defined input `--searchrange` (-5 to 30 s in this case). The patched run successfully mapped the full range of delays, importantly recovering longer hemodynamic delays within the lesioned region.

![](https://github.com/user-attachments/assets/ebe7f3ca-c426-40ec-879b-324865d30d62)

### Additional Examples of Pre/Post-Fix

- Here are some additional examples of how the bug affected our outputs in version 3.1.10 (specifically truncating the search range for lag detection and causing widespread fit failures); and how our patched run resolved the issue consistently across participants. You can see that both the `Overlay Histogram` and `Correlation function` pick up/reflect delays across the full search range.

![](https://github.com/user-attachments/assets/9657a3a3-4bd8-402e-9993-89b4fa6454c3)
***
