# 05 — The maths: from green values to beats per minute

All of it is in `src/bin/00_heart_rate_face_cv/signal.rs`, about 150
lines, no maths library except `sin`/`cos`.

## 1. One number per frame

Each pixel is RGB565: `RRRRRGGG GGGBBBBB`. Green has 6 bits: 0–63.

```rust
let green = (pixel >> 5) & 0x3f;   // drop 5 blue bits, keep 6 green bits
```

We average all 38,400 pixels of the view. One pixel is noisy, but random
noise shrinks with the square root of the count: √38400 ≈ 196 times less.
That is how a 6-bit channel can show changes far smaller than 1 level.

## 2. Sampling and Nyquist

We get one sample per frame, 20 per second (20 Hz). To see a frequency,
you need **more than 2 samples per cycle** (Nyquist). 180 BPM = 3 Hz needs
more than 6 Hz. 20 Hz is plenty.

`BPM = Hz × 60`. So 40–180 BPM = 0.67–3 Hz.

## 3. Resampling onto an even grid

Frames arrive about every 50 ms, but not exactly, and a few are lost. We
read the signal at exact 50 ms steps by **linear interpolation**: between
two samples, draw a straight line and read it at the grid time.

```text
value = before + (after − before) × (t − t_before) / (t_after − t_before)
```

After this, every later step can assume even spacing.

## 4. Removing the trend

Slow changes (light drifting, a small movement) are much bigger than the
pulse. We fit the **least-squares line** through the samples and subtract
it:

```text
slope = Σ (k − k̄)(x − x̄) / Σ (k − k̄)²      the line goes through (k̄, x̄)
```

Why not "subtract a 1 s moving average"? That is a filter too, and its
effect falls **inside** our band: −40% at 40 BPM, +20% near 90 BPM. It
would move the peak. A line over 10 s only removes changes slower than
~6 BPM.

## 5. The Hann window

Cutting 10 s out of a longer signal makes hard edges. The DFT sees those
edges as extra frequencies ("spectral leakage"). Multiplying by a smooth
bump that is 0 at both ends fixes that:

```text
w(k) = 0.5 − 0.5 · cos(2π k / (N − 1))
```

## 6. The DFT: "how much of this frequency is in there?"

For each candidate frequency *f*, multiply the signal by a cosine and a sine
of *f* and add everything up:

```text
power(f) = (Σ x_k · cos 2πf t_k)² + (Σ x_k · sin 2πf t_k)²
```

If the signal contains *f*, the products line up and the sums get big.
Otherwise they cancel out. We check 141 frequencies: 40, 41, …, 180 BPM.

**The rotating pointer.** `(cos θ, sin θ)` is a point on a circle. On an
even grid, θ grows by the same step each sample, and turning a point by a
fixed step is one small multiplication:

```text
cos' = cos·c − sin·s        sin' = sin·c + cos·s      (c, s = cos/sin of the step)
```

So `sin`/`cos` are computed twice per frequency instead of 400 times.
Total: ~28,000 multiply-adds, about a millisecond.

## 7. Resolution and the peak

- **Resolution = 1 / window length.** 10 s → 0.1 Hz = **6 BPM**. 5 s would be
  12 BPM. Checking every 1 BPM draws a smooth curve, but two heart rates
  closer than ~6 BPM still look like one peak.
- **Parabola refinement.** Fit a parabola through the highest point and its
  two neighbours, and take its top:
  `offset = 0.5 · (left − right) / (left − 2·middle + right)`.
- **Trade-off:** a longer window = sharper and steadier, but slower to react.
  A change in heart rate shows fully only after ~10 s.

## Checked, not assumed

The same algorithm in 60 lines of Python, on a fake pulse with drift, noise,
jittered frame times and a missing frame, found 72, 55, 110 and 160 BPM to
within 0.2 BPM. A quick simulation like this catches maths bugs before
they are on a device where they are hard to see.
