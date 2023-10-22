# Basic

[Download](https://www.archlinux.org/download/ "Arch download")
the latest Iso

## TL;DR

Before you tackle this ~1h guide it should be mentioned that you can install
arch in a guided manner with the
[archinstall](https://wiki.archlinux.org/title/archinstall) helper library. To
get an idea of this 'easy Arch install' visit their
[docs](https://python-archinstall.readthedocs.io/en/latest/installing/guided.html).
Lets further add that you could with the same libary install Arch directly from
a configuration file. If you decide to use the guided way you can start the
guide at section [After first reboot](#After-first-reboot)

While automated tools like archinstall streamline the installation process,
going through this manual guide offers you the advantage of a deeper
understanding of your system's architecture. This guide equips you with valuable
insights into component configuration, allowing for a level of control and
customization that's hard to achieve otherwise. Additionally, should your system
become 'bricked' or unresponsive, this knowledge could be invaluable for
troubleshooting and recovery. Consider this manual process an investment in your
own skills and system mastery.

## Partitioning the Harddrive

This guide covers both EFI and Legacy boot modes.

### Pre-requisites

1. Ensure that `Secure Boot` is disabled in your system BIOS.
2. If you are using VirtualBox, it's recommended to avoid EFI mode as it can
   introduce complications. For those who still wish to proceed with EFI on
   VirtualBox, consult this [workaround](https://askubuntu.com/a/573672).

To determine if your system has booted in EFI or Legacy mode, run the following
command:

```bash
$ ls /sys/firmware/efi/efivars
```

If the directory does not exist, you have booted into Legacy mode.

### Identifying the Target Drive

To list all available drives, execute:

```bash
$ lsblk
```

This should display output similar to:

```bash
$ lsblk
NAME        MAJ:MIN  RM    SIZE  RO  TYPE  MOUNTPOINT
loop0         7:0     0  408.5M   1  loop  /run/archiso/sfs/airootfs
sda           8:0     0     10G   0  disk
sr0          11:0     1    523M   0   rom  /run/archiso/bootmnt
nvme0n1     259:0     0    500G   0  disk
```

In this output example `sda` or `nvme0n1` could be the target drive where you intend to install Arch Linux. The drive names depend on the type of drive you have:

- sda, sdb, etc. are usually SATA, SCSI, or older types of drives.
- nvme0n1, nvme0n2, etc., are NVMe drives.

In our example we go with `sda` as the target drive where we intend to install Arch Linux. Next, we'll create two partitions: one for the bootloader and another for LVM.

### Creating Partitions

Run `gdisk` to interactively partition your target drive:

[A sidenode from the arch wiki:](https://wiki.archlinux.org/title/partitioning)

> "A suggested size for /boot is 200 MiB unless you are using EFI system partition
> as /boot, in which case at least 300 MiB is recommended. If you want to install
> multiple kernels or want to be future proof, you can use 1 GiB to be on the safe
> side."

```bash
$ gdisk /dev/sda
```

If neccesary delete old partitions first.
Then start with the boot partition.

```
Command (? for help): n
Partition number (1-128, default 1): <enter>
First sector (34-20971486, default = 2048) or {+-}size{KMGTP}: <enter>

# Don't hesitate to pick 2048M (2GB) for the last sector.
Last sector (2048-20971486, default = 20971486) or {+-}size{KMGTP}: +1024M

Hex code or GUID (L to show codes, Enter = 8300): ef00  # for EFI, ef02 for LEGACY
```

Continue with the LVM partition

```
Command (? for help): n
Partition number (2-128, default 2): <enter>
First sector (34-20971486, default = 20973568) or {+-}size{KMGTP}: <enter>
Last sector (20973568-20971486, default = 20971486) or {+-}size{KMGTP}: <enter>
Hex code or GUID (L to show codes, Enter = 8300): 8e00
```

### (Optional) Securely Wiping the Drive

If you are using a new disk, this step is not mandatory but can be performed for added security.

```bash
$ shred -v -n 1 /dev/sda  # use -n 1 for a single pass instead of the default 3 passes
```

### Setting Up Encrypted LVM

Before configuring the encrypted LVM (Logical Volume Manager), it's essential to
load the required kernel module, `dm-crypt`. This module enables disk encryption
on Linux.

Load the `dm-crypt` Kernel Module

```bash
$ modprobe dm-crypt
```

The next step is to encrypt the LVM partition. In this guides example, the LVM
partition is /dev/sda2.

```
$ cryptsetup -c aes-xts-plain64 -y -s 512 luksFormat /dev/sda2
  >> YES  # Confirm that all Data will be overriden
  Enter passphrase: >> *Secred Password*
  Verify passphrase: >> *Secred Password*
```

After successfully encrypting your LVM partition, you'll need to unlock it to
proceed with the installation. To unlock your encrypted LVM partition, use the
following command:

```bash
$ cryptsetup luksOpen /dev/sda2 lvm  # this will mount the device under /dev/mapper/lvm
```

Note: If you ever face an issue that prevents you from accessing your hard
drive, you can use this command from a recovery environment to unlock the
partition.

After unlocking the encrypted partition, it's essential to set up Logical Volume
Management (LVM) and initialize the file systems for the installation.

Initialize `/dev/mapper/lvm` as a physical volume:

```bash
$ pvcreate /dev/mapper/lvm
```

Create a volume group (VG) named `main`: all the subsequent logical volumes (root,
swap, home) will be part of this volume group.

```bash
$ vgcreate main /dev/mapper/lvm # "main" is the name of your volume group.
```

Create `root` and `swap` Logical Volumes: These logical volumes will reside inside
the `main` volume group.

Note: Consider using a [swap file](https://wiki.archlinux.org/title/swap) with [the right size](https://opensource.com/article/19/2/swap-space-poll).
If you want the ability to hibernate or handle files that are larger than your RAM, a swap space is generally recommended. This is because, during hibernation, the contents of RAM are written to the swap space.
A swap space as large as your RAM up to 16GB is often sufficient for most users. If you have more than 16GB of RAM, you can opt for a smaller swap space depending on your specific needs. However, for performance-intensive tasks like video editing or large-scale data processing, a larger swap space may still be beneficial.

```bash
# "XGB" is the size of the swap, "swap" is the logical volume name, and "main" is the name of your volume group.
$ lvcreate -L XGB -n swap main

# Will give root the rest of the aviable Space. Note the lowercase l
$ lvcreate -l 100%FREE -n root main
```

Create the file systems.

```bash
$ mkfs.ext4 -L root /dev/mapper/main-root

# Only EFI - Warning when using lowercase: lowercase labels might not work properly with DOS or Windows
$ mkfs.fat -F 32 -n BOOT /dev/sda1
# Only LEGACY - disable 64bit
$ mkfs.ext4 -L boot -O '^64bit' /dev/sda1

# Only if you created the swap partition!
$ mkswap -L swap /dev/mapper/main-swap
```

If you ever need help with LVM see
[here](https://wiki.archlinux.org/index.php/LVM "LVM Arch Wiki")

Mount the Filesystem

```bash
$ mount /dev/mapper/main-root /mnt
$ mkdir /mnt/boot
$ mount /dev/sda1 /mnt/boot

# Only if you created the swap partition!
$ swapon /dev/mapper/main-swap
```

## Install the Base System

Check if your Internet connection works

```bash
$ ping -c3 www.archlinux.org
```

If you are not connected to the Internet and wish to use Wi-Fi, you'll need to
identify your wireless network interface and connect using wpa_supplicant. Use
the following command to list network interfaces and find your wireless network
device:

```bash
$ ip a
```

Then connect through wpa_supplicant.
Figure out your <wifi-name> with `ip a`.

```bash
$ wpa_supplicant -B -i <wifi-name> -c <(wpa_passphrase "Your_SSID" Your_passphrase) && dhclient <wifi-name>
```

For more details, consult the [official documentation](https://wiki.archlinux.org/title/Wpa_supplicant).
If you encounter issues or prefer a different method for connecting to the Internet, consult the
[Arch Linux Installation Guide](https://wiki.archlinux.org/index.php/Installation_guide#Connect_to_the_Internet).

If you're facing issues with obtaining an IP address, you can manually trigger
the DHCP client as follows:

```bash
$ dhcpcd
```

## Configure the Mirror List (Optional)

Note: The need to adjust the mirror list can depend on various factors, such as
your geographical location, network speed requirements, and specific use-cases.
For most users, the default mirror list should suffice, especially given that
Arch Linux in 2023 utilizes a ranked system for mirrors. This makes manual
intervention less necessary. However, if you experience slower-than-expected
download speeds and are not ( a digital nomad ) frequently changing your
geographical location, you may want to optimize the mirror list.

Before making any changes, backup your current mirror list:

```bash
$ cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak
```

To use mirrors from a specific country (replace XX with your country's domain
code, e.g., de for Germany):

```bash
$ grep -E  ".*\.XX.*$" /etc/pacman.d/mirrorlist.bak > /etc/pacman.d/mirrorlist
```

Check to make sure your mirror list now only contains entries for your chosen country:

```bash
$ cat /etc/pacman.d/mirrorlist
```

## Installing the Base System

Optional packages are:

1. [intel-ucode](https://wiki.archlinux.org/index.php/Microcode),
   for Intel processors.
2. [amd-ucode](https://wiki.archlinux.org/index.php/Microcode),
   for AMD processors.
3. [wpa_supplicant](https://wiki.archlinux.org/index.php/WPA_supplicant),
   if you're using WIFI
4. [linux-lts](https://archlinux.org/packages/core/x86_64/linux-lts/),
   if you want a secondary stable LTS linux kernel for backup usecases or as an alternative to the rolling release
5. Other packages may be essential based on your specific needs.

Install everything you need for your System with (`[]` are optional dependencies):

```bash
$ pacstrap /mnt base linux linux-firmware grub efibootmgr base-devel vim dhcpcd \
cryptsetup lvm2 which inetutils man-db man-pages sudo curl gzip unzip \
[intel-ucode] [linux-lts] [wpa_supplicant] [usbutils] [git] [diffutils] \
[jfsutils] [less] [logrotate] [wget] [neovim]
```

After the installation finished we generate the Filesystem Table (fstab) to
ensure that all filesystems are correctly mounted during boot

```bash
$ genfstab -U -p /mnt >> /mnt/etc/fstab
```

This will generate the fstab file using UUIDs for identifying partitions and
append the generated entries into `/mnt/etc/fstab`.

Verifying the fstab file:

```bash
$ cat /mnt/etc/fstab
```

Should bring something like this

```
#
# /etc/fstab: static file system information
#
# <file system> <dir>  <type>  <options>       <dump>  <pass>

# Root partition
UUID=your-root-uuid  /       ext4    rw,relatime,data=ordered  0  1

# EFI Boot partition (if using EFI)
UUID=your-boot-uuid  /boot  vfat  rw,relatime,fmask=0022,dmask=0022,codepage=437,iocharset=ascii,shortname=mixed,errors=remount-ro  0  2

# Swap partition
UUID=your-swap-uuid  none    swap    sw                        0  0
```

If you are using a LEGACY system with a non-EFI boot, the boot partition is
likely formatted as ext4. You might need to consider SSD optimizations if
applicable.

If any of these file systems are on an SSD, consider modifying the ext4 entry like this:

```
UUID=your-uuid  /your/mount/point  ext4  rw,defaults,noatime,discard  0  2
```

If you have a swap partition

```
UUID=your-swap-uuid  none  swap  defaults,noatime,discard  0  0
```

After you finished the Filesystem check, change root into the new System

```bash
$ arch-chroot /mnt
```

## Configure the New System

Set your Host Name

```bash
$ echo YourHostName >> /etc/hostname
```

Set your systems time zone by creating a symbolic link from the appropriate
zoneinfo file to /etc/localtime. Replace `Region` and `City` with your specific time
zone details. E.g. Region (Europe) and City (Berlin) for Germany.

```bash
$ ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
```

Run the `hwclock` command to generate `/etc/adjtime` and synchronize the hardware
clock to the system clock:

```bash
$ hwclock --systohc
```

Edit the `/etc/locale.gen` file to uncomment the locales you want to generate.
Then generate it:

```bash
$ vim /etc/locale.gen
  >> Uncomment every localisation for your Language e.g.:
  >> en_US.UTF-8 UTF-8
  >> en_US ISO-8859-1
  >> save and exit
$ locale-gen
```

To check if everything went right run

```bash
$ locale
```

The `mkinitcpio.conf` file is used to configure the initial RAM filesystem
(initramfs) in Arch Linux. The `MODULES` and `HOOKS` lines in this file specify
which modules and hooks are included in the initramfs.

```bash
  $ vim /etc/mkinitcpio.conf
  # Add or modify the MODULES line to include the `ext4` file system module,
  # or any other filesystem you're using for your root partition
  >> MODULES=(ext4)
  # Add or modify the HOOKS line to include the base hooks needed for booting.
  # This example includes support for encryption and LVM by adding `encrypt` and `lvm2`.
  >> HOOKS=(base udev autodetect modconf block keyboard keymap encrypt lvm2 filesystems fsck shutdown)
```

To generate the initramfs images for the kernels you have installed (linux and
optionally linux-lts), run the following commands:

```bash
$ mkinitcpio -p linux  # If you get an Error like preset not Found --> the l is lowercase
# Only if you installed the linux-lts as secondary Kernel option
$ mkinitcpio -p linux-lts
```

Set root password (Don't use your User password here, think of a new one
and write it down)

```bash
$ passwd
  *Secred Password*
  *Secred Password*
```

Enable `dhcpcd` so you have wired internet after reboot.

```bash
$ systemctl enable dhcpcd.service
```

If you need WIFI internet after reboot you could revisit the wpa_supplicant
command above to enable that manually once rebooted. Later in this guide `nmcli`
is itroduce as the 'automatic way of WIFI connecting'.

Configure your Grub bootloader:

```bash
$ vim /etc/default/grub
  >> GRUB_CMDLINE_LINUX="cryptdevice=/dev/sda2:lvm"
  # If Grub already has something in here add lvm, do not delete the oder Modules
  >> GRUB_PRELOAD_MODULES="lvm"
```

Generate the GRUB configuration file using:

```bash
$ mkdir -p /boot/grub
$ grub-mkconfig -o /boot/grub/grub.cfg
```

Now if you installed grub and efibootmgr run this command

```bash
# EFI
$ grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=arch_grub --recheck --debug
# LEGACY
$ grub-install /boot
```

If you run into Errors like this:

```
WARNING: failed to connect to lvmetad: No such file or directory. Falling back to internal scanning.
/run/lvm/lvmetad.socket: connect failed: No such file or directory
```

Ignore them this ist because you are in a chroot env.

Now exit and reboot into your new system:

```bash
$ exit
$ umount /mnt/{boot,home,}
$ reboot
```

## After First Reboot

Login and create a non-root user:

```bash
$ useradd -m -g users -s /bin/bash your-username
$ passwd your-username
  *Secred Password*
  *Secred Password*
$ EDITOR=vim visudo
  >> Uncomment %wheel ALL=(ALL) ALL
  >> save and exit (:wq)
$ gpasswd -a your-username wheel
```

## (Optional) Connect to the WIFI via NetworkManager

If you dont have Internet you could plug in your wired connection or connect
once again manually via wpa_supplicant.
First figure out your `<wifi-name>` with `ip a` then use the following command:

```bash
$ wpa_supplicant -B -i <wifi-name> -c <(wpa_passphrase "Your_SSID" Your_passphrase) && dhcpcd <wifi-name>
```

If you have not already you can install, enable and start NetworkManager (Note
that the package name is plain lowercase and the service name is PascalCase):

```bash
$ pacman -s networkmanager
$ systemctl enable NetworkManager
$ systemctl start NetworkManager
```

Connect to your WIFI

```bash
$ nmcli device wifi connect 'Your_SSID' password 'Your_Password'
```

To make sure your system automatically reconnects to this WIFI network whenever
it's available, you can modify the connection settings to enable auto-connect.

```bash
$ nmcli connection show
```

Then if desired, set the connection to auto-connect and/or disable powersave to
bolster a stable wifi connection and restart:

```bash
$ nmcli connection modify 'Your_Connection_Name' connection.autoconnect yes
$ nmcli connection modify 'Your_Connection_Name' 802-11-wireless.powersave 2
$ systemctl restart NetworkManager
```

## Install Useful Services

1. `acpid,` a modue that handles ACPI events (Such as closing the lid of
   a notebook)
2. `dbus,` message System for inter-process communication
3. avahi, a zero configuration network tool for easy networking
4. cups, a module for printer support
5. cronie, a time based scheduler

Install all Services you want for your System (Install the others when
you need them)

```bash
$ pacman -S acpid dbus avahi cups cronie
$ systemctl enable acpid
$ systemctl enable avahi-daemon
$ systemctl enable cups.service
$ systemctl enable cronie
```

Now your base system is ready, you can now install your preferred
Desktop or use arch from the command Line

## (Optional) Install an AUR Package Manager.

There are various package managers e.g. `yay` but here we install `rua` as a build tool
for the AUR.

```bash
$ pacman -S --needed --asdeps git base-devel bubblewrap-suid libseccomp xz shellcheck cargo
1 # Choose rust.
su - [your_username] # You should avoid building AUR packages as root.
tmpdir=$(mktemp -d)
cd $tmpdir
git clone https://aur.archlinux.org/rua.git
cd rua
makepkg -si
cd ~
rm -rf $tmpdir
```

## Desktop Installation

First of all we need to install our Graphic drivers. To see which
Graphics your System is using type

```bash
$ lspci | grep VGA
```

This should print something like this

```
00:02.0	VGA compatible controller: InnoTek Systemberatung GmbH VirtualBox Graphics Adapter
```

Now you can look [here(de)](https://wiki.archlinux.de/title/Anleitung_f%C3%BCr_Einsteiger#Grafiktreiber_installieren)
for a full list of aviable Graphic drivers or type

```bash
$ pacman -Ss xf86-video | less
```

In my case (Virtual Box) I installed virtualbox-guest-utils

```bash
$ pacman -S virtualbox-guest-utils
```

If you are on a Notebook you can install this Touchpad driver

```bash
$ pacman -S xf86-input-synaptics
```

Now we need our Display Manager, if you don't know which to take take the XORG
which can handle every Window-Manager. If you want a specific one for your
Window-Manager see [here](https://wiki.archlinux.org/index.php/Display_manager)

This installation will take the SDDM Display-Manager (Usually used for KDE) and XFCE as Desktop-Manager

```bash
$ pacman -S xfce4
$ pacman -S sddm
$ sddm --example-config > /etc/sddm.conf
$ vim /etc/sddm.conf
  >> Session=startxfce4
  >> save and exit
$ systemctl enable sddm
$ systemctl start sddm
```

After the last Command the Graphics should start and SDDM will promt for your User

Now you have your System ready to use, all basic functionallity is given.
You can now enjoy your Arch Linux
