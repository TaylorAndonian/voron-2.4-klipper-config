# Voron 2.4 — BTT Eddy Duo + PZ piezo commissioning runbook

Work top to bottom. Do not skip ahead — several steps exist purely to make the
next one safe.

**Keep a hand near the emergency stop for steps 9, 11 and 12.**

---

## The one thing to understand before you start

Klipper allows exactly one object named `probe`. Both `[probe]` and
`[probe_eddy_current]` call `add_object('probe', self)` internally, so the two
sections **cannot coexist** — Klipper aborts with *"Printer object 'probe'
already created"*.

That is why your piezo has been demoted from `[probe]` to a plain
`[stepper_z] endstop_pin`. It behaves identically at the hardware level (it is
still a digital trigger on `EBBCan:PB9`, still active-low), but it now homes Z
as an endstop rather than probing as a probe. The Eddy takes over the `probe`
role and does QGL and meshing.

This is the only arrangement in mainline Klipper that gives you what you asked
for. It is not a workaround; it is the correct topology.

---

## Step 1 — Back up

```bash
cd ~/printer_data/config
tar czf ~/config-backup-$(date +%F).tar.gz *.cfg *.conf
```

Confirm the tarball exists before continuing. If anything below goes wrong,
extracting this puts you back exactly where you were.

---

## Step 2 — Check your Klipper version

```bash
cd ~/klipper && git log -1 --format='%h %ci'
```

Current mainline uses `descend_z` in `[probe_eddy_current]` and has deprecated
`z_offset` there. `eddy.cfg` is written for the current form. If Klipper
refuses to start with *"Option 'descend_z' is not valid"*, comment out
`descend_z` and uncomment the `z_offset: 2.5` line right below it.

If your Klipper is more than a few months old, update it before starting — Eddy
support has moved quickly.

---

## Step 3 — Flash the Eddy Duo

First check whether it is already flashed:

```bash
ls /dev/serial/by-id/
```

If you see an `rp2040` entry, skip to step 4. Otherwise:

```bash
cd ~/klipper
make menuconfig
```

Set:

- Micro-controller: **Raspberry Pi RP2040**
- Bootloader offset: **No bootloader**
- Flash chip: **W25Q080 with CLKDIV 4** ← mandatory, it will not boot otherwise
- Communication interface: **USB**

Then:

```bash
make clean
make
```

Put the Duo in bootloader mode (hold BOOT while plugging in the USB), then:

```bash
make flash FLASH_DEVICE=2e8a:0003
```

---

## Step 4 — Get the serial ID

```bash
ls /dev/serial/by-id/
```

Copy the `rp2040` path in full into `eddy.cfg` → `[mcu eddy]` → `serial:`.
Replace the whole placeholder line. This is **MEASURE ME 1/3**.

---

## Step 5 — Install the config files

Copy into `~/printer_data/config/`:

- `printer.cfg` (replaces yours)
- `eddy.cfg` (new)
- `macros.cfg` (new)
- `KAMP_Settings.cfg` (replaces yours)

Then **delete these two lines from the autosave block at the bottom of the new
printer.cfg** if they carried over:

```
#*# [stepper_z]
#*# position_endstop = 0.231
```

That value is stale — it was written while the machine was on
`probe:z_virtual_endstop`, so it never controlled anything. Left in place it
silently overrides the `position_endstop: 0.0` in the config body.

Also delete the duplicate `KAMP_Settings (1).cfg` from your config folder if it
exists on the printer. Only one should be present.

Restart Klipper. It should come up clean. If it complains about a missing
`LINE_PURGE` or `SMART_PARK`, your KAMP install lives somewhere other than
`~/printer_data/config/KAMP/` — fix the include paths in `KAMP_Settings.cfg`.

---

## Step 6 — Measure the Eddy X/Y offsets  ✅ DONE

**Result: `x_offset: 0.0`, `y_offset: 22.5`** — written into `eddy.cfg`, along
with recomputed mesh bounds (`20,28` → `325,330`) and `scan_overshoot: 8`,
which is only safe because `x_offset` came out at 0.

All four QGL points verified reachable, but the two front ones now put the
toolhead at **Y=2.5** — 2.5 mm from the front limit, and the tightest clearance
in the whole config. Still legal. If the Eddy moves back any further they will
fail with "Move out of range" (an error, not a crash) and you should raise both
front points from Y=25 to Y=35.

⚠ **Re-check the rear clearance.** At Y home the toolhead goes to 350, which
now puts the coil at **372.5** — 5 mm deeper behind the bed than the previous
measurement. If the mount moved, that clearance moved with it. Push the gantry
to Y max by hand and confirm the Eddy clears the rear extrusion before you send
`G28` at homing speed.

The method is kept below for reference / re-measuring.

### Method

I checked both the Printables page for Drthsideous's Eddy Duo mount and the
parent `Eddie_Module.stl` in the Xol repo. **Neither publishes offsets.** The
remix description only says the openings were widened a few millimetres. So
this has to come off your machine — I am not going to invent it.

1. `G28`
2. Put masking tape near bed centre; mark a fine dot on it.
3. Jog until the **nozzle tip** is over the dot. `GET_POSITION` → `Xn`, `Yn`
4. Jog until the **coil centre** is over the dot. `GET_POSITION` → `Xp`, `Yp`
5. `x_offset = Xn − Xp`   and   `y_offset = Yn − Yp`

**Mind the order — nozzle minus probe, not probe minus nozzle.** Klipper's docs:
*"The x_offset is then the `nozzle_x_position - probe_x_position`"*. Reversing
it flips the sign and silently puts your mesh on the wrong part of the bed.

Sanity check: on the Xol the coil sits behind the nozzle, so `y_offset` must
come out **positive** and `x_offset` should be small. A negative `y_offset`
means you subtracted the wrong way round.

**Send me the two raw coordinate pairs rather than the subtraction** — sign
convention is the single easiest thing to get backwards here, and a sign error
puts your mesh on the wrong part of the bed without any error message.

Once the offsets are real, widen the mesh bounds using the formula in
`eddy.cfg` to recover the bed area the conservative placeholders are giving up.

---

## Step 7 — Verify the mount height

Target: coil PCB underside **2–3 mm above the nozzle tip**, ~2.95 mm is ideal.

The plastic case adds roughly 1.2 mm, so if you measure to the bottom of the
case you want about **1.75 mm**.

Too low and the coil clips bed clips during a scan. Too high and it falls out of
sensing range at `horizontal_move_z: 2` and the scan fails.

Also confirm the coil is roughly parallel to the bed. It does not need to be
perfect — a steady tilt calibrates out — but it must not *change* as the
toolhead moves. The Xol hangs the Eddy off the carriage as a cantilever, so
check the carriage-to-rail screws are properly seated.

---

## Step 8 — Piezo Z homing sanity check

Still no Eddy involvement.

```
G28 X Y
G28 Z
```

Watch it. It should descend at 20 mm/s, touch, retract 3 mm, and touch again at
10 mm/s. Those are your existing known-good values, carried over unchanged.

Then set the real zero:

```
Z_ENDSTOP_CALIBRATE
```

Paper test, `TESTZ Z=-0.05` etc., `ACCEPT`, `SAVE_CONFIG`.

**Do not skip this.** `position_endstop` currently reads `0.0`, which assumes
the piezo triggers at exactly nozzle-touching-bed. It is close, but a positive
error here means the nozzle digs into your bed on the first print.

---

## Step 9 — Eddy drive current  ⚠ hand on the E-stop

```
EDDY_DRIVE_CURRENT
```

Homes, moves to bed centre, lifts to Z=20, runs
`LDC_CALIBRATE_DRIVE_CURRENT CHIP=btt_eddy`. Then `SAVE_CONFIG`.

If it errors about the sensor not reading, the coil is either too far from the
bed or facing the wrong way.

---

## Step 10 — Map frequency to height

```
EDDY_CALIBRATE
```

Because the piezo homes Z independently, you do **not** have the usual
chicken-and-egg problem here — no `force_move`, no `SET_KINEMATIC_POSITION`
fakery. The macro homes for real and calibrates against a genuine zero, which
is both safer and more accurate than BTT's stock `..._AUTO` macro.

When the manual probe prompt appears, the nozzle is already at true zero, so
accept at Z=0. Then `SAVE_CONFIG`.

Check it:

```
TEST_EDDY
```

Standard deviation should be well under 0.01 mm.

---

## Step 11 — First QGL  ⚠ watch this one

`horizontal_move_z` changed from 5 to 2, because at Z=5 the coil sits ~8 mm off
the bed and out of range. That means the toolhead now traverses 3 mm lower than
it used to during levelling.

Run it manually, once, watching:

```
G32
```

If it will not converge within 5 retries, raise `retry_tolerance` in
`[quad_gantry_level]` from `0.0075` to `0.015`.

---

## Step 12 — First mesh  ⚠ watch this one too

```
MESH
```

Full-bed rapid scan, then saves as the `default` profile. Should take well
under a minute.

Sanity check the result in Mainsail's heightmap. A real Voron bed is a gentle
bowl or saddle with maybe 0.15 mm of range. If you see a steep linear ramp
across the whole bed, that is gantry tilt or a mount that flexes — not bed
shape. Come back to me with a screenshot if it looks odd.

---

## Step 13 — Thermal drift calibration

This is the reason the Duo was the right buy over the Coil, and on a 2.4 with a
hot chamber it is not optional.

```
TEMPERATURE_PROBE_CALIBRATE PROBE=btt_eddy TARGET=<your usual chamber max> STEP=2
```

Set TARGET to the hottest chamber you actually print at. It takes a while and
prompts for manual probes along the way. `SAVE_CONFIG` at the end.

---

## Step 14 — Nozzle brush  ← MEASURE ME 3/3

**There is no brush anywhere in your existing config**, so I have assumed you do
not have one fitted. `brush_enable` in `_CFG` is `False`, and `CLEAN_NOZZLE`
currently prints a warning and does nothing.

`PRINT_START` still runs correctly with it disabled — it just skips the wipes.

When you fit one, measure and fill in, in `macros.cfg` → `_CFG`:

| Variable | What to measure |
|---|---|
| `brush_start_x`, `brush_end_x` | X span the nozzle sweeps across the bristles |
| `brush_y` | Y centre line of the bristles |
| `brush_z` | Z with the nozzle *into* the bristles — start 1 mm high, creep down 0.2 at a time |
| `brush_approach_z` | safe travel height above the brush, default 10 |

Then set `brush_enable: True`.

I will not guess these. A wipe macro with invented coordinates is a toolhead
crash, and you asked me not to risk that.

---

## Step 15 — Before the first real print

- `TEST_PIEZO` — homes Z repeatedly and uses the Eddy as a ruler to measure how
  consistently the piezo lands. Spread under 0.010 mm is healthy. Over
  0.025 mm, go hunting for loose heatsets, slack Z belts, or coldside-to-carriage
  interference — the classic PZ-on-a-printed-carriage failure mode.
- `LOAD_FILAMENT` once, watching. If filament reaches the nozzle before the
  45 mm feed finishes, reduce `load_fast`; if it is still short, increase it.
- Print a single-layer 100 mm square and check first-layer squish.
- Your `rotation_distance: 4.69` is already calibrated, so extruder e-steps are
  left alone. Your PID values for both heaters carried over untouched.

---

## What changed, in one page

**Removed**

- `[probe]` — the piezo. Conflicts with the Eddy; now `[stepper_z] endstop_pin`.
- `[bed_mesh]` from printer.cfg — moved to `eddy.cfg` with Eddy-appropriate values.
- Duplicate `[virtual_sdcard]`, `[pause_resume]`, `[display_status]` — these were
  declared in *both* printer.cfg and mainsail.cfg.
- Commented-out `exhaust_fan` and `caselight` blocks — dead weight.
- `TEST_PZ` — it called `PROBE_ACCURACY`, which now belongs to the Eddy, so it
  silently stopped testing what its name said. Replaced by `TEST_PIEZO` and
  `TEST_EDDY`.
- KAMP adaptive meshing — see below.

**Kept**

- QGL. You agreed. A mesh cannot substitute for it on a 2.4: four independent Z
  motors have to be brought parallel to the bed before a mesh means anything.
  It now runs on the Eddy, which is ideal for it — QGL is a purely *relative*
  measurement between four points, so Eddy thermal drift cancels out.
- `max_z_accel: 200` and the reduced Z current during homing. Both still matter
  for the piezo.
- Your calibrated `rotation_distance`, pressure advance, and both PID sets.

**Why adaptive meshing is now off**

KAMP's `Adaptive_Meshing.cfg` redefines `BED_MESH_CALIBRATE` and calls the base
command with a fixed argument list. It does not forward `{rawparams}`. So
`BED_MESH_CALIBRATE METHOD=rapid_scan` would have **silently dropped METHOD**,
and the Eddy would have fallen back to slow point-by-point probing with no
error message.

Separately: mainline Klipper takes `METHOD=[manual|automatic|scan|rapid_scan]`
and has **no `SCAN_MODE` parameter**. BTT's own guides and sample configs still
print `METHOD=scan SCAN_MODE=rapid`. That does not error — the unknown
parameter is just ignored — so you get a slow scan while believing you asked
for a rapid one. Everything here uses `METHOD=rapid_scan`.

Adaptive meshing existed to make slow contact probing bearable. A full-bed Eddy
rapid scan takes under a minute and the result is cacheable. If you want it
back, copy KAMP's file into your own config dir and append
`METHOD=scan SCAN_MODE=rapid` to its `_BED_MESH_CALIBRATE` line — don't just
re-enable the include.

---

## What I could not do from here

I have no network path to `voron.local` — this session runs in an isolated
Anthropic cloud sandbox, not on your LAN, and mDNS names only resolve locally.
I verified: no DNS, no route, no SSH client, and the desktop bridge has no
network either.

So everything above is written but nothing has been executed or tested against
your actual hardware. In particular I could not:

- read the Eddy's real serial ID
- measure the probe offsets
- run any calibration
- confirm your Klipper version's `descend_z` vs `z_offset` behaviour
- confirm whether `KAMP_Settings.cfg` or `KAMP_Settings (1).cfg` is the live
  one (they differ: the `(1)` copy has the includes enabled, and I assumed that
  one is current since it is the newer download)

If you want me driving this directly, enable the Claude Chrome extension in a
browser on the same network and I can work through Mainsail's console and file
editor.
