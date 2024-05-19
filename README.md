# My Arch Linux Installation <!-- omit in toc -->

## Summary <!-- omit in toc -->
- [Pre-installation](#pre-installation)
	- [Set the console keyboard layout and font](#set-the-console-keyboard-layout-and-font)
	- [Verify the boot mode](#verify-the-boot-mode)
	- [Connect to the internet](#connect-to-the-internet)
		- [Authenticate to the wireless network](#authenticate-to-the-wireless-network)
	- [\[OPTIONAL\] ssh connection](#optional-ssh-connection)
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
	- [Kernel cmdline](#kernel-cmdline)
	- [Initramfs](#initramfs)
	- [Boot loader](#boot-loader)
		- [Installing the EFI boot manager](#installing-the-efi-boot-manager)
		- [Configuration](#configuration)
			- [Loader configuration](#loader-configuration)
	- [Root password](#root-password)
	- [Manage users](#manage-users)
	- [Configure iwd service](#configure-iwd-service)
	- [Setup ssh server to persiste after reboot](#setup-ssh-server-to-persiste-after-reboot)
	- [Reboot](#reboot)
- [Post-installation](#post-installation)
	- [Reconnect Wifi](#reconnect-wifi)
	- [SSH reconnection](#ssh-reconnection)
	- [Secure boot](#secure-boot)
		- [Create Secure boot keys](#create-secure-boot-keys)
		- [Sign unsigned files](#sign-unsigned-files)
	- [Automatic decrypt LUKS device with TPM](#automatic-decrypt-luks-device-with-tpm)

## Pre-installation

### Set the console keyboard layout and font

```bash
ls /usr/share/kbd/keymaps/**/*.map.gz	# layouts can be listed with
loadkeys us	# to set the keyboard layout

setfont -d	# set font with default value, other fonts are located in "/usr/share/kbd/consolefonts/" 
```

### Verify the boot mode

```bash
cat /sys/firmware/efi/fw_platform_size	# To verify the boot mode
# '64' a 64-bit system is booted in x64 UEFI mode
# '32' a 32-bit system is booted in IA32 UEFI mode
```

### Connect to the internet

```bash
ping archlinux.org	# The connection may be verified with 
```

If it's works, then go to [Update the system clock](#update-the-system-clock) chapter else connect Ethernet cable and retry or connect internet by Wi-Fi following the [next chapter](#authenticate-to-the-wireless-network).

#### Authenticate to the wireless network

```bash
iwctl	# get an interactive prompt
device list	# list all Wi-Fi devices
station <device> scan	# initiate a scan for networks
station <device> get-networks	# list all available networks
station <device> connect <SSID>
exit
```

### [OPTIONAL] ssh connection

I want remotely connect to the PC from another because you can easily copy-paste the commands

```bash
passwd # a password like '1234' it's ok

ip addr # see which IP address has
```

Now from another pc...

```bash
ssh root@<IPaddress>
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

# open disk utility to modify the disk table of <the_disk_to_be_partitioned>
fdisk /dev/<the_disk_to_be_partitioned>
g	# make new GPT partition table
n	# new partition
p	# print to see the situation
t	# change partition type
w	# write changes and exit
```

> In this point I've 2 partitions:
>
> | Partition number | Size                      | Type                     |
> | ---------------- | ------------------------- | ------------------------ |
> | 1                | 1G                        | EFI System [1]           |
> | 2                | all free sector remaining | Linux root (x86_64) [23] |

#### Encrypt root partition

```bash
cryptsetup -y -v luksFormat /dev/<root_partition>
cryptsetup open /dev/<root_partition> cryptlvm
```

#### Preparing the logical volumes

```bash
pvcreate /dev/mapper/cryptlvm
vgcreate vg /dev/mapper/cryptlvm

lvcreate -l 100%FREE vg -n root
lvreduce -L -256M vg/root	# see https://wiki.archlinux.org/title/Dm-crypt/Encrypting_an_entire_system#Preparing_the_logical_volumes to understand why
```

### Format the partitions

```bash
mkfs.fat -F 32 /dev/<efi_system_partition>

mkfs.ext4 /dev/vg/root
```

### Mount the file systems

```bash
mount /dev/vg/root /mnt
mount --mkdir /dev/<efi_system_partition> /mnt/efi
```

## Installation

### Select the mirrors

```bash
reflector -l 10 --sort rate -p https --save /etc/pacman.d/mirrorlist
```

### Install essential packages

```bash
pacstrap -K /mnt base linux-lts linux-firmware intel-ucode lvm2 sof-firmware iwd nano vim man-db man-pages sudo openssh sbctl tpm2-tss
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
nano /etc/mkinitcpio.d/linux-lts.preset
```

... to make like this:

```bash
# mkinitcpio preset file for the 'linux-lts' package

# ...

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
nano /etc/mkinitcpio.conf
```

... and modify MODULES and HOOKS:

```bash
# ...
HOOKS=(base systemd autodetect microcode modconf kms keyboard sd-vconsole block sd-encrypt lvm2 filesystems fsck)
# ...
```

### Kernel cmdline
 
```bash
vim /etc/kernel/cmdline
```

write in this file...

```bash
rd.luks.name=<root_partition_UUID_value>=cryptlvm root=/dev/vg/root rw quiet loglevel=3 systemd.show_status=auto rd.udev.log_level=3

# to insert the <root_partition_UUID_value>, in Normal mode, write ":read ! blkid /dev/<root_partition>" and take only the UUID value (" " inclused).
```

### Initramfs

```bash
mkdir -p /efi/EFI/Linux
mkinitcpio -p linux-lts
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
nano /efi/loader/loader.conf
```

in this...

```bash
default  arch-linux-lts.efi
timeout  0
console-mode auto
editor   no
```

### Root password

```bash
passwd # the real root password
```

### Manage users

```bash
useradd -mG wheel <user_name>
passwd <user_name>
EDITOR=vim visudo # to uncomment '%wheel ALL=(ALL:ALL) ALL'
```

### Configure iwd service

write in file...

```bash
mkdir -p /etc/iwd/

touch /etc/iwd/main.conf

nano /etc/iwd/main.conf
```

... to this:

```bash
[Scan]
DisablePeriodicScan=true

[General]
EnableNetworkConfiguration=true

[Network]
RoutePriorityOffset=300
EnableIPv6=true
NameResolvingService=systemd
```

```bash
systemctl enable systemd-resolved iwd
```

### Setup ssh server to persiste after reboot

```bash
systemctl enable sshd
```

Uncomment `Port` in `/etc/ssh/sshd_config` and change the port number with a random higher number (1000~9999)

### Reboot

```bash
exit
umount -R /mnt
cryptsetup close cryptlvm
reboot
```

## Post-installation

### Reconnect Wifi

Follow these steps to [connect a wifi](#connect-to-the-internet) and check that is it works with `ping`.

```bash
ip addr # check the IP address
```

### SSH reconnection

From another PC...

```bash
ssh <user_name>@<IP_address> -p <port_number>
```

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
sudo sbctl sign -s <path/to/file_to_sign>
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
sudo systemd-cryptenroll /dev/<root_partition> --wipe-slot=1
```

Reboot to check that works.
