# No Space Left on Disk

truncate -s size

creates sparse file if too big and supported

# Storage Models and Disks

OS, Storage, Filesystem, Application software

## Direct Attached Storage (DAS)

A physical drive is attached to the computer.

Advantages
- failure due to network is impossible
- performance penalty due to network latency is impossible

Disadvantages
- isolated from other devices

## Network Attached Storage (NAS)

File server - has filesystem
Each client - uses NFS
NFS and filesystem of file server help abstract over the network.




data stored on a central location.

Network Filesystem provides the abstractions to communicate to NAS

allows multiple clients to access the same fs requiring the NFS interface.

## Storage Area Network (SAN)

High performance networks specifically dedicated to the management of data storage

We want to scale up and allow different clients to interact with the storage.

Central storage media is accessed via interfaces like FCP, FCoE, iSCSI

These appear local on device as if it was Direct attached storage

It is a Network. You can build infrastructure with fully-switched fabrics.

## Cloud Storage

file level (DropBox, GDrive, iCloud)
object level (AWS S3)
block level (AWS EBS)

S3 vs EBS

```sh
aws s3 mb s3://path
aws s3 cp --recursive --exclude "pattern" html s3://path
aws s3 ls --human-readable s3://path
aws s3 rb --force s3://path

aws ec2 create-volume --size 4 --availability-zone us-east-1a # size in GB
```

# Device Interfaces

SCSI - used to be the standard for connecting devices, with ribbon cables.

Obsoleted by Advanced Technology Attachment (ATA) standards.

Variation: Serial Attached SCSI (SAS)

The ATA standard is often equated with Integrated Device Electronics interface (IDE).

Nowadays, Serial ATA (SATA) is more common.

Solid state drive (SSD)

Fibre Channel Protocol - used in switched fabric.

These protocols may be used on top of one another.
SCSI might be above everything, RDMA (remote direct memory access) over infiniband

ATA over Ethernet (AoE)
- create low-cost SAN
- ATA encapsulated into Ethernet frames
Fibre Channel over Ethernet (FCoE)
- consolidate IP and FC/SAN networks
- FC encapsulated into Ethernet frames

*oE (anything over Ethernet)
- no TCP/IP overhead
- restricted to Layer 2
- no inherent security features

iSCSI
- includes authen
- SCSI encapsulated in TCP/IP

Serial Attached SCSI (SAS)
- Backwards compatible with SATA
- 6-layered architecture comprised of three protocols
  - Serial SCSI Protocol (SSP)
  - Serial ATA Tunneling Protocol (STP)
  - Serial Management Protocol (SMP)

## Capacity

18TB is about $600 dollars as of the video

JBOD - just a bunch of disks

RAID

100TB SSD is about $40k

Flash Arrays - combining high performance flash memory

# Storage Virtualization

separating physical and logical storage - physically backed by e.g. disk array

Storage device-based: RAID

Host-based: device mapper, logical volume management, ZFS

## RAID

Allow filesystems to be larger than a physical disk

increase I/O performance when striped

fault tolerant when mirrored or plexed

RAID 0 Striped array (block level), no parity or mirroring
RAID 1 Mirrored array, no parity or striping
RAID 2 Striped array (bit level) with dedicated parity
RAID 3 Striped array (byte level) with dedicated parity
RAID 4 Striped array (block level) with dedicated parity
RAID 5 Striped array (block level) with distributed parity
RAID 6 Striped array (block level) with double distributed parity

RAID 0+1 Mirrored array of stripes
RAID 1+0 Striped array of mirrors
RAID 5+0 Striped array of multiple RAID 5

## Logical Volume Management

Divide physical storage units into physical volumes (PVs)

Pool physical volume groups into logical volumes (LVs)

Combine and create logical volumes to allow for
- combining/concatenating individual disks (JBOD)
- hot swapping
- dynamically resizing filesystem
- adding RAID functionality across logical volumes
- implicit backup via snapshots

LVM
dev/mapper is the LVM manager in Linux

lsblk shows info on which is managed by lvm

`swapon --show` shows device mapper again

## ZFS

`format` shows partition table on omnios

df shows disk is on rpool

`zpool list -v` shows zfs pool

`zfs list` shows the filesystems created on the pool

To add new disks use `aws ec2 create-volume ...`

After attaching 2 new drives, we can use zfs to incorporate those drives.

`diskinfo` shows disks

`zpool create <pool name> <disk names>`

There is overhead involved. 2GB -> 1.88GB

`zfs create extra/space` to create a new fs on the pool extra
`zfs set mountpoint=/mnt extra/space`

`zpool add extra <disk name>` adds a new disk to an existing pool extra.

# Physical Disk Structure

Tracks mark distance from center

Each Track is divided into sectors

512 byte block size is an actual hardware limit

Zone bit recording - more sectors on outer zones

Hard Drive Performance
- Transfer rate
- Seek Time - time for movement of arm
- Rotational Latency - on average, half a rotation
- a few other negligible factors (external data rate, command overhead, access time, etc.)

Hard Drive Capacity

Cylinder-Head-Sectors scheme (CHS)

ATA Spec
- 64k cylinders,
- 16 heads,
- 255 sectors per track,
- 512 bytes per sector
- -> 137GB of maximum storage

BIOS limit
- 1024 cylinders,
- 256 heads,
- 63 sectors per track,
- 512 bytes per sector,
- -> 8.5GB of maximum storage

Combined (take minimum of each)
- 1024 cylinders,
- 16 heads,
- 63 sectors per track,
- 512 bytes per sector,
- -> 528MB

Logical Block Addressing (LBA)
- 28-bit LBA in ATA-1: 2^28 * 512 = 137GB
- 48-bit LBA in ATA-6: 2^48 * 512 = 144PB

- MBR uses 32-bit sectors: 2^32 * 512 = 2.1TB
- GPT uses 64-bit sectors: 2^64 * 512 = 9.4ZB

# Partitions

How are partitions distributed on the physical disk?

To avoid seeking, we use cylinders

/boot is usually at the beginning of the disk. This is the outermost cylinder.

## Editing partition table on NetBSD
```sh
disklabel -i xbd1
partition>a
Filesystem type [4.2BSD]:
Start offset ...: 63s
Partition size ...: 100M
...

partition>b
...

partition>e
...

partition>p
...

partition>W # write
...
partition>Q # quit
```
on BSD: partition c is usually reserved for the os and partition d describes the whole disk

## Editing partition table on OmniOS
```sh
format c1t5d0
format> fdisk
...
format> verify # print out the table
...
format> partition
partition> 0 # change partition 0
... boot
... flags[wm]
... cyl[0]
... 100mb

...
partition> print # prints the table
...
partition> label # save table
partition> quit
format> quit
prtvtoc /dev/dsk/c1t5d0
```
partition 2 describes the whole disk
partitions 8 and 9 are reserved
prtvtoc is print volume table of contents

## Editing partition table on Fedora Linux
```sh
sudo fdisk -l ...
sudo sfdisk /dev/xvdf
>>> , 100MB
>>> , 128MB, S
>>> , 256MB
>>> ,
... # save
```

reason to use partitions
- separate user and system data
- separate boot partition from the rest
- wish to use different types of filesystems
- wish to apply different mount flags

---

# Weekly Checkpoint

> What is the difference between NAS and a SAN?

Network Attached Storage (NAS) provides a way to store data and make it
available on the network. Then, one can use the Network File System to
interface with this storage.

Storage Area Network (SAN) is a high performance network specifically
dedicated to the management of data storage. There are a variety of interfaces
to the central storage media: FCP, FCoE, iSCSI. These appear local on device
as if it was Direct Attached Storage.

> What kind of storage device do the EC2 instances you've created use?

An EBS volume is attached to the EC2 instance. It virtualizes physical volumes.

> What types of disk interfaces do you have experience with?

I don't really have much knowledge or experience with disk interfaces. I'm not
sure what is expected here.

> Name three different types of file systems. Give one example for each.

File Allocation Table (FAT) - FAT32
File System in User Space (FUSE) - SSHFS (after some googling)
Extended File System (EXT) - ext4

> What is an inode? What information is not found in an inode?

An index-node (inode) is a data representation of a file/directory in the disk.
In the case of a directory, it contains a mapping of children filenames to
inodes. It also stores file size, link count, access mode, etc. Indirect inode
blocks can be used to support larger files.

inodes do not contain the name of the file. That is mapped in the inode of the
parent directory.

> Please note any particular (sub)topics or aspects of this week's materials that you would like me to revisit in class:

What is a "type" of filesystem?

---

# Class Notes

UTC or GTFO

load averages are average number of tasks in queue for last
1, 5, 15 minutes.

IPv6
- Internal: fe80
- Public: 2600

There are other filesystems that may not show up on df but do on mount.

/etc/fstab shows the filesystems mounted at boot time

check `dmesg | grep dk1` for the actual logical disk.

use disklabel on netbsd for partitions

ioctl - io control
DIOCGDINFO -

what is the difference between dk1 and ld4?
- dk(4) is a disk partition driver
- ld(4) is a logical disk driver
dk was introduced almost 2004
- wedges are an abstraction of hardware partitions

/dev/ld4 grants us access at the block level
/dev/rld4 grants us access at the character level

manpages
1 - command
2 - syscalls
3 - library functions
4 - kernel interfaces
5 - misc/confs
6 - games
7 - admin config
8 - admin

label is "ficticious" if disklabel doesn't get the real disk label

How do SSD blocks work?
- flash memory stores information in NAND memory cells by adding or
  removing electrons
- cells are combined into strings (32 - 128 cells), pages (8KB),
  blocks (32, 64, 128 pages), planes (1024 blocks), and dies
- controller exposes e.g. a SATA or PCIe interface
- logical blocks (e.g. 512B, 4KB) from the higher level system (using LBA) are mapped
  into physical pages (8KB) and blocks via the Flash Translation Layer (FTL)
- logical addressing is now random; I/O requests may be queued
- wear-leveling is used to balance finite program-erace cycles
- many filesystems are optimized for classical disks; specialized filesystems may
  optimize for flash memory (e.g. F2FS)

---

Do the following

Exercise 1: LVM
- attach 2 volumes to aws ubuntu
- initialize volumes for use by lvm using pvcreate(8)
  sudo pvcreate /dev/nvme1n1
  sudo pvcreate /dev/nvme2n1
- create a new volume group using vgcreate(8)
  sudo vgcreate cs615 /dev/nvme1n1 /dev/nvme2n1
  lsblk
  sudo lvmdiskscan -l
- create a new logical volume using lvcreate(8)
- create a new filesystem on the new volume
- mount the filesystem
- ...

Exercise 2: ZFS
- start a new FreeBSD instance
- create two new volumes
- attach volumes to the instance
- create a new pool using zpool(8)
- create another volume and attach it
- add the new disk to the existing pool

wow, freebsd filesystem has compression enabled.
10G of /dev/zero is compressed into 96K


Do the Week 03 Resize Exercise
