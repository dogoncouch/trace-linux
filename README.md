# Gentoo for Laptops

![tuxscii.png](./tuxscii.png)

### Version 0.4-beta

- [Introduction](#introduction)
- [Installing Gentoo](#installing-gentoo)
    - [LUKS/LVM](#lukslvm)
    - [Configuration](#configuration)
- [Installing Gnome](#installing-gnome)
- [Post-Gnome](#post-gnome)
    - [@world](#world)
    - [Sets](#sets)
    - [Bluetooth](#bluetooth)
    - [Laptop Sensors](#laptop-sensors)
- [Resources/Thanks](#resourcesthanks)
- [Contributing](#contributing)


## Introduction

### This Repository
This repository contains documentation and configuration resources for installing [Gentoo Linux](https://gentoo.org) on laptops. It is currently geared toward network testing, with plans to use more generic package keywords in the future to. The goal is to have something that is useful for KVM virtualization, network testing, and development.

This file (README.md) is a companion guide to the [Gentoo install documentation](https://wiki.gentoo.org/wiki/Handbook:AMD64#Installing_Gentoo). It can be deployed quickly (about a day of compile time on an i3; it is Gentoo), supports a wide range of laptop hardware, and features sets to quickly install groups of software.

If you are installing using our configuration files, make sure that this file (README.md) and the configuration files are the same version.

### The Kernel
The kernel and initramfs are compiled using genkernel-next. Kernel configuration is largely based on the [Kali Linux](https://www.kali.org/) configuration, which has excellent laptop hardware support. It is configured to use `systemd` as an init system, for compatibility with the Gnome desktop environment.

It supports Docker containers (with the devicemapper storage driver), QEMU/KVM, VirtualBox (currently broken), and has guest support for most virtualization hosts.

Our kernel modules are quite expansive. You can expect a ~12G source tree, with ~2.8G worth of compiled modules in `/lib/modules/`.

### 2 in 1s
This has been tested on at least one 2in1 laptop, and everything works. Laptop hardware support has come a long way since the 1990s, in everything from the kernel to desktop environments.

### Disk Encryption
Full disk and swap encryption are a necessity in a lot of industries. We use LUKS on LVM to encrypt everything except the boot partition.


## Installing Gentoo

### LUKS/LVM
This section can be used to replace a lot of the instructions in the [Preparing the disks section](https://wiki.gentoo.org/wiki/Handbook:AMD64/Installation/Disks) of the gentoo install documentation.

#### Important Note
The `/usr` directory can get quite large when compiling a kernel using the included configuration. 20G will not be enough. The filesystem size recommendations that follow were arrived at through extensive testing.

#### Arch Linux Wiki
Large parts of our dmcrypt and lvm setup were inspired by an [Arch Linux Wiki article](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system). The Arch Linux documentation is an invaluable resource to all Linux users, especially gentoo users.

#### Partitioning
Create a grub partition and boot partition as normal, and create a third partition that takes up the remainder of the disk. We will encrypt this, and create logical volumes on top of it.

#### LUKS/LVM setup
1. Format the partition with LUKS
```
cryptsetup luksFormat /dev/sda3
cryptsetup open /dev/sda3 cryptolvm
```

2. Then create a physical volume and a logical volume group:
```
pvcreate /dev/mapper/cryptolvm
vgcreate MyVG /dev/mapper/cryptolvm
```
3. Then create some LVM partitions:
##### For 250G+ drives:
```
lvcreate -L 8G MyVG -n swap
lvcreate -L 20G MyVG -n root
lvcreate -L 40G MyVG -n usr
lvcreate -L 20G MyVG -n var
lvcreate -l 100%FREE MyVG -n home
```

##### For smaller drives:
```
lvcreate -L 8G MyVG -n swap
lvcreate -l 100%FREE MyVG -n root
```

#### Create/Mount Filesystems
1. Start with swap:
```
mkswap /dev/mapper/MyVG-swap
swapon /dev/mapper/MyVG-swap
```

2. Then create the filesystems:
##### For 250G+ drives:
```
mkfs.ext4 /dev/mapper/MyVG-root
mkfs.ext4 /dev/mapper/MyVG-usr
mkfs.ext4 /dev/mapper/MyVG-var
mkfs.ext4 /dev/mapper/MyVG-home
```

##### For smaller drives:
```
mkfs.ext4 /dev/mapper/MyVG-root
```

3. Then mount everything:
##### For 250G+ drives:
```
mount /dev/mapper/MyVG-root /mnt/gentoo
mkdir /mnt/gentoo/usr
mount /dev/mapper/MyVG-usr /mnt/gentoo/usr
mkdir /mnt/gentoo/var
mount /dev/mapper/MyVG-var /mnt/gentoo/var
mkdir /mnt/gentoo/home
mount /dev/mapper/MyVG-home /mnt/gentoo/home
```

##### For smaller drives:
```
mount /dev/mapper/MyVG-root /mnt/gentoo
```

### Configuration

#### Files
This repository contains configuration files needed for installation in `gentoo-laptop/files/`. You can use them as examples, copy them to `/etc`, or ignore them entirely. If you are going to copy them all, a good time to do it is right after [unpacking the stage tarball](https://wiki.gentoo.org/wiki/Handbook:AMD64/Installation/Stage#Unpacking_the_stage_tarball).

To download this repo, use links:
```
links https://github.com/dogoncouch/gentoo-laptop/releases/latest
```

Navigate to the link labeled `Source code (tar.gz)`, and hit `d` to download. Then hit `q` to quit, and unpack the tarball:
```
tar -xzf v*.tar.gz
```

Then remove the version number from the repo directory, so the instructions in the rest of this guide will work:
```
mv gentoo-laptop* gentoo-laptop
```

To copy all configuration files:
```
cp -ir gentoo-laptop/files/* .
```

#### Profile
We use Gnome for a desktop environment, which requires using the systemd init system. If you don't mind a bit of extra work, you can use something else. That is one of the reasons we like Gentoo.

If you plan on using Gnome and systemd, make sure to select the right profile during the [Configuring Portage section](https://wiki.gentoo.org/wiki/Handbook:AMD64/Installation/Base#Choosing_the_right_profile) of installation. For example:
```
eselect profile set default/linux/amd64/13.0/desktop/gnome/systemd
```

#### USE variable
Our initial make.conf leaves out some USE flags to avoid circular dependencies during the initial system install. After updating your @world set, switch to the final make.conf and re-do the update:
```
mv /etc/portage/make.conf.final /etc/portage/make.conf
emerge --ask --deep --newuse @world
```

#### Kernel
If you are using `LVM` and `LuKS`, you will need to use `genkernel-next` instead of `genkernel` to compile a kernel and/or initramfs. This section will replace the [Configuring the Linux Kernel](https://wiki.gentoo.org/wiki/Handbook:AMD64/Installation/Kernel) section of the Gentoo Handbook.

##### Installing

To install `genkernel-next`, `grub` (version 2), `gentoo-sources`, and some bootsplash utilities, install our `dogoncouch-kernel` set:
```
emerge --ask --verbose @dogoncouch-kernel
```

##### Compiling

Our genkernel configuration will automatically configure the Grub bootloader after installing the kernel. Before compiling, install grub on `/dev/sda`:
```
grub-install /dev/sda
```

To compile and install a kernel, modules, and an initramfs:
```
genkernel --install all
```

`genkernel` will use the kernel configuration and `genkernel.conf` in `/etc`. If you want to be able to edit the kernel configuration, use menuconfig:
```
genkernel --menuconfig --install all
```


## Installing Gnome
This should be done once installation is complete, after rebooting into the new system. For this phase, do whatever you did to get networking to work on the install medium.

Once networking is up, install Gnome:

1. Install Gnome:
```
emergse --ask --verbose gnome
```

This will take a while.

2. Enable GDM:
```
systemctl enable gdm
```

3. Enable NetworkManager:
```
systemctl enable NetworkManager
```

Our `/etc/NetworkManager/conf.d/20-clone.conf` will enable stable mac address cloning (one different address for each network), and keep the vendor ID portion of the address.

4. Reboot
```
reboot
```


## Post-Gnome

### Sets
The incuded configuration files come with portage sets for various purposes. Portage sets are groups of packages that can be easily installed together. Available sets are:

- dogoncouch-core
    - Core packages
- dogoncouch-kernel
    - Packages related to our kernel
- dogoncouch-laptop
    - Non-model-specific laptop utilities (lm\_sensors, etc.)
- dogoncouch-net
    - Network analysis tools (non-GUI)
- dogoncouch-disk
    - Disk forensics utilities (non-GUI)
- dogoncouch-wifi
    - Wireless network analysis tools (non-GUI)
- dogoncouch-guisec
    - Graphical forensics and network analysis programs
- dogoncouch-office
    - Graphical programs for office productivity
- dogoncouch-gfx
    - Graphical programs for multi-media

To emerge a set:
```
emerge -av @dogoncouch-core
```

This will emerge the `dogoncouch-core` set.

### Bluetooth:
Use the [Gentoo Wiki article](https://wiki.gentoo.org/wiki/Bluetooth) to get bluetooth working. Our kernel contains a lot of bluetooth hardware modules, so the kernel configuration should be ok.

There is an issue with bluetooth and Gnome. Gnome seems to use rfkill to turn off the bluetooth interface, and then for some reason it disappears out of the menu and has to be unblocked manually, or through the settings GUI.

If bluetooth isn't working, make sure it is running/up/unblocked:
```
rfkill unblock bluetooth
hciconfig bluetooth up
systemctl start bluetooth
```

To Do: Full bluetooth control via gnome

### Laptop Sensors

#### Screen Rotation:
##### Installing iio-sensor-proxy
`iio-sensor-proxy` handles accelerometers and light sensors, and is used for screen rotation and automatic brightness control, if the system has the necessary sensors. Our kernel includes a lot of modules for sensors.

If you are using our all of our config files, and you have installed our `dogoncouch-laptop` set, `iio-sensor-proxy` should already be installed. Check with `emerge -pv iio-sensor-proxy`. If it is, skip ahead to [using iio-sensor-proxy](#using-iio-sensor-proxy)

First, download the [iio-sensor-proxy ebuild](https://bugs.gentoo.org/show_bug.cgi?id=565904). Then put it in the `/usr/portage` tree:
```
mkdir /usr/portage/sys-apps/iio-sensor-proxy
cp iio-sensor-proxy-2.0.ebuild /usr/portage/sys-apps/iio-sensor-proxy
```

Then emerge it:
```
emerge sys-apps/iio-sensor-proxy
```

##### Using iio-sensor-proxy
The iio-sensor-proxy service has to be started at first; after a while it becomes static, and no longer needs to be started.
```
systemctl start iio-sensor-proxy
```

In the beginning, it may not work on startup until after a suspend/resume cycle. That issue seems to go away on its own after a few days/weeks.

#### lm\_sensors
`lm_sensors` is a set of laptop hardware sensor utilities, and part of our `dogoncouch-laptop` set. Once installed, it can be configured using `sensors-detect` (go through the prompts, and pay attention to the warnings), and enabled with `systemctl enable --now lm_sensors`.


## Resources/Thanks

### General
- [Gentoo Handbook](https://wiki.gentoo.org/wiki/Handbook:Main_Page)
- [Kali Linux](https://www.kali.org/)

### LuKS/LVM
- [LuKS/LVM: Arch Linux Wiki](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system)
- [Dm-crypt: Gentoo Wiki](https://wiki.gentoo.org/wiki/Dm-crypt_full_disk_encryption)

### iio-sensor-proxy
- [iio-sensor-proxy ebuild](https://bugs.gentoo.org/show_bug.cgi?id=565904)
- [iio-sensor-proxy GitHub](https://github.com/hadess/iio-sensor-proxy)

### Bluetooth
- [Bluetooth: Gentoo Wiki](https://wiki.gentoo.org/wiki/Bluetooth)


## Contributing

### Experience
If you have a bug, an anomaly, or a story about anything from odd configurations to setting up complex software on top of our system, we would love to hear about it. File an issue on our [GitHub page](https://github.com/dogoncouch/gentoo-laptop), or email the developer at [dpersonsdev@gmail.com](mailto:dpersonsdev@gmail.com).

