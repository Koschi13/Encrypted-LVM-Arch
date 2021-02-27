# Basic
[Download](https://www.archlinux.org/download/ "Arch download")
latest Iso

## Partitioning the Harddrive
This Guide is will use EFI and LEGACY for booting.

For an EFI System make sure that Secure-Boot is disabled in the BIOS.

If you are on a VirtualBox don't use the Efi mode because it's causing
problems that are difficult so solve. If you want EFI anyway see
[here](https://askubuntu.com/a/573672) for a workaround

You can check if Arch booted into EFI with this command, if the Folder
does not exist you booted into LEGACY

```bash
$ ls /sys/firmware/efi/efivars
```

Before we begin we must know which drive to select

```bash
$ lsblk
```

Will print all your Drives and should look like this:

```bash
$ lsblk
NAME   MAJ:MIN  RM    SIZE  RO  TYPE  MOUNTPOINT
loop0    7:0     0  408.5M   1  loop  /run/archiso/sfs/airootfs
sda      8:0     0     10G   0  disk
sr0     11:0     1    523M   0   rom  /run/archiso/bootmnt
```

sda is our Harddrive where we want to install Arch, now we create two
partitions, one for the bootloader and the other for the LVM

```bash
$ gdisk /dev/sda
# This is our Boot partiton
n
  <enter>
  <enter>
  +100M  # some say use 300 others 500. I used 100 a lot and never got any problems
  ef00 *for EFI* | ef02 *for LEGACY*

# This will be our LVM partition
n
  <enter>
  <enter>
  <enter>
  8e00

w
Do you want to proceed? >> y
```


The next Step is to shred all Data on the Disk, if the disk is new this
is not necessary but you can do it anyway (Note: The Bigger the drive
the longer it will take to shred)

```
$ shred -v (n 1) /dev/sda  # use the n 1 if you want only one cycle insted of two
```

Now we are ready to setup the encrypted LVM, for that we need to load
the module dm-crypt

```
$ modprobe dm-crypt
```

Encrypt /dev/sda2. The more secure your password the more secure the
System will be

```
$ cryptsetup -c aes-xts-plain64 -y -s 512 luksFormat /dev/sda2
  >> YES  # Confirm that all Data will be overriden
  Enter passphrase: >> *Secred Password*
  Verify passphrase: >> *Secred Password*
 ```

Now Your Harddrive is encrypted to open it use this command (Remember
that, if you can't connect to your harddrive due to a failure you can
enter that partition with this command from a recovery image)

```bash
$ cryptsetup luksOpen /dev/sda2 lvm  # this will mount the device under /dev/mapper/lvm
```

Create Physical Volume

```bash
$ pvcreate /dev/mapper/lvm
```

Create Volume Group

```bash
$ vgcreate main /dev/mapper/lvm  # main is the Name for your Group
```

Create logical Volumes (Note: Programs are installed on root so give
this partiton enough space for all your Programs):

```bash
$ lvcreate -L XGB -n root main  # Will create a root partition with X GB size. If
                                # you want no extra volume for your home, use
                                # -l 100%FREE like in the last command.
$ lvcreate -L XGB -n swap main  # Only do this if you want to hibernate or open file
                                # that are larger than your RAM. The size should be
                                # at least the size of your RAM, if you wan't to
                                # hibernate.
$ lvcreate -l 100%FREE -n home main  # Will give home the rest of the aviable Space. Note the lowercase l
```

Now write the Filesystem:

```bash
$ mkfs.ext4 -L root /dev/mapper/main-root
$ mkfs.ext4 -L home /dev/mapper/main-home

# EFI - Warning when using lowercase: lowercase labels might not work properly with DOS or Windows
$ mkfs.fat -F 32 -n BOOT /dev/sda1
# LEGACY - disable 64bit
$ mkfs.ext4 -L boot -O '^64bit' /dev/sda1

# Only if you created the swap partition!
$ mkswap -L swap /dev/mapper/main-swap
```

If you ever need help with LVM see
[here](https://wiki.archlinux.org/index.php/LVM "LVM Arch Wiki")

Mount the Filesystem

```bash
$ mount /dev/mapper/main-root /mnt
$ mkdir /mnt/home
$ mount /dev/mapper/main-home /mnt/home
$ mkdir /mnt/boot
$ mount /dev/sda1 /mnt/boot
$ swapon /dev/mapper/main-swap # If swap was ceated
```

## Install the Base System

Check if your Internet connection works

```bash
$ ping -c3 www.archlinux.org
```

If not see here
([de](https://wiki.archlinux.de/title/Anleitung_f%C3%BCr_Einsteiger#Netzwerkverbindung_herstellen),
[en](https://wiki.archlinux.org/index.php/Installation_guide#Connect_to_the_Internet))
how to make it work or try first

```bash
$ dhcpcd
```

Now configure the Mirrors, first backup the File

<details>
  <summary>Old version</summary>
  
  For some reason, the old mirrorlist format isn't used anymore (2021.02.01) and therefore this code won't work anymore...
  
  ```bash
  $ cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak
  # XXX is your region e.g. Germany. use cat /etc/pacman.d/mirrorlist to see examples
  $ grep -E -A 1 ".*XXX.*$" /etc/pacman.d/mirrorlist.bak | sed '/--/d' > /etc/pacman.d/mirrorlist
  ```
  
</details>

```bash
$ cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.bak
# XX is your country domain (de for Germany)
$ grep -E  ".*\.XX.*$" /etc/pacman.d/mirrorlist.bak > /etc/pacman.d/mirrorlist
```

Now check if everything went right (You should only see Your entered
Region)

```bash
$ cat /etc/pacman.d/mirrorlist
```

Now download and install the Base System

Optional packages are:
1. [intel-ucode](https://wiki.archlinux.org/index.php/Microcode),
 if you have an Intel Kernel
2. [amd-ucode](https://wiki.archlinux.org/index.php/Microcode),
 if you have an AMD Kernel
3. [wpa_supplicant](https://wiki.archlinux.org/index.php/WPA_supplicant),
 if you're using WLAN
4. [wifi-menu(dialog)](https://www.archlinux.org/packages/core/x86_64/dialog/),
 if you want to setup your wifi with the Console
5. the others can be installed, you will know if you need them so I
  won't explain them here.

install everything you need for your System with (`[]` are optional dependencies):

```bash
$ pacstrap /mnt base linux linux-firmware grub efibootmgr base-devel vim dhcpcd \
cryptsetup device-mapper e2fsprogs lvm2 which inetutils man-db man-pages [nano] \
[neovim] [intel-ucode] [wpa_supplicant] [wifi-menu] [usbutils] [diffutils] \
[jfsutils] [less] [logrotate]
```

After the installation finished we generate our fstab

```
$ genfstab -U -p /mnt >> /mnt/etc/fstab
```

 Check the fstab
```bash
$ cat /mnt/etc/fstab
```

Should bring something like this

```
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
```

If you're on a LEGACY System the boot is also ext4 and if it's on a SSD
it should be adjusted too
With swap:

```
# /dev/sda3 LABEL=swap
UUID=blabla                                     none            swap        defaults    0   0
```

If one of these Filesystems is on a SSD change the ext4 to

```
UUID=blabla                                     /foo/bar        ext4        rw,defaults,noatime,discard 0   2
```

If you have a swap partition

```
UUID=blabla                                     none            swap        defaults,noatime,discard    0   0
```

After you finished the Filesystem check, change root into the new System

```bash
$ arch-chroot /mnt
```

## Configure the new System

Set your Host Name

```bash
$ echo YourHostName >> /etc/hostname
```

Set your Time Zone

```bash
$ ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
$ hwclock --systohc
```

Generate the locale

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

Load modules at boot (Change the Values MODULES and HOOKS to look like
below

```bash
$ vim /etc/mkinitcpio.conf
  >> MODULES=(ext4)
  >> HOOKS=(base udev autodetect modconf block keyboard keymap encrypt lvm2 filesystems fsck shutdown)
```

Generate Kernel

```bash
$ mkinitcpio -p linux  # If you get an Error like preset not Found --> the l is lowercase
```

Set root password (Don't use your User password here, think of a new one
and write it down)

```bash
$ passwd
  *Secred Password*
  *Secred Password*
```

Configure your Grub

```bash
$ vim /etc/default/grub
  >> GRUB_CMDLINE_LINUX="cryptdevice=/dev/sda2:lvm"
  # If Grub already has something in here add lvm, do not delete the oder Modules
  >> GRUB_PRELOAD_MODULES="lvm"
  >> save and exit (:qw)
```

Now configure your grub with:

```bash
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

Now exit and reboot into your new System

```bash
$ exit
$ umount /mnt/{boot,home,}
$ reboot
```

Now login and create your User

```bash
$ useradd -m -g users -s /bin/bash username
$ passwd username
  *Secred Password*
  *Secred Password*
$ EDITOR=vim visudo
  >> Uncomment %wheel ALL=(ALL) ALL
  >> save and exit (:wq)
$ gpasswd -a username wheel
```

Install useful Services
1. acpid, a modue that handles ACPI events (Such as closing the lid of
a notebook)
2. dbus, message System for inter-process communication
3. avahi, a zero configuration network tool for easy networking
4. cups, a module for printer support
5. cronie, a time based scheduler

Install all Services you want for your System (Install the others when
you need them)

```bash
$ pacman -S acpid dbus avahi cups cronie
$ systemctl enable acpid
$ systemctl enable avahi-daemon
$ systemctl enable org.cups.cupsd.service
$ systemctl enable cronie
```

Now your Base System is ready, you can now install your preferred
Desktop or use arch from the command Line

## Desktop installation
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
