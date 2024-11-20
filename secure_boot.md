# Secure Boot <!-- omit in toc -->

## Summary <!-- omit in toc -->

- [Setup](#setup)
- [Check Secure Boot status](#check-secure-boot-status)
- [Create Secure Boot keys](#create-secure-boot-keys)
- [Sign unsigned files](#sign-unsigned-files)
- [Automatic signig with pacman hook](#automatic-signig-with-pacman-hook)

## Setup

For this chapter your pc needs `sbctl` package installed

```bash
sudo pacman -S sbctl
```

## Check Secure Boot status

```bash
sbctl status # check that sbctl is not installed and secure boot is disabled
```

## Create Secure Boot keys

Before continue, go to your BIOS settings and set secure boot mode to Setup mode

```bash
sudo sbctl create-keys
sudo sbctl enroll-keys -m
sbctl status # re-check
```

## Sign unsigned files

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

```bash
sudo sbctl verify
```

## Automatic signig with pacman hook

```bash
# KERNEL=(linux | linux-lts)
sudo mkinitcpio -p $KERNEL # check that at the end you'll see sbctl log about signing the updated initramfs
```

Reboot and enable Secure Boot to check that works
