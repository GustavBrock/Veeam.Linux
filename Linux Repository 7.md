# Build an immutable backup repository for Veeam Backup & Replication. Part 7

![Title](images/EE-title-veeam-linux.png)

This guide will show you, step by step, how to create and implement a disk-based immutable Veeam backup repository from scratch. In this part: Bare Metal Recovery of the Linux server.

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

---
## Part 7. Bare Metal Recovery of the Linux server

In the previous sections, we installed the *Veeam Agent for Linux*, created recovery media, and configured regular backup of the Linux system.

In this section we will, using the *Veeam Agent*, show how to restore the Linux system from scratch - to perform a *bare metal recovery* of the server - in case its system drive should cease to function.

### Concept

As you already have *Veeam Backup & Replication* running, we have used a repository of this to store the backup of the server.

> As mentioned in the previous section, the backup files could have been stored elsewhere - on a network share or a USB drive. To use either of these options is trivial, thus will not be covered here.

The created recovery media will be used to boot the server and then restore the server from the backup.


### Restoring the Linux server's system drive

The steps to take in case of a system drive failure are these:

- replace the system drive
- boot the server from the recovery media
- locate and select the backup to restore from
- restore the system drive from the backup

So, first replace the system drive. You know how to do this, and the only requirement is, that its capacity must be equal to or larger than the faulty drive.

Next, insert the recovery media and boot the machine from this:

![](images/Recovery%200.png)

Select *Veeam Recovery 5 ...*, press *Enter*, and the machine will boot from the recovery media.

When booted, you have the option to start the SSH server, allowing you to remote control the rest of the process:

![](images/Recovery%201.png)

You have three options:

- To start the SSH server, press *Esc*
- If you are operating the server physically, press *Enter* to *Proceed without SSH*
- Do nothing and wait, and the autostart will time out after 60 seconds

The license screen will display:

![](images/Recovery%202.png)

Mark the two entries and press *Continue* to reach the main menu:

![](images/Recovery%203.png)

Select *Restore volumes* and press *Enter* to select the backup location.

Scroll to mark: *Add VBR server (v10 or later) ...*:

![](images/Recovery%204.png)

Press *Enter*, and you can specify the name and port of the backup server and the credentials to use:

![](images/Recovery%205.png)

Press *Enter* to connect to the server, retrieve the stored restore points, and list these:

![](images/Recovery%2010.png)

Select (green bar) the *Job name* to use, and then navigate to the list of *Restore Points*:

![](images/Recovery%2011.png)

Mark the *Restore Point* to be used to restore the machine and press *Enter*.

You will now (at left) see a list of the current devices in the machine and (at right) of the devices held in the chosen backup:

![](images/Recovery%2012.png)

Mark the *sda* device, as this is the system drive, and press *Enter* to select what to do with the *sda* device:

![](images/Recovery%2013.png)

Select *Restore from ...* and press *Enter* to select what to restore from:

![](images/Recovery%2014.png)

Select the *sda* device as shown, and press *Enter* to finish the selection:

![](images/Recovery%2015.png)

It may appear, as nothing has been altered, but notice, that the bottom menu options have been expanded.

> Note: The menu option *Start restore* will not start the restore, only bring up the summary screen.

Press *S* - to view the summary screen:

![](images/Recovery%2016.png)

If everything seems OK, press *Enter* to actually start the restore.

In the usual Veeam style, every step of the process will be carefully listed, and you can watch the progress:

![](images/Recovery%2017.png)

When the restore is completed, a status screen is displayed, documenting the full process:

![](images/Recovery%2018.png)

> Note the last entry, that *Logs have been exported to the repository*.

Press *Esc* to return to the main menu:

![](images/Recovery%2019.png)

Select (as shown) *Reboot*, and press *Enter* to reboot the machine.

Also, remove the recovery media from the machine.

As noted above, the logs have been exported to the repository. However, at the VBR console, these are not to view directly, only that the restore has taken place and the status (Success or Failed) of this:

![](images/Recovery%2020.png)

The server's Linux system has now been restored completely from the backup stored at your *Veeam Backup & Replication* server using the previously recovery media to boot from, and the restore is documented in the *Veeam Backup & Replication* console.


### Conclusion

This concludes the basic setup of the immutable backup repository as well as enterprise level backup of the server itself.

What's missing could only be some added security for protecting the server from a worst case scenario. These steps and how to implement them will be explained in detail in **Part 8** of this series:

[Build an immutable backup repository for Veeam Backup & Replication. Part 8](https://github.com/GustavBrock/Veeam.Linux/blob/main/Linux%20Repository%208.md)