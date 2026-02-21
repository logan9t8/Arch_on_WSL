# Install Arch Linux on WSL

## Enable WSL
```powershell
wsl --install --no-distribution
wsl --update
# Reboot
```

## Fractional Scaling - https://github.com/microsoft/wslg/issues/23 (If required)
`C:\Users\<username>\.wslconfig`
```powershell
[system-distro-env]
WESTON_RDP_DISABLE_HI_DPI_SCALING=false
WESTON_RDP_DISABLE_FRACTIONAL_HI_DPI_SCALING=false
WESTON_RDP_DEBUG_DESKTOP_SCALING_FACTOR=125
```

## Install Arch 
```powershell
# amd64 - https://wiki.archlinux.org/title/Install_Arch_Linux_on_WSL
wsl --install archlinux

# arm64 - Option #1 https://archlinuxarm.org/platforms/armv8/generic
mkdir -p C:\Arch && cd C:\Arch
Invoke-WebRequest -URI http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz -OutFile Arch.tar.gz
wsl --import archlinux C:\archlinux archlinuxarm.tar.gz
rm .\archlinuxarm.tar.gz
wsl -d archlinux

# arm64 - Option #2 Self-build from existing alarm system
git clone https://gitlab.archlinux.org/archlinux/archlinux-wsl.git
# Prepend /rootfs/etc/pacman.d/mirrorlist with the below line
Server = http://mirror.archlinuxarm.org/$arch/$repo
# Append in /scripts/build-image.sh after line 'fakechroot -- fakeroot -- chroot "$BUILDDIR" pacman-key --populate'
fakechroot -- fakeroot -- chroot "$BUILDDIR" pacman-key --populate archlinuxarm
# Then from windows
wsl --install --from-file archlinux-<date>.wsl
```

## Update Keyring
```shell
pacman-key --init
pacman-key --populate
pacman-key --populate archlinuxarm #arm64 Option #1 only
pacman -Sy archlinux-keyring
```

## Configure pacman options
```shell
sed -ri '/HookDir|Color/s/^#//g' /etc/pacman.conf
sed -i 's/^NoProgressBar/#NoProgressBar/g' /etc/pacman.conf
```

## Update system
```shell
pacman -Syu
```

## Terminal packages
```shell
pacman -S sudo tmux vim zsh
ln -s /usr/bin/vim /usr/bin/vi
```

## Initialize systemd
```shell
systemd-machine-id-setup
```

## WSL Config
```shell
tee -a /etc/wsl.conf > /dev/null <<EOF

[network]
generateResolvConf=false

[user]
default=dinesh
EOF
```

## Custom DNS
```shell
tee /etc/resolv.conf > /dev/null << EOF
nameserver       1.1.1.1
nameserver       1.0.0.1
EOF
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
tee -a /etc/pam.d/login > /dev/null << EOF
session optional pam_systemd.so
```

## Reflector (If required)
```shell
pacman -S reflector
reflector --save /etc/pacman.d/mirrorlist --protocol https --latest 20 --sort rate --download-timeout 30
```

## Remove unneeded packages, services and user (arm64 Option #1)
```shell
pacman -Rns dhcpcd linux-aarch64 linux-firmware nano netctl vi

systemctl disable --now getty@.service getty@tty1.service console-getty.service
systemctl disable --now systemd-network-generator.service
systemctl disable --now systemd-networkd.service
systemctl disable --now systemd-networkd-wait-online.service
systemctl disable --now systemd-resolved.service
systemctl disable --now systemd-timesyncd.service
systemctl disable --now systemd-networkd.socket systemd-networkd-varlink.socket
systemctl disable --now systemd-resolved-monitor.socket systemd-resolved-varlink.socket

userdel alarm
rm -rf /home/alarm
```

## Create user
```shell
useradd -m -G wheel -s /usr/bin/zsh <username>
EDITOR='sed -i "/^# %wheel ALL=(ALL:ALL) ALL/s/^# //"' visudo
passwd <username>
```

## Set root password
```shell
passwd root
```

# Switch to new user
```powershell
exit
wsl --terminate archlinux
wsl -d archlinux
```

## Lock root account
```shell
sudo passwd -l root
```

## Install make utils (For building AUR packages)
```shell
sudo pacman -S --needed base-devel
sudoedit /etc/makepkg.conf
# Replace MAKEFLAGS=-j2 with -j$(nproc)
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

## Man
```shell
sudo pacman -S man-db man-pages
```

## File Handling
```shell
sudo pacman -S xdg-utils perl-file-mimeinfo
```

## Home Directory
```shell
sudo pacman -S xdg-user-dirs
xdg-user-dirs-update
sudo pacman -R xdg-user-dirs
rmdir ~/*
# This way ~/.config/user-dirs.dirs gets created

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

## Install yay
```shell
sudo pacman -S git
git clone https://aur.archlinux.org/yay-bin.git
cd yay-bin && makepkg -si
cd .. && rm -rf yay-bin
```

## Shell
```shell
sudo pacman -S fd fzf powerline vim-powerline python-pygments ripgrep \
  zsh-autosuggestions zsh-completions zsh-history-substring-search zsh-syntax-highlighting
yay -S autojump oh-my-posh-bin
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
mv .zshrc .zshrc.omz
# Ensure dotfiles .zshrc, .vimrc, .tmux.conf, .tmux_bindings.sh, .ripgreprc
chmod +x .tmux_bindings.sh
mkdir -p $HOME/.local/share/oh-my-posh/themes/
cp -p /usr/share/oh-my-posh/themes/powerlevel10k_rainbow.omp.json $HOME/.local/share/oh-my-posh/themes/powerlevel10k_rainbow.omp.json
sed -i 's/{{\.Icon}}/\\uF303/g' $HOME/.local/share/oh-my-posh/themes/powerlevel10k_rainbow.omp.json

## Journal size
```shell
sudo sed -i 's/^#SystemMaxUse=/SystemMaxUse=50M/g' /etc/systemd/journald.conf
sudo systemctl restart systemd-journald
journalctl --vacuum-size=100M
```

## Remove package cache and unneeded locales
```shell
sudo paccache -ruk0
yay -Qtdq | yay -Rns -
yay -Scc
sudo pacman -S localepurge-hook
sudo sed -i 's/^NEEDSCONFIGFIRST/#NEEDSCONFIGFIRST/g' /etc/locale.nopurge
sudo sed -i 's/^#DONTBOTHERNEWLOCALE/DONTBOTHERNEWLOCALE/g' /etc/locale.nopurge
sudo localepurge
```

# GUI (If needed)
```shell
sudo pacman -S gtk3 gtk4 qt5-base qt6-base
yay -S breeze breeze-gtk windows8-cursor gnome-tweaks qt6ct
gsettings set org.gnome.desktop.interface cursor-size 12
sudo tee /usr/share/icons/default/index.theme 1> /dev/null << EOF
[Icon Theme]
Inherits=Windows8-cursor
EOF
cat > $HOME/.zprofile 1> /dev/null << EOF
export GTK_THEME=Breeze:dark
export QT_QPA_PLATFORMTHEME=qt6ct
export MOZ_GTK_TITLEBAR_DECORATION=client
export XCURSOR_THEME=Windows8-cursor
export XCURSOR_SIZE=12
EOF
```

## If x11 is broken
```shell
ln -s /mnt/wslg/.X11-unix /tmp/.X11-unix
```

## Trim ext4.vhdx
```powershell
exit
# (Option #1 - Automatic)
wsl --manage archlinux --set-sparse true
# (Option #2 - Manual - Optimize-VHD (If Hyper-V is installed))
Optimize-VHD -Path C:\Users\Dinesh\AppData\Local\wsl\<hash>\ext4.vhdx -Mode Full
# (Option #3 - Manual - DiskPart)
wsl --shutdown
diskpart
select vdisk file=C:\Arch\ext4.vhdx
attach vdisk readonly
compact vdisk
detach vdisk
exit
```

## Check disk usage
```shell
sudo pacman -S ncdu
sudo ncdu -x /
```

# System info
```shell
yay -S fastfetch
fastfetch
```