---
title: Simple Arch Linux setup with EFISTUB
date: 2020-10-01 03:00:00 Z
categories:
- linux
tags:
- arch
- efi
- setup
---

# Install
```shell
cfdisk -z /dev/sda # choose gpt, make small 256MB boot partition with type EFI System (sda1) and the rest Linux Filesystem (sda2)
mkfs.vfat -F 32 /dev/sda1
mkfs.ext4 -L rootfs /dev/sda2
mount /dev/sda2 /mnt
mkdir -p /mnt/boot/efi
mount /dev/sda1 /mnt/boot/efi
pacstrap /mnt base linux linux-firmware efibootmgr vim
genfstab /mnt > /mnt/etc/fstab
arch-chroot /mnt
efibootmgr -c -d /dev/sda -p 1 -L arch -l "\vmlinuz-linux" -u "root=LABEL=rootfs initrd=\initramfs-linux.img net.ifnames=0"
passwd
cp /boot/{vmlinuz-linux,initramfs-linux.img} /boot/efi/
exit
umount -R /mnt
reboot
```

# Locale (language, keyboard layout)
uncomment your desired locale in `/etc/locale.gen`
```shell
locale-gen
localectl set-keymap uk
localectl set-locale en_GB.UTF-8
```

# Simple DHCP with systemd-networkd and systemd-resolved
```text
/etc/systemd/network/dhcp.network
[Match]
Name=eth0

[Network]
DHCP=yes
```

```shell
systemctl enable systemd-networkd systemd-resolved
ln -sf /lib/systemd/resolv.conf /etc/resolv.conf
systemctl start systemd-networkd systemd-resolved
```

# Create user
```shell
useradd -m -G wheel,audio,video,power,network <username>
```

# My personal packages
```shell
pacman -S man-db firefox bash-completion pulseaudio pavucontrol htop git xorg xfce4 xfce4-pulseaudio-plugin sudo ttf-ubuntu-font-family papirus-icon-theme
```

# Autologin
```text
/etc/systemd/system/getty@tty1.service.d/override.conf
[Service]
ExecStart=
ExecStart=-/usr/bin/agetty --autologin <username> --noclear %I $TERM
```
```text
~/.xinitrc
exec xfce4-session
```
```text
~/.bash_profile
...
if [[ ! $DISPLAY && $XDG_VTNR -eq 1 ]]; then
    exec startx
fi
```

# Running as VMWare Guest
```text
/etc/mkinitcpio.conf
...
MODULES=(vsock vmw_vsock_vmci_transport vmw_balloon vmw_vmci vmwgfx)
...
```
```shell
pacman -S open-vm-tools gtk2 gtkmm3 xf86-video-vmware
systemctl enable vmtoolsd
mkinitcpio -P
cp /boot/initramfs-linux.img /boot/efi/initramfs-linux.img
```
add `vmware-user` to `~/.xinitrc` if not using xfce (it has an autostart entry in /etc/xdg/autostart/ already)
