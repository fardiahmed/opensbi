# bootloader/spacemit/opensbi: android16-riscv

Changes made for the Android 16 (AOSP, riscv64) bring-up of the BananaPi BPI-F3 (SpacemiT K1) and the BananaPi BPI-SM10 (SpacemiT K3), on branch `android16-riscv`.

OpenSBI for the SpacemiT K1 boards (BananaPi F3, MusePi Pro), replacing SpacemiT's v1.3 fork (pi-opensbi). Based on the RISE OP-TEE branch `dev-optee-mpxy-v8` of https://gitlab.com/riseproject/riscv-optee/opensbi (upstream OpenSBI 1.8 + MPXY/RPMI TEE service work), the base for running OP-TEE on the K1.

## Changes

- **platform: spacemit: k1: match the SpacemiT U-Boot device trees**: pi-u-boot (the BananaPi F3 / MusePi Pro bootloader) names the SoC "spacemit,k1x" in the device tree it hands to OpenSBI, so neither the K1 platform code (CCI snoops, reset vectors) nor the SpacemiT HSM driver probed.
- **platform: spacemit: k1: keep the console UART enabled**: The K1 UART is a PXA/XScale 8250 that only runs while IER.UUE is set, but the SpacemiT device trees describe it as "ns16550", so uart8250_init() cleared IER and silenced the console (and the U-Boot that follows).
- **platform: spacemit: k1: declare the X60 extensions that need menvcfg**: The SpacemiT device trees only give riscv,isa = "rv64imafdcv", so Zicbom/Zicboz/Svpbmt were not detected and menvcfg kept CBCFE/CBIE/CBZE/PBMTE clear: U-Boot died on its first cbo.flush.
- **lib: sbi: keep the vector misaligned emulation within the hart stack**: mask[VLEN_MAX / 8] with VLEN_MAX = 65536 is 8 KiB, the whole default hart stack: the first misaligned vector store from Android userspace (bionic RVV string routines) overflowed it and the hart died in a nested store fault. Upstream reworked this in a8be5e9478 with a 1024-bit chunked buffer; cap VLEN_MAX the same way (the K1 has VLEN 256).
- **platform: generic: spacemit: k1: fix wrong address definitions**: PMU_AP_CORE2_IDLE_CFG and PMU_AP_CORE3_IDLE_CFG are not continuous with PMU_AP_CORE0_IDLE_CFG and PMU_AP_CORE1_IDLE_CFG. They are at PMU AP base + 0x160 and + 0x164, matching the vendor OpenSBI definitions. After fixing these addresses, the intermediate cluster offset macros are redundant now, so define the wakeup and idle registers directly as PMU_AP_BASE offsets. This makes the actual register addresses easier to inspect and compare against the vendor code.

## Notes

- Verified on the BananaPi F3: Android boots, 8 harts via the SpacemiT HSM driver, SBI 3.0 (FWFT, PMU, MPXY), reboot through the P1 PMIC driver.
- Compared with pi-opensbi: no system suspend (`mem_sleep` is s2idle only) and no M-mode reset device (U-Boot `reset` fails; Linux reboots through spacemit-p1-reboot).
- OP-TEE: with pi-u-boot's k1-x-optee.dtsi, OpenSBI starts OP-TEE (optee_os plat-spacemit) in a trusted domain at 0x36000000, then U-Boot in an untrusted domain; OP-TEE and Linux talk over SBI MPXY/RPMI. Verified on the F3: OP-TEE initializes all 8 harts and Linux brings up 8 CPUs.
- Built by `build-bootloaders` (opensbi.src = opensbi, generic platform, `defconfig`) from `./build.sh k1`.

## Build

```
./build.sh k1 --bootloader-only   # -> vendor/spacemit/{k1,musepi-pro}/bootloader/fw_dynamic.itb
```
