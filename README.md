# Gentoo UEFI secure short guide

**Disclaimer:** This guide is based on
[Sakaki's EFI Install Guide](https://wiki.gentoo.org/wiki/Sakaki%27s_EFI_Install_Guide) that guides through installing
Gentoo securely creating a multiboot environment.
This mostly just gives the plain commands and presents author's visions on some choices given in original guide.
Some parts may have been adapted from official [Gentoo Handbook](https://wiki.gentoo.org/wiki/Handbook:AMD64).
This documentation is licensed under [Creative Commons Attribution-ShareAlike 4.0 Unported License](https://creativecommons.org/licenses/by-sa/4.0/).
Only actual commands have been copied or adapted from the sources.

**Note:** This guide assumes some basic knowledge about managing a Gentoo Linux installation.
This is not a suitable guide for beginners in Gentoo nor Linux (but you are welcome to try to understand purpose of
different steps done here and learn by that).

Through this guide (and in some extent, the original guide) following result is being pursued:

- Full disk encryption with LUKS and GPG,
- UEFI Secure Boot to kernel stored in removable device,
- held chain of trust from Gentoo releases to finished installation,
- possibility to boot to original OS of the computer (not so much covered here).
- MISSING: Signing the kernel and enabling secure boot...

**Note:** This guide contains hard disk device names such as sdY, sdY1, sdZ and sdZn.
Take care when replacing them with actual identifiers!

- sdX is the USB key used for LiveCD. Not needed, if actual CD is used.
- sdY should be the USB key used to boot the final system.
- sdY1 is the actual EFI partition on that USB key, containing kernel and encrypted disk key.
- sdZ should be the target device hard drive.
- sdZn is the encrypted partition that is going to contain the whole gentoo, i.e. the lvm volume.

[TOC]

## Requirements

- UEFI PC
- USB key of size at least 250MB
- Internal or external CD-drive OR another USB key of size at least 500MB
  (sufficient for holding Gentoo minimal installation medium)


## Preparing the PC

If you need to make space for the installation from existing Windows, see
[corresponding part in Sakaki's guide](https://wiki.gentoo.org/wiki/Sakaki%27s_EFI_Install_Guide/Preparing_Windows_for_Dual-Booting)
for that.

TL;DR: Turn off system protection, swap and hibernation, reboot, defragment drive, shrink the drive using disk manager
(possibly multiple times using at most half of the free space), reboot, restore hibernation, swap and system protection.


## Preparing the installation medium

See [corresponding part in Sakaki's guide](https://wiki.gentoo.org/wiki/Sakaki%27s_EFI_Install_Guide/Creating_and_Booting_the_Minimal-Install_Image_on_USB)
for more details.

TL;DR:

Retrieve .iso, .iso.DIGESTS.asc and optionally .iso.CONTENTS -files from a close [Gentoo mirror](https://gentoo.org/downloads/mirrors/).
Check that the signature present matches [Gentoo Release media signatures](https://www.gentoo.org/downloads/signatures/):

```bash
gpg --keyserver hkps.pool.sks-keyservers.net --recv-keys 0xBB572E0E2D182910
gpg --fingerprint 0xBB572E0E2D182910  # ensure this matches one found in above signatures link.
gpg --verify *.iso.DIGESTS.asc  # should contain line like  gpg: Good signature from ...
sha512sum -c *.iso.DIGESTS.asc  # should contain line like  install-amd64-minimal-*.iso: OK
```

Burn the image to CD (using your preferred writer software) or USB key
(`dd if=path/to/install-amd64-minimal-*.iso of=/dev/sdX bs=8192k status=progress && sync`, remember to replace sdX with
correct USB device).

Finally, boot the target machine using the live "CD" in EFI mode (instead of Legacy Boot).


## First steps and networking

[Original part in Sakaki's guide](https://wiki.gentoo.org/wiki/Sakaki%27s_EFI_Install_Guide/Setting_Up_Networking_and_Connecting_via_ssh)

```bash
date  # check UTC time
date +MMDDhhmmYYYY  # replace with correct values
ifconfig  # see if you have network
net-setup  # if network was not automatically found

# If ssh is used:
passwd
/etc/init.d/sshd start
# See fingerprints by either first or second (for outdated clients) command:
for K in /etc/ssh/ssh_host_*key.pub; do ssh-keygen -l -f "${K}"; done
for K in /etc/ssh/ssh_host_*key.pub; do ssh-keygen -l -E md5 -f "${K}"; done
```

## Preparing boot key

### Partitioning

[Original part in Sakaki's guide](https://wiki.gentoo.org/wiki/Sakaki%27s_EFI_Install_Guide/Preparing_the_LUKS-LVM_Filesystem_and_Boot_USB_Key).

Find your USB key that will be used to boot the finished system.

```bash
lsblk
```

Partition the key drive with either option A or B.
I prefer A as mildly safer because it doesn't change the disk before committing to the changes.

#### Option A: gdisk

```bash
gdisk /dev/sdY
```
```text
    o   # create an empty GUID partition table.
        # if you skip this, rest of the commands may work differently.
        y   # yes
    n   # new partition
        <enter>   # default number, first partition
        <enter>   # default start, 2048
        <enter>   # default end, end
        ef00   # partition type EFI system
    c   # change partition name, optional?
        primary
    p   # see what we are committing to
    w   # commit changes and exit
```

#### Option B: parted

```bash
parted -a optimal /dev/sdY
```
```text
    mklabel gpt
        yes
    mkpart primary fat32 0% 100%
    set 1 BOOT on
    quit
```

### Filesystem

After partitioning the key is done, continue with creating filesystem and mounting it.

```bash
mkfs.vfat -F32 /dev/sdY1
mkdir /mnt/efiboot
mount -t vfat /dev/sdY1 /mnt/efiboot
```

### Creating keyfile for LUKS

```bash
export GPG_TTY=$(tty)
dd if=/dev/urandom bs=8388607 count=1 | gpg --symmetric --cipher-algo AES256 --output /mnt/efiboot/luks-key.gpg
# bs value is 8MiB - 1
```


## Preparing the main drive

### Partitioning

Find the main drive device and start `gdisk` on it:

```bash
lsblk
gdisk /dev/sdZ
```

If the drive was shrunk from Windows in first part of this guide,
the drive should have a single contiguous free section at the end of the drive.
In `gdisk`, do

```text
    n   # new partition
        <enter>   # default, next partition number, mark the number down.
        <enter>   # default, first free block
        <enter>   # default, last free block
        <enter>   # default, 8300 Linux Filesystem
    p   # see what we are committing to
    w   # commit changes and exit
```

### Formatting the new partition with LUKS

Replace `sdZn` with identifier for the newly created partition on the main drive.

```bash
# Optionally, reload gpg agent if you want gpg to ask the password in next step.
echo RELOADAGENT | gpg-connect-agent

gpg --decrypt /mnt/efiboot/luks-key.gpg | cryptsetup --cipher serpent-xts-plain64 --key-size 512 --hash whirlpool --key-file - luksFormat /dev/sdZn

# Check that the formatting worked
cryptsetup luksDump /dev/sdZn
```

#### Optional: Add fallback passphrase

In other VT, do
```bash
mkfifo /tmp/gpgpipe
echo RELOADAGENT | gpg-connect-agent
gpg --decrypt /mnt/efiboot/luks-key.gpg | cat - >/tmp/gpgpipe
```

In original VT, do
```bash
cryptsetup --key-file /tmp/gpgpipe luksAddKey /dev/sdZn

# Check that key was added
cryptsetup luksDump /dev/sdZn

# Cleanup
rm -f /tmp/gpgpipe
```

### Creating filesystems on LVM

Open the encrypted disk and create LVM structures.

```bash
gpg --decrypt /mnt/efiboot/luks-key.gpg | cryptsetup --key-file - luksOpen /dev/sdZn gentoo
pvcreate /dev/mapper/gentoo
vgcreate vg1 /dev/mapper/gentoo

grep MemTotal /proc/meminfo  # For suspend to disk work aka hibernate, swap needs to be bigger than memory.
lvcreate --size 10G --name swap vg1
lvcreate --size 50G --name root vg1
lvcreate --extents 95%FREE --name home vg1

pvdisplay
vgdisplay
lvdisplay

vgchange --available y
```

On the LVM, create actual filesystems.

```bash
mkswap -L "swap" /dev/mapper/vg1-swap
mkfs.ext4 -L "root" /dev/mapper/vg1-root
mkfs.ext4 -m 0 -L "home" /dev/mapper/vg1-home  # -m 0: no need to reserve space for superuser.
```

Activate and mount.
```bash
swapon /dev/mapper/vg1-swap
mount -t ext4 /dev/mapper/vg1-root /mnt/gentoo
```

Unmount the key (for now):
```bash
umount /mnt/efiboot
```

## Installing the Gentoo installation files

[Original part in Sakaki's guide](https://wiki.gentoo.org/wiki/Sakaki%27s_EFI_Install_Guide/Installing_the_Gentoo_Stage_3_Files).
[Corresponding Gentoo Handbook part](https://wiki.gentoo.org/wiki/Handbook:AMD64/Installation/Stage).

Retrieve `stage3-amd64-*.tar.xz`, `.tar.xz.DIGESTS.asc` and optionally `.tar.xz.CONTENTS` -files from Gentoo mirror.
The stage3 tarball (etc) is usually placed in `/mnt/gentoo` in the target machine, but you may also use `/tmp` for that.
Adjust the commands accordingly.

```bash
# These two commands are optional if you have the keys on the retrieving machine already.
gpg --keyserver hkps.pool.sks-keyservers.net --recv-keys 0xBB572E0E2D182910
gpg --fingerprint 0xBB572E0E2D182910

gpg --verify *.tar.xz.DIGESTS.asc  # should contain line like  gpg: Good signature from ...
sha512sum -c *.tar.xz.DIGESTS.asc  # should contain line like  stage3-amd64-*.tar.xz: OK
```

Unpack the stage3 contents:
```bash
cd /mnt/gentoo
tar xJpf stage3-amd64-*.tar.xz --xattrs-include='*.*' --numeric-owner

ls
```

### Configuring compile options

```bash
nproc  # mark the resulting number..
nano -w /mnt/gentoo/etc/portage/make.conf
```
Change `MAKEOPTS` to contain `-jNN` where `NN` is replaced with value the `nproc` printed.
Save and exit (^X).

## Building the Gentoo Base System.

[Original part in Sakaki's guide](https://wiki.gentoo.org/wiki/Sakaki%27s_EFI_Install_Guide/Building_the_Gentoo_Base_System_Minus_Kernel).

```bash
mirrorselect -i -o >> /mnt/gentoo/etc/portage/make.conf
mkdir -p /mnt/gentoo/etc/portage/repos.conf
cp -nv /mnt/gentoo/usr/share/portage/config/repos.conf /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
nano -w /mnt/gentoo/etc/portage/repos.conf/gentoo.conf
```
Change `sync-webrsync-verify-signature = true`.
Temporarily change `sync-type = webrsync`.
Optionally, change the `sync-uri` to some closer one; see choices in [Gentoo rsync mirrors](https://gentoo.org/support/rsync-mirrors/) page.
Save and exit (^X).

```bash
cp -v -L /etc/resolv.conf /mnt/gentoo/etc/

mount --types proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
```

Enter to chroot environment.

```bash
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"
```

### Updating Portage tree

First step update, a coarse approach (using webrsync as set in the configuration):
```bash
emaint sync --auto
```
The command should output

> gpg: Good signature from "Gentoo ebuild repository signing key (Automated Signing Key)"

```bash
nano -w etc/portage/repos.conf/gentoo.conf
```
Change `sync-type = rsync` back.
Save and exit (^X).

Final step, update the tree to current (using actual rsync as set in the configuration):
```bash
emaint sync --auto
```
The command should still output `Good signature …`.

Read news. Or skip them.
```bash
eselect news list
eselect news read
```

### Timezone, locale and keymap

```bash
ls /usr/share/zoneinfo
echo "Europe/London" > /etc/timezone
emerge -v --config sys-libs/timezone-data

nano -w /etc/locale.gen
locale-gen
eselect locale set C
env-update && source /etc/profile && export PS1="(chroot) $PS1"

ls /usr/share/keymaps/i386/qwerty
nano -w /etc/conf.d/keymaps
```

### CPU flags

```bash
emerge -v1 app-portage/cpuid2cpuflags
cpuid2cpuflags | sed -e 's/: /="/;s/$/"/;' >> /etc/portage/make.conf
nano -w /etc/portage/make.conf  # just to check...
```

### System profile

```bash
eselect profile list
# Optionally correct it with (replacing NN with either the profile number or the actual name)
eselect profile set NN

emerge -avuDN --with-bdeps=y @world
# -a = --ask, -v = --verbose, -u = --update, -D = --deep, -N = --newuse
# bdeps = build-time dependencies.
```


## Kernel

```bash
emerge -av gentoo-sources
eselect kernel list  # find correct kernel number, probably 1 since this should be first installed kernel ever.
eselect kernel set X
```


### "Required" options

Following list is scraped and slightly modified from Sakaki's scripts. Not all may be required, YMMV, and some are even pre-selected.
Use `cd /usr/src/linux && make menuconfig` to edit (currently selected) kernel config.

```
GENTOO_LINUX
GENTOO_LINUX_UDEV
GENTOO_LINUX_INIT_SCRIPT
GENTOO_LINUX_INIT_SYSTEMD

SECCOMP
DMIID
TMPFS_POSIX_ACL
AUDIT
AUDITSYSCALL
UEVENT_HELPER_PATH=""
CMDLINE_BOOL=y
# Following setting is wrapped for readability, but should actually be a single line, without comments, and variables expanded.
CMDLINE="
    root=/dev/ram0
    crypt_root=/dev/disk/by-partuuid/$CRYPTPARTUUID
    dolvm
    real_root=/dev/mapper/vg1-root
    rootfstype=ext4
    real_init=/lib/systemd/systemd  # (or /usr/lib/systemd/systemd)
    root_keydev=/dev/disk/by-partuuid/$EFIPARTUUID
    root_key=luks-key.gpg
    real_resume=/dev/mapper/vg1-swap
    keymap=fi
    udev.log-priority=3  # hide systemd version number in early boot
"
INITRAMFS_SOURCE="/boot/initramfs.cpio"
INITRAMFS_COMPRESSION_XZ


CRYPTO_SERPENT
CRYPTO_WP512
CRYPTO_SHA512
VFAT_FS
BLK_DEV_DM
DM_CRYPT

IKCONFIG       # .config support
IKCONFIG_PROC  # /proc/config
PARTITION_ADVANCED
EFI_PARTITION
EFI
EFIVAR_FS
EFI_VARS
EFI_STUB

RTC_CLASS
RTC_HCTOSYS
RTC_SYSTOHC
RTC_HCTOSYS_DEVICE="rtc0"
RTC_INTF_SYSFS
RTC_INTF_PROC
RTC_INTF_DEV
RTC_DRV_CMOS

SUSPEND
HIBERNATION

SYSFS_DEPRECATED=n
```

### Tools

Install efitools, efibootmgr and genkernel-next.

```bash
mkdir -p -v /etc/portage/env
mkdir -p -v /etc/portage/package.{env,keywords,use}

echo 'MAKEOPTS="-j1"' >> /etc/portage/env/serialize-make.conf
echo 'app-crypt/efitools serialize-make.conf' >> /etc/portage/package.env/efitools
echo 'app-crypt/efitools' >> /etc/portage/package.keywords/monolithic
emerge -av app-crypt/efitools efibootmgr

echo 'sys-kernel/genkernel-next cryptsetup gpg' >> /etc/portage/package.use/monolithic
emerge -av sys-kernel/genkernel-next
```


### Firmware

Install linux-firmware (and other firmware) as needed:
```bash
emerge -av linux-firmware
```

*Caveat:* If `savedconfig` use-flag is not set or you don't have already present `/etc/portage/savedconfig/sys-kernel/linux-firmware*`-file,
the resulting initramfs and kernel will be significantly larger.
Including all firmware files is a safe choice however.

After the package has been installed first time, the saved config file mentioned above can be edited to contain only
selected files, and after that enabling `savedconfg`-useflag installs only the selected files
and thus genkernel will include only them to the initrd image.
Genkernel has a parameter to include only given firmware files, but when this documentation was written the flag behaved incorrectly.


### Initramfs and Kernel

Create initramfs with self-supplied gpg.
The gnupg-1.4 is not actually installed, but created as a package.
This allows us to have >=gnupg-2 installed on the actual system while simultaneously using the old gnupg in the initrmafs.

```bash
echo "=app-crypt/gnupg-1.4* static" >> /etc/portage/package.use/monolithic
emerge -avB =app-crypt/gnupg-1.4*
# -B = --buildpkgonly

# Create initial initrd.
cd /usr/kernel/linux
make -j4 modules  # Adjust -j4 as needed.
genkernel --no-install --no-mountboot --luks --lvm --no-gpg --udev --busybox \
    --no-compress-initramfs --all-ramdisk-modules --firmware \
    --no-postclear initramfs

# Patch the initrd to contain the old pre-built gnupg 1.4 binary.
mkdir -p /tmp/initrd
cd /tmp/initrd
tar -xf /usr/portage/packages/app-crypt/gnupg-1.4*.tbz2 ./usr/bin/gpg
cp /var/tmp/genkernel/initramfs-* .
echo usr/bin/gpg | cpio -H newc -o --append -F initramfs-*
mv $(echo initramfs-*){,.cpio}  # or append .cpio to the filename.

# Move patched initrd back.
mv initramfs-* /var/tmp/genkernel
```

Fix new initrd to be used: change INITRAMFS_SOURCE variable to contain the filepath to .cpio archive created above.

```bash
cd /usr/src/linux
ls /var/tmp/genkernel/initramfs-*.cpio
make menuconfig
```

Build final (pre-secured) kernel, copy it to the key drive and install a boot entry to EFI.

```bash
make -j4 && make modules-install  # Adjust -j4 as needed.
mkdir /mnt/efi
mount /dev/sdYn /mnt/efi  # Replace sdYn with the EFI USB stick device.
mkdir -p /mnt/efi/EFI/Boot
cp -aLvn arch/x86_64/bzImage /mnt/efi/EFI/Boot/bootx86.efi && sync
mount -o remount,rw /sys/firmware/efi/efivars
```

#### Script (for future)

You may want to use [makeefi script](makeefi) to build initrd and kernel in future with less typing.
It does not prepare the gpg ebuild nor the boot key, so it is not too useful for first install.
You should check its `MAKEOPTS` option configation found in top of the script.

To install the script, copy it either to `/usr/local/sbin/`, `/usr/src/` or `/root/`
(only first of these is available in PATH).
The script must be run by root within kernel source directory, so usually in `/usr/src/linux/`.

The three code blocks in above section exist in the script as "parts".
They can be run separately like `makeefi 1` or `makeefi 2`, or together like `makeefi 1 2 3`

The script keeps (only) one backup kernel in the USB key.
If you need to use it (due broken kernel config or other), you may be able to use UEFI Shell (if your BIOS provides such) to boot directly to the old kernel.
The EFI shell command is something like `FS0:\EFI\Boot\bootx86-old.efi`, see available drives (like `FS0`) with `map` command.
Otherwise you need to manually rename the `bootx86-old.efi` to `bootx86.efi`,
which may require a separate machine or use of fallback OS...


### EFI

```bash
# List current EFI boot menu
efibootmgr -v
# Note sdY is the disk, not partition. Also note backslashes in loader argument, instead of regular slashes.
efibootmgr --create --disk "/dev/sdY" --part 1 --loader '\EFI\Boot\bootx64.efi' --label "Gentoo Linux (USB)"
# See if the entry became automatically active (has asterisk (*) after `BootNNNN`):
efibootmgr -v
```

If not, activate it with `efibootmgr -a NNNN`  where NNNN is the boot entry index.
Also check BootOrder from above listing. It should have the newly created index as first.
If not, adjust the order with `efibootmgr -o NNNN,OOOO,PPPP,QQQQ` replacing the quartets with the new boot index,
and original list of entries (without the new index).

```bash
umount /mnt/efi
```

## Finalizing initial installation

Find out partuuid of the EFI USB stick to be used in fstab: `lsblk -dso PATH,PARTUUID,FSTYPE`
Add required entries to fstab (replace `XXXX-XXX` with the EFI USB stick partuuid):
```text
/dev/mapper/vg1-root    /       ext4    defaults,noatime,errors=remount-ro,discard      0 1
/dev/mapper/vg1-swap    none    swap    defaults,noatime,discard        0 0
/dev/mapper/vg1-home    /home   ext4    defaults,noatime,discard        0 2
/dev/disk/by-partuuid/XXXX-XXX  /mnt/efi vfat defaults,noauto           0 2
```

```bash
emerge -av net-misc/dhcpcd dosfstools
passwd root
```

Onto reboot. `exit` from chroot. Then

```bash
swapoff /dev/mapper/vg1-swap
umount -lv /mnt/gentoo/dev{/shm,/pts,}
umount -Rv /mnt/gentoo
vgchange --available n
cryptsetup luksClose gentoo

reboot
```

### First boot on new system

The USB key should boot automatically when inserted, and UEFI bios should fallback to other previously available boot
options when the key is not inserted in boot.

The initrd should ask for password for the encrypted volume, after which the boot will continue to bringing systemd up.

After logging in with `root` account:

```bash
# Remove now unneeded base system image.
rm /stage3-amd64-*

# Add regular user.
useradd -m -G users,wheel,audio,video -s /bin/bash ...  # Replace ... with your regular username.
passwd ...  # Same here.
```


## Live Boot chroot

In case you need to re-enter the chroot environment from fresh live boot session:

```bash
mkdir -p /mnt/efiboot
mount /dev/sdY1 /mnt/efiboot
gpg --decrypt /mnt/efiboot/luks-key.gpg | cryptsetup --key-file - luksOpen /dev/sdZn gentoo
umount /mnt/efiboot
vgchange --available y

swapon /dev/mapper/vg1-swap
mount -t ext4 /dev/mapper/vg1-root /mnt/gentoo

mount -t proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev

chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) ${PS1}"
```

and later, after `exit`:
```bash
swapoff /dev/mapper/vg1-swap
#umount -lv /mnt/gentoo/home
umount -lv /mnt/gentoo/dev{/shm,/pts,}
umount -Rv /mnt/gentoo
vgchange --available n
cryptsetup luksClose gentoo
```
