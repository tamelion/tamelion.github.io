---
title: Installing Arch linux with dm-crypt and btrfs
tags: [arch, linux, dm-crypt, btrfs]
---
Having recently purchased a new laptop, it was a good opportunity to try out a new filesystem, btrfs. Just to complicate things I wanted to secure my system with whole drive encryption, so today we're throwing dm-crypt (LUKS) into the mix!

## Preparation

### Installation media

First, [download](https://www.archlinux.org/download/) and [write](https://wiki.archlinux.org/index.php/USB_Installation_Media) the installation image.

### Cleaning hard drive

As the drive will be encrypted, we'll want to first fill it with random data. This is so that the encrypted areas are not easily identifiable.

```bash
cryptsetup open --type plain /dev/sda container
dd if=/dev/zero of=/dev/mapper/container
```

This took about 3hrs on my 500Gb drive

```bash
cryptsetup luksClose container
```

### Partitioning

I'm going to be using a 500Mb [EFI](https://wiki.archlinux.org/index.php/Unified_Extensible_Firmware_Interface#EFI_System_Partition) partition mounted as /boot and the rest of the disk for  [btrfs](https://wiki.archlinux.org/index.php/Btrfs) mounted on a [dm-crypt](https://wiki.archlinux.org/index.php/Dm-crypt) partition. I have plenty of RAM so won't be using a swap partition for this setup (which also neatly avoids any issues with [swap encryption](https://wiki.archlinux.org/index.php/Dm-crypt/Swap_encryption)).

```bash
gdisk /dev/sda
```

* `/dev/sda1` 512Mb for `/boot`
* `/dev/sda2` remainder for btree filesystem

## Filesystem and encryption

### Set up dm-crypt container

I'm encrypting using sha512 instead of the default, sha1. Remember to increase iteration time when doing this, as each iteration will take longer with a stronger hash.

```bash
cryptsetup --hash sha512 --iter-time 2000 --use-random luksFormat /dev/sda2
```

Open the new encrypted container. I'm using the name "btree" which will open the container in `/dev/mapper/btree`.

```bash
cryptsetup luksOpen /dev/sda2 btree
```

### Create filesystem

Create the filesystems on both partitions. I'm labelling my btrfs partition "honey". Bee tree. Get it? Never mind.

```bash
mkfs.fat -F32 /dev/sda1
mkfs.btrfs -L honey /dev/mapper/btree
```

Mount root of btrfs partition for use. I'm using the same [mount options](https://btrfs.wiki.kernel.org/index.php/Mount_options) that I intend to use in fstab.

```bash
mkdir /broot
mount -o autodefrag,compress=lzo,noatime,space_cache /dev/mapper/btree /broot
```

### Set up subvolumes

Enter the root of the btrfs filesystem:

```bash
cd /broot
```

There are no difinitive rules (as yet) around subvolume heirarchy. There seem to be two popular methods:

* using `__active` (or `__current`) and `__snapshots` directories
* using `@` for root subvolume, `@home` for home subvolume, etc. directly in the root of the device

The first way seems to favour using `__active` as the root, so all subvolumes can be created underneath and therefore will be auto-mounted. The issue with this comes when you want to restore a snapshot of `/` (root) -- you need to move all the subvolumes first. The suggestion in the [official wiki](https://btrfs.wiki.kernel.org/index.php/SysadminGuide#Managing_snapshots) is to create both the root filesystem and other subvolumes on the same level, then mount the subvolumes separately in fstab, which is closer to the second method listed above.

I like both approaches: the subdirectories I find neater, but the flat structure allows for better root filesystem restores. I don't see why I can't combine **both** approaches -- so I will! First I'll create directories to hold the active and snapshot subvolumes (these don't need to be subvolumes themselves as I don't intend to mount them anywhere):

```bash
mkdir active snapshots
```

 One thing to note is that when taking snapshots, we probably want to keep some paths **out** of the backup. The Suse documentation discusses each of these [paths and the reason to have them as a separate subvolume](https://www.suse.com/documentation/sles11/stor_admin/data/sec_filesystems_major.html#b15tkr5j). Of course, not all of these will be applicable to everyone and due to the nature of the COW in btrfs, we shouldn't worry too much about paths which aren't modified often (or by much). I've taken a look at the modified dates on these paths and determined that I ought to keep `tmp`, `run`, and `var/log` separate, along with `/home` of course. Arch creates tmpfs mounts for `/run` and `/tmp` so we don't need to worry about those.

```bash
btrfs subvolume create active/root
btrfs subvolume create active/home
btrfs subvolume create active/log
```

### Mount subvolumes

Mount the root filesystem subvolume:

```bash
mount -o subvol=active/root /dev/mapper/btree /mnt
```

Create mount points and mount the boot partition and other subvolumes:

```bash
mkdir -p /mnt/{boot,home,var/log}
mount /dev/sda1 /mnt/boot
mount -o subvol=active/home /dev/mapper/btree /mnt/home
mount -o subvol=active/log /dev/mapper/btree /mnt/var/log
```

## Install and config

### Base system

```bash
pacstrap /mnt base btrfs-progs
genfstab -p /mnt >> /mnt/etc/fstab
```

Edit fstab if needs be -- in my case I had to change `relatime` to `noatime`.

Now `arch-chroot` into new system and set hostname, time zone, locale, etc.

### Initial RAM disk

Add `encrypt` to the MODULES array in `/etc/mkinitcpio.conf` and create a new initial RAM disk:

```bash
mkinitcpio -p linux
```

### Bootloader

```systemd-boot``` (previously gummiboot) is my bootloader of choice and is already installed with systemd.

```bash
bootctl install
vi /boot/loader/loader.conf
  default  arch
vi /boot/loader/entries/arch.conf
  title          Arch Linux
  linux          /vmlinuz-linux
  initrd         /initramfs-linux.img
  options        cryptdevice=/dev/sda2:btree root=/dev/mapper/btree rootflags=subvol=active/root ro
```

### Restart

Exit chroot, unmount partitions and reboot

```bash
exit
umount -R /mnt
reboot
```
