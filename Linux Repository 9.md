# Build an immutable backup repository for Veeam Backup & Replication. Part 9

![Title](images/EE-title-veeam-linux.png)

This guide will show you, step by step, how to create and implement a disk-based immutable Veeam backup repository from scratch.
In this part: Maintenance and deactivation/reactivation of MFA/2FA.

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
## Part 9. Maintenance and deactivation/reactivation of MFA/2FA

In the previous section, we tightened the security on the server. However, in certain cases, it will be convenient (or even a demand) to turn off 2FA/MFA temporarily.

One example is, when it is time to update the *Veeam Linux Transport* running on the Linux server. This will be the case, if *Veeam Backup & Replication* has been updated and requires a matching (updated) transport service installed on the Linux server.

The route to follow isn't convoluted:

- Turn off MFA/2FA authentication
- Perform the maintenance required
- Turn on MFA/2FA authentication

In the following, we will first show how to turn off and on MFA/2FA authentication, and then - having MFA/2FA turned off - how to update the *Veeam Linux Transport* on the Linux server. It is not difficult, only not completely straight-forward.

### Deactivate MFA/2FA temporarily

MFA/2FA can be turned off with a few commands.

First, open the *pam* configuration file in the editor with this command:

    sudo nano /etc/pam.d/sshd

Locate the line (probably the last line):

    auth required pam_google_authenticator.so

Comment out that line by prefixing it with an *octothorpe* (#) like this:

    # auth required pam_google_authenticator.so

Press *Ctrl+O* and *Enter*, and exit the editor with *Ctrl+X*.

Next, open the *sshd* configuration file, again using the *nano* editor:

    sudo nano /etc/ssh/sshd_config

In this file, locate the line:

    ChallengeResponseAuthentication yes

and change its status from "yes" to "no". This will instruct SSH to not ask for an authentication code whenever someone attempts to log in to the system.

Press *Ctrl+O* and *Enter* to save the file, then *Ctrl+X* to exit.

Finally, restart the service:

    sudo systemctl restart sshd.service

You can now log in via SSH without a verification code. No reboot is required.


### Reactivate MFA/2FA

If you wish to turn on again the MFA/2FA authentication, all that is needed is to reverse the changes in the two configuration files you made above.

First, open the *pam* configuration file with the command we used previously:

    sudo nano /etc/pam.d/sshd

This time, remove the octothorpe (#) from the line:

    # auth required pam_google_authenticator.so

to make it read:

    auth required pam_google_authenticator.so

Press *Ctrl+O* and *Enter* to save the file, and exit with *Ctrl+X*.

Next, open the *sshd* configuration file, again using the *nano* editor:

    sudo nano /etc/ssh/sshd_config

In this file, again locate the line:

    ChallengeResponseAuthentication no

and change its status from "no" to "yes". This will instruct SSH to ask for an authentication code whenever someone attempts to log in to the system.

Press *Ctrl+O* and *Enter* to save the file, then *Ctrl+X* to exit.

Finally, restart the service:

    sudo systemctl restart sshd.service

MFA/2FA has again been activated for remote connections to the Linux server.


### Updating the Veeam Linux Transport

After an upgrade of *Veeam Backup & Replication*, it will be out of sync with the version of the *Veeam Linux Transport* installed on the Linux server. In the VBR console, you may see the repository on the Linux server listed as **Unavailable**:

![](images/trans%201.png)

If you right-click the repository and select *Rescan*, this will complete with a status of *Warning*:

![](images/trans%202.png)

Likewise will the Linux server itself be listed as **Unavailable**:

![](images/trans%203.png)

If you right-click the server and select *Rescan*, this will report a status of *Failed*:

![](images/trans%204.png)

Simply put: The *Veeam Linux Transport* is out of date, and the immutable repository is inaccessible.

To resolve this, we must take the steps as described in detail in *Part 4*, which enabled *Veeam backup & Replication* to actively install the transport on the Linux server.


#### Update the Linux system

Open a SSH connection to the Linux server for remote operation (or go to the physical keyboard and monitor of the Linux server).

As an initial step, make sure that the Ubuntu installation is current, as this may be required for the new version of the transport to be installed, and because the updates may improve security in general.

Use this command - as described in detail in *Part 3* of this series - to retrieve and install the updates:

    sudo apt-get upgrade

When prompted, enter *Y* to install the updates. This may take some minutes.

You now have a firm starting point for the update of the *Veeam Linux Transport*.


#### Deactivate MFA/2FA

If MFA/2FA authentication is enabled on the server, the first step is to *deactivate MFA/2FA* as described above.
When done, continue.


#### Elevate user veeamuser

As you recall, the Linux account (in this series named *veeamuser*) used by *Veeam Backup & Replication* has extremely limited capabilities, thus can't be used for installing anything.

Temporarily, however, we wish to use this account to install the updated transport.
To be able to do that, call this command to assign our Veeam user *administrator rights* by signing the account in to the *sudo* group - as explained in detail in *Part 4* of this series:

	sudo usermod -a -G sudo veeamuser

In the command, *veeamuser* is the name of the Linux user account we have set up to be used by VBR.

#### Update the transport

Now, return to the *Veeam Backup & Replication* console and *right-click* on the Linux server and select *Properties* to view these:

![](images/trans%205.png)

The *Properties* will display with the option to edit these:

![](images/trans%206.png)

Click *Next* to reach the *SSH Connection* settings:

![](images/trans%207.png)

Click *Add* and select *Single-use credentials for hardened repository ...* :

![](images/trans%208.png)

to open the *Credentials* window:

![](images/trans%209.png)

Type in the username and password as shown.

> Don't just select the previously entered *veeamuser* account. Retyping it ensures, that Veeam will attempt to use the account as an administrator account - as indicated by the checkmark in:
>
> *Elevate account privileges automatically*

Click *OK* to use these credentials:

![](images/trans%2010.png)

Click *Next* to review and be ready for the next step:

![](images/trans%2011.png)

Click *Apply*, and the transport service will be installed while displaying the progress:

![](images/trans%2012.png)

Click *Next* to view the *Summary*:

![](images/trans%2013.png)

Click *Finish* to exit the wizard.

You'll notice, that under *Managed Servers*, the Linux server is no longer listed under **Unavailable**:

![](images/trans%2017.png)

#### Demote user veeamuser

For a moment, return to the terminal window, because now is the time to demote the veeamuser user account. Again, as explained in detail in *Part 4*, use this command to remove the Veeam user account from the *sudo* group:

    sudo deluser veeamuser sudo

As previously noted, here *veeamuser* is the name of the Linux user account we have set up to be used by VBR. When executed, the command will have reset the user account to only have the limited rights necessary to access the immutable repository on the Linux server.


#### Reactivate MFA/2FA

Also, not to forget if you had MFA/2FA authentication activated, *reactivate* this as described above.

When done, you can log out from the Linux server.


### Reactivate the immutable repository

Finally, go to the VBR console and navigate to the Linux server's repository, *right-click* on it, and select *Rescan*:

![](images/trans%2014.png)

The connection to the repository will now be verified and resynchronised, and - after a little while where you can follow the progress - status will report *Success*:

![](images/trans%2015.png)

Click *Close* to close the window.

In the console window, check that the hardened repository is no longer listed as **Unavailable**:

![](images/trans%2016.png)


### Conclusion

In this section, it has been shown how to deactivate and reactivate MFA/2FA authentication, for example to enable special maintenance. Also, the steps to update the *Veeam Linux Transport* has been documented in detail. Mastering these tasks should ensure, that you can keep your Linux server highly secure, while still being able to perform the traditional tasks like updating the services running on the server.

---

This is the last part of the series of articles that started with:

[Build an immutable backup repository for Veeam Backup & Replication. Part 1](https://github.com/GustavBrock/Veeam.Linux/blob/main/Linux%20Repository%201.md)
