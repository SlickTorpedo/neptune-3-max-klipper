# neptune-3-max-klipper

> ### ⚠️ Read this first
>
> **Until this note is removed, most of the writing in this repo — this README and the docs
> alongside it — is Claude's, not a record of a finished build.** It's a plan being drafted
> alongside the work, not instructions that have been executed and verified.
>
> If you found this repo looking to do the same conversion yourself: **don't follow it yet.**
> Nothing here has been proven on a physical machine. Pin assignments, wiring notes, config
> values and part choices may be wrong, out of date, or simply untested. Come back when this
> note is gone — that's the signal the build is actually done and the docs describe reality.

Gutting an Elegoo Neptune 3 Max and rebuilding it as a real printer: Klipper on a BTT Octopus v1.1,
an SB2209 CAN toolhead on a single umbilical, and a Stealthburner + CW2 carrying an X1C hotend.
The frame, the gantry and the 420 × 420 × 500 build volume stay. Almost nothing else does.

This repo is the config, the wiring notes, and the build log for that conversion.

> **Status: planning / teardown.** Parts are essentially all in hand — the scanning probe is the
> only outstanding purchase. Nothing is flight-tested. Anything marked `TBD` is still an open
> decision. Don't copy this onto your own machine expecting it to boot.

---

## Why

The Neptune 3 Max is a big printer sold at a small price, and every corner that got cut shows up
the moment you ask it for anything. The stock story:

- A closed-ish Marlin build on an Elegoo/MKS board with no meaningful tuning surface — no input
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

| Subsystem | Stock | Target |
|---|---|---|
| Firmware | Marlin (Elegoo build) | **Klipper** + Moonraker + Mainsail |
| Host | — | **Raspberry Pi** |
| Mainboard | Elegoo/MKS ZNP Robin Nano | **BTT Octopus v1.1** |
| Stepper drivers | Onboard, fixed | **TMC2209** UART |
| Toolhead wiring | Loose harness to mainboard | **CAN bus**, single umbilical |
| Toolhead board | — | **BTT SB2209** |
| CAN bridge | — | **None** — Octopus v1.1 in USB-to-CAN bridge mode |
| Extruder | Elegoo direct drive | **Stealthburner + CW2** |
| Hotend | PTFE-lined, ~260 °C max | **X1C / Bambu-style** ceramic-heater hotend |
| Probe | Elegoo ABL sensor | **TBD — scanning probe** (Beacon / Cartographer / BTT Eddy) |
| Z | Dual screws, belt-synced | **Independent dual Z** + `z_tilt` |
| Bed leveling | 4 manual knobs + mesh | `z_tilt` + **large adaptive mesh** (KAMP) |
| Part cooling | Single stock blower | Stealthburner dual 3010 + 4010 hotend fan |
| LEDs / feedback | Stock LCD | Stealthburner **WS2812B** status LEDs |

---

## Bill of materials

`[x]` = in hand, `[ ]` = not sourced yet.

### Electronics
- [x] BTT Octopus v1.1 + TMC2209 drivers
- [x] BTT SB2209 CAN toolhead board
- [x] Raspberry Pi host
- [x] Umbilical: CAN-rated 4-conductor (24 V, GND, CANH, CANL), sheathed
- [x] Second Z motor for independent Z
- [x] 24 V PSU — still worth **checking its rating against the new total load** before wiring
- [x] Ferrules, heat-shrink, crimp tooling

No U2C: the Octopus v1.1 runs USB-to-CAN bridge firmware and hosts the bus itself. That means
**the CAN jumpers and the 120 Ω termination jumper on the Octopus have to be set correctly**, and
the bus is terminated at exactly two points — the Octopus and the SB2209, nothing in between.

### Toolhead
- [x] Stealthburner shroud + CW2 kit
- [x] X1C-style hotend
- [x] 2× 3010 part cooling blowers, 1× 4010 hotend fan
- [x] Stealthburner LED PCB / WS2812B
- [ ] Hotend→CW2 adapter mount (the same problem already solved on the Voron — reuse that solution)
- [ ] **Scanning probe — the one part still to buy**

X1C hotend notes to nail down before it's wired to the SB2209: the **ceramic heater's wattage vs.
what the SB2209's heater output can actually supply**, and the **exact thermistor type** so
`sensor_type` is right the first time. Getting either wrong is a thermal-safety problem, not a
tuning problem.

### Mechanical
- [x] **Carriage adapter — designed.** Stealthburner mounts assume a Voron X carriage and the
      Neptune's X axis is not that, so this is a purpose-drawn plate. Design exists; it still has
      to be printed and fit-checked against the real carriage, and it is the part most likely to
      need a revision or two. Source files live in `cad/`.
- [x] Hardware to de-sync the Z screws and mount the second motor
- [x] Belts / tensioners
- [x] Umbilical anchor points

---

## Repo layout

Planned shape — directories appear as the work reaches them.

```
config/          Klipper configs (printer.cfg and everything it includes)
  printer.cfg      Root config
  hardware/        Steppers, heaters, fans, probe, MCU/CAN definitions
  macros/          PRINT_START / PRINT_END / PAUSE / RESUME / homing / etc.
docs/            Build notes, wiring diagrams, teardown photos, calibration records
firmware/        Klipper build .config files per MCU (Octopus bridge + SB2209), flashing notes
cad/             Adapter plates and printed brackets specific to this conversion
```

---

## Build phases

### Phase 0 — Document the stock machine *(before the screwdriver comes out)*
- [ ] Photograph the wiring harness end to end, both ends of every connector
- [ ] Record stock steps/mm, belt pitch, pulley tooth counts, lead-screw pitch
- [ ] Note thermistor types and heater wattage (bed especially — it gets reused)
- [ ] Record endstop types and wiring polarity (NO/NC)
- [ ] Back up the stock firmware and anything worth remembering from its settings

### Phase 1 — Teardown
- [ ] Strip the stock toolhead, mainboard, display and harness
- [ ] Keep: frame, gantry, rails, motors (for now), bed, PSU (pending verification)
- [ ] De-sync the Z screws and fit the second Z motor
- [ ] Inspect rails and lead screws; clean and re-lube before anything goes back on

### Phase 2 — Electronics bring-up (on the bench, not in the printer)
- [ ] Flash Katapult + Klipper to the Octopus as **USB-to-CAN bridge**; confirm it enumerates
- [ ] Set the Octopus CAN jumpers and confirm the 120 Ω termination jumper
- [ ] Flash the SB2209 over CAN; `canbus_query.py` returns its UUID
- [ ] Verify termination is present at both ends of the bus and nowhere else
- [ ] Every stepper moves the right direction under `FORCE_MOVE` before any endstop is trusted
- [ ] Both Z motors move independently and in the same direction

### Phase 3 — Mechanical install
- [ ] Mount the Octopus and the Pi, route the umbilical, strain-relieve both ends
- [ ] Fit the carriage adapter + Stealthburner + X1C hotend
- [ ] Re-tension belts, square the gantry, check both Z screws by hand

### Phase 4 — First motion
- [ ] Endstops read correctly *before* homing is attempted
- [ ] Home each axis individually with a finger on the kill switch
- [ ] `z_tilt` points defined for the bed's actual geometry; first `Z_TILT_ADJUST` watched closely
- [ ] Set and sanity-check `position_max` for the full 420 × 420 × 500 envelope
- [ ] Probe offsets, then Z offset — slowly, on a sacrificial surface

### Phase 5 — Heat and tune
- [ ] PID: hotend, then bed
- [ ] `verify_heater` / thermal runaway verified by actually provoking it
- [ ] `rotation_distance` calibration for XY and E
- [ ] Input shaping (ADXL on the SB2209), pressure advance, flow
- [ ] Mesh strategy for 420 mm²: full mesh baseline, then KAMP adaptive per print

### Phase 6 — Make it pleasant
- [ ] PRINT_START / PRINT_END with real pre-flight checks that abort early and loudly
- [ ] LED status states, filament runout handling, pause/resume that doesn't compound errors
- [ ] Timelapse, camera, remote monitoring

---

## Open decisions

1. **Which scanning probe.** Beacon, Cartographer or BTT Eddy. All three sidestep the fact that
   this carriage isn't Voron geometry (so Tap is out), and all three handle a 420 mm bed far
   better than point-by-point probing. Choice affects the carriage adapter design, so it wants
   deciding before the plate is cut.
2. **Bed.** The stock DC bed is slow and mediocre across 420 mm². AC bed + SSR is a real upgrade
   and a real safety project. Deferred, not dismissed.
3. **PSU headroom.** Add up the new load (bed + hotend + steppers + fans + Pi) against the
   supply's rating before it's all wired together.
4. **Carriage adapter revisions.** Design is done; how many print-and-fit cycles it takes is the
   open question, and it gates Phase 3.

---

## Safety

Blunt, because this build touches heaters and possibly mains:

- **Verify the X1C heater's power draw against the SB2209's heater output** before it's ever
  energized. An undersized MOSFET on an oversized heater is a fire, not a fault.
- **Mains wiring is not a "figure it out as you go" area.** If the AC bed happens, it gets proper
  ferrules, a grounded bed plate, a correctly rated SSR with a heatsink, and a thermal fuse.
- **Thermal runaway protection gets tested, not assumed.** Klipper's `verify_heater` only helps
  if the config is right and you've watched it trip.
- **Never leave the first heat soak unattended.**
- No unattended printing until PID, runaway protection and the first hundred hours are behind it.

---

## References

- Klipper — https://www.klipper3d.org/
- Klipper CAN bus guide — https://www.klipper3d.org/CANBUS.html
- Moonraker — https://moonraker.readthedocs.io/
- Mainsail — https://docs.mainsail.xyz/
- BTT Octopus v1.1 docs — https://github.com/bigtreetech/BIGTREETECH-OCTOPUS
- BTT SB2209 docs — https://github.com/bigtreetech/EBB
- Katapult (CAN bootloader) — https://github.com/Arksine/katapult
- Voron Stealthburner — https://github.com/VoronDesign/Voron-Stealthburner
- KAMP (adaptive meshing) — https://github.com/kyleisah/Klipper-Adaptive-Meshing-Purging

---

## License

TBD.
