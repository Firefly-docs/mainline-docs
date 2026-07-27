# 主线 U-Boot 使用指南

## 1. 概述

本文介绍在 Rockchip平台上使用主线 U-Boot 的基本流程，包括源码准备、选择已有板级配置、编译、产物和烧录。

Rockchip 主线 U-Boot 构建通常同时使用两个仓库：

| 仓库 | 作用 |
| --- | --- |
| U-Boot | U-Boot 源码、板级 defconfig 和设备树 |
| rkbin | Rockchip DDR 初始化代码、BL31 等二进制固件 |

## 2. 准备源码

### 2.1下载源码
建议将 U-Boot 和 rkbin 放在同一个工作目录下：

```bash
mkdir -p rockchip-mainline
cd rockchip-mainline

git clone https://github.com/u-boot/u-boot.git
git clone https://github.com/rockchip-linux/rkbin.git
```

目录结构：

```text
rockchip-mainline/
├── u-boot/
└── rkbin/
```

产品开发应固定到已经验证的 tag 或 commit，不建议在量产构建中直接跟随持续变化的默认分支。

查看当前版本：

```bash
git -C u-boot describe --always --dirty --tags
git -C rkbin describe --always --dirty --tags
```

### 2.2添加版型

添加版型需要添加对应版型的defconfig和设备树。

- 在configs/下创建对应版型配置，可基于configs/evb-rk3588_defconfig配置文件根据版型增加和删除对应功能配置项生成新的配置文件roc-pc-rk3588s_defconfig
- 2024.05主线开启了OF_UPSTREAM特性，大量设备树迁移到了dts/upstream/src/arm64/路径下，不需要在arch/arm/dts/Makefile中添加设备树了，更改为通过deconfig的CONFIG_DEFAULT_DEVICE_TREE设置默认的设备树。dts/upstream/src/arm64/下的设备树和内核使用的设备树相同，只需将内核设备树拷贝进来使用即可。

## 3. 编译

### 3.1 工具链和 Python

安装 AArch64 交叉编译工具链和基础依赖。Debian/Ubuntu 示例：

```bash
sudo apt update
sudo apt install \
  build-essential bc bison flex swig python3 python3-dev \
  device-tree-compiler libssl-dev libgnutls28-dev \
  gcc-aarch64-linux-gnu
```

### 3.2 指定 TPL 和 BL31

进入 U-Boot 源码目录：

```bash
cd rockchip-mainline/u-boot
```

以 PDF 中的 RK3588 固件版本为例：

```bash
export ROCKCHIP_TPL=../rkbin/bin/rk35/rk3588_ddr_lp4_2112MHz_lp5_2400MHz_v1.18.bin
export BL31=../rkbin/bin/rk35/rk3588_bl31_v1.48.elf
```

`ROCKCHIP_TPL` 是 DDR 初始化固件，必须与 SoC、DDR 类型和板级设计匹配。`BL31` 是 ARM Trusted Firmware-A 阶段固件。rkbin 的文件名会随版本更新，如果示例文件不存在，应在当前 `rkbin/bin/rk35/` 中选择与硬件匹配的版本：

```bash
find ../rkbin/bin/rk35 -maxdepth 1 \
  \( -name 'rk3588_ddr_*.bin' -o -name 'rk3588_bl31_*.elf' \) \
  -print
```

### 3.3 执行编译

先查看主线 U-Boot 已有的 RK3588/RK3588S defconfig：

```bash
find configs -maxdepth 1 -name '*rk3588*_defconfig' -printf '%f\n' | sort
```

选择与目标板卡匹配的已有 defconfig。以主线 EVB 配置为例：

```bash
export DEFCONFIG=evb-rk3588_defconfig
```

清理旧配置，加载 defconfig，然后编译：

```bash
make mrproper
make "$DEFCONFIG"
make CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
```

也可将架构和工具链导出到环境变量：

```bash
export ARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-

make "$DEFCONFIG"
make -j$(nproc)
```

编译后保存版本和配置：

```bash
git describe --always --dirty --tags
grep -E 'CONFIG_DEFAULT_DEVICE_TREE|CONFIG_OF_UPSTREAM' .config
```

## 4. TEE 警告

未指定 BL32/OP-TEE 时，编译末尾可能出现：

```text
Image 'simple-bin' is missing optional external blobs but is still functional: tee-os
```

`tee-os` 在该配置中是可选外部固件时，上述警告不会阻止 U-Boot 产物生成。

## 5. 编译产物

| 文件 | 用途 |
| --- | --- |
| `idbloader.img` | Rockchip 前级 Loader，烧录到 `idbloader` 分区 |
| `u-boot.itb` | U-Boot FIT 固件，烧录到 `uboot` 分区 |
| `u-boot-rockchip.bin` | 包含 `idbloader.img` 和 `u-boot.itb` 的指定偏移布局，可直接写入 SD 卡或 eMMC |
| `u-boot-rockchip-spi.bin` | 用于 SPI NOR Flash；只有开启对应配置时才会生成 |

检查产物：

```bash
ls -lh idbloader.img u-boot.itb u-boot-rockchip*.bin
```

如果产物缺失，先检查编译日志、TPL/BL31 路径、defconfig 和目标存储介质配置。

## 6. 烧录

### 6.1 SD 卡或 eMMC

`u-boot-rockchip.bin` 已包含子固件的正确偏移。PDF 中的写入位置从存储设备的第 64 个 512 Byte 扇区开始，即 32 KiB 偏移：

```bash
sudo dd \
  if=u-boot-rockchip.bin \
  of=/dev/mmcblk0 \
  bs=32K \
  seek=1 \
  conv=fsync,notrunc \
  status=progress
```

> 危险：`dd` 会直接覆盖块设备。`of` 必须是目标 SD/eMMC 整个设备，不是文件系统分区，也不得是开发主机的系统盘。

写入前建议：

```bash
lsblk -o NAME,SIZE,MODEL,TYPE,MOUNTPOINTS
```

卸载目标设备上已挂载的文件系统，再执行 `dd`。写入完成后执行：

```bash
sync
```

### 6.2 SPI NOR Flash

开启 SPI 启动配置并生成 `u-boot-rockchip-spi.bin` 后，可用 `flashcp` 写入 MTD 设备：

```bash
sudo flashcp -v -p u-boot-rockchip-spi.bin /dev/mtd0
```

写入前先确认 MTD 设备、容量和现有分区：

```bash
cat /proc/mtd
mtdinfo /dev/mtd0
```

修改 SPI 启动固件可能导致设备无法启动。操作前应备份原始固件，并准备 Maskrom/Loader 恢复方法。
