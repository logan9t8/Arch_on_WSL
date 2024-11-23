# Install ArchLinux on WSL

## Enable WSL
```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
# Reboot
```

## Display Scaling - https://github.com/microsoft/wslg/issues/23
C:\ProgramData\Microsoft\WSL\\.wslconfig
```powershell
[system-distro-env]
WESTON_RDP_DISABLE_HI_DPI_SCALING=false
WESTON_RDP_DISABLE_FRACTIONAL_HI_DPI_SCALING=false
WESTON_RDP_DEBUG_DESKTOP_SCALING_FACTOR=125
```

## Install ArchWSL
```powershell
mkdir -p C:\Arch && cd C:\Arch
Invoke-WebRequest -URI https://github.com/yuk7/ArchWSL/releases/latest/download/Arch_Online.zip  -OutFile Arch.zip
Expand-Archive .\Arch.zip -DestinationPath .
.\Arch.exe
rm .\Arch.zip && rm .\preset.json
wsl
```

## Update Keyring
```shell
pacman-key --init
pacman-key --populate
pacman -Sy archlinux-keyring
```

## Update system
```shell
pacman -Syu
```

## Locale
```shell
sed -i '/#en_US.UTF-8 UTF-8/s/^#//g' /etc/locale.gen
locale-gen
cat > /etc/locale.conf 1> /dev/null << EOF
LANG="en_US.UTF-8"
EOF
```

## D-Bus
```shell
cat > /etc/profile.d/dbus.sh 1> /dev/null << EOF
export \$(dbus-launch)
EOF
chmod +x /etc/profile.d/dbus.sh
dbus-uuidgen --ensure
```

## Reflector
```shell
pacman -S reflector
reflector --save /etc/pacman.d/mirrorlist --protocol https --latest 20 --sort rate --download-timeout 30
```

## Uninstall unneeded packages
```shell
pacman -Rns arch-install-scripts nano
```

## Install needed packages
```shell
pacman -S tmux zsh
```

## Annoyances
```shell
ln -s /usr/bin/vim /usr/bin/vi
```

## Create user
```shell
useradd -s /usr/bin/zsh -G wheel <username>
echo "%wheel ALL=(ALL) ALL" > /etc/sudoers.d/wheel
passwd <username>
```

## Set root password
```shell
passwd root
```

## Enable systemd & Custom DNS
```shell
cat > /etc/wsl.conf 1> /dev/null << EOF
[boot]
systemd=true
generateResolvConf=false
EOF
cat > /etc/resolv.conf 1> /dev/null << EOF
nameserver       1.1.1.1
nameserver       1.0.0.1
EOF
```

## Set default user
```powershell
exit
wsl --shutdown
.\Arch.exe config --default-user <username>
wsl
```

## Lock root account
```shell
sudo passwd -l root
```

## Install make utils
```shell
sudo pacman -S --needed base-devel
sudoedit /etc/makepkg.conf
# Replace MAKEFLAGS=-j2 with -j$(nproc)
```

## LSB Packages
```shell
sudo pacman -S --needed bc nss perl time
sudo pacman -S --needed gtk3 gtk4 libxslt qt5-base qt6-base
```

## Enable required pacman options
```shell
sudo sed -ri '/HookDir|Color|ParallelDownloads/s/^#//g' /etc/pacman.conf
```

## Pacman related packages
```shell
sudo pacman -S pacman-contrib pacutils
```

## Zsh rehash hook
```shell
sudo mkdir -p /etc/pacman.d/hooks
sudo tee /etc/pacman.d/hooks/zsh-rehash.hook 1> /dev/null << EOF
[Trigger]
Type = Package
Operation = Install
Target = *

[Action]
When = PostTransaction
Exec = /usr/bin/zsh -c "zstyle ':completion:*' rehash true"
EOF
```

## Command not found
```shell
sudo pacman -S pkgfile
sudo systemctl enable --now pkgfile-update.timer
sudo pkgfile -u
```

## File Handling
```shell
sudo pacman -S xdg-utils perl-file-mimeinfo
```

## Home Directory
```shell
sudo mkdir /home/<username>
sudo chown <username>:<username> /home/<username>
sudo pacman -S xdg-user-dirs
xdg-user-dirs-update
sudo pacman -R xdg-user-dirs
rmdir ~/*
# This way ~/.config/user-dirs.dirs gets created

# Option 1 - Let Windows & WSL home be the same
# NOTE: OneDrive dirs don't map! C:\Users\<Username> contains dummy dirs!
rmdir /home/<username>
ln -s /mnt/c/Users/<Username> /home/<username>
mkdir ~/{Public,Documents,Templates}

# Option 2 - Let WSL home be separate
ln -s /mnt/c/Users/<username>/Downloads ~
ln -s /mnt/c/Users/<username>/OneDrive/Music ~
ln -s /mnt/c/Users/<username>/OneDrive/Videos ~
ln -s /mnt/c/Users/<username>/OneDrive/Desktop ~
ln -s /mnt/c/Users/<username>/OneDrive/Documents ~
ln -s /mnt/c/Users/<username>/OneDrive/Pictures ~
mkdir ~/{Public,Templates}
```

## xdg-open
```shell
mkdir -p ~/.local/bin
curl -o ~/.local/bin/xdg-open https://raw.githubusercontent.com/4U6U57/wsl-open/master/wsl-open.sh
chmod +x ~/.local/bin/xdg-open
#Make sure ~/.local/bin/ is PREPENDED to path to override /usr/sbin/xdg-open. i,e. PATH="$HOME/.local/bin:$PATH"
```

## Explorer
```shell
cat > $HOME/.local/bin/explorer 1> /dev/null << EOF
##!/bin/bash
[[ "\$1" ]] || set -- "\$HOME"
explorer.exe "\$(wslpath -w "\$1")"
EOF
chmod +x $HOME/.local/bin/explorer
```

## Firefox
```shell
cat > $HOME/.local/bin/firefox 1> /dev/null << EOF
##!/bin/bash
[[ "\$1" ]] || set -- "about:newtab"
"/mnt/c/Program Files/Mozilla Firefox/firefox.exe" "\$1"
EOF
chmod +x $HOME/.local/bin/firefox
```

## VLC
```shell
cat > $HOME/.local/bin/vlc 1> /dev/null << EOF
##!/bin/bash
"/mnt/c/Program Files/VideoLAN/VLC/vlc.exe" "\$@"
EOF
chmod +x $HOME/.local/bin/vlc
```

## Install yay
```shell
sudo pacman -S git
git clone https://aur.archlinux.org/yay-bin.git
cd yay-bin && makepkg -si
cd .. && rm -rf yay-bin
```

## Zsh
```shell
sudo pacman -S cmake fzf powerline vim-powerline python-pygments zsh-completions zsh-history-substring-search zsh-syntax-highlighting zsh-theme-powerlevel10k-bin-git
yay -S autojump
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
mv .zshrc .zshrc.omz
mv .zshrc.pre-oh-my-zsh .zshrc
ln -sf /usr/share/zsh-theme-powerlevel10k $HOME/.oh-my-zsh/themes/powerlevel10k
source /usr/share/zsh-theme-powerlevel10k/powerlevel10k.zsh-theme
mkdir -p $HOME/.config/powerline
cp -r $(python -c "import site; print(site.getsitepackages()[0])")/powerline/config_files/* $HOME/.config/powerline
```

## Theming
```shell
yay -S breeze unzip windows8-cursor
sudo tee /etc/profile.d/gui_vars.sh 1> /dev/null << EOF
export GTK_THEME=Adwaita:dark
export QT_QPA_PLATFORMTHEME=qt6ct
export MOZ_GTK_TITLEBAR_DECORATION=client
EOF
sudo chmod +x /etc/profile.d/gui_vars.sh
sudo tee /usr/share/icons/default/index.theme 1> /dev/null << EOF
[Icon Theme]
Inherits=Windows8-cursor
EOF
yay -S gnome-tweaks qt6ct-kde
```

## Fuse for AppImages (If required)
```shell
sudo pacman -S fuse
```

## Install Disk Utilities (If required)
```shell
sudo pacman -S nautilus baobab gnome-disk-utility
```

## Chrome Dark mode (If required)
```shell
cat > $HOME/.config/chrome-flags.conf 1> /dev/null << EOF
--force-dark-mode
--enable-features=WebUIDarkMode
EOF
```

## Journal size
```shell
sudoedit /etc/systemd/journald.conf
    SystemMaxUse=100M
sudo systemctl restart systemd-journald
journalctl --vacuum-size=100M
```

## Remove packages & cache
```shell
sudo paccache -rk0
yay -Qtdq | yay -Rns -
sudo pacman -Scc
```

## If x11 is broken
```shell
ln -s /mnt/wslg/.X11-unix /tmp/.X11-unix
```

## Trim ext4.vhdx
```powershell
exit
wsl --shutdown
diskpart
select vdisk file=C:\Arch\ext4.vhdx
compact vdisk
```
