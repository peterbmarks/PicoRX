# Pi Pico Rx

A software-defined radio (SDR) receiver built around a [Raspberry Pi Pico](https://www.raspberrypi.com/products/raspberry-pi-pico/). With little more than a Pico, an analogue switch, and an op-amp, it forms a capable receiver covering the LW, MW, and SW bands — able to pull in signals from the other side of the world.

![concept](images/concept.svg)

## Features

- 0 – 30 MHz coverage
- 250 kHz bandwidth quadrature SDR receiver
- CW / SSB / AM / FM reception
- OLED (and optional ILI934x colour) display
- Simple spectrum scope and waterfall
- Headphone / speaker output (PWM audio) and USB audio streaming
- CAT control over USB serial
- 500 general-purpose memory channels
- Runs on 3 AAA batteries at less than 50 mA

For the full background, hardware design, and theory of operation, see the write-up at [Read the Docs](https://101-things.readthedocs.io/en/latest/radio_receiver.html) and the [user manual](user_manual/).

## How it works (software overview)

The firmware runs across both cores of the RP2040 / RP2350:

- **Signal capture & DSP** — A quadrature sampling detector feeds the ADC. The DSP chain ([`rx.cpp`](rx.cpp), [`rx_dsp.cpp`](rx_dsp.cpp), [`nco.cpp`](nco.cpp), [`fft_filter.cpp`](fft_filter.cpp), [`noise_reduction.cpp`](noise_reduction.cpp)) handles down-conversion, filtering, AGC, demodulation, and noise reduction.
- **Audio output** — Demodulated audio is sent to a PWM audio sink ([`pwm_audio_sink.cpp`](pwm_audio_sink.cpp)) and/or streamed over USB ([`usb_audio_device.c`](usb_audio_device.c)).
- **User interface** — Menus, the spectrum scope, and the waterfall are rendered to the display ([`ui.cpp`](ui.cpp), [`waterfall.cpp`](waterfall.cpp)) via the [u8g2](external/u8g2) graphics library and the [`ili934x.cpp`](ili934x.cpp) colour driver. Tuning and control come from a rotary encoder / buttons ([`rotary_encoder.cpp`](rotary_encoder.cpp), [`button.cpp`](button.cpp)).
- **Persistence** — Settings, memory channels, and autosave state are stored in flash ([`settings.cpp`](settings.cpp), [`memory.cpp`](memory.cpp), [`autosave_memory.cpp`](autosave_memory.cpp)). Factory defaults live in [`settings.h`](settings.h).
- **Remote control** — CAT command handling over USB serial lives in [`cat.cpp`](cat.cpp).

The build also produces a small [`battery_check`](battery_check.cpp) utility.

## Getting the code

```sh
git clone https://github.com/dawsonjon/PicoRX.git
cd PicoRX
git submodule update --init --recursive
```

The submodules (notably [u8g2](external/u8g2)) are required to build.

A precompiled binary is also available on the [releases page](https://github.com/dawsonjon/PicoRX/releases) if you don't want to build from source.

## Prerequisites

You need the **Raspberry Pi Pico SDK** and an **Arm GNU toolchain**. The easiest route is the official setup:

- **Linux:** follow [Getting started with the Raspberry Pi Pico](https://datasheets.raspberrypi.com/pico/getting-started-with-pico.pdf), or run the `pico_setup.sh` script.
- **Windows:** use the [Pico Windows installer](https://www.raspberrypi.com/news/raspberry-pi-pico-windows-installer/).
- **VS Code (any platform):** install the official **Raspberry Pi Pico** extension, which bundles a matching CMake, Ninja, and toolchain under `~/.pico-sdk`. This project is already configured for it (`.vscode/`).

## Building

The project uses CMake. The board is selected at configure time, so use a separate build directory per target.

### Raspberry Pi Pico (RP2040) — default

```sh
cmake -B build -DPICO_BOARD=pico -DPICO_SDK_PATH=~/pico/pico-sdk
cmake --build build
```

This produces `build/picorx.elf` / `picorx.uf2` and the `battery_check` utility.

### Raspberry Pi Pico 2 (RP2350, Arm)

```sh
cmake -B build -DPICO_BOARD=pico2 -DPICO_PLATFORM=rp2350-arm-s -DPICO_SDK_PATH=~/pico/pico-sdk
cmake --build build
```

Produces `build/pico2rx.elf` / `.uf2`.

### Raspberry Pi Pico 2 (RP2350, RISC-V)

```sh
cmake -B build -DPICO_BOARD=pico2 -DPICO_PLATFORM=rp2350-riscv -DPICO_SDK_PATH=~/pico/pico-sdk
cmake --build build
```

Produces `build/pico2rx-riscv.elf` / `.uf2`.

> **Note:** If `PICO_SDK_PATH` is not set, CMake will automatically fetch the SDK from GitHub on the first configure (this takes longer). Point `-DPICO_SDK_PATH` at your local SDK to avoid that.

### Build options

- `-DBUTTON_ENCODER=ON` — build for a button-based encoder instead of the default rotary encoder.

### Building in VS Code

With the Raspberry Pi Pico extension installed, open the folder and run **"Configure CMake"** from the command palette (once, or after editing `CMakeLists.txt`), then use the **"Compile Project"** build task. The bundled CMake/Ninja/toolchain under `~/.pico-sdk` are used automatically.

## Flashing

Hold the **BOOTSEL** button while plugging the Pico into USB, then copy the generated `.uf2` file (e.g. `build/picorx.uf2`) onto the `RPI-RP2` mass-storage drive that appears. The board reboots into the firmware automatically.

Alternatively use `picotool load build/picorx.uf2`, or flash over SWD with the VS Code "Flash" task.

## Controls

The receiver is operated with three buttons and a rotary encoder.

| Control | Action |
|---|---|
| Encoder (turn) | Tune frequency at the current step size |
| Menu + Encoder | Tune at 10× step |
| Back + Encoder | Tune at 1/10× step |
| Menu + Back + Encoder | Tune at 100× step |
| Encoder button + Encoder | Adjust volume |
| Encoder button + Menu + Encoder | Change demodulation mode |
| Encoder button + Back + Encoder | Adjust squelch |
| Menu button | Open main menu |
| Encoder button | Open memory recall directly |
| Back button | Cycle through display views |

Pressing **Back** past the last display view enters Memory Scanner mode (see [Scanning](#scanning)).

## Display views

Eight views are available; cycle through them with the **Back** button.

| View | Description |
|---|---|
| Original | Frequency, mode, signal strength, and narrow spectrum |
| Big Spectrum | Full-screen spectrum scope |
| Combined Spectrum | Status bar with large spectrum below |
| Waterfall | Scrolling waterfall display |
| Oscilloscope | Audio waveform |
| Status | Numeric status (frequency, mode, AGC, S-meter) |
| S-Meter | Analogue S-meter needle |
| Fun | Alternative layout |

## Menu reference

Open the main menu with the **Menu** button. Navigate with the encoder; confirm with **Menu** or the encoder button; cancel with **Back**.

### Main menu

| Item | Options / Range | Notes |
|---|---|---|
| Frequency | digit-by-digit | Set receive frequency (Hz) |
| Recall | channel 000–499 | Load a stored memory channel |
| Store | channel 000–499 | Save current settings to a memory channel |
| Volume | 0–9 | Audio output level |
| Mode | AM / AM-Sync / LSB / USB / FM / CW | Demodulation mode |
| AGC | Fast / Normal / Slow / Very Slow / Manual | AGC speed |
| AGC Gain | 0–60 dB (6 dB steps) | Manual AGC gain (active in Manual AGC mode) |
| Bandwidth | V Narrow / Narrow / Normal / Wide / Very Wide | IF filter width |
| Squelch | S0–S9+30 dB | Squelch open threshold |
| Squelch Timeout | 50 ms–5 s | How long the squelch stays open after the signal drops |
| Noise Reduction | → submenu | Spectral noise reduction |
| Impulse Blanker | Off / 3.0 / 2.8 / 2.6 / 2.4 / 2.2 / 2.0 | Impulse noise blanker threshold (σ) |
| Auto Notch | Off / On | Automatic notch filter |
| De-Emphasis | Off / 50 µs / 75 µs | FM de-emphasis time constant |
| Bass | Off / +5 / +10 / +15 / +20 dB | Bass boost |
| Treble | Off / +5 / +10 / +15 / +20 dB | Treble boost |
| IQ Correction | Off / On | Software IQ imbalance correction |
| Spectrum | → submenu | Spectrum display settings |
| Aux Display | Waterfall / SSTV | Content shown in the auxiliary panel |
| Band Start | digit-by-digit | Lower edge of the current tuning band (Hz) |
| Band Stop | digit-by-digit | Upper edge of the current tuning band (Hz) |
| Frequency Step | 10 Hz – 100 kHz | Encoder tuning step size |
| CW Tone Frequency | 100–3000 Hz | CW sidetone pitch |
| USB Stream | Audio / Raw IQ | Content sent over USB audio |
| HW Config | → submenu | Hardware and display settings |

### Noise Reduction submenu

| Item | Options | Notes |
|---|---|---|
| Enable | Off / On | Master on/off for spectral NR |
| Noise Estimation | Very Fast / Fast / Medium / Slow / Very Slow | Rate at which the noise floor is tracked |
| Noise Threshold | Adaptive / Low / Normal / High / Very High | Aggressiveness of noise suppression |

### Spectrum submenu

| Item | Range | Notes |
|---|---|---|
| Spectrum Zoom | 1–4 | Zoom factor applied to the spectrum display |
| Spectrum Smoothing | 1–4 | Temporal smoothing of the spectrum trace |

### HW Config submenu

| Item | Options / Range | Notes |
|---|---|---|
| Tuning Options | None / Tristate / Ground | Antenna switch control behaviour |
| Display Timeout | Never / 5 s / 10 s / 15 s / 30 s / 1 min / 2 min / 4 min | Auto-dim timeout |
| Regulator Mode | FM / PWM | Switching regulator mode (pin 23) |
| Reverse Encoder | Off / On | Swap encoder rotation direction |
| Encoder Resolution | Low / High | Match encoder detent count per revolution |
| Swap IQ | Off / On | Swap I and Q channels (mirrors the spectrum) |
| Gain Cal | 1–100 dB | Receiver gain calibration offset |
| Freq Cal | −100 – +100 ppm | Crystal frequency calibration |
| Flip OLED | Off / On | Rotate OLED 180° |
| OLED Type | SSD1306 / SH1106 | OLED controller IC — set to SH1106 for the common 1.3″ modules |
| Display Contrast | 0–15 | OLED brightness |
| TFT Settings | Off / Rotation 1–8 | Enable and orient the optional ILI934x colour TFT |
| TFT Colour | RGB / BGR | Colour byte order for the TFT panel |
| TFT Invert | Off / On | Invert TFT colours |
| TFT Driver | Normal / Alternate | TFT controller variant |
| Bands | → submenu | User-defined band boundary frequencies |
| IF Mode | Lower / Upper / Nearest | Which IF sideband the hardware uses |
| IF Frequency | 0–12000 Hz | IF offset in Hz (in 100 Hz steps) |
| External NCO | Off / On | Drive an external numerically-controlled oscillator |
| USB Upload | confirm prompt | Reboot into USB mass-storage bootloader for firmware update |
| Watchdog Test | Off / On | Intentionally hang the firmware to verify the watchdog resets it |

### Bands submenu

Defines the upper frequency boundary (in kHz) for each of 7 bands. The radio uses these boundaries to set the limits of the tuning range shown in the spectrum and to wrap the frequency when scanning.

| Item | Range |
|---|---|
| Band 1 ≤ | 0–31875 kHz |
| Band 2 ≤ | 0–31875 kHz |
| Band 3 ≤ | 0–31875 kHz |
| Band 4 ≤ | 0–31875 kHz |
| Band 5 ≤ | 0–31875 kHz |
| Band 6 ≤ | 0–31875 kHz |
| Band 7 ≤ | 0–31875 kHz |

## Scanning

Cycling past the last display view with the **Back** button enters two automatic scanning modes in sequence.

**Memory Scanner** — steps through the 500 stored memory channels, skipping blank slots and pausing for 3 seconds on any channel whose signal level exceeds the squelch threshold. The encoder adjusts scan speed (1–4 channels/s) and direction. **Menu** opens the main menu without leaving the scanner; **Back** exits to Frequency Scanner.

**Frequency Scanner** — sweeps through the current band one step at a time, pausing on signals above the squelch threshold. Speed and direction are controlled with the encoder. **Menu** opens the main menu; **Back** returns to the normal idle display.

## Credits

The project uses the Universal 8-bit Graphics Library (u8g2) by olikraus@gmail.com, the pico_ssd1306 display library by David Schramm, and the ILI934X display driver by Darren Horrocks.

Special thanks to **Mariusz Ryndzionek** (IQ imbalance correction, de-emphasis filter, USB audio, U8G2 integration, synchronous AM demodulation, and more), **Penfold42** (gain calibration, frequency scanning, spectrum zoom, multiple home-screen views, animated splash screen, multi-display support, and more), and **Robert Nickels (W9RAN)** and **Jim Reagan (W0CHL)** for extensive testing, debugging, feedback, and encouragement — along with everyone else who has contributed.

See [`COPYING.txt`](COPYING.txt) for licensing.
```

