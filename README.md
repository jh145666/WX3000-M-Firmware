# WX3000-M-Firmware

中国移动 GS2-A（WX3000-M）路由器固件全量备份（2026-10-24 备份，已脱敏）。

- 产品型号：`GS2-A`（bootenv 内 productname）
- 主控：MediaTek MT7981（BL2 头 `SPINAND!`）
- 存储：SPI NAND 128MB（NMBM 布局）
- 系统：ARM64 OpenWrt（CMCC 定制，Linux 5.4.231）

## 备份文件

压缩包：`WX3000-M_full_flash_20261024.zip`（解压后共 9 个文件）

| 文件 | 大小 | 实测内容 |
|---|---|---|
| full_flash.bin | 128MB | SPI NAND 全量备份（含 NAND 空闲区 gap 0xFF 填充修复） |
| bl2.bin | 1MB | MT7981 BL2 启动加载器（Built 22:22:50 2024-05-22） |
| bootenv.bin | 512KB | U-Boot 环境变量区（明文，双副本） |
| factory.bin | 2MB | 工厂分区（仅前 4KB 有效，含厂商标识 0x000C4326） |
| fip.bin | 2MB | FIP（magic `0xAA640001`）：BL31 ATF + BL33 U-Boot.armv8 |
| exp.bin | 1MB | 预留分区，当前全 0xFF（未使用） |
| firmware.bin | 64MB | 固件主分区：OpenWrt FIT 镜像（Linux 5.4.231） |
| nmbm_meta.bin | 8MB | NMBM 元数据（头 `NMM1`） |

## 固件内容分析（实测）

### bootenv 环境变量（关键项）

```
baudrate=115200
bootargs=console=ttyS0,115200n1 rootfstype=squashfs loglevel=3 root=/dev/mtdblock8
bootversion=102
productname=GS2-A
ipaddr=192.168.1.1    serverip=192.168.1.2    netmask=255.255.255.0
mtdparts=nmbm0:1024k(bl2),512k(Boot-env),2048k(factory),2048k(fip),1024k(exp),65536k(firmware)
bootmenu_0=Startup system (Default)=mtkboardboot
bootmenu_1=Upgrade firmware=mtkupgrade fw
bootmenu_2=Upgrade ATF BL2=mtkupgrade bl2
bootmenu_3=Upgrade ATF FIP=mtkupgrade fip
bootmenu_4=Upgrade single image=mtkupgrade simg
bootmenu_5=Load image=mtkload
```

### firmware.bin

- 0x00 起 64 字节为私有头（魔数 `RGS_+kkf`），其后为 FIT 镜像
- FIT 标识：`ARM64 OpenWrt FIT (Flattened Image Tree)` / `ARM64 OpenWrt Linux-5.4.231`
- DTB：`mt7981-spim-nand-rfb` 参考板 + CMCC 定制节点（mediatek,mt7981-* 系列）
- rootfs：squashfs（约 3.5MB 处），另含 UBI 卷（约 52.4MB 处）
- 含 `Env=Linux-Openwrt`、`CMCC` 定制字符串 —— 为原厂 CMCC OpenWrt 固件

### fip.bin / bl2.bin

- FIP TOC magic `0xAA640001`，version `0x12345678`
- 组件：BL31（mediatek ATF，`Hello BL31!!!` / `mediatek_sip_svc`）+ BL33（`U-Boot.armv8`）
- 编译时间：BL2 2024-05-22 22:22:50 / FIP 2024-05-22 22:22:59（同批原厂镜像）

## NAND 分区布局（bootenv mtdparts 实测）

```
0x0000000  bl2        (1MB)   1024k
0x0100000  Boot-env   (512KB) 512k
0x0180000  factory    (2MB)   2048k
0x0380000  fip        (2MB)   2048k
0x0580000  exp        (1MB)   1024k
0x0680000  firmware   (64MB)  65536k
0x7800000  nmbm_meta  (8MB)   （NMBM 元数据区）
```

## 脱敏处理说明（2026-10-24）

备份已做脱敏：bootenv 双副本中 ethaddr/manucode 已清零（环境 CRC 已按 U-Boot 算法同步重算）、factory 中疑似 MAC 数据已清零、full_flash 对应区域已同步更新。firmware.bin 内部的 manucode（厂商批次码，非设备唯一标识）因修改会破坏镜像校验，保留未动。刷写回设备后需自行重新设置 MAC 等参数。

## 校验值（2026-10-24 脱敏版）

### 压缩包

```
WX3000-M_full_flash_20261024.zip
SHA256: ee069039d0cab6c48e206eea662245eb7b7248135e2f47d131a399c464decea8
```

### 解压后各文件

```
full_flash.bin  sha256: dd9ed6d48944d40e02031c92bc5fa4e1366c0093ed956f91cd85fa786cd6cd0e
bl2.bin         sha256: 171a4b46bdb2f3852da4bdfa8cddbc8620ae9a45d7c07d2be52404d26f09ef4d
bootenv.bin     sha256: 41ee1875bcd722513892451fc7e0a86478e46f377b65f819a2344a212a981ec8
factory.bin     sha256: 5d40990181a7f3a8b990c0f258d3a93f630b4989f7a7d3b750833405311de776
fip.bin         sha256: 65597fa27d7cf35ff8cd23525ef7177af8de86abd1aec1d22787da2d0011703e
exp.bin         sha256: f5fb04aa5b882706b9309e885f19477261336ef76a150c3b4d3489dfac3953ec
firmware.bin    sha256: 0b4785ec40a808229b0508768dd876a8cd8a6fac0ed9ee54610dca1d99e986b7
nmbm_meta.bin   sha256: f9b67baf465204c3cbfe8b27cd375e8182f6c1611c06c697c17a5d727defb9bf
```

## 刷写提示

- 刷写前务必先备份原厂固件，并核对各分区 CRC32/SHA256
- full_flash.bin 为整片 NAND 备份，整片写入可还原到备份时状态（含已修复的 0xFF 空洞）
- 单独刷写某分区请严格使用上表偏移（与 mtdparts 一致），切勿错位写入
- factory（MAC/射频校准）分区请特别谨慎，误写可能导致变砖或信号异常
- 本仓库固件为脱敏版本（MAC/manucode 已清零），请勿将原始未脱敏固件公开传播
