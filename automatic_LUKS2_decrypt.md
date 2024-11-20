# Post-installation <!-- omit in toc -->

## Summary <!-- omit in toc -->

- [Reconnect Wifi](#reconnect-wifi)
- [\[OPTIONAL\] SSH configuration](#optional-ssh-configuration)
  - [\[OPTIONAL\] SSH reconnection](#optional-ssh-reconnection)
  - [\[OPTIONAL\] Setup ssh server to persiste after reboot](#optional-setup-ssh-server-to-persiste-after-reboot)
- [Automatic decrypt LUKS device with TPM](#automatic-decrypt-luks-device-with-tpm)
  - [List all avaiable TPM devices](#list-all-avaiable-tpm-devices)
  - [Insert LUKS passphrase in TPM device](#insert-luks-passphrase-in-tpm-device)
    - [\[OPTIONAL\] Troubleshoot TPM2](#optional-troubleshoot-tpm2)

## Reconnect Wifi

Follow these steps to [connect a wifi](./base.md#connect-to-the-internet) and check that is it works with `ping`.

```bash
ip addr # check the IP address
```

## [OPTIONAL] SSH configuration

For this step, must need `openssh` package

```bash
sudo pacman -S openssh

sudo systemctl start sshd
```

Uncomment `Port` in `/etc/ssh/sshd_config` and change the default port number with a random higher number (1000~9999) for better security

### [OPTIONAL] SSH reconnection

From another PC...

```bash
ssh $USERNAME@$IP_ADDR -p $SSH_PORT
```

### [OPTIONAL] Setup ssh server to persiste after reboot

To keep ssh server enabled after reboots, use this command:

```bash
systemctl enable sshd
```

## Automatic decrypt LUKS device with TPM

> THIS CHAPTER NEEDS THAT YOUR PC HAS SECURE BOOT ENABLED

For this chapter your pc needs `tpm2-tss` package installed

```bash
sudo pacman -S tpm2-tss
```

### List all avaiable TPM devices

Find TPM path with...

```bash
systemd-cryptenroll --tpm2-device=list
```

### Insert LUKS passphrase in TPM device

```bash
# ROOT_PARTITION=nvme0n1p2
sudo systemd-cryptenroll --tpm2-device=auto /dev/$ROOT_PARTITION # this command, write the LUKS2 passwod in TPM slot 1

# If it don't work, use the following command to wipe the tpm slot and retry
sudo systemd-cryptenroll /dev/$ROOT_PARTITION --wipe-slot=$TPM_SLOT
```

Reboot to check that works

#### [OPTIONAL] Troubleshoot TPM2

If don't work, then modify `/etc/mkinitcpio.conf` to add a driver in MODULE

```bash
MODULE=($TPM2_DRIVER)
```

recreate initramfs

```bash
# KERNEL=(linux | linux-lts)
sudo mkinitcpio -p linux
```

Reboot to check that works
