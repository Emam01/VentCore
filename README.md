# Chamber Exhaust Fan Control — Snapmaker U1

Adds one custom PWM fan to a 4-hotend Snapmaker U1 running Klipper:

* **Chamber exhaust (PA8)** — watches all 4 hotends, auto-picks a speed by material, Auto/Manual mode
* **PE15 power rail** — mirrors the fan's own commanded speed: on above 0%, off at exactly 0%

PA9 (previously an auxiliary cooling fan) is no longer used by this config — that fan was removed by request.

## Files in this package

|File|What it is|
|-|-|
|`chamber\_exhaust\_fan.cfg`|New file — all fan hardware and logic lives here (renamed from `chamber\_and\_cooling\_fans.cfg` now that it's chamber-exhaust-only)|
|`Klipper\_code.txt`|Your `printer.cfg`, with the stock `\[purifier]` block disabled and one new `\[include]` line added|
|`start\_code.txt`|OrcaSlicer **Machine start g-code**, with 1 line added after `PRINT\_START`|
|`end\_code.txt`|OrcaSlicer **Machine end g-code** — unchanged from what you originally uploaded|

## Installation

1. Copy `ventcore.cfg` into your Klipper config directory, alongside `printer.cfg`.

   * **If you'd already installed `ventcore.cfg` from an earlier version, delete it** — it's superseded by this file and referencing both would double up the chamber exhaust logic.
2. Update `printer.cfg`:

   * Apply the edits shown in step 4 by hand (see below).
3. In OrcaSlicer → Printer Settings → Custom G-code, replace **Machine start g-code** with the contents of `start\_code.txt`. `end\_code.txt` is not affected.
4. Confirm `chamber\_exhaust` shows up as a slider in Fluidd's fan panel.

### Manual edits (if not replacing printer.cfg outright)

* Comment out the entire `\[purifier]` section.
* Add this near your other `\[include]` lines, close to the top of the file:

```
  \[include ventcode.cfg]
  ```

## Why `\[purifier]` was commented out

The U1's stock config already used PA8/PA9 for Snapmaker's official air-purifier accessory (`exhaust\_pin: !PA8`, `inner\_pin: !PA9` inside a `\[purifier]` block). Since your fan is wired to the PA8 header, that block had to be disabled — Klipper won't let two config sections claim the same pin. It's commented out, not deleted, so it's easy to restore if you ever install the genuine accessory instead. PA9 is now simply unclaimed/free.

That block also gated the header's 24V rail through `power\_enable\_pin: PE15`. That's now handled separately by `\[output\_pin fan\_header\_power\_enable]` in the new file — see **24V Rail Power (PE15)** below for how it's switched.

## Chamber Exhaust Fan (PA8)

Every 2 seconds, a background loop checks the temperature of all four hotends (`extruder` .. `extruder3`). If any is ≥ 50°C, the fan runs at a speed chosen by the currently active material; once all four drop below 50°C, it turns off. It only sends a new speed command when the target actually changes.

|Material|Speed|Material|Speed|
|-|-|-|-|
|PLA|10%|PET-CF|40%|
|PETG|25%|ABS-CF|90%|
|TPU|0%|ASA-CF|90%|
|ABS|80%|PC-CF|100%|
|ASA|80%|Nylon / PA|100%|
|PC|100%|*unrecognised*|50% (fallback)|

**Modes:**

* **Auto** (default) — speed follows the table above automatically. The slicer sets the material for you at the start of every print via `SET\_CHAMBER\_FAN\_MATERIAL`, so normally you don't need to touch this.
* **Manual** — the background loop stops adjusting the fan, and the slider in Fluidd drives it directly. Stays in Manual until you switch back.

## 24V Rail Power (PE15)

A 0% PWM command on PA8 wasn't fully stopping the fan on its own — many 4-wire PWM fans have a minimum-speed floor built into their own controller and keep the motor turning at low RPM any time 24V is present, no matter what the PWM signal says. Cutting the 24V rail itself is the only way to guarantee the fan is genuinely at 0 RPM.

PE15 now directly mirrors `chamber\_exhaust`'s own commanded speed, checked on the same 2-second loop as the exhaust logic itself:

* **On** whenever speed is above 0%.
* **Off** whenever speed is exactly 0%.
* This applies equally in Auto mode, after a manual slider change, and to a hotend heated outside a formal print (e.g. preheating for maintenance) — there's no separate "is a print running" concept involved, so it can't get out of sync with what the fan is actually doing.
* Off by default at Klipper startup.

There's no manual PE15-only override button. Manual control already exists one level up: switch to Manual mode and use the Fluidd slider, and the rail will automatically follow whatever speed you set within the same 2-second loop. A separate "force the rail on regardless of speed" button would risk recreating the exact residual-spin issue this change fixes, so it's been left out — say the word if you'd like a diagnostic-only version added back (e.g. for confirming the wiring is live at all).

## Command reference

All of these work typed into the Fluidd console, and the non-underscore ones also appear as clickable buttons in Fluidd's **Macros** panel.

|Command|Effect|
|-|-|
|`CHAMBER\_FAN\_AUTO`|Chamber exhaust → Auto mode|
|`CHAMBER\_FAN\_MANUAL`|Chamber exhaust → Manual mode (use the slider)|
|`SET\_CHAMBER\_FAN\_MATERIAL MATERIAL=ABS`|Set the material used for Auto speed selection|
|`CHAMBER\_FAN\_STATUS`|Reports mode/material/speed and PE15 power state|

## Customising

**Material speed table** — edit the `speeds` dictionary inside the `\_CHAMBER\_FAN\_UPDATE` macro in `ventcode.cfg`. Values are 0.0–1.0 (fraction of full speed), keyed by the uppercased material name.

**50°C on/off threshold** — also inside `\_CHAMBER\_FAN\_UPDATE`, the two `>= 50` comparisons.

## Additional Info

* **PA8 polarity** — originally inherited as inverted (`!PA8`) from the OEM `\[purifier]` block, but confirmed backwards for this fan (it ran even at a 0% commanded speed) and has been corrected to `pin: PA8`. If the fan now stays off at every speed including 100%, put the `!` back.
* **PE15 polarity** — assumed active-high (no `!` in the original config). Not confirmed against a schematic.
* **"Nylon" → `PA`** — OrcaSlicer's internal `filament\_type` string for nylon filaments is very likely `PA`, not `Nylon`; both are mapped to 100% to be safe. Confirm with `CHAMBER\_FAN\_STATUS` after slicing a nylon print.
* Materials outside the table above (PP, HIPS, PVA, etc.) fall back to 50%, not 0%.
* PLA Fan speed has been set to 10% so as to keep the cooling fan on. This is a workaround. Whenever filter fan status is zero, PE15 pin is off which cuts power to the whole module.

## Troubleshooting

|Symptom|Likely cause|
|-|-|
|Fan runs even when commanded to 0% (in either mode)|PWM polarity backwards — already fixed by removing `!` from `pin: PA8`; if it's still happening, something other than pin inversion is at play|
|Fan stays off at every speed, including 100%|Pin inversion was needed after all — add `!` back: `pin: !PA8`|
|Fan still creeps/spins slightly even with SPEED=0|Check `CHAMBER\_FAN\_STATUS` — it should report `Fan power (PE15): OFF` any time speed is 0%; if it's still spinning with power confirmed OFF, the leakage path isn't PE15/PA8 and is worth investigating separately (e.g. a wiring or connector issue)|
|Fan doesn't spin at all, ever|PE15 polarity — try `pin: !PE15` on `\[output\_pin fan\_header\_power\_enable]`|
|Exhaust never turns on while hot during a print|Check mode with `CHAMBER\_FAN\_STATUS` — if it says `manual`, run `CHAMBER\_FAN\_AUTO`; also confirm it reports `Fan power (PE15): ON`|
|Exhaust speed doesn't match expected material|Run `CHAMBER\_FAN\_STATUS` right after slicing starts to see what material string actually arrived|



