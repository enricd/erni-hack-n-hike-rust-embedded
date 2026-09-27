---
name: m5stack-cores3-rust-apps
description: Measured rules and ready patterns for writing Rust no_std apps on the M5Stack CoreS3 Lite (ESP32-S3) in the hack-and-hike-rust-2026 repo - camera, display and dashboards, touch, IMU, audio, ESP-NOW network, light/proximity sensors, on-device maths (averaging, filters, DFT/FFT), frame-rate and timing debugging, and changes to the shared hack_and_hike library. Use this skill whenever creating or changing any app under src/bin/, adding features to src/capabilities/, investigating low FPS, dropped camera frames, stack or PSRAM problems, or processing sensor data on this board - even when the user only says "new app", "game", "demo", "hack" or "simulation" for the board.
---

# Rust apps on the M5Stack CoreS3 Lite

Everything here was learned, and measured on the board, while building
`src/bin/00_heart_rate_face_cv/` (camera → green signal → heart rate). Its
`PLAN.md` has the full story; this skill is the reusable part.

## Workflow in this repo

- **Cargo only runs inside the dev container.** The host has no Rust
  toolchain. The repo is bind-mounted into the container, so files you edit
  are visible there at once. You write the code; the user runs the commands
  in the container terminal and reports the result (FPS, log lines, a
  photo of the screen). Ask for exactly what you need to see.
- **An app** is `src/bin/<name>.rs`, or a folder `src/bin/<name>/main.rs`
  plus modules. Cargo finds it by itself. Names may start with digits
  (`00_heart_rate_face_cv` works).
- **Build and flash:** `cargo dist --bin <name>` writes `firmware.bin` to the
  repo root; autoflash (browser, Web Serial) flashes it. Without `--bin`,
  espflash fails with "multiple build artifacts".
- **CI checks every app:** `cargo fmt --check` and
  `cargo clippy --release -- -D warnings`. Since you cannot run rustfmt,
  ask the user to run `cargo fmt` before building. Every warning is an
  error: no unused imports or variables. Prefer `#[allow(lint)]` over
  `#[expect(lint)]` unless you are sure the lint fires, because an
  unfulfilled `expect` is itself a warning.
- **Before claiming something works,** say what was not verified. Code you
  could not compile is "written", not "done".
- A `PLAN.md` in the app folder with steps and a **"Measured"** section
  (numbers, what was running, what they mean) pays off: later decisions
  rely on those numbers.

## App skeleton

Start from `src/bin/template.rs` (it passes CI). Its shape:

```rust
#![no_std]
#![no_main]

use embassy_executor::Spawner;
use embassy_time::{Duration, Timer};
use hack_and_hike::{Board, capabilities::{display::{SCREEN, SIZE}, touch::TouchEvent}, ui::{Canvas, theme}};

esp_bootloader_esp_idf::esp_app_desc!(); // exactly once per app

#[esp_rtos::main]
async fn main(_spawner: Spawner) -> ! {
    let Board { mut display, mut touch, .. } = Board::init(); // once; `..` drops the rest
    let mut canvas = Canvas::new(SIZE);                          // create before the loop
    let mut pressed = false;
    let mut shown = None;
    loop {
        while let Some(event) = touch.next_event() {             // 1. read input
            pressed = !matches!(event, TouchEvent::Released(_));
        }
        if shown != Some(pressed) {                              // 2. draw only on change
            shown = Some(pressed);
            canvas.clear(if pressed { theme::LIGHT_BLUE } else { theme::DARK_BLUE });
            canvas.show(&mut display.surface(SCREEN));
        }
        Timer::after(Duration::from_millis(10)).await;           // 3. NOT with the camera, see rule 2
    }
}
```

Every import and binding must be used (`-D warnings`): delete what you
don't need rather than leaving it. Dropped handles do not stop the
hardware: their CPU1 tasks keep running.

## What the board gives you (`Board` fields)

| Field | Read / use | Notes |
| --- | --- | --- |
| `display` | `display.surface(rect)` → `render_from(&mut impl ScanlineSource)` or `render_scanlines(\|y, row\| ..)` | 320x240 RGB565, SPI 40 MHz |
| `touch` | `while let Some(e) = touch.next_event()` → `TouchEvent::{Pressed, Moved, Released}(Point)` | queue from CPU1 |
| `imu` | `imu.latest() -> Option<Sample>`: `attitude.{roll_deg, pitch_deg, heading_deg, down}`, `acceleration_m_s2`, `angular_velocity_deg_s` | 100 Hz |
| `camera` | `Option<Camera>`: `begin_frame()`, `frame.scanline(y)`, `frame.finish()`, `pump()`, `pause()`, `set_auto_adjust(bool)` | 320x240, ~20 FPS; see rules |
| `light`, `proximity` | `Option<..>`; `.latest()` → `lux: f32` / `percent: u8, raw: u16` | same chip: both `Some` or both `None` |
| `microphone`, `speaker` | `next_block(&mut [i16; 1024])`, `write(&[i16])`, `available_frames()` | 16 kHz stereo interleaved; `hack_and_hike::synth::SineWave` |
| `network` | `broadcast(&msg)`, `send_to(peer, &msg)`, `next_message()` → `.decode::<T>()`, `peers()` | ESP-NOW; a message is a serde type with `impl Message { const NAME: &str = "team.kind"; }` |
| `backlight` | `set(Brightness::new(30))` | |
| `log` | log history | `log::info!` etc. also works |

UI helpers: `ui::Canvas` (draw with `embedded-graphics`, `show` sends only
changed pixels), `ui::common::{text, centered_text, DENSE_FONT, BODY_FONT,
TITLE_FONT}`, `ui::theme::*` colours, `ui::widgets::Slider`,
`psram::leaked_slice` for big buffers. `hack_and_hike_core` (in
`crates/core`) holds hardware-free logic with host tests.

## The rules that cost us the most time

1. **The camera never pauses, and its DMA ring holds only ~5 ms.** Something
   must call `camera.pump()` / `frame.pump()` every few milliseconds, or the
   frame is dropped and capture restarts (costs ~2 frame periods). Pump
   inside every long job: per row, per chunk of maths, between drawing
   steps.
2. **With the camera active, the loop must not sleep.** Use
   `embassy_futures::yield_now().await` instead of `Timer::after`. Without
   the camera, sleeping is fine and saves power.
3. **`while_transferring` runs only while the CPU waits for the display.**
   If your `fill_row` is slower than the SPI bus (any per-pixel work), the
   wait loop runs zero times and nobody pumps. Pump at the start of every
   `fill_row` too.
4. **PSRAM is slow: about 5 MB/s** reading fresh camera data. Reading one
   160x240 crop took ~16 ms, and clearing a 77 KB image in one go most
   likely ~15 ms. Touching pixels, not arithmetic, is the cost on this chip.
   Split big writes into slices with pumps in between.
5. **The display is the other bottleneck:** ~31 ms for a full screen,
   ~15 ms for half. `Canvas::show` sends only changed pixels, which is great
   without the camera, but it cannot pump, so with the camera use your own
   buffer + `render_from` (pattern in `references/camera-and-display.md`).
   Simple shapes (bars, gauges) need no buffer: compute each row in
   `fill_row`. Redraw dashboards at 2 Hz, and show every Nth camera frame
   if needed.
6. **No big arrays on the stack.** Task stacks are small (CPU1: 16 KiB for
   all tasks); clippy's `large_stack_frames` is denied. Put buffers in PSRAM
   with `psram::leaked_slice(len, value)` (never freed: create once).
7. **Logging blocks for milliseconds** and can itself cost a camera frame.
   Never log per frame; show debug numbers on the screen instead.
8. **Taps happen by accident** when the board is held. Don't put important
   or destructive actions on a plain tap.
9. **Budget:** the camera gives 20 FPS = 50 ms per frame. With the heart rate
   app (show every 3rd frame, green sum, 2 Hz dashboard) ~26 ms per frame
   were still free.

## Measure, don't guess

Twice, reading the code produced a wrong explanation for lost frames. An
on-screen timing panel (ms per second for each loop phase, and restart
counters by what ran before) found the cause in one run. Build that panel
early whenever speed matters: `references/timing-and-debugging.md`.

## Changing the shared library (`src/`)

Other apps depend on it. Ask the user first, change as little as possible,
and keep it backward compatible: add functions, do not change start-up
behaviour. To give an app access to hardware owned by CPU1 (e.g. anything
on the I2C bus after start-up), copy the backlight pattern: handle on CPU0,
task on CPU1, `Signal` between them. Details and the camera example
(`Camera::set_auto_adjust`): `references/cpu1-and-library-changes.md`.

## Rust gotchas we hit here

- A block expression at the start of a statement ends the statement:
  `match x { .. }.expect(..)` and `if c { a } else { b }.expect(..)` do
  not parse. Bind first: `let r = match x { .. }; r.expect(..);`
- Clippy wants `n.is_multiple_of(k)` instead of `n % k == 0`
  (`manual_is_multiple_of`).
- `while let Some(v) = self.iter().next() && .. { self.mutate(); }` keeps
  the borrow alive. Use a helper that returns a copy.
- `let Ok(()) = shape.draw(&mut target);` works because drawing errors are
  `Infallible`. Text draws return `Ok(Point)`: `let Ok(_) = ..`.
- Text without a heap: `ArrayString::<N>` + `core::fmt::Write`.
- `f32` maths without `std`: `libm::sinf`, `cosf`, `sqrtf`; `f32::min/max`
  are in `core`.
- Compile-time checks catch layout mistakes before flashing:
  `const _: () = assert!(A + B == display::WIDTH);`

## Maths on the device

For signal processing (averaging, detrending, windows, DFT, frequency
resolution, peak refinement), read `references/signal-processing.md`.
Check new maths in Python on the host first (`python3`, no numpy
installed): a 60-line simulation with fake noise catches bugs that are hard
to see on the device.

## Reference files

| File | Read when |
| --- | --- |
| `references/camera-and-display.md` | using the camera, streaming images, drawing a dashboard next to the camera, per-pixel work |
| `references/timing-and-debugging.md` | FPS too low, dropped frames, "is there time for X?", adding a timing panel |
| `references/cpu1-and-library-changes.md` | a feature needs I2C or another CPU1 resource, adding a function to `src/capabilities/`, registers and bit masks |
| `references/signal-processing.md` | sampling rates, filtering, spectra, heart rate or any periodic signal, precision questions |
