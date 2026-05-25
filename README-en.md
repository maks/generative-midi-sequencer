# Generative Techno MIDI Sequencer

## Project Overview

This project is a hardware-optimized **4-channel generative MIDI sequencer** that leverages the dual-core architecture of the Raspberry Pi Pico (RP2040). Inspired by the classic Eurorack module "Turing Machine," it combines stochastic shift registers (S&H emulation) with Bjorklund algorithm-based Euclidean rhythm generation to autonomously produce accidental yet highly musical techno and minimal sequences.

The firmware is optimized for repurposing surplus **[Pico Tracker hardware (3.2-inch ILI9341 LCD, 9 Kailh Choc V1 key switches, PCM5102 I2S DAC)](https://github.com/ijnekenamay/picotracker_alt-pcb)**, and can be operated in real-time from an ergonomically designed high-contrast monochrome UI.
![UI](uiai.png)

---

## 🛠 Core Feature Specifications (Current Implementation)

### 1. 4ch Independent MIDI Sequencer & Clock Sync

- **4 fully independent tracks**: All tracks operate as independent sequence engines (`Track 1` ~ `Track 4`).
- **Per-track clock divide (multiplication) settings**:
  For each track, you can select and set individual multipliers (`x1`, `x2`, `x3`, `x4`, `x6`, `x8`, `/2`, `/3`, etc.) from the grid relative to the master clock. This is extremely powerful for building polyrhythms.
- **Per-channel step length (LEN)**: The sequence length can be changed in real-time for each channel from `1` to `32` steps.
- **Standard MIDI clock output**: Outputs standard serial MIDI signals at 31,250 bps via UART0 (GP0 TX) (supports MIDI Start / Stop / Clock sync).

### 2. Generative Rhythm & Pitch Generation Engine

- **Euclidean Rhythm Generator (`EuclideanGenerator`)**:
  A lightweight, real-time computable Euclidean pattern generator based on the Bresenham algorithm. Controlled by three parameters: Step Count (Length), Pulse Count (Density), and Start Shift (Shift).
- **Stochastic Shift Register Pitch Generator (`NoteGenerator`)**:
  Contains a 16-bit shift register (initial value `0x9E37`). The **Mutation** parameter allows continuous control from 0% to 100% of whether the register's bits loop as-is or mutate via hardware random number generation (Pico's `get_rand_32()`).
- **High-Precision Scale Quantizer**:
  Generates random CV values quantized to the selected scale and root note (Root Note). Five scales can be instantly switched:
  1.  **Chromatic**
  2.  **Natural Minor**
  3.  **Phrygian** (dark, for acid techno)
  4.  **Dorian** (deep techno)
  5.  **Minor Pentatonic** (classic blue notes)

### 3. Signature Groove Generation (Groove & Microtiming)

- **Per-channel microtiming jitter (JIT)**:
  Jitter can be adjusted from 0% to 100% per channel. Using the Pico's random number generator and microsecond timer, an asynchronous scheduling engine randomly delays note-on timing by milliseconds, breaking mechanical timing to produce extremely organic, "swinging" groove (swing/drunk feel).
- **Stochastic asynchronous gate length control (GAT)**:
  Gate length for each step can be set individually from 10% to 100%. Dynamically converts step width to microseconds linked to BPM, and adds up to 33% stochastic length variation (fluctuation emulation), scheduling note-offs asynchronously. Produces expressive grooves ranging from staccato ("tat-ta-tan") to legato ("ta-ta-ta-ta-ta").
- **Real-time adjustable master BPM**:
  Tempo can be adjusted in real-time from `40` to `250`. The Core 1 sequence timer follows tempo changes instantly with microsecond precision.

### 4. Multi-Core IPC Design to Unleash Hardware Potential

- **Spinlock-based shared memory (Spinlock IPC)**:
  Communication between Core 0 (UI/rendering thread) and Core 1 (ultra-low-latency real-time sound/MIDI transmission) uses RP2040 hardware spin locks (`spin_lock_t*`) for IPC (Inter-Core Communication). This guarantees zero data races and zero data tearing during screen updates or user input configuration changes, maintaining a fully synchronized, low-jitter playback environment.
- **Analog clock sync output (via I2S DAC)**:
  For perfectly synchronized performance with external analog gear, an analog clock (trigger pulse) 100% synced to the master clock is output from the audio I2S DAC (GP26 Data, GP21 BCLK, GP22 LRCK). Ultra-low-latency operation via a PIO-based stereo 16-bit I2S driver and a 22.05 kHz background timer interrupt.
- **Safe multicore lockout flash save**:
  Configuration data is saved directly to the final sector (`0xFFF000`) of the Pico's built-in 16MB QSPI flash memory. During write execution, Core 0 calls the standard Pico SDK Multicore Lockout API (`multicore_lockout_start_blocking` / `end_blocking`) to completely pause Core 1, achieving a highly robust safety design that perfectly prevents CPU hangs (`HardFault`) caused by code execution from XIP flash.

### 5. Ergonomically Refined Monochrome UI

- **Header area**:
  - **"BPM display"** with maximized space efficiency (dynamically variable in real-time).
  - Large tempo value display (16x24 bold font) that **flashes negative/positive** in perfect sync with beats.
  - Pixel art icons showing current status (play ▶ / stop ■ / disk 💾 / note 🎵).
- **8-column parameter grid (LEN, DEN, SHF, MUT, JIT, GAT, ROT, SCL)**:
  - Leverages the high-resolution 320x240 screen to display all 8 parameters in a stylish single row.
  - Clock multiplier text (e.g., `x4`) **flashes negative/positive** in sync with each channel's note trigger hits.
  - Selected parameters are **highlighted with white background and black text** (unselected parameters have dark gray outlines).
  - **MUTE dark-out**: Pressing the `RT` button immediately mutes the corresponding channel, the label changes to `MUTx`, and the entire row dims (disabled display). Sound production is also muted, but the step indicator (playhead) continues to move, so synchronization status is always visible.

---

## 🎛 Physical Key Mapping (9-Key Controls)

A shortcut system using a modifier key (Shift) is implemented to enable comfortable and fast operation with just 9 key switches.

| Physical Key Name | Pico GPIO | Single Press Action                  | `LT` (Shift) Simultaneous Press Action |
| :---------------- | :-------- | :----------------------------------- | :------------------------------------- |
| **UP**            | GP5       | Move cursor to the track above       | Increase global BPM by **+5**          |
| **DOWN**          | GP3       | Move cursor to the track below       | Decrease global BPM by **-5**          |
| **LEFT**          | GP2       | Move cursor to the left parameter    | -                                      |
| **RIGHT**         | GP4       | Move cursor to the right parameter   | -                                      |
| **A**             | GP8       | Increase selected parameter by **+1**| Increase selected parameter by **+10** |
| **B**             | GP7       | Decrease selected parameter by **-1**| Decrease selected parameter by **-10** |
| **RT** (Page)     | GP6       | **MUTE / UNMUTE toggle**             | -                                      |
| **PLAY**          | GP10      | Pause / Resume sequence              | **Save current settings to flash (SAVE)** |
| **LT** (Shift)    | GP9       | Hold as modifier (Shift key)         | -                                      |

---

## 🔌 Physical Pin Assignment (Hardware Pinout)

Connection wiring map for Pimoroni Pico LiPo (or standard Raspberry Pi Pico).

```text
       [ Pimoroni Pico LiPo ]
         +-----------------+
    TX0  | [GP0]     [VBUS] | ---> DAC 5V Power (VIN)
    RX0  | [GP1]     [VSYS] |
   Left  | [GP2]    [3V3_O] | ---> Display / MIDI / Common 3.3V
   Down  | [GP3]     [GND]  | ---> Common GND
  Right  | [GP4]    [GP28]  |
     Up  | [GP5]    [GP27]  |
     RT  | [GP6]    [GP26]  | ---> DAC I2S DATA
      B  | [GP7]     [RUN]  |
      A  | [GP8]    [GP22]  | ---> DAC I2S LRCK
     LT  | [GP9]    [GP21]  | ---> DAC I2S BCLK
   Play  | [GP10]   [GND]   |
  TFT_CS | [GP11]   [GP20]  |
 TFT_SCK | [GP12]   [GP19]  | ---> TFT_MOSI (SPI0)
TFT_MISO | [GP13]   [GP18]  |
 TFT_DC  | [GP14]   [GP17]  | ---> TFT_RST
         | [GP15]   [GP16]  |
         +-----------------+
```

_(※ For detailed hardware verification and design notes, refer to `PINOUT.md`)_

---

## 📂 Directory Structure (Workspace Layout)

```text
generative-midi-sequencer/
├── CMakeLists.txt              # Root CMake build configuration
├── pico_sdk_import.cmake       # Pico SDK import definition
├── README.md                   # This documentation
├── README-en.md                # English documentation (this file)
├── ARCHITECTURE.md             # Software architecture detailed documentation
├── PINOUT.md                   # Hardware wiring & verification documentation
├── src/
│   ├── CMakeLists.txt          # Source build & library link settings
│   ├── main.cpp                # Entry point (Core 0/1 initialization and binding)
│   ├── engine/
│   │   ├── track.hpp / .cpp         # Track management (sequence progression & MIDI note control)
│   │   ├── euclidean.hpp / .cpp     # Euclidean rhythm calculation (Bresenham algorithm)
│   │   ├── note_generator.hpp / .cpp # Turing Machine emulation & scale quantization
│   │   ├── midi_handler.hpp / .cpp   # UART0 31.25kbps serial MIDI transmission control
│   │   └── storage_manager.hpp / .cpp # Internal flash non-volatile save/load
│   └── ui/
│       ├── display_controller.hpp / .cpp # SPI ILI9341 LCD driver (8x8 font built-in)
│       ├── input_manager.hpp / .cpp       # 9-key GPIO debounce & long-press scanner
│       └── custom_assets.hpp              # Large BPM font, pixel art icon set
```

---

## 🚀 Build & Flash Instructions

### Prerequisites

1. **Raspberry Pi Pico C/C++ SDK** installed on your host PC with the `PICO_SDK_PATH` environment variable properly set.
2. `arm-none-eabi-gcc` toolchain and `CMake` installed.

### Build Steps

From the project root directory, run the following commands:

```powershell
# 1. Create build directory
mkdir build
cd build

# 2. Configure CMake project
cmake ..

# 3. Compile
cmake --build . --config Release
```

### Flashing Steps

#### Via Bootloader

On successful build, the following compiled artifact is generated in the `build/src/` directory:

- **`generative_midi_sequencer.uf2`**: **Device flash binary**

Connect the Pico in **BOOTSEL mode** (hold the Pico's BOOTSEL button while connecting via USB to PC), and drag & drop the `generative_midi_sequencer.uf2` file onto the virtual drive (`RPI-RP2`) that appears on your PC. It will automatically flash, and the sequencer will boot.

#### Via OpenOCD

If you have a picoprobe you can flash via OpenOCD:
```
openocd -f interface/cmsis-dap.cfg  -f target/rp2040.cfg   -c "adapter speed 5000" -c "program src/generative_midi_sequencer.elf verify reset exit"
```

