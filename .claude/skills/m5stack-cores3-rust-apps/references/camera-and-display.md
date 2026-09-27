# Camera and display patterns

Working code: `src/bin/00_heart_rate_face_cv/` (`view.rs`, `dashboard.rs`,
`main.rs`) and the demo's `src/bin/demo/screens/camera.rs`.

## How a frame moves

```text
sensor ─8-bit bus─> LCD_CAM ─DMA─> ring (internal RAM, 40 rows ≈ 5 ms)
                                      │ pump() copies it out (never waits)
                                      ▼
            capture buffer ─VSYNC─> ready buffer ─finish()─> display buffer   (3 × 150 KB PSRAM, pointer swaps)
                                                                  │ frame.scanline(y): 640 bytes
                                                                  ▼
                                      your fill_row ─SPI DMA, batches of 7 rows─> panel
```

- `camera.begin_frame() -> Option<Frame>`: the first call after start-up,
  `pause()` or an error waits for a whole frame ("prime"); later calls return
  at once. `None` = capture failed, the next call retries.
- `frame.scanline(y)`: row `y`, 320 pixels × 2 bytes, **big-endian RGB565**
  (`RRRRRGGG GGGBBBBB`), the same format as the panel: copying is enough.
- `frame.finish()`: swap in the newest complete frame; waits if none is
  ready (normally < 1 frame period, at most 250 ms).
- `camera.pump()` works only between `finish` and the next `begin_frame`;
  inside a frame use `frame.pump()`.
- The sensor is mounted rotated; the driver already mirrors it like a
  phone's front camera.

Pixel channels from the two bytes:

```rust
let v = u16::from_be_bytes([pixel[0], pixel[1]]);
let red = (v >> 11) & 0x1f;   // 0..31
let green = (v >> 5) & 0x3f;  // 0..63 (one bit more than red/blue)
let blue = v & 0x1f;          // 0..31
```

## Show a crop and compute something on the way

The display asks a `ScanlineSource` for each row. Do per-pixel work there:
it happens in the pass that sends the image anyway.

```rust
pub(crate) const WIDTH: usize = 160;                       // left half
const CROP_LEFT: usize = (camera::WIDTH - WIDTH) / 2;
const SOURCE_BYTES: Range<usize> =
    CROP_LEFT * BYTES_PER_PIXEL..(CROP_LEFT + WIDTH) * BYTES_PER_PIXEL;

pub(crate) struct CameraView<'a, 'f> {
    frame: &'a mut Frame<'f>,
    green_sum: u32,
}

impl ScanlineSource for CameraView<'_, '_> {
    fn fill_row(&mut self, y: usize, row: &mut [u8]) {
        // Per-pixel work makes filling slower than the SPI bus, and then
        // `while_transferring` never runs. So pump here too.
        self.frame.pump();
        row.copy_from_slice(&self.frame.scanline(y)[SOURCE_BYTES]);
        self.green_sum += green_sum(row);   // compute BEFORE painting overlays into `row`
    }
    fn while_transferring(&mut self) {
        self.frame.pump();
    }
}

// in the loop:
if let Some(mut frame) = camera.begin_frame() {
    let mut view = CameraView { frame: &mut frame, green_sum: 0 };
    display.surface(LEFT_HALF).render_from(&mut view);
    let result = view.green_sum;   // `view` must be unused after this line
    frame.finish();
}
```

**Overlays** (boxes, markers): paint into `row` after computing on it. For a
rectangle: on its top and bottom rows fill the column range; on rows in
between set only the left and right pixel. Colour bytes:
`RawU16::from(color).into_inner().to_be_bytes()` (needs
`embedded_graphics::prelude::*` for `RawData`).

**Frames you don't show** can still be analysed, but reading PSRAM costs
~16 ms per 160x240 crop, so pump per row:

```rust
for y in 0..camera::HEIGHT {
    frame.pump();
    sum += green_sum(&frame.scanline(y)[SOURCE_BYTES]);
}
```

Showing only every Nth frame (`RENDER_EVERY`) while analysing every frame
freed time; showing every frame also reached 20 FPS once the dashboard was
fixed.

## A dashboard next to the camera: own buffer, not `Canvas`

`Canvas::show` cannot pump, and a big redraw lost frames (20 → 12 FPS).
Draw into your own PSRAM image and send it like the camera:

```rust
struct PixelBuffer { pixels: &'static mut [Rgb565] }       // psram::leaked_slice(W * H, BG)

impl OriginDimensions for PixelBuffer { fn size(&self) -> Size { SIZE } }

impl DrawTarget for PixelBuffer {
    type Color = Rgb565;
    type Error = Infallible;
    fn draw_iter<I: IntoIterator<Item = Pixel<Rgb565>>>(&mut self, pixels: I) -> Result<(), Infallible> {
        for Pixel(p, c) in pixels {
            if let (Ok(x), Ok(y)) = (usize::try_from(p.x), usize::try_from(p.y)) && x < W && y < H {
                self.pixels[y * W + x] = c;
            }
        }
        Ok(())
    }
}

struct Rows<'a> { pixels: &'a [Rgb565], camera: &'a mut Camera }

impl ScanlineSource for Rows<'_> {
    fn fill_row(&mut self, y: usize, row: &mut [u8]) {
        self.camera.pump();
        for (bytes, px) in row.chunks_exact_mut(2).zip(&self.pixels[y * W..(y + 1) * W]) {
            bytes.copy_from_slice(&RawU16::from(*px).into_inner().to_be_bytes());
        }
    }
    fn while_transferring(&mut self) { self.camera.pump(); }
}
```

Drawing: clear in slices (`pixels.chunks_mut(W * 20)` → `fill` → `pump`),
pump every ~64 plotted points, then `surface.render_from(&mut Rows { .. })`.
`common::text` only takes a `Canvas`; for your own target use
`Text::with_baseline(s, origin, MonoTextStyle::new(font, color), Baseline::Top).draw(&mut img)`.
Cost measured: ~57 ms per 160x240 redraw, so redraw at 2 Hz.

## Simple shapes: compute each row, no buffer

Bars, gauges, borders and solid areas can be computed per row in
`fill_row`: no PSRAM image, no clear, far cheaper than the buffer above.
Use the buffer only for text and free-form plots.

```rust
struct BarRows<'a> { camera: &'a mut Camera, filled: usize, color: Rgb565 } // filled: 0..=H

impl ScanlineSource for BarRows<'_> {
    fn fill_row(&mut self, y: usize, row: &mut [u8]) {
        self.camera.pump();
        let c = if y >= H - self.filled { self.color } else { theme::CHARCOAL };
        let be = RawU16::from(c).into_inner().to_be_bytes();
        for bytes in row.chunks_exact_mut(2) { bytes.copy_from_slice(&be); }
    }
    fn while_transferring(&mut self) { self.camera.pump(); }
}
```

## Plots that show small changes

Scale the y axis from the window's min to max (guard `max - min` with a
floor such as 0.01), put the newest sample at the right edge by age, and
draw `Line` segments between consecutive points. Print min and max as text
so the viewer knows the scale.

## Camera exposure

Auto exposure and auto white balance change the image by many levels
within a second. For measurements, let them settle a few seconds, then
`camera.set_auto_adjust(false)`; `true` re-enables. The log confirms the
registers (`Camera auto adjust locked: AEC 0x10, AWB 0x55`).
