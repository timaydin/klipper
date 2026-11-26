# Klipper printer

The Klipper configuration of the DIY printer: a CoreYX with a 300 mm cube build volume, a BTT SKR Pro v1.2 board
and a heavy direct-drive toolhead. Klipper replaces Marlin on the printer to get accelerometer-calibrated input
shaping, so the toolhead can be driven faster without ringing.

## Components

- **Klipper MCU firmware** runs on the SKR Pro. It only drives the steppers, heaters, fans and sensors, on the
  host's commands.
- **Klipper host (Klippy)** runs on the host `klipper`, a Raspberry Pi 3 Model B+ connected to the board over USB.
  It reads `printer.cfg`, plans the motion, including input shaping, and streams the steps to the board. The Pi is
  an infra host: its system, network and access are infra's; the Klipper stack on it is this project's.
- **Moonraker** gives Klippy a web API.
- **Mainsail** is the web UI, served next to Moonraker; slicers upload jobs to Moonraker.
- **An accelerometer** on the toolhead measures the frame's resonances for input-shaper calibration. Because the
  printer is a CoreYX, one sensor on the head measures both X and Y.

## Layout

The project is a branch of the Klipper fork. Upstream's files stay unchanged, so the branch follows upstream
without conflicts; `README.md` is Klipper's own.

- `config/printer-diy-skr-pro.cfg`: the printer's Klipper configuration, everything but what is tuned on the
  printer. The host reads it from its own clone of this branch, so a Klipper update in Mainsail also updates the
  configuration.
- The host's `printer.cfg`, in its Klipper config folder, is not in the repo: it includes Mainsail's macros and the
  configuration above, and holds what belongs to this host and this printer alone, the board's serial port and the
  values `SAVE_CONFIG` writes (PID, probe offset, bed mesh, input shaper).
- `HANDOFF.md`: the printer's settings carried over from its Marlin configuration, until the printer's
  configuration holds them.
- `REBUILD.md`: the rebuild guide of the Klipper host, until infra holds it as the host's own.

## Fallback

The printer's Marlin configuration stays in the Marlin fork, on the branch for this board, so the board can be
flashed back to Marlin.
