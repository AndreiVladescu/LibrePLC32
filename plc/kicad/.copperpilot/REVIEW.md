# LibrePLC32 Design Review

Generated via `kicad-cli` ERC/DRC (KiCad 10.0.5) against `LibrePLC32.kicad_sch` (7 hierarchical sheets) and `LibrePLC32.kicad_pcb`.

- ERC: 1919 violations total (2 errors-class types + warnings, see breakdown)
- DRC: 260 violations total, 0 unconnected items, 0 footprint errors

## What is done well

- Full hierarchical sheet structure (Power, MCU, CAN, RS-485, Ethernet, Terminal Blocks, Ext Interface) is clean and modular.
- PCB has 0 unconnected ratsnest items and 0 footprint errors — routing is topologically complete.
- GND pour zone present and mostly well-connected (only 10 starved thermal spokes out of presumably hundreds of pads).
- Local DRC overrides already in use for known-acceptable padstack/via cases (mechanical/mounting pads), showing prior review passes.
- Custom footprint library (`LibrePLC32.pretty`) is comprehensive and appropriately named per package.

## Prioritized fix list

| Severity | Issue | Location | Recommendation |
|---|---|---|---|
| Critical | ESP32-S3-MINI **EN (pin 45)** is completely unconnected (`pin_not_connected` + `pin_not_driven`) | U10, MCU sheet | EN floating risks erratic boot/brown-out resets. Add 10k pull-up to 3V3 plus 1uF cap to GND (standard Espressif reference) at minimum a pull-up resistor. |
| High | `bus_to_net_conflict`: `MCU_DI` hierarchical label mixed with `MCU_DI[1..4]` bus label on the same node (2 instances) | Root sheet, ~(158–172mm, 72mm) and ~(238–241mm, 25–37mm) | Bus and scalar net with the same base name cannot coexist; rename the scalar net (e.g. `MCU_DI_0`) or fully convert to bus members `MCU_DI0..MCU_DI3`. |
| High | `power_pin_not_driven` (15 instances) across Terminal Blocks, MCU, Power, CAN, RS-485, Ethernet sheets — rails such as `V_FIELD`, `PWR_CAN`, and local 3V3/5V power-input symbols have no visible Output-type power pin driving them | Multiple sheets | Verify each flagged rail: either the source regulator pin's electrical type isn't marked `power_out`, or the sheet lacks a `PWR_FLAG` where the source is off-sheet/external (e.g. field power in from terminal blocks). Add PWR_FLAG symbols on true external-supply nets. |
| Medium | `multiple_net_names`: aliasing collisions — `GPIO_MUX_INT`/`MUX_INT`, `GND`/`~{PPIC_FLT}`, `V_FIELD`/`PPIC_OUT`, `PWR_CAN`/`V_IN_UNPROTECTED` (6 instances) | Various | Same physical net carries two different label names; KiCad silently picks one for the netlist. Rename to a single consistent label per net to avoid confusing the BOM/netlist and future maintainers. |
| Medium | PCB: `via_dangling` (8) / `track_dangling` (2) on nets `V_IN_MAIN`, `V_FIELD` (x5), `Net-(R94-Pad1)`, `SD_CS`, `T1_FLT1` | PCB layout | Stub vias/tracks with only one connected end — remove dead copper or confirm intended stitching-via use (if intentional GND/plane stitching, reassign net or suppress warning explicitly). |
| Medium | `starved_thermal` (10): several GND-zone pad connections only get 1 spoke vs. min 2 | PCB, various C/U/J pads on GND pour | Increase zone thermal spoke count or pad-to-zone connection width for listed pads (C36, C23, C22, C20, U1, J2, X1) for reliable soldering/thermal relief. |
| Low | `silk_overlap` (199) concentrated at CN1/CN6/CN7 (terminal block cluster) and TP9/USB1/U11 | PCB silkscreen | Cosmetic only but very dense in this cluster — reduce/rotate reference/ terminal silkscreen text at CN1/CN6/CN7 for legibility during assembly. |
| Low | `padstack`/`annular_width` (~26): zero/near-zero annular ring on unnetted mechanical PTH pads (RJ1 shield tabs, USB1/USB2 shells, Card1 mounting, SW4/SW5 mounting legs) | PCB | Expected for mechanical-only pads; confirm local overrides are intentional (already flagged "Local override" — likely already accepted, just verify drill/pad sizing matches connector datasheets). |
| Low | `endpoint_off_grid` (1505 warnings) | Whole schematic | High volume suggests schematic wasn't drawn on a consistent grid (many wires/pins at non-50mil coordinates). Not electrically wrong but hampers future edits/tool automation. Consider a grid-snap cleanup pass. |
| Info | `pin_to_pin` (389 warnings): mostly Unspecified-type pins on U1 (LTV-814S opto) and similar connected to Passive/Power pins | Ext Interface / Terminal Blocks | Cosmetic — driven by generic "Unspecified" pin electrical types in the opto/relay symbols rather than a real conflict. Optionally tighten pin types in the custom symbols for cleaner ERC signal-to-noise. |

## Follow-up datasheet cross-check (completed via netlist trace)

| Block | Finding | Verdict |
|---|---|---|
| CAN (ISO1042BDWVR) | `CAN_H`/`CAN_L` correctly wired to U14 pins 7/6; R90 (120R) placed across H/L with SW4 as a selectable split-termination switch — standard practice. TXD/RXD (R91/R92, 0R) are stuffing options, not signal-degrading in default population. | Correct |
| RS-485 (ISO1500DBQR, confirmed integrated isolated transceiver, not a bare isolator) | `EXT_RS485_A/B` wired to U18 A/B pins 12/13 through R95/R96; R94 (120R) + SW5 gives selectable termination, mirroring the CAN block. DIR/RXD/TXD (RS485_DIR, RS485_RXD, RS485_TXD) correctly tied to U10 GPIOs. | **Gap found**: R95/R96 are in-line series resistors only — there are no fail-safe bias resistors (pull-up on A to VCC, pull-down on B to GND) on the isolated side. Without a driver actively on the bus, receiver output can be indeterminate/chattering when idle. Recommend adding a bias resistor pair (e.g. 560R–1k) at one node on the isolated A/B lines. |
| ESP32-S3-MINI-1U (U10) | Confirmed via netlist: **EN (pin 45) is a true dangling net** (`unconnected-(U10-EN-Pad45)`), not just an ERC/ratsnest artifact — matches the Critical finding above. `~{MCU_BOOT}` (GPIO0/pin4) is correctly strapped with a button (SW2) + auto-reset transistor (Q4) + RC (C14/R48), consistent with a standard USB-UART auto-program circuit. | EN fix confirmed mandatory; boot-strap circuit looks correct. |
| Ethernet (W5500, U20) | SPI (MISO/MOSI/SCLK), `~{ETH_CS}`, `~{ETH_INT}`, `~{ETH_RST}` all correctly netted to U10 GPIOs with series/pull resistors (R101–R103) present. | Correct at netlist level; magnetics/RJ45 bias network not independently re-derived from W5500 datasheet in this pass — low risk given HR913129A is a standard integrated-magnetics RJ45. |
| Terminal Blocks / Ext Interface (LTV-814S opto, U1) | Opto pins 1/2 (LED side) wired through R23 to `~{GPIO_MUX_INT}`/MUX path; pins 3/4 (transistor side) on 3V3/GND-referenced logic. Matches typical optoisolated digital input pattern. | Correct at netlist level; LED forward-current resistor sizing not independently recomputed against LTV-814S CTR curve in this pass. |

## Still open (not yet independently re-derived from datasheet numeric specs)
- W5500 magnetics/RJ45 bias resistor value cross-check against HR913129A datasheet.
- LTV-814S current-limiting resistor value vs. supply voltage/CTR curve.
- TPS54560/TPS7A5201/TPS26630 (buck/LDO/eFuse) feedback-divider and compensation component values on the Power sheet.
