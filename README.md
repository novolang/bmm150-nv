# bmm150-nv

A driver for Bosch Sensortec's BMM150, a three-axis geomagnetic sensor with an
I2C and an SPI interface, specified in the data sheet "BMM150: Geomagnetic
Sensor", BST-BMM150-DS001-05, revision 1.4, April 2020, and in the
compensation of Bosch Sensortec's BMM150_SensorAPI.  The driver implements the
`Magnetometer` and `Identifiable` traits of
[devices-nv](https://novo-lang.org/packages/devices-nv) and reads the field in
nanotesla.

## Install

```sh
novo pkg add bmm150-nv
```

## Example

A device program reads the part over I2C.  The breakout's SDA and SCL go to the
board's I2C pins, its CSB and SDO pins to ground, which puts the part at 0x10,
and its PS pin to the supply, which selects I2C.  The program brings the part
up and prints one line per sample over RTT.  The bring-up powers the part,
waits the 3 ms start-up by polling the chip id (the driver has no timer),
resets it, reads its trim data, and starts normal mode at the regular preset
and 10 Hz.  The Earth's field is between 25 000 and 65 000 nanotesla.

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio }
use dev_bus
use dev_mag.{ MagNanoT, Magnetometer }
use bmm150
use bmm150.{ Bmm150 }

// The part's address with CSB and SDO low.
const ADDR = 0x10

// A label and a number, written without building a string.
fn field(label: Str, v: Int) [hw]
    hal.uart.write(label)
    hal.uart.write(str.from_int(v))

@tier(embedded)
fn main() [hw, mutate]
    let part = Bmm150.attach(BoardI2c { id: 0 }, BoardGpio { id: 0 }, dev_bus.wiring_i2c(0, ADDR))
    if not part.begin()
        hal.uart.write("bmm150: no answer\n")
        return
    loop
        match part.mag_read_new()
            Some(m) =>
                hal.uart.write("bmm150")
                field(" x=", m.x)
                field(" y=", m.y)
                field(" z=", m.z)
                hal.uart.write("\n")
            None    =>
                hal.timer.delay_ms(5)
```

The program is `tests/device_read.nv`, and `tests/device_build.sh` builds it for
the boards below.  The nRF52840 DK's I2C bus is TWIM0 on SDA P0.26 and SCL P0.27,
and the nRF54LM20 DK's is TWIM22 on SDA P1.03 and SCL P1.04 (`docs/hal/README.md`
in the novo-lang repository, "The Nordic DKs' buses").

```sh
novo build --target=nrf52840-dk tests/device_read.nv
novo build --target=nrf54lm20-dk tests/device_read.nv
novo build --target=frdm-mcxn947 tests/device_read.nv
```

**Hardware: NOT VERIFIED.**  No board carries a BMM150 yet.  The program builds
for the three boards, the image for the emulated board runs under QEMU and
reports that the part does not answer, and the driver's functions run over a
double of the part that follows the data sheet's register map, power states and
bus protocols.  A reading from a part on a board has not been taken, and a reply
from a BMM150 over the Nordic buses is NOT VERIFIED until the driver rig exists.

### The tier, the boards and the footprint

The package is checked at the embedded tier (`[tier]` in `novo.toml`), so every
function in it is also valid at `rt` and `system`; no migration knob is on.
Each row is the memory report of `novo build` for the program above, with the
driver's reading path in the image.

| Board | Flash | Of which the program | `.data` and `.bss` | Module state | Static | Arena provable |
|---|---|---|---|---|---|---|
| nRF52840 DK | 19,544 bytes | 8,704 bytes | 3,396 bytes | 136 bytes | 510 bytes | 736 bytes |
| nRF54LM20 DK | 20,544 bytes | 8,894 bytes | 3,380 bytes | 136 bytes | 510 bytes | 736 bytes |
| FRDM-MCXN947 | 21,512 bytes | 10,396 bytes | 2,892 bytes | 136 bytes | not recorded | not recorded |

The rest of each image is the runtime, the compiler's support library and
tables.  Every buffer of the driver is a fixed-size `Vec` in the caller's frame,
`MagNanoT` and `Bmm150Trim` are `@value` structs, and the only state outside a
frame is the module's trim data, 11 integers in a `Vec[Int; 16]`.  "Arena
provable" is the compiler's count of allocations at initialisation.

### The effects

Every function of the driver that reaches the bus declares `[hw]`.  The
functions that read or keep the trim data also declare `[mutate]`, because the
trim data is module state, and so does the program above.  The driver performs no
input or output of its own.  The bus functions of devices-nv build one short frame
per transaction inside a `system` block and release it when the call returns.
The two awaiting functions, `mag_measure` and `mag_next`, declare `[hw, async,
mutate]` and are `@tier(rt)`.

## What it is

A magnetometer measures the magnetic field along three axes.  The Earth's field
at the surface is between about 25 and 65 microtesla, and a magnetometer that
reads it tells a program which way the board faces, as a compass does.  Iron and
current-carrying wires near the part add their own field.

The BMM150 measures X and Y as 13-bit values and Z as a 15-bit value, within
±1300 microtesla on X and Y and ±2500 on Z (Table 1).  The raw values are not
calibrated: the part also measures the resistance of its Hall plate (RHALL, 14
bits), and the host compensates each axis with it and with factory trim data
that the part holds in 11 registers (section 4.3.2).  The package does that in
integers, with the formulas of the SensorAPI, and answers nanotesla.

The part has four power states (section 4.2).

| State | How it is entered | What works |
|---|---|---|
| Suspend | after power-on, or by clearing the power control bit (POWER bit 0) | only POWER; a read of the chip id answers 0x00 on I2C and floats on SPI |
| Sleep | by setting the power control bit, after 3 ms; or from an active mode by writing opmode 11 | every register |
| Normal | opmode 00 | periodic measurements at the data rate |
| Forced | opmode 01 | one measurement, then the opmode reads sleep again |

The data rate in normal mode is 2, 6, 8, 10, 15, 20, 25 or 30 Hz (Table 17).  How
long and how noisy a measurement is depends on the repetitions on X and Y and on
Z, which Bosch groups into four presets (Table 3).

| Preset | X and Y | Z | Rate | Noise on X, Y, Z |
|---|---|---|---|---|
| Low power | 3 | 3 | 10 Hz | 1.0, 1.0, 1.4 microtesla |
| Regular | 9 | 15 | 10 Hz | 0.6, 0.6, 0.6 microtesla |
| Enhanced regular | 15 | 27 | 10 Hz | 0.5, 0.5, 0.5 microtesla |
| High accuracy | 47 | 83 | 20 Hz | 0.3, 0.3, 0.3 microtesla |

A forced measurement takes about 145 us for each repetition on X and Y, 500 us
for each on Z, and 980 us (section 4.2.4); `forced_time_us` gives it.  New data
sets DRDY in bit 0 of RHALL's low byte, and the part can drive the same event
onto its DRDY pin (`drdy_pin_write`).

The part has two bus interfaces on the same pins, chosen by the PS pin.

| | I2C | SPI |
|---|---|---|
| Address | 0x10 to 0x13, from CSB and SDO | chip select low |
| Several registers in one read | automatic | keep clocking |
| Read | register, repeated start, data | command byte with bit 7 set (RW), then data |
| Clock | 400 kHz | 4 MHz, mode 3 (modes 0 and 3 to 10 MHz) |

| CSB | SDO | I2C address |
|---|---|---|
| ground | ground | 0x10 |
| ground | supply | 0x11 |
| supply | ground | 0x12 |
| supply | supply | 0x13 |

## What the package contains

| Module | What is in it |
|---|---|
| `bmm150` | The types `Bmm150` (I2C) and `Bmm150Spi` (SPI), `BmmMode`, `BmmPreset`, `Bmm150Trim`, the trait `Bmm150Port`, the bus framing, the register access, the trim data and the compensation, the modes, the presets, the forced measurement, the self-test, the awaiting functions and the bring-up |

## How to choose an entry point

A program on a board uses `Bmm150` for a part on I2C and `Bmm150Spi` for a part on
SPI.  Each is built by `attach` with the board's bus, its GPIO and the wiring, as a
board's device table builds it.  Both implement `Magnetometer` and `Identifiable`.

| Method | What it does |
|---|---|
| `attach(bus, pins, w)` | the part at `w.addr` (I2C) or with the chip select `w.cs` (SPI); nothing is sent |
| `begin()` | brings the part up, see below |
| `mag_read()`, `mag_ready()`, `mag_read_new()` | `Magnetometer`: the latest sample in nanotesla, the data-ready flag, a sample not read before |
| `mag_rate_hz()` | the rate in normal mode; 0 in forced and sleep mode |
| `device_id()` | `Identifiable`: CHIP_ID, 0x32, which a part in suspend mode does not answer |
| `mode()`, `set_mode(m)` | normal, forced or sleep; forced starts one measurement |
| `set_rate_hz(hz)`, `set_preset(p)` | the slowest rate at or above `hz`, and the repetitions and rate of a preset |
| `measure()` | one forced measurement: starts it, polls the ready flag, answers the sample |
| `self_test()` | the normal self-test, answering the axes that passed as a mask |
| `reg(r)`, `set_reg(r, v)` | one register, and a write that is read back |

The rest are functions of the module that take the port, so they work for both
types: `bmm150.rep_write(part, 9, 15)`, `bmm150.axes_write(part, true, true,
false)`, `bmm150.drdy_pin_write(part, true, true)`, `bmm150.int_pin_write`,
`bmm150.thresholds_write`, `bmm150.int_enable_write`, `bmm150.int_status`,
`bmm150.soft_reset`, `bmm150.suspend` and `bmm150.power_on`.

The compensation is pure and public: `comp_xy`, `comp_z` and `compensate` take the
trim data and the raw values, so a program that reads the part itself can use
them.  `trim_load(part)` reads the 11 trim values, `trim_get()` answers them and
`trim_clear()` forgets them.

A program that holds its own bus, or a test with a double of the part, implements
`Bmm150Port` over its bus and calls the module's functions, which take the port:
`bmm150.sample_read(port)`, `bmm150.start(port)`.  The framing of each bus is
`i2c_read_into`, `i2c_write`, `spi_read_into` and `spi_write`, generic over `I2cBus`
and `SpiBus`.

### Waiting for a sample

`MagnetometerAsync` cannot be implemented by `Bmm150`: a `@value` struct takes no
`async fn` (SPEC section 14.5).  The package gives the same waits as functions for a
host's scheduler, each taking a `TimerAsync`:

- `mag_measure(part, timer)` starts one forced measurement, parks the task for the
  time the repetitions need (9 785 us at the regular preset), and answers the
  sample.  The part measures in the background while the task is parked.
- `mag_next(part, timer, period_us, tries)` parks the task between polls of the
  data-ready flag in normal mode.

A program at `@tier(embedded)` waits on the DRDY pin instead.  `irq_arm(edge, w, f)`
and `irq_disarm(edge, w)` arm and release a rising-edge wait on the pin of the wiring
through any `AsyncEdge` source, after `drdy_pin_write`.  No board's GPIO implements
`AsyncEdge` yet, so this path is NOT VERIFIED on a board.

## The rules a user needs

1. The chip id, and every other register but POWER, answers only after the power
   control bit is set and 3 ms have passed.  The driver has no timer, so `begin`
   polls the chip id; a program that reads the id before that finds no part.
2. The trim data is module state: the driver holds one BMM150 per program.
   `sample_read` loads it when it is missing, and `suspend` forgets it, since the
   part loses its registers.
3. A reading is in nanotesla with 62.5 nT of resolution, which is the 1/16
   microtesla that the formula carries.  An axis that the compensation cannot
   compute reads `OVERFLOW_NT`, -32 768 000 nanotesla, which is the SensorAPI's
   -32768 microtesla: the ADC's overflow marker (-4096 on X and Y, -16384 on Z), a
   zero in the trim data or in the resistance, or a zero divisor.  `is_overflow`
   tests for it.
4. The ready flag lives in a data register, so reading it clears it.
   `mag_ready` followed by `mag_read` is the sequence for a new sample.
5. Forced mode ends by itself and the opmode reads sleep again, so a write of forced
   mode is not read back.  `measure_forced` and `mag_measure` poll the ready flag.
6. If an axis is disabled with `axes_write`, its last value stays in the registers.
   With Z disabled the part reports no resistance, and the compensation of every
   axis then answers `OVERFLOW_NT`.
7. The soft reset keeps the trim data and the power control bit, and the part
   ignores it in suspend mode.  A full reset goes by way of suspend, which `begin`
   does.
8. The threshold registers and the interrupt enables are written in the part's own
   units and bits (the enables are active low).  The data sheet's text gives no weight
   for a threshold step, so none is converted.
9. `device_id` answers `None` for a bus that no part drives or a part in suspend: an
   SPI bus reads 0xFF.

### An open question

The data sheet gives the Z repetitions as `nZ = 1 + REPZ` (Table 26) and the presets as
3, 15, 27 and 83 (Table 3), which are REPZ 2, 14, 26 and 82.  The SensorAPI writes 0x01,
0x07, 0x0D and 0x29 for the presets, and Zephyr's driver computes the Z register as it
does X and Y, `(n - 1) / 2`: about half.  The driver follows the data sheet.  Which is
right for the silicon is NOT VERIFIED; the effect is the noise and the time of a
measurement, not its correctness.

## What is not included

- The advanced self-test (section 4.4.2): it needs thresholds from an application note
  that could not be fetched.  The normal self-test is implemented.
- The SPI three-wire mode (POWER's SPI3en): the 4-wire mode is implemented.
- The conversion of the threshold interrupts' registers into microtesla; they are raw.
- The Hall-plate resistance as a reading of its own.  It is an input of the
  compensation, not a temperature.
- The other parts that hold the same magnetometer core, which are separate data sheets
  with other addresses and register maps: the BMM350, the BMM156, the BMC150 and the
  BMX055.  This driver is for the BMM150.
- Any module that wires CSB and SDO to something other than ground or the supply.

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the traits this driver
  implements, the wiring record and the register helpers it uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names the `I2cBus`,
  `SpiBus`, `GpioOut`, `TimerAsync` and `AsyncEdge` traits and the board handles.
- [lis3mdl-nv](https://novo-lang.org/packages/lis3mdl-nv) drives the LIS3MDL, a
  magnetometer of ST with factory-compensated output.
- [qmc5883l-nv](https://novo-lang.org/packages/qmc5883l-nv) drives the QMC5883L, an I2C
  magnetometer of QST.

## Tests

`novo test` runs the module over a double of the part that answers on an I2C bus and on
an SPI bus: the register and the command byte, the suspend mode in which only POWER
answers, the start-up time, the data registers at their widths, the ready flag that a
read clears, the forced mode that ends by itself, the self-test, the soft reset, the
bring-up and its refusals, the two awaiting functions over a timer double, and the two
types over the host's board handles.  The log of bytes sent is compared for each entry
point.  The compensation is checked against a vector derived by hand from the formula,
with every intermediate value in the comment of the test, and against the overflow
cases.  The suite covers every line of `src/`.

**Hardware: NOT VERIFIED.**  See the example section.

The sources read for this driver, for their fixes and not ported, are the data sheet
above, Bosch Sensortec's BMM150_SensorAPI (`bmm150.c`) and Zephyr's
`drivers/sensor/bosch/bmm150`.  No Rust driver for the part was found.  Each fix kept
is named in `src/bmm150.nv` where it is kept:

- the chip id is not readable until the power control bit is set and the start-up has
  passed, and a soft reset is ignored in suspend mode, so `begin` powers the part
  first; it then goes by way of suspend for a full reset, as Zephyr does;
- the SensorAPI's compensation divides by a value that can be zero for some trim data;
  the driver answers the overflow sentinel instead;
- a forced mode write read back races with the end of the measurement and fails on a
  short one; it is not read back;
- the SensorAPI answers whole microtesla; the driver divides by 16 after scaling to
  nanotesla, so the 1/16 microtesla of the formula is kept.

## Licence

Apache-2.0.
