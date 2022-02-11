# Build an immutable backup repository for Veeam Backup & Replication

![Title](images/EE-title-veeam-linux.png)

This guide will show you, step by step, how to create and implement a disk-based immutable Veeam backup repository from scratch. In this part: Prepare the install of Linux.

---
## Introduction

### Purpose of these articles

You are a Windows administrator running [Veeam Backup & Replication](https://www.veeam.com/vm-backup-recovery-replication-software.html) and wish to raise protection against malware attacks and hackers without reverting to shuffle or rotate physical media.

This you can accomplish by *immutable backups* stored on a physical server running Linux.
However, you have no Linux servers running and don't want to.

But, like it or not, that is your only option, as the XFS file system is the only one capable of immutability, and XFS only runs under Linux.

Thus, a Linux server is a must. When you have accepted this fact, then what? Where to start?

Like me, you have about zero experience with Linux and, therefore, hesitate to set up a Linux server, indeed in a production environment.

If so, this guide is for you. Here, nothing about Linux is taken for granted.

### Sections

The guide has been split in eight parts. This allows you to skip parts you are either familiar with or wish to implement later if at all.

1. [Prepare the install of Linux](https://github.com/GustavBrock/Veeam.Linux/blob/main/Linux%20Repository%201.md)
2. [Install Linux on the server](https://github.com/GustavBrock/Veeam.Linux/blob/main/Linux%20Repository%202.md)
3. [Prepare the Linux server for Veeam](https://github.com/GustavBrock/Veeam.Linux/blob/main/Linux%20Repository%203.md)
4. [Create the immutable Veeam backup repository](https://github.com/GustavBrock/Veeam.Linux/blob/main/Linux%20Repository%204.md)
5. [Prepare for backup of the Linux server itself](https://github.com/GustavBrock/Veeam.Linux/blob/main/Linux%20Repository%205.md)
6. [Backup of the Linux server itself](https://github.com/GustavBrock/Veeam.Linux/blob/main/Linux%20Repository%206.md)
7. [Bare Metal Recovery of the Linux server](https://github.com/GustavBrock/Veeam.Linux/blob/main/Linux%20Repository%207.md)
8. [Tighten security on the Linux server (MFA/2FA)](https://github.com/GustavBrock/Veeam.Linux/blob/main/Linux%20Repository%208.md)
9. [Maintenance and deactivation/reactivation of MFA/2FA](https://github.com/GustavBrock/Veeam.Linux/blob/main/Linux%20Repository%209.md)

### Requirements

You are familiar with:

- the usual tasks administering at least a small network with one Windows Server
- *Veeam Backup & Replication* and have it installed and running
- the command line - from PowerShell, Command Prompt, or even DOS

> *Veeam Backup & Replication* is assumed to be of *version 11* or later.
> It can be a licensed trial or paid version or even the free [*Community Edition*](https://www.veeam.com/virtual-machine-backup-solution-free.html).

### XFS and the virtual air gap

The [XFS](https://en.wikipedia.org/wiki/XFS) file system was introduced by SGI in 1993 for its [IRIX 5.0](https://en.wikipedia.org/wiki/IRIX) operation system which was based on UNIX System V Release 4.

XFS was ported to Linux in 2001. As SGI ceased operations in 2009, Linux is today the only operating system supporting XFS.

Why is this important? Because XFS is *the only file system offering immutability*:
> Once the file is set immutable, this file is impervious to change for any user. Even the root cannot modify, remove, overwrite, move or rename the file. You will need to unset the immutable attribute before you can tamper with the file again.

For the details about handling this, study *Dan Nanni*'s blog on Xmodulo: [How to make a file immutable on Linux](https://www.xmodulo.com/make-file-immutable-linux.html).

Applying immutability to your backup files hosted on a physical server introduces a virtual *air gap* in your backup chain, protecting the backup files from anything else than direct physical access. This way, the backup files will be protected from any attack caused by advanced malware or possible hackers.

The effect is the same as if you back up to tape or DVD and, when done, remove the media from its drive.

As XFS does not exist in the Windows world, implementing this feature requires a physical Linux server.

---
