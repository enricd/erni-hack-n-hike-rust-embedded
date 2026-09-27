# 04 — Two cores, shared buses and sensor registers

What happened in step 3, when we locked the camera's auto exposure.

## Two CPU cores, two executors

The ESP32-S3 has two cores. This firmware uses them like this:

| Core | Runs | Scheduler |
| --- | --- | --- |
| CPU0 | our app's `main` loop (camera, display, maths) | its own embassy executor |
| CPU1 | the library's tasks: radio, audio, IMU, touch, light sensor, backlight, and now the camera control task | a second embassy executor |

Each executor is cooperative (see [01](01_embedded_rust.md)): tasks switch
only at `.await`. The two cores run truly **at the same time**, so data
they share needs protection.

## Talking across cores: `Signal`

```rust
static SERVICE: Service = Service { request: Signal::new() };

// CPU0, our app: never waits
camera.set_auto_adjust(false);          // → SERVICE.request.signal(false)

// CPU1, the camera task: sleeps until a value arrives
let enabled = runtime.next_request().await;
```

- A `Signal` holds **at most one** value. A new value replaces an old one
  that nobody took yet: only the newest request matters.
- It is a `static` that both cores use. That is safe because it is guarded
  by a **critical section** (`CriticalSectionRawMutex`): a lock that works
  across cores and interrupts.
- The backlight uses exactly the same pattern. We copied it.

## One bus, many chips: I2C

I2C is a 2-wire bus (clock SCL, data SDA). Many chips share it; each has a
7-bit **address** (camera `0x21`, power chip `0x34`, ...). Most chips expose
**registers**: numbered bytes that hold settings.

- Write: send `[register, value]` to the address.
- Read: send `[register]`, then read one byte back.

After start-up, the bus belongs to CPU1, behind a `Mutex`. A task
`lock().await`s it for one transaction, then releases it. Our camera task
does the same, so it never collides with touch or the IMU using the bus.

The camera was programmed at 100 kHz at start-up; the shared bus runs at
400 kHz. We tried 400 kHz and read the registers back: it works.

## Bits and masks

A register holds 8 on/off switches. To change **one** switch without
touching the others: **read, modify, write** (`update_bits`).

```text
AEC register 0xd2 at start-up:  0x90 = 1001 0000
                   bit 7 mask:  0x80 = 1000 0000
locked = value & !mask:         0x10 = 0001 0000   (bit 7 cleared, others kept)
on     = value |  mask:         0x90 = 1001 0000
```

| Operator | Does | Example |
| --- | --- | --- |
| `a & b` | keep bits set in both | `0x57 & 0x02 = 0x02` → bit 1 is on |
| `a \| b` | set bits | `0x55 \| 0x02 = 0x57` |
| `!a` | flip all bits | `!0x02 = 0xfd` |
| `a >> n`, `a << n` | shift | `(pixel >> 5) & 0x3f` = green of RGB565 |

Hex (`0x..`) is used because each hex digit is exactly 4 bits.

## What AE and AWB do

- **Auto exposure (AEC)**: the sensor changes exposure time and gain so the
  image is not too dark or too bright.
- **Auto white balance (AWB)**: it changes the red/green/blue gains so white
  looks white under any light.

Both are great for photos and bad for us: each adjustment changes the
green value by many levels, while the pulse changes it by less than one.
Locking them freezes the last settings.

## Backward compatible

The change in `src/capabilities/camera/` adds a function and a task, and
changes nothing at start-up. An app that never calls `set_auto_adjust`
cannot tell the difference. That is why it could go into a shared library
that other projects use.
