# HDD week-1 checks

> English translation (2026-10-27) of the Chinese original; numbers, plans and conclusions are unchanged. The Chinese text is in the earlier project repository at [`b209e2e`](https://github.com/Euswbnix/CSC413_Project/blob/b209e2e/docs/hdd_week1_checks_2026-09-21.md). On 2026-10-15 internal code names in it were replaced with plain terms; the earlier wording is at commit [`766d141`](https://github.com/Euswbnix/CSC413_Project/tree/766d141) and before.

Date: 2026-09-21. Covers the "checks to do in week 1" of [`dataset_survey_2026-09-21.md`](https://github.com/Euswbnix/CSC413_Project/blob/b209e2e/docs/dataset_survey_2026-09-21.md) and the first round of checks (data) of
[`decision_2026-09-21_keep_driving.md`](https://github.com/Euswbnix/CSC413_Project/blob/b209e2e/docs/decision_2026-09-21_keep_driving.md).

**No frames were viewed at any point.** The checks use only the CAN, GPS and timestamp CSVs and the video container metadata (frame count, frame rate, resolution). The data stays on the server, readable only by the owner.
This document contains only aggregate statistics: no coordinates, trajectories or per-frame data. Any session IDs mentioned are already public in the TRN.pytorch repository (136 of the 137 are public).

> **Corrections (same day)**: several conclusions and numbers in the first version were wrong and have been corrected:
> - The first version said "a split by trip cannot avoid overlapping roads", but it had only computed the overlap of single sessions with all the others and had not tried holding out groups. Holding out whole sessions grouped by route is in fact feasible;
> - For dropped-frame time the first version counted the full length of each interval containing a gap; the time actually missing is about half of that;
> - The "15 Hz session" is in fact a mixed frame rate;
> - Overlap shares now count only points that are moving, have video and have a valid label, and exclude sessions with faulty GPS;
> - Steering labels are no longer interpolated across CAN gaps;
> - **The first version of the sync check on 2026-09-22 was wrong**: it accumulated the per-frame signals into a heading before cross-correlating, and normalised by a fixed length instead of the overlap length; both pull the estimate toward 0. The "lag of −0.004 s" reported then was a fixed point of the estimator, not a measurement, and is withdrawn. The corrected method and results are in section 4;
> - The first version said the "100 Hz steering angle synchronised with the frames" was confirmed. Only timestamp alignment had been confirmed, and it was later found that in some sessions the ROS timestamps drift against the camera clock, so synchronisation was not yet confirmed;
> - The first version said 2 km blocks give "about 120 test blocks, more than the 36 sessions, so better statistical power". The blocks are very uneven in size and the effective number weighted by data is only about 17; moreover one drive passes through several blocks and neighbouring blocks share roads and weather, so the blocks are not independent and 17 is only an upper bound. That claim is withdrawn;
> - The block split now also removes test points within 50 m of val and drops samples whose 5 s history window is incomplete; the numbers changed accordingly.
>
> The split options are compared in [`discussion_2026-09-21_next_steps.md`](https://github.com/Euswbnix/CSC413_Project/blob/b209e2e/docs/discussion_2026-09-21_next_steps.md).

The scripts are [`hdd/week1_checks.py`](../hdd/week1_checks.py), [`hdd/timestamp_gaps.py`](../hdd/timestamp_gaps.py), [`hdd/geo_split_feasibility.py`](../hdd/geo_split_feasibility.py), [`hdd/drift_exclusion.py`](https://github.com/Euswbnix/CSC413_Project/blob/b209e2e/hdd/drift_exclusion.py), [`hdd/extract_features.py`](../hdd/extract_features.py) and [`hdd/sync_check.py`](../hdd/sync_check.py); each runs in about half a minute on the server.

## Conclusions

**All four requirements of the first round of checks are met: multiple trips, per-frame timestamps and frame–steering synchronisation are confirmed, and a feasible split exists.**
- Confirmed: 137 separate sessions; per-frame timestamps; the 100 Hz steering angle aligned at the timestamp level.
- **Synchronisation confirmed (2026-09-22, corrected version)**: the frames are aligned with the **steering angle**; over the 134 sessions with usable windows the median lag is 0.000 s; 3 sessions have no usable window and are unchecked. The CAN **yaw rate** lags the frames by about 0.1 s and cannot be the reference for timing the labels. Sessions with clock drift show no extra misalignment, so the 2.2 hours that were to be excluded do not need to be (section 4).
- Split: holding out whole sessions grouped by route and splitting by geographic block are both feasible. Which one to use is for the team to decide.

1. **The public split cannot be used as a geographic hold-out.** In the TRN repository's split (100 training sessions, 36 test sessions), counting only moving points, the 36 test sessions have on average 37% of their points within 50 m of a training trajectory (28% pooled); 12 test sessions are above half. 9 training sessions have no usable GPS and were not counted, so this is a lower bound.
2. **Single sessions overlap heavily with the others, but holding out groups by route is feasible.**
   - The median session has 78% of its moving points within 50 m of another session's trajectory; only 3 sessions are below 10%. With a random split by session, 64% of the test points are within 50 m of a training or validation trajectory.
   - A greedy search finds a group of 49 sessions that holds 25% of the moving data, with only 10% of its points within 50 m of the remaining sessions.
3. **A split by 2 km geographic blocks is also feasible, but has few effective units.**
   - Assign blocks at random to train/val/test (63/12/25%), remove val points within 50 m of train and test points within 50 m of train or val, then drop samples whose history window (5 s) is incomplete: the test split still holds 24% of the moving data (at least 19% over 20 random draws).
   - But the test points come from 97 sessions, 92 of which are also in training; the roughly 93 test blocks are very uneven in size, and their effective number weighted by data (Kish) is only about 17.
4. **The raw data already contains real irregular sampling.**
   - By the camera clock about 98,000 frames (0.9%) are missing in total, mostly single frames; there are 97 gaps longer than 1 s, in only 7 sessions (by ROS timestamps, 100 gaps in 8 sessions).
   - By ROS timestamps, 17 sessions are missing more than 1% of their time and 1 is missing more than 10%.
   - 1 further session has a mixed frame rate: about 3/4 of the time at 15 Hz, the rest at 30 Hz.
5. **Curves are even rarer than in SullyChen.** Among moving frames, |steering angle| < 5° makes up 62% (SullyChen training set: 42.6%). But the dataset is large, so there are still about 1.1 million frames at ≥ 15°.
6. **The steering sign is the opposite of nuScenes.** In HDD a positive value means a right turn; in nuScenes it means a left turn.

## What is in the package

The download is 149 GB and contains 7 sub-archives, all fully extracted once, with CRC checks passing.

| Content | Size | Notes |
|---|---|---|
| `release_2019_07_08` | 115 GB | 137 sessions, each with one front-camera video (mp4), 6 CAN CSVs and one EAF annotation file; 132 sessions also have 2 GPS CSVs |
| `release_2019_07_25` | 0.5 GB | per-frame timestamps of each session (`png_timestamp.csv`) |
| `20200710_images` | 34 GB | 1.13 million JPEG frames pre-extracted for the TRN paper (only the directory was listed, not extracted) |
| `20200710_sensors`, `_labels` | 10 MB | 8-dim sensor data and 12-class driving-behaviour labels pre-extracted by TRN |
| `EAF_parsing` | 3 MB | official annotation parsing scripts |

- Video: all 137 open, all 1280×720, nominal 30 fps. **In every video the frame count equals the number of rows in the timestamp file (137/137)**, so frame i corresponds to row i of the timestamp file.
- CAN: steering angle, steering speed, yaw rate, vehicle speed, throttle, brake and turn signal, in all 137 sessions, at about 100 Hz.
- GPS: 132 sessions have RTK positions; the GPS of the 4 sessions from 2017-06-08 is clearly faulty, leaving 128 usable.

## Checks in detail

### 1. Timestamps

| | Frames | CAN steering |
|---|---|---|
| Median interval | 33.3 ms (136 sessions), 65 ms in 1 session (mixed frame rate, see below) | 10.0 ms |
| Jitter (1%–99% quantiles, session median) | 26.9–40.2 ms | — |
| Time going backwards | 0 | 0 |

- Every session starts with 2–6 frames spaced more than 0.1 s apart (1–2 s in the first session) before it settles. **The first few frames of each session have to be discarded.**
- By ROS timestamps there are 92,440 gaps where the frame interval exceeds 1.5 times the median (about 50 ms), on average one every 120 frames.
  - These intervals add up to 1.77 hours, but each of them would have contained one normal frame spacing anyway; after subtracting that, about 0.9 hours, roughly 100,000 frames (0.9%), are really missing. Most gaps lose a single frame.
  - There are 100 gaps longer than 1 s, in only 8 sessions; the longest is 23.6 s.
  - 17 sessions are missing more than 1% of their time, only 1 more than 10% (21%).
- The mixed-frame-rate session (1 recording, 37 minutes): 37% of the intervals are about 33 ms and 63% about 65 ms, alternating, 18.4 fps on average.
- 4 sessions have CAN gaps longer than 1 s, the longest 32 minutes. 7,990 frames fall outside the CAN time range, mostly at the start and end of sessions, and have to be trimmed.
- The videos total 105.1 hours and CAN 107.9 hours. In 6 sessions the video covers less than 90% of the CAN duration, the lowest only 40%.
- **ROS timestamps against the camera clock** (`hdd/timestamp_gaps.py`):
  - Frame timestamps are taken on the receiving side (ROS). The timestamp file has another column with the camera's internal clock, which wraps every 128 s; once unwrapped, its frame intervals are very regular. A further "counter" column increases by 1 on every row except in 4 places across all sessions, and none of the roughly 90,000 camera-clock gaps show up in it: it is a sequence number from the recorder and cannot be used to detect dropped frames.
  - This script always skips the first 6 frames of each session and finds 92,432 ROS gaps. 98% of them are also gaps on the camera clock, so frames really are missing; the other roughly 1,750 only arrived late. By the camera clock about 98,000 frames (0.9%) are missing, in line with the ROS-based estimate above.
  - The rate difference between the two clocks has a median of 30 ppm, a normal crystal error. With that fitted out, the median deviation of the ROS timestamps from the camera clock is 0.7 ms and the 99% quantile about 7 ms (session medians).
  - **But in 13 sessions (9.3 hours) the deviation left after removing the rate difference exceeds 1 s at its largest, up to about 12 s**; in 15 sessions (12.0 hours) it exceeds 0.2 s at its largest. By duration, the deviation is above 1 s for about 16 minutes in total and above 0.2 s for about 2.2 hours.
  - The deviation changes gradually: in these sessions it changes by at most 79 ms between neighbouring frames, with no jumps, and every gap on the camera clock is also visible in the ROS timestamps.
  - Excluding the stretches where the deviation is above 0.2 s, together with their 5 s history windows, would remove 2.3% of all prediction samples (6.2% of validation under the whole-drive split of the discussion document). Of the 15 sessions only 6 have the deviation on prediction samples; in the rest it falls only on frames while stopped or without labels (the earlier repository's [`hdd/drift_exclusion.py`](https://github.com/Euswbnix/CSC413_Project/blob/b209e2e/hdd/drift_exclusion.py)).
  - The two possible causes are a backlog on the receiving side (which would offset frames against steering) or the host clock being adjusted by a time-sync service (which would not). **Checked against the frames: it is the latter.** For these 13 sessions, with a search range of ±15 s and the steering angle as reference, the median frame–steering lag is 0.000 s and the slope of the lag against the clock deviation is 0.14; a receive-side backlog would give a slope close to 1 (section 4).

### 2. Units and signs

- **Vehicle speed is in km/h.** Against the speed derived from the GPS track, the ratio is 3.610 ± 0.015 (126 sessions, range 3.59–3.75; GPS with a ratio outside 3.4–3.8 is treated as faulty and those sessions are converted with 3.6).
- **The steering angle is the steering-wheel angle (°).** It is essentially proportional to curvature (yaw rate ÷ speed): median coefficient 2490 °·m (quartiles 2471–2508), median correlation 0.996. That corresponds to a steering ratio × wheelbase of about 43.5 m, typical of a car.
- **A positive value means a right turn.** Two cross-checks agree:
  - steering angle and yaw rate have the same sign: median 100%, worst session 99.7%;
  - yaw rate and the change in GPS heading (clockwise positive) have the same sign: median 99.9%.
- **The header of `rtk_pos.csv` is swapped.** It says `lng,lat`, but the values are latitude first, then longitude (in all 132 sessions).
- **Do not use the heading column of `rtk_track.csv`.** Its sign agrees with the yaw rate about 50% of the time, i.e. at chance. Heading is derived from the GPS positions instead.
- **The GPS of the 4 sessions from 2017-06-08 is unusable.** The speed derived from it does not match the CAN speed, and its heading changes are unrelated to the yaw rate. These 4 sessions, together with the 5 sessions that have no GPS at all, cannot take part in a geographic split.

### 3. Steering-angle distribution

Only frames with speed above 3 m/s and a valid label are counted: 6.68 million frames, 60% of the frames with a valid label. The bins are the macro MAE bins of `metrics.py`.

| \|steering angle\| | HDD (moving) | SullyChen training set |
|---|---|---|
| < 5° | 62.1% | 42.6% |
| 5–15° | 21.3% | 30.0% |
| ≥ 15° | 16.6% | 27.4% |

- The share at ≥ 15° varies a lot between sessions: 5% quantile 3.6%, median 18.8%, 95% quantile 33.6%.
- So the macro-averaged metric and the comparison against a constant are still needed; the class imbalance seen on SullyChen does not go away on HDD.

### 4. Frame–label synchronisation

- Distance from each frame to the nearest CAN steering sample: median 2.6 ms, 99% quantile 6 ms (session medians).
- The 100 Hz steering angle is interpolated to frame times, but only where the two neighbouring CAN samples are at most 50 ms apart, never across a CAN gap. Under this rule about 74,000 frames (0.7%) have no valid label. All steering statistics in this document follow this rule.
- **The frames and the steering angle are indeed aligned (2026-09-22, corrected version).**
  - **Method**: while decoding for feature extraction, `hdd/extract_features.py` also records each frame's horizontal image shift against the previous frame. `hdd/sync_check.py` computes, in 60 s windows, the normalised cross-correlation between this shift and the steering rate derived from CAN, with the Pearson correlation computed on the overlap only, over a search range of ±5 s. Only numbers are involved throughout; nobody looks at the frames.
  - **Calibration**: known offsets injected into real windows are recovered one for one (slope 0.999, `--selftest`); offsets of 100/300/500/1000 ms injected into smooth synthetic signals are estimated as 100/300/499/999 ms. [`tests/test_sync_estimator.py`](../tests/test_sync_estimator.py) guards this property with synthetic signals.
  - **Result (frames against the steering angle, the signal the labels use)**: 134 sessions have usable windows, 2348 windows in all, median |correlation| 0.74; the median of the per-session lags is **0.000 s**. Over the 88 sessions with ≥ 10 windows, p10 is −0.100 s and p90 +0.067 s; the resolution of the estimate is one frame, about 33 ms.
  - **Frames against the yaw rate give −0.100 s**: the CAN yaw rate lags the frames by about 0.1 s, which looks like gyro filtering delay. **Label timing must be referenced to the steering angle**; using the yaw rate would wrongly suggest shifting the labels by 0.1 s.
  - **Sessions with clock drift show no extra misalignment**: 51 drift windows (6 sessions), median lag 0.000 s, slope against the clock deviation 0.138; a receive-side backlog would give a slope close to 1. Searching the 13 drifting sessions alone with ±15 s gives the same conclusion.
  - **Effect of the search range**: the per-session medians are 0.000 s with both ±2 s and ±5 s; widening to ±5 s increases the usable windows from 1816 to 2348 and reduces the sessions without a usable window from 9 to 3. The default is now ±5 s.
  - **Unchecked**: 3 sessions have no usable window (too few or too short turns) and do not count as confirmed.
  - **Single windows are noisy**: the per-frame shift is quantised to whole pixel columns (about 0.4°), so conclusions use per-session medians, not single windows. Quantisation lowers the correlation but does not move its peak, as a test on synthetic signals shows.
- **A visual spot check is still recommended** as an independent confirmation of the automatic check above, but it is no longer a prerequisite for the single-frame experiment. A few videos with the steering curve overlaid could be rendered to files on the server and downloaded for the owner to watch.

## Implications for the design

The decisions for the team are in [`discussion_2026-09-21_next_steps.md`](https://github.com/Euswbnix/CSC413_Project/blob/b209e2e/docs/discussion_2026-09-21_next_steps.md); this document does not decide for the team. The main ones:

1. **Main question and split**: whether to test on "new drives" or on "new geographic areas"; the discussion document compares holding out whole sessions grouped by route, a random split by session and 2 km geographic blocks.
2. ~~**Sync check**~~: done, see section 4; the frames and the steering angle are aligned, and stretches with clock drift need not be excluded.
3. **Use moving frames only**: the steering angle while stopped can hardly be seen in the frames.
4. **Supplementary condition "gaps in the original timestamps"**: the main condition uses only samples whose history window has no long gap; samples with long gaps are evaluated separately; Δt is computed from the camera clock.
5. **The mixed-frame-rate session and the 9 sessions without usable GPS**: suggested not to be used in the main experiment.
6. **If combined with nuScenes**: convert radians to degrees first, then unify the sign. If SullyChen is used as an out-of-domain test set, check its sign convention first as well.

## Disk

- On the server: the downloaded raw archives, 149 GB (kept), and the extracted CSVs, 23 GB.
- The videos are still inside the archives and have not been extracted.
- Reading the videos does not require extracting them all (that would need another 112 GB): each video is written to disk temporarily and deleted once processed. All 137 were read this way for the metadata check.

## Reproduction

```bash
python hdd/week1_checks.py --raw ~/workspace/hdd/raw --split trn_data_info.json --out ~/workspace/hdd/checks --fallback-ratio 3.6
python hdd/timestamp_gaps.py --raw ~/workspace/hdd/raw --out ~/workspace/hdd/checks/timestamp_gaps.json
python hdd/geo_split_feasibility.py --tracks ~/workspace/hdd/checks/tracks_1hz.npz --split trn_data_info.json \
    --sessions-csv ~/workspace/hdd/checks/sessions.csv --exclude <4 sessions; IDs are in the split file on the server>
# drift_exclusion.py is in the earlier repository
python hdd/drift_exclusion.py --raw ~/workspace/hdd/raw --checks ~/workspace/hdd/checks \
    --exclude <4 sessions; IDs are in the split file on the server>
python hdd/extract_features.py --videos ~/workspace/hdd/video --raw ~/workspace/hdd/raw \
    --out ~/workspace/hdd/features/dinov2_s10 --stride 10 --workers 6
python hdd/sync_check.py --features ~/workspace/hdd/features/dinov2_s10 --raw ~/workspace/hdd/raw \
    --out ~/workspace/hdd/checks/sync_check.json --signal steer --max-lag-s 5
python hdd/sync_check.py --features ~/workspace/hdd/features/dinov2_s10 --raw ~/workspace/hdd/raw --selftest
```

- `trn_data_info.json` is `data/data_info.json` from the TRN.pytorch repository.
- `--fallback-ratio 3.6` is used only for sessions without GPS, to convert km/h to m/s.
- The CAN, GPS, timestamp and split statistics come from a single run of the current three scripts on 2026-09-21; the split numbers count only moving points with video and a valid label. The video container metadata (frame count, resolution, frame rate) was read by the earlier repository's [`hdd/probe_videos.sh`](https://github.com/Euswbnix/CSC413_Project/blob/b209e2e/hdd/probe_videos.sh); the interval breakdown of the mixed-frame-rate session was computed separately.
