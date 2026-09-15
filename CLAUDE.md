# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file [ESPHome](https://esphome.io) YAML configuration (`light-controller.yaml`) for an ESP32-PoE-ISO
board that controls the impulse relays ("relè passo-passo") for the lights across two floors of a house,
while reading back physical relay state through separate feedback contacts. There is no application code
beyond this YAML plus its embedded C++ lambdas — ESPHome generates the firmware project into `.esphome/`
(gitignored build output, do not hand-edit).

## Commands

Requires the `esphome` CLI (invoked here via `uv`/pip in prior sessions — use whatever is available,
e.g. `pipx run esphome ...` or a project venv).

- Validate config: `esphome config light-controller.yaml`
- Compile only: `esphome compile light-controller.yaml`
- Compile + flash (USB or OTA) + attach logs: `esphome run light-controller.yaml`
- Tail logs only (device must already be running): `esphome logs light-controller.yaml`
- Clean build artifacts: `esphome clean light-controller.yaml`

There is no test suite. "Correctness" is checked via `esphome config` (schema/lambda compile check) and,
when hardware is available, `esphome run` + watching the UART/API logs. There are no lint commands beyond
ESPHome's own config validation.

`secrets.yaml` (gitignored) must exist alongside the YAML; copy `secrets-sample.yaml` to create it. Note
that Wi-Fi is currently disabled in favor of Ethernet, so the `!secret wifi_*` entries aren't presently
referenced anywhere active in `light-controller.yaml` — the `wifi:` block is commented out.

## Architecture

### Hardware topology

- ESP32-PoE-ISO with Ethernet (LAN8720 PHY), no Wi-Fi in use.
- Two I2C-connected MCP23017 16-bit GPIO expander pairs, one pair per floor, sharing bus `bus_i2c`:
  - **P0 (ground floor, 12 circuits)**: `mcp_p0_out` @ `0x20` (relay outputs), `mcp_p0_in` @ `0x21` (feedback inputs)
  - **P1 (first floor, 15 circuits)**: `mcp_p1_out` @ `0x22` (relay outputs), `mcp_p1_in` @ `0x23` (feedback inputs)
- The two input expanders' interrupt lines are mirrored (`IOCON` MIRROR bit) and wired-OR into a single
  ESP32 GPIO (`GPIO5`, active-low), configured on `on_boot` by writing `GPINTENA/B` and `IOCON` directly via
  `write_byte` — MCP23017 interrupts aren't natively supported by ESPHome's component, so this is a manual
  register-level workaround.

### ID naming convention

Every circuit `NN` (zero-padded, per floor `0`/`1`) has up to four related entities — keep this pattern when
adding/editing circuits:
- `p{floor}_state_{NN}` — `binary_sensor` reading the physical feedback contact (ground truth)
- `p{floor}_relay_{NN}` — raw GPIO `output` on the *_out expander (physical relay coil, `inverted: true`)
- `p{floor}_logic_{NN}` — template `output` implementing the pulse-fire logic (see below)
- `light_p{floor}_{NN}` — the HA-facing `light` entity (`platform: binary`) bound to `p{floor}_logic_{NN}`

### Pulse (impulse-relay) logic, not a level output

The physical relays are bistable/impulse ("passo-passo") relays: pulsing the coil toggles the circuit, it
isn't driven continuously. Each `p{floor}_logic_{NN}` template output only fires a pulse on the matching
`p{floor}_relay_{NN}` when the requested light state actually differs from the live feedback
(`p{floor}_state_{NN}`) — this prevents redundant pulses (which would flip the relay to the wrong state)
when HA/ESPHome and the physical world are already in sync. Pulse width is the `PULSE_DELAY` substitution
(500ms), the single place to retune it.

### State sync: physical feedback is authoritative

Because the physical relay can be toggled by a normal wall switch outside of ESPHome's control, the
feedback `binary_sensor`s are the source of truth and always win over whatever state HA/ESPHome thinks the
light is in. This is implemented twice, deliberately:
1. Each `p{floor}_state_{NN}` binary_sensor has its own `on_state` automation that turns the paired light
   on/off to match.
2. The `read_all_inputs` script (mode `queued`) bulk-polls every input pin on both expanders and force-syncs
   any light whose current state disagrees with the I2C feedback, logging when it has to correct one. This
   runs on a 100ms `interval` *and* is also triggered immediately by the `GPIO5` interrupt line, so real
   button presses get near-instant sync while the interval is the polling fallback/safety net.

When touching this sync logic, preserve the invariant that i2c/feedback state always overrides in-memory
light state, never the other way around.

### Diagnostics

- `dump_mcp23017_registers` script (parametrized by I2C address) full-dumps every MCP23017 register
  (IODIR/IPOL/GPINTEN/DEFVAL/INTCON/IOCON/GPPU/INTF/INTCAP/GPIO/OLAT) plus a per-pin decode to the log —
  wired to four diagnostic buttons ("Dump MCP 0x20".."0x23").
- A 20s `interval` pings all four I2C addresses (`0x20`-`0x23`) and logs presence, for spotting a dropped
  expander.
- A commented-out `on_boot` block (priority `-100`) shows the intended pattern for a hard `esp_restart()` if
  any MCP fails to initialize after boot — currently disabled, not wired up.

### Safety/OTA

`safe_mode` will auto-rollback a bad OTA after `num_attempts` failed boots. The `ESPHome Restart` button
explicitly calls `esp_ota_mark_app_valid_cancel_rollback()` before rebooting so that a manual restart isn't
mistaken for a boot failure. `Clear Stored States` wipes NVM preferences and reboots via the restart button
(reuses its logic rather than duplicating it).

## Conventions to preserve

- Room/circuit names (`name:` on `light` entities) are in Italian and correspond to real physical rooms —
  keep new entries in the same language and naming style.
- Comments are English throughout (translated from a former Italian/English mix) — write any new comment in
  English, per the user's standing preference across all projects.
- `.gitignore` was copied from an unrelated KiCad hardware project ("KinderLight RGB") and still contains
  KiCad/gerber-specific rules that don't apply here — harmless, but don't take it as a sign of another
  project living in this repo.
