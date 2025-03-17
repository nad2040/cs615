# Boot Process and MBR

Kernel outputs log messages in green and eventually passes control over to
init.

Booting:
- power on hardware
- POST (power-on self-test) and other hardware initialization
- first stage boot loader (like MBR)
- second stage boot loader (like GRUB)
- kernel

BIOS (Basic Input/Output System)

UEFI (Unified Extensible Firmware Interface)

## Master Boot Record BIOS expects to find the Master Boot Record in the first
512 byte block.

Bytes 510 and 511 are 0x55AA

Bytes 446 to 509 (64 bytes) hold the partition table
- This is the BIOS partition table, not the OS partitions.
- each BIOS partition entry is 16 bytes in size, so only 4 BIOS partitions can
exist.
  - each OS then chooses what to do within those partitions.

All other 446 bytes are used for the stage 1 boot loader code.
- stage 1 may reach into other sections of first track or jump to boot code in
OS partitions for stage 2 boot loaders.

Partition Table Entry Layout 0x00: Status Active (0x80) 0x01: 3 byte CHS (cyl
head sector addressing scheme) of first sector 1 byte head, 2 bits (high bits
of cylinder addr), 6 bits sector, 8 bits cylinder addr (10 bits for cyl total)
0x04: Partition Type 0x05: 3 byte CHS addr of last sector 0x08: 4 byte LBA of
first sector 0x0c: 4 byte LBA of last sector

LBA -> CHS: C, rem = LBA divmod (heads per cyl * sectors per track) H = rem /
sectors per track S = rem % sectors per track + 1

as root: `dd if=/dev/xbd0 count=1 2>/dev/null | hexdump -C | more`
- NOTE: not working on new NetBSD ec2 because of GPT Protective MBR?

`dd if=/dev/xbd0 bs=1 count=16 skip=446 2>/dev/null | hexdump -C | more`

... manually writing bytes to the disk to set up partition table

When writing the LBA, it is least significant byte first!!!

# Filesystem

Creating by hand a naive custom fs.

`tar` as a filesystem format

# UNIX Filesystem

Implemented in version 7 and superseded by Fast File System (BSD ffs)

A disk can be divided into logical partitions (e.g. as in the MBR)

A logical partition has a disklabel describing the geometry of the disk and the
filesystem partitions

A filesystem partition is a collection of cylinder groups, on which you can
create a new filesystem

We need to describe the cylinder groups in each filesystem, so we store that in
the superblock, along with metainformation about the filesystem itself.

If the partition is a boot partition, it might also include boot blocks.

Each cylinder group has the data blocks:
- superblock copy
- inode map
- block bitmap
- inode blocks
- data blocks

inode data blocks
- inode data / file metadata
- data block references

file data blocks
- actual data in files

```sh
# first had to reboot!!! # aws ec2 reboot-instances --instance-ids
disklabel -e ld5 newfs /dev/ld5a
```

inode #2 is the root directory of the disk

total number of inodes is fixed at filesystem creation time.

tuning of filesystems can be done to support more tiny files for example

check out fs(5) `man 5 fs`

# Files go hier(7)

on omnios: mnttab man vfstab more /etc/vfstab

on fedora linux: /etc/fstab /etc/mtab -> /proc/self/mounts

on NetBSD: `man hier`

Exercises:
- look at mount options
- when/why would the list of mounted filesystems != the list in /etc/fstab

---

# Checkpoint

> What is the difference between a BIOS partition and an operating system
> partition?

BIOS partitions are declared in the Master Boot Record. There are only 4 such
partitions. Inside each BIOS partition, you can install a filesystem, and
within each filesystem, you can make a new operating system partition on top of
a cylinder group.

> How does a server boot up? After powering on a server, what happens next?

First, the hardware powers on and BIOS does a power-on self-test and other
hardware initialization. Then the first stage boot loader is run and that often
then calls a second-stage boot loader. Finally, the kernel of your OS is called
and run.

> Why are there both primary and secondary boot loaders?

The primary boot loader chooses between the BIOS partitions. Since it is
limited by size, it does the bare minimum setup necessary to boot the system.
Then it passes control over the secondary boot loader which actually boots the
OS.

The second stage bootloader is used to do more stuff because the primary bootloader
can only do very little.

> When creating a file system, what (if any) aspects or properties are fixed?

The number of inodes.

> Is there a way to grow or shrink a file system? Think about how that would be
> implemented.

If the BIOS partition doesn't take up all the sectors until the beginning of
the next partition, you could probably increase the size. However, the
filesystem in that partition would need to support resizing. Software like LVM
can abstract over this and allows you to grow or shrink logical volumes.

---

GPT layout

Protective MBR is used to protect legacy tool. It only sees the 32-bit address space

Primary GPT and Backup GPT
- The backup GPT is a 1:1 mirror/backup of the primary GPT

CRC32 - cyclical redundancy check - determines if something was corrupted.

GUID - Global Unique Identifier

---

# [Netbooting](https://www.intel.com/content/www/us/en/developer/articles/technical/network-boot-in-a-zero-trust-environment.html)

BIOS / UEFI set to boot via Preboot eXecution Environment (PXE)

Requires support in Network Interface Card (NIC) firmware

Bootstrap Protocol (BOOTP) (RFC951, 1985); then Dynamic Host Configuration Protocol (DHCP) (RFC2131, 1997)

DHCPDISCOVER broadcast on UDP port 67

DHCPOFFER unicast to UDP port 68 (includes bootserver IP, name of boot file)

Downloads Network Bootstrap Program (NBP) using Trivial File Transfer Protocol (TFTP) (1981, now RFC1350) to UDP port 69

VERY INSECURE! No authN.

[iPXE]() is a modern alternative that is secure

HPC netboots; if you don't want to store files
RAM boot

The key that a TPM holds cannot be taken without being broken

---

HW1

Available -28. How?
- there is a certain amount of bytes reserved for the root user?
- more data can go into separate files
- dd writes in multiples of 512 blocks
- `tunefs -N /dev/ld5a` minimum percentage of free space 5% (NetBSD)
  - this disk space is reserved
  - it gives root a chance to check logs
  - it lets users still log in
  - could be for performance
  - %cap shows user capacity (NetBSD)
  - the filesystem has some optimized metadata layout and it can still write
    more data by stuffing it with the file metadata.

disk size vs filesystem useable blocks
- usually you lose 10% capacity due to filesystem itself


```sh
    1  lsblk
    2  fdisk /dev/nvm1n1
    3  fdisk /dev/nvme1n1
    4  sudo fdisk /dev/nvme1n1
    5  sudo mkfs /dev/nvme1n1p1
    6  sudo mkfs /dev/nvme1n1p2
    7  mount /dev/nvme1n1p1 /mnt
    8  sudo mount /dev/nvme1n1p1 /mnt
    9  df /mnt
   10  sudo echo foo | sudo tee /mnnt/file
   11  sudo echo foo | sudo tee /mnt/file
   12  ls
   13  ls /
   14  ls /mnt
   15  lsblk
# Expanding filesystem
   17  sudo fdisk /dev/nvme1n1
   18  umount /mnt
   19  sudo umount /mnt
   20  sudo fdisk /dev/nvme1n1
   21  lsblk
   22  mount /dev/nvme1n1p1 /mnt
   23  sudo mount /dev/nvme1n1p1 /mnt
   24  mkfs /dev/nvme1n1p1
   25  sudo resize2fs /dev/nvme1n1p1
   26  sudo e2fsck -y /dev/nvme1n1p1
   27  sudo mount /dev/nvme1n1p1 /mnt
   28  ls /mnt
   29  history
   30  df -h
   31  sudo umount /mnt
   32  sudo resize2fs /dev/nvme1n1p1
   33  sudo e2fsck -f /dev/nvme1n1p1
   35  sudo resize2fs /dev/nvme1n1p1
   36  sudo mount /dev/nvme1n1p1 /mnt
   37  df -h
# Shrinking filesystem (practically opposite order)
   39  sudo umount /mnt
   40  sudo e2fsck -f /dev/nvme1n1p1
   41  sudo resize2fs /dev/nvme1n1p1 256M
   42  lsblk
   43  sudo fdisk /dev/nvme1n1
   44  lsblk
   45  sudo mount /dev/nvme1n1p1 /mnt
   46  sudo e2fsck -f /dev/nvme1n1p1
   47  sudo mount /dev/nvme1n1p1 /mnt
   48  df -h /mnt
   49  history
```

Experiment to find the answers:

how many files can you create?
- inode count. look for inode density

how many entries can a single directory have?

how large can a single file be?
- look for filesystem limitations

what characters can a filename contain?
- all sorts of characters

how long is the longest file name?

how deep can a pathname be?

how many links can a file have?
