# Build an immutable backup repository for Veeam Backup & Replication. Part 3

![Title](images/EE-title-veeam-linux.png)

This guide will show you, step by step, how to create and implement a disk-based immutable Veeam backup repository from scratch. In this part: Prepare the Linux server for Veeam.

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

1. Prepare the install of Linux
2. Install Linux on the server
3. Prepare the Linux server for Veeam
4. Create the immutable Veeam backup repository
5. Prepare for backup of the Linux server itself
6. Backup of the Linux server itself
7. Bare Metal Recovery of the Linux server
8. Tighten security on the Linux server (MFA/2FA)


### Requirements

You are familiar with:

- the usual tasks administering at least a small network with one Windows Server
- *Veeam Backup & Replication* and have it installed and running
- the command line - from PowerShell, Command Prompt, or even DOS

> *Veeam Backup & Replication* is assumed to be of *version 11* or later.
> It can be a licensed trial or paid version or even the free [*Community Edition*](https://www.veeam.com/virtual-machine-backup-solution-free.html).

#### XFS and the virtual air gap

The [XFS](https://en.wikipedia.org/wiki/XFS) file system was introduced by SGI in 1993 for its [IRIX 5.0](https://en.wikipedia.org/wiki/IRIX) operation system which was based on UNIX System V Release 4.

XFS was ported to Linux in 2001. As SGI ceased operations in 2009, Linux is today the only operating system supporting XFS.

Why is this important? Because XFS is the only file system offering immutability:

> Once the file is set immutable, this file is impervious to change for any user. Even the root cannot modify, remove, overwrite, move or rename the file. You will need to unset the immutable attribute before you can tamper with the file again.

For the details about handling this, study *Dan Nanni*'s blog on Xmodulo: [How to make a file immutable on Linux](https://www.xmodulo.com/make-file-immutable-linux.html).

Applying immutability to your backup files hosted on a physical server introduces a virtual *air gap* in your backup chain, protecting the backup files from anything else than direct physical access. This way, the backup files will be protected from any attack caused by advanced malware or possible hackers.

The effect is the same as if you back up to tape or DVD and, when done, remove the media from its drive.

---
## Part 3. Prepare the Linux server for Veeam

In this section we will prepare a Linux server running Ubuntu Server (as installed in Part 2) for use with Veeam.

### Remote control

First step is to establish a secure connection to the Linux server to be used for remote control via Windows PowerShell on your Windows administrator machine.

So, open PowerShell and enter this command:

	ssh linuxadmin@ip-address

or, if you have recorded the Linux server in DNS:

	ssh linuxadmin@hostname

The first time, before login, ssh will ask for confirmation of the key to use for the secure connection. Type **yes** (net just **y**) and press Enter and enter the password.

The connection will open, and the welcome message will be displayed:

![](images/ssh%201.PNG)

### Updates and basic configuration

As the very first, install updates and upgrades. An upgrade will install both.

To view the list of available upgrades, enter this command:

	apt list --upgradable

This command is not "dangerous" - it reads only and doesn't change anything. The command to upgrade does, however, thus must be called including the [sudo](https://en.wikipedia.org/wiki/Sudo) command.

> **sudo**
>
> Where we in Windows have **RunAs**, Linux has **sudo**. Whenever a command needs elevated user rights, precede it with sudo.
>
> When called the first time in a session, it will ask for your password, then use this throughout the session while you are active.

Installing upgrades will modify the system, therefore *sudo* must be used to run the *upgrade* command:

	sudo apt-get upgrade

At first, it will display a compressed list of the potential upgrades and ask for your confirmation:

![](images/ssh%202.PNG)

Press **Y** and Enter, and the upgrades will now be installed:

![](images/ssh%203.PNG)

That finishes the upgrades and updates.

### Timezone

By default, the local timezone will be set for timezone UTC. If that is not yours, adjust the timezone.

To view the current setting, run this command:

	timedatectl

That will list the information like this:

	               Local time: Fri 2021-11-19 10:01:08 UTC
	           Universal time: Fri 2021-11-19 10:01:08 UTC
	                 RTC time: Fri 2021-11-19 10:01:08
	                Time zone: Etc/UTC (UTC, +0000)
	System clock synchronized: yes
	              NTP service: active
	          RTC in local TZ: no

If you know your timezone, run this command:

	sudo timedatectl set-timezone CET

If you don't, look it up:

	timedatectl list-timezones

However, as Linux has very good information about timezones, the list is overwhelming. Reduce it to those of your continent/region, for example for *Europe*, by passing (that's done with the *pipe* symbol, **|** ) the list to the Linux command *grep*, which is a filter command:

	timedatectl list-timezones | grep Europe

> Be very aware, that - contrary to Windows - everything in Linux is *case-sensitive*. Thus, if you in the command above use *europe*, nothing will be listed.

From the returned list, find the region/city closest to you, say, *Europe/Copenhagen*. Now, as Linux knows the timezone of this location, use this in the command to set the local timezone:

	sudo timedatectl set-timezone Europe/Copenhagen

To list the adjusted setting, run again:

	timedatectl

That will now list the information like this:

	               Local time: Fri 2021-11-19 11:15:51 CET
	           Universal time: Fri 2021-11-19 10:15:51 UTC
	                 RTC time: Fri 2021-11-19 10:15:51
	                Time zone: CET (CET, +0100)
	System clock synchronized: yes
	              NTP service: active
	          RTC in local TZ: no



### Veeam Linux user account

To access the repository on the Linux server, a strictly limited user account will be used by *Veeam Backup & Replication*. This is because Veeam has to store the password for the account and, though stored encrypted, it would represent a potential security risk, if the account had administrator rights.

We will name the use *veeamuser*. To create the user, call these two commands:

```
sudo useradd veeamuser --create-home -s /bin/bash
sudo passwd veeamuser
```
You will be prompted for entering the password for the user and to retype this.

Finally, you'll see:

```
passwd: password updated successfully
```

### XFS storage

The first physical disk is not intended to be used for anything else than the system. Thus, the second physical disk will be allocated to serve the Veeam backup repository.

For this, it must be formatted using the XFS file system (read about XFS above).

First, verify that XFS is installed. It will be with Ubuntu, but if you use another distribution, it may not.

Use this command:

    modinfo xfs

It will return an output like this:

![](images/ssh%204.PNG)


> If XFS is not installed, install it with these commands:
>
> ````
> sudo apt-get install xfsprogs
> sudo modprobe -v xfs
> ````

Next, list the *disk devices* installed in the machine:

    sudo fdisk -l

Note: The screenshot is from a virtual machine, thus *Disk model* is reported at *Virtual Disk*. For a physical machine, it will be the true disk model number.

![](images/ssh%205.png)

> Harddisks in Linux are named like **/dev/sdx** where **x** is **a** for the first disk, **b** for the second, etc.
>
> Partitions of a disk are named like **/dev/sdxy** where **y** is **1** for the first partition, **2** for the second, etc.
>
> You can also have *logical volumes* (lv). This will, however, not be explained here.

Locate at the bottom of the list the device name of the backup disk. Here it is **/dev/sdb**:

````
Disk /dev/sdb: 1.84 TiB, 1999307276288 bytes, 3904897024 sectors
Disk model: ST2000LM015
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
````

*Write this down or memorise it* - that device name shall be used in the following.

Now it is time to format the backup drive using XFS. The basic command is simple:

    sudo mkfs.xfs /dev/sdx

However, we need to format the partition with the parameters required by Veeam to leverage *Fast-Clone* technology: *reflink*. Also, *CRC* must be enabled.

Thus, the full command (here for device /dev/sdb) will be:

    sudo mkfs.xfs -b size=4096 -m reflink=1,crc=1 /dev/sdb -f

The output will be similar to:

![](images/ssh%206.png)

We now have a disk formatted with XFS. To expose it to Veeam, it must also be *mounted*. In Windows, it is like bringing a disk online and assigning it a volume.

For a start, we must know the *disk id*. It can be read with this command:

	sudo blkid /dev/sdb

The output will show the id, a GUID string:

    /dev/sdb: UUID="c84e532c-f519-40de-94f9-21adfcd3cf57c" TYPE="xfs"

This must be listed in a config file, */etc/fstab*, to be mounted automatically at boot.

The name of the mount point is not fixed, but we will use the name, that Veeam expects by default:

    /mnt/veeam

This can be specified in two ways:

**Manually** by calling this command to load the file with the *nano* editor:

    sudo nano /etc/fstab

This will - if you are old enough - bring back memories of Quick Basic, but the positive thing is, that the shortcut keys to interact with the editor are displayed at the bottom.

The two lines to append in the editor will be these (of course, your GUID retrieved above should be used).

The first is a comment line, the second the actual configuration entry (Note: In total, two lines only. No line break in the second line):

````
# /mnt/veeam was on /dev/sdb
/dev/disk/by-uuid/c84e532c-f519-40de-94f9-21adfcd3cf57 /mnt/veeam xfs defaults 0 0
````

**From the command line** by calling these two commands that pass the two strings to the utility *tee* (that is similar to *edlin* of DOS) which each will append one line to the */etc/fstab* file (Again: Two lines in total, no line breaks):

````
echo '# /mnt/veeam was on /dev/sdb' | sudo tee -a /etc/fstab
echo '/dev/disk/by-uuid/c84e532c-f519-40de-94f9-21adfcd3cf57 /mnt/veeam xfs defaults 0 0' \ | sudo tee -a /etc/fstab
````

You may confirm the result by this command:

    sudo nano /etc/fstab

and you'll see:

![](images/ssh%2010.png)

We are now ready to create the mounting point. It is a two-step process.

Step one is to establish the name of the mounting point using the command *mkdir*. This may confuse you, as *mkdir* doesn't just *make a directory* ready to use:

	sudo mkdir /mnt/veeam

Step two is to mount the drives as specified in the */etc/fstab* configuration file:

	sudo mount -a

Finally, confirm that we are set, by listing the devices with this command:

	df -Th

The last commands and outputs will appear as here:

![](images/ssh%207.png)

You will notice our XFS drive listed having the *Type* of xfs and the mounting point for Veeam.

### Permissions

The last task is to set the very limited permissions to the backup drive.

First, assign our veeam user account as the *owner* of the mounting point for the drive:

	sudo chown -R veeamuser:veeamuser /mnt/veeam/

Next, assign the
- *owner* (**u** in the command line) *read*, *write*, and *execute* rights
- *groups* and *other users* (**g** and **o** in the command line) *no rights*

by calling this command:

	sudo chmod u+rwx,go-rwx /mnt/veeam

Confirm the settings:

	ll /mnt

Here, it is the last line where the tripled dashes, **---**, indicate no rights:

	drwx------  2 veeamuser veeamuser    6 Nov 12 13:36 veeam/

 here:

![](images/ssh%208.png)

### Conclusion

The server is now running Ubuntu Server ready for the next step - installation of the immutable backup repository - which is explained in detail in **Part 4** of this series:

[Build an immutable backup repository for Veeam Backup & Replication. Part 4](link)