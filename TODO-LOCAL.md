# To do

## Decisions for the admin

- Which accelerometer to buy, and whether it connects to the host over USB or to the board or host by SPI.

## Hardware facts to check

- The TMC5160 module's sense resistor.
- The motors' step angle (1.8° or 0.9°).
- The supply voltage.
- How the hotend fan is wired.
- That the TMC2209 DIAG jumpers are removed.

## Steps

- Save the printer's `M503` output before Marlin is replaced.
- Hand the rebuild guide to an infra session as the klipper host's rebuild guide, then delete it here.
- Build the Klipper MCU firmware for the SKR Pro and flash it.
- Write the printer's configuration from the handoff, then delete the handoff.
- Bring the printer up as the project's rules describe: directions, endstops and probe, PID, probe Z offset, bed
  mesh, tramming.
- Mount the accelerometer, calibrate the input shaper, then calibrate pressure advance.
- Point the slicer's upload at Moonraker.
