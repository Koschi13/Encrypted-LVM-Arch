# Basic
[Download](https://www.archlinux.org/download/ "Arch download")
latest Iso

## Partitioning the Harddrive
This Guide is for EFI boot, for and EFI System make sure that
Secure-Boot is disabled in the BIOS

You can check if Arch booted into EFI with this command, if the Folder
does not exist you booted into LEGACY

    $ ls /sys/firmware/efi/efivars

Before we begin we must know which drive to select

    $ lsblk

Will print all your Drives and should look like this:

    NAME   MAJ:MIN  RM    SIZE  RO  TYPE  MOUNTPOINT
    loop0    7:0     0  408.5M   1  loop  /run/archiso/sfs/airootfs
    sda      8:0     0     10G   0  disk
    sr0     11:0     1    523M   0   rom  /run/archiso/bootmnt

sda is our Harddrive where we want to install Arch, now we create two
partitions, one for the bootloader and the other for the LVM

    $ gdisk /dev/sda
    # This is our Boot partiton
    >> n
    >> enter
    >> +100M
    >> ef00

    # This will be our LVM partition
    >> n
    >> enter
    >> enter
    >> 8e00

    >> w
    Do you want to proceed? >> y


The next Step is to shred all Data on the Disk, if the disk is new this
is not necessary but you can do it anyway (Note: The Bigger the drive
the longer it will take to shred

    $ shred -v (n 1) /dev/sda  # use the n 1 if you want only one cycle insted of two

Now we are ready to setup the encrypted LVM, for that we need to load
the module dm-crypt

    $ modprobe dm-crypt

Encrypt /dev/sda2. The more secure your password the more secure the
System will be

    $ cryptsetup -c aes-xts-plain64 -y -s 512 luksFormat /dev/sda2
    >> YES  # Confirm that all Data will be overriden
    Enter passphrase: >> *Secred Password*
    Verify passphrase: >> *Secred Password*

Now Your Harddrive is encrypted to open it use this command (Remember
that, if you can't connect to your harddrive due to a failure you can
enter that partition with this command from a recovery image)

    $ cryptsetup luksOpen /dev/sda2 lvm  # this will mount the device under /dev/mapper/lvm

Create Physical Volume

    $ pvcreate /dev/mapper/lvm

Create Volume Group

    $ vgcreate main /dev/mapper/lvm  # main is the Name for your Group

Create logical Volumes (Note: Programs are installed on root so give
this partiton enough space for all your Programs)

    $ lvcreate -L XGB -n root main  # Will create a root partition with X GB size
    $ lvcreate -L XGB -n swap main  # Only use this on a Laptop X is the Size of your RAM but max. 8GB
    $ lvcreate -l 100%FREE -n home main  # Will give home the rest of the aviable Space

Now write the Filesystem

    $ mkfs.ext4 -L root -O \^64bit /dev/mapper/main-root
    $ mkfs.ext4 -L home -O \^64bit /dev/mapper/main-home
    $ mkfs.fat -F 32 -n boot /dev/sda1
    $ mkswap -L swap /dev/mapper/main-swap  # Only if you created one!

If you ever need help with LVM see
[here](https://wiki.archlinux.org/index.php/LVM "LVM Arch Wiki")

Mount the Filesystem

    $ mount /dev/mapper/main-root /mnt
    $ mkdir /mnt/home
    $ mount /dev/mapper/main-home /mnt/home
    $ mkdir /mnt/boot
    $ mount /dev/sda1 /mnt/boot

## Install the Base System

Check if your Internet connection works

    $ ping -c3 www.archlinux.org

If not see here
([de](https://wiki.archlinux.de/title/Anleitung_f%C3%BCr_Einsteiger#Netzwerkverbindung_herstellen),
[en](https://wiki.archlinux.org/index.php/Installation_guide#Connect_to_the_Internet))
how to make it work or try first

    $ dhcpcd

Now configure the Mirrors, first backup the File

    $ cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak
    $ grep -E -A 1 ".*XXX.*$" /etc/pacman.d/mirrorlist.bak | sed '/--/d' > /etc/pacman.d/mirrorlist  # XXX is your region e.g. Germany

Now check if everything went right (You should only see Your entered
Region)

    $ cat /etc/pacman.d/mirrorlist

Now download and install the Base System

Optional packages are:
1. [base-devel](https://www.archlinux.org/groups/x86_64/base-devel/),
 install this only not if you know, what you are doing
2. [intel-ucode](https://wiki.archlinux.org/index.php/Microcode),
 if you have an Intel Kernel
3. [wpa_supplicant](https://wiki.archlinux.org/index.php/WPA_supplicant),
 if you're using WLAN
4. [wifi-menu(dialog)](https://www.archlinux.org/packages/core/x86_64/dialog/),
 if you want to setup your wifi with the Console
5. [grub and efibootmgr(if you're on an EFI System)](https://wiki.archlinux.org/index.php/GRUB),
 for the grub installation (You can do this also later)

install everything you need for your System with:

    $ pacstrap /mnt base [base-devel] [intel-ucode] [wpa_supplicant] [wifi-menu] [grub] [efibootmgr]

After the installation finished we generate our fstab

    $ genfstab -U -p /mnt >> /mnt/etc/fstab

 Check the fstab

    $ cat /mnt/etc/fstab

Should bring something like this

    #
    # /etc/fstab: static file system information
    #
    # <file system> <dir>   <type>  <options>       <dump>  <pass>
    # /dev/mapper/main-root LABEL=root
    UUID=blabla                                     /               ext4        rw,realtime,data=orderes    0   2

    # /dev/mapper/main-home LABEL=home
    UUID=blabla                                     /home           ext4        rw,realtime,data=ordered    0   2

    # /dev/sda1 LABEL=boot
    UUID=blabla                                     /boot           vfat        rw,realtime,....    0   2

With swap:

    # /dev/sda3 LABEL=swap
    UUID=blabla                                     none            swap        defaults    0   0

If one of these Filesystems is on a SSD change the ext4 to

    UUID=blabla                                     /foo/bar        ext4        rw,defaults,noatime,discard 0   2

If you have a swap partition

    UUID=blabla                                     none            swap        defaults,noatime,discard    0   0

After you finised the Filesystem check, change root into the new System

    $ arch-root /mnt

## Configure the new System

Set your Host Name

    $ echo YourHostName >> /etc/hostname

Set your Time Zone

    $ ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
    $ hwclock --systohc

Generate the locale

    $ nano /etc/locale.gen
    >> Uncomment every localisation for your Language e.g.:
    >> en_US.UTF-8 UTF-8
    >> en_US ISO-8859-1
    >> save and exit
    $ locale-gen

To check if everything went right run

    $ locale

Load modules at boot (Change the Values MODULES and HOOKS to look like
below

    $ nano /etc/mkinitcpio.conf
    >> MODULES=(ext4)
    >> HOOKS=(base udev autodetect modconf block keyboard keymap encrypt lvm2 filesystems fsck shutdown)

Generate Kernel

    $ mkinitcpio-p linux  # If you get an Error like preset not Found --> the l is lowercase

Set root password (Don't use your User password here, think of a new one
and write it down)

    $ passwd
    >> *Secred Password*
    >> *Secred Password*

Configure your Grub

    $ nano /etc/default/grub
    >> GRUB_CMDLINE_LINUX="cryptdevice=/dev/sda2:lvm"
    >> sve and exit

Now if you installed grub and efibootmgr run this command

    $ grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=arch_grub --recheck --debug

If you run into Errors like this:

    WARNING: failed to connect to lvmetad: No such file or directory. Falling back to internal scanning.
    /run/lvm/lvmetad.socket: connect failed: No such file or directory

Ignore them this ist because you are in a chroot env.

Now exit and reboot into your new System

    exit
    umount /mnt/{boot,home,}
    reboot

Now login and create your User

    $ useradd -m -g users -s /bin/bash username
    $ passwd username
    >> *Secred Password*
    >> *Secred Password*
    $ EDITOR=nano visudo
    >> Uncomment %wheel ALL=(ALL) ALL
    >> save and exit
    $ gpasswd -a username wheel

Install useful Services
1. acpid, a modue that handles ACPI events (Such as closing the lid of
a notebook)
2. dbus, message System for inter-process communication
3. avahi, a zero configuration network tool for easy networking
4. cups, a module for printer support
5. cronie, a time based scheduler

Install all Services you want for your System (Install the others when
you need them)

    $ pacman -S acpid dbus avahi cups cronie
    $ systemctl enable acpid
    $ systemctl enable avahi-daemon
    $ systemctl enable org.cups.cupsd.service
    $ systemctl enable cronie

Now your Base System is ready, you can now install your preferred
Desktop or use arch from the command Line

# Desktop installation


