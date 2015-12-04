---
title: Setting up a btrfs subvolume with noCOW
tags: [btrfs, mysql, arch, linux]
---
As touched on in my previous post about [btrfs]({% post_url 2015-09-21-installing-arch-with-dmcrypt-and-btrfs %}), the copy on write (COW) capabilities are very handy. However, as [COW shouldn't be used for large files with small random writes](https://wiki.archlinux.org/index.php/Btrfs#Copy-On-Write) (e.g. databases and VMs), I wanted to turn it off for these things.

Turning COW off for a single directory (e.g. where my VMs are stored) was as simple as `chattr +C`, but what if we want to turn off COW for a whole subvolume? As of today, the [nodatacow option applies to the whole filesystem](https://btrfs.wiki.kernel.org/index.php/Mount_options) and only the options for the first mounted subvolume are taken into account.

Here is the solution I came up with -- I'd be keen to receive feedback.

## Create a directory just for noCOW files
This can be anywhere, but I chose `/var/lib` as this is where my large databases are kept.

```bash
cd /var/lib
sudo mkdir noCOW
# optional, to remind when this dir is not mounted
sudo touch noCOW/UNMOUNTED
```

## Create filesystem
### Mount broot
If it's not already mounted on your system, mount the btrfs root filesystem

```bash
mkdir /broot
mount -o autodefrag,compress=lzo,noatime,space_cache /dev/mapper/btree /broot
```

### Create subvolume

```bash
sudo btrfs subvolume create /broot/active/noCOW
```

## Mount the filesystem

```bash
sudo vim /etc/fstab
  /dev/mapper/btree   	/var/lib/nocow	btrfs     	rw,noatime,compress=lzo,space_cache,autodefrag,subvol=active/nocow	0 0
sudo mount -a
```

## Set attribute
As we're doing this on directories within the subvolume, we don't have the issue of not being able to use attributes on the subvolume itself.

```bash
sudo mkdir noCOW/mysql
sudo chattr +C noCOW/mysql
# check nocow attribute is set
lsattr noCOW
```

## Symlink to "real" location

```bash
sudo ln -s noCOW/mysql .
```
