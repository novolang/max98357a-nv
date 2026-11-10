# max98357a-nv

A driver for the MAX98357A and the MAX98357B, digital PCM input class D audio
power amplifiers from Maxim (now Analog Devices), specified in the data sheet
"MAX98357A/MAX98357B PCM Input Class D Audio Power Amplifiers", document
19-6779; Rev 7; 2/16. The part has no register and no control bus: it takes
I2S samples on three lines and has two pins that matter to a program, SD_MODE
(shutdown and channel select) and GAIN_SLOT (gain). The package is the
amplifier seen as a sink for the samples of the `I2s` handle of
[embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) that a board
gives it, with the two pins driven through `GpioOut`.

It is a package of its own, and not part of
[inmp441-nv](https://novo-lang.org/packages/inmp441-nv), because that package
is named for the microphones it drives and takes samples in; this one sends
them out. It implements no trait of devices-nv, because a part with no
register has nothing for `AudioCodec` to set that a pin does not already set,
and so it depends on embedded-hal-nv only.

**Hardware verification: NOT VERIFIED.** No board on the bench carries this
amplifier, and no board package has an I2S block yet. The driver has run only
against doubles of the I2S handle and the pins, in the host suite, and the
device program builds for the nRF52840 DK and the nRF54LM20 DK against a
stand-in handle that refuses to be configured. A tone from a part on a board
has not been heard.

## Install

```sh
novo pkg add max98357a-nv
```

The device program is `examples/tone.nv`.  Make it the `main` of a package that
depends on this one and build it for a board:

```sh
novo build --target=nrf52840-dk src/main.nv
novo build --target=nrf54lm20-dk src/main.nv
```

## The device program

`examples/tone.nv` plays 440 Hz for half a second, a ramp to zero and half a
second of silence, again and again, through a MAX98357A on a breakout, and
prints one line, `playing` or `refused`, over the board's UART:

```novo norun:needs-pkg
use embedded_hal.{ I2s, I2sConfig, BoardGpio }
use max98357a
use max98357a.{ AmpGain, AmpWiring }

// The board's I2S block goes here. This stand-in refuses to be configured,
// as the Nordic board packages' absent hardware does.
@value
struct NoI2s
    id: Int

impl I2s for NoI2s
    fn configure(self, c: I2sConfig) -> Bool [hw]
        false
    fn sample_rate(self) -> Int [hw]
        0
    fn enable(self) -> Bool [hw]
        false
    fn disable(self) [hw]
        pass
    fn transmit(self, var buf: Vec[i32; 256]) -> Int [hw]
        -1
    fn receive(self, var buf: Vec[i32; 256], n: Int) -> Int [hw]
        -1
    fn xruns(self) -> Int [hw]
        0

@tier(embedded)
fn main() [hw]
    bsp.board.init()
    let wiring = AmpWiring { sd_mode: 28, gain: Some(29), gain_via_100k: false }
    let amp = max98357a.attach(NoI2s { id: 0 }, BoardGpio { id: 0 }, wiring, max98357a.config(16000))
    let _ = amp.amp_set_gain(AmpGain.Gain9Db)
    let settle_us = amp.amp_start()
    if settle_us < 0
        hal.uart.write("refused\n")
    else
        hal.uart.write("playing\n")
        hal.timer.delay_ms(settle_us / 1000 + 1)
    while true
        let sent = amp.amp_tone(440, 500, 16384)
        if sent > 0
            let _ = amp.amp_stop_soft(0)
        hal.timer.delay_ms(500)
```

The program is `examples/tone.nv`. A board package that depends on this
package builds it as its `main`. `tests/device_build.sh` does so from a scratch
package for the two boards below and for the emulated board, where it runs the
image and checks that the program prints `refused`:

```sh
bash tests/device_build.sh
probe-rs run --chip nRF52840_xxAA main.elf
```

| Image | Flash | Of which program | RAM (`.data` and `.bss`) |
|---|---|---|---|
| `nrf52840-dk` | 13,348 B | 1,602 B | 2,644 B |
| `nrf54lm20-dk` | 14,868 B | 2,110 B | 2,644 B |

The numbers are from the build's memory report. Flash also holds the runtime,
the compiler's support library and the program's tables, among them the
66-point sine table. The arena, 65,536 B on both boards, is the runtime's heap
and is reserved whether or not the program uses it.

**The I2S block is a stand-in.** embedded-hal-nv has a trait for an I2S
interface and no board handle for one, and neither Nordic board package drives
its I2S block. The program is written against the `I2s` trait and is given
`NoI2s`, so on both images `amp_start` answers -1 and the program prints
`refused`. The one argument of `attach` is what changes when a board package
gives a handle. The wiring the program assumes: the board's bit clock, word
clock and data output to BCLK, LRCLK and DIN; SD_MODE to pin 28 and GAIN to
pin 29 of the board's GPIO; a speaker of 4 to 8 ohms on the output terminals;
Vin from 2.5 to 5.5 V.

### The effects

Every function of the driver that reaches the I2S handle or a pin declares
`[hw]`, and nothing else. The driver performs no input or output of its own.
`attach` allocates one cell of the arena for the amplifier, because a struct
with a generic field cannot be a `@value`; a tone is built in a
`Vec[i32; 256]` that the caller or `play_tone` keeps in its frame. The module
checks at every tier, `embedded` included.

## What the part is

The MAX98357A is a 3.2 W (4 ohm, 5 V) class D amplifier with a digital input
and a filterless output. It needs no master clock: it takes BCLK and LRCLK and
reads DIN on the rising edge of BCLK. The MAX98357A takes standard I2S data
(the first bit of a word one BCLK after the LRCLK edge, the left word while
LRCLK is low); the MAX98357B takes left-justified data (no delay, the left
word while LRCLK is high). The driver covers both with `AmpPart`.

| | |
|---|---|
| Word clocks | 8, 16, 32, 44.1, 48, 88.2 and 96 kHz only; 11.025, 12, 22.05 and 24 kHz are not supported |
| BCLK | 32, 48 or 64 times LRCLK: a two-channel frame of 16, 24 or 32-bit words |
| Supply | 2.5 to 5.5 V |
| Turn-on time | 7.5 ms at most |
| Shutdown current | 0.6 uA typical, SD_MODE low |
| Standby | 340 uA, outputs high impedance, entered when BCLK stops |

### SD_MODE

SD_MODE is shutdown and channel select in one pin (Table 5). The pin has a
100 kohm pull-down inside, and four states:

| SD_MODE | Selected |
|---|---|
| high, above the 1.4 V trip point | the left word |
| high through RSMALL, from 0.77 to 1.4 V | the right word |
| high through RLARGE, from 0.16 to 0.77 V | (left + right) / 2 |
| low, below 0.16 V | shutdown |

`rsmall_tenth_kohm(vddio_mv)` and `rlarge_tenth_kohm(vddio_mv)` give the two
resistors from the data sheet's equations, RSMALL = 94.0 x VDDIO - 100 kohm
and RLARGE = 222.2 x VDDIO - 100 kohm, in tenths of a kilo-ohm: 210.2 and
633.2 kohm at 3.3 V, 69.2 and 299.9 kohm at 1.8 V. The data sheet's Table 6
lists values of a standard series next to them (69.8 and 300 kohm at 1.8 V,
210.2 and 634 kohm at 3.3 V). `channel_of_mv(mv)` answers the channel a
voltage on the pin selects, by the typical trip points.

The channel is chosen by the resistor between the board's GPIO and SD_MODE, so
the driver can turn the part on and off and cannot change the channel. A
breakout usually has the resistor fitted, and the program drives the pin high
to turn the part on and low to shut it down. Driving SD_MODE straight to logic
high from a rail above the part's supply violates the absolute maximum rating;
the data sheet asks for VDD above VDDIO, or about 2 kohm in series.

### GAIN_SLOT

The pin is read, not clocked: the gain is the state it is strapped to
(Table 8).

| GAIN_SLOT | Gain |
|---|---|
| to ground through 100 kohm | 15 dB |
| to ground | 12 dB |
| unconnected | 9 dB |
| to VDD | 6 dB |
| to VDD through 100 kohm | 3 dB |

A board that drives the pin from a GPIO gets three of the five, depending on
whether it fits 100 kohm between the pin and GAIN_SLOT. `AmpWiring.gain_via_100k`
says which. Without the resistor a GPIO makes 12 dB (low), 6 dB (high) and
9 dB (released, an input); with it, 15 dB (low), 3 dB (high) and 9 dB.
`gain_drive(g, via_100k)` answers what the pin has to do, and `gain_set`
does it and answers false for a gain the wiring cannot make. A board that
straps the pin leaves `gain` as `None` and `gain_set` drives nothing.

### The frame

`i2s_config_of(cfg)` gives the `I2sConfig` the handle is configured with: the
chip is the master, two channels, the word `bit_depth` bits, Philips for the
MAX98357A and left-justified for the MAX98357B. The data is a stereo stream;
the part plays the word SD_MODE selects. A program that has one channel of
audio sends each sample twice. `start` refuses a rate or a word the part does
not take (`config_ok`) with -1, before any pin or clock is touched but
SD_MODE, which it drives low.

## What the package contains

| Module | What is in it |
|---|---|
| `max98357a` | `AmpPart`, `AmpGain`, `AmpChannel`, `AmpConfig`, `AmpWiring`, the stream checks, the SD_MODE and gain-pin functions, `start`, `stop` and `stop_soft`, the tone generator and the type `Max98357a` over a board's handles |

## How to choose an entry point

A program on a board builds an `AmpConfig` with `config(rate)` (16-bit words,
the MAX98357A) and an `AmpWiring` with the pins, passes both with the board's
`I2s` and `GpioOut` handles to `attach`, and uses the methods of the result.

| Function | What it does |
|---|---|
| `config(rate)` | a MAX98357A stream of 16-bit words at `rate` hertz |
| `attach(i2s, gpio, wiring, cfg)` | the amplifier on the handles; nothing is driven |
| `amp_start()` | SD_MODE low, the handle configured and enabled, SD_MODE high; answers 7500, the turn-on time in microseconds, or -1 |
| `amp_stop()` | SD_MODE low, then the handle disabled |
| `amp_stop_soft(last)` | one buffer that falls from the last sample sent to zero, then `amp_stop`; false when the ramp was refused, the part is stopped either way |
| `amp_write(buf)` | sends the samples of `buf`, a stereo `Vec[i32; 256]`; answers the count or -1 |
| `amp_tone(freq, ms, amp)` | plays a sine of `freq` hertz for about `ms` milliseconds at amplitude `amp`, 0 to 32767; answers the samples sent or -1 |
| `amp_set_gain(g)` | sets the gain by the gain pin |
| `amp_lost()` | the handle's count of lost buffers |
| `config()`, `wiring()` | what the amplifier was built with |
| `rate_ok(rate)`, `bclk_ratio_ok(bits)`, `config_ok(cfg)`, `bclk_hz(cfg)` | the checks on a stream, and the bit clock it implies |
| `sine_q15(phase)`, `tone_step(freq, rate)`, `tone_sample(phase, amp, bits)`, `tone_fill(buf, phase, step, amp, bits)` | the tone generator |

The functions `start`, `stop`, `stop_soft`, `write`, `play_tone` and
`gain_set` take the handles directly, for a program that holds them itself.

## The tone

The tone generator needs no floating point and no allocation. The table
`SINE_QUARTER` holds 65 points of a sine from 0 to 90 degrees, scaled to
32767, and a 66th equal to the last. A phase is 24 bits to the cycle;
`sine_q15` takes the quadrant from the top two bits, interpolates linearly
between two points with eight more bits, and mirrors the quarter into the
other three. `tone_step(freq, rate)` is the phase step of a frequency at a
word clock. `tone_sample` scales the sine by an amplitude and moves it to the
top of a 16, 24 or 32-bit word; the sign is applied after the scaling so that
the negative half wave is the mirror of the positive one. `tone_fill` writes
128 frames (256 samples, the same sample on both channels) and answers the phase after
them, and `play_tone` sends as many buffers as `ms` milliseconds need, carrying
the phase across the buffers so the wave has no join.

The table is checked against `32767 * sin(i * pi / 128)` at points of the
data sheet's quarter: 12539 at 22.5 degrees, 23170 at 45 degrees, 30273 at
67.5 degrees. The interpolated value is within 3 counts of the sine at the
points the suite checks.

## The rules a user needs

1. The part is off until SD_MODE is high, and `start` raises it last. The data
   sheet says the click-and-pop is the same whether the clocks come before or
   after SD_MODE; the order keeps the part from seeing half a frame.
2. Wait `amp_start`'s answer, 7.5 ms, before the first sound.
3. Do not remove LRCLK while BCLK is present: the output may show a large DC
   voltage. `amp_stop` lowers SD_MODE before the clocks stop for that reason.
4. The part has no ramp on shutdown. Send a ramp to zero first with
   `amp_stop_soft`, or end the sound at a zero crossing.
5. The word clock must be one of the seven the part takes; 24 kHz is the one
   most often asked for and is refused.
6. The gain pin is a strap. A board that wants a gain the GPIO cannot make
   straps the pin and passes `None`.
7. Send the buffers in time: the handle counts a buffer it was not given in
   time as lost (`amp_lost`), and a gap in the stream is silence.
8. A board whose start is audible with SD_MODE raised at once, which the
   kernel driver's `sdmode-delay` exists for, waits between the clocks and the
   pin. The free functions are the pieces: configure and enable the handle,
   wait, then `sd_on`; `amp_start` is the three without the wait.

## What is not included

NOT IMPLEMENTED:

- TDM mode (16 or 32-bit words, up to eight slots, a frame-sync pulse on LRCLK,
  gain fixed at 12 dB, the slot chosen by SD_MODE and GAIN_SLOT together,
  Table 7). The driver sends the two-channel frame.
- The MAX98357B's falling-edge TDM clocking, which a board's I2S block owns.
- A volume or a mute: the part has none, and a program scales its samples.
- The channel select by software: it is a resistor.
- The async form of the sink (`I2sAsync`).
- Mono frames: the handle is configured for two channels.

## Related packages

- [embedded-hal-nv](https://novo-lang.org/packages/embedded-hal-nv) names the
  `I2s` and `GpioOut` traits and their settings.
- [inmp441-nv](https://novo-lang.org/packages/inmp441-nv) drives two I2S
  microphones, the capture side of the same `I2s` handle.
- [es8311-nv](https://novo-lang.org/packages/es8311-nv) and
  [wm8960-nv](https://novo-lang.org/packages/wm8960-nv) drive audio codecs with
  an amplifier output and a control bus.

## Tests

`novo test` runs the module over doubles of the `I2s` handle and of the pins
that share one event log. The suite checks the seven word clocks and the four
the data sheet names as not supported, the three word sizes, the settings the
handle gets for each part, the resistor equations against the data sheet's
worked values, the trip points at their edges, every gain on both wirings
against Table 8, the order of the pin and clock events of `start` (SD_MODE low,
configure, enable, SD_MODE high), each refusal of it, the order of `stop` and
`stop_soft`, the ramp, the sine table and its four quadrants against hand
values, the scaling into 16, 24 and 32-bit words, a buffer of the tone against
the samples worked by hand, the phase carried from one buffer to the next, and
the type over the doubles. The suite covers every line of `src/`:

```sh
NOVO=bin/novo bash tests/coverage.sh
```

**Hardware: NOT VERIFIED.** See the device program section.

## Sources

The sources read for this driver, for their fixes and not ported, are the data
sheet above and the Linux kernel's `sound/soc/codecs/max98357a.c` (GPL, read
for the sequence and the lists only). The kernel driver raises SD_MODE at the
start of a stream and lowers it at the stop, the same order as here, takes the
same seven rates and the 16, 24 and 32-bit words, and has an optional delay
(`sdmode-delay`, milliseconds) between the clocks and SD_MODE for boards where
the start is audible; this driver raises SD_MODE at once, and a board that
needs the delay uses the free functions (see rule 8). The Zephyr tree, the Embassy repository, the esp-hal
repository and the Rust crates have no driver for this part, because it has no
register; the I2S drivers a board's handle will come from (Embassy's nRF I2S,
Zephyr's `i2s_nrfx`, esp-hal's I2S master) were read for the slot widths by
the inmp441-nv package, and this package asks the handle for 16, 24 or 32-bit
words as that package does. The pitfalls taken from the data sheet are named in
`src/max98357a.nv` at the sites that avoid them:

- LRCLK must not be removed while BCLK is present; the order of `stop` follows.
- The part has no volume ramp on shutdown; `stop_soft` sends one.
- LRCLK accepts seven rates only, and 11.025, 12, 22.05 and 24 kHz are not
  supported; `start` refuses them.
- SD_MODE driven directly to a logic rail above the supply is outside the
  absolute maximum ratings.

## Licence

Apache-2.0.
