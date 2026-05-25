# picoTracker (RP2040) Hardware Reference

## SoC
- **Raspberry Pi RP2040** — dual-core ARM Cortex-M0+ @ up to 133 MHz (overclocked to 220.5 MHz in firmware)

## Clock Configuration

| Clock | Frequency | Source |
|-------|-----------|--------|
| `clk_usb` | 48 MHz | PLL_USB (1536 MHz ÷ 4 ÷ 4) |
| `clk_sys` | 220.5 MHz | PLL_SYS (882 MHz ÷ 4 ÷ 1) |
| `clk_peri` | 96 MHz | PLL_USB |

- **PLL_USB**: REF=1, FBDIV=1536, PD1=4, PD2=4 → VCO 1536 MHz
- **PLL_SYS**: REF=2, FBDIV=147, PD1=4, PD2=1 → VCO 882 MHz (overclocked)
- Core voltage: default (not raised), though commented-out configs show 1.10V–1.20V for higher overclocks

---

## Display — ILI9341 (240×320 RGB TFT)

| Signal | GPIO | Peripheral |
|--------|------|------------|
| CS | 20 | GPIO (active-low) |
| DC (RS) | 21 | GPIO (high=command, low=data) |
| RESET | 22 | GPIO (active-low) |
| SCK | 26 | SPI1 |
| MOSI | 27 | SPI1 |
| MISO | 28 | SPI1 |
| LED (backlight PWM) | 23 | PWM (channel B, 8-bit: 0–255) |

- **SPI bus**: `spi1`, init at 0.5 MHz → boosted to 75 MHz for actual transfers
- Backlight: PWM slice auto-detected from GPIO 23, divider = 220.5K, period = 255

---

## Audio — I2S via PIO0 (Mono, 16-bit samples → 32-bit output stream)

| Signal | GPIO | Peripheral |
|--------|------|------------|
| SDATA (data out) | 17 | PIO0, pin 1 (out) |
| BCLK (bit clock) | 18 | PIO0, sideset pin 0 |
| LRCLK (word select) | 19 | PIO0, sideset pin 1 |

- **PIO**: `pio0`, state machine 0
- **DMA**: channel 0, IRQ 0
- **Sample rate**: 44100 Hz (clock divider tuned for 220.5 MHz sysclk)
- **DREQ**: `DREQ_PIO0_TX0`
- **Output format**: 32-bit I2S words — real 16-bit sample data is preceded by sign-extension (MSB + copies) and followed by zero-padding. The firmware pads 6 bits from MSB and backfills 7 zeros.
- **Volume attenuation**: configurable via PIO instruction patching (3 levels: default, headphone-high, line-level)

---

## SD Card — SDIO 4-bit via PIO1

| Signal | GPIO | Peripheral |
|--------|------|------------|
| CLK | 2 | PIO1 |
| CMD | 3 | PIO1 (in/out/set/jmp) |
| D0 | 4 | PIO1 (input) |
| D1 | 5 | PIO1 |
| D2 | 6 | PIO1 |
| D3 | 7 | PIO1 |

- **PIO**: `pio1`, 2 state machines:
  - `SDIO_CMD_SM` (0): command/response + clock generation (sideset on CLK)
  - `SDIO_DATA_SM` (1): 4-bit data RX/TX
- **DMA**: channels 4 (RX) and 5 (chaining for RX block descriptors)
- **Block size**: 512 bytes
- **Bus width**: 4-bit SDIO
- **Clock divider**: 7 (→ ~17.8 MHz from 125 MHz base)
- Input synchronizer bypassed on all 6 pins
- All pins have pull-ups

---

## MIDI — UART0 (Standard MIDI, 31250 baud)

| Signal | GPIO | Peripheral |
|--------|------|------------|
| MIDI OUT | 0 | UART0 TX |
| MIDI IN | 1 | UART0 RX |

- **Baud**: 31250 (standard MIDI)
- **Format**: 8N1
- FIFOs disabled (character-by-character processing)
- No hardware flow control (CTS/RTS off)
- No CRLF translation

---

## Buttons / Inputs — 9 GPIOs, active-low, pull-up

| Button | GPIO |
|--------|------|
| LEFT | 8 |
| DOWN | 9 |
| RIGHT | 10 |
| UP | 11 |
| ALT | 12 |
| EDIT | 13 |
| ENTER | 14 |
| NAV | 15 |
| PLAY | 16 |

- All configured as inputs with pull-ups enabled
- Key state read via `(~gpio_get_all() & 0x0001FF00) >> 8` — returns a 9-bit bitmask
- Bit 0 = LEFT, bit 8 = PLAY

---

## USB — TinyUSB (Device mode)

- **MIDI class**: enabled (CFG_TUD_MIDI=1), 64-byte TX/RX buffers
- **CDC (serial)**: enabled (CFG_TUD_CDC=1), 64-byte TX/RX buffers
- **MSC/HID/VENDOR**: disabled
- **Endpoint 0**: 64 bytes
- CDC used for `picoRemoteUI` (remote keyboard input over serial)

---

## Debug UART — UART1

- Defined as `DEBUG_UART uart1` (no pin assignments in gpio.h — likely USB CDC serves as debug)

---

## Battery / Power

- `BATT_VOLTAGE_IN` = 29 (ADC channel number, though `battery_health()` currently returns -1 / unimplemented)
- No ADC implementation present in current code

---

## Resources Summary

| Resource | Usage |
|----------|-------|
| **PIO0** | SM0 → I2S audio |
| **PIO1** | SM0 → SDIO CMD/CLOCK, SM1 → SDIO DATA |
| **SPI1** | ILI9341 display |
| **UART0** | MIDI IN/OUT |
| **UART1** | Debug (unused) |
| **DMA CH0** | I2S audio TX |
| **DMA CH4** | SDIO data RX |
| **DMA CH5** | SDIO DMA descriptor chaining |
| **DMA IRQ0** | Audio ISR |
| **DMA IRQ1** | SDIO TX completion ISR |
| **Core 0** | Main application |
| **Core 1** | Audio buffer processing thread |
| **PWM** | Display backlight (GPIO 23) |
| **Watchdog** | Reboot mechanism |
| **USB BOOTROM** | Bootloader entry via `reset_usb_boot()` |

---

## Platform Functions

- `platform_init()` — clocks, display, MIDI, buttons, audio
- `platform_reboot()` — watchdog reboot
- `platform_bootloader()` — USB bootrom entry
- `platform_brightness(uint8_t)` — sets PWM backlight level (0–255)
- `millis()` / `micros()` — time via `absolute_time_t`
