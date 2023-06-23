# Void Linux Installation Guide

SSD/NVMe, EXT4, swapfile, FDE with **_dedicated_** encrypted /boot partition using LVM+LUKS matryoshka

Tested on void-live-x86_64-20221001.

# Introduction

In fact, this guide is the collective work of many people, based on whose work I began my amazing journey into the Void.

I spent a lot of time testing a scenario with a dedicated encrypted /boot partition.

Well,
«Qui bene distinguit, bene docet».

Enter the Void!
## Current status: 
- _*always*_ beta
## Features:
* UEFI
* SSD or NVME disk
* FDE with dedicated /boot partition
* LVM
* EXT4 /root
* swapfile instead of swap partition
## Notes
Some steps require more investigation, testing and optimization:
- SSD optimization (trim/discard configuration)
- fstab mount options
- SecureBoot steps?
- cryptsetup iter-time value (to find a good balance between security and system startup time)
- resume and resume_offset paramaters (grub, kernel)?  
- ?
## Disk layout:
- ESP / efi partition: 350M - fat32
- boot partition: 1G - EXT4 <-- LUKSv1 container
- root partition: 100%FREESPACE - ext4 <-- LVM <-- LUKSv2 container;
- swapfile: 4G - /swap/swapfile
## Go!
## Table of contents
0. [Introduction] 
    1. [Current status](#current-status)
    2. [Features](#features)
    3. [Notes](#notes)
    4. [Disk layout](#disk-layout)
1. [Initial settings](#initial-settings)
    1. [Live ISO](#live-iso)
        1. [Logging in (locally)](#logging-in-(locally))
        2. [Connecting to the network (wi-fi)](#connecting-to-the-network-(wi-fi))
        3. [Establishing SSH-connection](#establishing-ssh-connection)
	2. [BIOS or UEFI?](#bios-or-uefi)
    3. [Creating file systems and swap]
    	1. [Set environment variables]
        2. [Partitioninig disks]
        3. [Partitions encryption]
           1. [LUKS-encrypted boot]
           2. [LUKS-encrypted root]
           3. [Open encrypted partitions]
	4. [Configuring LVM]
	5. [Formatting partitions]
	6. [Mounting partitions]
	7. [Making swapfile]		
2. [Installing the system]
   	1. [Base installation (XBPS Method)]
    2. [Chroot]
	3. [Basic configuration]
		1. [G(root) password and shell] 
		2. [Hostname]
		3. [Console fonts]
		4. [Locales]
		5. [fstab]
		6. [GRUB]
		7. [Encryption settings]
			1. [GRUB]
			2. [LUKS keys]
			3. [crypttab]
			4. [dracut]
	4. [Finishing installation]
	   	1. [Install bootloader]
    	2. [Autostart services]
		3. [Regenerating configurations]
3. [Post-installation actions]
4. [Refs]
5. [See also]
6. [Bonus №1]
## Initial settings
## Live ISO
### Logging in (locally)
There are two available users, `root` (superuser) and `anon`. The password of both is `voidlinux`. I often deal with virtual machines and prefer to manage them via SSH.

At this stage, it does not make much sense to connect to a live system locally if the LAN is working correctly (by ethernet, for example). If not, then you need to set up a network connection locally. The installation method used in this guide requires internet access.
### Connecting to the network (wi-fi)
```bash
cp /etc/wpa_supplicant/wpa_supplicant.conf /etc/wpa_supplicant/wpa_supplicant-<wlan-interface>.conf
wpa_passphrase <ssid> <passphrase> >> /etc/wpa_supplicant/wpa_supplicant-<wlan-interface>.conf
sv restart dhcpcd
ip link set up <interface>
```
### Establishing SSH-connection
Initiate SSH-session to the live instance with anon/voidlinux credentials from your prefered SSH-client.
```console
ssh anon@IP
sudo -s
```

### BIOS or UEFI?
To determine the platform, run:
```bash
mount | grep efivars
  efivarfs on /sys/firmware/efi/efivars type efivarfs (rw,nosuid,nodev,noexec,relatime)
  # it is UEFI.
```

## Creating file systems and swap
### Set environment variables
Check disk device (sdX, nvmeXnY) first.

_SATA device_
```console
lsblk
  NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS 
  loop0     7:0    0 849.1M  1 loop
  loop1     7:1    0     4G  1 loop /run/rootfsbase
  sda       8:0    0    20G  0 disk
  sr0      11:0    1   983M  0 rom  /run/initramfs/live
```
_NVME device:_
```console
lsblk 
  NAME    MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
  loop0     7:0    0 849.1M  1 loop
  loop1     7:1    0     4G  1 loop /run/rootfsbase
  sr0      11:0    1   983M  0 rom  /run/initramfs/live
  nvme0n1 259:0    0    20G  0 disk
```

Set variables:
```bash
# for SATA
export DEV="/dev/sda"
# OR for NVME
export DEV="/dev/nvme0n1"
export DM="${DEV##*/}"
export VGNAME="vg_void"
# I use swapfile instead of swap partition but it really can be used if needed
#export LVSWAPNAME="lv_swap" 
export LVROOTNAME="lv_root"
export DEVP="${DEV}$( if [[ "$DEV" =~ "nvme" ]]; then echo "p"; fi )"
export DM="${DM}$( if [[ "$DM" =~ "nvme" ]]; then echo "p"; fi )"
```
### Partitioninig disks
Let's create partitions with cfdisk
```bash
cfdisk "${DEV}"
  select label type: gpt
  Free space
  [NEW]
  Partition size: 350M #size safely can be 100M+; i prefer 350M personnaly;
  Type: EFI System
  Free space
  [New]
  Partition size: 1G
  Free space
  New
  Partition size: 18.7G (100% free space)
  [Write]
  yes
  [Quit]
```
Check final scheme.

_SATA device:_
```console
lsblk -l

  NAME  MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
  loop0   7:0    0 849.1M  1 loop
  loop1   7:1    0     4G  1 loop /run/rootfsbase
  sda     8:0    0    20G  0 disk
  sda1    8:1    0   350M  0 part
  sda2    8:2    0     1G  0 part
  sda3    8:3    0  18.7G  0 part
  sr0    11:0    1   983M  0 rom  /run/initramfs/live
```
_NVME device:_
```console
lsblk -l
  NAME      MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
  loop0       7:0    0 849.1M  1 loop
  loop1       7:1    0     4G  1 loop /run/rootfsbase
  sr0        11:0    1   983M  0 rom  /run/initramfs/live
  nvme0n1   259:0    0    20G  0 disk
  nvme0n1p1 259:4    0   350M  0 part
  nvme0n1p2 259:5    0     1G  0 part
  nvme0n1p3 259:6    0  18.7G  0 part
```
### Partitions encryption
#### LUKS-encrypted boot
GRUB only supports opening LUKSv1
>Note: Passphrase iteration count is based on time and hence security level depends on CPU power of the system the LUKS container is created on. Depending on security requirements, this may need adjustment.  For LUKS1, you can just look at the iteration count on different systems and select one you like.  You can also change the benchmark time with the -i parameter to create a header for a slower system.
See: https://gitlab.com/cryptsetup/cryptsetup/-/wikis/FrequentlyAskedQuestions#2-setup

```bash
cryptsetup luksFormat --type=luks1 -i 500 "${DEVP}2" #-i 1000? More value - longer loading!
  WARNING!
  ========
  This will overwrite data on /dev/sda2 irrevocably.

  Are you sure? (Type 'yes' in capital letters): YES
  Enter passphrase for /dev/sda2:
  Verify passphrase:
```

> Some games around LUKSv2 container for /boot partition. Not working for grub v2.06 but theoretically can... Use LUKSv1 container for /boot. 
```console
#cryptsetup luksFormat --type=luks2 --pbkdf pbkdf2 --key-size 512 ${DEV}2
```
#### LUKS-encrypted root
```console
cryptsetup luksFormat --type luks2 "${DEVP}3"
   =same here=
```

#### Open encrypted partitions
Open both LUKS containers now and check results:
```bash
cryptsetup open ${DEVP}2 boot_crypt
  Enter passphrase for /dev/sda2:
  #Enter passphrase for /dev/nvme0n1p2:

cryptsetup open ${DEVP}3 ${DM}3_crypt
  Enter passphrase for /dev/sda3:
  #Enter passphrase for /dev/nvme0n1p3:
```
_SATA device:_
```console
ls /dev/mapper/
  boot_crypt  control  sda3_crypt
```
_NVME device:_
```console
ls /dev/mapper/
  boot_crypt  control  nvme0n1p3_crypt
```

### Configuring LVM
Configuring LVM layer on top of LUKSv2; it adds some complexity but i use VOID inside VM too.
```bash
pvcreate /dev/mapper/${DM}3_crypt
  #sata device:
  Physical volume "/dev/mapper/sda3_crypt" successfully created.
  #nvme device:
  #Physical volume "/dev/mapper/nvme0n1p3_crypt" successfully created.
```
```bash
vgcreate "${VGNAME}" /dev/mapper/${DM}3_crypt
  Volume group "vg_void" successfully created
```
```bash
#lvcreate -L 4G -n "${LVSWAPNAME}" "${VGNAME}" #in case of swap partition. 
lvcreate -l 100%FREE -n "${LVROOTNAME}" "${VGNAME}"
  Logical volume "lv_root" created.
```
Check final disk layout with encrypted partitions and LVM:
```console
lsblk
  #SATA device
  NAME                  MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
  loop0                   7:0    0 849.1M  1 loop
  loop1                   7:1    0     4G  1 loop  /run/rootfsbase
  sda                     8:0    0    20G  0 disk
  ├─sda1                  8:1    0   350M  0 part
  ├─sda2                  8:2    0     1G  0 part
  │ └─boot_crypt        254:0    0  1022M  0 crypt
  └─sda3                  8:3    0  18.7G  0 part
    └─sda3_crypt        254:1    0  18.6G  0 crypt
      └─vg_void-lv_root 254:2    0  18.6G  0 lvm
  sr0                    11:0    1   983M  0 rom   /run/initramfs/live

  #NVME device
  NAME                  MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINTS
  loop0                   7:0    0 849.1M  1 loop
  loop1                   7:1    0     4G  1 loop  /run/rootfsbase
  sr0                    11:0    1   983M  0 rom   /run/initramfs/live
  nvme0n1               259:0    0    20G  0 disk
  ├─nvme0n1p1           259:4    0   350M  0 part
  ├─nvme0n1p2           259:5    0     1G  0 part
  │ └─boot_crypt        254:0    0  1022M  0 crypt
  └─nvme0n1p3           259:6    0  18.7G  0 part
    └─nvme0n1p3_crypt   254:1    0  18.6G  0 crypt
      └─vg_void-lv_root 254:2    0  18.6G  0 lvm
```
### Formatting partitions
ESP (efi) partition - FAT32:
```console
mkfs.vfat -nESP -F32 ${DEVP}1
```
boot patition - EXT4
```console
mkfs.ext4 -L boot /dev/mapper/boot_crypt
```
#in case of dedicated swap partition:
#mkswap /dev/mapper/${VGNAME}-${LVSWAPNAME}

root partition - EXT4
```console
mkfs.ext4 -L root /dev/mapper/${VGNAME}-${LVROOTNAME}
```
### Mounting partitions
```console
mount /dev/mapper/${VGNAME}-${LVROOTNAME} /mnt
mkdir /mnt/boot/
mount /dev/mapper/boot_crypt /mnt/boot/
mkdir /mnt/boot/efi
mount ${DEVP}1 /mnt/boot/efi/
```
### Making swapfile
```console
mkdir /mnt/swap/
touch /mnt/swap/swapfile
chmod 600 /mnt/swap/swapfile
dd if=/dev/zero of=/mnt/swap/swapfile bs=1M count=4096 #4G swapfile
#dd if=/dev/zero of=/mnt/swap/swapfile bs=1M count=8192 #8G swapfile
#dd if=/dev/zero of=/mnt/swap/swapfile bs=1M count=16384 #16G swapfile
mkswap /mnt/swap/swapfile
swapon /mnt/swap/swapfile
```
## Installing the system
### Base installation (XBPS Method)
```bash
REPO=https://alpha.de.repo.voidlinux.org/current
ARCH=x86_64
XBPS_ARCH=$ARCH xbps-install -S -r /mnt -R "$REPO" base-system btrfs-progs xfsprogs cryptsetup grub-x86_64-efi lvm2 nano
  Do you want to import this public key? [Y/n] y
  
  Size to download:              342MB
  Size required on disk:         869MB
  Space available on disk:        14GB
  
  Do you want to continue? [Y/n] y
```
### Chroot
Copy the DNS configuration into the new root so that XBPS can still download new packages inside the chroot:
```console
cp /etc/resolv.conf /mnt/etc/
```
_Entering the chroot_
```bash
mount --rbind /sys /mnt/sys && mount --make-rslave /mnt/sys
mount --rbind /dev /mnt/dev && mount --make-rslave /mnt/dev
mount --rbind /proc /mnt/proc && mount --make-rslave /mnt/proc
```
```bash
# Chroot into the new installation:
PS1='(chroot) # ' chroot /mnt/ /bin/bash
```
> Note: Some old stuff. Just rock art, let it be...

#chown root:root /

#chmod 755 /

### Basic configuration
#### G(root) password and shell
> I AM GROOT:
```bash
(chroot) #
passwd root
chsh -s /bin/bash
```
#### Hostname
Set hostname of new Void instance:
```console
echo void-machine > /etc/hostname
```
#### Console fonts
Install console font
> Note: console fonts is stored in /usr/share/kbd/consolefonts. You can test the font like this: `setfont ter-c16n`
```console
xbps-install -Suy terminus-font
echo KEYMAP=ru >> /etc/rc.conf # '=us' '=de' and so on
echo FONT=ter-c16n >> /etc/rc.conf
```
#### Locales
Set system locales - /etc/default/libc-locales
```bash
sed -i 's/#en_US.UTF-8/en_US.UTF-8/g' /etc/default/libc-locales
sed -i 's/#ru_RU.UTF-8/ru_RU.UTF-8/g' /etc/default/libc-locales
xbps-reconfigure -f glibc-locales
  Generating GNU libc locales...
    en_US.UTF-8... done.
    ru_RU.UTF-8... done.
  glibc-locales: configured successfully.
```
#### fstab
/etc/fstab
```bash
export UEFI_UUID=$(blkid -s UUID -o value blkid -s UUID -o value "${DEVP}1")
export BOOT_UUID=$(blkid -s UUID -o value /dev/mapper/boot_crypt)
export ROOT_UUID=$(blkid -s UUID -o value /dev/mapper/${VGNAME}-${LVROOTNAME})
#export SWAP_UUID=$(blkid -s UUID -o value /dev/mapper/${VGNAME}-${LVSWAPNAME})

cat <<EOF > /etc/fstab

UUID=$ROOT_UUID 		/ ext4 defaults,discard           0 1 
UUID=$BOOT_UUID			/boot ext4 defaults,discard          0 1 
UUID=$UEFI_UUID 		/boot/efi vfat defaults,noatime 0 2
#UUID=$SWAP_UUID none swap defaults 0 1
/swap/swapfile none swap defaults 0 0

tmpfs /tmp tmpfs defaults,noatime,mode=1777 0 0
EOF
```
#### LVM.conf
/etc/lvm/lvm.conf

enable discard on LVM-level too; check dmesg for errors and warnings!
```console
sed -i 's/issue_discards = 0/issue_discards = 1/g' /etc/lvm/lvm.conf
```
#### Encryption settings
##### GRUB
/etc/default/grub
> Note: Only the first two are required options. The rest will make the high resolution screen easy to read.
```console
cat <<EOF >> /etc/default/grub
GRUB_ENABLE_CRYPTODISK=y
GRUB_PRELOAD_MODULES="part_gpt part_msdos cryptodisk luks lvm" #btrfs?
GRUB_GFXPAYLOAD=keep
GRUB_TERMINAL=gfxterm
GRUB_GFXMODE=1920x1080x32

EOF

export LUKS_UUID=$(blkid -s UUID -o value /dev/mapper/"${DM}3_crypt")
sed -i "/GRUB_CMDLINE_LINUX_DEFAULT=/s/\"$/ rd.auto=1 cryptdevice=UUID=$LUKS_UUID:lvm:allow-discards&/" /etc/default/grub
```

##### LUKS Keyfile
> Note: the values from https://docs.voidlinux.org/installation/guides/fde.html#luks-key-setup
> 
> #dd bs=1 count=64 if=/dev/urandom of=/boot/volume.key
> 
> I use instead of official values others:

Create a randomised key-file of 4096 bits (512 bytes), secure it, and add it to the LUKS volumes (Man-pages for dd chmod):
```bash
dd bs=512 count=1 if=/dev/urandom of=/boot/volume.key
cryptsetup luksAddKey ${DEVP}2 /boot/volume.key
  Enter any existing passphrase:
cryptsetup luksAddKey ${DEVP}3 /boot/volume.key
  Enter any existing passphrase:

chmod 000 /boot/volume.key
chmod -R g-rwx,o-rwx /boot
```
##### crypttab
/etc/crypttab
```console
echo "boot_crypt UUID=$(blkid -s UUID -o value ${DEVP}2) /boot/volume.key luks,discard" >> /etc/crypttab
echo "${DM}3_crypt UUID=$(blkid -s UUID -o value ${DEVP}3) /boot/volume.key luks,discard" >> /etc/crypttab
```

##### dracut
```bash
cat <<EOF >> /etc/dracut.conf.d/10-crypt.conf
install_items+=" /boot/volume.key /etc/crypttab "
EOF

echo 'add_dracutmodules+=" crypt lvm resume "' >> /etc/dracut.conf
echo 'tmpdir=/tmp' >> /etc/dracut.conf

dracut --force --hostonly --kver 6.1.31_1
  dracut: Cannot find module directory /lib/modules/6.1.31_1/
  dracut: and --no-kernel was not specified
#OK. Check current version:
  (chroot) # ls /lib/modules/
6.1.34_1
dracut --force --hostonly --kver 6.1.34_1
```

#### Finishing installation
##### Install bootloader
```console
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=void --boot-directory=/boot  --recheck
```
##### Autostart services
Set DHCP client to autostart to get IP address from network after reboot
```console
ln -s /etc/sv/dhcpcd /var/service
```
Set OpenSSH Server to autostart and accept connections for root user (unsecure!)
```console
echo "PermitRootLogin yes" >> /etc/ssh/sshd_config
ln -s /etc/sv/sshd /var/service
```
##### Regenerating configurations
```console
(chroot) # xbps-reconfigure -fa
(chroot) # exit
umount -R /mnt
shutdown -r now
```
After reboot GRUB should ask for a password like this:

![Alt text](/images/Welcome_to_GRUB2.JPG?raw=true "GRUB ask pass")

## Refs:
This guide is a compilation and adaptation of data from many sources, including the following:

[-1] https://help.ubuntu.com/community/Full_Disk_Encryption_Howto_2019

[0] https://docs.voidlinux.org/installation/guides/fde.html

[1] https://gist.github.com/tobi-wan-kenobi/bff3af81eac27e210e1dc88ba660596e

[2] https://gist.github.com/gbrlsnchs/9c9dc55cd0beb26e141ee3ea59f26e21

[3] https://habr.com/ru/articles/497004/

[4] https://wiki.archlinux.org/title/dm-crypt/Encrypting_an_entire_system

Looks like the list of sources will be updated....

## See also

## Bonus №1
```bash
# Make grub great again!
sudo -s
xbps-install -Suy wget
wget -P /tmp https://github.com/shvchk/fallout-grub-theme/raw/master/install.sh
chmod +x /tmp/install.sh
/tmp/install.sh
reboot
```
![Alt text](/images/GRUB_Splash.PNG?raw=true "GRUB fallout theme")
