# CAN bring-up: U2C, SB2209 and Eddy Duo

Status: **bus up, all three MCUs ready in Klipper (2026-09-26)**; next is calibration (started 2026-09-25). Nothing on this page has been powered up yet unless a
step is ticked.

## Hardware as identified

| Part | Detail | How it was confirmed |
|---|---|---|
| Host | Raspberry Pi, `neptune.local`, user `pi`, Klipper already installed | user |
| Mainboard | BTT Octopus v1.1, **STM32F446ZET6**, 12 MHz crystal. Already runs USB Klipper (`usb-Klipper_stm32f446xx_330034000951343437373238-if00` in the bench config). | photo of the chip |
| CAN adapter | **BTT U2C v2.1** (arrives 2026-09-27). Replaces the planned Octopus USB-to-CAN bridge. | user decision |
| Toolhead board | BTT EBB SB2209 CAN v1.0, **STM32G0B1** | user read the chip |
| Probe | BTT Eddy Duo, switch set to **CAN**, own RP2040 node | user |
| Eddy connection | 4-pin CAN BUS port (5V / CAN-H / CAN-L / GND) on the SB2209's fan board (SB0000). That board gets CAN_P/CAN_N and 5V through the SB2209's 2×7 header (J1/J2), so the Eddy is a short stub on the same bus, powered by the SB2209's 5V 1.5A buck. | BTT SB2209 V1.0 schematic |

## Bus topology and termination

```
Pi ──USB-C── U2C [120R ON] ══ umbilical (24V, GND, CAN-H, CAN-L) ══ SB2209 [120R OFF] ── fan board ── Eddy Duo [own 120R, fixed/sealed]
                   ▲ 24V/GND screw terminals ← PSU V+/V− (pass-through only)
Pi ──USB── Octopus (plain USB Klipper; its RJ12 CAN port is unused)
```

- The bus is terminated at the U2C and at the SB2209. The Eddy stub is only a few cm long, so at
  1 Mbit it counts as the same end as the SB2209.
- **Check with power off:** measure between CAN-H and CAN-L. **~60 Ω** is correct. ~40 Ω means a
  third terminator is active somewhere (most likely on the Eddy), so find it and turn it off.
  ~120 Ω means one end is missing its jumper.
- The Octopus's permanent 120 Ω resistor isn't on this bus, because nothing plugs into its RJ12.

## SB2209 jumpers

| Jumper | Setting | Why |
|---|---|---|
| V_FAN1, V_FAN2 | **VIN** (bottom row, jumper horizontal) | fans are 24 V (**check the fan labels**). Each row connects that voltage to V_FAN. A vertical jumper would short two supply voltages together. |
| V_4W FAN | none | 4-pin fan port unused |
| V_Proximity, NPN | none | no inductive probe |
| 2.2K (JP2, "PT1000") | **none** | puts 4.12k in parallel with 4.7k (= 2.2k) as the TH0 pull-up, which is only for a PT1000. The hotend uses Generic 3950. |
| 120R | **OFF** | With it on, CAN-H↔CAN-L read 41 Ω: the Eddy Duo has its own terminator switched on. The Eddy is physically the last node, so it terminates that end. With this jumper off: 60 Ω (2026-09-26). |
| USB_5V | **ON only while flashing over USB**, and never with 24V connected | powers the board from USB for DFU |

## Runbook

### A. Host prep (Pi on; toolhead disconnected)
- [x] SSH key access for Claude (`ssh neptune`); narrow NOPASSWD sudoers rule in `/etc/sudoers.d/010-klipper-flash` (dfu-util, systemctl klipper, dmesg, ip)
- [x] `git clone https://github.com/Arksine/katapult ~/katapult` (ec59b9b)
- [x] (already in place on 2026-09-25, matched the guide exactly) `can0` via systemd-networkd, following the current Esoterical "Getting Started" page. The old
      ifupdown `interfaces.d/can0` method has been retired there.
  - enable, start (unmask first if needed) `systemd-networkd`; disable `systemd-networkd-wait-online`
  - `/etc/udev/rules.d/10-can.rules`: `SUBSYSTEM=="net", ACTION=="change|add", KERNEL=="can*"  ATTR{tx_queue_len}="128"`
  - `/etc/systemd/network/25-can.network`: `[Match] Name=can*` / `[CAN] BitRate=1M` / `[Link] RequiredForOnline=no`
  - reboot
- [x] Pi Klipper is `77d5d942e` (2026-05-04). All new images are built from this same checkout, so the Octopus needs no reflash. Built images are in `~/fw-out/` on the Pi.

### B. SB2209 over USB (umbilical **disconnected**)
- [x] USB_5V jumper on, USB-C from Pi to SB2209. **The first cable failed to enumerate** (`device descriptor read/64, error -32` on two ports); a different cable worked.
- [x] Hold BOOT and RESET, release RESET, release BOOT. `lsusb` shows `0483:df11` (DFU serial 204F37854130)
- [x] 2026-09-25: flashed the `katapult_sb2209` build (4564 bytes, mass-erase, OK; the trailing `get_status` error is expected): `sudo dfu-util -R -a 0 -s 0x08000000:mass-erase:force:leave -D ~/katapult/out/katapult.bin -d 0483:df11` (from Esoterical toolhead_flashing)
- [ ] Remove USB, **remove USB_5V jumper**

### C. U2C and umbilical (from 2026-09-27)
- [x] 2026-09-26: U2C flashed with Esoterical's fixed `G0B1_U2C_V2.bin` (sha256 6c5c462b…0ff, matches the guide repo). It was running stock `budgetcan`, and `dfu-util -e -d 1d50:606f` put it into DFU without the BOOT button. Replugged: `1d50:606f`, `can0` UP at 1000000, qlen 128, `gs_usb` loaded.
- [ ] U2C 120R jumper on; U2C 24V/GND terminals to PSU V+/V−
- [ ] Umbilical plugged into the U2C and the SB2209
- [ ] **Power off:** ~60 Ω CAN-H↔CAN-L; no continuity from 24V to either CAN line
- [ ] Power on: `ip -details link show can0` shows UP and bitrate 1000000
- [ ] `sudo systemctl stop klipper`, then `~/katapult/scripts/flashtool.py -i can0 -q`
- [ ] Flash Klipper to the SB2209 through Katapult
- [ ] Eddy: listed as Katapult/CanBoot? If yes, flash `klipper_eddy`. If no, **stop** and go over the options.

## eddy-ng (installed 2026-09-25)

The user chose [eddy-ng](https://github.com/vvuk/eddy-ng) over BTT's stock Eddy flow. It replaces BTT's hot
temperature-compensation calibration with a cold `PROBE_EDDY_NG_SETUP` plus a nozzle "tap" before each print.
It's installed on the Pi, and `~/fw-out/klipper_eddy.bin` (52696 bytes) includes it. The eddy-ng docs say
tap works best with the **bottom of the Eddy's coil PCB about 2.95 mm above the nozzle tip** (about 1.75 mm
from the case bottom to the tip). Homing and meshing work at other heights; tap is the sensitive part.

## UUIDs

| Node | UUID | Application seen |
|---|---|---|
| SB2209 | `e0ee669c3821` | Katapult (ours, app 0x8002000) → Klipper, flashed 2026-09-26 |
| Eddy Duo | `6e4e53261cb4` | BTT factory Katapult/CanBoot (protocol 1.0.0, app 0x10004000) → Klipper + eddy-ng, flashed 2026-09-26, verified SHA 756A370B… |

## First power-on (2026-09-26)

- `can0` UP at 1M. First query: both nodes in Katapult. Identified with `flashtool.py -s`: stm32g0b1 = SB2209, rp2040 = Eddy.
  flashtool refuses to flash an image whose MCU type doesn't match, which is a useful safety net.
- The Pi came back as `neptune-2.local` (mDNS name clash). Connect with `ssh -o HostName=neptune-2.local -o HostKeyAlias=neptune.local neptune`.
- Klipper ready. Readings at room temperature: extruder 26.6 °C (bed 26.7 °C), SB2209 board 31 °C, Eddy NTC 36 °C, Eddy MCU 41 °C.
  ADXL (x, y, z) = (-919, -444, 9697), with gravity on Z. `PROBE_EDDY_NG_STATUS`: about 3.198 MHz, status 0x48 UNREADCONV1 DRDY (healthy, not calibrated).
- CAN stats: SB2209 rx_error 44, flat since startup; Eddy tx_retries about 23, flat.
- eddy-ng needs a non-zero probe offset and a `[bed_mesh]` section. Both are **placeholders** for now (x 0, y 20).
- The bench `[temperature_sensor hotend]` on the Octopus (PF4) reads about -51 °C: nothing is plugged in. It's stale and can go.
- Octopus firmware is v0.13.0-533 and the host is -642. It works; reflash the Octopus from the same tree when convenient.

## Function checks (2026-09-26)

- Part fan (M106 S102): the top fan spins and the bottom (hotend) fan doesn't, so FAN2 = part fan is correct.
- Stealthburner LEDs: not fitted; config section commented out.
- Filamatrix sensor on PB6: flips from not detected to detected when filament is inserted. Pin and polarity correct.
- **Known quirk: "Failed automated reset of MCU 'eddy'".** It happens whenever a config change forces the boards to reset (seen twice).
  The log shows the Eddy's clock carrying on, so it never executed the `reset` command. Klipper sends `reset`, pauses 15 ms and
  disconnects (`mcu.py _restart_via_command`), so there's no time for a host retransmit. The Eddy's software CAN (can2040 on the
  RP2040) can miss a frame that another node ACKs. **Workaround: run FIRMWARE_RESTART again; it has worked every time.**

## Calibration results

### Gantry rough level (2026-09-26)
Nozzle-to-bed with a ruler: X30 about 28 mm, X390 about 30 mm. `FORCE_MOVE STEPPER=stepper_z DISTANCE=2` (left up 2 mm), then both matched.
Z direction confirmed: positive = up, both motors together.

### Eddy offsets (2026-09-26)
The nozzle is centred side to side on the Eddy case. The case's square end is 16 mm behind the nozzle, and BTT's drawing puts the coil
centre 11.33 mm in from that end, so **x 0, y +27.33**. Y=0 is the front of the bed (the nozzle sits over the front edge at Y0), confirming +Y.
Case bottom is about 2.0 mm above the nozzle tip.

### Z_TILT (2026-09-26)
First attempt diverged (0.99 → 1.83 → 3.37, aborted) because **stepper_z is the RIGHT motor**, not the left as the bench config
labelled it. Proven with the Eddy: raising stepper_z 1 mm moved X390 +0.98 and X30 +0.17. The earlier rough level had therefore
raised the already-high side. The aborted adjustments were reversed with FORCE_MOVE, z_positions reordered (445 then -25), and the
labels fixed. Rerun: 0.982 → 0.021 → **0.0013 mm** in 3 rounds.

### eddy-ng setup (2026-09-26, bed cold, ~27 °C)
`PROBE_EDDY_NG_SETUP` at X210 Y210 with a paper test:
`Drive current 15: valid height: 0.000 to 15.000, freq spread 2.34% (3198802.6 - 3273722.6), Fit 0.0104` → reg/tap drive current 15.
First `G28 Z` via the Eddy was OK. `PROBE_EDDY_NG_PROBE_STATIC`: Z2 → 2.008 (±0.007, 3×), Z1 → 1.019, Z3 → 2.919. Saved with SAVE_CONFIG.
Redo warm (bed ~55 °C) once `[heater_bed]` is configured.

### PID (2026-09-26)
Hotend: `PID_CALIBRATE HEATER=extruder TARGET=250` → Kp 38.020 Ki 12.673 Kd 28.515 (reached 200 °C in about 25 s).
The hotend fan was found jammed (grinding) during this run and fixed; confirmed spinning at 60 °C.
Bed: `PID_CALIBRATE HEATER=heater_bed TARGET=60` → Kp 71.440 Ki 0.776 Kd 1644.914 (about 2 °C per 10 s at full power, DC MOSFET on HE0).

### eddy-ng setup, warm (bed 55 °C, Eddy NTC 44 °C) (2026-09-26)
Drive current 15, valid 0.001–15.000, spread 2.22%, fit 0.0105 (very close to cold). Static at Z2 → 2.004.
Tap 0.038/0.035/0.035 → **0.036, stddev 0.001**. Saved with SAVE_CONFIG (replaces the cold calibration).

### First mesh (bed 55 °C, after Z_TILT 0.291 → 0.014, tap stddev 0.017) (2026-09-26)
`BED_MESH_CALIBRATE METHOD=rapid_scan`, 21×21, saved as `default`. **Range 0.911 mm** (min -0.501, max +0.410).
Corners: FL +0.341, FR -0.454, BL +0.295, BR -0.152, centre -0.004. Middle row +0.28 … 0 … +0.30 (gantry bow / dish).
The front row falls from +0.34 on the left to -0.45 on the right, a left-right twist mostly at the front, which the gantry can't correct.
Tramming the bed with its corner screws should remove most of it.

### Bed tram, round 1 (cold, after Z_TILT) (2026-09-26)
SCREWS_TILT_CALCULATE (6 outer screws, LF reference), repeated twice with near-identical results:
LM CW 00:12, LB CW 00:13, RB CW 00:57, RM CW 00:06, RF CW 01:21 (RF 0.67 mm low vs LF).
Klipper CW-M3: CW = less gap = raises that corner.

### First tap (2026-09-26, cold nozzle, clean, no filament)
`PROBE_EDDY_NG_TAP` at the bed centre: taps -0.113 / -0.100 / -0.105 → **-0.106, stddev 0.005**, overshoot 0.035, sensor offset 0.105 at z=2.
The contact point was about 0.1 mm (one paper thickness) below the paper-test zero, as expected.
Paper check after the tap: Z0.3 free, Z0.2 slight drag, Z0.1 more drag, Z0 drag (eddy-ng's target is "a good amount of friction" at Z0).
`tap_adjust_z` left at 0 until a first-layer print says otherwise.
