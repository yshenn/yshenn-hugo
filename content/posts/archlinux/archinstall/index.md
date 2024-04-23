+++
title = '  Archlinux-Installation'
date = 2024-04-23T17:28:15+08:00
draft = false
+++

----

Archlinux Installation
本文是以记录为目的，可以作为一个参考，网上有很多很详细的教程，还有官方的ArchWiki。

----

## 安装前准备
**关于双系统**  
双系统的安装分两种：  
- 一种是将win 和 linux 安装在同一张硬盘上，这时候，需要提前在win中分出一个空闲的区域，为linux留出空间
- 另外一种是将win 和 linux安装在两张独立的硬盘上，此时和单系统安装一样，但安装后的系统引导设置不同

----

**下载iso**  
去 [archlinux官网](https://archlinux.org/download/) 或者一些开源镜像站([tsinghua](https://mirrors.tuna.tsinghua.edu.cn/archlinux/iso/))，下载最新iso文件  

----

**下载安装盘刻录软件**  
我使用的是 [ventoy](https://www.ventoy.net/cn/index.html)，开源。它和一般的刻录软件不同，一般软件是将安装引导系统刻录进整个U盘，这样系统安装完之后，U盘就不能直接使用了，需要进行格式化；而使用ventoy安装系统后，U盘还是能够正常使用，也不需要备份U盘内的文件；并且操作起来也很简单，只需要把ios文件直接拷贝至u盘即可，无需其他操作。

----

**主板BIOS设置**  
开始安装之前，需要对bios进行简单的设置  
- 关闭Secure Boot，archwiki指出`Arch Linux installation images do not support Secure Boot`
- 设置安装盘（也就是U盘）优先启动

每台电脑进入Bios的方式都不一样，具体可以根据电脑型号，参考官方说明书

----

## 安装
### 基本设置
设置好bios，插上U盘，就准备开始安装了。
当电脑屏幕跳出引导加载器菜单(boot loader menu), 也就是一个可以选择的黑框框，选择第一个Arch Linux install medium, 直接回车，就进入了archlinux安装环境

----

**设置合适的字体和键盘布局(optional)**  
安装环境下，这个console中的字体特别得小，看着很费劲，所以可以修改一下字体，选择一个更大的字体，系统所有可用字体都在`/use/share/kbd/consolefonts/`目录下，可以用`setfont`命令进行设置：
```bash
#以/usr/share/kbd/consolefonts/ 下的文件名(省略扩展名)，作为命令参数
setfont LatGrkCyr-12x22 #这是一个比较大的字体
```

同时你也可以更改键盘布局，一般不需要更改，默认是US布局，所有支持的键盘布局都在`/usr/share/kbd/keymaps/`下的每一个目录中，使用`loadkeys`命令进行修改：
```bash
# 同样，下面的参数就是键盘布局的名字，不加扩展名
loadkeys keyboardlayout
```

----

**确定电脑的启动模式(optional)**  
电脑启动在加载硬件驱动方面的程序，主要有早期的BIOS和现在的UEFI两种，两种模式的启动方式不一样，按照archwiki介绍，可以通过以下命令确认电脑是UEFI模式：
```bash
cat /sys/firmware/efi/fw_platform_size
# 返回64 或者 32，分别代表64bit 和 32bit的UEFI，若文件不存在，则有可能是BIOS
```

----

**连接网络**  

首先，检查网络设备是否打开：
```bash
# 该命令，列出无线网卡设备的名字（wlan0），以及其状态，如果其状态中<...,...,UP,...>有UP标志，标志网卡设备已经打开，否则为关闭
ip link

# 如果未打开，使用以下命令打开
ip link set wlan0 up # 这里网卡名为wlan0

# 如果以上命令出现了 Operation not possible due to RF-kill，则需要先将使用rfkill解锁网卡
# rfkill - tool for enabling and disabling wireless devices
rfkill # 列出无线设备及状态
rfkill unblock wlan  # unblock网卡
```

接着使用iwctl来设置无线网络，这是一个使用命令交互形式的网络设置程序
```bash
# 查看帮助信息
 help
# 查看当前网卡的连接状态
 station wlan0 show
# 扫描网络
 station wlan0 scan
# 查看可连接的网络
 station wlan0 get-networks
# 连接网络，名字为“P40 pro”
 station wlan0 connect "P40 pro"
```

此时可以检查网络是否连通：
```bash
ping baidu.com
```

如果无法ping通，极有可能是没有配置dhcp，使用：
```bash
dhcpcd &
```

----

**更新系统时钟**  
```bash
timedatectl set-ntp true # 将系统时间与网络时间进行同步
```
----


### 磁盘分区
使用fdisk进行磁盘分区

命令：  
- m：查看帮助
- p：打印当前分区表
- g：创建一个新的空的分区表，清理原来的分区
在使用w命令之前，新创建的分区不会被写入，原来的分区也不会被覆盖

- n：创建一个新的分区


主要有三个分区：
boot分区 + 主分区 + swap分区

我这里创建四个分区
第一个是系统引导分区，挂载在`/boot`目录下  
第二个是主分区，挂载在根目录`/`下  
第三个是用于存储用户文件数据分区，挂载在家目录`/home`下  
第四个是swap分区  

----

**格式化，创建文件系统**  
```bash
# /boot 分区应该为fat格式
mkfs.fat -F32  /dev/nvme0n1p1

# 根分区和家目录分区选择 ext4 文件系统
mkfs.ext4 /dev/nvme0n1p2
mkfs.ext4 /dev/nvme0n1p3

# swap分区
mkswap /dev/nvme0n1p4
```

----

**挂载分区**  
```bash
# swap挂载
swapon /dev/nvme0n1p4

# 主分区
mount /dev/nvme0n1p2 /mnt 

# boot分区
mkdir /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot

# home分区
mkdir /mnt/home
mount /dev/nvme0n1p3 /mnt/home

# or 
# omit mkdir
# mount --mkdir /dev/nvme0n1p2 /mnt/home

```

挂载完成后，可以说系统的安装已经进行一大半了

----

### 正式安装
**选择国内的软件源**  
在安装前要禁用reflector服务，该服务会自动更新pacman的mirrorlist，如果关闭晚了，也没关系，可以手动添加国内源  
```bash
# /etc/pacman.d/mirrorlist
# 将其中China下的镜像源提到最前面即可
```

----

**安装系统（Install essential packages）**  
安装base 、linux 和 linux-firmware
```bash
pacstrap -K /mnt base linux linux-firmware
```

----

**创建fstab文件**  
fstab 用来定义磁盘分区。它是 Linux 系统中重要的文件之一。使用 genfstab 自动根据当前挂载情况生成并写入 fstab 文件
```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

----

### 安装后，新系统中的配置

将当前安装环境切换到新系统环境下：
```bash
arch-chroot /mnt
```

----

**设置时区，同步时间**  

```bash
# 在 /etc/localtime 下用 /usr 中合适的时区创建符号链接
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

# 同步时间
hwclock --systohc
```

----

**本地化设置**  
编辑 `/etc/locale.gen` 并注释 `en_US.UTF-8 UTF-8，然后使用`local-gen`
```bash
locale-gen
```

在`/etc/locale.conf`中设置LANG变量
```bash
# /etc/locale.conf
LANG=en_US.UTF-8
```
----

**主机名和host配置**  
新建/etc/hostname 文件，向其中添加主机名  
并在/etc/hosts 文件中，添加配置
```bash
# /etc/hostname
glede

# /etc/hosts
127.0.0.1 localhost
::1	  localhost
127.0.0.1 glede.localdomain glede
```

----

**设置root密码**  
```bash
passwd
# SheN_2327
```

----

**安装一些软件**  
```bash
# intel微码
# intel-ucode

# 引导程序
# grub efibootmgr os-prober
pacman -S grub efibootmgr intel-ucode os-prober

```

----

**配置boot loader(grub)** 
```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=ARCH

# --efi-directory=/boot —— 将 grubx64.efi 安装到之前的指定位置（EFI 分区）
# --bootloader-id=ARCH —— 取名为 Arch

# 接下来使用 vim 编辑 /etc/default/grub 文件
# GRUB boot loader configuration

GRUB_DEFAULT=0
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="Arch"
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=5 nowatchdog"
GRUB_CMDLINE_LINUX=""
GRUB_DISABLE_OS_PROBER=false
...

# 生产grub所需配置文件
grub-mkconfig -o /boot/grub/grub.cfg
```

----

**安装自需软件**  
```bash
# 这里可以安装一些必需的软件
pacman -S iwd dhcpcd zsh vim
# 网络配置： iwd dhcpcd
# shell：zsh
# 编辑器：vim
```

----

**重启** 
```bash
exit  # 退回到安装引导系统中
reboot  # 重启，在重启的过程中，将U盘拔出
```


## 重启后的配置
重启后，系统可以说已经安装完成，但要达到可用的程度，还差很远，需要做很多工作，下载许多软件

最重要的莫过于安装桌面环境，可以选择个人喜欢的任何桌面环境 (Desktop Environment) 或者 窗口管理器 (Window Manager)  
- [GNOME](https://wiki.archlinux.org/title/GNOME) 
- [KDE](https://wiki.archlinux.org/title/KDE) 
- [Hyprland](https://hyprland.org/) 
- [Dwm](https://dwm.suckless.org/) 

我选择使用的是Hyprland，但由于我的电脑使用的是Nvidia显卡，安装hyprland环境的流程极其繁琐，本文不多作记录，可以参考hyprland官方wiki

下面记录一些安装后，所需的基本配置

----

**再次设置vconsole**  
上文已经讲述过设置vconsole字体和keymap，但那是在安装环境下，便于我们安装而设置的，现在进入新系统中，需要重新进行设置。  
vconsole的配置在`/etc/vconsole.conf` 文件中  
```bash
# 在该文件中添加要修改的字体，以及键盘设置
KEYMAP=us
FONT=LatGrkCyr-12x22
```

----

**设置双系统引导**  
如果是双系统，并且分别安装在两个盘中，那么现在开机时，你是无法进入win系统的，需要对boot loader(grub) 进行设置
```bash
# 先挂在windows的系统引导分区
mount /dev/nvme1n1p1 /mnt
ls /mnt 

# 使用os-prober 检测其他系统的引导
os-prober

# 重新配置grub文件
cp /boot/grub/grub.cfg /boot/grub/grub.cfg.bak
grub-mkconfig -o /boot/grub/grub.cfg
```

----

**配置网络**  
启用iwd
```bash
systemctl enable iwd.service
systemctl start iwd.service

# 这里网络的配置和上述一致，也有可能会遇到rfkill的问题，使用同样的方法即可
ip link
ip link set wlan0 up
rfkill
rfkill unblock wlan

# 然后就可以使用iwctl连接网络了
```

----

**创建非root用户**  
```bash
useradd -m -G wheel -s /bin/bash username
passwd username # 设置密码

# 如果没用下载vi，可以直接将vim链接到vi
ln -s /usr/bin/vim /usr/bin/vi

# 使用visudo 编辑sudo file，提升wheel用户组权限
#注释以下行
%wheel ALL=(ALL:ALL) ALL
```

----

**安装man 和sudo** 
```bash
pacman -S man-db sudo
```

----

**添加multilib && archlinuxcn 源**  
```bash
# 在pacman的配置文件中修改和添加
# /etc/pacman.conf 

# 开启multilib
[multilib]
Include = /etc/pacman.d/mirrorlist

# 开启 archlinuxcn
[archlinuxcn]
Server = https://mirrors.nju.edu.cn/archlinuxcn/$arch
# 这里可以参考官网，添加离你最近的镜像源

pacman -S archlinuxcn-keyring archlinuxcn-mirrorlist-git
# 将上面archlinuxcn下的Server一行改为
Include = /etc/pacman.d/archlinuxcn-mirrorlist
```

----

**关闭令人抓狂的pc扬声器，蜂鸣声**  
将 pcspkr 模块加入黑名单的方法可以阻止 udev 在启动时加载它
```bash
# /etc/modprobe.d/nobeep.conf
blacklist pcspkr
```

## Links
1.[archlinux installation](https://wiki.archlinux.org/title/installation_guide)
2.[thecw](https://www.bilibili.com/video/BV11J411a7Tp/?spm_id_from=333.999.0.0)
3.[archlinux简明指南](https://arch.icekylin.online/)

