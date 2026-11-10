# ESP32 Racing Tachometer: Custom PCB

A custom ESP32 board designed in **KiCad 10** for my [ESP32 Racing Tachometer](../) project. It replaces the breadboard and dev board with a single PCB built around the bare **ESP32-WROOM-32E** module, with USB-C power, on-board USB programming, and the tachometer's display, throttle, and paddle-shift controls.

> **Status: work in progress.** The schematic and footprint assignment are complete. PCB layout is next.

<!-- Add a screenshot of the full schematic here (or link the PDF: docs/schematic.pdf) -->

## Progress

- [x] Architecture and part selection
- [x] Design calculations (regulator, RC filter, LED resistor, strapping pins, antenna placement)
- [x] Schematic, in 4 blocks
- [x] Electrical Rules Check: 0 errors (1 documented exclusion, see below)
- [x] Footprints assigned
- [ ] PCB placement
- [ ] Routing and ground pour
- [ ] Design Rules Check
- [ ] 3D render, Gerbers, and BOM

## Block Diagram

```
                 ┌──────────────┐
           ┌────►│  3.3V LDO    ├──────────────┐
           │     └──────────────┘              ▼
 ┌───────┐ │     ┌──────────────┐       ┌──────────────┐     ┌──────────────────┐
 │ USB-C ├─┴────►│  USB-UART    ├──────►│ ESP32-WROOM  ├────►│ Round display    │
 └───────┘       │  (CH340C)    │       │    -32E      │     │ Pot + RC filter  │
                 └──────┬───────┘       └──────▲───────┘     │ Shift buttons    │
                        ▼                      │             │ Shift LED        │
                 ┌──────────────┐              │             └──────────────────┘
                 │ Auto-program ├──────────────┘
                 └──────────────┘
```

## Schematic Blocks

### A. USB-C + Power
- **J1:** 16-pin USB-C receptacle (USB 2.0)
- **R1, R2 (5.1kΩ):** separate pull-downs on CC1 and CC2, so USB-C sources supply 5V
- **U2 (USBLC6-2SC6):** ESD protection on D+ and D−, placed right after the connector
- **U1 (AP2112K-3.3):** 5V → 3.3V LDO, with 10µF input and 10µF + 0.1µF output capacitors

### B. ESP32 Core
- **U3 (ESP32-WROOM-32E):** 22µF + 0.1µF decoupling at the 3V3 pin (per the datasheet's peripheral schematic)
- **Reset (EN):** 10kΩ pull-up + 1µF RC delay, plus a RESET button
- **Boot (IO0):** 10kΩ pull-up, plus a BOOT button
- Every unused pin is explicitly marked No Connect, including **IO12**

### C. USB-UART + Auto-Program
- **U4 (CH340C):** USB-to-serial bridge, running at 3.3V so its TX/RX levels match the ESP32. No external crystal needed.
- **Q1, Q2 (MMBT3904) + 10kΩ base resistors:** the standard cross-coupled auto-program circuit that drives EN and IO0 from DTR and RTS, so code uploads without pressing BOOT

### D. Tachometer I/O
- **J2:** 8-pin socket for the 1.28" GC9A01 round display (pin 1 = BLK)
- **RV1:** 10k throttle potentiometer with a 10kΩ / 1µF RC low-pass filter into the ADC
- **SW3, SW4:** upshift and downshift buttons (internal pull-ups enabled in firmware)
- **D1 + R8 (270Ω):** shift light

## GPIO Map

These are the same pins as the breadboard prototype, so the existing tachometer firmware runs without changes.

| GPIO | Net | Function |
|---|---|---|
| IO0 | `IO0` | Boot strap / BOOT button / auto-program |
| IO1 (TXD0) | `ESP_TX` | → CH340C RXD |
| IO3 (RXD0) | `ESP_RX` | ← CH340C TXD |
| IO2 | `LED_SHIFT` | Shift light |
| IO4 | `BTN_UP` | Upshift |
| IO21 | `BTN_DOWN` | Downshift |
| IO5 | `TFT_CS` | Display chip select |
| IO16 | `TFT_DC` | Display data/command |
| IO17 | `TFT_RST` | Display reset |
| IO18 | `TFT_SCK` | Display SPI clock |
| IO23 | `TFT_MOSI` | Display SPI data |
| IO34 | `THROTTLE` | Throttle (ADC1) |

## Design Decisions

### Regulator sizing (AP2112K-3.3)
- **Current budget:** about 440 mA worst-case peak (ESP32 Wi-Fi TX ≈ 380 mA, display ≈ 40 mA, CH340C ≈ 15 mA, LED ≈ 5 mA) against a 600 mA rating, for about 25% margin. This also meets Espressif's requirement of a supply that can deliver at least 500 mA.
- **Dropout:** 400 mV max at 600 mA, so the input needs at least 3.7V. USB's 5V (4.75V minimum) leaves headroom.
- **Thermal:** at 300 mA, P = (5 − 3.3) × 0.3 = 0.51 W. With θJA = 184 °C/W for SOT-23-5, that's about a 94 °C rise. That's acceptable, but it means the regulator needs copper pour for heat spreading. At this application's realistic draw without Wi-Fi (about 100 mA), the rise is only about 31 °C.

### Throttle RC filter
- R = 10kΩ and C = 1µF give f_c = 1 / (2πRC) ≈ **15.9 Hz**, with τ = 10 ms. That's fast enough for a throttle and slow enough to reject noise.
- The pot's own source impedance (up to R/4 = 2.5kΩ at mid-travel) adds in series, which shifts the cutoff to about 12.7 Hz at mid-position.
- The filter capacitor sits at the ADC pin, where it also supplies charge to the ADC's sampling capacitor.

### Strapping pins
| Pin | Use | Safe? |
|---|---|---|
| GPIO 0 | BOOT / auto-program | Pulled low on purpose for download mode |
| GPIO 2 | Shift LED (GPIO → R → LED → GND) | Yes. Stays low at boot, so download mode still works |
| GPIO 5 | Display CS | Yes. Its default pull-up matches CS idle, and it only affects SDIO timing |
| GPIO 12 | **Not connected** | Pulled high at boot, it would set the flash supply to 1.8V and prevent boot |

### Antenna placement
Following Espressif's hardware design guidelines, the module will sit on the board edge with the antenna overhanging, with no copper under the antenna and stitched ground vias beside it. USB-C, the CH340C, and the regulator go on the opposite edge to keep switching noise away from the antenna, and the display is kept out of the antenna area.

### Auto-program circuit
| DTR | RTS | EN | IO0 | Result |
|---|---|---|---|---|
| 1 | 0 | LOW | high | Reset |
| 0 | 1 | high | LOW | Enter download mode |
| 1 | 1 | high | high | Run |
| 0 | 0 | high | high | Run |

The cross-coupled transistors mean that when both lines are asserted together, which some serial terminals do when opening the port, the chip is **not** held in reset.

### Other choices
- **Shift LED resistor:** (3.3V − 2.0V) / 5 mA = 260Ω, so 270Ω. A red LED was chosen because a blue LED's ~3.0V forward drop leaves almost no headroom from 3.3V.
- **0805 passives:** easy to hand-solder and rework on a prototype with plenty of board space.
- **22µF bulk capacitor in 1206:** ceramic capacitors lose capacitance under DC bias, and a larger package derates much less than an 0805 at 3.3V.
- **CH340C in 3.3V mode:** V3 is tied to VCC per the datasheet. KiCad's symbol types V3 as a power output, which triggers an ERC "power output to power output" error against the LDO. This is documented as an ERC exclusion.

## Bill of Materials

| Ref | Part | Value | Footprint |
|---|---|---|---|
| U3 | ESP32-WROOM-32E (N4/N8, no PSRAM) | — | RF_Module:ESP32-WROOM-32E |
| U1 | AP2112K-3.3 | 3.3V LDO | SOT-23-5 |
| U2 | USBLC6-2SC6 | ESD | SOT-23-6 |
| U4 | CH340C | USB-UART | SOIC-16 |
| Q1, Q2 | MMBT3904 | NPN | SOT-23 |
| J1 | HRO TYPE-C-31-M-12 | USB-C | USB_C_Receptacle_HRO_TYPE-C-31-M-12 |
| J2 | 1×8 female header | 2.54mm | PinSocket_1x08_P2.54mm_Vertical |
| RV1 | Alps RK09K | 10k | Potentiometer_Alps_RK09K_Single_Vertical |
| SW1–SW4 | Tactile switch | 6mm | SW_PUSH_6mm |
| D1 | LED (red) | — | LED_0805 |
| R1, R2 | Resistor | 5.1kΩ | 0805 |
| R3–R7 | Resistor | 10kΩ | 0805 |
| R8 | Resistor | 270Ω | 0805 |
| C1, C2 | Capacitor (X5R/X7R, ≥16V) | 10µF | 0805 |
| C3, C5, C7 | Capacitor | 0.1µF | 0805 |
| C4 | Capacitor (X5R/X7R) | 22µF | 1206 |
| C6, C8 | Capacitor | 1µF | 0805 |

## Repository Contents

```
esp32-tachometer.kicad_pro    KiCad project
esp32-tachometer.kicad_sch    Schematic
esp32-tachometer.kicad_pcb    PCB (layout in progress)
docs/schematic.pdf            Schematic export
```

## References

- [ESP32-WROOM-32E Datasheet](https://documentation.espressif.com/esp32-wroom-32e_esp32-wroom-32ue_datasheet_en.pdf)
- [ESP Hardware Design Guidelines](https://docs.espressif.com/projects/esp-hardware-design-guidelines/en/latest/esp32/index.html)
- [AP2112 Datasheet](https://cdn.sparkfun.com/assets/2/e/d/f/2/AP2112_datasheet.pdf)

## Acknowledgments

Design guidance and review were provided with help from Claude (AI). The schematic capture, part selection, calculations, and KiCad work were done by me.

## Author

Michelle Xie
