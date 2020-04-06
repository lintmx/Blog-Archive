---
title: "Arch Linux 的安装与配置"
date: 2016-02-11T14:00:00+08:00
tags: ["Arch", "Linux", "双系统"]
draft: false
---

利用放假在家不被锐捷的~~喂屎~~限制的时间，好好的折腾了一下 Arch Linux ，不得不说它足够简洁，优雅，并且定制性高，对喜欢折腾的人来说简直赛高！不过缺点也明显，对新手极度不友好，安装时没有提供图形界面，就算是老手也难免需要查 [Wiki](https://wiki.archlinux.org/) ，所以记录一下安装配置过程方便以后重装。

# 安装准备

## 连接互联网~~局域网~~

安装程序会自动启动 `dhcpcd` ，有线连接可以直接使用，无线网络可以使用 `wifi-menu` 连接，然后用 Ping 命令检查连接是否成功。

```bash
$ ping -c 4 www.lintmx.com
```

Ps: 锐捷用户应事先准备 `mentohust` 。

<!--more-->

## 更新系统时间

启动 `systemd-timesyncd` 服务用于同步时间。

```bash
# timedatectl set-ntp true
```

查看服务状态：

```bash
# timedatectl status
```

## 磁盘分区

我有两块硬盘，其中一块装了 Windows 10 有 `EFI System` 分区，所以直接将另一个硬盘整个作为一个分区。

```
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0 238.5G  0 disk
├─sda1   8:1    0   450M  0 part
├─sda2   8:2    0   100M  0 part /boot/efi
├─sda3   8:3    0    16M  0 part
└─sda4   8:4    0 237.9G  0 part /run/media/lintmx/Windows
sdb      8:16   0 119.2G  0 disk
└─sdb1   8:17   0 119.2G  0 part /
```

## 格式化分区

直接格式化为 `ext4` 分区。

```bash
# mkfs.ext4 /dev/sdb1
```

## 挂载分区

将分区挂载到 `/mnt` 目录， Windows 的 EFI 分区挂载到 `/mnt/boot/efi` 目录。

```bash
# mount /dev/sdb1 /mnt
# mkdir -p /mnt/boot/efi
# mount /dev/sda2 /mnt/boot/efi
```

# 安装

## 选择镜像

编辑 `/etc/pacman.d/mirrorlist` 文件，将国内源复制到前面。

## 安装基本系统

安装基本系统和自己需要的一些软件。

```bash
# pacstrap /mnt base base-devel
```

## 配置基本系统

生成 `fstab` 文件：

```bash
# genfstab -U -p /mnt >> /mnt/etc/fstab
```

Change root 到系统：

```bash
# arch-chroot /mnt
```

设置主机名：

```bash
# echo computer_name >> /etc/hostname
```

设置时区：

```bash
# ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

生成 locale 信息：

编辑 `/etc/locale.gen` 文件，反注释需要的语言，然后使用 `locale-gen` 生成 locale 信息。

设置系统 locale 偏好：

```bash
# echo LANG=en_US.UTF-8 >> /etc/locale.conf
```

创建初始化内存盘：

```bash
# mkinticpio -p Linux
```

设置 root 密码：

```bash
# passwd
```

安装并配置引导程序：

```bash
# pacman -S grub efibootmgr
# grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id="Arch Linux" --recheck
# grub-mkconfig -o /boot/grub/grub.cfg
```

开启 DHCP 服务：

```bash
# systemctl enable dhcpcd.service
```

Ps: wifi 用户需要安装 `wpa_supplicant dialog iw` 包，锐捷用户需要使用 `yaourt` 安装 `mentohust` 包以保证重启后能够正常联网。

## 重启

现在一个基本的 Arch Linux 系统已经安装好了，现在只需要退出 chroot ，卸载分区然后重启就行了。

```bash
$ exit
# umount -R /mnt
# reboot
```

# 配置系统

## 新建用户和密码

一直使用 `root` 用户进行操作绝对是在作死，所以需要新建一个用户用于日常使用。

```bash
# useradd -m -G wheel -s /bin/bash lintmx
# passwd
```

使用 `visudo` 命令反注释 `%wheel ALL=(ALL) ALL` 语句让用户可以使用 `sudo` 。

## 安装 X 窗口管理

```bash
# pacman -S xorg-server xorg-utils xorg-server-utils
```

## 安装相关驱动

```bash
# pacman -S xf86-video-intel(Intel 显卡) xf86-input-synaptics(笔记本触摸板)
```

其余显卡用户请自行查阅 [Wiki](https://wiki.archlinux.org/) 。

## 安装桌面环境

个人还是比较喜欢 Gnome 的，所以就不折腾 xfce4 了。

```bash
# pacman -S gnome
```

启用 GDM：

```bash
# systemctl enable gdm.service
```

## 安装 `yaourt`

编辑 `/etc/pacman.conf` ,在文件最后添加：

```
[archlinuxcn]
SigLevel = Optional TrustAll
Server = http://repo.archlinuxcn.org/$arch
```

然后：

```bash
# pacman -S yaourt
```

## 自动挂载 Windows 分区

虽然 Gnome 的文件管理器里可以访问 Windows 分区，但它必须在你访问过一次后才算挂载，所以有些时候显得有些麻烦，所以使用 `fstab` 文件实现自动挂载。

安装 `ntfs-3g` 驱动：

```bash
# pacman -S ntfs-3g
```

编辑 `/etc/fstab` 文件：

UUID 可以使用 `lsblk -f` 查询。

```
UUID=<NTFS UUID> /mnt/windows ntfs-3g uid=USERNAME,gid=wheel 0 0
```

# 小结

Arch Linux 的安装虽然有些麻烦，但也有一些诀窍：

- [Arch Wiki](https://wiki.archlinux.org/)
- [Google](https://www.google.com/)
- [Arch Wiki](https://wiki.archlinux.org/)
- [Google](https://www.google.com/)
- [Arch Wiki](https://wiki.archlinux.org/)
- [Google](https://www.google.com/)
- [Arch Wiki](https://wiki.archlinux.org/)
- [Google](https://www.google.com/)

只要你用好这些，相信你一定可以熟悉的掌握 Arch Linux 的。

以及

![Fuck Nvidia](https://i.loli.net/2018/06/16/5b252deece078.jpg)