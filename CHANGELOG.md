# Changelog

## 0.1.0

The first release: a driver for TI's BQ27441-G1 single-cell fuel gauge on
an I2C bus.

- `Bq27441`, the type a board's device table builds, implementing the
  battery side of devices-nv's `PowerManager` and `Identifiable`, with its
  own methods for the voltage, the average current, the state of charge,
  the remaining capacity, the temperature, the Flags word, the seal and
  the configuration update.  The gauge supplies no rail.
- The standard commands as functions over `Bq27441Port`, a trait that a
  type holding a bus implements: voltage, temperature, internal
  temperature, average, standby and maximum-load current, power, state of
  charge, the four capacities, state of health with its status, OpConfig
  and Design Capacity.
- The Control subcommands as 1-byte packets, low byte then high byte, with
  the 66 microsecond bus-free time after every packet that the board's
  type waits; CONTROL_STATUS, DEVICE_TYPE, FW_VERSION, DM_CODE, CHEM_ID,
  BAT_INSERT, BAT_REMOVE, SET_HIBERNATE, CLEAR_HIBERNATE, SHUTDOWN_ENABLE,
  SHUTDOWN, TOGGLE_GPOUT, RESET; the unseal key written twice in a row and
  SEALED.
- The data memory in 32-byte blocks with the block checksum, most
  significant byte first, with the checksum read against the bytes before a
  block is written back, the 5 ms and 10 ms waits, and the checksum read
  back after the write; `checksum_after` is the technical reference's quick
  method.
- The configuration update of the technical reference's section 3.1 as
  `config_update`: unseal, CONFIG UPDATE mode, the design capacity, design
  energy, terminate voltage and taper rate and OpConfig as an option,
  SOFT_RESET, EXIT_RESIM or EXIT_CFGUPDATE, seal.  Fields that already hold
  the value are not written, and an error inside the mode still leaves it.
- Two async forms: `alert_arm` and `alert_disarm`, which arm the GPOUT pin
  on any `AsyncEdge` source for the falling edge, and `host_of`, which gives
  the host scheduler's `PowerManagerAsync` and waits for INITCOMP.
- The suite runs over a double of the gauge with its data memory and covers
  every line of `src/`.

### Known

- Hardware verification: NOT VERIFIED.  No board carries a BQ27441.
- The data sheet read is SLUSBH1C (revised December 2014), the brief names
  it SLUSBH2; the technical reference is SLUUAC9A (May 2015).
- The OpConfig and Design Capacity extended commands are read as little
  endian words, like the standard commands.  The data memory fields are
  most significant byte first.  Neither is verified on a part.
- The data sheet limits how often the standard commands may be issued; the
  sentence is garbled in the copy read, and the driver does not limit them.
- The calibration and learning subclasses, 8-bit and 32-bit data memory
  fields, and the -G2 and BQ27427 offsets are not implemented.
- No board's GPIO implements `AsyncEdge`, so `alert_arm` has no source on a
  board yet.
