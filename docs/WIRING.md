# Plantbot wiring plan

> **Building it?** Follow the illustrated step-by-step [direct-wiring build guide](build-guide.html). It uses soldered harnesses that plug onto the ESP32, with no breadboard or perfboard.

Target board: **ESP32-S3-DevKitC-1 N8R8** (8 MB flash, 8 MB octal PSRAM).

## 1. Pins we must NOT use on the N8R8

| GPIO | Why |
|---|---|
| 26–32 | Wired to the SPI flash |
| 35, 36, 37 | Used by the **octal PSRAM** (the "R8" in N8R8) |
| 0, 3, 45, 46 | Strapping pins (boot mode / flash voltage). If one is pulled the wrong way at reset, the board won't boot |
| 19, 20 | Native USB D-/D+ |
| 43, 44 | UART0, which carries the USB-serial console (logs and flashing) |
| 48 (or 38 on v1.1) | Onboard RGB LED. Usable, but we leave it as a status LED |

Analog input has to go on **ADC1 (GPIO 1–10)**, because ADC2 doesn't work while Wi-Fi is on, and we want Wi-Fi for NTP time and "quiet hours".

## 2. Pin map

Everything sits on the **left header (J1)**, so one side of the board takes all the wiring.

| Function | ESP32-S3 GPIO | Goes to | Notes |
|---|---|---|---|
| Soil moisture (analog) | **GPIO 4** (ADC1_CH3) | Moisture sensor `AOUT` | 12 dB attenuation, so the input range is about 0–3.1 V |
| I²C SDA | **GPIO 8** | BH1750 `SDA`, AHT20 white, VL53L0X white | Arduino-ESP32 default SDA on the S3 |
| I²C SCL | **GPIO 9** | BH1750 `SCL`, AHT20 yellow, VL53L0X yellow | Arduino-ESP32 default SCL on the S3 |
| E-paper CS | **GPIO 10** | EPD `CS` | Default FSPI pins, so this is the fastest path |
| E-paper MOSI | **GPIO 11** | EPD `DIN` / `SDA` | |
| E-paper SCK | **GPIO 12** | EPD `CLK` / `SCL` | |
| E-paper DC | **GPIO 13** | EPD `DC` | The display needs no MISO, so we reuse 13 |
| E-paper RST | **GPIO 14** | EPD `RST` | |
| E-paper BUSY | **GPIO 15** | EPD `BUSY` | Input |
| DFPlayer BUSY | **GPIO 16** | DFPlayer `BUSY` | LOW while a clip is playing |
| UART1 TX → player | **GPIO 17** | **1 kΩ resistor** → DFPlayer `RX` | The resistor cuts digital hiss in the speaker |
| UART1 RX ← player | **GPIO 18** | DFPlayer `TX` | |
| *(optional)* Touch pad | **GPIO 2** (TOUCH2) | Copper-tape pad / wire on the pot | A "poke me" pad that makes the plant talk on demand |

Free for later: GPIO 1, 5 (reserved for e-paper PWR if needed), 6, 7, 21, 38–42, 47.

## 3. Power

```
USB-C 5 V (≥ 2 A charger) ──► DevKitC "USB" or "UART" port
                                   │
             ┌─────────────────────┼──────────────────────────┐
         5V header pin        3V3 header pin (onboard LDO)    GND
             │                     │                          │
         DFPlayer VCC        all sensors + e-paper       everything
         (+ 470–1000 µF       (≈ 30–40 mA total)
          electrolytic and
          optional 100 nF
          across VCC/GND,
          close to module)
```

* **DFPlayer runs on 5 V.** That gives the 3 W / 4 Ω speaker its full swing. Its logic is still 3.3 V, so the UART is safe.
* **All sensors and the e-paper run on 3.3 V.** This matters most for the capacitive moisture sensor. At 5 V, some clones (the ones with an NE555 instead of a TLC555) can output more than 3.3 V into the ADC.
* The speaker draws current spikes on loud passages. The bulk capacitor plus a firmware volume cap (about 20/30) prevent brown-out resets. Use a real 2 A phone charger, not a laptop port through a long thin cable.

## 4. Component-by-component wiring

### 4.1 I²C bus (three sensors share 2 wires)

| Sensor | Address | Conflicts? |
|---|---|---|
| Adafruit BH1750 | `0x23` (ADDR left unconnected or tied to GND) | – |
| Grove AHT20 | `0x38` | – |
| Grove VL53L0X | `0x29` | – |

No address clashes, so everything goes in parallel on SDA/SCL. All three breakouts already have pull-up resistors, so don't add more.

**Grove cable colours:** black = GND, red = VCC, **white = SDA**, **yellow = SCL**.
To connect them you need **Grove-to-female-jumper cables** (or cut a Grove cable and crimp Dupont ends).

```
           3V3 ──┬──────────────┬───────────────┬──────────── 
           GND ──┼──┬───────────┼──┬────────────┼──┬───────── 
 GPIO 8  (SDA) ──┼──┼──┬────────┼──┼──┬─────────┼──┼──┬────── 
 GPIO 9  (SCL) ──┼──┼──┼──┬─────┼──┼──┼──┬──────┼──┼──┼──┬─── 
                 │  │  │  │     │  │  │  │      │  │  │  │
               VIN GND SDA SCL  red blk wht yel  red blk wht yel
               ┌───────────┐   ┌───────────┐    ┌────────────┐
               │  BH1750   │   │   AHT20   │    │  VL53L0X   │
               │ (Adafruit)│   │  (Grove)  │    │  (Grove)   │
               └───────────┘   └───────────┘    └────────────┘
```

Adafruit BH1750: use `VIN` (it has its own regulator), not `3Vo`. You can also use its STEMMA-QT socket with a STEMMA-QT-to-jumper cable.

### 4.2 Capacitive soil moisture sensor (v1.2 / v2.0 style)

| Sensor pin | Connects to |
|---|---|
| VCC | 3V3 |
| GND | GND |
| AOUT | GPIO 4 |

Calibration happens in firmware. Record the raw value in dry air and in a glass of water, then map between the two. Typically about 2.5 V dry and 1.2 V wet at a 3.3 V supply. Seal the exposed electronics at the top edge with nail polish, hot glue or heat-shrink. Water creeping in there is what kills these sensors. Push it into the soil only up to the printed line.

### 4.3 Waveshare 1.54" e-Paper Module (200×200 B/W)

Waveshare has sold two generations of this module:

| Version | How to tell | Controller | GxEPD2 driver class |
|---|---|---|---|
| **V2** (anything bought in recent years) | "V2" printed on the sticker on the back of the panel or on the PCB | SSD1681 | `GxEPD2_154_D67` |
| V1 (old stock) | no "V2" marking | IL3829 / SSD1608 | `GxEPD2_154` |

Use the Waveshare colour-coded cable that comes with the module:

| Module pin | Cable colour | Connects to |
|---|---|---|
| VCC | grey or red | 3V3 |
| GND | brown or black | GND |
| DIN | blue | GPIO 11 |
| CLK | yellow | GPIO 12 |
| CS | orange | GPIO 10 |
| DC | green | GPIO 13 |
| RST | white | GPIO 14 |
| BUSY | purple | GPIO 15 |
| PWR (only on newer 9-pin boards) | red/brown | 3V3, or GPIO 5 if we later want to cut the display's power in software |

> The cable colours vary between batches, so **go by the pin labels printed on the PCB**, not the colours.
> If your board has a 9th pin labelled **PWR**, it must be driven HIGH or the display stays blank. Tie it to 3V3.
> If there is a small **BS / interface switch**, set it to **4-line SPI** (`0`).
> This module is SPI, so do **not** put it on the I²C bus.

### 4.4 DFPlayer Mini (clone) + speaker

```
               DFPlayer Mini (top view, SD slot facing up)
            ┌──────────────────────────────┐
  5V  ──────┤ VCC                     BUSY ├────── GPIO 16
  GPIO17 ─[1kΩ]─┤ RX                  USB- │
  GPIO18 ───┤ TX                      USB+ │
            │ DAC_R                ADKEY2  │
            │ DAC_L                ADKEY1  │
  SPK+ ─────┤ SPK_1                    IO2 │
  GND  ─────┤ GND                      GND ├────── GND
  SPK- ─────┤ SPK_2                    IO1 │
            └──────────────────────────────┘
   + 470–1000 µF electrolytic (+ to VCC) (+ optional 100 nF) across VCC/GND
```

* The speaker goes **between SPK_1 and SPK_2**. Never connect either speaker wire to GND: the onboard amp is a bridged (BTL) design, and grounding one side can kill it.
* Clone modules (MH2024K, GD3200B, JL chips…) are sometimes quirky. In firmware we'll disable ACKs and fall back to the BUSY pin to tell when a clip has finished.

## 5. microSD layout (planned now, used by the firmware later)

FAT32, card ≤ 32 GB. The DFPlayer plays `/NN/MMM.mp3` with `playFolder(NN, MMM)`. Every "mood" gets its own folder full of variations, and the firmware picks one at random while avoiding recent repeats:

```
/01/  greetings / neutral small talk     001.mp3 … 0xx.mp3
/02/  thirsty (dry soil)
/03/  drowning (too wet)
/04/  too dark  (vampire mode)
/05/  sunny & happy (sunglasses mode)
/06/  too hot
/07/  too cold
/08/  too dry air
/09/  all good / cheerful
/10/  thank you (just watered / light improved)
/11/  poked (touch pad)
/12/  night / sleepy
```

Copy the files onto a freshly formatted card **in order** (the DFPlayer indexes files by copy order on some clones), and turn off macOS `._` hidden files if you copy from a Mac.

## 6. Shopping list for the bits that aren't on your list

* Female–female Dupont jumper wires, 20 cm (40 pcs). Cut in half, they become plug-ended "pigtails"
* Grove 4-pin cables (usually included with the Grove modules)
* Heat-shrink tubing assortment (1.5 / 2.5 / 6 / 12 mm)
* 1× **1 kΩ** resistor (DFPlayer RX)
* 1× **470 µF / 10 V** electrolytic capacitor (soldered across DFPlayer VCC/GND)
* microSD card (≤ 32 GB), formatted FAT32
* 5 V / 2 A USB-C power supply and cable
* Optional: copper tape for the touch pad, hot glue, zip ties

### Ground and 3V3 pin allocation (direct wiring)

| ESP32 pin | Feeds |
|---|---|
| J1-1 3V3 | Sensor harness (BH1750, AHT20, VL53L0X, moisture) |
| J1-2 3V3 | E-paper VCC |
| J1-21 5V | DFPlayer VCC |
| J1-22 GND | DFPlayer GND |
| J3-1 GND | Sensor harness |
| J3-21 GND | E-paper GND |

## 7. Bring-up order (test each block before adding the next)

1. Board alone: flash a blink + serial "hello".
2. I²C scan, which should find `0x23`, `0x29` and `0x38`.
3. Moisture ADC readings (calibrate dry/wet).
4. E-paper: draw a test face.
5. DFPlayer: play `/01/001.mp3` at low volume, then raise it while watching for resets.
6. Everything together, then on to the firmware.
