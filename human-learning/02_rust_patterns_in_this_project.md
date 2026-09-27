# 02 — Rust patterns used in the heart rate app

Builds on [00](00_rust_intro.md) and [01](01_embedded_rust.md). Each item
points at real code in `src/bin/00_heart_rate_face_cv/`.

## Modules and visibility

A folder app is one crate: `main.rs` plus modules declared with `mod`.

```rust
mod dashboard;   // loads dashboard.rs
mod signal;
mod view;
```

| Written | Visible in |
| --- | --- |
| *(nothing)* | this module only |
| `pub(super)` | the parent module and its children (`gc0308.rs` → all of `camera/`) |
| `pub(crate)` | the whole crate (our app, or the library) |
| `pub` | everyone, other crates too (`Camera::set_auto_adjust`) |

Child modules can always use private items of their parents: that is how
`camera/runtime.rs` calls the private `Runtime::next_request` in
`camera/mod.rs`.

## Implementing a library's trait for your own type

The library says *what* it needs with a trait; we provide *how*.

```rust
impl ScanlineSource for CameraView<'_, '_> {      // the display asks for rows
    fn fill_row(&mut self, y: usize, row: &mut [u8]) { ... }
    fn while_transferring(&mut self) { ... }      // has a default; we override it
}

impl DrawTarget for PixelBuffer {                 // embedded-graphics draws on it
    type Color = Rgb565;                          // associated types: the trait
    type Error = Infallible;                      // asks us to name them
    fn draw_iter<I>(&mut self, pixels: I) -> Result<(), Self::Error> { ... }
}
```

After `impl DrawTarget`, every line, rectangle and text of
`embedded-graphics` draws into our PSRAM buffer, although that library has
never heard of it.

**`Infallible`** is a type with no values: an error that can never happen.
So `let Ok(()) = line.draw(&mut image);` needs no `else`: the compiler
knows `Err` is impossible.

## Lifetimes in structs

A struct that **borrows** must say how long the borrowed thing lives:

```rust
pub(crate) struct CameraView<'a, 'f> {
    frame: &'a mut Frame<'f>,   // we borrow the Frame for 'a;
}                               // the Frame borrows the Camera for 'f
```

The compiler then proves that `CameraView` is gone before `frame.finish()`
consumes the frame. `'_` means "some lifetime, no need to name it".

## Borrowing in practice: three situations from this project

1. **A borrow that lives too long.** This fails:
   ```rust
   while let Some(oldest) = self.iter().next() && ... {
       self.remove_oldest();          // needs &mut self, but iter() still borrows
   }
   ```
   Fix: a helper that returns a **copy** (`fn oldest(&self) -> Option<Sample>`),
   so no borrow outlives the call.
2. **`&mut` → `&`.** `&*self.image.pixels` turns our `&mut [Rgb565]` into a
   shared `&[Rgb565]` for a moment (a *reborrow*).
3. **A closure that borrows.** `signal::spectrum(&history, now, || camera.pump())`
   borrows `camera` mutably only while `spectrum` runs. `signal.rs` never
   sees the camera, only "a function to call now and then".

## Compile-time values

```rust
const PERIOD: Duration = Duration::from_millis(500);          // computed at compile time
pub(crate) const fn right(&self) -> usize { ... }              // callable in a const
const _: () = assert!(camera::HEIGHT == display::HEIGHT);     // fails the BUILD, not the board
```

A wrong screen size is an error before anything is flashed.

## `'static` memory and PSRAM

A `static` lives for the whole program. `psram::leaked_slice(len, value)`
allocates once in PSRAM and never frees, so it can hand out a
`&'static mut [T]`. This is how the history (1000 samples) and the
dashboard image (77 KB) stay off the small stack.

## Iterators instead of index loops

```rust
row.chunks_exact(2)                      // 2 bytes = 1 pixel
    .map(|pixel| u16::from_be_bytes([pixel[0], pixel[1]]))
    .map(|value| u32::from((value >> 5) & 0x3f))
    .sum()
```

Also used: `enumerate` (index + item), `zip` (walk two sequences
together), `fold` (min/max in one pass), `iter_mut` (change in place).
No bounds mistakes, and the compiler optimises them like a loop.

## Small `Option` helpers used here

| Code | Means |
| --- | --- |
| `lock_at.is_some_and(\|at\| now >= at)` | "there is a time, and it has passed" |
| `since.is_none_or(\|s\| s.elapsed() >= P)` | "never drawn, or long enough ago" |
| `(count > 0).then(\|\| sum / count)` | `Some(value)` only if the condition holds |
| `a.checked_duration_since(b)` | `None` instead of a panic when `b` is later |
| `let Some(camera) = camera else { ... };` | unwrap, or leave the function |

## Text without a heap

```rust
let mut text = ArrayString::<16>::new();          // 16 bytes, on the stack
write!(text, "{} fps", fps).expect("fits");       // core::fmt::Write
```

A gotcha we hit twice: a `match` at the start of a statement ends there, so
`match x { ... }.expect(...)` does not parse. Bind it first:
`let written = match x { ... }; written.expect(...);`

## Tools that keep the code clean

- `cargo fmt` formats. CI runs `cargo fmt --check`: unformatted code fails.
- `cargo clippy -- -D warnings`: every lint warning is an error in CI.
- `#[allow(lint)]` silences a lint. `#[expect(lint)]` does too, but it
  becomes an error if the lint does **not** fire, so use it only when you
  are sure.
