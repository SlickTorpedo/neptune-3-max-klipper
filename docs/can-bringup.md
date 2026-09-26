# CAN bring-up: U2C, SB2209 and Eddy Duo

Status: **in progress** (started 2026-09-25). Nothing on this page has been powered up yet unless a
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
Pi ──USB-C── U2C [120R ON] ══ umbilical (24V, GND, CAN-H, CAN-L) ══ SB2209 [120R ON] ── fan board ── Eddy Duo
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
| 120R | **ON** | far end of the bus |
| USB_5V | **ON only while flashing over USB**, and never with 24V connected | powers the board from USB for DFU |

## Runbook

### A. Host prep (Pi on; toolhead disconnected)
- [ ] SSH key access for Claude (user runs `ssh-copy-id`)
- [ ] `git clone https://github.com/Arksine/katapult ~/katapult`
- [ ] `can0` via systemd-networkd, following the current Esoterical "Getting Started" page. The old
      ifupdown `interfaces.d/can0` method has been retired there.
  - enable, start (unmask first if needed) `systemd-networkd`; disable `systemd-networkd-wait-online`
  - `/etc/udev/rules.d/10-can.rules`: `SUBSYSTEM=="net", ACTION=="change|add", KERNEL=="can*"  ATTR{tx_queue_len}="128"`
  - `/etc/systemd/network/25-can.network`: `[Match] Name=can*` / `[CAN] BitRate=1M` / `[Link] RequiredForOnline=no`
  - reboot
- [ ] Confirm Octopus Klipper is current; reflash over USB only if Klipper reports a version mismatch

### B. SB2209 over USB (umbilical **disconnected**)
- [ ] USB_5V jumper on, USB-C from Pi to SB2209
- [ ] Hold BOOT and RESET, release RESET, release BOOT. `lsusb` shows `0483:df11`
- [ ] Flash `katapult_sb2209` build: `sudo dfu-util -R -a 0 -s 0x08000000:mass-erase:force:leave -D ~/katapult/out/katapult.bin -d 0483:df11` (from Esoterical toolhead_flashing)
- [ ] Remove USB, **remove USB_5V jumper**

### C. U2C and umbilical (from 2026-09-27)
- [ ] U2C: flash Esoterical's fixed `G0B1_U2C_V2.bin` (hold BOOT while plugging it in, then `dfu-util`)
- [ ] U2C 120R jumper on; U2C 24V/GND terminals to PSU V+/V−
- [ ] Umbilical plugged into the U2C and the SB2209
- [ ] **Power off:** ~60 Ω CAN-H↔CAN-L; no continuity from 24V to either CAN line
- [ ] Power on: `ip -details link show can0` shows UP and bitrate 1000000
- [ ] `sudo systemctl stop klipper`, then `~/katapult/scripts/flashtool.py -i can0 -q`
- [ ] Flash Klipper to the SB2209 through Katapult
- [ ] Eddy: listed as Katapult/CanBoot? If yes, flash `klipper_eddy`. If no, **stop** and go over the options.

## UUIDs

| Node | UUID | Application seen |
|---|---|---|
| SB2209 | TBD | |
| Eddy Duo | TBD | |

## Calibration results

None yet.
