## Summary of the Bug and Fix

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

### Solution/Outcome

- ***Important note: The observed failure pattern was driven by how `lagmin`/`lagmax` drifted during voxel-wise despeckling (which I tracked). In some cases, this resulted in a ~20% fit failure rate (manageable), but in many cases it rose to ~70% ([problematic](https://github.com/suchitag07/Hemodynamic-delays-using-Rapidtide-BHP/blob/main/Debugging_Log.md#replication-of-prepost-bug-fix-output-with-revised-command)).***
- To fix this, I reset the search window to the original user-defined values immediately after the despeckling routine is executed in the code. This resoved the issue!
- This bug-fix was reviwed and merged into the latest release of [rapidtide version 3.1.11](https://github.com/bbfrederick/rapidtide/releases/tag/v3.1.11)

### [Jump to Pre/Post-Fix Outputs here](https://github.com/suchitag07/Hemodynamic-delays-using-Rapidtide-BHP/blob/main/Debugging_Log.md#replication-of-prepost-bug-fix-output-with-revised-command)

***

## Code/Troubleshooting Process: Summary

### Problem Example Case
- Lags beyond +/-8s were being flagged as outliers despite a broad user-defined search range of 
`[-5:40]` ([see example case and initial test command](https://github.com/suchitag07/Hemodynamic-delays-using-Rapidtide-BHP/blob/main/Debugging_Log.md#Initial-test-command)). This was showing up as a high proportion of fit failures in our logs (`initlaghigh, fitlaghigh`). 

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

### Minimal patch
- I attempted a small patch in `fitSimFuncMap.py line 970-973`, that resets the lagmin and lagmax to the user specifed window after `fitcorr` is called.

```
global_lagmin = optiondict["lagmin"]
global_lagmax = optiondict["lagmax"]
theFitter.setrange(global_lagmin, global_lagmax)
```
### Outcome
- This worked! It restored the global fitter window to `[-5, 40]` while leaving the internal despeckling behavior unchanged (`initiallag` values and `numdespeckled` were unchanged). With this patch, passes2+> used the full search range, long delays (~15 s in lesion/infarct territory) were recovered, and high-lag failures occured only at the true 40 s boundary rather than at an unintended ~8 s ceiling.

### Jump to Pre/Post-Fix Outputs here
#### [Initial Test Command Pre/Post-Fix](https://github.com/suchitag07/Hemodynamic-delays-using-Rapidtide-BHP/blob/main/Debugging_Log.md#example-1)

***

## Step-by-Step Tracing

- In the process of tracing the bug, I inserted a few print statements to track what was happening to lagmin and lagmax during the (i) initial similarity function fit, and (ii) despeckling/refitting routine

### Quick links
  - **[Initial similarity function fit check](https://github.com/suchitag07/Hemodynamic-delays-using-Rapidtide-BHP/blob/main/Debugging_Log.md#i-initial-similarity-function-fit-check)**
  - **[Despeckling routine check](https://github.com/suchitag07/Hemodynamic-delays-using-Rapidtide-BHP/blob/main/Debugging_Log.md#ii-despeckling-routine-check)**
  - **[Print statement outputs of raw/native rapidtide code](https://github.com/suchitag07/Hemodynamic-delays-using-Rapidtide-BHP/blob/main/Debugging_Log.md#print-statement-outputs-of-originalnative-rapidtide-code)**
  - **[Print statement outputs post minor patch](https://github.com/suchitag07/Hemodynamic-delays-using-Rapidtide-BHP/blob/main/Debugging_Log.md#print-statement-outputs-post-minor-patch)**
  - **[Tidepool output example](https://github.com/suchitag07/Hemodynamic-delays-using-Rapidtide-BHP/blob/main/Debugging_Log.md#tidepool-output-example)**

*** 

### (i) Initial similarity function fit check

- To track per voxel lagmin/lagmax and assigned fail reasons (focused on cases with highlag fails - codes 8224 & 8192) 
- Note: This is before despeckling starts. 

#### simFuncClasses.py line 1616: 
```
print("BEFORE REFINEMENT (INITLAGHIGH):","lagmin=", self.lagmin,"lagmax=", self.lagmax,"maxlag_init=", maxlag_init)
```
![](https://github.com/user-attachments/assets/0c9107c9-8629-45ff-993c-e1eaa8916d8c)

#### simFuncClasses.py line 1902: 
```
print("BAD LAG AFTER REFINEMENT (FITLAGHIGH):","lagmin=", self.lagmin,"lagmax=", self.lagmax,"(refined)maxlag=", maxlag)
```
![](https://github.com/user-attachments/assets/10b68da0-8b95-401e-b91e-a479917a0c5f)

#### simFuncClasses.py line 1925: 
```
print("FIT FAIL SUMMARY:","maxlag_init=", maxlag_init, "maxlag=", maxlag, "maxval_init=", maxval_init, "maxval=", maxval, "maxsigma_init=", maxsigma_init, "maxsigma=", maxsigma, "maskval=", maskval, "failreason=", failreason, "fitfail=", fitfail)
```
![](https://github.com/user-attachments/assets/97b72de0-7b02-4a27-b166-b41781bd74d5)

### (ii) Despeckling routine check
- To track changes in lagmin/lagmax between despeckling subpasses, I inserted the following print statements

#### fitSimFuncMap.py line 863: 
```
print(f"START despeckle pass {thepass}, subpass {despecklepass + 1}:","theFitter.lagmin=", theFitter.lagmin,"theFitter.lagmax=", theFitter.lagmax,"theFitter.lagmod=", theFitter.lagmod,"optiondict_lagmin=", optiondict["lagmin"],"optiondict_lagmax=", optiondict["lagmax"])
```
![](https://github.com/user-attachments/assets/0e842ea6-5bd0-467e-a053-355b8c9c2d8d)

#### fitSimFuncMap.py line 936: 
```
print(f"BEFORE fitcorr despeckle pass {thepass}, subpass {despecklepass + 1}:","theFitter.lagmin=", theFitter.lagmin,"theFitter.lagmax=", theFitter.lagmax,"theFitter.lagmod=", theFitter.lagmod,"optiondict_lagmin=", optiondict["lagmin"],"optiondict_lagmax=", optiondict["lagmax"],"numdespeckled=", numdespeckled)
``` 
![](https://github.com/user-attachments/assets/2b453122-65ba-49da-94ea-c97219c5700b)

#### simfuncfit.py line 383 (fitcorr): 
```
print("FITCORR ENTRY:","thefitter.lagmin=", thefitter.lagmin,"thefitter.lagmax=", thefitter.lagmax)
```
![](https://github.com/user-attachments/assets/de9bcc7d-7663-41d9-bcb5-2c185862203b)

- During despeckling (`initiallag is not None`), `onesimfuncfit` calls on the method `setrange` (line 122/123), which accepts the input values as: `thefitter.setrange(initiallag - despeckle_thresh / 2.0, initiallag + despeckle_thresh / 2.0)`. `setrange` takes the first argument and assigns it to `lagmin`, and the second to `lagmax`. 

- So to check what lagmin/lagmax values were per despeckled voxel, I inserted the following:

#### simuncfit.py line 125-127 (onesimfuncfit): 
```
new_lagmin = initiallag - despeckle_thresh / 2.0
new_lagmax = initiallag + despeckle_thresh / 2.0
print("onesimfuncfit :","initiallag=", initiallag,"despeckle_thresh=", despeckle_thresh,"setrange.lagmin=", new_lagmin,"setrange.lagmax=", new_lagmax)
```
![](https://github.com/user-attachments/assets/3be941e6-2c22-4310-bc9e-89e659ce04fb)

#### simuncfit.py line 579 (fitcorr): 
```
print("FITCORR EXIT:","thefitter.lagmin=", thefitter.lagmin,"thefitter.lagmax=", thefitter.lagmax)
```
![](https://github.com/user-attachments/assets/3f9127ec-9c00-4302-a8fd-f5f5bc63154b)

#### fitSimFuncMap.py line 975: 
```
print(f"AFTER fitcorr despeckle pass {thepass}, subpass {despecklepass + 1}:","theFitter.lagmin=", theFitter.lagmin,"theFitter.lagmax=",theFitter.lagmax,"theFitter.lagmod=", theFitter.lagmod,"optiondict_lagmin=", optiondict["lagmin"],"optiondict_lagmax=", optiondict["lagmax"],"numdespeckled=", numdespeckled) 
```
![](https://github.com/user-attachments/assets/9f1b6d6a-b12c-4fae-a01c-e4348793958b)

***

### Print statement outputs of original/native rapidtide code 

#### Despeckling pass 1, subpass 1

```
# Subpass 1 - Original Version output
START despeckle pass 1, subpass 1: theFitter.lagmin= -5.0 theFitter.lagmax= 40.0 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0
BEFORE fitcorr despeckle pass 1, subpass 1: theFitter.lagmin= -5.0 theFitter.lagmax= 40.0 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0 numdespeckled= 1390
FITCORR ENTRY: thefitter.lagmin= -5.0 thefitter.lagmax= 40.0
(Last voxel despeckled in subpass 1) onesimfuncfit : initiallag= -2.267278988361595 despeckle_thresh= 17.2578757010614 setrange.lagmin= -10.896216838892295 setrange.lagmax= 6.361658862169106
FITCORR EXIT: thefitter.lagmin= -10.896216838892295 thefitter.lagmax= 6.361658862169106
AFTER fitcorr despeckle pass 1, subpass 1: theFitter.lagmin= -10.896216838892295 theFitter.lagmax= 6.361658862169106 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0 numdespeckled= 1390
```

- **Immediately you can see** that the global `theFitter.lagmin/lagmax` starting at -5/40, has been truncated down to the lagmin/max values used when despeckling the last voxel in this pass (`FITCORR EXIT: thefitter.lagmin= -10.896216838892295 thefitter.lagmax= 6.361658862169106`). 

#### Despeckling pass 1, subpass 2

```
# Subpass 2 - Original Version output
START despeckle pass 1, subpass 2: theFitter.lagmin= -10.896216838892295 theFitter.lagmax= 6.361658862169106 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0
BEFORE fitcorr despeckle pass 1, subpass 2: theFitter.lagmin= -10.896216838892295 theFitter.lagmax= 6.361658862169106 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0 numdespeckled= 380
FITCORR ENTRY: thefitter.lagmin= -10.896216838892295 thefitter.lagmax= 6.361658862169106
(Last voxel despeckled in subpass 2) onesimfuncfit : initiallag= -2.3876965115932287 despeckle_thresh= 17.2578757010614 setrange.lagmin= -11.016634362123929 setrange.lagmax= 6.241241338937472
FITCORR EXIT: thefitter.lagmin= -11.016634362123929 thefitter.lagmax= 6.241241338937472
AFTER fitcorr despeckle pass 1, subpass 2: theFitter.lagmin= -11.016634362123929 theFitter.lagmax= 6.241241338937472 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0 numdespeckled= 380
```
#### Despeckling pass 1, subpass 3

```
# Subpass 3 - Original Version output
START despeckle pass 1, subpass 3: theFitter.lagmin= -11.016634362123929 theFitter.lagmax= 6.241241338937472 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0
BEFORE fitcorr despeckle pass 1, subpass 3: theFitter.lagmin= -11.016634362123929 theFitter.lagmax= 6.241241338937472 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0 numdespeckled= 155
FITCORR ENTRY: thefitter.lagmin= -11.016634362123929 thefitter.lagmax= 6.241241338937472
(Last voxel despeckled in subpass 3) onesimfuncfit : initiallag= -0.08438081954338436 despeckle_thresh= 17.2578757010614 setrange.lagmin= -8.713318670074084 setrange.lagmax= 8.544557030987317
FITCORR EXIT: thefitter.lagmin= -8.713318670074084 thefitter.lagmax= 8.544557030987317
AFTER fitcorr despeckle pass 1, subpass 3: theFitter.lagmin= -8.713318670074084 theFitter.lagmax= 8.544557030987317 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0 numdespeckled= 155
```
#### Despeckling pass 1, subpass 4

```
# Subpass 4 - Original Version output
START despeckle pass 1, subpass 4: theFitter.lagmin= -8.713318670074084 theFitter.lagmax= 8.544557030987317 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0
BEFORE fitcorr despeckle pass 1, subpass 4: theFitter.lagmin= -8.713318670074084 theFitter.lagmax= 8.544557030987317 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0 numdespeckled= 105
FITCORR ENTRY: thefitter.lagmin= -8.713318670074084 thefitter.lagmax= 8.544557030987317
(Last voxel despeckled in subpass 4) onesimfuncfit : initiallag= -0.4848587174544552 despeckle_thresh= 17.2578757010614 setrange.lagmin= -9.113796567985156 setrange.lagmax= 8.144079133076245
FITCORR EXIT: thefitter.lagmin= -9.113796567985156 thefitter.lagmax= 8.144079133076245
AFTER fitcorr despeckle pass 1, subpass 4: theFitter.lagmin= -9.113796567985156 theFitter.lagmax= 8.144079133076245 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0 numdespeckled= 105
```

#### Start of Main Pass 2

```
Pass number 2
checking reference regressor autocorrelation properties
check_autocorrelation: 87 87 10 87
searching for sidelobes with amplitude > 0.1 with abs(lag) < 40.0 s
WARNING: check_autocorrelation found bad sidelobe at 14.303746082002721 seconds (0.06991175558256178 Hz)...
Significance estimation, pass 2
BEFORE REFINEMENT (INITLAGHIGH): lagmin= -9.113796567985156 lagmax= 8.144079133076245 maxlag_init= 11.49999338388443
BAD LAG AFTER REFINEMENT (FITLAGHIGH): lagmin= -9.113796567985156 lagmax= 8.144079133076245 (refined)maxlag= 11.512116957117014
FIT FAIL SUMMARY: maxlag_init (self.lagmax + rangeextension + binwidth)= 8.604079141420895 maxlag= 8.144079133076245 maxval_init= 0.30147209128851254 maxval= 0.30281860166818214 maxsigma_init= 1.7580961584890198 maxsigma= 1.9631519406637865 maskval= 0 failreason= 8224 fitfail= True
```

- **Start of new Pass number 2**:  
- You can see the global fitter lagmin and lagmax (lagmin= -9.113796567985156, lagmax= 8.144079133076245) have held on to the values set in pass1 subpass 4! As a result, nothing exceeds the despeckle_thresh (17s), and no voxels are patched. Voxels with lags > 8s are automatically failed. 

***

### Attempted minor patch

- To reset lagmin/max after each despeckling subpass, I added these lines:

#### fitSimFuncMap.py line 970-973: 
```
global_lagmin = optiondict["lagmin"]
global_lagmax = optiondict["lagmax"]
theFitter.setrange(global_lagmin, global_lagmax)
```
### Print statement outputs post minor patch

#### Despeckling for pass 1, subpass 1

```
# Subpass 1 - Post-minor-patch output 
START despeckle pass 1, subpass 1: theFitter.lagmin= -5.0 theFitter.lagmax= 40.0 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0
BEFORE fitcorr despeckle pass 1, subpass 1: theFitter.lagmin= -5.0 theFitter.lagmax= 40.0 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0 numdespeckled= 1390
FITCORR ENTRY: thefitter.lagmin= -5.0 thefitter.lagmax= 40.0
(Last voxel despeckled in subpass 1) onesimfuncfit : initiallag= -2.267278988361595 despeckle_thresh= 17.2578757010614 setrange.lagmin= -10.896216838892295 setrange.lagmax= 6.361658862169106
FITCORR EXIT: thefitter.lagmin= -10.896216838892295 thefitter.lagmax= 6.361658862169106
AFTER fitcorr despeckle pass 1, subpass 1: theFitter.lagmin= -5.0 theFitter.lagmax= 40.0 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0 numdespeckled= 1390
```
- **Post soft patch** you can see that the global fitter `theFitter.lagmin= -5.0 theFitter.lagmax= 40.0` is restored to the original serach range, while per-voxel despeckling used its conservative range. The number of voxels despeckled in this pass has not been affected, nor has the `initiallag` estimate. The soft patch appears to have worked!

#### Despeckling for pass 1, subpass 2

```
# Subpass 2 - Post-minor-patch output 
START despeckle pass 1, subpass 2: theFitter.lagmin= -5.0 theFitter.lagmax= 40.0 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0
BEFORE fitcorr despeckle pass 1, subpass 2: theFitter.lagmin= -5.0 theFitter.lagmax= 40.0 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0 numdespeckled= 380
FITCORR ENTRY: thefitter.lagmin= -5.0 thefitter.lagmax= 40.0
(Last voxel despeckled in subpass 2) onesimfuncfit : initiallag= -2.3876965115932287 despeckle_thresh= 17.2578757010614 setrange.lagmin= -11.016634362123929 setrange.lagmax= 6.241241338937472
FITCORR EXIT: thefitter.lagmin= -11.016634362123929 thefitter.lagmax= 6.241241338937472
AFTER fitcorr despeckle pass 1, subpass 2: theFitter.lagmin= -5.0 theFitter.lagmax= 40.0 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0 numdespeckled= 380
```

#### Despeckling for pass 1, subpass 3

```
# Subpass 3 - Post-minor-patch output 
START despeckle pass 1, subpass 3: theFitter.lagmin= -5.0 theFitter.lagmax= 40.0 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0
BEFORE fitcorr despeckle pass 1, subpass 3: theFitter.lagmin= -5.0 theFitter.lagmax= 40.0 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0 numdespeckled= 155
FITCORR ENTRY: thefitter.lagmin= -5.0 thefitter.lagmax= 40.0
(Last voxel despeckled in subpass 3) onesimfuncfit : initiallag= -0.08438081954338436 despeckle_thresh= 17.2578757010614 setrange.lagmin= -8.713318670074084 setrange.lagmax= 8.544557030987317
FITCORR EXIT: thefitter.lagmin= -8.713318670074084 thefitter.lagmax= 8.544557030987317
AFTER fitcorr despeckle pass 1, subpass 3: theFitter.lagmin= -5.0 theFitter.lagmax= 40.0 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0 numdespeckled= 155
```

#### Despeckling for pass 1, subpass 4

```
# Subpass 4 - Post-minor-patch output 
START despeckle pass 1, subpass 4: theFitter.lagmin= -5.0 theFitter.lagmax= 40.0 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0
BEFORE fitcorr despeckle pass 1, subpass 4: theFitter.lagmin= -5.0 theFitter.lagmax= 40.0 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0 numdespeckled= 105
FITCORR ENTRY: thefitter.lagmin= -5.0 thefitter.lagmax= 40.0
(Last voxel despeckled in subpass 4) onesimfuncfit : initiallag= -0.4848587174544552 despeckle_thresh= 17.2578757010614 setrange.lagmin= -9.113796567985156 setrange.lagmax= 8.144079133076245
FITCORR EXIT: thefitter.lagmin= -9.113796567985156 thefitter.lagmax= 8.144079133076245
AFTER fitcorr despeckle pass 1, subpass 4: theFitter.lagmin= -5.0 theFitter.lagmax= 40.0 theFitter.lagmod= 1000.0 optiondict_lagmin= -5.0 optiondict_lagmax= 40.0 numdespeckled= 105
```

#### Start of Main Pass 2

```
Pass number 2
checking reference regressor autocorrelation properties
check_autocorrelation: 87 87 10 87
searching for sidelobes with amplitude > 0.1 with abs(lag) < 40.0 s
WARNING: check_autocorrelation found bad sidelobe at 14.303746082002721 seconds (0.06991175558256178 Hz)...
Significance estimation, pass 2
BAD LAG AFTER REFINEMENT (FITLAGHIGH): lagmin= -5.0 lagmax= 40.0 (refined)maxlag= 40.001004202933686
FIT FAIL SUMMARY: maxlag_init (self.lagmax + rangeextension + binwidth)= 39.559993892908096 maxlag= 40.0 maxval_init= 0.1768387720270732 maxval= 0.18741887538366417 maxsigma_init= 0.5860320528296732 maxsigma= 1.2936507456436694 maskval= 0 failreason= 8192 fitfail= True
```
- **Start of new Pass number 2** - You can see that the post each subpass, the global `theFitter.lagmin/lagmax` are reset to optiondict["lag/min/max"]. In the new outer pass, these values stick, and outliers on the edge of this search range are caught and despeckled (see sub-VASC_desc-postsoftpatchrun-ADDEDprintstamentout.txt for full log). 

***

## Tidepool output example

### Initial test command

- ***Important sidenote***: Some of the rapidtide parameters I mention in this log may not be considered THE most optimal. That said, the goal here isn’t to tune parameters, but rather to investigate whether any unexpected behavior arises under the hood given arbitrary input values. 
- I processed a few subjects with this base command and compared their pre/post-soft patch outputs. 
- Re the flag options here: I went with a search range of `-5 to 40` primarily becuase, for this dataset, the corr function seems to vary slowly over time. The peaks are broad/drawn out due to autocorrelation in our reference regressor (I locked that down to the cerebellum, it seemed to do better than the SSS (superior sag sinus) and the cortical gray matter reference).
- That said, it's possible that I might go back and play around with the search range/smoothing level etc
- For other flags, see discussion [here](https://neurostars.org/t/question-re-searchrange-flag-in-rapidtide/35307/2).

```
/Users/suchitaganesan/anaconda3/envs/rapidtide_env/bin/rapidtide \
    /path_to_data/fmriprep/sub-VASC/ses-01/func/sub-VASC_ses-01_task-rest_desc-preproc_bold.nii.gz \
    /path_to_data/SG_rapidtide_patched/v310_sharpenreg_NOpatchdetect_test_subjects_spatialfilt4_LOCALRUN/sub-VASC/sub-VASC \
    --numnull 10000 \
    --spatialfilt 4 \
    --filterband lfo \
    --preppass \
    --sharpenregressor \
    --searchrange -5 40 \
    --passes 3 \
    --nofitfilt \
    --simcalcrange 130 -1 \
    --corrmask /path_to_data/fmriprep/sub-VASC/ses-01/func/sub-VASC_ses-01_task-rest_desc-brain_mask.nii.gz \
    --globalmeaninclude /path_to_data/fmriprep/sub-VASC/ses-01/fMRIPrep_parc_bold_space/sub-VASC_ses-01_task-rest_space-bold_desc-aparcaseg.nii.gz:8,47 \
    --refineinclude /path_to_data/fmriprep/sub-VASC/ses-01/fMRIPrep_parc_bold_space/sub-VASC_ses-01_task-rest_space-bold_desc-aparcaseg.nii.gz:8,47 \
    --whitemattermask /path_to_data/fmriprep/sub-VASC/ses-01/fMRIPrep_parc_bold_space/sub-VASC_ses-01_task-rest_space-bold_desc-WM_probseg_bin.nii.gz \
    --csfmask /path_to_data/fmriprep/sub-VASC/ses-01/fMRIPrep_parc_bold_space/sub-VASC_ses-01_task-rest_space-bold_desc-CSF_probseg_bin.nii.gz \
    --motionfile /path_to_data/fmriprep/sub-VASC/ses-01/func/sub-VASC_ses-01_task-rest_desc-confounds-trimmed_timeseries_revised_final.tsv \
    --motpowers 2 \
    --mklthreads 1 \
    --nprocs 1 \
    --numskip 0 \
    --outputlevel max

```

### Example 1

- sub-VASC_01: This is the example case we've discussed thus far.
- Per their T1, we can see they have some lesion-like pathology in their frontal lobe.

![](https://github.com/user-attachments/assets/4198f121-f8a5-40e3-9569-8f389d1c9f3f)

### Native rapidtide code output

- sub-VASC_01: With the valid mask turned on, the correlation function graph just says "No valid fit" (`init lag high, fit lag high`) even though there are two clear peaks around 12s and 25s. Per the logs, the search window was truncated down to `theFitter.lagmin= -9.113796567985156, theFitter.lagmax= 8.144079133076245`, naturally the histogram of lagtimes on the GUI reflects the same limits. Lags outside this range were automatically failed.

![](https://github.com/user-attachments/assets/7dcd2de3-790b-41f9-a928-0de220e29af5)

### Post soft patch output 

- sub-VASC_01: Post editing the code to restore `theFitter` lag limits (per user defined range of `-5:40s`, the lag value of 26s is now picked up and considered valid.

![](https://github.com/user-attachments/assets/cbb02817-197d-4425-a80d-d7f26a68acff)

### Example 2

- sub-VASC_02: Per their T1, we can see they have some widespread WMH-like pathology.

![](https://github.com/user-attachments/assets/98ab25b6-0c04-46a0-8666-38da2c6b1233)

### Native rapidtide code output

- sub-VASC_02: By the end of pass 3, the lagmin/max was truncated down from -5:40 to 1:6 `FITCORR EXIT: theFitter.lagmin= 1.0288625190571636 theFitter.lagmax= 6.028862519057164`. The distribution of lagtimes (valid fits) appears crammed into this range. Naturally this has resulted in a massive number of fit failures across the board (`total initfails: 22779, total fitfails: 26574` per the logs).

![](https://github.com/user-attachments/assets/b8854631-796f-42ef-a5b6-9be61e3cd2b9)

### Post soft patch output 

- Post soft-patch it's using the full search window.

![](https://github.com/user-attachments/assets/ce1d350b-9736-41d4-bf6f-acf0a5fd3d5a)

*** 

### Replication of Pre/Post-bug-fix Output with Revised Command

- I went back and performed a more systematic pre/post-patch comparison as a sanity check. This time, I restricted the search range to `-5s to 30s`. I kept smoothing at the default level (voxel-dim/2 --> 3/2 ~ 1.5mm sigma).

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

![](https://github.com/user-attachments/assets/82e2ab41-7f2e-4ed5-9e76-92615aa5e922)
![](https://github.com/user-attachments/assets/e7c8dfd7-47b9-4874-9eaa-c1dc1270459b)

### Additional Examples

![](https://github.com/user-attachments/assets/2f03adce-0f3f-46d7-94df-ed149a800e07)

***

### Example Group Level Maps - v3.1.11

- We compared the impact of different smoothing factors (sigma 1.5/2.5/3 etc) and found that at 2.5mm ~ 5.9FWHM
	- The distribution of delay times appeared normal and centered around 0 (fewer erroneous long delays)
   	- And the distribution of correlation values appeared negatively skewed (concentrated around R=0.6-1)
   	- To check this I calculated the Fisher-Pearson coefficient (measure of skewness) for each subject's max-correlation distribution. Roughly, what I noticed is that - negative skewness (a long left tail with most max-corr values shifted to the right) - tended to correspond to spatially coherent maps (longer delays near WM/CSF, shorter delays around the GM).

![](https://github.com/user-attachments/assets/1cc99c39-df26-4b12-8673-c10f7273c9d8)

![](https://github.com/user-attachments/assets/b9097bb1-1abd-4639-a50f-86eeadad4b5a)

