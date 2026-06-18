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

## Credits

The project uses the Universal 8-bit Graphics Library (u8g2) by olikraus@gmail.com, the pico_ssd1306 display library by David Schramm, and the ILI934X display driver by Darren Horrocks.

Special thanks to **Mariusz Ryndzionek** (IQ imbalance correction, de-emphasis filter, USB audio, U8G2 integration, synchronous AM demodulation, and more), **Penfold42** (gain calibration, frequency scanning, spectrum zoom, multiple home-screen views, animated splash screen, multi-display support, and more), and **Robert Nickels (W9RAN)** and **Jim Reagan (W0CHL)** for extensive testing, debugging, feedback, and encouragement — along with everyone else who has contributed.

See [`COPYING.txt`](COPYING.txt) for licensing.
```

