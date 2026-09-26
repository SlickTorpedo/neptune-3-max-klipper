# Firmware build configs

One `.config` per board and per image. Build on the Pi with:

```bash
cd ~/klipper   # or ~/katapult
make clean KCONFIG_CONFIG=<path>/klipper_sb2209.config
make KCONFIG_CONFIG=<path>/klipper_sb2209.config
```

The files were generated with `make olddefconfig` against Klipper `ce7002b` (2026-09-18) and Katapult
`ec59b9b` (2026-03-20). If the Pi's checkouts are newer, run `make menuconfig KCONFIG_CONFIG=...`
once and check that the values below still hold before building.

The CAN adapter is a **BTT U2C v2.1**, so the Octopus is **not** a USB-to-CAN bridge.

| File | Board | Key values | Source |
|---|---|---|---|
| `klipper_octopus_usb.config` | Octopus v1.1, STM32F446ZET6 (chip read off the board 2026-09-25) | 32KiB bootloader, 12 MHz crystal (crystal can reads 12.000), USB on PA11/PA12 | Esoterical guide, Octopus page, F446 screenshots. The bridge config there differs only in the communication interface. |
| `katapult_sb2209.config` | EBB SB2209 CAN v1.0, STM32G0B1 | 8KiB deployer + 8KiB app offset, 8 MHz crystal, CAN on PB0/PB1, 1,000,000, double-reset entry, status LED PA13 | Esoterical guide, "BigTreeTech SB2209 and SB2240" page, Katapult screenshot |
| `klipper_sb2209.config` | same | 8KiB bootloader, 8 MHz crystal, CAN on PB0/PB1, 1,000,000 | same page, Klipper screenshot. 8 MHz crystal (X1) and FDCAN2 on PB0/PB1 are also confirmed in BTT's `BIGTREETECH EBB SB2209 CAN V1.0_SCH.pdf`. |
| `katapult_eddy.config` | Eddy Duo, RP2040 | W25Q080 CLKDIV 2, 16KiB bootloader, CAN RX gpio4 / TX gpio5, 1,000,000, no status LED | Esoterical guide, "BigTreeTech Eddy Duo" page, Katapult screenshot. **Only needed for a USB recovery.** The plan is to use whatever bootloader the Eddy already has. |
| `klipper_eddy.config` | same | 16KiB bootloader, CAN RX gpio4 / TX gpio5, 1,000,000 | same page, Klipper screenshot |

Notes:

- **Eddy offset:** on the RP2040, Katapult always launches the application at `0x10004000` (fixed in
  `katapult/src/rp2040/Kconfig`, `LAUNCH_APP_ADDRESS`). So any Katapult/CanBoot the Eddy shipped
  with requires the 16KiB bootloader offset, and there's nothing to guess.
- **Eddy I2C bus:** BTT's sample config uses `i2c0f` (i2c0 on GPIO20/21), which doesn't clash with
  the CAN pins (GPIO4/5).
- **CAN bitrate:** every image uses 1,000,000, matching `can0` on the Pi.
