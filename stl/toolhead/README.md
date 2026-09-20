# Toolhead STLs

Stealthburner + Clockwork 2 (FilamATrix fork) with a Bambu/X1C-style hotend, sorted by
**print color** rather than by upstream source.

Each colour folder is meant to be imported whole: select the entire folder, drop it on the
plate, set one colour, print. No file-by-file picking, no leftover parts you didn't want.

## Folders

| Folder | Print in | What's in it |
|---|---|---|
| `primary/` | Primary colour | Main structural parts with no filename prefix |
| `accent/` | Accent colour | Everything upstream marked `[a]` |
| `clear/` | Clear / translucent | The LED diffuser — it has to pass light |
| `opaque/` | Opaque, ideally black | LED carrier and diffuser mask — these **block** light; a translucent print here bleeds and ruins the effect |
| `jigs/` | Anything | Assembly tools, not printer parts. Print once, keep in the toolbox |
| `alternates/` | **Don't print** | The roads not taken — see below |

## Quantities

**Every part here is quantity 1.** Nothing needs duplicating.

Voron's own file convention encodes quantity in the filename with an `_x2` / `_x3` / `_x4`
suffix — a part needed in multiples ships pre-named that way. None of the files in this set
carry a suffix, upstream or here, so a single copy of each is correct. If a part ever does
need multiples, it'll show up here as `name_x1.stl`, `name_x2.stl`, `name_x3.stl` so the
whole folder can still be imported in one go.

Filenames are otherwise unchanged from upstream, so they stay traceable to the source repos.

## What's in `alternates/` and why

These are the choose-one variants that lost. They're kept for reference in case a decision
gets revisited — **they are not part of the build as specified.**

| File | Why it's not being printed |
|---|---|
| `main_body_clockwork2_no_switch.stl` | AFC default (relies on buffer ramming instead of a toolhead sensor) |
| `main_body_clockwork2_dual_switch.stl` | Two switches; only one is needed here |
| `cable_cover_pcb.stl` | Pairs with the no-switch body |
| `[a]_knife_holder_olfa.stl` | Olfa blade variant; this build uses a standard razor |
| `razor_safety_cap_olfa.stl` | Same — Olfa variant |
| `chain_anchor_2hole.stl` / `chain_anchor_3hole.stl` | For cable chains. This build runs a CAN umbilical, so there's no chain to anchor |

**Chosen configuration:** single-switch CW2 main body + `cable_cover_pcb_w_sensor`, standard
razor blade, Filametrix cutter included, no cable chain.

## Print settings

Standard Voron guidance applies:

- **ABS or ASA.** These parts sit next to a hotend; PLA will sag and deform.
- 4 perimeters, 5 top/bottom layers, 40% infill
- No supports needed on any part — they're all oriented to print as-is
- The `clear/` diffuser wants a translucent filament; SLA prints of this part are common and
  look better than FDM if that's an option

## Sources

- FilamATrix (CW2 fork with the toolhead cutter) — https://github.com/thunderkeys/FilamATrix
- Voron Stealthburner (stock Stealthburner + CW2 parts) — https://github.com/VoronDesign/Voron-Stealthburner

The upstream READMEs that came with the download are preserved verbatim in `reference/`.
