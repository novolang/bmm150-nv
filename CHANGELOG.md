# Changelog

## 0.2.0 — 2026-11-24

The register reads and writes, and the bus set-up, are devices-nv's `RegPort` and `wiring_*` functions, and `Bmm150Port` is smaller by them.  The release depends on devices-nv 0.4.1.

### Breaking

- **`Bmm150Port` has the supertrait `RegPort` and no longer declares `read_regs` and `write_reg`.**  A type that implements `Bmm150Port` for a bus of its own moves those two methods, unchanged, into an `impl RegPort`.  *Migration:* with `use dev_bus.{ RegPort }`,

  ```novo
  impl RegPort for MyPort
      fn read_regs(self, reg: Int, n: Int, var dst: DevBytes) -> Bool [hw]
          ...
      fn write_reg(self, reg: Int, v: Int) -> Bool [hw]
          ...

  impl Bmm150Port for MyPort
      ...                      // the other methods stay where they are
  ```

  The package's own types are migrated.  A caller that holds one of them is unaffected.

### Deprecated

- `i2c_setup`, `spi_setup`, `irq_level_of`, `reg_read`, `reg_write`, `reg_update`: each is a one-line function over the method or the `wiring_*` function that replaces it, warns W1001 at a call with the replacement named, and is removed in the next release.  `Bmm150Port` itself is the driver's trait and stays.

### Changed

- The register helpers the module carried (`reg_read`, `reg_write`, `reg_update` over its port) are devices-nv's `RegPort` methods (`read_reg`, `write_reg_verified`, `update_reg_verified` and the others); the module calls them as `p.read_reg(reg)`.
- The bus set-up and the interrupt-level read are devices-nv's `wiring_i2c_setup`, `wiring_spi_setup` and `wiring_irq_level`, called with the part's rate and mode.
- Image size on nrf52840-dk, `.text` plus read-only data, before and after: 19,828 bytes before and 19,764 after on the read-out (nrf52-qemu: 7,952 and 7,952).

## 0.1.1 — 2026-11-12

The release depends on devices-nv 0.4.0 and embedded-hal-nv 0.4.0.

- `Magnetometer.mag_start()` starts one forced measurement and answers its time in
  microseconds, from the repetitions the part is set to; -1 when the part does not
  answer.  The module function is `mag_start_us`.
- `Magnetometer.mag_status()` reads INT STATUS's overflow flag: `MagOverflow` when it is
  set, `MagValid` otherwise, `MagNotReady` when the part does not answer.  The sample
  still carries `OVERFLOW_NT` for an axis the compensation cannot compute.  The module
  functions are `mag_status_read` and `status_of`.
- `mag_range` and `mag_set_range` keep the trait's bodies: the range is fixed per axis.

## 0.1.0

The first release: a driver for Bosch Sensortec's BMM150 three-axis geomagnetic
sensor on an I2C or an SPI bus.

- `Bmm150` (I2C) and `Bmm150Spi` (SPI), the types a board's device table builds,
  implementing devices-nv's `Magnetometer` and `Identifiable`, with their own methods
  for the mode, the rate, the presets, one forced measurement and the self-test.
- The register access as functions over `Bmm150Port`, a trait that a type holding a
  bus implements, and the framing of each bus as functions over `I2cBus` and `SpiBus`.
- Readings in nanotesla, from the raw values, the RHALL resistance and the factory
  trim data, with the SensorAPI's integer compensation and its overflow sentinel.
  The trim data is read once and kept as module state.
- The power states: the power control bit, the start-up waited by polling the chip id,
  the soft reset and a full reset by way of suspend.
- The normal, forced and sleep modes, the eight data rates, the repetitions and the
  four presets, and the forced time.
- The axes, the data-ready pin, the interrupt pin, the thresholds and the interrupt
  enables as configuration.
- Two functions for a host's scheduler that park the task for a forced measurement
  (`mag_measure`) or for the data-ready flag (`mag_next`), and `irq_arm` and
  `irq_disarm` for the DRDY pin through an `AsyncEdge` source.
- The normal self-test as a function that answers the axes that passed.
- The suite runs over a double of the part on both buses and covers every line of
  `src/`.

### Known

- Hardware verification: NOT VERIFIED.  No board carries a BMM150.
- The data sheet read is BST-BMM150-DS001-05, revision 1.4 (April 2020), the copy
  published by Bosch Sensortec.  The brief named BST-BMM150-DS001, which is the same
  document without its suffix.
- The repetitions on Z for the presets follow the data sheet's Table 26, which is
  about twice what the SensorAPI and Zephyr write.  NOT VERIFIED on a part.
- `MagnetometerAsync` is not implemented: a `@value` struct takes no `async fn` (SPEC
  section 14.5).  `AsyncDeviceReady` is not implemented either, since no board's GPIO
  implements `AsyncEdge`; `irq_arm` is the part that does not depend on a board.
- The device program builds for the nRF52840 DK, the nRF54LM20 DK and the
  FRDM-MCXN947, with images of 19,544, 20,544 and 21,512 bytes of flash.  A reply from
  a BMM150 over the Nordic buses is NOT VERIFIED until the driver rig.
- The advanced self-test, the SPI three-wire mode and the conversion of the threshold
  registers into microtesla are not implemented.
- The compensation computes in 64-bit integers where the SensorAPI wraps two
  intermediate values to 16 bits; the results differ only above about 2 000
  microtesla, beyond the sensor's range.
