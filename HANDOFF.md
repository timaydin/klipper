# Handoff: Marlin settings to carry into Klipper

This document lists the settings of the DIY printer's Marlin configuration, translated into Klipper terms, for
writing its first `printer.cfg`. Once `printer.cfg` holds them, this document has done its job and is deleted.

The source is the Marlin fork's branch `local_skrpro_v1.2_bugfix` (commit `0c2a21d218`, in the sibling `Marlin`
folder), whose `LOCAL_CONFIG.md` explains what the choices are for. The pin names come from Marlin's pins file for
the board (`Marlin/src/pins/stm32f4/pins_BTT_SKR_PRO_common.h`). Cross-check them against Klipper's sample config
for the SKR Pro (`config/generic-bigtreetech-skr-pro*.cfg` in the Klipper source), which is the starting point for
`printer.cfg`; the MCU build settings (`make menuconfig`) come from that sample's header.

Values marked **verify** depend on hardware facts the Marlin config does not record.

## Before Marlin is replaced

The printer's EEPROM may hold values that differ from the config files (tuned with `M301`, `M304`, `M851`, `M92`,
`M900`). Send `M503` to the printer and save its output next to this file before flashing Klipper. The probe's Z
offset in particular is only there: the config leaves it at 0.

## Board and connection

| Item           | Value                                                                                            |
| -------------- | ------------------------------------------------------------------------------------------------ |
| Board          | BTT SKR Pro v1.2, STM32F407                                                                      |
| Host link      | The board's native USB (Marlin `SERIAL_PORT -1`), as `[mcu] serial: /dev/serial/by-id/...`       |
| Second serial  | Marlin enables UART 1 at 250000 baud; what is connected to it is not recorded (**verify**)       |

## Kinematics

Marlin's `COREYX` computes its motors as A = Y + X and B = Y − X; Klipper's `corexy` uses A = X + Y and
B = X − Y. A is the same; B is the opposite sign. So the printer is Klipper `kinematics: corexy` with the motor
on the X socket as `stepper_x` and the one on the Y socket as `stepper_y`, and the Y direction flipped once
relative to Marlin. Marlin's `INVERT_Y_DIR` is true, so the two flips cancel and `stepper_y`'s `dir_pin` starts
uninverted. All starting directions in the table below are expectations: check each motor with `STEPPER_BUZZ`
and small moves before the first homing.

| Setting              | Marlin                                    | Klipper                                                                   |
| -------------------- | ----------------------------------------- | ------------------------------------------------------------------------- |
| Build volume         | 300 × 300 × 300 mm, min 0                 | `position_min: 0`, `position_max: 300` on X, Y and Z                      |
| X direction          | `INVERT_X_DIR false`                      | `dir_pin: PF1` (not inverted)                                             |
| Y direction          | `INVERT_Y_DIR true`, COREYX               | `dir_pin: PE8` (not inverted, see above)                                  |
| Z direction          | `INVERT_Z_DIR true`                       | `dir_pin: !PC2`                                                           |
| E0 direction         | `INVERT_E0_DIR true`                      | `dir_pin: !PA0`                                                           |
| Max velocity         | X/Y 300 mm/s, Z 5 mm/s, E 25 mm/s         | `max_velocity: 300`, `max_z_velocity: 5`, `max_extrude_only_velocity: 25` |
| Max acceleration     | X/Y 3000, Z 1000, E 10000 mm/s²           | `max_accel: 3000`, `max_z_accel: 1000`                                    |
| Print/travel accel   | 3000 / 3000 mm/s², retract 3000           | `max_accel: 3000`; `max_extrude_only_accel: 3000`                         |
| Cornering            | Junction deviation 0.013 mm at 3000 mm/s² | `square_corner_velocity: 9.7` (the same cornering)                        |
| S-curve acceleration | On                                        | Klipper has none; input shaping takes its place                           |
| Arcs                 | `ARC_SUPPORT`                             | `[gcode_arcs]`                                                            |

## Steppers and drivers

`rotation_distance` = full steps per turn × microsteps ÷ Marlin's steps per mm. The values assume 1.8° motors
(200 full steps per turn); a 0.9° motor doubles them (**verify**).

| Axis | Socket pins (step / dir / enable) | Steps/mm | Microsteps | `rotation_distance` | Driver  | Interface                  | Current (RMS) |
| ---- | --------------------------------- | -------- | ---------- | ------------------- | ------- | -------------------------- | ------------- |
| X    | PE9 / PF1 / !PF2                  | 80       | 16         | 40                  | TMC2209 | UART on PC13               | 0.8 A         |
| Y    | PE11 / PE8 / !PD7                 | 80       | 16         | 40                  | TMC2209 | UART on PE3                | 0.8 A         |
| Z    | PE13 / PC2 / !PC0                 | 200      | 4          | 4                   | TMC5160 | Software SPI, CS PB9       | 2.0 A         |
| E0   | PE14 / PA0 / !PC3                 | 932      | 16         | 3.433               | TMC2209 | UART on PD4                | 0.8 A         |

- The TMC5160's software SPI uses MOSI PC12, MISO PC11 and SCK PC10 (`spi_software_mosi_pin` and the like).
- Marlin's sense resistor for all four is 0.11 Ω. That is right for the TMC2209 sockets, but TMC5160 modules
  usually carry 0.075 Ω, and Klipper's `[tmc5160]` needs the real value in `sense_resistor` to set the current
  right (**verify** on the module).
- Marlin's Z microstepping of 4 is a choice, not a limit; Klipper with `interpolate` (on in Marlin) can run 16 if
  the step rate allows it.
- Marlin runs stealthChop with hybrid thresholds: X/Y 100 mm/s, Z 3 mm/s, E 30 mm/s (`stealthchop_threshold`).
  For fast printing with input shaping, Klipper recommends spreadCycle on X and Y (`stealthchop_threshold: 0`).
- Marlin's chopper timing is its 12 V default preset, left unchanged; the supply voltage is not recorded
  (**verify**). Klipper's defaults need no preset.
- Sensorless homing is not used. The TMC2209 DIAG jumpers must be removed, because the X, Y and Z DIAG lines
  share the endstop pins.

## Endstops, homing and probe

| Setting               | Marlin                                                       | Klipper                                                                           |
| --------------------- | ------------------------------------------------------------ | --------------------------------------------------------------------------------- |
| X endstop             | X min, triggered LOW, pull-up on                             | `endstop_pin: ^!PB10`, `position_endstop: 0`                                      |
| Y endstop             | Y min, triggered LOW, pull-up on                             | `endstop_pin: ^!PE12`, `position_endstop: 0`                                      |
| Z endstop             | The probe, on the Z min socket                               | `endstop_pin: probe:z_virtual_endstop`                                            |
| Probe                 | Fixed-mounted, on Z min (PG8), triggered LOW                 | `[probe] pin: ^!PG8`                                                              |
| Probe offset          | X −43, Y −4; Z from EEPROM                                   | `x_offset: -43`, `y_offset: -4`, `z_offset` from `M503`                           |
| Homing speed          | X/Y 50 mm/s, Z 4 mm/s                                        | `homing_speed: 50` (X/Y), `homing_speed: 4` (Z)                                   |
| Second homing pass    | Bump 5 / 5 / 2 mm at half, half and a quarter of the speed   | `homing_retract_dist: 5` (X/Y), `2` (Z); `second_homing_speed: 25` (X/Y), `1` (Z) |
| Safe Z homing         | At the bed centre                                            | `[safe_z_home] home_xy_position: 150, 150`                                        |
| Probe speed           | 4 mm/s fast, 2 mm/s slow; one sample                         | `speed: 4`; `samples: 1`                                                          |
| Lift between probes   | 5 mm, travel at 133 mm/s                                     | `horizontal_move_z: 5`, `speed: 133` in `[bed_mesh]`                              |
| Probe repeatability   | `M48`                                                        | `PROBE_ACCURACY`                                                                  |

## Bed mesh and tramming

| Setting      | Marlin                                                   | Klipper                                                                  |
| ------------ | -------------------------------------------------------- | ------------------------------------------------------------------------ |
| Levelling    | UBL, saved in EEPROM                                     | `[bed_mesh]`, saved with `SAVE_CONFIG`                                   |
| Grid         | 3 × 3                                                    | `probe_count: 3, 3`; Klipper handles more points easily                  |
| Probed area  | 10 mm from the edges, limited by where the probe reaches | `mesh_min: 10, 10`, `mesh_max: 257, 290` (probe coordinates)             |
| Preheat      | Hotend 120 °C and bed 50 °C before probing               | A `BED_MESH_CALIBRATE` wrapper macro that heats first                    |
| Tramming     | `G35`, probe points 50 mm in from each corner, M4 screws | `[screws_tilt_adjust]`, `screw_thread: CW-M4`                            |
| Babystepping | On                                                       | Built in: `SET_GCODE_OFFSET Z_ADJUST=...`                                |

Marlin's tramming points are probe positions (`probe_at_point` is probe-relative); Klipper's `screwN` is the nozzle
position that puts the probe over the screw. With the probe offset that gives `screw1: 93, 54`, `screw2: 293, 54`,
`screw3: 293, 254`, `screw4: 93, 254` (check against Klipper's documentation for `screws_tilt_adjust`).

## Heaters and temperatures

| Setting              | Hotend                                                                                                      | Bed                                                                                     |
| -------------------- | ----------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Heater pin           | PB1                                                                                                         | PD12                                                                                    |
| Sensor pin           | PF4                                                                                                         | PF3                                                                                     |
| Sensor               | PT1000 with 4.7 kΩ pull-up (Marlin type 1047): `sensor_type: PT1000`, `pullup_resistor: 4700`               | EPCOS 100K (Marlin type 1): `sensor_type: EPCOS 100K B57560G104F`                       |
| Control              | PID, `max_power: 0.98` (Marlin `PID_MAX` 250 of 255)                                                        | PID, `max_power: 1.0`                                                                   |
| Marlin PID values    | Kp 14.96, Ki 0.87, Kd 64.62 (Biqu H2)                                                                       | Kp 161.12, Ki 31.47, Kd 549.95 (CoreMax 300 Pro)                                        |
| Temperature limits   | `min_temp: 5`, `max_temp: 275`                                                                              | `min_temp: 5`, `max_temp: 150`                                                          |
| Runaway protection   | 4 °C hysteresis over 40 s; must gain 2 °C in 40 s                                                           | 2 °C hysteresis over 20 s; must gain 2 °C in 60 s                                       |
| Klipper equivalent   | `[verify_heater extruder]`: `hysteresis: 4`, `max_error` to match, `check_gain_time: 40`, `heating_gain: 2` | `[verify_heater heater_bed]`: `hysteresis: 2`, `check_gain_time: 60`, `heating_gain: 2` |

Run `PID_CALIBRATE` for both heaters under Klipper rather than reusing the Marlin values; they are listed as a
reference.

## Extruder

| Setting              | Marlin                         | Klipper                              |
| -------------------- | ------------------------------ | ------------------------------------ |
| Filament             | 1.75 mm                        | `filament_diameter: 1.750`           |
| Cold extrusion guard | 170 °C                         | `min_extrude_temp: 170`              |
| Longest E-only move  | 200 mm                         | `max_extrude_only_distance: 200`     |
| Linear advance       | Off                            | Calibrate `pressure_advance`         |

The toolhead is the Biqu H2 direct drive. Marlin also keeps a commented profile for a BMG with a Super Volcano
(steps/mm 80 / 80 / 800 / 415, hotend PID Kp 16.14, Ki 0.76, Kd 85.54, no probe offset recorded). Its Z value
of 800 steps/mm differs from the active 200; why is not recorded.

## Fans

| Fan      | Pin  | Marlin                                                               |
| -------- | ---- | -------------------------------------------------------------------- |
| Part fan | PC8  | `FAN0`, the part-cooling fan                                         |
| FAN1     | PE5  | Not configured; no hotend auto-fan is set (`E0_AUTO_FAN_PIN -1`)     |

How the hotend fan is wired is not recorded (**verify**); in Klipper it would be a `[heater_fan]`, with Marlin's
usual threshold of 50 °C.

## Preheat presets

| Preset | Hotend | Bed    |
| ------ | ------ | ------ |
| 1      | 180 °C | 70 °C  |
| 2      | 240 °C | 110 °C |

In Klipper these become macros, or presets in Mainsail.

## Diagnostics

Marlin's `MONITOR_DRIVER_STATUS` and `TMC_DEBUG` correspond to Klipper's `DUMP_TMC` and its built-in driver
error checks; `DEBUG_LEVELING_FEATURE` has no counterpart.
