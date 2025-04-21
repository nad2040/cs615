# System Tools Checkpoint

> What programming languages are you confident in?

Python, C, Java

> Approximately how many KLOC was the largest program you wrote?

2-4KLOC.

> How many people have you closely collaborated with on a single software project?

20 maybe.

> What is the difference between sh(1) and bash(1)? What about csh(1), ksh(1), or zsh(1)?

sh is the original bourne shell that is POSIX-compliant. bash is the
bourne-again shell. bash adds features like arrays to the language. Such
features might not be POSIX-compliant.

On some systems, /bin/sh is a symlink to another shell like dash or
bash (on MacOS).

csh, ksh and zsh are all different shells. They all have different features.
Not sure what's being asked of here.

> Without looking it up online, write a regular expression to match an IPv4 address and one to match any one word.

I think it's:
`(25[0-5]|2[0-4]\d|1\d\d|[1-9]\d|\d)\.(25[0-5]|2[0-4]\d|1\d\d|[1-9]\d|\d)\.(25[0-5]|2[0-4]\d|1\d\d|[1-9]\d|\d)\.(25[0-5]|2[0-4]\d|1\d\d|[1-9]\d|\d)`

word characters are `\w` iirc
`\w+`

> Run the command 'echo import this | python'.
> Which of these "rules" resonates most with you? Do they apply to Python only or more than in other languages?

In the face of ambiguity, refuse the temptation to guess.

This phrase resonates the most with me as it relates to how I feel that we all
need to make our implicit assumptions explicit. I believe that doing it helps
with communication, learning, and problem solving. I still forget sometimes,
but I try to remember.

No these rules don't only apply to Python. They are pretty general programming
style advice.

# Backups Notes

Backups are a means to accomplish the goal of being able to restore data.

Full backup - entire snapshot
Incremental backup - diff since last backup
Differential backup - diff since last full backup

file-level or block-level

metadata, file data, live data / open files

journaling vs snapshot filesystems

business terms:
Recovery Point Objective (RPO) / Recovery Time Objective (RTO)
- RPO is the fault tolerance window you are willing to accept. when is the last backup
- RTO is how long it takes to restore data

Business Continuity Plan (BCP)

Full backup:
- very slow
- lots of storage / bandwidth
- fast full recovery

Differential backup:
- improved backup performance
- better storage / bandwidth utilization
- slower restore
- at most two datasets required for recovery

Incremental backup:
- highest backup performance
- best storage / bandwidth utilization
- slowest restore
- higher risk of failed recovery

Data storage media and properties
- magnetic tape
- hard disk
- ssd
- cloud

- i/o performance
- reusability and degradation
- longevity
- data integrity (write once, read many - WORM)
- data compression and encryption
- deduplication

when do we need backups?
- long-term storage / archival
  - complete backup
  - separate set from regular backups
  - usually stored off-site
  - recovery / retrieval takes time
  - limited granularity
  - storage media considerations
  - backup encryption and recovery key management
- recover from data loss due to:
  - user failure
  - software failure
  - equipment failure
  - security breach
  - natural disaster
- Type of Restore:
  - individual file(s)
  - individual system recovery
  - disaster recovery

loss of e.g. entire file system
leads to downtime (of individual systems)
RAID may help
takes long time to restore
may require retrieval of archival backups from long-term storage
often involves some data loss
3-2-1 Rule:
- keep at least 3 copies of your data
- keep at least 2 copies on different storage media
- keep at least 1 copy offsite

disasters could scale faster than your backup strategy

Trusting backups:
- backups require superuser
- if data is corrupt, backup might become corrupt
- to restore data, you can only use trusted tools
- verify the authenticity and integrity of your backups

## Backups by Example

(Accidentally) deleted files ought to be recoverable for a certain amount of time:
- "Undo"
- Recovery Point Objective (time window and granularity requirements)
- Recovery Time Objective (restore time), including:
  - staff availability
  - waiting until resources permit the restore
  - actual time spent restoring
- Self-service restore

But note: sometimes people do want to delete data and it be gone!

1. tar(1)
```sh
tar cvf - . | tar xf - -C /backup/date/ # backup to local disk
tar cvf - . | ssh ip "dd of=/dev/xbd1" # backup to remote into a block device
dd if=/dev/xbd1 | tar tvf - # read backup on remote system
ssh ip "dd if=/dev/xbd1" | tar xvf - -C /usr/local # restore backup from remote

# add compression and encryption
tar cvf - . |
    bzip2 -c |
    openssl enc -aes-256-cbc -pass env:PASSWORD |
    ssh ip "dd of=/dev/xbd1"

ssh ip "dd if=/dev/xbd1" |
    openssl enc -aes-256-cbc -pass env:PASSWORD -d |
    bzip2 -d |
    tar xvf - -C /usr/local
```

2. dump(8) - fs backups - supports full and incremental backups
```sh
dump -u -0 -f - / | ssh ip "cat >backup.0" # full backup, level 0
cat /etc/dumpdates # tracks which level backup was created

dump -u -i -f - / | ssh ip "cat >backup.i" # incremental backup
ssh ip "cat backup.i" | restore xvf - /usr/local # restore from incremental backup

# restore in interactive mode by copying the dump locally
restore if /path/to/dump
> what # display info about dump
> ? # help
> ls # list contents
> add . # restore this data
> extract # actually restore
> setmodes # set the permission
> quit
```

3. rsync(1) - syncs a directory hierarchy to another location
```sh
rsync -e ssh -azv . ip:/backup/.

# ensure remote syncs file deletion
rsync -e ssh -azv --delete . ip:/backup/.

# how do you roll back deletion?
```

## Summary
- tar(1) creates an archive of a filesystem hierarchy
- data can be written to a file (e.g., tar), a raw block device (e.g., via | dd), a remote system (e.g., via | ssh), or any combination thereof
- tar (1) is implicitly comprehensive (i.e., no incremental backups)
- dump(8) supports incremental backups
- dump(8) allows for filesystem backup and integrates with e.g., /etc/fstab, /etc/dumpdates
- restore(8) can extract full datasets, select files, and lets you interactively browse the backup
- rsync(1) can incrementally sync filesystem hierarchies
- rsync(1) can delete data in the destination if removed from the source

## Time Travel

Fast Filesystem (ffs) supports fs snapshots.
fss (fs snapshot) is a pseudo device that gives the view of the fs when the snapshot was taken

```sh
fssconfig -cv fss0 / /backup # very fast to snapshot
# creates large file as big as fs. but can't compress.
# can't change permissions
# ls -lo shows snapshot file

mount /dev/fss0 /mnt # provides readonly filesystem snapshot
# unmount when done
umount /mnt
# unconfigure pseudo device
fssconfig -u fss0
```

snapshot creation is near immediate and are immutable

but snapshots are bound to the system on which it was taken :(

MacOS Time Machine
- start with full backup to separate device
- every hour it creates a full copy via hardlinks for unchanged files, and copies for changed files.
- changed files are determined by last-modified time of directories

Write Anywhere File Layout (WAFL)
- used by NetApp's "Data ONTAP" OS
- uses regular snapshots ("consistency points", every 10 seconds) to allow for speedy recovery from crashes
- a snapshot is a read-only copy of a file system (cheap and near instantaneous, due to Redirect-on-Write (RoW))

Redirect on write has the backup point to the old blocks and the current fs point to the new blocks

ZFS Snapshots
```sh
# create snapshot
zfs snapshot rpool/ROOT/omnios-...@$(date +...)

ls -l /.zfs/snapshot # show snapshots

diff -bur /root/. /.zfs/snapshot/.../root/.

zfs list -t snapshot
zfs rollback <snapshot-name>
```

# Backups Checkpoint

> How do you currently back up your laptop?
> How do you back up the data you have on other systems?

I don't... I probably should've invested in a thunderbolt drive yesterday

My phone is backed up through iCloud.

> When was the last time you restored data from your backup?

N/A for laptop.

When I was still using my iPhone 6 before coming to Stevens and I had to
restore from an iCloud backup after a jailbreak failure.

> Read the dump(8) manual page. What would be a suitable invocation to create a
> full system backup of the file hierarchy under /data?

```sh
dump -u -0 -f - /data | ssh <ip> "cat >backup.0"
```

> Give two examples of filesystems or operating systems which support
> filesystem level backups. How are they different / what do they have in
> common?

Fast File System & ZFS - filesystem snapshot
NetBSD - dump(8) provides full backups and incremental backups

Snapshots are different from true backups because they are bound to the
original machine. True backups can be saved to different locations. Snapshots
can be implemented through redirect-on-write and uses more blocks on the drive.
True backups make a copy of all the data blocks.


