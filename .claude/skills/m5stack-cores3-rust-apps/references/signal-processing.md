# Signal processing on the device

Working code: `src/bin/00_heart_rate_face_cv/signal.rs` (ring buffer,
resampling, detrend, Hann window, DFT, peak). About 150 lines, only `libm`.

## Keep samples in a ring buffer in PSRAM

A fixed array used in a circle (`start`, `len`, index `(start + i) % CAP`),
allocated once with `psram::leaked_slice`. Store `(Instant, value)` and drop
samples older than the history length on `push`. Nothing is shifted.

## Averaging many pixels

Random noise shrinks with √N: averaging 38,400 pixels of a 6-bit channel
resolves changes far below 1 level. It does not remove slow or correlated
disturbances (movement, light changes, auto exposure).

## Sampling limits

- **Nyquist:** the fastest rhythm you can see is fs / 2. Camera 20 FPS →
  10 Hz = 600 BPM.
- **Frequency resolution:** Δf = fs / N = 1 / T. Only the window length T
  matters: 10 s → 0.1 Hz = 6 BPM, 5 s → 12 BPM. More FPS does not sharpen
  the peak; it only averages random noise a bit better (frequency error
  ~ 1 / (T · √(N · SNR)), so double FPS ≈ 1.4× better, double T ≈ 2.8×).
- Trade-off: longer windows are steadier but react slower (a change shows
  fully only after ~T).

## The pipeline that worked

1. **Resample** the last T seconds onto an even grid (e.g. 20 Hz) by linear
   interpolation between neighbouring samples. Handles jitter and dropped
   frames once, so the rest can assume even spacing.
2. **Remove the trend** with a least-squares line:
   `slope = Σ(k−k̄)(x−x̄) / Σ(k−k̄)²`, subtract `x̄ + slope·(k−k̄)`.
   Avoid "subtract a short moving average": it is a filter whose response
   lies inside a typical band (a 1 s average removes ~40% at 0.67 Hz and
   boosts ~20% near 1.5 Hz), so it moves peaks.
3. **Hann window** `w(k) = 0.5 − 0.5·cos(2πk/(N−1))` against leakage from
   the cut edges. It widens peaks ~1.4× (a clean 72 BPM peak at half power:
   17 BPM wide with 5 s, 8 BPM with 10 s).
4. **DFT only over the band you need**, in steps finer than the
   resolution if you want a smooth curve. With a rotating phasor, sin/cos
   are needed once per frequency:

```rust
let (s, c) = (libm::sinf(omega * dt), libm::cosf(omega * dt));
let (mut cos, mut sin, mut re, mut im) = (1.0_f32, 0.0_f32, 0.0_f32, 0.0_f32);
for &x in &signal {
    re += x * cos;
    im += x * sin;
    (cos, sin) = (cos * c - sin * s, sin * c + cos * s);
}
let power = re * re + im * im;
```

   141 frequencies × 200 samples ≈ 28k multiply-adds, about a millisecond.
   Pass a `pump: impl FnMut()` closure and call it every ~16 frequencies so
   the maths module never needs the camera.
5. **Peak refinement:** parabola through the maximum and its neighbours,
   `offset = 0.5·(l − r) / (l − 2m + r)` (only when the curvature is
   negative).

## Verify on the host first

`python3` is on the host (no numpy). Re-implement the pipeline in plain
Python and feed it a known signal with drift, noise, jittered timestamps and
a missing sample; the heart rate version recovered 55–160 BPM within
0.2 BPM. The same script can generate chart data for slides or docs.
