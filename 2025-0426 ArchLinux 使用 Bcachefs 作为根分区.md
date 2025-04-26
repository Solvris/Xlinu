# Arch Linux 使用 Bcachefs 作为根分区（体验版教程）

> ⚡ **注意**：Bcachefs 当前仍处于**积极开发阶段**，功能可能**不稳定**，**不建议在生产环境中使用**。本教程以**虚拟机环境**为例进行演示体验。

---

## 目录

- [1. 前期准备](#1-前期准备)
- [2. 磁盘分区](#2-磁盘分区)
- [3. 格式化磁盘](#3-格式化磁盘)
- [4. 挂载文件系统](#4-挂载文件系统)
- [5. 安装基本系统](#5-安装基本系统)
- [6. 配置新系统](#6-配置新系统)
- [7. 生成 UKI 镜像](#7-生成-uki-镜像)
- [8. 配置引导](#8-配置引导)
  - [8.1 手动使用 efibootmgr](#81-手动使用-efibootmgr)
  - [8.2 推荐使用 systemd-boot](#82-推荐使用-systemd-boot)
- [9. 小结与注意事项](#9-小结与注意事项)
- [10. 参考资料](#9-参考资料)

---

## 1. 前期准备

请参阅官方 [Arch Wiki - Installation Guide](https://wiki.archlinux.org/title/Installation_guide) 进行基本环境准备，包括：

- 网络连接
- 磁盘清理
- UEFI 启动模式确认

确保加载了 `bcachefs` 模块，可以通过：

```bash
modprobe bcachefs
```

如果未找到模块，请提前安装内核模块或使用支持 bcachefs 的 ISO。

---

## 2. 磁盘分区

假设目标磁盘为 `/dev/sda`，将其分为：

| 分区 | 挂载点 | 文件系统 | 说明 |
|:---|:---|:---|:---|
| /dev/sda1 | /efi | FAT32 | EFI 系统分区，空间建议 512MB-1GB |
| /dev/sda2 | swap | swap | 交换分区 |
| /dev/sda3 | / (根分区) | Bcachefs | 根分区 |

> 📢 说明：
> - 本教程使用统一内核镜像（UKI）方式引导，因此 EFI 分区需要足够空间容纳内核和 initramfs。
> - 由于 bcachefs 在 GRUB 支持有限，**本教程不使用 GRUB**。
> - 也可以选择添加一个 ext4 的 `/boot` 分区（不是必须）。

关于 `/boot` 单独分区：
> 在一些复杂文件系统（如 LVM、Btrfs、Bcachefs、ZFS）中，常因引导器不完全支持而将 `/boot` 单独分区，使用简单文件系统（ext4, xfs等）存储引导文件。

---

## 3. 格式化磁盘

执行以下命令格式化各分区：

```bash
mkfs.vfat -F 32 -n EFI /dev/sda1
mkswap -L SWAP /dev/sda2
mkfs.bcachefs -L ROOT /dev/sda3
```

---

## 4. 挂载文件系统

挂载分区到 `/mnt` 准备安装系统：

```bash
mount /dev/sda3 /mnt
mkdir /mnt/efi
mount /dev/sda1 /mnt/efi
swapon /dev/sda2
```

---

## 5. 安装基本系统

使用 `pacstrap` 安装基本系统：

```bash
pacstrap -K /mnt base linux linux-firmware bcachefs-tools mkinitcpio
```

---

## 6. 配置新系统

生成文件系统表 `/etc/fstab`：

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

检查 `/etc/fstab`，特别是 `/efi` 挂载参数，应包含 `fmask=0077,dmask=0077`。

```bash
# 示例 fstab 行
UUID=xxxx-xxxx  /efi  vfat  defaults,fmask=0077,dmask=0077  0  1
```


进入新系统环境：

```bash
arch-chroot /mnt
```

配置主机名、locale、root密码等基本设置（参照 ArchWiki）。

---

## 7. 生成 UKI 镜像

### 配置启动参数

在 `/etc/kernel/cmdline` 中写入启动参数：

```bash
root=UUID=<bcachefs分区UUID> rootfstype=bcachefs rw
```

创建 `/etc/kernel/cmdline_fallback`：

```bash
root=UUID=<bcachefs分区UUID> rootfstype=bcachefs rw
```

查找 UUID：

```bash
blkid /dev/sda3
```

---

### 配置 mkinitcpio

编辑 `/etc/mkinitcpio.conf`：

```bash
MODULES=(bcachefs)
HOOKS=(base udev autodetect modconf block filesystems keyboard fsck bcachefs)
```

创建 `/etc/kernel/uki.conf`：

```ini
[UKI]
Splash=/usr/share/systemd/bootctl/splash-arch.bmp
```

创建 `/etc/kernel/install.conf`：

```bash
layout=uki
uki_generator=mkinitcpio
```

编辑 `/etc/mkinitcpio.d/linux.preset`：

```bash
# mkinitcpio preset 文件

ALL_kver="/boot/vmlinuz-linux"
ALL_config="/etc/mkinitcpio.conf"
PRESETS=('default' 'fallback')

default_uki="/efi/EFI/Linux/archlinux.efi"
default_options="--splash /usr/share/systemd/bootctl/splash-arch.bmp"

fallback_uki="/efi/EFI/Linux/archlinux-fallback.efi"
fallback_options="-S autodetect --cmdline /etc/kernel/cmdline_fallback"
```

生成 UKI 镜像：

```bash
mkinitcpio -P
```

删除传统 initramfs 文件（可选清理步骤）：

```bash
rm /boot/initramfs-*.img
```

查看生成的 UKI：

```bash
ls /efi/EFI/Linux/
# 应该看到 archlinux.efi 和 archlinux-fallback.efi
```

---

## 8. 配置引导

生成 UKI 镜像后，可以直接配置引导。

### 8.1 手动使用 efibootmgr

使用 `efibootmgr` 添加启动项：

```bash
efibootmgr --create \
  --disk /dev/sda --part 1 \
  --label "Arch Linux (UKI)" \
  --loader '\EFI\Linux\archlinux.efi'
```

调整启动顺序（可选）：

```bash
efibootmgr --bootorder 0003,0001,0000
```

完成后，即可重启进入系统。

---

### 8.2 推荐使用 systemd-boot

为了方便维护，推荐使用 `systemd-boot`：

安装 systemd-boot 到 EFI 分区：

```bash
bootctl install
```

列出当前条目：

```bash
bootctl list
```

应该可以看到自动识别到的 `archlinux.efi` 和 `archlinux-fallback.efi`。

**可选**：自定义配置文件：

- `/efi/loader/loader.conf`
- `/efi/loader/entries/arch.conf`

---

## 9. 小结与注意事项

- 本教程采用 **Bcachefs 根分区 + UKI 引导**。
- **GRUB** 目前**不适合直接引导 bcachefs 根分区**，即使强行安装到 `/efi`，也是曲线救国，建议分离 `/boot`。
- **rEFInd** 未测试。
- **efibootmgr** 和 **systemd-boot** 均测试通过。
- 生产环境请谨慎使用 Bcachefs，留意其开发进度和稳定性更新。

---

明白了！你要的是**完整参考资料部分**，整合你补充的内容，标准清晰地列出来。

这里是最终版参考资料部分：

---

好的，这里是**编号版参考资料**格式，适合发布到 GitHub 或文档里：

---

## 10. 参考资料

[1] [Bcachefs - 官方网站](https://bcachefs.org/)

[2] [Arch Wiki - Installation Guide](https://wiki.archlinux.org/title/Installation_guide)

[3] [Arch Wiki - Bcachefs](https://wiki.archlinux.org/title/Bcachefs)

[4] [Arch Wiki - Unified Kernel Image (UKI)](https://wiki.archlinux.org/title/Unified_kernel_image)

[5] [Arch Wiki - systemd-boot](https://wiki.archlinux.org/title/Systemd-boot)

[6] [cascade.moe - 使用 UKI 启动 Linux 教程](https://cascade.moe/posts/uki-linux-boot/)

[7] [Arch Wiki - User:Bai-Chiang: Arch Linux installation with Bcachefs, unified kernel image (UKI), secure boot, and common setups](https://wiki.archlinux.org/title/User:Bai-Chiang/Arch_Linux_installation_with_Bcachefs,_unified_kernel_image_(UKI),_secure_boot,_and_common_setups)

---

如果你还想要更进一步，比如在正文中引用比如 "(参考 [2])" 这种，我也可以帮你加！要继续吗？🚀

---


# 🎯 结束

重启系统：

```bash
exit
umount -R /mnt
reboot
```

如果一切顺利，你现在应该已经用 bcachefs 成功启动了一个 Arch Linux 系统！

---
