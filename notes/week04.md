# Software Types

Firmware
- boot stuff
- close to hardware
- may be proprietary

- software running on WiFi Access Point
- remote controls
- car's infotainment and navigational system
- electronics in hardware drives and USB


Kernel

OS
- system software

Add-on software
- web server
- database
- program languages

Middleware
- containers makes things messy

# OS Installation

Variable: data expected to be modified at runtime
Static: data not expected to change

Shareable: data shared among multiple hosts
Non-shareable: data that is unique to a system

Static Shareable:
- /usr /opt

Variable Shareable:
- /var/data /home

Static Non-shareable:
- /boot /etc

Variable Non-shareable:
- /var/run /var/log

Steps:
1. Find the disk and partition it.
  - copy boot code into boot sector. mark partition 0 as active
2. add new filesystem onto the partition
3. mount it (with -o async so writes happen faster when extracting data)
4. copy and decompress binaries into the mountpoint
5. copy bootstrap code into place
6. install primary bootloader with 5 second countdown
7. create all device nodes under /dev with (e.g. `./MAKEDEV all`)
8. `chroot` into the mountpoint
9. modify rc.conf (or similar startup configuration)
  - enable dhcp, ntp
  - set a hostname
10. update /etc/fstab to mount the root filesystem `/dev/wd0a / ffs rw 1 1`
11. exit chroot, unmount disk
12. reboot

# Package Management

System software vs 3rd party (packaged?)
kernel 1 0 ?
drivers 1 1 ?
firmware 0 1 ?
libc 1 0 ?
shell 1 1 ?
ssh/sshd 1 1 ?
mail server 1 1 ?
http server ? 1 ?
database 0 1 ?
python ? 1 ?

package management spans all software, kernel and up.

You routinely have have to install from source and repackage.

Package managers provide:
- a software inventory

for Debian
`dpkg-query -S /path` shows which package a path belongs to.
`dpkg -L package` shows the paths a package uses

- a file listing and lookup tool
- always package all your software

for Fedora
`rpm -qa` list all installed packages
`rpm -ql sudo` list files installed with the package
`sudo rpm -V sudo` verifies package integrity
`sudo rpm -Va` verifies for all packages

- implicit intrusion detection

for NetBSD
`pkg_info` lists packages
`pkg_admin -v -v fetch-pkg-vulnerabilities` to fetch the vulnerability list
`pkg_admin audit -s` we need the pgp key
`curl https://pkgsrc.org/pkgsrc-security_pgp_key.asc | gpg --import`
`gpg --sign-key ...`
now redo package audit
`sudo pkgin install bash` because vuln in bash

- identification of vulnerable packages

# Package Manager Pitfalls

having python packages from your OS package manager and from pip at the same time!

Integrity - pgp keys for trust

Left-pad situation

npm: create public version of a private local package with the same name but higher version number.
- package manager defaults to highest version number.

Summary
- you DEPEND on your dependencies
- mirror internally
- integrity is recursive

# Checkpoint

>  Give three examples of where you might find firmware:

- WiFi Access Point
- Remote Controls
- Car Dashboard

> Try to provide a concise description of what comprises an "Operating System".
> Where do you draw the boundary between the "core OS" and (optional) "add-on software"?

An operating system consists of a kernel and userland code. I think that the
classification of add-on software depends on user. It should be software that
isn't critical to the execution of the OS. But that depends on what the user
wants, like a desktop environment, etc.

> Have you ever installed an OS from scratch? What kind? What did the process entail?

Yes, I installed Arch Linux by following the installation wiki. The process is
pretty much the same. First partition, install the filesystem, bootstrap
packages, chroot, configure the system, install the boot loader, leave chroot
and reboot.

> Which package manager(s) do you have experience using?
> Have you created packages of software for use with a package manager before?

I use homebrew, and I've used pacman and apt. Recently I installed nix-darwin.
I haven't created packages of software for use with a package manager before.

> What, in your opinion, are the three biggest advantages of using a package manager?

- easy-to-use interface for downloading all sorts of software
- dependency management
- file listing

---

# Presentations

WSL
- original WSL translated linux syscalls to windows syscalls
- windows doesn't have permission bits
- WSL2 uses virtualization and HyperV. needs network drivers

WINE (WINE Is Not an Emulator)


---

In-class

What happens before the kernel starts?

Interaction between firmware and OS?
- firmware is hardware specific. Think NIC, RAID controller, etc
- kernel needs access to update firmware

Why 63 sectors offset?
- one for boot loader
- backwards compatibility

Sometimes AWS instances come up, then update and boot you off.

Need a list of source repo lists
cat /etc/yum.repos.d/fedora.repo

There is a magic database for filetype detection

rpm -qpi shows the information from the structure in the format

rpm -ql lists files. though some packages might be metapackages and not have paths.

cpio is another archival format

rpm2cpio a.rpm | cpio -t

rpms aren't magic. there's just information there and you can read and write it.

Identifying vulnerabilities in packages
- You need to be able to verify against trusted sources (e.g. trusted checksums) or offline

how do we trust mirrors?
- make your own mirror or have internal repositories

why gpg/pgp/signing? why can't we just publish a checksum/sha256?
- you get the checksum and the package usually at the same time. with pgp signing, you
  get assurance that the software was genuinely created by whoever claimed to do so.

how do we know keys and checksums are legitimate?
- root cert or one single thing that you trust
- trusting trust https://dl.acm.org/doi/10.1145/358198.358210

---

# Multiuser fundamentals


different environments have different trust models
human interactions in small groups strengthen trust

Trust doesn't scale

Implement Zero Trust principle

Implications of a Multi-user system
- different users may want to keep files private
- share files
- require different permissions

mapping from people to users is not a surjective or injective mapping

nobody is a uid commonly used for privilege separation

Authentication
- proof of identity, not proof of authorization

Kerberos authentication

raising privileges is critical
- setuid

