# Feedback Breakout Rev C

KiCad 9 projects for the 16-channel input feedback conditioning board — the manufactured
replacement for the hand-soldered perfboard between the H11AA1 optoisolator boards and the
MCP23017 expanders.

| Directory | Build | Board | Notes |
|---|---|---|---|
| `feedback-breakout/` | through-hole | 160 × 34 mm | uses the parts already on hand |
| `feedback-breakout-smd/` | 0805 / SOT-23 | 130 × 26 mm | green LEDs only; `bom.csv` + `cpl.csv` for an assembled quote |

Both are built entirely from stock KiCad 9 libraries and carry their own `sym-lib-table` /
`fp-lib-table`, so they open without depending on a global symbol table.

## Where this board sits

```
   12 V field side                              5 V logic side (opto board)
RET+12_SIGn ─[2K7]─[RED]─┐                  ┌──────── +5V
                      H11AA1 ───────────────┤ emitter
GND_SIG ─────────────────┘                  │
                                       DIGn ─┬─ [10K] ── GND
                                             ├─ BC337 ─[680]─[GRN]
                                             │
                                       ~10 m of small-gauge wire
                                             │
                          THIS BOARD ────────┼─ [100k] ─────────── GND   fail-safe level
                                             ├─ [10nF] ─────────── GND   line filter
                                             ├─ [100k] ─ Q base          indicator driver
                                             ├─ test pad
                                             └─ MCP23017 input

                          indicator:  VCC ─[RE]─[LED]─ collector, emitter ─ GND
```

## Why the indicator uses a transistor

The H11AA1 output is an emitter follower — a *current* source limited by LED drive × CTR, not a
stiff voltage source. From the opto board's Eagle design: 3.24 mA through the opto LED, and at the
datasheet minimum CTR of 20 % that leaves **0.66 mA** available at the node.

| Branch | Old perfboard | LED direct | **This board** |
|---|---:|---:|---:|
| Opto board 10k pull-down | 0.470 | 0.470 | 0.470 |
| Opto board BC337 base | 0.014 | 0.014 | 0.014 |
| This board's pull-down | 0.470 (10k) | 0.047 (100k) | 0.047 (100k) |
| This board's LED branch | — | 0.368 | **0.040** |
| MCP23017 input | 0.001 | 0.001 | 0.001 |
| **Total drawn from the node** | **0.955 mA** | **0.900 mA** | **0.572 mA** |
| vs 0.66 mA available | over | over | **fits** |

A common-emitter stage takes its LED current from the rail through the collector and only ~40 µA
of base current from the node. That is what allows an indicator *and* a lower node load than the
board has today.

The opto board's own BC337 cannot serve this purpose: its emitter goes straight to its own LED and
never reaches the connector, and it is 10 m away. A follower buffers where it sits.

> **Note.** Today's board is already over budget at 0.955 mA against a worst-case 0.66 mA — a
> plausible cause of the one channel that reads stuck LOW. The 0.66 mA figure uses the datasheet
> minimum CTR at I<sub>F</sub> = 10 mA while these optos run at 3.3 mA, where it is unspecified;
> real parts are usually better, which is why 15 of 16 channels work today. Use the test pads to
> measure the node against the 4.0 V threshold.

## Rail-generic

A common emitter sets the LED current from the rail, not from the node, so the same board works on
either rail with one value change:

| Rail | RB | RE | LED current | Base current |
|---|---|---|---:|---:|
| 5 V | 100k | **1k** | 2.60 mA | 40 µA |
| 3.3 V | 100k | **470R** | 1.91 mA | 23 µA |

An emitter *follower* would not survive this: at 3.3 V its emitter reaches only 2.3 V against a
2.2 V green LED. Note also that **blue LEDs (Vf ≈ 3.1 V) will not light on a 3.3 V rail at all**,
and at 5 V they run dimmer than green for the same RE.

## No ground pour

GND is an ordinary 54-pad net on both boards — **no zones, no stitching vias** — to be routed by
hand. This is deliberate. The two mechanisms a pour would help do not apply at this scale:

| | |
|---|---|
| Filter integrity | The node sees a 9.09 kΩ source (100k ∥ the opto board's 10k) into 10 nF, cutoff 1.75 kHz. A 160 mm return track is ~160 nH → **0.002 Ω at the cutoff**, 0.00002 % of the source impedance. Even at 100 MHz it is 1.1 %. |
| Shared-impedance offset | All 16 LEDs lit is 41.6 mA through ~80 mΩ of 1 mm track → **3.3 mV**, 0.08 % of the 4.0 V threshold. |

The usual reasons for a plane are absent too: no ns-edge digital (state changes at human speed,
100 ms poll), no µV analog, no radiated-emissions requirement, and no surge role — the H11AA1
provides galvanic isolation.

The trade-off, accepted knowingly: removing the pour *adds* routing work rather than removing it.
GND is the largest net on the board at 54 pads.

## Specification

- 2 layer, 1.6 mm, all through-hole (or 0805/SOT-23 for the SMD variant)
- **Mounting** 2 × M3, ⌀3.2 mm non-plated, **68.58 mm apart, centred both axes**
- **Connectors** 4 × 1×10, 2.54 mm — `JINA`/`JINB` on the top edge, `JOUTA`/`JOUTB` on the bottom,
  banks side by side so each channel's IN and OUT pins are vertically aligned
- **Per channel** 100k pull-down, 10 nF filter, 100k base resistor, transistor, RE, LED, test pad
- **Test points** 16 channel pads plus VCC and GND pads at each end
- **120 footprints**, 66 nets

### Header pinout — all four identical

| Pin | Net |
|---:|---|
| 1 | VCC |
| 2 | GND |
| 3–10 | CH0–CH7 of that bank |

`J*A` → MCP23017 GPA0–7, `J*B` → GPB0–7.

### Parts

| Ref | Value | THT footprint | SMD footprint |
|---|---|---|---|
| R, RB | 100k | `R_Axial_DIN0204_L3.6mm_D1.6mm_P2.54mm_Vertical` | `R_0805_2012Metric` |
| RE | 1k (5 V) / 470R (3V3) | same | same |
| C | 10 nF | `C_Disc_D3.0mm_W1.6mm_P2.50mm` | `C_0805_2012Metric` |
| Q | BC337 (C-B-E) / **BC817-40 (B-E-C)** | `TO-92_Inline` | `SOT-23` |
| D | green LED | `LED_D1.8mm_W3.3mm_H2.4mm` | `LED_0805_2012Metric` |
| TP | — | `TestPoint_THTPad_D1.5mm_Drill0.7mm` | same |
| J | 1×10 | `PinHeader_1x10_P2.54mm_Vertical` | same |

**The transistor pinout differs between the two builds** — BC337 in TO-92 is C-B-E, BC817 in
SOT-23 is B-E-C. The boards wire them accordingly; do not cross-substitute.

The THT resistor footprint suits both a generic 1/6 W (3.2 × 1.8 mm) and the TE CFR16 series
(3.5 × 1.85 mm, 0.25 W) — interchangeable here, both mounted **vertically**. A 6.3 mm DIN0207 body
will *not* fit. Dissipation is under 1.2 mW worst case, so 1/6 W has ~145× margin.

Reference designators are on F.Fab, not silkscreen — no room at this density. The rail-dependent
value is called out on both silkscreen layers.

## BOM for 2 boards

| Part | Qty | |
|---|---:|---|
| 100k (R + RB) | 64 | |
| **1k (RE)** | **32** | needed for the 5 V build |
| 10 nF | 32 | |
| BC337 / BC817-40 | 32 | |
| green LED | 32 | |
| 1×10 header | 8 | |

For the SMD variant, `bom.csv` and `cpl.csv` are in the format an assembly house expects:
**96 SMD placements per board across 5 unique parts**, with the 4 headers and 20 test pads left
as hand-soldered through-hole.

## Status

| Check | THT | SMD |
|---|---|---|
| `kicad-cli sch erc` | **0 violations** | **0 violations** |
| Schematic ↔ PCB netlist | **identical**, 66 nets | **identical**, 66 nets |
| Every `CH_*` net | 6 nodes (JIN, JOUT, R, C, RB, TP) | same |
| `kicad-cli pcb drc` | 15 violations | 21 violations |
| Unconnected | 202 | 202 |

**Open items, both deliberate:**

1. **The M3 holes overlap four channels** — 3, 4, 11, 12 on the THT board and 2, 3, 11, 12 on the
   SMD one. At 68.58 mm spacing they land inside the component band; the spacing was kept to match
   the existing standoffs, and the placement around them is to be adjusted by hand. Every DRC
   violation names `H1`, `H2`, or one of the parts sitting on them.
2. **Not routed.** The 202 unconnected items are the ratsnest, including the 53 GND connections.

## Connector plating

At ~0.57 mA these are dry-circuit contacts, far below the ~10 mA that reliably breaks through oxide
on tin plating — the regime where fretting corrosion appears as intermittent, channel-specific
faults, which is what the seven stuck-high channels looked like. Specify **gold-plated** contacts on
both the header and the mating connector.

## Source hardware

Values above are read from the Eagle project for the optoisolator board (`LightRetaggioSchema`,
2020-04-23): 8 channels, H11AA1 AC-input optocouplers, 2K7 field resistors, 10K pull-downs,
BC337 + 680R LED drivers, manual test buttons SW1–SW8, and no capacitors anywhere on the board.
