# Changelog

## 0.1.2 — 2026-11-27

The README example and `examples/tone.nv` call `amp.amp_set_gain` and `amp.amp_stop_soft` without a binding.  No signature of the package changed.

## 0.1.1 — 2026-11-23

The package asks for embedded-hal-nv ^0.4.0 (was ^0.3.3), the minor of both trait packages that the other drivers and the board packages ask for, so one version of each serves a program that takes this package beside any of them.  Under the older ranges, a program that named this package and a driver on 0.4 stopped with "no version of 'embedded-hal-nv' satisfies every requirement".

No signature of the package changed.  `i2s_config_of` gives the board's `I2sConfig` the `slot_bits` that embedded-hal-nv 0.4.0 adds, 0, a slot as wide as the sample.

The device program builds for the boards its README names, flash nrf52840-dk 9,208 B before and 9,148 B after; nrf54lm20-dk 10,544 B before and 10,484 B after; nrf52-qemu 9,036 B before and 8,976 B after.

## 0.1.0

The first release: a driver for the MAX98357A and MAX98357B I2S class D audio
amplifiers, which have no register, over a board's `I2s` handle and two GPIO
pins.

- `Max98357a`, the amplifier built by `attach` from the board's `I2s` and
  `GpioOut` handles, the wiring (`AmpWiring`: the SD_MODE pin, the gain pin and
  whether 100 kohm sits before GAIN_SLOT) and the stream (`AmpConfig`: the part,
  the word clock, the word size).
- `start`, which keeps the part in shutdown while the handle is configured and
  its clocks start and raises SD_MODE last, refusing a stream the part does not
  take; `stop`, which lowers SD_MODE before the clocks stop; and `stop_soft`,
  which sends a ramp to zero first.
- The stream checks of the data sheet: the seven word clocks, the 32, 48 and 64
  times BCLK ratios, I2S for the MAX98357A and left-justified for the
  MAX98357B.
- SD_MODE: the pin sequence, the equations for RSMALL and RLARGE from the
  logic level, and the channel a voltage selects by the trip points.
- The gain pin: the five gains of Table 8, what a GPIO can make of them with
  and without a 100 kohm resistor, and `gain_set`.
- A tone generator: a 66-point quarter-wave sine table, a 24-bit phase,
  interpolation, scaling into 16, 24 and 32-bit words, buffers of 128 frames
  with the phase carried across them, and `play_tone`.
- The device program `examples/tone.nv`, a 440 Hz tone every second. Built for
  the nRF52840-DK it is 13,348 B of flash and for the nRF54LM20-DK 14,868 B.

### Known

- Hardware verification: NOT VERIFIED. No board on the bench carries the
  amplifier, and neither Nordic board package drives its I2S block, so the
  device program is given a stand-in handle that refuses to be configured and
  prints `refused`.
- Not implemented: TDM mode, a volume or mute (the part has none), the channel
  select by software (it is a resistor), mono frames and the async sink.
- `embedded-hal-nv` is the registry range `^0.3.3`, the first release with the
  `I2s` trait. The package needs no `devices-nv`: the part has no register for
  a trait to set.
- The `[tier]` of the manifest is `embedded`, the strictest, so the module also
  checks at `rt` and `system`.
