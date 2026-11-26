# To do

- Hand the rebuild guide to an infra session as the klipper host's rebuild guide, then delete it here.

## Driver optimization

- Z: run spreadCycle only (`stealthchop_threshold: 0`); stealthChop gives less torque under load and Klipper's TMC
  documentation advises spreadCycle for positional accuracy.
- Z: switch off step interpolation (`interpolate: False`); at the coarse microstepping it costs close to half a
  microstep of position error while moving.
