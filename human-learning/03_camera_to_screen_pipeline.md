# 03 — From the camera to the screen

How pixels move on this board, and the timing rules that come with it. We
learned most of this by breaking it (see "Measured" in the app's
`PLAN.md`).

## The path of one frame

```text
sensor ──8-bit parallel bus──> LCD_CAM ──DMA──> ring (internal RAM, 40 rows ≈ 5 ms)
                                                   │ pump() copies it out
                                                   ▼
                              capture buffer ──> ready buffer ──> display buffer
                              (3 × 150 KB in PSRAM, swapped, never copied)
                                                                   │ scanline(y)
                                                                   ▼
                                           our fill_row ──SPI + DMA──> LCD panel
```

| Term | Meaning |
| --- | --- |
| **Parallel bus** | 8 wires, one byte per clock tick. The GC0308 sends 320x240 RGB565 = 153,600 bytes per frame. |
| **VSYNC** | A signal from the sensor: "a frame ends here". |
| **DMA** | Direct memory access: hardware copies bytes to/from memory **without the CPU**. The CPU only starts it and checks it. |
| **Ring buffer** | A small fixed buffer used in a circle. The DMA writes, `pump()` reads. If nobody reads for ~5 ms, it is full: the DMA stops and the frame is lost. |
| **Triple buffering** | 3 frame buffers: one being filled, one complete and waiting, one being shown. Swapping pointers is instant. The waiting one lets capture continue while the display is still busy. |
| **SPI** | The serial bus to the LCD, 40 MHz: a full screen takes ~31 ms, half a screen ~15 ms. |

## The rule of this board: pump, always

The sensor **never pauses**, so something must empty the ring every few
milliseconds, whatever the program is doing:

- `camera.pump()` / `frame.pump()`: copy what has arrived. Never waits,
  cheap when there is nothing new.
- The loop never sleeps: `yield_now().await` instead of `Timer::after`.
- Every long job pumps as it goes: per row (rendering, reading green), per
  16 frequencies (DFT), per 20 rows (clearing the dashboard).

## The trap: `while_transferring` runs only while the CPU **waits**

The display sends rows in batches of 7. While a batch is on the bus, it
calls our `while_transferring()` again and again:

```rust
while !transfer.is_done() {
    while_transferring();   // our pump()
}
```

If our `fill_row` is **slower** than the bus, the batch is already done
when we get there, the loop body runs **zero times**, and nobody pumps.
Fix: pump at the start of every `fill_row` too.

## Memory: internal RAM vs PSRAM

| | Internal SRAM | PSRAM |
| --- | --- | --- |
| Size | ~512 KB, shared with stacks | 8 MB |
| Speed | fast | slow, behind a cache; we measured ~5 MB/s reading fresh camera data |
| Used for | stacks, DMA rings, small buffers | frame buffers, canvases, our history |

So on this board, **touching pixels in PSRAM is the expensive part**, not
the arithmetic: reading one 160x240 crop takes ~16 ms. Clearing a 77 KB
image in one go most likely took ~15 ms without a pump, enough to overflow
the ring.

## Measure, don't guess

Two of our guesses about lost frames were wrong. What found the cause was
a timing panel on the screen, not reading more code:

| Line | What it counts |
| --- | --- |
| `begin` | ms per second in `begin_frame`. Large = capture restarts = dropped frames |
| `render`, `green` | our work per frame |
| `finish` | waiting for the next frame (= free time) |
| `dashboard` | drawing and sending the dashboard |
| `drop …` | restarts, counted by what ran just before |

Why on the screen and not in the log? Logging blocks for milliseconds, and
the camera code notes that a log line can itself cost a frame.

## Numbers to remember (this board, this app)

- The sensor gives **20 FPS** = 50 ms per frame.
- Showing a frame: ~28 ms. Reading it for green only: ~16 ms.
- Dashboard redraw: ~57 ms, at 2 Hz.
- Free time per frame with `RENDER_EVERY = 3`: ~26 ms. That is the budget
  for face detection.
