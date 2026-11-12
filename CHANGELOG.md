# Changelog

## Unreleased

The package takes devices-nv and embedded-hal-nv by path until 0.4.0 is published.

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
