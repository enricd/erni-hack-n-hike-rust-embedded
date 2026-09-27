# Timing and debugging

## An on-screen timing panel

Add time per phase over one second, reset it every second, and draw the
previous second's numbers on the screen (not the log: logging blocks for
milliseconds and can cost a camera frame).

```rust
#[derive(Default)]
struct LoopTimes { begin: u64, work: u64, finish: u64, dashboard: u64, restarts: u32 } // µs

let start = Instant::now();
if let Some(mut frame) = camera.begin_frame() {
    let t = Instant::now();
    let begin = t.duration_since(start);
    times.begin += begin.as_micros();
    if begin > Duration::from_millis(10) { times.restarts += 1; } // begin normally returns at once
    // ... work ...
    let worked = Instant::now();
    times.work += worked.duration_since(t).as_micros();
    frame.finish();
    times.finish += worked.elapsed().as_micros();
}
```

Format each value as ms per second (`value / 1000`) into
`ArrayString<24>` lines, e.g. `write!(line, "{name:<10}{value:>5}")`.

How to read it (the parts add up to ~1000 ms per second):

| Pattern | Meaning |
| --- | --- |
| `begin` large (hundreds) | frames are dropped: each drop re-primes the capture (~2 frame periods) |
| `finish` large, `begin` ≈ 0 | waiting for the sensor: this is free time |
| one work phase large | that phase is the bottleneck |

**Attribute drops to their cause:** remember what ran just before
(`enum Last { Render, Analyse, Dashboard }`) and count each restart for it.
That separated "the dashboard drops frames" from "rendering drops frames"
in one run.

## Case history (heart rate app, 20 FPS sensor)

| Version | FPS | What the numbers showed |
| --- | --- | --- |
| camera only | 20 | baseline, limited by the sensor |
| + plot via `Canvas::show` at 4 Hz | 12.5 | `Canvas` cannot pump; 4 redraws × ~2 lost frames ≈ 8 frames/s |
| + own image, pump in every `fill_row` | 8–9 | `begin` 582 ms/s: still restarting almost every frame |
| + clear image in slices, 2 Hz, show every 3rd frame | 20, 0 drops | also 20 with every frame shown: the one-piece 77 KB clear was the likely cause |

Per-call costs measured then: show a 160x240 camera half ~28 ms, read one
for analysis only ~16 ms, dashboard redraw ~57 ms, DFT (141 frequencies ×
200 samples) about a millisecond.

## Other facts

- The camera logs only the 1st and then every 32nd dropped frame
  (`Camera frame dropped: ... (N bad frames so far)`). Silence is not zero.
  FPS or a restart counter is the better check.
- Drops at start-up (one warning) are normal.
- Time: `embassy_time::Instant::now()`, `elapsed()`, `duration_since()`,
  `checked_duration_since()` (no panic when out of order), `as_micros()`.
- Tell the user exactly what to report: which numbers, which log lines, a
  photo of the screen. A photo of the screen with the panel on it is the
  fastest round trip.
