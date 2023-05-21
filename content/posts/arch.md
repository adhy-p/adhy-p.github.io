+++
date = 2022-11-01T16:42:36+08:00
title = "BTW I use Arch"
description = "Arch Linux Installation"
slug = ""
authors = []
tags = ["writeup"]
categories = []
externalLink = ""
series = []
+++

# Note: this is a work in progress.

# Part 1: Arch installation

## 0. Preparation

Resize the partition using windows disk utility. Allocate some free space for arch.

## 1. Downloading the image and create an installation media

### 1.1. Download

```
https://download.nus.edu.sg/mirror/archlinux/iso/2022.10.01/
```

Download archlinux-x86_64.iso and archlinux-x86_64.iso.sig

Verify the key

```
gpg --keyserver-options auto-key-retrieve --verify archlinux-x86_64.iso.sig
```

Confirm that the key fingerprint is shown at the the Developer page of arch.

### 1.2. Create media

Insert USB
Get the USB device name from

```
lsblk
```

```
sudo -i # cat won't work with sudo
cat archlinux-x86_64.iso > /dev/sda; sync
```

## 2. Booting using the USB

Press F1 (Thinkpad).
Disable the secure boot and move the boot priority for usb devices
Restart

## 3. Installation

### 3.1. Setting up network

```
vim /etc/wpa_supplicant/wpa_supplicant.conf
```

```
# /etc/wpa_supplicant/wpa_supplicant.conf
ctrl_interface=/run/wpa_supplicant
network={
  ssid="NUS_STU"
  scan_ssid=1
  key_mgmt=WPA-EAP
  eap=PEAP
  identity="nusstu\***"
  password="***"
  phase2="autheap=MSCHAPV2"
}
```

```
wpa_supplicant -B -i wlan0 /etc/wpa_supplicant/wpa_supplicant.conf
systemctl restart iwd.service
```

### 3.2. Update system clock

```
timedatectl status
```

### 3.3. Partition the disks

After that,

```
lsblk
fdisk /dev/nvme0n1
```

Inside fdisk:

```
Command (m for help): n
Paritition number (5-128, default 5): <enter>
First sector (XXX-YYY, default XXX): <enter>
Last sector, +/-sectors or +/-size: +8G # for swap partition

Command (m for help): t
Paritition number (5-128, default 5): <enter>
Partition type or alias (type L to list all): swap


Command (m for help): n
Paritition number (6-128, default 6): <enter>
First sector (XXX-YYY, default XXX): <enter>
Last sector, +/-sectors or +/-size: <enter> # root partition

Command (m for help): v
Command (m for help): w
```

### 3.4. Format the parititions

```
mkfs.ext4 /dev/nvme0n1p6 # we set part no. 6 as our root partition
mkswap /dev/nvme0n1p5
```

### 3.5. Mount the partitions

```
mount /dev/nvme0n1p6 /mnt
mount --mkdir /dev/nvme0n1p1 /mnt/boot
swapon /dev/nvme1n1p5
```

### 3.6. Install essential packages

```
pacstrap -K /mnt base linux linux-firmware
```

alternatively, at this stage, we can install more

```
pacstrap -K /mnt base linux linux-firmware
networkmanager vim man-db man-pages texinfo grub
efibootmgr
```

## 4. System configuration

```
genfstab -U /mnt >> /mnt/etc/fstab
arch-chroot /mnt
ln -sf /usr/share/zoneinfo/Singapore /etc/localtime
hwclock --systohc
```

```
# uncomment en_US.UTF-8 UTF-8 from /etc/locale.gen
locale-gen
echo "LANG=en_US.UTF-8" >> /etc/locale.conf
```

```
echo "thinkpad" > /etc/hostname
```

```
passwd # set up root password
```

### 4.1. Setup Boot loader

```
pacman -S grub efibootmgr os-prober
grub-install --target=x86_64-efi
--efi-directory=/boot --bootloader-id=GRUB
os-prober
uncomment
GRUB_DISABLE_OS_PROBER=false /etc/default/grub
grub-mkconfig -o
/boot/grub/grub.cfg
```

After that, reboot the system and arch is installed.

# Part 2: Setting up the system

## 1. Connecting to WiFi

iwctl is not installed by default; our previous attempt with wpa_supplicant does not work. But, now we have nmcli.

```
nmcli device wifi list
nmcli con add type wifi ifname wlp1s0 con-name NUS ssid NUS_STU
nmcli con edit id NUS

# inside nmcli
set ipv4.method auto
set 802-1x.eap peap
set 802-1x.phase2-auth mschapv2
set 802-1x.identity nusstu\\e0407797
set wifi-sec.key-mgmt wpa-eap
set 802-1x.password ***********
save
activate
exit
```

## 2. Setting up pacman

For some reason, the pacman keyring used to verify the packages were broken. The pacman keyring needs to be reinstalled.

```
vim /etc/pacman.conf

# do not check the packages
SigLevel = Never
```

After changing the config file, reinstall the keys:

```
pacman -S archlinux-keyring
pacman-key --init
pacman-key --refresh-keys
```

After that, return the config file to the default settings

```
vim /etc/pacman.conf
SigLevel =
```

## 3. Setup non-root user account

## 4. Install window manager

```
pacman -S xf86-video-amdgpu xinit xorg awesome
pacman -S xorg-server xorg-xinit xterm awesome
```

### 4.1. Start awesome on login:

```
vim .xinitrc

exec awesome
```

```
vim .zprofile

if [ -z "${DISPLAY}" ] && [ "${XDG_VTNR}" -eq 1 ]; then
	exec startx
fi
```

### 4.2. Customise awesome

Follow the instructions from [awesome-copycats](https://github.com/lcpz/awesome-copycats).
