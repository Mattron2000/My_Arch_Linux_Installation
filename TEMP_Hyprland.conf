# Hyprland <!-- omit in toc -->

## Summary <!-- omit in toc -->

- [Install Hyprland](#install-hyprland)
  - [Start Hyprland](#start-hyprland)
- [Configure Hyprland](#configure-hyprland)
  - [Monitors](#monitors)
- [Better pacman](#better-pacman)
- [Install paru](#install-paru)
- [Install VScode](#install-vscode)
- [Java](#java)
- [Git configuration](#git-configuration)

## Install Hyprland 

```bash
sudo pacman -S hyprland kitty pipewire wireplumber xdg-desktop-portal-hyprland qt6-wayland qt5-wayland polkit-kde-agent
```

### Start Hyprland

```bash
Hyprland
```

## Configure Hyprland

```bash
nano .config/hypr/hyprland.conf
```

### Monitors

```bash
# ...
monitor=<monitor_name>,1920x1080@60,0x0,1 # take <monitor_name> from "hprctl monitors all" command, whats means other values? See here: https://wiki.hyprland.org/Configuring/Monitors/
# ...
#################
## AUTOSTART ###
#################
# ...
exec-once = /usr/lib/polkit-kde-authentication-agent-1
# ...
```

## Better pacman 

Under `[options]` and `# Misc options` of `/etc/pacman.conf`, uncomment and write these lines...

```bash
# ...
[options]
# ...
# Misc options
# ...
Color
# ...
VerbosePkgLists
ParallelDownloads = 5
ILoveCandy
```

## Install paru 

```bash
sudo pacman -S git
sudo pacman -S --needed base-devel
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si
```

<!-- TODO: To see the full size of OS: du -h / 2> /dev/null | tail -n 1 -->

## Install VScode

```bash
paru visual-studio-code-bin
paru -S firefox 
```

<https://youtu.be/mmqDYw9C30I>

## Java

```bash
paru -S java-runtime-common jdk21-openjdk

archlinux-java status
archlinux-java set <JAVA_VERSION>
```

## Git configuration

```bash
git config --global user.name "John Doe"
git config --global user.email johndoe@example.com
```

<!-- 
# Ricing (Optional, personal setup)

## Install non-fontamental packages

```bash
sudo pacman -S 
  neofetch  \
  bat       \
  fzf       \
  zoxide    \
  broot     \
  lsd       \
  git

broot # select Yes
```

## Organize .bashrc

Write bash files in this way...

```bash
#
# "/.bashrc
#

# If not running interactively, don't do anything 
[[ $- != *i* ]] && return

PS1='[\u@\h\W]\$ '

#zoxide configuration 
eval "$(zoxide init bash)"

#fastest way to source other file 
# example: include FILE 
include () { 
  [[ -f "$1" ]] && source "$1"
}

include ~/.bash_aliases
```

```bash
#
# "/.bash_aliases
#

alias sb='source "/.bashrc'

alias cl='clear' 

alias grep='grep --color=auto' 

alias cd='z'
alias cat='bat'

alias ls='lsd'
alias ll='ls -l' 
alias la='ls -la'
alias lt='ls --tree'

alias rm_unused_pkg='paru -Qdtq | paru -Rs -'
```

## Pacman wrapper

```bash
sudo pacman -S --needed base-devel
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si
cd ..
sudo rm -r paru
```

## Add man

```bash
paru -S man-db man-pages
```

## Install KDE

```bash
# select: all, all, 2 (pipewire-jack), 2 (wireplumber), 1 (gnu-free-fonts), 2 (phonon-qt5-vlc), 1 (pyside2), 1 (cronie), 30 (tesseract-data-eng)
paru -S plasma plasma-wayland-session kde-applications power-profiles-daemon xdg-desktop-portal-gtk xsettingsd sof-firmware packagekit-qt5

# remove legacy audio packages
paru -S pipewire-alsa pipewire-pulse # substitute 'pulseaudio' for 'pulseaudio'

# for better audio EQ
paru -S easyeffects

paru sddm-git --mflags si # workaround of 90 sec before shutdown

systemctl enable bluetooth
systemctl enable sddm
sudo reboot
```

## KDE Wallet

Open KWalletManager, if the wallet is open, close and delete it. Create a new wallet called `kdewallet` and the password is the same of `$USER`.

## CUPS service

```bash
paru -S cups system-config-printer
systemctl enable cups
```

## Fingerprint 

```bash
paru -S frintd
fprintd-enroll  # enroll right-index finger signature
```

### Login configuration

Write the following to the top of these files:

```bash
#
# /etc/pam.d/sddm
#

auth 			[success=1 new_authtok_reqd=1 default=ignore]  	pam_unix.so try_first_pass likeauth nullok
auth 			sufficient  	pam_fprintd.so
```

```bash
#
# /etc/pam.d/{login,su,sudo,system-local-login,kde}
#

auth 			sufficient  	pam_unix.so try_first_pass likeauth nullok
auth 			sufficient  	pam_fprintd.so
```

## CPU frequency scaling

```bash
paru -R power-profiles-daemon
git clone https://github.com/AdnanHodzic/auto-cpufreq.git
cd auto-cpufreq && sudo ./auto-cpufreq-installer  # select [I]nstall
auto-cpufreq --install

paru -S thermald btop
systemctl enable thermald

sudo reboot
```

## Firewall

```bash
paru -S ufw
```

To enable, open 'Application launcher' and search 'Firewall'.

## SDDM theme fix

```bash
touch /etc/sddm.conf
echo "[Theme]" > /etc/sddm.conf
echo "Current=breeze" >> /etc/sddm.conf
```

<!-- 
## Better bash

```bash
paru blesh-git
touch ~/.blerc
```

Copy and paste in .bashrc the follwing line to work: 
- `[[ $- == *i* ]] && source /usr/share/blesh/ble.sh --noattach --rcfile ~/.blerc` on top;
- `[[ $(BLE_VERSION-) ]] && ble-attach` on bottom. 
--> 
-->
