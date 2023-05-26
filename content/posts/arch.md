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

## Motivation
I've been using Ubuntu as my daily driver for around one year. While there was no issue whatsoever, I feel that Ubuntu is quite bloated; there are lots of pre-installed packages which I don't really use. I want to have a fast and clean OS, but it's kinda annoying to manually clean the packages one by one. Some things might also break in the process, which is not ideal at all since I don't have a backup laptop.

Then, my laptop actually broke (lol) and I decided to buy a new laptop. I find that this is a good opportunity to make a switch, and I decided to give Arch a try.

This post is mainly a tutorial on how to install Arch from scratch (Part 1 and 2). At the end, I will share a snippet of my attempt to configure everything by hand (Part 3), and give some thoughts about using Arch as my daily driver.

# Part 1: Preparation
## 1. Allocating space to install Arch
Before we can install Arch, we need to allocate some space on the disk. My laptop is pre-installed with Windows, so the first thing to do is to use the Windows 
Disk Utility and shrink the current disk partition. It's GUI-based and quite intuitive to use, so I won't cover the details here. From my 500GB disk, I allocated around 300GB for Windows and 200GB for Arch.

## 2. Downloading the image and create an installation media
After we partitioned our disk, we need to download the base image and create an installation media.

### 2.1. Download
First, we download `archlinux-x86_64.iso` and `archlinux-x86_64.iso.sig`. The first file is the actual Arch image, and the second one is simply the signature to ensure that we downloaded the correct image.

You can go to `https://archlinux.org/download/` and find the appropriate server nearest to your location to download the image and signature. I used NUS' mirror at `https://download.nus.edu.sg/mirror/archlinux/iso/`.

You can download the files directly by clicking the link directly or by using `wget`:
```
ubuntu$ wget https://download.nus.edu.sg/mirror/archlinux/iso/latest/archlinux-x86_64.iso
ubuntu$ wget https://download.nus.edu.sg/mirror/archlinux/iso/latest/archlinux-x86_64.iso.sig
```

After the download has finished, verify the image using the signature.

```
ubuntu$ gpg --keyserver-options auto-key-retrieve --verify archlinux-x86_64.iso.sig
```

Confirm that the key fingerprint is shown at the the Developer page of arch.

### 2.2. Create media
After we have our Arch image ready, we need to prepare an empty USB to be used as an installation media. 

> note: I used my Ubuntu machine to do this step; you might want to see the instructions on how to create bootable USB on other platforms on Arch's official website.

After inserting the USB to the computer, we can get the USB device name by invoking `lsblk`.

After we get the device name, we simply copy the image to the USB. This can be done by either `cat` or `dd`.
```
ubuntu$ sudo -i

ubuntu# lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda           8:0    1    64M  0 disk
nvme0n1     259:0    0 476.9G  0 disk
├─nvme0n1p1 259:1    0   260M  0 part /boot
├─nvme0n1p2 259:2    0    16M  0 part
├─nvme0n1p3 259:3    0 287.2G  0 part
├─nvme0n1p4 259:4    0     2G  0 part
├─nvme0n1p5 259:5    0     8G  0 part [SWAP]
└─nvme0n1p6 259:6    0 179.5G  0 part /

# option 1
ubuntu# cat /home/adhy/Downloads/archlinux-x86_64.iso > /dev/sda; sync

# option 2
ubuntu# dd if=/home/adhy/Downloads/archlinux-x86_64.iso of=/dev/sda bs=512k
```

## 3. Booting using the USB
After the installation media is ready, we can start booting into Arch. To boot using the USB, we need to go to UEFI and change the boot priority for USB devices. You might also need to disable the secure boot to allow UEFI to load Arch for the first time. On my ThinkPad, the key to go to UEFI is `F1`. This varies depending on your computer vendor.

## 4. Network and Disk Setup
Once we successfully boot into Arch, the first thing to do is to connect to the internet and download the required packages.

### 4.1. Setting up network
I spent a good 2-3 hours just to connect to the WiFi. It was quite hard to find the documentation on how to connect to my school network. In the end, I used the `iwd` service and edited the `wpa_supplicant.conf` file directly to connect to the network.

I am sure this can be done using `nmcli`, but this is what I did when I first installed Arch.

```
# Edit the connection configuration file
root# vim /etc/wpa_supplicant/wpa_supplicant.conf

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
# Load the config file and restart the service
root# wpa_supplicant -B -i wlan0 /etc/wpa_supplicant/wpa_supplicant.conf
root# systemctl restart iwd.service
```

### 4.2. Update system clock
By right, `systemd-timesyncd` is enabled by default and the time will be synced automatically when a connection to the internet is established. Use `timedatectl` to ensure that the time is accurate.
```
root# timedatectl status
        ... outputs omitted ....
System clock synchronized: yes
              NTP service: active
        ... outputs omitted ....
```

### 4.3. Partition the disks

After that, we need to partition the disks. You can create as many partitions you want, but I just created one partition for root and one swap partition. 

I used `fdisk` to create the partitions, but other tools such as `parted` can be used as well. I allocated 8GB of swap space and allocated the rest of the disk for the root partition. 

> Note: If you are not familiar, the swap space is a location in your disk that can be used as memory (additional RAM, free real estate!). However, it is very very slow and will only be used when your RAM is full. In other words, if you are seeing that your swap space is constantly being used, maybe it is a good time to upgrade your RAM.

```
root# lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
nvme0n1     259:0    0 476.9G  0 disk             # the disk we want to partition
├─nvme0n1p1 259:1    0   260M  0 part /boot
├─nvme0n1p2 259:2    0    16M  0 part
├─nvme0n1p3 259:3    0 287.2G  0 part
└─nvme0n1p4 259:4    0     2G  0 part

root# fdisk /dev/nvme0n1
```

Inside `fdisk`:

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

### 4.4. Format the partitions
After we have formatted the disks, we need to create a file system. We use `mkfs` to create the file system of our root storage and `mkswap` to initialize our swap partition.

I used `ext4` which is the standard for Linux, but other formats (e.g. `XFS`) can be used as well.
```
root# mkswap /dev/nvme0n1p5
root# mkfs.ext4 /dev/nvme0n1p6 
```

### 4.5. Mount the partitions
At this point, our storage is ready for use. We mount the disk on the `/mnt` folder of the live environment by using `mount`.
```
root# mount /dev/nvme0n1p6 /mnt
root# mount --mkdir /dev/nvme0n1p1 /mnt/boot
root# swapon /dev/nvme1n1p5
```

# Part 2: Arch Installation
This is the part where we actually install Arch on our computer.

## 1. Install essential packages
First, we need to download the essential programs and packages for our computer to run. We use the magical `pacstrap` command to automatically install the packages we specified.

If you want a very basic installation, you can type:

```
root# pacstrap -K /mnt base linux linux-firmware
```

Alternatively, at this stage, we can also install more packages. We simply need to specify the name of the packages to `pacstrap`.

```
root# pacstrap -K /mnt base linux linux-firmware networkmanager \
    vim man-db man-pages texinfo grub efibootmgr
```

At this point, Arch and its related packages are already installed on the disk. However, the disks is not permanently mounted yet during boot up. We need to create `/etc/fstab` file to tell the OS which disks should be mounted persistently. We can create the `fstab` file manually, but in this case we will simply use the `genfstab` command.
```
root# genfstab -U /mnt >> /mnt/etc/fstab
```

# 2. Basic system configuration
Now, we need to configure our system. These include setting the location and the time zone, as well as creating a user account and password.

Recall that we are now still in the live environment. If we make changes here, nothing will be committed to the disk. We need to first change the root directory to our root partition, which was mounted at `/mnt`.
```
root# arch-chroot /mnt
```

After that, we can start to configure our system. We start by selecting the location, language, and timezone.
```
root# ln -sf /usr/share/zoneinfo/Singapore /etc/localtime
root# hwclock --systohc

root# vim /etc/locale.gen # uncomment en_US.UTF-8 UTF-8 from /etc/locale.gen
root# locale-gen

root# echo "LANG=en_US.UTF-8" >> /etc/locale.conf
```
Then, we give a name to our machine and set the administrator password.
```
root# echo "thinkpad" > /etc/hostname
root# passwd  # set up root password
```

# 3. Setup boot loader
At this point, the OS is already installed and ready to use. However, our machine doesn't know how to load and run the OS. We need to configure (or in this case, install) boot loader to load the run Linux when the computer is powered on. I'm not sure whether the default Windows boot loader supports Linux, but just to be safe, we shall install GRUB.
```
root# pacman -S grub efibootmgr os-prober
root# grub-install --target=x86_64-efi \
        --efi-directory=/boot --bootloader-id=GRUB
root# os-prober              # use this to allow GRUB to detect Windows
root# vim /etc/default/grub  # uncomment GRUB_DISABLE_OS_PROBER=false
root# grub-mkconfig -o /boot/grub/grub.cfg
```

We are now good to go. Reboot the system and Arch should be ready to use.

# Part 3: Configuring the system
Now, we have a fully working system. However, it is not very usable. 
The first thing I noticed is that everything is text-based and I can't even run a browser to surf the web. Let's start to improve our experience by installing a window manager.

## 1. Connecting to internet

As usual, we need to connect to internet before we can download and install new packages for our system. In this section, we shall use `nmcli` to connect to internet.

```
thinkpad# nmcli device wifi list
thinkpad# nmcli con add type wifi ifname wlp1s0 con-name NUS ssid NUS_STU
thinkpad# nmcli con edit id NUS

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

## 2. Setting up (reinstalling) pacman
After we can connect to the internet, we need to download the relevant software. 
For some reason, the pacman (package manager) keyring (which is used to verify the packages downloaded) were broken. The pacman keyring needs to be reinstalled:

```
thinkpad# vim /etc/pacman.conf

# do not check the packages
SigLevel = Never
```

After changing the config file, reinstall the keys:

```
thinkpad# pacman -S archlinux-keyring
thinkpad# pacman-key --init
thinkpad# pacman-key --refresh-keys
```

After that, return the config file to the default settings

```
thinkpad# vim /etc/pacman.conf
SigLevel =
```

## 3. Install window manager

Now, we can start to install our GUI. 
```
thinkpad# pacman -S xf86-video-amdgpu xinit xorg 
thinkpad# pacman -S xorg-server xorg-xinit xterm awesome
```

### 3.1. Start awesome on login:
We have a window manager, but every time we restart our system, we need to manually start the X11 server. There must be a way to automate this process.

```
thinkpad# vim .xinitrc

exec awesome
```

```
thinkpad# vim .zprofile

if [ -z "${DISPLAY}" ] && [ "${XDG_VTNR}" -eq 1 ]; then
	exec startx
fi
```

### 3.2. Customize awesome

At this point, we have a window manager, but it's not the most aesthetically pleasing. I tried to configure the settings by myself, but in the end I followed the instructions from [awesome-copycats](https://github.com/lcpz/awesome-copycats).

Having installed a window manager, we can now surf the web and do some productive work. However, there are still lots of things that we can do as our system is still far from being usable and secure. We can set a non-root user account, configure power management, change font and display, and the list goes on. Putting everything in this post might not be the best idea, so let's wrap up here.

# Closing remarks
When I first started using Arch, I tried to not install a desktop environment and setup everything by myself. I started by installing a window manager (awesome-wm), but then I realized that you can't still do much with this. The audio doesn't work, the screen brightness is fixed, you need to manually detect and switch to external monitor, and the list goes on.

I learnt a lot in the process of configuring the system. I learnt how to configure the audio driver and added some macros to increase/decrease volume by using keyboard function keys. I learnt about special files that can be written to change screen brightness, as well as how to detect and use external screen using the command line. 
Most importantly, I learnt to appreciate the complexities that is abstracted out by modern operating systems and the effort put by the developers to make modern computers as user-friendly as possible.

After a while, I find myself spending too much time configuring everything by hand. While I really enjoyed it and learnt a lot in the process, I decided to install a desktop environment to manage everything for me and spend my time learning other things which are more practical and impactful for my school and career :)

Last updated: May 2023
