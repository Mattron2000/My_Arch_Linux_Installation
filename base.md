# My Arch Linux Base Installation <!-- omit in toc -->

## Summary <!-- omit in toc -->

- [Pre-installation](#pre-installation)
  - [Set the console keyboard layout and font](#set-the-console-keyboard-layout-and-font)
  - [Verify the boot mode](#verify-the-boot-mode)
  - [Connect to the internet](#connect-to-the-internet)
    - [Authenticate to the wireless network](#authenticate-to-the-wireless-network)
  - [\[OPTIONAL\] ssh connection](#optional-ssh-connection)
  - [Update the system clock](#update-the-system-clock)
  - [Identify the disks](#identify-the-disks)
  - [\[OPTIONAL\] Secure wipe and sanitize nvme](#optional-secure-wipe-and-sanitize-nvme)
  - [Partition the disks](#partition-the-disks)
    - [Encrypt root partition](#encrypt-root-partition)
    - [Preparing the logical volumes](#preparing-the-logical-volumes)
  - [Format the partitions](#format-the-partitions)
  - [Mount the file systems](#mount-the-file-systems)
- [Installation](#installation)
  - [Select the mirrors](#select-the-mirrors)
  - [Install essential packages](#install-essential-packages)
  - [Fstab](#fstab)
  - [Chroot](#chroot)
    - [Swap performance](#swap-performance)
  - [Time zone](#time-zone)
  - [Localization](#localization)
  - [Network configuration](#network-configuration)
  - [Setting Unified Kernel Image](#setting-unified-kernel-image)
  - [Setting mkinitcpio](#setting-mkinitcpio)
  - [Kernel cmdline](#kernel-cmdline)
  - [Initramfs](#initramfs)
  - [Boot loader](#boot-loader)
    - [Installing the EFI boot manager](#installing-the-efi-boot-manager)
    - [Configuration](#configuration)
      - [Loader configuration](#loader-configuration)
  - [Root password](#root-password)
  - [Manage users](#manage-users)
  - [Configure iwd service](#configure-iwd-service)
  - [Exit, umount and shutdown](#exit-umount-and-shutdown)

## Pre-installation

### Set the console keyboard layout and font

```zsh
localectl list-keymaps # layouts can be listed with
# KEYMAP=us
loadkeys $KEYMAP # to set the keyboard layout

ls /usr/share/kbd/consolefonts/ # all fonts are located here

# CONSOLEFONT=ter-120b
setfont $CONSOLEFONT
# or
setfont -d # set font with default value
```

### Verify the boot mode

```zsh
cat /sys/firmware/efi/fw_platform_size # To verify the boot mode
# '64' a 64-bit system is booted in x64 UEFI mode
# '32' a 32-bit system is booted in IA32 UEFI mode
```

### Connect to the internet

```zsh
ping archlinux.org # The connection may be verified with 
```

If it's works, then go to [ssh connection](#optional-ssh-connection) or [Update the system clock](#update-the-system-clock) chapter else connect Ethernet cable and retry or connect internet by Wi-Fi following the [next chapter](#authenticate-to-the-wireless-network).

#### Authenticate to the wireless network

```zsh
iwctl # get an interactive prompt
device list # list all possible Wi-Fi devices for $WIFI_DEVICE
station $WIFI_DEVICE scan # initiate a scan for networks
station $WIFI_DEVICE get-networks # list all available networks ($NETWORK_NAME)
station $WIFI_DEVICE connect $NETWORK_NAME
exit
```

### [OPTIONAL] ssh connection

I want remotely connect to the PC from another because you can easily copy-paste the commands

```zsh
passwd # a password like '1234' it's ok

ip addr # see which IP address has $IP_ADDR
```

Now from another pc...

```zsh
ssh root@$IP_ADDR
```

### Update the system clock

```zsh
timedatectl set-ntp true # ensure the system clock is accurate
```

### Identify the disks

```zsh
# to identify disk devices ($DISK)
fdisk -l
lsblk
```

### [OPTIONAL] Secure wipe and sanitize nvme

```zsh
# DISK=nvme0

# check that nvme has format|sanitize support
nvme id-ctrl /dev/$DISK -H | grep -E 'Format|Crypto Erase|Sanitize'

# secure wipe
nvme format /dev/$DISK -s 2 -n 0xffffffff

# cryptographic sanitize nvme disk
nvme sanitize /dev/$DISK -a start-crypto-erase

## now the disc is practically as good as new as if it had just come out of the factory or sell the disk without privacy concern
```

### Partition the disks

```zsh
# open disk utility to modify the disk table of the selected disk
# DISK=${DISK}n1
fdisk /dev/$DISK
g # make new GPT partition table
n # new partition
p # print to see the situation
t # change partition type
w # write changes and exit
```

> In this point I've 2 partitions:
>
| Partition number | Size                          | Type                     |
| ---------------- | ----------------------------- | ------------------------ |
| 1                | 1G                            | [1]  EFI System          |
| 2                | 100% of free sector remaining | [23] Linux root (x86_64) |

#### Encrypt root partition

```zsh
# ROOT_PARTITION=${DISK}p2
cryptsetup luksFormat -v /dev/$ROOT_PARTITION

cryptsetup luksDump /dev/$ROOT_PARTITION

# LUKS_NAME=cryptroot
cryptsetup open --perf-no_read_workqueue --perf-no_write_workqueue --allow-discards --persistent -v /dev/$ROOT_PARTITION $LUKS_NAME

# check that $DISK (ssd or nvme) have TRIM SUPPORT
lsblk --discard # https://wiki.archlinux.org/title/Solid_state_drive#TRIM

cryptsetup luksDump /dev/$ROOT_PARTITION | grep Flags
```

#### Preparing the logical volumes

```zsh
pvcreate /dev/mapper/$LUKS_NAME
# LVM_VG=vg
vgcreate $LVM_VG /dev/mapper/$LUKS_NAME

# LVM_SWAP=swap
lvcreate $LVM_VG -L 4G -n $LVM_SWAP
# LVM_ROOT=root
lvcreate $LVM_VG -L 80G -n $LVM_ROOT
# LVM_HOME=home
lvcreate $LVM_VG -l 100%FREE -n $LVM_HOME

lvreduce -L -256M $LVM_VG/$LVM_HOME # to allow to use https://man.archlinux.org/man/e2scrub.8
```

### Format the partitions

```zsh
# EFI_PARTITION=${DISK}p1
mkfs.fat -F 32 /dev/$EFI_PARTITION

mkfs.ext4 /dev/$LVM_VG/$LVM_ROOT
mkfs.ext4 /dev/$LVM_VG/$LVM_HOME

mkswap /dev/$LVM_VG/$LVM_SWAP
```

### Mount the file systems

```zsh
mount /dev/$LVM_VG/$LVM_ROOT /mnt
mount -m /dev/$LVM_VG/$LVM_HOME /mnt/home
# ESP=efi
mount -m /dev/$EFI_PARTITION /mnt/$ESP # that's because systemd-boot recommended this (https://man.archlinux.org/man/bootctl.1#OPTIONS)
swapon /dev/$LVM_VG/$LVM_SWAP
```

check that all volumes/partitions is mounted

```zsh
lsblk
```

## Installation

### Select the mirrors

```zsh
# select the 10 most recently synchronized HTTPS mirrors, sort them by download speed, and overwrite the file /etc/pacman.d/mirrorlist
reflector -l 10 --sort rate -p https --save /etc/pacman.d/mirrorlist
```

### Install essential packages

```zsh
# KERNEL=(linux | linux-lts)
# EDITOR=(nano | vim | neovim)
pacstrap -K /mnt base $KERNEL linux-firmware intel-ucode lvm2 sof-firmware $EDITOR man-db sudo iwd terminus-font
```

### Fstab

```zsh
genfstab -U /mnt >> /mnt/etc/fstab
```

### Chroot

```zsh
arch-chroot /mnt
```

#### Swap performance

```zsh
sysctl vm.swappiness # gives 60
```

open `/etc/sysctl.d/99-swappiness.conf` and write...

```zsh
echo "vm.swappiness = 10" > /etc/sysctl.d/99-swappiness.conf
```

### Time zone

```zsh
ln -sf /usr/share/zoneinfo/$REGION/$CITY /etc/localtime
hwclock --systohc
```

### Localization

Edit `/etc/locale.gen` uncommenting wanted locales.

```zsh
# EDITOR=(nano | vim | nvim)
$EDITOR /etc/locale.gen

locale-gen

locale # display the current locale
localectl list-locales # available locales

echo "LANG=$LOCALE" > /etc/locale.conf
# KEYMAP=us
echo "KEYMAP=$KEYMAP" > /etc/vconsole.conf
# CONSOLEFONT=ter-120b
echo "FONT=$CONSOLEFONT" >> /etc/vconsole.conf
```

### Network configuration

```zsh
echo $HOSTNAME > /etc/hostname
```

### Setting Unified Kernel Image

Modify this file...

```zsh
$EDITOR /etc/mkinitcpio.d/linux.preset
```

...to make like this

```zsh
# mkinitcpio preset file for the 'linux' package

#ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux"

PRESETS=('default' 'fallback')

#default_config="/etc/mkinitcpio.conf"
#default_image="/boot/initramfs-linux.img"
default_uki="esp/EFI/Linux/arch-linux.efi"
#default_options="--splash=/usr/share/systemd/bootctl/splash-arch.bmp"

#fallback_config="/etc/mkinitcpio.conf"
#fallback_image="/boot/initramfs-linux-fallback.img"
fallback_uki="esp/EFI/Linux/arch-linux-fallback.efi"
fallback_options="-S autodetect"
```

### Setting mkinitcpio

```zsh
$EDITOR /etc/mkinitcpio.conf
```

...to add some HOOKS:

```zsh
# ...
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)
# ...
```

### Kernel cmdline

```zsh
$EDITOR /etc/kernel/cmdline
```

write in this file...

```zsh
rd.luks.name="$UUID_ROOT_PARTITION"=$LUKS_NAME $LVM_ROOT=/dev/$LVM_VG/$LVM_ROOT rw quiet loglevel=3 systemd.show_status=auto rd.udev.log_level=3

# to insert the $UUID_ROOT_PARTITION, in Normal mode, write ":read ! blkid /dev/$ROOT_PARTITION" and take only the UUID value.
```

### Initramfs

```zsh
# ESP=efi
mkdir -p /$ESP/EFI/Linux # like here https://wiki.archlinux.org/title/Unified_kernel_image#Building_the_UKIs
# KERNEL=(linux | linux-lts)
sudo mkinitcpio -p $KERNEL
```

### Boot loader

#### Installing the EFI boot manager

```zsh
bootctl --esp-path=/$ESP install
```

#### Configuration

##### Loader configuration

set this file...

```zsh
$EDITOR /$ESP/loader/loader.conf
```

in this...

```zsh
default arch-$KERNEL.efi
timeout 0
console-mode auto
editor no
```

### Root password

```zsh
passwd # the real root password
```

### Manage users

```zsh
# USERNAME=johndoe
useradd -mG wheel $USERNAME
passwd $USERNAME

EDITOR=$EDITOR visudo # to uncomment '%wheel ALL=(ALL:ALL) ALL'
```

### Configure iwd service

write in file...

```zsh
mkdir -p /etc/iwd/

$EDITOR /etc/iwd/main.conf
```

... to this:

```zsh
[Scan]
DisablePeriodicScan=true

[General]
EnableNetworkConfiguration=true

[Network]
RoutePriorityOffset=300
EnableIPv6=true
NameResolvingService=systemd
```

```zsh
systemctl enable systemd-resolved iwd
```

### Exit, umount and shutdown

```zsh
exit
umount -R /mnt
swapoff /dev/$LVM_VG/$LVM_SWAP
cryptsetup close $LUKS_NAME
shutdown now
```
