# Klipper for the T3 Dizayn CoreMax 300

The printer's branch of the Klipper fork: its Klipper configuration, the build and update of its board's firmware,
and the rebuild guide of its host. Klipper replaces Marlin on the printer to get accelerometer-calibrated input
shaping, so the toolhead can be driven faster without ringing. The printer itself, its hardware and wiring, is
described in the printer folder's README.

## Components

- **Klipper MCU firmware** runs on the SKR Pro. It only drives the steppers, heaters, fans and sensors, on the
  host's commands.
- **Klipper host (Klippy)** runs on the host `klipper`. It reads `printer.cfg`, plans the motion, including input
  shaping, and streams the steps to the board. The host is an infra host: its system, network and access are
  infra's; the Klipper stack on it is this project's.
- **Moonraker** gives Klippy a web API.
- **Mainsail** is the web UI, served next to Moonraker; slicers upload jobs to Moonraker.

## Layout

The project is a branch of the Klipper fork. Upstream's files stay unchanged, so the branch follows upstream
without conflicts; `README.md` is Klipper's own.

- `config/printer-t3-dizayn-coremax-300.cfg`: the printer's Klipper configuration, everything but what is tuned on the
  printer. The host reads it from its own clone of this branch, so a Klipper update in Mainsail also updates the
  configuration.
- The host's `printer.cfg`, in its Klipper config folder, is not in the repo: it includes Mainsail's macros and the
  configuration above, and holds what belongs to this host and this printer alone, the board's serial port and the
  values `SAVE_CONFIG` writes (PID, probe offset, bed mesh, input shaper).
- `REBUILD.md`: the rebuild guide of the Klipper host, until infra holds it as the host's own.

## Updating the MCU firmware

The board runs firmware built from the same commit as the host's Klipper: after a Klipper update in Mainsail, Klipper
reports the board's older version, and the firmware is built and flashed again.

The firmware is built on the host, in its clone of this branch; the build settings stay in the clone's `.config`,
which git ignores, and the settings to choose are in the configuration file's header. The board's bootloader flashes
a `firmware.bin` it finds on the board's SD card at power-on.

- **The first flash** is by hand: `out/klipper.bin` copied as `firmware.bin` onto a FAT32 card, the card in the
  board, the board powered on. The bootloader renames the file to `FIRMWARE.CUR` once flashed. The card stays in the
  board.
- **Every later flash** goes over the USB link: with Klipper stopped, Klipper's `scripts/flash-sdcard.sh` sends the
  build through the running firmware onto that card, resets the board so the bootloader flashes it, and checks the
  new version. Its board name is `btt-skr-pro-v1.2`, its device the board's serial port from `printer.cfg`:

  ```bash
  sudo systemctl stop klipper
  cd ~/klipper && make clean && make
  ./scripts/flash-sdcard.sh /dev/serial/by-id/usb-Klipper_stm32f407xx_SERIAL-if00 btt-skr-pro-v1.2
  sudo systemctl start klipper
  ```

  A card that does not start up at the script's default SPI speed needs its `-s` option.
