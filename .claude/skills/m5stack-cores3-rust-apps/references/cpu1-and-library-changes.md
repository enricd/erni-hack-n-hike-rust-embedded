# CPU1 tasks and changes to the shared library

## Who owns what

| Core | Runs |
| --- | --- |
| CPU0 | the app's `main`: its own embassy executor. The camera capture also runs here. |
| CPU1 | `src/board/cpu1.rs` starts one executor with the library tasks: ESP-NOW (2), audio (2), IMU, touch, light/proximity, backlight, camera control |

After `Board::init`, the **system I2C bus belongs to CPU1** (async driver
behind a `Mutex<NoopRawMutex, _>`, `src/board/i2c.rs`). CPU0 code cannot
talk to any I2C chip (power chip, IO expander, IMU, touch, camera settings)
directly anymore.

Data flows both ways: requests CPU0 → CPU1 (backlight level, camera lock,
network sends) and results CPU1 → CPU0 (touch events, IMU and light
samples, received messages).

## Pattern: a new request from the app to CPU1

Copy `src/capabilities/backlight/` (handle + `runtime.rs`). The camera
lock was built exactly like this (`src/capabilities/camera/mod.rs`,
`runtime.rs`, `gc0308.rs`).

```rust
// capability/mod.rs
struct Service { request: Signal<CriticalSectionRawMutex, bool> } // newest value wins
static SERVICE: Service = Service { request: Signal::new() };      // safe on both cores

impl Camera {                        // an inherent impl may live in another module of the crate
    pub fn set_auto_adjust(&mut self, enabled: bool) { SERVICE.request.signal(enabled); } // never waits
}

#[derive(Clone, Copy)]
pub(crate) struct Runtime { service: &'static Service }
impl Runtime { async fn next_request(self) -> bool { self.service.request.wait().await } }
pub(crate) fn runtime() -> Runtime { Runtime { service: &SERVICE } }

// capability/runtime.rs
pub(crate) fn spawn(spawner: &Spawner, bus: SystemI2cBus, runtime: Runtime) {
    spawner.spawn(control_task(bus, runtime).expect("task already spawned"));
}
#[embassy_executor::task]
async fn control_task(bus: SystemI2cBus, runtime: Runtime) {
    loop {
        let enabled = runtime.next_request().await;
        let result = { let mut i2c = bus.lock().await; chip::apply(&mut *i2c, enabled).await };
        if let Err(e) = result { warn!("update failed: {:?}", e); }
    }
}
```

Wiring (outside the capability folder, keep it tiny):
- `src/board/mod.rs`: create the runtime (only if the hardware answered,
  e.g. `camera.is_some().then(camera::runtime)`) and pass it in `Cpu1 { .. }`.
- `src/board/cpu1.rs`: a field on `Cpu1` and
  `if let Some(r) = cpu1.camera { camera::spawn(&spawner, system_bus, r); }`.

The library has `#![warn(missing_docs)]` and
`clippy::missing_docs_in_private_items`: document every item, private
fields and constants too.

## Registers and bit masks

- Sync driver at start-up: `Registers::new(i2c, ADDRESS)` with `read`,
  `write`, `write_all`, `update_bits(reg, mask, value)`.
- On CPU1: `AsyncRegisters` has `write` and `update_bits` (no `read`; use
  `i2c.write_read(ADDRESS, &[reg], &mut buf).await`).
- `update_bits` = read, `(old & !mask) | (value & mask)`, write: changes one
  switch, keeps the rest.
- After a write, **read back and log** the register: it proves the bus and
  the address work and shows the real value.
- GC0308 camera (address `0x21`, page 0 after start-up): auto exposure =
  `0xd2` bit 7, auto white balance = `0x22` bit 1 (bits from Espressif's
  esp32-camera driver; verified by read-back). It was programmed at 100 kHz
  at start-up, and 400 kHz on the shared bus works too.

## Rules for library changes

- Ask the user before changing anything outside their app folder.
- Backward compatible: add functions, never change start-up behaviour; an
  app that does not call the new function must behave exactly as before.
- Then build the whole workspace and check the demo:
  `cargo clippy --release -- -D warnings` and `cargo dist --bin demo`.
