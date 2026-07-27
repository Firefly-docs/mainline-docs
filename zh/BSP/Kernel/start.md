# 主线 Linux Kernel 使用指南

## 1. 概述

本文介绍平台主线 Linux Kernel 的基本使用流程，包括获取源码、查看主线已支持的 Firefly 板型、配置和编译 Kernel。

以 Firefly ROC-RK3588-RT 为例，使用的设备树为：

```text
arch/arm64/boot/dts/rockchip/rk3588-roc-rt.dts
```

主线 Kernel 与 Rockchip BSP Kernel 的功能覆盖并不完全相同。某个主线版本包含板级 DTS 并能够启动，不代表 VPU、NPU、GPU、ISP、HDMI、Camera、PCIe 和低功耗等功能均已达到 BSP Kernel 的支持状态。

## 2. 获取源码

Linux 官方主线源码由 Linus Torvalds 维护：

```bash
git clone https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git
cd linux
```

如果选定版本中不存在目标板型的 DTS，需要改用已包含该 DTS 的更新主线版本。

## 3. 主线支持的 Firefly 板型

| SoC | 板型 | DTS | 编译目标 |
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

> 板型列表随 Linux 版本变化。DTS 存在仅表示主线包含该板级描述，具体接口是否可用仍取决于对应驱动在所用版本中的支持情况。

## 4. 准备编译环境

Debian/Ubuntu 主机可安装以下软件包：

```bash
sudo apt update
sudo apt install \
  build-essential bc bison flex libssl-dev libelf-dev \
  device-tree-compiler gcc-aarch64-linux-gnu lz4 u-boot-tools
```

设置 ARM64 交叉编译环境：

```bash
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
```

如果使用其他 AArch64 工具链，请将 `CROSS_COMPILE` 改为实际前缀。不要在同一次构建中混用 `aarch64-linux-gnu-` 和 `aarch64-none-linux-gnu-`。

## 5. 编译Kernel

### 5.1 配置 Kernel

使用ARM64 默认配置和自己需求配置文件linux-rockchip-rk3588-current.config

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig linux-rockchip-rk3588-current.config
```

### 5.2 编译 Image、DTB 和模块

设置板型名称：

```bash
export BOARD=rk3588-roc-rt
```

编译内核、压缩内核、ROC-RK3588-RT DTB 和模块：

```bash
make -j$(nproc) \
  ARCH=arm64 \
  CROSS_COMPILE=aarch64-linux-gnu- \
  Image Image.lz4 "rockchip/${BOARD}.dtb" modules
```

主要产物如下：

```text
arch/arm64/boot/Image
arch/arm64/boot/Image.lz4
arch/arm64/boot/dts/rockchip/rk3588-roc-rt.dtb
```

检查产物：

```bash
ls -lh \
  arch/arm64/boot/Image \
  arch/arm64/boot/Image.lz4 \
  arch/arm64/boot/dts/rockchip/rk3588-roc-rt.dtb
```

### 5.3 安装模块

将模块安装到临时目录：

```bash
mkdir -p output/extboot

make -j$(nproc) \
  ARCH=arm64 \
  CROSS_COMPILE=aarch64-linux-gnu- \
  INSTALL_MOD_STRIP=1 \
  INSTALL_MOD_PATH="$PWD/output/extboot" \
  modules_install
```

模块会安装到：

```text
output/extboot/lib/modules/<kernel-release>/
```

查看本次构建的 Kernel release：

```bash
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- kernelrelease
```

将模块部署到根文件系统时，必须保证 `/lib/modules/<kernel-release>/` 与实际启动的 Kernel 完全匹配。
