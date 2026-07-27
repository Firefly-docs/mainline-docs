# Mainline U-Boot User Guide

## 1. Overview

This document describes the basic workflow for using mainline U-Boot on Rockchip platforms, including preparing the source code, selecting an existing board configuration, building, checking the output files, and flashing.

Mainline Rockchip U-Boot builds generally use two repositories together:

| Repository | Purpose |
| --- | --- |
| U-Boot | U-Boot source code, board defconfigs, and device trees |
| rkbin | Rockchip DDR initialization code, BL31, and other binary firmware |

## 2. Preparing the Source Code

### 2.1 Obtaining the Source Code

It is recommended to place U-Boot and rkbin in the same working directory:

```bash
mkdir -p rockchip-mainline
cd rockchip-mainline

git clone https://github.com/u-boot/u-boot.git
git clone https://github.com/rockchip-linux/rkbin.git
```

Directory structure:

```text
rockchip-mainline/
├── u-boot/
└── rkbin/
```

For product development, pin the repositories to a verified tag or commit. Following the continuously changing default branch directly in a production build is not recommended.

Check the current versions:

```bash
git -C u-boot describe --always --dirty --tags
git -C rkbin describe --always --dirty --tags
```

### 2.2 Adding a Board

To add a board, add its corresponding defconfig and device tree.

- Create a configuration for the board under `configs/`. You can use `configs/evb-rk3588_defconfig` as the base, add or remove configuration options according to the board's features, and generate a new configuration file such as `roc-pc-rk3588s_defconfig`.
- Mainline U-Boot 2024.05 enabled the `OF_UPSTREAM` feature, and many device trees were migrated to `dts/upstream/src/arm64/`. You no longer need to add the device tree to `arch/arm/dts/Makefile`. Instead, set the default device tree through `CONFIG_DEFAULT_DEVICE_TREE` in the defconfig. The device trees under `dts/upstream/src/arm64/` are the same as those used by the Kernel, so you can copy the Kernel device tree there for use.

## 3. Building

### 3.1 Toolchain and Python

Install the AArch64 cross-compilation toolchain and basic dependencies. On Debian/Ubuntu, for example:

```bash
sudo apt update
sudo apt install \
  build-essential bc bison flex swig python3 python3-dev \
  device-tree-compiler libssl-dev libgnutls28-dev \
  gcc-aarch64-linux-gnu
```

### 3.2 Specifying TPL and BL31

Enter the U-Boot source directory:

```bash
cd rockchip-mainline/u-boot
```

Using the RK3588 firmware versions in the PDF as an example:

```bash
export ROCKCHIP_TPL=../rkbin/bin/rk35/rk3588_ddr_lp4_2112MHz_lp5_2400MHz_v1.18.bin
export BL31=../rkbin/bin/rk35/rk3588_bl31_v1.48.elf
```

`ROCKCHIP_TPL` is the DDR initialization firmware and must match the SoC, DDR type, and board-level design. `BL31` is the ARM Trusted Firmware-A stage firmware. The rkbin file names change as versions are updated. If the example files do not exist, select versions matching the hardware from the current `rkbin/bin/rk35/` directory:

```bash
find ../rkbin/bin/rk35 -maxdepth 1 \
  \( -name 'rk3588_ddr_*.bin' -o -name 'rk3588_bl31_*.elf' \) \
  -print
```

### 3.3 Running the Build

First, list the existing RK3588/RK3588S defconfigs in mainline U-Boot:

```bash
find configs -maxdepth 1 -name '*rk3588*_defconfig' -printf '%f\n' | sort
```

Select an existing defconfig that matches the target board. The following example uses the mainline EVB configuration:

```bash
export DEFCONFIG=evb-rk3588_defconfig
```

Clean the old configuration, load the defconfig, and then build:

```bash
make mrproper
make "$DEFCONFIG"
make CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
```

Alternatively, export the architecture and toolchain as environment variables:

```bash
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-

make "$DEFCONFIG"
make -j$(nproc)
```

Record the version and configuration after the build:

```bash
git describe --always --dirty --tags
grep -E 'CONFIG_DEFAULT_DEVICE_TREE|CONFIG_OF_UPSTREAM' .config
```

## 4. TEE Warning

If BL32/OP-TEE is not specified, the following warning may appear near the end of the build:

```text
Image 'simple-bin' is missing optional external blobs but is still functional: tee-os
```

When `tee-os` is optional external firmware in the selected configuration, this warning does not prevent U-Boot output files from being generated.

## 5. Build Output

| File | Purpose |
| --- | --- |
| `idbloader.img` | Rockchip first-stage loader, flashed to the `idbloader` partition |
| `u-boot.itb` | U-Boot FIT firmware, flashed to the `uboot` partition |
| `u-boot-rockchip.bin` | Contains `idbloader.img` and `u-boot.itb` at their specified offsets and can be written directly to an SD card or eMMC |
| `u-boot-rockchip-spi.bin` | Used for SPI NOR Flash; generated only when the corresponding configuration is enabled |

Check the output files:

```bash
ls -lh idbloader.img u-boot.itb u-boot-rockchip*.bin
```

If any output file is missing, first check the build log, TPL/BL31 paths, defconfig, and target storage-medium configuration.

## 6. Flashing

### 6.1 SD Card or eMMC

`u-boot-rockchip.bin` already contains its component firmware at the correct offsets. The write location specified in the PDF starts at the 64th 512-byte sector of the storage device, which is an offset of 32 KiB:

```bash
sudo dd \
  if=u-boot-rockchip.bin \
  of=/dev/mmcblk0 \
  bs=32K \
  seek=1 \
  conv=fsync,notrunc \
  status=progress
```

> Warning: `dd` directly overwrites a block device. `of` must be the entire target SD/eMMC device, not a file system partition, and it must not be the development host's system disk.

Before writing, it is recommended to run:

```bash
lsblk -o NAME,SIZE,MODEL,TYPE,MOUNTPOINTS
```

Unmount any mounted file systems on the target device before running `dd`. After writing is complete, run:

```bash
sync
```

### 6.2 SPI NOR Flash

After enabling the SPI boot configuration and generating `u-boot-rockchip-spi.bin`, use `flashcp` to write it to an MTD device:

```bash
sudo flashcp -v -p u-boot-rockchip-spi.bin /dev/mtd0
```

Before writing, confirm the MTD device, capacity, and existing partitions:

```bash
cat /proc/mtd
mtdinfo /dev/mtd0
```

Modifying the SPI boot firmware may prevent the device from booting. Back up the original firmware and prepare a Maskrom/Loader recovery procedure before proceeding.
