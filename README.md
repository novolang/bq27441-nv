# bq27441-nv

A driver for TI's BQ27441-G1, a system-side fuel gauge for a single
lithium cell with an I2C interface, specified in TI's data sheet
"bq27441-G1 System-Side Impedance Track Fuel Gauge", SLUSBH1C, November
2013, revised December 2014 (the brief of this package names it SLUSBH2),
and in TI's "bq27441-G1 Technical Reference", SLUUAC9A, December 2013,
revised May 2015.  The driver implements the battery side of the
`PowerManager` trait and the `Identifiable` trait of
[devices-nv](https://novo-lang.org/packages/devices-nv).  It also writes the
cell's design capacity, design energy, terminate voltage, taper rate and
OpConfig into the gauge's data memory, which the gauge needs for a cell
other than its default.

## Install

```sh
novo pkg add bq27441-nv
```

## Example

A device program watches a cell over I2C.  The breakout's SDA and SCL go
to the board's I2C pins and the gauge answers at 0x55.  The cell is 1200
mAh at 3.7 V.  The program waits 300 ms for the gauge to start, checks
DEVICE_TYPE, writes the cell's parameters to the gauge when it reports a
power-on reset, and once a second prints one line over RTT.  The current is
positive while the cell charges.

```novo norun:needs-pkg
use embedded_hal.{ BoardI2c, BoardGpio }
use dev_bus
use bq27441
use bq27441.{ Bq27441 }

// The gauge's address.
const ADDR = 0x55

// A label and a number, written without building a string.
fn field(label: Str, v: Int) [hw]
    hal.uart.write(label)
    hal.uart.write(str.from_int(v))

@tier(embedded)
fn main() [hw]
    let gauge = Bq27441.attach(BoardI2c { id: 0 }, BoardGpio { id: 0 }, dev_bus.wiring_i2c(0, ADDR))
    hal.timer.delay_ms(300)
    if not gauge.begin()
        hal.uart.write("bq27441: no answer\n")
        return
    if gauge.needs_config() == Some(true)
        let wrote = gauge.configure(bq27441.config_for(1200, 4440))
        if wrote < 0
            field("bq27441: configuration failed ", wrote)
            hal.uart.write("\n")
    loop
        match gauge.battery()
            Some(b) =>
                hal.uart.write("bq27441")
                field(" mV=", b.voltage.mv)
                field(" mA=", b.current.ma)
                field(" pct=", b.percent ?? -1)
                match gauge.battery_temperature()
                    Some(t) => field(" mC=", t.milli_c)
                    None    => hal.uart.write(" mC=?")
                hal.uart.write("\n")
            None =>
                hal.uart.write("bq27441: no battery\n")
        hal.timer.delay_ms(1000)
```

The program is `tests/device_read.nv`.  Build it for a board with:

```sh
novo build --target=nrf52840-dk tests/device_read.nv
novo build --target=nrf54lm20-dk tests/device_read.nv
novo build --target=frdm-mcxn947 tests/device_read.nv
```

| Board | Flash | Of which the program | RAM at start |
|---|---|---|---|
| nRF52840 DK | 18,648 bytes | 7,818 bytes | 3,252 bytes |
| nRF54LM20 DK | 19,632 bytes | 7,988 bytes | 3,236 bytes |
| FRDM-MCXN947 | 21,664 bytes | 10,554 bytes | 2,748 bytes |

The Nordic DKs drive I2C with a TWIM (the nRF52840 DK's TWIM0 on SDA P0.26
and SCL P0.27, the nRF54LM20 DK's TWIM22 on SDA P1.03 and SCL P1.04), on the
pins `docs/hal/README.md` names under "The Nordic DKs' buses", so the
images carry the driver's reading path.  The driver's own code allocates
nothing: every buffer is a fixed-size `Vec` in the caller's frame.

**Hardware: NOT VERIFIED.**  No board carries a BQ27441 yet.  The program
builds, runs under QEMU to its refusal (the emulated board has no I2C
device), and the driver's functions run over a double of the gauge that
follows the register map, the data memory and the Control subcommands of
the two documents above.  A reply from a gauge over a board's bus has not
been taken.

### The effects

Every function of the driver that reaches the bus declares `[hw]`, and
nothing else.  The conversions and the checksums are pure.  The driver
performs no input or output of its own and allocates nothing of its own.
The bus functions of devices-nv build one short frame per transaction
inside a `system` block and release it when the call returns.  The module
checks at every tier, `embedded` included; the exceptions are the
host-scheduler async impl at the end of the module, which carries
`@tier(app)`, and `alert_arm`, which takes the effect of the pin-edge
source it is handed.  The waits are the port's: the board's type waits with
its timer, `[hw]`.

## What it is

A fuel gauge reports how full a cell is.  The BQ27441-G1 measures the
cell's voltage and temperature, and the current through a sense resistor of
10 milliohm (typical) between its SRN and SRP pins.  TI's Impedance Track
algorithm turns them into the state of charge in percent and the remaining
and full-charge capacity in mAh.  It lives on the system board, powered
from the cell through an internal regulator.

| Reading | Command | Unit |
|---|---|---|
| Temperature | 0x02 | 0.1 K |
| Voltage | 0x04 | mV |
| Flags | 0x06 | bits, see below |
| NominalAvailableCapacity, FullAvailableCapacity | 0x08, 0x0A | mAh |
| RemainingCapacity, FullChargeCapacity | 0x0C, 0x0E | mAh |
| AverageCurrent, StandbyCurrent, MaxLoadCurrent | 0x10, 0x12, 0x14 | mA, signed |
| AveragePower | 0x18 | mW, signed |
| StateOfCharge | 0x1C | % |
| InternalTemperature | 0x1E | 0.1 K |
| StateOfHealth | 0x20 | % in the low byte, a status in the high byte |

Each is a 16-bit word, low byte first.  A current is positive into the cell
while it charges and negative while it discharges.  The part updates its
readings once a second in NORMAL mode.

Flags holds DSG (bit 0), SOCF (1), SOC1 (2), BAT_DET (3), CFGUPMODE (4),
ITPOR (5) and OCVTAKEN (7) in the low byte, and CHG (0), FC (1), UT (6) and
OT (7) in the high byte.  BAT_DET says a battery is detected: "gauge
predictions are not valid unless it is set", so `battery()` answers `None`
while it is clear.  ITPOR says a power-on reset or a RESET has put the RAM
configuration back at the ROM defaults, and the host should write the
cell's parameters again.

The part is sealed or unsealed.  A sealed part answers the standard
commands and the Control subcommands marked "Yes" in the manual's table.  An
unsealed part also answers the data memory commands.  The default after a
reset is unsealed.  The key that unseals a sealed part is 0x8000, written
twice to Control with no other write between the two.

The bus is I2C at 400 kHz, address 0x55.  The data sheet asks for a bus-free
time of at least 66 microseconds between all packets addressed to the part,
and, above 100 kHz, for individual 1-byte writes.  The driver writes a
Control subcommand as two 1-byte packets, register 0x00 then 0x01; the
board's type waits 66 microseconds after every packet, through the bus.

## What the package contains

| Module | What is in it |
|---|---|
| `bq27441` | The type `Bq27441`, the trait `Bq27441Port`, `Bq27441Config` and the host-scheduler `Bq27441Host`; the bus framing, the readings, the Control subcommands, the seal, the data memory with its block checksum and the configuration update |

## How to choose an entry point

A program on a board uses `Bq27441`.  It is built by `attach` with the
board's bus, its GPIO and the wiring, as a board's device table builds it.
The waits go through the bus (`I2cBus.delay_us` and `delay_ms`), so there is
no timer slot.  It implements `PowerManager` (the battery side) and
`Identifiable`.

| Method | What it does |
|---|---|
| `attach(bus, pins, w)` | the gauge at `w.addr`; nothing is sent |
| `begin()` | makes the bus ready and checks DEVICE_TYPE; the program waits 250 ms or more after power-up first |
| `needs_config()` | whether ITPOR is set |
| `configure(cfg)` | the configuration update, see below |
| `battery()` | `PowerManager`: the voltage, the average current, the phase and the percent; `None` while no battery is detected |
| `battery_capacity()` | `PowerManager`: the remaining and the full-charge capacity in mAh, two reads |
| `battery_temperature()` | `PowerManager`: the cell temperature in millidegrees Celsius |
| `battery_health()` | `PowerManager`: cold or overheat from Flags' UT and OT, good between, unknown without a battery |
| `voltage()`, `current()`, `soc()`, `flags()` | one reading each |
| `device_id()` | `Identifiable`: DEVICE_TYPE, 0x0421, with the firmware version as `rev` |
| `seal()`, `unseal()` | the access mode |
| `gpout()` | the level of the GPOUT line, -1 for a wiring with none |

The gauge supplies no rail: `rail_count` answers 0, and the other rail
methods answer as for a rail the part does not have.  The remaining and the
full-charge capacity are `PowerManager.battery_capacity`; the uncompensated
capacities stay the module's own functions, `nominal_available_mah` and
`full_available_mah`, beside `remaining_mah` and `full_charge_mah`.  The
module's functions take a `Bq27441Port`, the trait a type that holds a bus
implements, so a test runs them over a double.

`battery()` reads Flags, Voltage, AverageCurrent and StateOfCharge, four
transactions.  The phase is not the gauge's to know: a full cell (FC) is
`ChargeDone`, a current into the cell is `ChargeConstantCurrent`, and any
other state is `ChargeIdle`.  The data sheet limits how often a host may
issue standard commands (its section 9.5.4.3); the sentence is garbled in
the copy read, and the driver does not limit them.

### The configuration update

A gauge that has reset runs on the ROM defaults, which are for a 1000 mAh
cell.  `config_update(port, cfg, exit)` writes the cell's parameters, in the
sequence of the technical reference's section 3.1:

1. read CONTROL_STATUS; when the SS bit says sealed, unseal;
2. send SET_CFGUPDATE and poll Flags until CFGUPMODE is set, every 25 ms at
   most 100 times (the manual says up to 1 second);
3. for each field of `Bq27441Config` that differs from the part's, select
   its block (BlockDataControl 0x00, DataClass, DataBlock), wait 5 ms, read
   the checksum and the 32 bytes and check the checksum against the bytes,
   write the two new bytes, write the new checksum to 0x60, wait 10 ms,
   select the block again and check that the checksum and the bytes read
   back are the ones written;
4. leave the mode with SOFT_RESET (an OCV reading and a resimulation),
   EXIT_RESIM or EXIT_CFGUPDATE and poll Flags until CFGUPMODE is clear;
5. seal the part again if it was sealed.

The fields are the State subclass's Design Capacity (offset 10, mAh),
Design Energy (12, mWh), Terminate Voltage (16, mV) and Taper Rate (27, in
tenths of an hour rate: the capacity times 10, divided by the taper current
in mA), and OpConfig of the Registers subclass (offset 0) when
`cfg.opconfig` is `Some`.  A data memory field is stored most significant
byte first; the standard commands are least significant first.  The answer
is the number of fields written, 0 to 5, or a negative code: -1 the gauge
did not answer, -2 CFGUPMODE did not set, -3 a block did not match its
checksum, -4 a checksum read back differently, -5 CFGUPMODE did not clear.
After an error inside the mode the driver asks the gauge to leave it, so the
gauge is not left without gauging for the 240 seconds of its time-out.

The new checksum of a block is 255 minus the low 8 bits of the sum of the
32 bytes.  The manual gives a quick method, which `checksum_after` is: with
a block whose checksum is 0xE8 and a Design Capacity of 0x03E8 (1000 mAh)
changed to 0x04B0 (1200 mAh), x = 255 - 0xE8 - 0x03 - 0xE8 = -212, which is
44 modulo 256; 44 + 0x04 + 0xB0 = 224; and 255 - 224 = 31 = 0x1F, the value
the manual writes to 0x60.

### OpConfig

OpConfig is the gauge's option word: BIE (battery insertion by the BIN pin),
BI_PU_EN, GPIOPOL (the polarity of GPOUT), SLEEP, RMFCC, BATLOWEN (GPOUT
shows BAT_LOW instead of the state-of-charge pulse) and TEMPS (the host
writes the temperature).  The constants are `OPC_*`.  `opconfig_read` reads
it, sealed or not, and `config_update` writes it when the configuration
carries one.  With BIE clear the host tells the gauge a cell is present with
`battery_present(port, true)` (BAT_INSERT).

## Async forms

The gauge's background work is its initialization, which "could take up to
10 seconds" after a power-up or a wake from HIBERNATE, and the alerts on
GPOUT.

On a device, `alert_arm(edge, w, f)` arms the GPOUT pin on a pin-edge
source (`AsyncEdge` of embedded-hal-nv) for the falling edge, the gauge's
active-low default, and resolves the Future `f` that the caller awaits with
`core.rt.future_await`; the caller then reads Flags.  `alert_disarm` takes
the arm back.  No board's GPIO implements `AsyncEdge` today, so the source is
a program's own interrupt glue; the suite runs the function over a double.

On the host scheduler, `host_of(port)` answers a `Bq27441Host`, which
implements `PowerManagerAsync`: `battery_measure` parks the task, 100 ms at
a time, until INITCOMP is set in CONTROL_STATUS (at most 100 times), and
then answers the battery.  The struct is not a `@value` struct, since an
`async fn` takes none, and its impl is `@tier(app)`, since the embedded tier
refuses an `async fn`.

## The rules a user needs

1. The gauge needs 250 ms (typical) after power-up before it answers.  The
   program waits, and `begin` does not.
2. DEVICE_TYPE is 0x0421 for the -G1A, the -G1B and the -G1; `chem_id`
   (0x0128 for the -G1A, 0x0312 for the -G1B) and `dm_code` tell them apart.
3. Check `needs_config` after a start.  A set ITPOR means the configuration
   is the default of the ROM, and the percentages are for another cell.
4. The configuration update leaves the gauge unsealed or sealed as it found
   it.  A program that leaves it unsealed has the data memory open to any
   write; `seal` closes it.
5. The remaining capacity is compensated for load and temperature;
   `nominal_available_mah` is the version for a light load.
6. The temperature is the internal sensor's unless OpConfig's TEMPS is set.
   It is in tenths of a kelvin: `temp_milli_c` gives millidegrees Celsius.
7. A configuration update takes a second or more, because of the waits; the
   caller should not run it from a task that cannot wait.

## What is not included

- The calibration subclasses, the Ra table and the learning cycle's
  parameters (Update Status and Qmax): `dm_read_word` and `dm_write_word`
  reach any 16-bit field in a class and block, and the package names no more
  of them.
- The 8-bit and 32-bit data memory types.
- The SHUTDOWN exit by the GPOUT pin: the board raises the pin for 200
  microseconds.
- The OCV table and the BQ27441-G2 and BQ27427 variants, whose offsets
  differ (Zephyr's `bq27427_regs`).
- A rate limit for the standard commands.  NOT IMPLEMENTED; see the note
  under "How to choose an entry point".

## Related packages

- [devices-nv](https://novo-lang.org/packages/devices-nv) names the traits
  this driver implements, the wiring record and the register helpers it
  uses.
- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names
  the `I2cBus` (which also waits), `GpioOut` and `AsyncEdge` traits and the
  board handles.
- max17048-nv drives Analog Devices' MAX17048, a fuel gauge without a sense
  resistor, on the same trait; axp2101-nv and bq25895-nv drive a PMIC and a
  charger on the charger side of it.

## Tests

`novo test` runs the module over a double of the gauge that answers on an
I2C bus with the register file, the Control subcommands and the seal, the
unseal key that must come twice in a row, the Flags bits, and a data memory
of 32-byte blocks with a checksum that the gauge verifies before it takes
a write.  It checks the framing of every entry point, the conversions, the
readings and the refusals when the gauge does not answer, the phase, the
mode subcommands, the data memory fields in both directions, the
configuration update through each of its steps and each failure inside it,
the awaitable line over a pin-edge double, the host-scheduler form and the
type over the host's board handles.  The suite covers every line of `src/`.

The block checksum is checked against the manual's example, derived above,
and against the sum of the changed block for six synthetic blocks, so the
quick method and the full one agree.  `tests/coverage.sh` merges the suite's
coverage over `src/`.

Two mutations were checked and the suite turned red: writing the unseal
key once, and writing the two bytes of a data memory field least significant
byte first.

**Hardware: NOT VERIFIED.**  See the example section.

The sources read for this driver, for their fixes and not ported, are the
two documents above, Zephyr's `drivers/sensor/ti/bq274xx` and the `bq27441`
crate (leftger/bq27441, 0.3.0).  Embassy and esp-hal hold no driver for
the part.  Each fix kept is named in `src/bq27441.nv` where it is kept: the
1-byte packets and the bus-free time of the data sheet, the 5 ms and 10 ms
waits after a block is selected and a checksum written, the most
significant byte first order of the data memory, and the checksum read
against the bytes before a block is written back.

## Licence

Apache-2.0.
