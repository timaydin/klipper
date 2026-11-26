# Klipper printer

The Klipper configuration of the DIY printer: a CoreYX with a 300 mm cube build volume, a BTT SKR Pro v1.2 board
and a heavy direct-drive toolhead. Klipper replaces Marlin on the printer to get accelerometer-calibrated input
shaping, so the toolhead can be driven faster without ringing.

## Components

- **Klipper MCU firmware** runs on the SKR Pro. It only drives the steppers, heaters, fans and sensors, on the
  host's commands.
- **Klipper host (Klippy)** runs on a host computer connected to the board over USB. It reads `printer.cfg`,
  plans the motion, including input shaping, and streams the steps to the board.
- **Moonraker** gives Klippy a web API.
- **Mainsail** is the web UI, served next to Moonraker; slicers upload jobs to Moonraker.
- **An accelerometer** on the toolhead measures the frame's resonances for input-shaper calibration. Because the
  printer is a CoreYX, one sensor on the head measures both X and Y.

## Layout

The project is a branch of the Klipper fork. Upstream's files stay unchanged, so the branch follows upstream
without conflicts; `README.md` is Klipper's own.

- `config/printer-diy-skr-pro.cfg`: the printer's Klipper configuration, installed into the host's Klipper
  config folder as `printer.cfg`.
- `HANDOFF.md`: the printer's settings carried over from its Marlin configuration, until the printer's
  configuration holds them.

## Fallback

The printer's Marlin configuration stays in the Marlin fork, on the branch for this board, so the board can be
flashed back to Marlin.
