# Handover: "Sleepy Eyes" children's clock

Firmware project for a child's bedside clock built on the **Waveshare ESP32-S3-DualEye-LCD-1.28**. The two round screens show a pair of eyes:

- **Eyes closed** during sleep time and **open** at wake-up time. This is the core feature: the clock tells the child whether it's OK to get up.
- When the eyes are open, they **follow the child** as they move in front of the clock, using an mmWave radar.
- The clock reacts to **hand claps** and a small set of **offline voice commands**.

Language: **Rust** (`std`, `esp-idf-svc` / `esp-idf-hal` / `esp-idf-sys`).

## 0. Platform (added in Claude Code, 13 Nov 2026)

This handover was written as if for a fresh `esp-idf-template` project. It is built instead on Bruce's own platform,
like his other ESP32 firmware:

- **bruce-embedded-std** for the platform, its features, configuration and scheduled tasks, and **bruce-esp-idf** for
  ESP-IDF, with the DualEye as a new board there, e.g. `boards/wave-esp32s3-dualeye-128`, giving its pins, both
  displays and the audio chips, as the other boards do. The clock is this repository's firmware on that board.
- **Flashing only with bruce-embedded-flash**, never `espflash` or `idf.py` directly: it checks the firmware was built
  for the board on the port, from the record each image carries. The ESP-SR models' separate `model` partition
  (section 3) needs the flasher to write a second image, which is work in bruce-embedded-flash.
- **Logging with the `log` macros**, never `println!`, over USB-Serial-JTAG (`CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y`),
  which keeps UART0, on the expansion header, free for the radar; the board crate sets this up.
- Rust formatted with `cargo fmt`, `max_width = 120`; MIT OR Apache-2.0; dependencies' licences checked before adding.

---

## 1. Hardware

### Main board: Waveshare ESP32-S3-DualEye-LCD-1.28
Docs: https://docs.waveshare.com/ESP32-S3-DualEye-LCD-1.28

- ESP32-S3R8: dual-core LX7 at 240 MHz, **8 MB PSRAM**, **16 MB flash**
- Wi-Fi 2.4 GHz and BLE 5
- **Two 1.28" round LCDs**, 240×240, 65K colours, with I2C touch
- Audio: **ES7210** ADC with an **onboard microphone**, **ES8311** codec, and a speaker header
- TF card slot, Li-ion battery header (MX1.25 2-pin) with a charger, USB-C
- **No onboard RTC and no IMU**
- GPIO1 is used for battery voltage sensing by default

### Pins (verified 13 Nov 2026)
Checked against Waveshare's schematic, `ESP32-S3-DualEye-LCD-1.28-Schematic.pdf`
(https://files.waveshare.com/wiki/ESP32-S3-DualEye-LCD-1.28/ESP32-S3-DualEye-LCD-1.28-Schematic.pdf), and its demo
code, https://github.com/waveshareteam/ESP32-S3-DualEye-Touch-LCD-1.28 (`examples/arduino/examples/`).

| Function | GPIO | Notes |
|---|---|---|
| LCD controller | — | **GC9A01** on both screens (the demo's `esp_lcd_gc9a01`), 240×240 |
| LCD SPI bus, shared | SCLK 41, MOSI (LCD_DIN) 42, MISO (LCD_DOUT) 40, DC 45 | SPI2; the demo runs it at 80 MHz |
| LCD1 (left board) | CS 47, RST 48, backlight 46 | Backlight through a MOSFET, active high, PWM dimmable |
| LCD2 | CS 38, RST 8, backlight 39 | As LCD1 |
| Touch 1, CST816 | I2C on 10/11, RST 4, INT 5 | Its I2C address is not in the demo; a CST816 is normally 0x15 |
| Touch 2, CST816 | own I2C, SCL 2, SDA 3 (I2C_NUM_1), RST 6, INT 7 | |
| Main I2C bus | SCL 10, SDA 11 (I2C_NUM_0) | Shared by touch 1, the ES8311 and the ES7210, and on the header. The schematic: "only for I2C, not for other functions" |
| ES8311 codec | I2C 0x18 | CE pulled low (Espressif driver's `ES8311_ADDRRES_0`) |
| ES7210 mic ADC | I2C 0x40 | The schematic's label and the demo's `ES7210_I2C_ADDR` agree |
| I2S | MCLK 12, SCLK 13, LRCK 14, ASDOUT 15 (ES7210 to ESP32), DSDIN 16 (ESP32 to ES8311) | The demo samples at 16 kHz, MCLK 256× |
| Speaker amplifier enable | PA_CTRL 9 | NS4150B amplifier |
| TF card, SDMMC 1-bit | CLK 17, D0 18, CMD 21 | Resistor options move it to SPI, sharing the LCD bus, if the board is broken apart |
| Battery voltage | GPIO1 | Through a 200k/100k divider, so a third of the battery voltage |
| BOOT key | GPIO0 | |
| UART0 | TXD 43, RXD 44 | On the expansion header |
| USB D-/D+ | 19, 20 | On the header too; the schematic warns that using them as GPIOs means entering download mode by hand for every flash |
| Not available | 33–37 | Used inside the module, for the octal PSRAM |

### 14-pin expansion header (on the LCD1 board, JST **SH 1.0 mm**, 14-pin)
| Pin | Signal | GPIO | Use in this project |
|---|---|---|---|
| 1 | USB_5V | — | Radar 5V |
| 2 | GND | — | Radar and RTC ground |
| 3 | D_N | 19 | — |
| 4 | D_P | 20 | — |
| 5 | 3V3 | — | RTC VCC |
| 6 | GND | — | — |
| 7 | SDA | 11 | DS3231 SDA |
| 8 | SCL | 10 | DS3231 SCL |
| 9 | UART_RXD | 44 | ← Radar TX |
| 10 | UART_TXD | 43 | → Radar RX |
| 11 | TP2_SDA | 3 | Touch 2's bus. **Do not use.** |
| 12 | TP2_SCL | 2 | Touch 2's bus. **Do not use.** |
| 13 | GPIO0 | 0 | Boot strap. Avoid. |
| 14 | RESET | — | — |

Notes:
- This UART is the board's **log/console port**, UART0. Move console output to USB-Serial-JTAG (`CONFIG_ESP_CONSOLE_USB_SERIAL_JTAG=y`) and make sure the bootloader and log output don't reach the radar. The ROM bootloader still prints a few lines on UART0 at reset, which the radar ignores, but the parser must resync after them.
- The header's SDA and SCL are the **main I2C bus**, shared with the codec, the microphone ADC and touch 1. The DS3231 at **0x68** does not clash with 0x15, 0x18 or 0x40.
- **USB_5V** comes from the USB-C socket's VBUS, so it is only live on USB power. The clock is intended to be mains/USB powered. On battery, the radar would need a 5V boost converter.

### Radar: Hi-Link HLK-LD2450 (24 GHz mmWave)
- Tracks **up to 3 targets** and reports X/Y position (mm) and speed.
- Range about 6 m, ±60° horizontal, ±35° vertical. Reports at about 10 Hz.
- Power **5V**, about 120 mA. UART logic is **3.3V**, so no level shifter is needed.
- UART **256000 baud, 8N1**. Use a hardware UART.
- Connector: tiny 4-pin socket, usually **1.25 mm (MX1.25 / PicoBlade-style)**. Some listings say 1.5 mm. The cable is made by cutting the radar's own cable and splicing it to an SH1.0 14-pin pigtail.
- Pin order varies between sources. **Go by the silkscreen** (5V / GND / TX / RX).
- Weakness: poor at detecting **stationary** people. The firmware must handle losing the target gracefully.

### RTC: DS3231 module (to be added)
- On I2C pins 7/8 at 3.3V, address **0x68**.
- Keeps time without Wi-Fi. NTP sets it whenever Wi-Fi is available.

---

## 2. LD2450 protocol (verify against the Hi-Link datasheet)

Data frame, 30 bytes:
```
AA FF 03 00 | T1 (8 bytes) | T2 (8 bytes) | T3 (8 bytes) | 55 CC
```
Each target has four little-endian u16 fields: `x`, `y`, `speed`, `distance_resolution`.
- `x`, `y`, `speed` use **sign-magnitude with an inverted sign bit**: if bit 15 is **set**, value = `raw - 0x8000` (positive); if it's **clear**, value = `-(raw)` (negative). Units are mm and cm/s.
- An all-zero target block means no target in that slot.
- `y` is the distance away from the sensor and `x` is left/right. **Check the sign convention** relative to the sensor's mounting orientation, and mirror it so the eyes look *at* the child, not away.

Parser requirements: resync on the header, validate the tail, tolerate partial reads, and never block the render loop.

---

## 3. Behaviour spec

### Schedule states
| State | Eyes | Notes |
|---|---|---|
| `Awake` | Open, tracking, blinking naturally | Daytime |
| `WindDown` (optional, e.g. 15 min before bed) | Droopy, slow blinks | Signals bedtime is coming |
| `Asleep` | Closed (maybe gentle "breathing" lids, dim backlight) | Night |
| `NearlyMorning` (optional) | One eye peeks occasionally | "Not yet, but soon" |
| `WakeUp` | Open with a cheerful animation | Wake-up time reached |

- Bedtime and wake time are configurable, with different times for weekdays and weekends.
- Default timezone **Europe/London**, with GMT/BST handled via POSIX TZ: `GMT0BST,M3.5.0/1,M10.5.0`.
- Backlight should be **very dim or off** in `Asleep`.

### Eye tracking
- Pick a target: the nearest moving target, with hysteresis so the eyes don't flick between people.
- Angle = `atan2(x, y)`, clamped to ±60°. Map it to a pupil offset within the eye's travel.
- Optional: use distance to adjust vergence (eyes cross slightly when the child is close) and pupil size.
- Smooth with a critically-damped spring or low-pass filter. **No snapping.**
- If the target is lost (radar weakness with still children): hold its last position for about 3 s, then drift back to centre, blink, and do idle glances.
- **No tracking while `Asleep`.** The eyes stay closed whatever the radar sees.

### Claps (onboard mic via ES7210)
- Keep an adaptive noise floor (slow RMS average).
- A clap is a short, sharp energy spike well above the floor that decays within about 50–100 ms.
- Recognise **patterns**: double clap, triple clap, with 0.2–0.8 s between claps.
- Suggested mapping (TBC with the owner): double clap → "say hello" (wink or eye roll); triple clap → show the time or a silly face.
- **At night, claps must not change the schedule or open the eyes.** At most a tiny response, such as a dim nightlight glow.
- Ignore the mic while the speaker is playing, if audio output is added.

### Voice (stretch goal): ESP-SR, offline only
- Pull in `espressif/esp-sr` through `esp-idf-sys` `extra_components` → `remote_component`, with raw bindings in a `sr` module.
- Wake word: one from the built-in WakeNet9 list. **"Hi Joy"** (`wn9_hijoy_tts`) or **"Sophia"** suit a child's clock.
- Commands: English MultiNet7 (quantised), up to about 300 phrases, added at runtime from text. Initial set: "good night", "good morning", "what time is it", "how long until morning", "open your eyes", "do a silly face".
- Models live in a separate **`model` flash partition (`srmodels.bin`)**. This needs a custom partition table and a separate flash step, which bruce-embedded-flash must learn to do, see section 0.
- Expect lower accuracy with young children's voices, and with a single mic beyond 1–2 m.
- **No audio ever leaves the device.** No cloud speech.
- Prove the pipeline first with Espressif's C example (`esp-skainet/examples/en_speech_commands_recognition`) before porting it to Rust.

---

## 4. Suggested architecture

Threads or tasks communicate over channels. One owner per peripheral.

```
radar_task  (UART)      ──► TargetUpdate ──┐
audio_task  (I2S, core1) ──► Clap / Voice ─┤
clock_task  (RTC/NTP)    ──► Tick/State ───┼──► app_state (state machine)
settings    (NVS)        ──────────────────┘          │
                                                      ▼
                                   render_task (both LCDs, ~30–60 fps)
```

- `render_task`: draws both eyes procedurally (sclera, iris, pupil, eyelids as masks) into PSRAM framebuffers and pushes them over SPI with DMA. Render only the changed regions if bandwidth is tight. Each eye is drawn separately so they can wink, cross or look different.
- `audio_task`: needs a big stack and runs pinned to the other core from render. Clap detection always runs. ESP-SR runs only when enabled.
- `app_state`: a pure state machine with no I/O, unit-testable on the host.
- Settings (times, volume, features on/off) are stored in NVS. Configuration could later come through a simple Wi-Fi web page or BLE.

Keep the logic (`eyes` maths, LD2450 parser, clap detector, schedule) in **`no_std`-friendly, hardware-free modules** with host-side `cargo test` coverage.

---

## 5. Milestones

1. The DualEye as a board in bruce-esp-idf, and this firmware on bruce-embedded-std starting on it, logging over
   USB-Serial-JTAG, flashed with bruce-embedded-flash.
2. Bring up both LCDs and draw static eyes, with the pins above and the GC9A01 init sequence from Waveshare's demo.
3. Procedural eye animation: blink, look around, open/close.
4. DS3231 and NTP. Schedule state machine driving open/closed eyes. **This milestone is a usable clock.**
5. LD2450 parser (host tests first), then live tracking.
6. Clap detection.
7. ESP-SR voice commands (stretch).
8. Settings UI, night dimming, enclosure details (mic hole, radar window — plastic is fine, metal isn't).

---

## 6. The owner's answers (13 Nov 2026)

The principle throughout: start simple, then iterate.

- **Bedtime and wake-up times**: separate times for every day of the week, with a way to set all the weekdays at once
  and both weekend days at once. Later, perhaps, the clock could know the calendar and do the right thing on days
  without school.
- **Settings**: through the settings page the platform already serves. The clock is just another platform feature,
  with its own settings. Setting the five weekdays or the two weekend days at once may need something new in the
  platform's settings page.
- **Power**: USB to start with. Battery is a nice-to-have, later (see the USB_5V note: the radar would need a boost
  converter on battery).
- **Sound**: yes. The board comes with a small speaker. Each clock has its own set of WAV files, recorded for it, e.g.
  the child's mum's voice or a cartoon character, and which is played when is configurable. That needs somewhere to
  keep them, e.g. the TF card or a flash partition, and a way to put them there, e.g. uploading through the settings
  page, which the platform does not do yet.
- **Claps and voice commands**: to be decided once the hardware is here and we can see how reliable they are.
- **Eye style**: start with what is there, e.g. Waveshare's demo, `examples/esp-idf/LCD_03_Animated_Eye1`, which
  draws the eyes in code: a white circle, a brown iris and a pupil, moved together, with a blink. Later the owner's
  daughter, an artist, may draw custom eyes, so the drawing should take artwork, see below.

### Eye artwork, for later

Drawing every direction an eye can look is not needed. The usual way, e.g. Adafruit's "Uncanny Eyes", builds each eye
from a few layers, each drawn once:

- **The sclera**, the white of the eye with its veins or colour, drawn a little larger than the screen, so that the
  part shown can move with the iris, as an eyeball turns.
- **The iris and pupil**, drawn once, looking straight ahead, and moved across the sclera. Near the edge it can be
  squashed into an ellipse, which is what makes a flat picture look like a turning ball, and is cheap to do. The pupil
  can be scaled on its own, wider in the dark, narrower when close.
- **The eyelids**, upper and lower, as shapes or masks, drawn closed, open and in between, or as one shape moved down
  over the eye, for blinks, sleepiness and winks.
- **A highlight**, the reflection of a light, which stays still while the eye moves, which makes it look wet and
  alive.

So the artist draws a set of layers to a template: sizes, the centre and the transparent parts. The firmware moves
and squashes them. A cartoon or animal eye is a different set of layers, the same code.

---

## 7. Safety and privacy constraints

- Everything runs offline. No audio, radar or usage data leaves the device.
- No camera.
- The clock must never encourage waking at night: night interactions are minimal and dim.
- Keep the radar and wiring enclosed. Strain-relieve the tiny connectors with hot glue or cable ties.
