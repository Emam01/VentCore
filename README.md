# VentCore
VentCore Config File
# Chamber Exhaust Fan Control — Snapmaker U1

- **Chamber exhaust (PA8)** — watches all 4 hotends, auto-picks a speed by material, Auto/Manual mode

## Installation

1. Copy `ventcore.cfg` into your Klipper config directory, alongside `printer.cfg`.
2. In printer.cfg, add "[include ventcore.cfg]" after "[include fluidd.cfg]"
3. Restart Klipper (Fluidd → Settings → **Restart Klipper**, or `FIRMWARE_RESTART`).
4. In OrcaSlicer → Printer Settings → Custom G-code, add the following after "PRINT START"
';===== chamber exhaust fan setup (added) ==========
SET_CHAMBER_FAN_MATERIAL MATERIAL={filament_type[initial_extruder]}
;===================================================='

5. Confirm `chamber_exhaust` shows up as a slider in Fluidd's fan panel.

## Why `[purifier]` was disabled

The U1's stock config already used PA8/PA9 for Snapmaker's official air-purifier accessory (`exhaust_pin: !PA8`, `inner_pin: !PA9` inside a `[purifier]` block). Since your own fan is wired to the PA8 header, that block had to be disabled — Klipper won't let two config sections claim the same pin. It's commented out, not deleted, so it's easy to restore if you ever install the genuine accessory instead. PA9 is now simply unclaimed/free.

That block also gated the header's 24V rail through `power_enable_pin: PE15`. That's now handled separately by `[output_pin fan_header_power_enable]` in the new file, driven high unconditionally — kept on at all times (not scoped to only while printing), since the exhaust fan can also be needed for hotends heated manually outside of a formal print. An always-on rail trivially guarantees it's powered during printing as well.

## Chamber Exhaust Fan (PA8)

Every 2 seconds, a background loop checks the temperature of all four hotends (`extruder` .. `extruder3`). If any is ≥ 50°C, the fan runs at a speed chosen by the currently active material; once all four drop below 50°C, it turns off. It only sends a new speed command when the target actually changes.

| Material | Speed | Material | Speed |
|---|---|---|---|
| PLA | 0% | PET-CF | 40% |
| PETG | 25% | ABS-CF | 90% |
| TPU | 0% | ASA-CF | 90% |
| ABS | 80% | PC-CF | 100% |
| ASA | 80% | Nylon / PA | 100% |
| PC | 100% | *unrecognised* | 50% (fallback) |

**Modes:**
- **Auto** (default) — speed follows the table above automatically. The slicer sets the material for you at the start of every print via `SET_CHAMBER_FAN_MATERIAL`, so normally you don't need to touch this.
- **Manual** — the background loop stops adjusting the fan, and the slider in Fluidd drives it directly. Stays in Manual until you switch back.

## Command reference

All of these work typed into the Fluidd console, and the non-underscore ones also appear as clickable buttons in Fluidd's **Macros** panel.

| Command | Effect |
|---|---|
| `CHAMBER_FAN_AUTO` | Chamber exhaust → Auto mode |
| `CHAMBER_FAN_MANUAL` | Chamber exhaust → Manual mode (use the slider) |
| `SET_CHAMBER_FAN_MATERIAL MATERIAL=ABS` | Set the material used for Auto speed selection |
| `CHAMBER_FAN_STATUS` | Reports mode/material/current speed |

## Customizing

**Material speed table** — edit the `speeds` dictionary inside the `_CHAMBER_FAN_UPDATE` macro in `ventcore.cfg`. Values are 0.0–1.0 (fraction of full speed), keyed by the uppercased material name.

**50°C on/off threshold** — also inside `_CHAMBER_FAN_UPDATE`, the two `>= 50` comparisons.

## Known assumptions worth verifying

- **PE15 polarity** — assumed active-high (no `!` in the original config). Not confirmed against a schematic.
- **"Nylon" → `PA`** — OrcaSlicer's internal `filament_type` string for nylon filaments is very likely `PA`, not `Nylon`; both are mapped to 100% to be safe. Confirm with `CHAMBER_FAN_STATUS` after slicing a nylon print.
- Materials outside the table above (PP, HIPS, PVA, etc.) fall back to 50%, not 0%.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| Fan doesn't spin at all | PE15 polarity — try `pin: !PE15` on `[output_pin fan_header_power_enable]` |
| Fan runs full speed regardless of commanded % | Inverted pin logic — try removing/adding `!` on `pin: !PA8` |
| Exhaust never turns on while hot | Check mode with `CHAMBER_FAN_STATUS` — if it says `manual`, run `CHAMBER_FAN_AUTO` |
| Exhaust speed doesn't match expected material | Run `CHAMBER_FAN_STATUS` right after slicing starts to see what material string actually arrived |
