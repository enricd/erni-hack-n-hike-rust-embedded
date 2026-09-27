# 01 — Embedded Rust, and this repo's examples

Builds on [00_rust_intro.md](00_rust_intro.md). Here: what changes when Rust
runs on a microcontroller with no operating system, and what every `src/bin/`
file's top lines mean. Reference: [`src/bin/light_meter.rs`](../src/bin/light_meter.rs).

## `#![no_std]` — no operating system underneath

Normal Rust programs use `std`, the standard library. `std` assumes there's
an OS: it gets memory from the OS, opens files, spawns threads.

The ESP32-S3 in the board runs no OS. There's nothing under your program but
the chip itself. So every crate here (`Cargo.toml`'s `hack-and-hike` and
`hack-and-hike-core`) starts with:

```rust
#![no_std]
```

This says "don't assume an OS is there." You lose `std::` things like
`String`, `Vec`, `println!`, threads, files. You keep `core::`, the part of
the standard library that needs no OS: `Option`, `Result`, `match`, integer
math, `core::fmt::Write` for formatting text into a buffer.

Notice `light_meter.rs` uses `core::fmt::Write`, not `std::fmt::Write`, and
`ArrayString` (a string with a fixed, compile-time capacity) instead of
`String` (which needs a heap and grows). Same idea for numbers: `Point`-like
math uses `libm` (a `no_std` math library) instead of `std`'s `f32::sqrt()`
convenience methods, because those come from `std`.

A heap (dynamic allocation, `Box`, growable `Vec`) is *possible* in
`no_std` — this repo has `extern crate alloc;` in `src/lib.rs` and uses
`esp-alloc` to provide one — but it's opt-in and limited by the chip's RAM,
so the repo prefers fixed-size types where it can.

## `#![no_main]` — no `fn main() -> ()`  the normal way

Normally the OS loader calls your `fn main()`. Here, there's no loader —
so `main` can't be an ordinary function; it needs the chip-specific startup
code to call it. `#![no_main]` tells the compiler not to expect the usual
`main`, so a macro can define the real entry point instead:

```rust
#[esp_rtos::main]
async fn main(_spawner: Spawner) -> ! {
    ...
}
```

`#[esp_rtos::main]` expands into whatever startup code the chip needs (set
up the stack, zero memory, etc.), then calls this `async fn main`. The
return type `!` ("never") means this function does not return — makes sense,
there's no OS to return control to. It runs forever.

## `async`/`.await` — cooperative multitasking without an OS

`async fn` means this function can pause at an `.await` point and let other
work run, then resume later. Normally `async` needs an OS-backed executor
(tokio, etc.); here, `embassy_executor::Spawner` is a tiny executor that runs
on the chip itself, no OS needed. It's how the repo does several things "at
once" (read a sensor, redraw the screen, handle the network) on one CPU
without threads:

```rust
use embassy_time::{Duration, Timer};

loop {
    // ... read input, update state, draw what changed ...
    Timer::after(Duration::from_millis(10)).await;
}
```

`Timer::after(...).await` pauses *this* task for 10ms and lets any other
`embassy` task run meanwhile, then resumes. Compare to a plain
`for` loop with no `.await`: that would busy-loop and starve everything else.

## `esp_bootloader_esp_idf::esp_app_desc!();`

A macro call, not a function. It embeds a small metadata block (app name,
version) into the compiled binary, in the format the ESP-IDF bootloader
expects to find at a fixed offset. Every `src/bin/*.rs` has exactly one of
these, near the top. You don't call it again or use its output; it's there
purely so the bootloader can identify the firmware.

## `Board::init()` — the hardware, wrapped

```rust
let Board {
    mut display,
    light,
    proximity,
    ..
} = Board::init();
```

`Board::init()` (defined in `src/board/`, re-exported from `src/lib.rs`)
turns on the chip's peripherals once and hands back a `Board` struct: one
typed field per capability (`display`, `touch`, `imu`, `light`, `proximity`,
`network`, ...). This is destructuring — the `let Point { x, y } = p;` pattern
from 00_rust_intro.md, applied to picking only the fields this app needs.
`..` means "ignore the rest of the fields."

Why a struct instead of, say, one function per sensor: it guarantees
`init()` runs exactly once, and each handle (e.g. `display`) can only be
owned by one part of your code at a time — Rust's ownership rules
(00_rust_intro.md) apply to hardware pins the same way they apply to a
`String`. Two parts of your program can't accidentally fight over the same
pin at compile time.

## `Option` on hardware handles

```rust
let (Some(mut light), Some(mut proximity)) = (light, proximity) else {
    canvas.clear(theme::WHITE);
    ...
};
```

`light` and `proximity` come out of `Board::init()` as `Option<Light>` and
`Option<Proximity>`, not `Light`/`Proximity` directly — because both sensors
live on one chip, and that chip might fail to initialise (loose connection,
etc.). If it does, both are `None` instead of the program crashing. The
comment above this line in the file explains it: "Both handles come from one
chip. So either both are there, or neither." This is exactly the `Option`
pattern from 00_rust_intro.md, `let ... else` is a shortcut for "match, and
if it's the failure case, do this and leave the function/scope."

## What to read next in this repo

- `crates/core/src/lib.rs` — the hardware-independent logic (sensor maths,
  decoding), plain `no_std` Rust with unit tests. Good place to see `no_std`
  code with no chip-specific concerns at all.
- `src/lib.rs` — the top of the firmware crate, lists every capability
  module.
- `src/bin/template.rs` — presumably the smallest example; a good starting
  point before reading the fuller apps like `light_meter.rs` or `demo/`.

## Summary

| Seen in the code | Means |
| --- | --- |
| `#![no_std]` | no OS underneath; only `core`, not `std` |
| `#![no_main]` | no ordinary `main`; a macro supplies the real entry point |
| `#[esp_rtos::main] async fn main(...) -> !` | the real entry point, runs forever |
| `.await` / `Timer::after(...).await` | pause this task, let others run, without threads |
| `esp_app_desc!()` | embeds metadata the bootloader needs |
| `Board::init()` | turns on hardware once, hands back typed, owned handles |
| `Option<Light>` etc. on handles | "this sensor might not be there" |
