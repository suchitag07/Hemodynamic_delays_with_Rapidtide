## Post 3.1.11 Issue

### Issue: Lags on the edge of the search range
  - Some users have found that they see a small % of voxels with lag values that are at the edge of the search range (eg if your search range is `-5 to 15s`, then some voxels have values at the edge like `-5.9 or 16.01s`).
  - I see this too sometimes, but I know that Dr. Frederick has said in the past that if the correlation function is peaking at the edge of the search range, it can latch on to values there. If it does this for a large % of voxels, it means you have to adjust/expand your search range. 

### Logic - What's happening
- Now, I just had an idea of where this might be happening (the logic behind why a lag value at the edge of the search range might still win/pass):
- My understanding is that rapidtide performs an initial correlation fit, screens lag values against the user-defined/default thresholds - for the lag value, the peak amplitude, and the peak width, etc.
- Voxels that have fits outside the defined thresholds are failed and assigned a failure mode/code (later used to create the `valid mask`).
- Outlier voxels that are eligible will be re-fitted (undergo despeckling).

### Example Scenario
- Now, assuming a global user-defined `--search range −5 to 15 s` and a `--despeckle_thresh of 5 s`. During despeckling, `onesimfuncfit` sets a temporary/conservative range of `[initiallag-(despeckle_thresh/2), initiallag+(despeckle_thresh/2)]` (see main debugging log).
- So that would be, in this example case, `[initiallag-2.5, initiallag+2.5]`.
- Note: It does this so that, during re-fitting, we don't select/re-fit lags near sidelobe peaks (peaks in the corr func that arise through autocorrelation).

So:

- If initiallag is −4 s (within bounds), the temporary search range is −6.5 to −1.5 s. A refit could therefore produce a lag below the global lower bound of −5 s.
- If initiallag is 14 s (within bounds), the temporary search range is 11.5 to 16.5 s. A refit could therefore produce a lag above the global upper bound of 15 s.

### Back to the logic
- Typically, after these cases are despeckled and re-fit, they undergo another round of correlation fitting/screening against thresholds, and if they fall outside the range/thresholds again -> goodbye, assigned a failure mode, etc.
- Outlier voxels that are eligible will be re-fitted.
- This goes on in a loop. Pass-1 -> despeckling subpasses -> Pass-2 -> despeckling subpasses, etc.

### 3.1.11 patch
- Now, the patch in 3.1.11 fixed a state-leak of the search range across passes.
- Basically in 3.1.10 and before, the modified window (through despeckling) was leaking across passes, so a starting range of `[-5,40]` would slam down to, e.g., `[-4,+1]` by the end of pass 1 despeckling subpass 4, and carry on to main pass 2, etc., artificially restricting the range of allowed delays).

**However, after patching the state-leak, there might be one thing I missed**
- After the rapidtide's **FINAL pass + despeckle round**, we don't screen/check those re-fitted voxels again against the global thresholds, because we've reached the end of the loop.
- So it's possible a tiny handful of lags that have veered off the edges of the search range remain in the final map...
