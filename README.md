# neptune-3-max-klipper

> ### ⚠️ Read this first
>
> **Until this note is removed, most of the writing in this repo, including this README and the docs
> alongside it, is Claude's, not a record of a finished build.** It started as a plan drafted
> alongside the work.
>
> Since 2026-09-26, part of it has been **verified on the real machine**: the CAN bus, the toolhead
> board, the Eddy probe, homing, gantry levelling and heater tuning. Those parts are ticked below,
> and the measurements are in [`docs/can-bringup.md`](docs/can-bringup.md). Everything else is still
> plan. The printer has **not printed yet**.
>
> If you found this repo looking to do the same conversion yourself: **don't follow it blindly yet.**
> Come back when this note is gone. That's the signal the build is done and the docs describe reality.

Gutting an Elegoo Neptune 3 Max and rebuilding it as a real printer: Klipper on a BTT Octopus v1.1,
an SB2209 CAN toolhead on a single umbilical, and a Stealthburner + CW2 carrying an X1C hotend,
probed by a BTT Eddy Duo running [eddy-ng](https://github.com/vvuk/eddy-ng).
The frame, the gantry and the 420 × 420 × 500 build volume stay. Almost nothing else does.

This repo is the config, the wiring notes, and the build log for that conversion.

> **Status (2026-09-26): electronics up, motion and probing calibrated, not printed yet.**
> All three MCUs (Octopus, SB2209, Eddy Duo) are running Klipper. Z homes on the Eddy, `Z_TILT_ADJUST`
> levels the gantry to about 0.001 mm, and both heaters are PID-tuned.
> **Next:** silicone bed spacers and a tram, an extruder check, PRINT_START / PRINT_END, then a first-layer print.

---

## Why

The Neptune 3 Max is a big printer sold at a small price, and every corner that got cut shows up
the moment you ask it for anything. The stock story:

- A closed-ish Marlin build on an Elegoo/MKS board with no meaningful tuning surface: no input
  shaping worth the name, no pressure advance you can iterate on, no real macro system.
- A PTFE-lined hotend that caps out around 260 °C and clogs if you look at it funny, on a
  proprietary extruder assembly.
- A 420 × 420 DC-powered bed that takes forever to come up to temperature and never gets flat.
- Stock ABL that samples too few points for a bed this size and fights a gantry that sags.
- Wiring that makes the toolhead a bird's nest of loose conductors dragging across the gantry.

The frame, the Z height and the sheer footprint are the parts worth keeping. Everything that
touches filament or decides where the nozzle goes gets replaced.

---

## The plan, in one table

| Subsystem | Stock | Target | State |
|---|---|---|---|
| Firmware | Marlin (Elegoo build) | **Klipper** + Moonraker + Mainsail | ✅ running |
| Host | — | **Raspberry Pi** | ✅ |
| Mainboard | Elegoo/MKS ZNP Robin Nano | **BTT Octopus v1.1** (STM32F446), plain USB Klipper | ✅ |
| Stepper drivers | Onboard, fixed | **TMC2209** UART | ✅ |
| Toolhead wiring | Loose harness to mainboard | **CAN bus**, single umbilical, 1 Mbit | ✅ |
| CAN adapter | — | **BTT U2C v2.1** | ✅ |
| Toolhead board | — | **BTT SB2209** (STM32G0B1), Katapult + Klipper | ✅ |
| Extruder | Elegoo direct drive | **Stealthburner + CW2** (Filamatrix) | fitted; direction/rotation not checked |
| Hotend | PTFE-lined, ~260 °C max | **X1C / Bambu-style** ceramic heater, Generic 3950 | ✅ PID-tuned |
| Probe | Elegoo ABL sensor | **BTT Eddy Duo** in CAN mode (its own node) + **eddy-ng** | ✅ calibrated, tap works |
| Z | Dual screws, belt-synced | **Independent dual Z** + `z_tilt` | ✅ |
| Bed leveling | Knobs + mesh | `z_tilt` + eddy-ng tap + Eddy rapid-scan adaptive mesh | ✅ mesh / 🔜 tram |
| Heated bed | Stock DC bed | Stock DC bed via an **external DC MOSFET** on Octopus HE0 | ✅ PID-tuned |
| Part cooling | Single stock blower | Stealthburner part fan + 4010 hotend fan | ✅ both verified |
| LEDs / feedback | Stock LCD | Stealthburner WS2812B LEDs | not fitted (config ready, commented out) |

---

## Bill of materials

`[x]` = in hand, `[ ]` = not sourced yet.

### Electronics
- [x] BTT Octopus v1.1 + TMC2209 drivers
- [x] BTT SB2209 CAN toolhead board
- [x] BTT U2C v2.1 USB-to-CAN adapter (arrived early, 2026-09-26)
- [x] Raspberry Pi host
- [x] Umbilical: the SB2209 kit cable (24 V, GND, CANH, CANL) to the U2C; the U2C's 24 V/GND terminals are fed from the PSU
- [x] Second Z motor for independent Z
- [x] 24 V PSU. Still worth **checking its rating against the new total load**.
- [x] DC MOSFET module for the bed, driven from Octopus HE0
- [x] Ferrules, heat-shrink, crimp tooling

The CAN bus runs from the U2C to the toolhead. It's terminated at exactly two points: the **U2C**
and the **Eddy Duo**, which has its own built-in terminator. The SB2209's 120R jumper is **off**:
with it on, the bus measured 41 Ω; with it off, 60 Ω. Bring-up steps, jumpers, UUIDs and every
measurement are in [`docs/can-bringup.md`](docs/can-bringup.md).

### Toolhead
- [x] Stealthburner shroud + CW2 kit (Filamatrix, single filament switch on the SB2209, verified)
- [x] X1C-style hotend
- [x] Part cooling blower + 4010 hotend fan
- [x] Stealthburner LED PCB / WS2812B (not fitted yet)
- [ ] Hotend→CW2 adapter mount (the same problem already solved on the Voron; reuse that solution)
- [x] **BTT Eddy Duo** scanning probe (CAN mode, sealed in the Stealthburner; flashed over CAN without opening it)

Still open for the X1C hotend: confirm the **ceramic heater's wattage against what the SB2209's
heater output can safely supply**. The thermistor is settled: `Generic 3950` reads correctly at
room temperature and PID-tuned cleanly at 250 °C.

### Mechanical
- [x] **Carriage adapter.** Ben Ford's Neptune 3/4 Stealthburner plate (Printables 1185615), fitted.
      The Eddy coil sits 27.33 mm behind the nozzle, and the bottom of its case is about 2.0 mm
      above the nozzle tip.
- [x] Hardware to de-sync the Z screws and mount the second motor
- [x] Belts / tensioners
- [x] Umbilical anchor points
- [x] 18 mm silicone bed spacers (4 mm ID) to replace the bed springs; 7 needed (M3 screws)

---

## Repo layout

```
config/            Klipper configs (the Pi's ~/printer_data/config mirrors this)
  printer.cfg        Root config: Octopus, steppers, bed heater, and the SAVE_CONFIG block
  hardware/
    sb2209.cfg       Toolhead MCU, extruder, hotend, fans, ADXL, Filamatrix sensor
    eddy.cfg         Eddy Duo MCU + probe_eddy_ng
    eddy_homing.cfg  homing_override: Z always homes with the coil over the bed centre
    z_tilt.cfg       Dual-Z levelling (stepper_z is the RIGHT screw)
    screws_tilt.cfg  Bed tramming over the 6 outer bed screws
    bed_mesh.cfg     21×21 mesh, scanned at 2 mm
  macros/            PRINT_START / PRINT_END etc. (not written yet)
docs/
  can-bringup.md     The bring-up log: hardware IDs, jumpers, UUIDs, every calibration result
firmware/          Katapult/Klipper .config per MCU (Octopus, SB2209, Eddy Duo) + where each value came from
stl/               Printable parts, sorted by print colour
  toolhead/          Stealthburner + CW2 (FilamATrix) for the X1C hotend
cad/               Adapter plates and printed brackets specific to this conversion
```

`stl/` is organised so each colour folder can be imported whole and printed in one go. See
[`stl/toolhead/README.md`](stl/toolhead/README.md) for the layout and the variant choices.

**Keeping the Pi and the repo in sync:** `SAVE_CONFIG` writes calibration into `printer.cfg` on
the Pi. Copy it back into `config/` after every save.

---

## Build phases

### Phase 0: Document the stock machine *(before the screwdriver comes out)*
- [ ] Photograph the wiring harness end to end, both ends of every connector
- [ ] Record stock steps/mm, belt pitch, pulley tooth counts, lead-screw pitch
- [ ] Note thermistor types and heater wattage (bed especially, since it gets reused)
- [ ] Record endstop types and wiring polarity (NO/NC)
- [ ] Back up the stock firmware and anything worth remembering from its settings

### Phase 1: Teardown
- [x] Strip the stock toolhead, mainboard, display and harness
- [x] Keep: frame, gantry, rails, motors, bed, PSU
- [x] De-sync the Z screws and fit the second Z motor
- [ ] Inspect rails and lead screws; clean and re-lube

### Phase 2: Electronics bring-up
- [x] U2C on `can0` at 1 Mbit (fixed U2C firmware flashed); Octopus on plain USB Klipper
- [x] Katapult on the SB2209 via USB DFU, then Klipper over CAN; Eddy Duo flashed over CAN through its factory bootloader
- [x] Termination at both ends of the bus and nowhere else (60 Ω)
- [x] Both Z motors move together and in the same direction (positive = up)
- [ ] Extruder direction under a cold/hot extrude test

### Phase 3: Mechanical install
- [x] Mount the Octopus and the Pi, route the umbilical
- [x] Fit the carriage adapter + Stealthburner + X1C hotend
- [ ] Re-tension belts, square the gantry
- [ ] Swap bed springs for silicone spacers, then tram with `SCREWS_TILT_CALCULATE`

### Phase 4: First motion
- [x] X/Y homing on the stock switches
- [x] Z homes on the Eddy (`probe:z_virtual_endstop`, eddy-ng)
- [x] `z_tilt` defined for the real screw positions; `Z_TILT_ADJUST` converges to about 0.001 mm.
      The first attempt diverged because the bench config had the Z motors labelled backwards.
- [x] Probe offsets measured (x 0, y +27.33); Z offset comes from the eddy-ng tap before each print
- [ ] Sanity-check `position_max` for the full 420 × 420 × 500 envelope

### Phase 5: Heat and tune
- [x] PID: hotend (250 °C) and bed (60 °C)
- [x] eddy-ng setup, warm; tap standard deviation 0.001 mm
- [x] Mesh baseline: 21×21 rapid scan, range 0.91 mm (about 0.24 mm gantry bow plus a bed twist, to be trammed out)
- [ ] `verify_heater` / thermal runaway verified by actually provoking it
- [ ] `rotation_distance` calibration for E (XY too)
- [ ] Input shaping (ADXL on the SB2209, responding), pressure advance, flow

### Phase 6: Make it pleasant
- [ ] PRINT_START / PRINT_END: home, heat, Z_TILT, clean nozzle at 150 °C, tap, adaptive mesh, purge
- [ ] Filament runout handling (Filamatrix), pause/resume that doesn't compound errors
- [ ] Stealthburner LEDs (optional)
- [ ] Timelapse, camera, remote monitoring

---

## Open decisions

1. **Bed.** Now the stock DC bed through an external MOSFET. AC bed + SSR is still a real upgrade
   and a real safety project. Deferred, not dismissed.
2. **PSU headroom.** Add up the new load (bed + hotend + steppers + fans + Pi) against the
   supply's rating.
3. **Gantry.** It visibly bows in the middle (about 0.24 mm on the Eddy). The mesh compensates for
   now; the gantry is due to be replaced eventually.
4. **Stock Elegoo runout sensor.** Still wired (Octopus PG10) but not mounted. It pauses nothing and
   isn't used by any macro.
5. **Eddy height.** eddy-ng's ideal is about 1.75 mm from the case bottom to the nozzle tip; this
   mount gives about 2.0. Tap works well as is, so only revisit it if taps get unreliable.

## Known quirks

- **"Failed automated reset of MCU 'eddy'"** after a config change: the Eddy (software CAN on its
  RP2040) sometimes misses Klipper's reset command. Run `FIRMWARE_RESTART` again; it has worked every time.
- **The Pi sometimes comes back as `neptune-2.local`** (mDNS name clash). It returns to
  `neptune.local` on its own later.
- **eddy-ng patches Klipper's source.** Before updating Klipper: `~/eddy-ng/install.sh --uninstall`,
  update, reinstall, then rebuild and reflash the MCUs. See [`firmware/README.md`](firmware/README.md).

---

## Safety

Blunt, because this build touches heaters and possibly mains:

- **Verify the X1C heater's power draw against the SB2209's heater output.** An undersized MOSFET
  on an oversized heater is a fire, not a fault.
- **Mains wiring is not a "figure it out as you go" area.** Earth to every metal case, proper
  connectors, strain relief. If the AC bed happens, it also gets a grounded bed plate, a correctly
  rated SSR with a heatsink, and a thermal fuse.
- **Thermal runaway protection gets tested, not assumed.** Klipper's `verify_heater` only helps
  if the config is right and you've watched it trip.
- **The hotend fan has no speed sensor.** It was once found jammed mid-heat. A stalled hotend fan
  cooks the Stealthburner's printed parts, so look at it after any toolhead work.
- **Never leave the first heat soak unattended.**
- No unattended printing until PID, runaway protection and the first hundred hours are behind it.

---

## References

- Klipper: https://www.klipper3d.org/
- Klipper CAN bus guide: https://www.klipper3d.org/CANBUS.html
- Esoterical's CANBus guide: https://canbus.esoterical.online/
- Moonraker: https://moonraker.readthedocs.io/
- Mainsail: https://docs.mainsail.xyz/
- BTT Octopus v1.1 docs: https://github.com/bigtreetech/BIGTREETECH-OCTOPUS
- BTT SB2209 docs: https://github.com/bigtreetech/EBB
- BTT Eddy: https://github.com/bigtreetech/Eddy
- eddy-ng: https://github.com/vvuk/eddy-ng (wiki: https://github.com/vvuk/eddy-ng/wiki)
- Katapult (CAN bootloader): https://github.com/Arksine/katapult
- Voron Stealthburner: https://github.com/VoronDesign/Voron-Stealthburner
- Neptune 3 Max bed screw positions: https://github.com/TheFeralEngineer/Klipper-for-Elegoo-Neptune-series-3D-Printers

---

## License

TBD.
