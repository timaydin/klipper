# Rules for the printer's Klipper branch

Our standard documents carry the suffix `-LOCAL`, so they never collide with upstream's:

- `README-LOCAL.md`: what the project is and how it is put together.
- `TODO-LOCAL.md`: the action items.

## Configuration

- The printer's configuration follows Klipper's sample config for the SKR Pro in layout and naming; values
  that are tuned (`SAVE_CONFIG`) live in its auto-saved block at the end, not by hand in the sections above it.
- Pin names are the board's own (`PB10`, `^!PG8`), written as Klipper's sample config writes them.
