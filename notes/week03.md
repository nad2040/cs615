# Boot Process and MBR

Kernel outputs log messages in green and eventually passes control over to init.

Booting:
- power on hardware
- POST (power-on self-test) and other hardware initialization
- first stage boot loader (like MBR)
- second stage boot loader (like GRUB)
- kernel

BIOS (Basic Input/Output System)

UEFI (Unified Extensible Firmware Interface)

## Master Boot Record
BIOS expects to find the Master Boot Record in the first 512 byte block.

Bytes 510 and 511 are 0x55AA

Bytes 446 to 509 (64 bytes) hold the partition table
- This is the BIOS partition table, not the OS partitions.
- each BIOS partition entry is 16 bytes in size, so only 4 BIOS partitions can exist.
  - each OS then chooses what to do within those partitions.

All other 446 bytes are used for the stage 1 boot loader code.
- stage 1 may reach into other sections of first track or jump to boot code in OS partitions for
  stage 2 boot loaders.

Partition Table Entry Layout
0x00: Status Active (0x80)
0x01: 3 byte CHS (cyl head sector addressing scheme) of first sector
  1 byte head, 2 bits (high bits of cylinder addr), 6 bits sector, 8 bits cylinder addr (10 bits for cyl total)
0x04: Partition Type
0x05: 3 byte CHS addr of last sector
0x08: 4 byte LBA of first sector
0x0c: 4 byte LBA of last sector

LBA -> CHS:
C, rem = LBA divmod (heads per cyl * sectors per track)
H = rem / sectors per track
S = rem % sectors per track + 1

as root:
`dd if=/dev/xbd0 count=1 2>/dev/null | hexdump -C | more`
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

A logical partition has a disklabel describing the geometry of the disk
and the filesystem partitions

A filesystem partition is a collection of cylinder groups, on which you
can create a new filesystem

We need to describe the cylinder groups in each filesystem, so we store that
in the superblock, along with metainformation about the filesystem itself.

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
# first had to reboot!!!
# aws ec2 reboot-instances --instance-ids

disklabel -e ld5
newfs /dev/ld5a
```

inode #2 is the root directory of the disk

total number of inodes is fixed at filesystem creation time.

tuning of filesystems can be done to support more tiny files for example

check out fs(5) `man 5 fs`

# Files go hier(7)

on omnios:
mnttab
man vfstab
more /etc/vfstab

on fedora linux:
/etc/fstab
/etc/mtab -> /proc/self/mounts

on NetBSD:
`man hier`

Exercises:
- look at mount options
- when/why would the list of mounted filesystems != the list in /etc/fstab

---

# Checkpoint

> What is the difference between a BIOS partition and an operating system partition?

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

> When creating a file system, what (if any) aspects or properties are fixed?

The number of inodes.

> Is there a way to grow or shrink a file system? Think about how that would be implemented.

If the BIOS partition doesn't take up all the sectors until the beginning of
the next partition, you could probably increase the size. However, the
filesystem in that partition would need to support resizing. Software like LVM
can abstract over this and allows you to grow or shrink logical volumes.


