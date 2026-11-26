# Rules for the Klipper printer

What the project is and how it is put together is in the local README.

## Never

- Never flash the board, start a print or move the printer without the admin's go-ahead for that step: the
  printer is real hardware, and a wrong direction or current can crash the toolhead or damage a motor.
- Never raise a heater's `max_temp`, `max_power` or its runaway-protection limits without the admin's OK.

## Bringing up the printer

- Check each motor's direction with `STEPPER_BUZZ` and small moves before the first homing. Check every endstop
  and the probe with `QUERY_ENDSTOPS` and `QUERY_PROBE`, triggered and not triggered, before the first homing.
- Run `PID_CALIBRATE` for both heaters before the first print; values tuned under Marlin are only a reference.
- Calibrate the input shaper with the accelerometer (`SHAPER_CALIBRATE`) before raising the acceleration, and
  keep the maximum acceleration at or below what the calibration recommends.

## Configuration

- The printer's configuration follows Klipper's sample config for the SKR Pro in layout and naming; values
  that are tuned (`SAVE_CONFIG`) live in its auto-saved block at the end, not by hand in the sections above it.
- Pin names are the board's own (`PB10`, `^!PG8`), written as Klipper's sample config writes them.

## Gotchas

- The TMC5160 on Z needs its module's real sense resistor in `sense_resistor`; the value from Marlin's
  configuration is the TMC2209 default and may not match.
- The TMC2209 DIAG jumpers must stay removed: their lines share the endstop pins.
