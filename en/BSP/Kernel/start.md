# Mainline Linux Kernel User Guide

## 1. Overview

This document describes the basic workflow for using the mainline Linux Kernel on the platform, including obtaining the source code, checking the Firefly boards supported by mainline, and configuring and building the Kernel.

The Firefly ROC-RK3588-RT is used as an example. Its device tree is:

```text
arch/arm64/boot/dts/rockchip/rk3588-roc-rt.dts
```

The mainline Kernel and the Rockchip BSP Kernel do not provide exactly the same feature coverage. The presence of a board-level DTS in a mainline release and the ability to boot with it do not mean that features such as the VPU, NPU, GPU, ISP, HDMI, camera, PCIe, and low-power modes have reached the same level of support as in the BSP Kernel.

## 2. Obtaining the Source Code

The official mainline Linux source code is maintained by Linus Torvalds:

```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
```

If the selected version does not contain the DTS for the target board, use a newer mainline version that includes it.

## 3. Firefly Boards Supported by Mainline

| SoC | Board | DTS | Build Target |
| --- | --- | --- | --- |
| PX30 | Core-PX30-JD4 + MB-JD4-PX30 | `px30-firefly-jd4-core-mb.dts` | `rockchip/px30-firefly-jd4-core-mb.dtb` |
| RK3308 | ROC-RK3308-CC | `rk3308-roc-cc.dts` | `rockchip/rk3308-roc-cc.dtb` |
| RK3328 | ROC-RK3328-CC | `rk3328-roc-cc.dts` | `rockchip/rk3328-roc-cc.dtb` |
| RK3328 | ROC-RK3328-PC | `rk3328-roc-pc.dts` | `rockchip/rk3328-roc-pc.dtb` |
| RK3399 | Firefly-RK3399 | `rk3399-firefly.dts` | `rockchip/rk3399-firefly.dtb` |
| RK3399 | ROC-RK3399-PC | `rk3399-roc-pc.dts` | `rockchip/rk3399-roc-pc.dtb` |
| RK3399 | ROC-RK3399-PC Mezzanine | `rk3399-roc-pc-mezzanine.dts` | `rockchip/rk3399-roc-pc-mezzanine.dtb` |
| RK3399 | ROC-RK3399-PC-PLUS | `rk3399-roc-pc-plus.dts` | `rockchip/rk3399-roc-pc-plus.dtb` |
| RK3566 | Station M2 | `rk3566-roc-pc.dts` | `rockchip/rk3566-roc-pc.dtb` |
| RK3568 | Station P2 | `rk3568-roc-pc.dts` | `rockchip/rk3568-roc-pc.dtb` |
| RK3576 | ROC-RK3576-PC | `rk3576-roc-pc.dts` | `rockchip/rk3576-roc-pc.dtb` |
| RK3588 | ITX-3588J | `rk3588-firefly-itx-3588j.dts` | `rockchip/rk3588-firefly-itx-3588j.dtb` |
| RK3588 | ROC-RK3588-RT | `rk3588-roc-rt.dts` | `rockchip/rk3588-roc-rt.dtb` |
| RK3588S | Station M3 | `rk3588s-roc-pc.dts` | `rockchip/rk3588s-roc-pc.dtb` |

> The board list varies with the Linux version. The presence of a DTS only means that mainline includes the board-level description. Whether specific interfaces are available still depends on driver support in the version being used.

## 4. Preparing the Build Environment

Install the following packages on a Debian/Ubuntu host:

```bash
sudo apt update
sudo apt install \
  build-essential bc bison flex libssl-dev libelf-dev \
  device-tree-compiler gcc-aarch64-linux-gnu lz4 u-boot-tools
```

Set up the ARM64 cross-compilation environment:

```bash
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
```

If you use another AArch64 toolchain, change `CROSS_COMPILE` to its actual prefix. Do not mix `aarch64-linux-gnu-` and `aarch64-none-linux-gnu-` in the same build.

## 5. Building the Kernel

### 5.1 Configuring the Kernel

Use the ARM64 default configuration together with the custom configuration file `linux-rockchip-rk3588-current.config`:

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig linux-rockchip-rk3588-current.config
```

### 5.2 Building the Image, DTB, and Modules

Set the board name:

```bash
export BOARD=rk3588-roc-rt
```

Build the kernel, compressed kernel, ROC-RK3588-RT DTB, and modules:

```bash
make -j$(nproc) \
  ARCH=arm64 \
  CROSS_COMPILE=aarch64-linux-gnu- \
  Image Image.lz4 "rockchip/${BOARD}.dtb" modules
```

The main output files are:

```text
arch/arm64/boot/Image
arch/arm64/boot/Image.lz4
arch/arm64/boot/dts/rockchip/rk3588-roc-rt.dtb
```

Check the output files:

```bash
ls -lh \
  arch/arm64/boot/Image \
  arch/arm64/boot/Image.lz4 \
  arch/arm64/boot/dts/rockchip/rk3588-roc-rt.dtb
```

### 5.3 Installing the Modules

Install the modules to a temporary directory:

```bash
mkdir -p output/extboot

make -j$(nproc) \
  ARCH=arm64 \
  CROSS_COMPILE=aarch64-linux-gnu- \
  INSTALL_MOD_STRIP=1 \
  INSTALL_MOD_PATH="$PWD/output/extboot" \
  modules_install
```

The modules are installed to:

```text
output/extboot/lib/modules/<kernel-release>/
```

Display the Kernel release for the current build:

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- kernelrelease
```

When deploying the modules to the root file system, ensure that `/lib/modules/<kernel-release>/` exactly matches the Kernel that is actually booted.
