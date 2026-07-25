## Rapidtide Troubleshooting (v3.1.10)

- **Background**: rapidtide is a software package that applies lag‑correlation based modelling to fMRI time‑series data to estimate when blood‑borne low‑frequency oscillations (sLFOs) arrive in each voxel. It does this by extracting each voxel’s sLFO, cross‑correlating it with a reference sLFO (eg from the superior sagittal sinus), and estimating the time delay that maximizes the correlation. This eventually produces a whole‑brain map of 'hemodynamic delay' estimates (ie an indirect 'vascular latency' map).
- This gist documents a small bug I identified and helped resolve in rapidtide v3.1.10.

### Summary of the Bug and Fix

- **Data**: The primary goal for this rsfMRI dataset was to examine how vascular risk impacts global delay patterns. To do this, we attempted to extract whole-brain lag maps using [rapidtide (version 3.1.10)](https://github.com/bbfrederick/rapidtide).
We have been working with 3T resting-state scans (5 min, TR = 0.46 s, MB factor = 8). The cohort spans ages 20–70+, mostly young and healthy, with a small subset of subjects exhibiting some vascular pathology (PVS/WMH/stroke etc).

### Failure Modes
- When running rapidtide on minimally preprocessed rsfMRI data (via fMRIPrep), we encountered two major failure modes:
  1) Program errors out (>90% of voxels fail) before the run completes. 
  2) Program runs to completion, but produces largely empty maps with strange lag distributions and error messages (`initlaghigh/fitlaghigh` for eligible delays well within the search range). 

- **Initial Troubleshooting Attempts**: We attempted to troubleshoot this behavior by adjusting a number of external parameters, including (but not limited to) changing the reference regressor (SSS, GM, cerebellum), search range limits,  motion regression confounds, and smoothing levels. None of these resolved the issue.

### Culprit
- After examining the source code, I identified the following behavior:
  - When rapidtide performs an initial correlation fit routine (`--passes`), it attempts to refit outlier delay estimates using a despeckling step (`--despecklepasses`) within each major pass.
  - During this despeckling step, the program resets the local search window (`lagmin`/`lagmax`) to find the "true/correct" delay for that outlier voxel.
  - This voxel-wise search window is intentionally conservative to avoid selecting delays near spurious/'sidelobe'peaks (which arise through 'autocorrelation' in our reference sLFO). 
  - However, after despeckling completes, this modified search window persists and effectively overwrites the global search range (eg -5s to 40s --> drifts down to -1s to 6s) severely restricting the set of allowable delays. This leads to widespread fit failures (`initlaghigh`, `fitlaghigh`) and, ultimately, empty maps.

***Detailed debugging log can be found here***: https://gist.github.com/suchitag07/cb6e6b1395bc52ab35c16c499edd798b

### Relevant Functions/Calls
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
- This causes the global range to drift from the starting/user defined `[-5, 40]` to about `[-9.1, 8.14]` (in our example case). Pass 2 then starts with this narrowed window, true long-lag voxels get systematically flagged as `FITLAGHIGH` and clipped/no longer eligible for despeckling (fall below that threshold).


### Solution/Outcome

- ***Important note: The observed failure pattern was driven by how `lagmin`/`lagmax` drifted during voxel-wise despeckling (which I tracked). In some cases, this resulted in a ~20% fit failure rate (manageable), but in many cases it rose to ~70% ([problematic](https://gist.github.com/suchitag07/cb6e6b1395bc52ab35c16c499edd798b#replication-of-prepost-bug-fix-output-with-revised-command)).***
- To fix this, I reset the search window to the original user-defined values immediately after the despeckling routine is executed in the code `fitSimFuncMap.py line 970-973 theFitter.setrange(optiondict["lagmin"], optiondict["lagmax"])`. This resoved the issue!
- This bug-fix was reviwed and merged into the latest release of [rapidtide version 3.1.11](https://github.com/bbfrederick/rapidtide/releases/tag/v3.1.11)

***Example Rapidtide Call***
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


![](https://github.com/user-attachments/assets/5307a29f-5f87-41b1-8ac4-409b759ea77d)

![](https://github.com/user-attachments/assets/ebe7f3ca-c426-40ec-879b-324865d30d62)

![](https://github.com/user-attachments/assets/892dfe89-be93-4186-b8c8-0e97fd27d5b4)
***
