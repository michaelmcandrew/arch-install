# Michael's Arch Linux installation guide for the ThinkPad x220

Arch Linux does not have an automated installer. Instead it has a comprehensive  installation guide, available at https://wiki.archlinux.org/index.php/installation_guide. That guide is fantastic, but 'too much information' to serve as a reference for how I got Arch Linux up and running on my ThinkPad. This guide documents the specific path I took through the installation. I'm writing it as a reference for the re-installation of my machine if that should become necessary, and also as a good starting point for an Arch Linux installation that meets my needs on other machines. It is also a learning exercise in Arch Linux and some 'low level' Linux topics.

My first attempt at systematising the installation was to write an installation script. This worked OK but not brilliantly and when I came to run the script again a few months later, it didn't complete. It was not very thoroughly documented and it took a while for me to work out  what the issues were. This, along with the fact that writing an install script is not a trivial task, lead my to try another approach. Hence this guide, completed in July 2017.

If you're reading this guide much later than July 2017, things are likely to have moved on, and you may want to consult the Arch Wiki for more up to date techniques and components, etc.

It's worth remembering that, although it is a manual process, the tasks we are completing in this installation are not very different from other operating system installations. In breif, we are:

* Partitioning (and in this instance, encrypting) the disk(s)
* Choosing regional settings (e.g. keyboard layout and timezone)
* Downloading and installing packages
* Configuring a boot loader

In the steps below, I've tried to follow the latest recommendations, to the extent that the hardware I am installing on will allow it (GPT, UEFI, systemd-boot, etc.)

## Installation media

Download an installation image and and create the installation media.

A recent installation image can be downloaded from  https://www.archlinux.org/download/.

You can create a bootable USB drive as follows:

1. insert the USB drive and note the device name
2. run `dd if=archlinux.img of=/dev/sdX bs=16M && sync` where X is the device letter

See https://wiki.archlinux.org/index.php/USB_flash_installation_media for more details.

## Booting the installation media

Ensure that you are booting in UEFI mode and that you can boot from USB. Insert the installion media into the ThinkPad and boot. Hit F12 while the machine is booting to bring up a menu that will allow you to boot from the USB drive. Select the USB drive to boot the install image.

The ThinkPad should boot up and log you in as the root user.

## Pre-installation

There are a few things to do before installing the base system.

### Set the keyboard

Set up the keyboard so that our keys work as expected. On my UK ThinkPad, this is as simple as `loadkeys uk`.

### Connect to the internet

Connect to the internet so you can download packages as part of this install.

The Arch installer uses `netctl` ('a CLI-based tool used to configure and manage network connections via profiles'). Any plugged in ethernet connection should connect automatically. You can connect wirelessly by using `wifi-menu`.

If you like, you can create a new connection profile manually based on one of the examples in /etc/netctl/examples. For more information, try `man netctl` or the [netctl](https://wiki.archlinux.org/index.php/Netctl) page on the Arch Linux wiki.

At this point, it probably makes sense to test network connectivity with a ping to google.com or similar.

### Update the system clock

Once you are connected to the internet, synchronise to network time with the following command: `timedatectl set-ntp true`

## Partition the disks

I have a 128GB SSD and want to keep things simple with as few partitions as possible. One partition would be ideal but an encrypted UEFI system will require at least two.

Create two partitions as follows:

* a large encrypted partition mounted at / (the capacity of the machine - 500M)
* a 500M unencrypted partition for UEFI booting at /boot

Use gdisk to create a GPT partition table and three partitions as follows:

| Partition            | Size  | Hex code | Device name |
|----------------------|-------|----------|-------------|
| Encrypted root       | -500M | 8300     | /dev/sda1   |
| EFI system partition | +500M | EF00     | /dev/sda2   |

Start gdisk with `gdisk /dev/sda` and delete the existing partition table with `o`.

Add the new partitions specified in the table above with the `n` command (once for each partition).

Write the table to disk with `w` and exit.

Note that the values in the *Size* and *Hex code* columns are written as they should be passed to gdisk (size should be passed as *Last sector*). Also note: passing a negative value for  *Last sector* will ensures that amount of space is left on the disk after the partition.

### Encrypt the root partition

Use `cryptsetup` to create an encrypted disk on /dev/sda1

`cryptsetup -y -v luksFormat /dev/sda1`

Open the encrypted drive so it is available at `/dev/mapper/cryptroot` with `cryptsetup open /dev/sda3 cryptroot`.

For more information on encryption, see https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Simple_partition_layout_with_LUKS.

### Format the drives

Format /dev/mapper/cryptroot and as ext4.

`mkfs.ext4 /dev/mapper/cryptroot`

Format /dev/sd2 as FAT32.

`mkfs.fat -F32 /dev/sda2`

### Mount the partitions

Mount the partitions as follows (we need to mount /mnt before mounting /mnt/boot as we need to create the mount point before mounting).

`mount /dev/mapper/cryptroot /mnt`
`mkdir /mnt/boot`
`mount /dev/sda2 /mnt/boot`

## Installation

The pacstrap script is designed for the initial installation of packages to a disk. It also takes care of creating directories like /etc (and more stuff as well - see https://git.archlinux.org/arch-install-scripts.git/tree/pacstrap.in or type `pacstrap` with no parameters for more information.

Run `pacstrap /mnt base` to create an initial base system.

Create an fstab file based on the above mounts as follows:

`genfstab -U /mnt >> /mnt/etc/fstab`

chroot into /mnt to continue the installation with the installers special chroot script `arch-chroot`

`arch-chroot /mnt`

Do some basic configuration of locales and timezones, etc.

Note that a couple of the following commands (those to do with the keyboard and time) repeat what we did in 'Pre-installation'. We repeat them here so tha they persist to the drives we mounted and hence the installed system.

`ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime`
`hwclock --systohc`
`echo "LANG=en_GB.UTF-8" > /etc/locale.conf`
`echo "KEYMAP=uk" > /etc/vconsole.conf`
`echo "thinkpad" > /etc/hostname`
`echo "127.0.1.1	thinkpad.localdomain	thinkpad" >> /etc/hosts`

Uncomment `en_US.UTF-8 UTF-8` and `en_GB.UTF-8 UTF-8` in `/etc/locale.gen`

Generate the locales with `locale-gen`.

### Microcode updates

We need to install microcode updates. See https://wiki.archlinux.org/index.php/microcode for more information.

`pacman -S intel-ucode`

### Wireless networking

`pacman -S iw wpa_supplicant` is necessary if we will only be able to connect wirelessly after booting up the installed system. It's probably worth installing at this point in time in any case, if you need to connect wirelessly.

TODO: ensure that I need to carry out this step

## Recreate initramfs

We need to recreate initramfs to work with the encrypted filesystem.

First, edit `/etc/mkinitcpio.conf` and ensure that the following three words (`keyboard`, `keymap`, and `encrypt`) are added to the `HOOKS=...` line if they are not already present.

Read the comments in the `/etc/mkinitcpio.conf` file and https://wiki.archlinux.org/index.php/Mkinitcpio for more information.

Then recreate the initramfs file with `mkinitcpio -p linux`.

## Boot loader

Configure the UEFI bootloader.

If you are interested in how UEFI works, then this is a great resource: https://www.happyassassin.net/2014/01/25/uefi-boot-how-does-that-actually-work-then/.

Install the systemd-bootloader with `bootctl --path=/boot install`. This copies the systemd-boot binary to the EFI System Partition here: /boot/EFI/systemd/systemd-bootx64.efi and also here: /boot/EFI/Boot/BOOTX64.EFI (both files are identical). It also adds systemd-boot to the boot loader and sets it as the default.

We now need to configure the bootloader.

Open `/boot/loader/loader.conf` and ensure that it looks like this

```
default arch
timeout 0
editor 0
```

This sets `/boot/efi/loader/entries/arch.conf` as the default entry. Since we only have one entry it makes sense to set the timeout to 0 which will boot it immediatley.

Create a new file `/boot/efi/loader/entries/arch.conf` to contain details of the Arch Linux system we want to boot. The file should look similar to the following example.

```
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options cryptdevice=UUID=284e9b4f-07ed-41eb-8838-84ec79f42f4a:cryptroot root=/dev/mapper/cryptroot
```

* `title` is required. It would show in the boot menu if the timeout wasn't set to 0.
* `linux` points to the linux kernal (relative to the boot partition root).
* `initrd` points to the initial ramdisk (relative to the boot partition root). It is also used to enable Intel microcode updates
* `options` is used to pass options to the kernel

The options merit a little more explanation. We need to add a `cryptdevice` option to give bootloader details of the encrypted drive. The bootloader will then prompt for the password of the encrypted drive at startup. `root` tells the kernal which device to mount as the root filesystem (the name is derived from the device mapper specified in cryptdevice: /dev/mapper/cryptroot).

Note: the UUID passed to cryptdevice is the UUID of the **parent** device that contains the encrypted device, not the UUID of the encrypted device itself. i.e in this case, it is the UUID of `/dev/sda1`, not of `/dev/mapper/cryptroot`.

You can find the UUID of the encrypted device with `lsblk -f` as long as it has been opened.

## Configure user accounts

Configure a password for root with `passwd`.

Create a user for michael `useradd -m -G wheel michael`.

Let users in the wheel group do passwordless sudo by commenting out the appropriate line in  `/etc/sudoers`.

### Reboot

Exit the chroot environment by typing `exit`.

Reboot the system by typing `reboot`

## Post installation configuration

See https://wiki.archlinux.org/index.php/General_recommendations for a recommended list of things to do once Arch is installed.
and https://wiki.archlinux.org/index.php/List_of_applications for some ideas on packages that you should install

```
atom
chromium
firefox
git
gnome
iw
sudo
wpa_supplicant
zsh
zsh-completions
```

is a random sample of packaes that I would recommend.

# TODO

* Check back against official install guide to see if I have missed anything
* reinstall using this guide
* Work out if I should switch from netctl to systemd-networkd
