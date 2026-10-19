# GB-LINK-USB

**An open-source USB adapter for the Game Boy Link Cable.**

![GB-LINK-USB top view](extras/gb-link-usb-top.png)

GB-LINK-USB is an RP2040-based hardware adapter that connects original Nintendo
Game Boy, Game Boy Color, and Game Boy Advance systems to a computer over USB.
Together with the open-source
[GBLink-Firmware](https://github.com/GB-Link/GBLink-Firmware), it brings modern
connectivity and community-built tools to original Game Boy hardware.

This hardware was developed with reference to the
[GB-Link-USB-DIY](https://github.com/GB-Link/GB-Link-USB-DIY) project and is
designed to be compatible with the GBLink firmware and its web-based clients.

> [!NOTE]
> GB-LINK-USB is the hardware interface. Available features depend on the
> installed firmware, client software, selected cable, game, and firmware
> version.

## What Can It Do?

With compatible firmware and client software, GB-LINK-USB can provide the
following functions:

| Mode | Description |
| --- | --- |
| **GBA online link** | Bridges two original Game Boy Advance systems over the Internet through a compatible client and server. |
| **Pokémon Gen III trade emulation** | Emulates a GBA link partner for supported Pokémon Generation III trading workflows. |
| **GB/GBC link passthrough** | Transfers Game Boy serial-link data between the console and a computer, enabling supported community clients and online play projects. |
| **Game Boy Printer emulation** | Receives images sent by compatible games so they can be viewed or saved on a computer without an original Game Boy Printer. |
| **Advance Wars link mode** | Supports communication features for *Advance Wars* and *Advance Wars 2*. |
| **Nintendo e-Reader emulation** | Emulates supported Nintendo e-Reader communication workflows. |

Depending on the frontend and firmware version, additional features such as
GBA multiboot transfers may also be available.

The firmware handles the timing-sensitive communication with the console
locally. The computer-side client can then exchange data with another adapter,
an online service, or an emulator without requiring the two consoles to be
connected by one physical Link Cable.

## Hardware Test Photos

The following photos show GB-LINK-USB working with original Game Boy hardware.

### Game Boy Printer Emulation

A Game Boy Color running *Pocket Camera* sends an image through GB-LINK-USB to
the browser-based Game Boy Printer emulator, where the received image can be
previewed and downloaded.

<p align="center">
  <img src="extras/gbc.jpg" alt="Game Boy Color Pocket Camera printer emulation test" width="480">
</p>

### GBA Multiboot and Cartridge Communication

These tests show a cartridge utility transferred to an original Game Boy
Advance through the Link Cable interface, followed by cartridge-data transfer
between the GBA and the browser client.

| GBA multiboot utility running on original hardware | Cartridge-data transfer through the browser client |
| --- | --- |
| ![GBA multiboot hardware test](extras/gba1.jpg) | ![GBA cartridge data transfer test](extras/gba2.jpg) |

## Hardware Features

- Integrated RP2040 microcontroller
- USB Type-C connection
- Game Boy Link interface
- Bidirectional level translation on the Link lines to the console
- Firmware-controlled 3.3 V and 5 V Link-interface reference voltage
- WS2812B RGB status indicator
- BOOTSEL firmware-update support
- SWD programming and debugging connection
- Four-layer, 1.2 mm PCB with an inner ground plane and inner power pours
- Complete KiCad 10 design files
- Manufacturing Gerber package

## Hardware Revision History

| Revision | Status | Summary |
| --- | --- | --- |
| **v1.4** | current | Regulated Link-interface reference with a TPS2116 power mux, level-shifted Link pin 1 line, four-layer PCB |
| **v1.0** | superseded | Two-layer PCB, Link pull-up rail taken from the connected console |

The v1.0 design files are kept under `hw/v1.0/` unchanged for reference.

## Changes in v1.4 (relative to v1.0)

v1.4 is a functional revision of the Game Boy Link interface plus a PCB
stack-up change. The RP2040 core, USB Type-C front end, QSPI flash, crystal,
3.3 V LDO, VBUS diode, status LED, BOOTSEL button, SWD header and the
23.2 mm × 36.05 mm board outline are carried over from v1.0. The USB-C
connector, the Link port and the SWD header keep their exact positions, so the
existing enclosure, Link cables and programming fixtures still fit.

### 1. Link-interface power (main functional change)

In v1.0 the console-side pull-up rail of the four BSS138 level shifters was
`/GBVCC`, which could only be supplied by the connected console through Link
connector pin 1. GPIO11 and GPIO12 were reserved as `/VSWITCH_3V3_PIN` and
`/VSWITCH_5V_PIN`, but no switching hardware was fitted, so the interface
level could not actually be selected.

In v1.4 that rail is generated on-board as `/LINK_VREF` by a TPS2116 power
mux, and firmware can select the interface level:

| Item | v1.0 | v1.4 |
| --- | --- | --- |
| Console-side pull-up rail | `/GBVCC`, taken from Link connector pin 1 | `/LINK_VREF`, generated by U4 |
| Voltage source | fixed, supplied by the console | U4 **TPS2116DRL** power mux: `VIN1` = `+3V3`, `VIN2` = `VIN` (VBUS after D1), `VOUT` = `/LINK_VREF` |
| Level control | none (reserved nets only) | GPIO11 → U2 **74LVC1G04GV** inverter → U4 `MODE`; GPIO12 → U4 `PR1` |
| Power-up state | — | R11 and R14 (10 kΩ) pull GPIO11 and GPIO12 high, so `MODE` idles low |
| `/LINK_VREF` load | — | R20, 4.7 kΩ to GND |
| Decoupling | — | C18 1 µF on `/LINK_VREF`, C19 100 nF on `+3V3`, C17 1 µF and C20 10 µF on `VIN` |
| U4 `ST` status output | — | not connected |

Practical effect: the Link interface no longer depends on the console or cable
delivering power on pin 1, and the console-facing bus runs at 3.3 V or 5 V
depending on the GPIO11/GPIO12 setting instead of always following the console.
Firmware has to drive GPIO11 and GPIO12 for the selection to take effect.

### 2. Link connector pin 1

| Item | v1.0 | v1.4 |
| --- | --- | --- |
| Net on J1 pin 1 | `/GBVCC` | `/LINK_PIN1` |
| Function | pull-up rail for R2, R4, R8, R10 only, no MCU connection | fifth level-translated Link channel on GPIO4 |
| Level shifter | — | Q5 **BSS138**, gate to `+3V3`, R12 10 kΩ on the 3.3 V side, R13 10 kΩ to `/LINK_VREF` on the Link side |
| Pull-ups R2, R4, R8, R10 | to `/GBVCC` | to `/LINK_VREF` |

GPIO4 was labelled `/SD2` and left unused in v1.0; it is now `/GP4` and
drives the added level-translation channel through Q5.

### 3. Schematic net changes

| Removed in v1.4 | Added in v1.4 |
| --- | --- |
| `/GBVCC`, `/SD2`, `/VSWITCH_3V3_PIN`, `/VSWITCH_5V_PIN` | `/LINK_VREF`, `/LINK_PIN1`, `/GP4`, `/GP11`, `/GP12`, `Net-(U4-MODE)` |

Net count: 62 in v1.0, 66 in v1.4. The schematic drawing sheet changed from A4
to A3 to fit the added circuitry; the design notes and the RP2040 reference
circuits are unchanged.

### 4. Bill of materials

| Reference | Change |
| --- | --- |
| U4 | added — TPS2116DRL power mux, SOT-583-8 |
| U2 | added — 74LVC1G04GV inverter, SOT-23-5 |
| Q5 | added — BSS138 level shifter, SOT-23-3 |
| R11, R14 | added — 10 kΩ pull-ups on the GPIO11/GPIO12 control lines |
| R12, R13 | added — 10 kΩ level-shifter pull-ups for the pin 1 line |
| R20 | added — 4.7 kΩ load on `/LINK_VREF` |
| C17, C18 | added — 1 µF, 0402 |
| C19 | added — 100 nF, 0402 |
| C20 | added — 10 µF, 0603 |
| C1, C2, C7, C8 | 1 µF parts moved from 0603 to 0402 |
| C3–C6, C9–C15 | footprint reference changed from the local `Library:` library to `Capacitor_SMD:`, pad geometry unchanged |
| OSHW logo footprint | enlarged from 5.7 mm × 6 mm to 11.4 mm × 12 mm |

No component was removed and no value changed other than the C1, C2, C7 and C8
package sizes. The BOM grew from 19 to 23 line items and from 45 to 57
placements.

### 5. PCB and fabrication

| Item | v1.0 | v1.4 |
| --- | --- | --- |
| Copper layers | 2 (`F.Cu`, `B.Cu`) | 4 (`F.Cu`, `In1.Cu`, `In2.Cu`, `B.Cu`) |
| Inner layers | — | `In1.Cu` continuous ground plane; `In2.Cu` power pours for `+3V3` and `+1V1` |
| Ground pours | `F.Cu` and `B.Cu` | `F.Cu`, `B.Cu` and `In1.Cu` |
| Board outline | 23.2 mm × 36.05 mm | unchanged |
| Board thickness | 1.2 mm | 1.2 mm |
| Footprints / pads | 48 / 200 | 60 / 234 |
| Track segments | 289 | 405 |
| Vias | 64 (52 × 0.5/0.3 mm, 12 × 0.4/0.15 mm) | 89 (all 0.5/0.3 mm) |
| Minimum track width | 0.1524 mm | 0.1524 mm, plus a new 0.1778 mm class |
| Copper zones | 13 | 51 |
| Nets | 62 | 66 |
| Bottom-side silkscreen | OSHW logo | OSHW logo, enlarged |
| Board marking | `GB-Link-USB HW Rev 1.0` | `GB-Link HW Rev 1.4` and date `20261007` |

The move to four layers adds a solid ground plane directly under the RP2040 and
the USB differential pair, plus `+3V3` and `+1V1` pours on `In2.Cu`. The
vias also moved from the v1.0 mix of 0.4/0.15 mm and 0.5/0.3 mm to 0.5/0.3 mm
only, which is a less aggressive drill process for low-cost fabrication.

### 6. Unchanged

RP2040 and its QSPI flash, 12 MHz crystal and load network, AP2112K-3.3 LDO,
1N5819WS VBUS diode, USB-C receptacle with its 5.1 kΩ CC resistors and 27 Ω
series resistors, level shifters Q1–Q4 with their 3.3 V-side pull-ups,
WS2812B indicator, BOOTSEL button and pull-up, RUN/reset button, SWD header,
Link port footprint with pins J1.2–J1.5, and the 1.2 mm, 23.2 mm × 36.05 mm
board outline.

## Supported Firmware

The recommended firmware is:

- [GBLink-Firmware](https://github.com/GB-Link/GBLink-Firmware)

The firmware turns the RP2040 into a USB-to-Game Boy Link Cable adapter and
provides the protocol modes used by the GB-Link web clients.

Firmware and browser tools are maintained separately from this hardware
repository. Visit the following pages for current releases and client
instructions:

- [GB-Link Launcher](https://launcher.gblink.io/)
- [GBLink-Firmware releases](https://github.com/GB-Link/GBLink-Firmware/releases)

## Quick Start

1. Manufacture and assemble the PCB using the files in this repository, or use
   an already assembled GB-LINK-USB board.
2. Install the latest GBLink firmware:
   - Use the [GB-Link Launcher](https://launcher.gblink.io/) for web-based
     flashing and updates; or
   - Download `gblink.uf2` from the
     [firmware releases](https://github.com/GB-Link/GBLink-Firmware/releases),
     hold the board's BOOTSEL button while connecting it to the computer, and
     copy the UF2 file to the RP2040 drive.
3. Connect the Game Boy system to the adapter with a compatible Link Cable.
4. Open the GB-Link Launcher, select the required mode, and follow the
   instructions for the game or client you want to use.

## Cable Compatibility

Cable construction differs between Game Boy generations and manufacturers.
According to the GBLink-Firmware documentation:

- A fully wired generic GBC Link Cable can support both GB/GBC and GBA modes.
- An original GBA Link Cable is intended for GBA modes only.
- For GBA operation, the adapter must be connected to the master side of the
  GBA cable.
- Some low-cost third-party GBA cables omit required conductors and may cause
  unreliable communication.

Always confirm the cable selection in the client before starting a link mode.

## Repository Structure

```text
.
|-- hw/
|   |-- v1.0/                    Superseded revision: KiCad schematic, PCB, libraries, 3D models
|   `-- v1.4/                    Current revision: KiCad schematic, PCB, schematic PDF, libraries
|-- gerber/
|   `-- GB-LINK-USB-V1.0-20260827.zip
|                                Manufacturing package for the v1.0 board
|-- extras/
|   |-- gb-link-usb-top.png      Top-side board render
|   |-- gb-link-usb-bot.png      Bottom-side board render
|   |-- gbc.jpg                  Game Boy Printer emulation test
|   |-- gba1.jpg                 GBA multiboot test
|   `-- gba2.jpg                 GBA cartridge-data transfer test
`-- README.md
```

## Manufacturing Notes

- Current hardware revision: **v1.4**
- PCB layer count: **4 layers** (outer signal, inner ground plane and power pours)
- PCB thickness: **1.2 mm**
- Board outline: **23.2 mm × 36.05 mm**, unchanged from v1.0
- KiCad project:
  [`hw/v1.4/GB-LINK-v1.4.kicad_pro`](hw/v1.4/GB-LINK-v1.4.kicad_pro)
- Schematic PDF:
  [`hw/v1.4/GB-LINK-v1.4.pdf`](hw/v1.4/GB-LINK-v1.4.pdf)
- Gerber package in `gerber/` is generated from the **v1.0** design; export
  fresh fabrication output from `hw/v1.4/GB-LINK-v1.4.kicad_pcb` before
  ordering v1.4 boards.

Before ordering assembled boards, review the schematic, PCB, component
availability, connector orientation, and fabrication outputs for the selected
manufacturer. Hardware is provided without warranty; build and use it at your
own risk.

## Acknowledgements

This project exists because many people chose to share their hardware designs,
firmware, documentation, testing, and knowledge with the community.

Special thanks to:

- The [GB-Link-USB-DIY](https://github.com/GB-Link/GB-Link-USB-DIY) project for
  the open hardware reference design and manufacturing documentation.
- [agtbaskara](https://github.com/agtbaskara/game-boy-pico-link-board), creator
  of the original Game Boy Pico link-board design.
- [fairy0715](https://github.com/fairy0715/game-boy-pico-link-board-with-gbaport)
  for the Link Port version on which later designs were based.
- The [GB-Link](https://github.com/GB-Link) maintainers and every contributor to
  [GBLink-Firmware](https://github.com/GB-Link/GBLink-Firmware).
- The [Celio-Link](https://github.com/Celio-Link/Celio-Firmware) project and its
  contributors, whose work forms an important part of the firmware's history.
- Everyone who has contributed code, hardware layouts, footprints, client
  software, documentation, issue reports, testing, and ideas to the wider
  Game Boy homebrew community.

Thank you to all of these open-source contributors. Your work keeps classic
hardware useful, accessible, and fun for a new generation of players and
developers.

## Disclaimer

This is an unofficial, community-developed open hardware project. It is not
affiliated with, endorsed by, or licensed by Nintendo, The Pokémon Company, or
any related game publisher. Game Boy, Game Boy Color, Game Boy Advance, Pokémon,
and other product names are trademarks of their respective owners.
