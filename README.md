<div align="center">

# BMS Cell Module — Isolated Single-Cell Battery Management Module

![Status](https://img.shields.io/badge/status-active-brightgreen)
![Tool](https://img.shields.io/badge/EDA-KiCad-orange)
![PCB](https://img.shields.io/badge/PCB-2%20Layer-blue)

</div>

A compact, per-cell Battery Management System (BMS) module for Lithium-ion cells, designed in **KiCad**. Each module monitors and passively balances a single cell, and communicates with a master controller (or other modules) over a **galvanically isolated I²C bus**, making it stackable across series-connected cells in a multi-cell pack without exposing the shared bus to full pack voltage.

---

## Overview

Traditional multi-cell BMS boards route every cell tap to one central board, which gets complicated (and dangerous) as series cell count grows. This project takes the opposite approach: **one small module per cell**.

Each module:
- Measures its own cell voltage
- Measures local temperature
- Passively balances (bleeds) its cell when instructed
- Talks to the rest of the stack over an **isolated I²C link**, so modules on different cells (at different absolute voltages) can safely share a common digital bus

This makes the architecture modular and scalable — add or remove cell modules without redesigning a monolithic board, and the isolation barrier protects the shared communication bus from the stacked cell voltages.

## How It Works

1. **Power** — The module taps its own cell directly through connector `J1` (`+BATT`). A Schottky diode (`D1`) protects against reverse polarity, and a fuse (`F1`) provides overcurrent protection.
2. **Regulation** — A REG710NA-3.3 LDO (`U1`) steps the raw cell voltage down to a clean 3.3 V rail (`+3.3V`) that powers the local MCU and logic.
3. **Sensing (Voltage)** — A resistor divider (`R6` 510 kΩ / `R7` 680 kΩ) scales the cell voltage into the ATtiny85's ADC range: a 1.8 V–4.5 V cell voltage maps to a 1.029 V–2.571 V signal at the ADC input (`VDIV`).
4. **Sensing (Temperature)** — An NTC thermistor (`TH1`, 10 kΩ B57891M0103K000) provides local temperature feedback for thermal protection/monitoring.
5. **Control** — An ATtiny85 MCU (`U3`) reads voltage and temperature, drives status LEDs, and switches the balancing MOSFET.
6. **Balancing** — When the cell needs to be bled down (passive balancing), the MCU drives the gate of MOSFET `Q1` (SI2312BDS-T1-E3) via `PB4/BYPASS`, connecting the cell through a 2 Ω / 10 W power resistor (`R1`) to dissipate excess charge as heat.
7. **Isolated Communication** — An ADUM1250ARZ digital isolator (`U2`) bridges a "strict" I²C side (facing the MCU/local bus) to a "relaxed" I²C side (facing the shared inter-module bus), providing galvanic isolation between modules sitting at different points in a series stack. `J2` and `J3` (both 4-pin connectors) daisy-chain the isolated bus between adjacent modules.
8. **Programming** — A standard 2×3 ISP header (`ISP1`) allows in-circuit programming of the ATtiny85 via SPI (MOSI/MISO/SCK/RST/VCC/GND).
9. **Status Indication** — Two onboard LEDs (`D2` blue, `D4` green) give simple visual status feedback.

## Block Diagram (Signal Flow)

```
        +BATT (cell)
           │
   ┌───────┼─────────────────────────────┐
   │  D1 (reverse protect) → F1 (fuse)    │
   │       │                              │
   │       ├── R6/R7 divider → VDIV ADC ──┼──► ATtiny85 (U3)
   │       │                              │        │  │  │
   │       ├── REG710-3.3 (U1) → +3.3V ───┼────────┘  │  │
   │       │                              │            │  │
   │       └── R1 (2R/10W) ── Q1 (MOSFET) │◄───PB4/BYPASS  │
   │                                      │               │
   │        TH1 (NTC) ────────────────────┼───────────────┘
   │                                      │
   │        D2/D4 (status LEDs) ◄─────────┤
   │                                      │
   │        ISP1 (SPI programming) ◄──────┤
   │                                      │
   │  U2 ADUM1250 (isolated I2C bridge)   │
   │   local I2C (strict) ◄──► MCU        │
   │   bus I2C (relaxed)  ◄──► J2 / J3 ───┼──► to adjacent cell modules
   └──────────────────────────────────────┘
```

## Schematic

<div align="center">
<img width="7016" height="4961" alt="BMS_Cell_Module-page-00001" src="https://github.com/user-attachments/assets/bf61324b-20b7-4437-b617-b8b404849817" />
</div>

## PCB Layout

<div align="center">
<img width="1248" height="788" alt="BMS PCB LAYOUT" src="https://github.com/user-attachments/assets/7d88f426-16d0-4b37-a403-69b52608e57c" />
</div>

## Repository Structure (planned)

> Note: This repository currently documents the design. Source files (schematic, PCB, BOM, Gerbers, firmware) are maintained separately and will be added incrementally.

```
BMS-Cell-Module/
├── README.md                # This file
├── hardware/
│   ├── schematic/            # KiCad schematic (.kicad_sch) + schematic image
│   ├── pcb/                  # KiCad PCB layout (.kicad_pcb) + layout image
│   ├── gerbers/               # Fabrication outputs
│   └── bom/                   # Bill of materials
├── firmware/                   # ATtiny85 firmware (balancing, ADC, I2C slave)
└── docs/                       # Datasheets, calculations, revision notes
```

## Bill of Materials (Key Components)

| Ref. Designator | Component | Function |
|---|---|---|
| **U1** | REG710NA-3.3 | 3.3 V LDO regulator — powers MCU and logic from raw cell voltage |
| **U2** | ADUM1250ARZ | 2-channel bidirectional digital isolator — isolates local I²C from the shared inter-module bus |
| **U3** | ATTINY85V-10SU (SOIC-8) | Main microcontroller — voltage/temp sensing, balance control, I²C slave |
| **Q1** | SI2312BDS-T1-E3 | N-channel MOSFET — switches the balancing (bleed) resistor path |
| **D1** | SS34 Schottky diode (40 V / 3 A) | Reverse-polarity protection on the raw battery input |
| **D2** | LED (Blue) | Status indicator |
| **D4** | LED (Green) | Status indicator |
| **F1** | Fuse | Overcurrent protection on the cell input |
| **TH1** | NTC Thermistor, B57891M0103K000 (10 kΩ) | Local temperature sensing |
| **R1** | 2 Ω / 10 W power resistor | Passive cell-balancing bleed resistor |
| **R6** | 510 kΩ | Voltage-divider resistor (battery sense, high side) |
| **R7** | 680 kΩ | Voltage-divider resistor (battery sense, low side) |
| **R2, R8, R9, R10, R12** | 4.7 kΩ | Pull-ups / I²C bus and gate-drive support resistors |
| **R3** | 10 kΩ | Thermistor bias/divider resistor |
| **R4** | 20 kΩ | Support resistor (thermistor network) |
| **C1** | 2.2 µF ceramic (X7R/X5R) | Input bulk/decoupling capacitor |
| **C2** | 2.2 µF ceramic (X7R/X5R) | Decoupling capacitor |
| **C3, C4, C5** | 0.1 µF ceramic | Local decoupling capacitors (regulator, MCU, isolator) |
| **C6** | 0.22 µF ceramic (X7R/X5R) | 3.3 V rail decoupling |
| **J1** | 2-pin connector | Battery cell input (+BATT / GND) |
| **J2, J3** | 4-pin connectors | Isolated I²C bus daisy-chain connectors (module-to-module) |
| **ISP1** | 2×3 pin header (odd/even) | AVR ISP programming header for the ATtiny85 |
| **TP1, TP2** | Test points | VDIV and GND test/debug points |

## Design Notes

- **Voltage sensing range:** The divider (R6/R7) is sized so a 1.8 V – 4.5 V cell range (safe low cutoff to safe high cutoff for typical Li-ion chemistries) maps linearly to a 1.029 V – 2.571 V ADC input, keeping the signal well within the ATtiny85's ADC input range with margin.
- **Balancing power dissipation:** At a nominal ~4.2 V full-charge voltage, the 2 Ω bleed resistor dissipates roughly 8.8 W — close to its 10 W rating, so balancing duty cycle should be managed in firmware (e.g., PWM or timed bursts) to avoid sustained full-power dissipation and excessive local heating.
- **Isolation rationale:** Because each module sits at a different absolute potential within a series stack, the ADUM1250ARZ prevents ground loops and protects the shared communication bus and any master controller from the stacked pack voltage — a standard requirement for scalable, stackable multi-cell BMS architectures.
- **Protection:** Reverse-polarity protection (D1) and fusing (F1) are placed directly at the cell input, ahead of all regulation and sensing circuitry.
- **Firmware role:** The ATtiny85 acts as an I²C slave, exposing cell voltage, temperature, and balance-control registers to a master BMS controller (e.g., an STM32-based pack controller), enabling closed-loop cell-level monitoring and balancing across the stack.

## PCB Layout Summary

- 2-layer PCB, labeled **B.M.S.** on the silkscreen (front and back)
- Component placement zones:
  - **Top-left:** voltage sense divider (R6/R7), status LEDs (D2/D4), fuse, and battery input connector (J1)
  - **Top-center:** ISP programming header and I²C bus connector (odd/even 2×3)
  - **Top-right:** isolator (U2/ADUM1250ARZ) and balancing MOSFET/sense resistor (Q1/R9)
  - **Bottom:** ATtiny85 MCU, LDO regulator, decoupling network, thermistor, and dual 4-pin daisy-chain connectors (J2/J3)
- Two mounting holes are provided in opposing corners for mechanical fixing.

## Possible Future Work

- PWM-based active balancing control instead of fixed on/off bleed
- Overcurrent/overtemperature hardware cutoff in addition to firmware limits
- Panelized/stackable mechanical design for multi-cell packs
- Master controller firmware/reference design for the pack-level side of the isolated I²C bus


