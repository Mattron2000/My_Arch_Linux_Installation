# My Arch Linux Installation <!-- omit in toc -->

## Summary <!-- omit in toc -->
- [Pre-installation](#pre-installation)
  - [Set the console keyboard layout and font](#set-the-console-keyboard-layout-and-font)
  - [Verify the boot mode](#verify-the-boot-mode)
  - [Connect to the internet](#connect-to-the-internet)
    - [Authenticate to the wireless network](#authenticate-to-the-wireless-network)
  - [Update the system clock](#update-the-system-clock)
  - [Partition the disks](#partition-the-disks)
    - [Encrypt root partition](#encrypt-root-partition)
    - [Preparing the logical volumes](#preparing-the-logical-volumes)
  - [Format the partitions](#format-the-partitions)
  - [Mount the file systems](#mount-the-file-systems)
- [Installation](#installation)
  - [Select the mirrors](#select-the-mirrors)
  - [Install essential packages](#install-essential-packages)
- [Configure the system](#configure-the-system)
  - [Fstab](#fstab)
  - [Chroot](#chroot)
  - [Time zone](#time-zone)
  - [Localization](#localization)
  - [Network configuration](#network-configuration)
  - [Setting mkinitcpio](#setting-mkinitcpio)
  - [Initramfs](#initramfs)
  - [Boot loader](#boot-loader)
    - [Installing the EFI boot manager](#installing-the-efi-boot-manager)
    - [Configuration](#configuration)
      - [Loader configuration](#loader-configuration)
  - [Kernel cmdline](#kernel-cmdline)
  - [Root password](#root-password)
  - [Manage users](#manage-users)
  - [Configure iwd service](#configure-iwd-service)
  - [Reboot](#reboot)
- [Post-installation](#post-installation)
  - [Reconnect Wifi](#reconnect-wifi)
  - [Secure boot](#secure-boot)
    - [Create Secure boot keys](#create-secure-boot-keys)
    - [Sign unsigned files](#sign-unsigned-files)
  - [Automatic decrypt LUKS device with TPM](#automatic-decrypt-luks-device-with-tpm)

## Pre-installation

### Set the console keyboard layout and font

```bash
ls /usr/share/kbd/keymaps/**/*.map.gz   # layouts can be listed with
loadkeys us # to set the keyboard layout
setfont ter-120b    # Console fonts are located in /usr/share/kbd/consolefonts/ and can likewise be set with
```

### Verify the boot mode

```bash
cat /sys/firmware/efi/fw_platform_size  # To verify the boot mode
```

### Connect to the internet

```bash
ping archlinux.org  # The connection may be verified with
```

If it's works, then go to [Update the system clock](#update-the-system-clock) chapter else connect Ethernet cable and retry or connect internet by Wi-Fi following the [next chapter](#authenticate-to-the-wireless-network).

#### Authenticate to the wireless network

```bash
iwctl   # get an interactive prompt
device list # list all Wi-Fi devices
station <device> scan # initiate a scan for networks
station <device> get-networks   # list all available networks
station <device> connect <SSID>
exit
```

### Update the system clock

```bash
timedatectl set-ntp true # ensure the system clock is accurate
```

### Partition the disks

```bash
# to identify disk devices
fdisk -l
lsblk
fdisk /dev/<disk name>
g # make new GPT partition table
n # new partition
t # change partition type
w # write changes and exit
```

> In this point I've 2 partitions:
>
> | Partition number | Size | Type |
> |---|---|---|
> |1 | 512M | EFI System [1] |
> |2 | all free sector remaining  | Linux root (x86_64) [23] |

#### Encrypt root partition

```bash
cryptsetup -y -v luksFormat /dev/<root_partition>
cryptsetup open /dev/<root_partition> cryptlvm
```

#### Preparing the logical volumes

```bash
pvcreate /dev/mapper/cryptlvm
vgcreate vg /dev/mapper/cryptlvm

lvcreate -L 128G vg -n root
lvcreate -l 100%FREE vg -n home
lvreduce -L -256M vg/home
```

### Format the partitions

```bash
mkfs.fat -F 32 /dev/<efi_system_partition>

mkfs.ext4 /dev/vg/root
mkfs.ext4 /dev/vg/home
```

### Mount the file systems

```bash
mount /dev/vg/root /mnt
mount --mkdir /dev/vg/home /mnt/home
mount --mkdir /dev/<efi_system_partition> /mnt/efi
```

## Installation

### Select the mirrors

```bash
reflector -l 10 --sort rate -p https --save /etc/pacman.d/mirrorlist
```

### Install essential packages

```bash
pacstrap -K /mnt base linux linux-firmware lvm2 intel-ucode vim sbctl networkmanager terminus-font sudo
```

## Configure the system

### Fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

### Chroot

```bash
arch-chroot /mnt
```

### Time zone

```bash
ln -sf /usr/share/zoneinfo/<Region>/<City> /etc/localtime
hwclock --systohc
```

### Localization

Edit `/etc/locale.gen` uncommenting wanted locales.

```bash
locale-gen
echo "LANG=en_US.UTF-8" > /etc/locale.conf
echo "KEYMAP=us" > /etc/vconsole.conf
echo "FONT=ter-120n" >> /etc/vconsole.conf
```

### Network configuration

```bash
echo "<hostname>" > /etc/hostname
```

### Setting mkinitcpio

Modify the file ...

```bash
vim /etc/mkinitcpio.d/linux.preset
```

... to make like this:

```bash
# mkinitcpio preset file for the 'linux' package

#ALL_config="/etc/mkinitcpio.conf"
ALL_kver="/boot/vmlinuz-linux"
ALL_microcode=(/boot/intel-ucode.img)

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

Modify this file...

```bash
vim /etc/mkinitcpio.conf
```

... and modify MODULES and HOOKS:

```bash
# ...
MODULES=(tpm_tis)
# ...
HOOKS=(base systemd autodetect modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)
```

### Initramfs

```bash
mkdir -p /efi/EFI/Linux
mkinitcpio -p linux
```

### Boot loader

#### Installing the EFI boot manager

```bash
bootctl --esp-path=/efi install
```

#### Configuration

##### Loader configuration

set this file...

```bash
vim /efi/loader/loader.conf
```

in this...

```bash
default  arch-linux.efi
timeout  0
console-mode auto
editor   no
```

### Kernel cmdline

```bash
vim /etc/kernel/cmdline
```

write in this file...

```bash
rd.luks.name=<root_partition_UUID_value>=cryptlvm root=/dev/vg/root rw

# to insert the <root_partition_UUID_value>, in Normal mode, write ":read ! blkid /dev/<root_partition>" and take only the UUID value.
```

### Root password

```bash
passwd
```

### Manage users

```bash
useradd -mG wheel <user_name>
passwd matteo
EDITOR=vim visudo # to uncomment '%wheel ALL=(ALL:ALL) ALL'
```

### Configure NetworkManager service

```bash
systemctl enable NetworkManager.service
```

### Reboot

```bash
exit
umount -R /mnt
cryptsetup close cryptlvm
reboot
```

## Post-installation

### Reconnect Wifi

Write and execute `nmtui` to manage cabled and wireless connections easily.

### Secure boot

#### Create Secure boot keys

```bash
sbctl status
sudo sbctl create-keys
sudo sbctl enroll-keys -m
```

#### Sign unsigned files

list all unsigned file to sign

```bash
sudo sbctl verify
```

sign all files...

```bash
sudo sbctl sign-all
sudo reboot
```

check that all file is signed

### Automatic decrypt LUKS device with TPM

Find TPM path with...

```bash
systemd-cryptenroll --tpm2-device=list
```

... and put LUKS passphrase in TPM

```bash
sudo systemd-cryptenroll --tpm2-device=</path/to/tpm2_device> --tpm2-pcrs=0+7 /dev/<root_partition>

# If it don't work, use the following command to wipe the tpm slot 1 and retry
systemd-cryptenroll /dev/<root_partition> --wipe-slot=1
```

Reboot to check that works and...

![That's all folks][That's_all_folks]

[That's_all_folks]: https://pixy.org/src/58/584218.jpg
