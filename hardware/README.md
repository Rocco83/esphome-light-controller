# Feedback Breakout Rev B

KiCad 9 projects for the 16-channel input feedback conditioning board — the manufactured
replacement for the hand-soldered perfboard that sits between the H11AA1 optoisolator boards
and the MCP23017 expanders.

| Directory | Rail | Pull-down | LED series |
|---|---|---|---|
| `feedback-breakout-5v/` | 5 V | 100k | 10k |

The 5 V rail is set by the opto boards' `+5VARD` output, so that is the only build that applies to
this installation. For a system without those boards in front, the same artwork works at 3.3 V by
fitting a 3k9 LED resistor instead of 10k — the pull-down and filter are unchanged.

Built entirely from stock KiCad 9 libraries. The project carries its own `sym-lib-table` /
`fp-lib-table`.

## Where this board sits

```
   12 V field side                                    5 V logic side
                                               ┌──────────────── +5V
RET+12_SIGn ─[2K7]─[RED LED]─┐                 │
                          H11AA1 ──────────────┤ emitter
GND_SIG ─────────────────────┘                 │
                                          DIGn ─┬─ [10K] ── GND     (on the opto board)
                                                ├─ BC337 ─[680]─[GRN LED]
                                                │
                                          ~10 m of small-gauge wire
                                                │
                                THIS BOARD ─────┼─ [100k] ── GND    fail-safe level
                                                ├─ [10nF] ── GND    line filter
                                                ├─ [10k] ─ [LED] ── GND   indicator
                                                ├─ test pad
                                                └─ MCP23017 input
```

The opto board already provides a 10k pull-down and two indicator LEDs per channel (red on the
field side, green buffered by a BC337). This board adds the line filtering, a fail-safe level,
a probe point, and a third indicator at the expander end.

## Why the pull-down is 100k, not 10k

The H11AA1 output is an emitter follower — a *current* source limited by LED drive × CTR, not a
stiff voltage source. Measured from the Eagle design:

| | |
|---|---:|
| Opto LED current: (12 − 1.9 − 1.35) / 2K7 | 3.24 mA |
| H11AA1 CTR, datasheet minimum | 20 % |
| Worst-case collector current available | 0.65 mA |

A 10k pull-down here would sit in **parallel** with the opto board's 10k, halving it to 5k:

| Configuration | Effective pull-down | Node load |
|---|---:|---:|
| Old perfboard (10k, no LED) | 5.00k | 0.97 mA |
| **This board (100k, LED via 10k)** | **9.09k** | **0.83 mA** |
| 100k, LED via 6k8 (brighter) | 9.09k | 0.96 mA |
| 10k + LED via 3k3 | 5.00k | 1.82 mA ✗ |

100k frees more current than the LED consumes, so every channel gets **more** margin than it has
today while still gaining an indicator. Swap the LED resistor to 6k8 for a brighter indication at
the old loading level.

Keeping 100k also preserves a defined level if the 10 m run ever opens — the channel reads OFF
rather than floating high.

### No transistor buffer

The textbook fix for the LED load is a BC337 emitter follower, as the opto board does. Sixteen
TO-92 packages are 26 mm² each and push the board to **97 % courtyard fill** — not routable on
2 layers at 80 × 20 mm. Sizing the LED resistor achieves the same end within the area budget.

## Specification

- **Board** 80.00 × 20.00 mm, 2 layer, 1.6 mm, all through-hole
- **Mounting** 2 × M3, ⌀3.2 mm non-plated, 68.58 mm apart, centred both axes
- **Connectors** J1 (IN, from opto boards) and J2 (OUT, to MCP23017) — 2×10, 2.54 mm, odd/even
- **Per channel** 100k pull-down, 10 nF filter, 1.8 mm LED + series resistor, 1 mm test pad
- **84 footprints** — 64 passives/LEDs, 16 test pads, 2 connectors, 2 mounting holes

### Pinout — J1 and J2 identical

| Position | Row A pin | Net | Row B pin | Net |
|---:|---:|---|---:|---|
| 1 | 1 | VCC | 2 | VCC |
| 2 | 3 | GND | 4 | GND |
| 3–10 | 5,7,…,19 | CH_A0…CH_A7 | 6,8,…,20 | CH_B0…CH_B7 |

`CH_A0..A7` → MCP23017 GPA0–7, `CH_B0..B7` → GPB0–7.

Reference designators are on F.Fab, not silkscreen — at 2.15 mm pitch there is no room for
readable silk text.

## Connector plating matters here

At ~0.83 mA these are **dry-circuit** contacts, far below the ~10 mA that reliably breaks through
oxide films on tin plating. Low-current signal contacts are exactly where fretting corrosion shows
up as intermittent, channel-specific faults — which is what the seven stuck-high channels looked
like. Specify **gold-plated** contacts on both the header and the mating connector, and prefer a
keyed, latching shroud so the housing cannot walk out or seat one row off.

Lowering the load from 0.97 mA to 0.83 mA does not change this — both are well below the wetting
threshold, so plating is the fix, not current.

## The 10 m run

- Wire resistance (~3 Ω round trip on thin wire) is negligible against a 9 kΩ source.
- Line capacitance is roughly 0.5–1 nF over 10 m; the 10 nF swamps it and sets the filter.
- Filter time constant: 9.09k × 10 nF ≈ **91 µs** — far faster than the 100 ms firmware poll, so
  it costs no responsiveness while still rejecting pickup.

## Verification

Checked with KiCad 9.0.2 itself:

| Check | Result |
|---|---|
| `kicad-cli sch erc` | **0 violations** |
| `kicad-cli pcb drc` | **0 violations** |
| Schematic ↔ PCB netlist | **identical** — 34 nets, same membership |
| Every `CH_*` net | exactly 6 nodes (J1, J2, R, C, LED anode, TP) |

DRC also reports 150 unconnected items — that is the ratsnest. **The board is placed and netlisted
but not routed.** Open it in pcbnew and route, or run an autorouter.

## Source hardware

Values above are read from the Eagle project for the optoisolator board
(`LightRetaggioSchema`, 2020-04-23): 8 channels, H11AA1 AC-input optocouplers (drawn with an
IL250 deviceset), 2K7 field resistors, 10K pull-downs, BC337 + 680R LED drivers, manual test
buttons SW1–SW8, and no capacitors anywhere on the board.
