# 07 — Sand in a box: IMU, particle physics, sound

App: `src/bin/01_2d_particle_physics_sim/` (see its `PLAN.md`).

## 1. Accelerometer vs gyroscope

The board's IMU chip (BMI270) has both:

| Sensor | Measures | Unit | Still board shows |
| --- | --- | --- | --- |
| Accelerometer | push on the chip, gravity included | m/s² | ~9.8 towards the ground |
| Gyroscope | how fast it rotates | °/s | ~0 |

The library also fuses them into `attitude` (roll, pitch, `down`). That
is smooth and ignores shakes. For sand we want the shakes too, so we use
the raw `acceleration_m_s2`.

## 2. Why the accelerometer is exactly "the grains' gravity"

Hold a box of sand still: the grains feel gravity. Jerk the box left: the
grains feel pushed right, as if gravity had tilted. What a grain feels is

```text
felt = gravity − acceleration of the box
```

and that is what the accelerometer reports (the library's sign convention
makes it point down at rest). So tilt **and** shake come for free.

## 3. From sensor axes to screen pixels

Every sensor has its own axes. The library turns them into a screen frame:
x out of the top edge, y to the right, z into the screen. Pixels have x to
the right and y **down**, so:

```text
pixel_x = +imu_y        pixel_y = −imu_x
```

Always check signs on the device first (step 0: an arrow must point the way
the board tilts). A wrong sign makes sand fall up.

## 4. Simulating motion

The simulation keeps a **state** (each grain's position and velocity) and
moves it forward in small time steps `dt` (a few ms):

```rust
v += a * dt;   // acceleration changes velocity
x += v * dt;   // velocity changes position
```

This is **Euler integration**. Updating `v` first ("semi-implicit") keeps
it stable. Use the real elapsed time for `dt`, capped, so a slow frame does
not make grains jump.

- **Damping:** `v *= 0.98` each step removes energy, like friction.
- **Bounce:** hitting a wall, `v = −v · r`. `r` (restitution) 0 = dead
  stop, 1 = perfect bounce. Sand ≈ 0.1–0.3.

## 5. Collisions with a grid

Checking every grain against every other one: 1,200 × 1,200 = 1.4 million
checks per step. Too slow. Instead, the box is a **grid of cells**, each
empty or holding one grain. A grain moving to a new cell looks up one cell:

- empty → move there;
- occupied → hop to a free neighbour cell that is **downhill** (the most
  downhill first). This hopping is what makes piles and slopes;
- nothing free → stop and bounce.

Speed is limited to under one cell per step, so a grain never jumps over
another. This is the "pixel dust" method used in many LED sand toys.

**Why "downhill" matters:** a move that only goes downhill always lowers
the energy, so everything must come to rest. No jitter at rest means real
silence.

**Sand vs water:** sand accepts only clearly downhill cells (within ~80°
of gravity), so it keeps a slope, like a real pile. Water accepts anything
downhill and also looks a few cells sideways for a lower spot, so its
surface ends up level (perpendicular to gravity, even when tilted).

**Test the rules off the device:** the same rules in 80 lines of Python
showed the first version locked up and water never levelled, before
anything was flashed.

## 6. Drawing from the grid

No image buffer: the display asks for one row of pixels at a time
(`render_scanlines`), and we compute it from the grid: cell occupied →
grain colour, else background. Cheap, and the whole box is redrawn every
frame (~31 ms for the full screen at 40 MHz SPI).

## 7. Making sound

The speaker plays **samples**: 16,000 numbers per second (`i16`, −32768 to
32767), two per frame (left, right). We push them into a queue that holds
64 ms; if we don't refill it in time, the sound breaks up.

- **White noise:** random samples, sounds like "ssss". A tiny generator:

  ```rust
  x ^= x << 13; x ^= x >> 17; x ^= x << 5;   // xorshift32
  ```

- **Low-pass filter:** `y += k · (noise − y)` with small `k` averages the
  noise, removing the high hiss → softer "shhh", closer to sand.
- **Envelope:** volume follows motion, but smoothed: rise fast (~20 ms),
  fall slower (~150 ms). Jumping the volume at once makes clicks.
- At rest the volume goes to 0: real silence.

## 8. Words to remember

| Word | Meaning |
| --- | --- |
| IMU | Inertial measurement unit: accelerometer + gyroscope (+ magnetometer) |
| State | Everything needed to compute the next step |
| `dt` | Time between two steps |
| Restitution | How much speed survives a bounce (0–1) |
| Damping | Losing a little speed every step |
| Sample rate | Audio numbers per second (16 kHz here) |
| Envelope | How volume changes over time |
