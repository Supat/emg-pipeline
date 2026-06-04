# emg-pipeline

Synchronized **surface-EMG + motion-capture** recordings and an exploratory analysis pipeline.

Each recording captures forearm muscle activity (Delsys Trigno EMG via an NI DAQ) together with
OptiTrack motion capture, streamed and time-synchronized through
[Lab Streaming Layer (LSL)](https://labstreaminglayer.org/) and stored as `.xdf` files.

## Repository layout

```
data/                 13 .xdf recordings (Iwami-{session}-{trial}.xdf)
fig/sensors/          EMG sensor-placement photos (sensors labeled 1–7)
explore_data.ipynb    dataset exploration notebook (structure + sample plots)
explore_data.pdf      rendered export of the notebook
```

## Dataset

13 recordings named `Iwami-{session}-{trial}.xdf` (~150 MB total):

| Session | Trials | Duration |
|---------|--------|----------|
| 1 | 1–5 | ~17–25 s |
| 2 | 1–5 | ~22–26 s |
| 3 | 1–2 | ~69–73 s |
| 4 | 1   | ~43 s |

Every file contains **two time-synchronized LSL streams**:

### EMG — "National Instruments USB-6225"
- **28 channels**, `float32`, nominal **2000 Hz** (effective 1846–2001 Hz; sessions 3-1 and
  4-1 drift lower).
- Raw analog volts; the stream carries **no channel labels**.
- The 28 channels are **7 Delsys Trigno sensors × 4 channels** in per-sensor blocks of
  `EMG, Acc_X, Acc_Y, Acc_Z`. So sensor *n* occupies channels `4·(n-1) … 4·(n-1)+3`, with the
  EMG signal on channels `0, 4, 8, …, 24`. (This layout is reconstructed and verified against
  the signal in the notebook — EMG channels are zero-mean with high-frequency content; the
  accelerometer channels carry a DC/gravity offset.)
- Sensor positions on the forearm (labeled 1–7) are documented in `fig/sensors/`.

### Mocap — "OptiTrack"
- **76 channels**, `float32`, nominal **240 Hz** but effectively **~120 Hz** across all files
  (trust the timestamp-derived rate, not the nominal field).
- Per-channel labels are present: raw marker positions, per-bone markers for two rigid bodies,
  then each rigid body's pose — position `X/Y/Z` (meters), quaternion `A/B/C/D`, and a
  confidence channel.

### Synchronization
Both streams share the same LSL clock and `pyxdf` applies clock-offset dejittering. Note the two
streams do not start at exactly the same instant (e.g. mocap starts ~0.45 s after EMG in
`Iwami-1-1`), so alignment/resampling onto a common time grid is required before joint analysis.

## Getting started

Requires Python 3.10+.

```bash
pip install pyxdf numpy matplotlib jupyter
jupyter notebook explore_data.ipynb
```

Loading a recording:

```python
import pyxdf

streams, header = pyxdf.load_xdf("data/Iwami-1-1.xdf")
by_type = {s["info"]["type"][0]: s for s in streams}
emg, mocap = by_type["EMG"], by_type["Mocap"]

emg_data = emg["time_series"]      # (n_samples, 28)
emg_t    = emg["time_stamps"]
```

## The notebook

`explore_data.ipynb` walks through the dataset:

0. **EMG sensor placement** — the `fig/sensors/` photos (7 Delsys Trigno sensors).
1. **File inventory** — channel counts, nominal vs. effective sampling rate, duration per stream.
2. **One recording** — stream shapes, dtypes, and the inter-stream start offset.
3. **EMG channel layout** — rebuilds and verifies the 7-sensor channel map.
4. **EMG time-series** — per-sensor EMG, a zoomed 1-second window, and accelerometer channels.
5. **Mocap time-series** — rigid-body positions and both 3-D trajectories.
6. **Shared timeline** — EMG vs. mocap on a common axis (starting point for alignment).

## Notes

- Raw `.xdf` data is committed directly to the repository. The largest file (~19 MB) is under
  GitHub's 100 MB limit; longer recordings added later may need [Git LFS](https://git-lfs.com/).
- The physical sensor → channel mapping (which of sensors 1–7 maps to which muscle, and the
  exact NI wiring order) depends on the acquisition notes and the `fig/sensors/` photos; it is
  not recorded in the `.xdf` files themselves.
