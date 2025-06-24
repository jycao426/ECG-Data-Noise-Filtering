# ECG-Data-Noise-Filtering

1. Load Data into a Folder in Google Drive
2. Create a Folder titled 'Filtered Data' within the folder
3. Set the path in the code to the path of the folder
   generate_ecg_summary(
    data_folder_path='/content/drive/MyDrive/ECG Report/Mouse_1/', # FOLDER PATH
    output_csv_path='/content/drive/MyDrive/ECG Report/Mouse_1/ECG_Metrics_Summary.csv', # SET TO FOLDER + /ECG_Metrics_Summary.csv
    output_folder_path='/content/drive/MyDrive/ECG Report/Mouse_1/Filtered Data' # FOLDER PATH + /Filtered Data
)
4. Then Run the Data and find filtered files in the 'Filtered Data' Folder and the 'ECG_Metrics_Summary.csv'

## Overall Workflow of the Code

1. **Load and Parse ECG Files**
    - Open each `.txt` file.
    - Extract time and voltage columns.
    - Skip header lines and format errors.
2. **Pre-Processing / Outlier Removal**
    - **Step 1:** Remove voltage values > ±5 mV.
    - **Step 1.5:** Remove voltage values in the top and bottom 1% (1st and 99th percentiles).
    - **Step 2:** Remove sharp spikes (where voltage jumps > `jump_threshold`, default = 0.4 mV).
        - calculate the **difference between consecutive voltage values** (i.e., first derivative), then choose a high percentile (e.g. 95th or 99th) as the `jump_threshold`
3. **Abnormal Segment Detection**
    - Scan using a moving window (`window_size=200`).
    - Flag segments with low or high standard deviation (`std_thresh_low=0.02`, `std_thresh_high=0.8`).
        - If the window has:
            - **Too little variation** (STD < 0.02) → may be disconnected lead or dropout
            - **Too much variation** (STD > 0.8) → likely a noisy artifact
    - Merge nearby flagged segments.
        - Instead of treating each as a separate event, the code **groups timepoints that are close together** (less than `merge_distance = 1.0 sec` apart).
    - Remove ±0.5 seconds around each flagged abnormal timepoint.
        - For each detected abnormal segment center, the code removes a **1-second window centered on it** (±0.5 seconds).
4. **Z-score Based Filtering**
    - Apply `zscore_filter()` to remove voltage outliers based on statistical deviation from the mean (default threshold = 2.0σ).
        - **Compute the z-scores** of all voltage values using `scipy.stats.zscore`.
        - **Create a mask**: keep only the values where the absolute z-score is less than the threshold (default = 2.0).
        - **Filter out** the rest — the outliers are excluded entirely from the time and voltage arrays.
5. **Concatenation**
    - After filtering, remove all 0-voltage placeholders and shift the signal to start from `t=0`.
6. **Additional Amplitude-Based Filtering**
    - **Mean-based filter:**
        
        Remove points that are slightly above average low-amplitude noise.
        
        - `perc = 99`: Only amplitudes in the bottom 99% are considered for computing the "quiet" baseline.
        - `thresh = 0.00001`: Sets the upper bound for keeping values to **(mean of quiet region) + (0.00001 × mean)**.
    - **Median-based filter:**
        
        Remove points beyond a fixed ratio of the median amplitude.
        
        - `thresh_ratio = 1.2`: Retains only points with amplitude ≤ **(median × 2.2)**.
7. **Heart Rate Variability (HRV) & Peak Detection**
    - Use `scipy.signal.find_peaks()` to detect R-peaks.
    
    ### **1. Input**
    
    - **`times`**: Array of time points (in seconds)
    - **`voltages`**: Corresponding ECG voltage readings (in mV)
    - **`fs`**: Sampling frequency (defaults to `2000 Hz` for mouse data)
    
    ---
    
    ### **2. Preprocessing: Sampling Validation**
    
    - Compute `dt = median(diff(times))`
    - Ensure `dt ≈ 1/fs`
        
        ⮕ *Warn if inconsistent sampling rate*
        
    
    ---
    
    ### **3. Mouse-Specific Constraints**
    
    - Set `rr_min_sec = 0.04` (i.e., 50 ms = ~1200 BPM max)
    - Calculate `min_distance_samples = rr_min_sec * fs`
        
        ⮕ *Prevents peaks that are too close together from being counted*
        
    
    ---
    
    ### **4. Adaptive Peak Detection**
    
    - Compute `prominence_mV = get_adaptive_prominence(voltages, lower=90, scale=0.3)`
        
        ⮕ *Dynamically adjusts peak detection to noise level*
        
        Computes a dynamic prominence threshold for peak detection based on signal amplitude distribution.
        Returns the value below which 92% of the absolute voltages fall, scaled by 0.45.
        This helps capture true R-peaks while ignoring minor fluctuations or noise.
        
    - Use `find_peaks()` with:
        - `distance = min_distance_samples`
        - `prominence = prominence_mV`
    
    ---
    
    ### **5. Validity Check**
    
    - **If no peaks found** → return all `None`
    - **Else**, proceed to compute intervals
    
    ---
    
    ### **6. RR Interval Extraction**
    
    - Get `r_peak_times = times[peaks]`
    - Compute `rr_intervals = diff(r_peak_times)`
    - If fewer than 2 RR intervals → return all `None`
    
    ---
    
    ### **7. HRV Metrics Calculation**
    
    - `SDNN = std(rr_intervals)` ⮕ *Heart rate variability*
    - `mean_rr = mean(rr_intervals)` ⮕ *Average RR interval (s)*
    - `hr_bpm = 60 / mean_rr` ⮕ *Heart rate in BPM*
    
    ---
    
    ### 
    
    - Calculate:
        - **SDNN** (standard deviation of RR intervals)
        - **Mean RR interval**
        - **Heart Rate (BPM)**
8. **Plotting**
    - Plot ECG data at different stages:
        - Raw ECG
        - Post-auto-filtering
        - Post-concatenation
        - After each type of amplitude filtering
9. **Output File Generation**
    - Saves the filtered signal (aligned with the original timestamps) into a `_filtered.txt` file.
    - Only includes clean, nonzero voltages aligned to the original time vector.

**10 Summary Metrics CSV**

- Collects HR, RR, SDNN, and % removed per file.
- Saves this data to `ECG_Metrics_Summary.csv` for easy review.
1. **Plot Correlation Plot for HRV(SDNN) and RR**
    1. Assess the HRV Correlation Plot
    - **Ideal Outcome**:
        - The dot is **close to** the red line.
        - Implies **filtering didn’t overly change** the RR variability structure.
    - **Slight deviation below the line**:
        - Expected if filtering removed noisy peaks or false detections.
        - Suggests HRV was **refined**, not distorted.
    - **Large deviation**:
        - Could indicate aggressive filtering that distorted beat spacing.
    
    b. Assess the RR Interval plot
    
    - **Tight clustering along the red line (`y = x`)** → Good correlation.
    - **Wider scatter** → Greater difference between original and filtered intervals.
    
    c. Print out the pearson correlation plot for HRV and HR

- LOOK AT IMAGES FOR EXAMPLE OUTPUTS OF THE CODE
