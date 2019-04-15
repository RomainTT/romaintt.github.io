---
title: "How I make backups of my Nextcloud server"
description: "My setup based on rsync to make backups of my personal cloud."
date: 2019-04-15
layout: post
categories: privacy raspberry 
language: english
---
## Introduction

In [another post]({{ site.baseurl }}{% post_url 2019-01-04-my-cloud %}),
I explain how I set up a personal cloud server at home using 
[Nextcloud](https://nextcloud.com/). This solves the problem of privacy
and the size of data I can save does not depend on how much I can pay 
each month. So far, I am very happy with this solution! Nextcloud is
a great software and I really feel like I am the master of my own data.

Now there is still one problem. What if my flat burns? or get robbed?
My nice setup is defenceless against this kind of threat, I would simply
lose everything. Another threat is data corruption, either by a wrong
user action or a malware (ransomwares are more and more common nowadays!).
RAID can prevent losing data because of a falty disk, but not because of
the reasons cited above.

The obvious solution I see is to make backups in another place than my home.
Or even several other places to maximize safety!
And it is better to have the whole thing automated of course.
The following figure illustrates the principle:

![Schema of principle](/images/backup_cloud/backup_principle.png)

Several pre-conditions are required:

- having other places to store the backup. In my case I can install a server
  in my parent’s home for instance.
- having a hardware to make backups. I found out that a Raspberry Pi does the 
  job quite well for my Nextcloud server. But as backups servers do not need 
  to host Nextcloud, and just regurlaly copy data into a hard drive, no need 
  for the best one. As for the hard drives, I don’t want to spend a lot of 
  money to haves disks in RAID like I did for the main server. They are just
  backups, a simple disk in the form of an external drive is enough.
- having a connection between the main server and the backup servers. All of
  them are connected to the Internet, and the main server is reachable from
  anywhere using SSH. So that’s it! Backup servers can connect to it using
  SSH thanks to their ssh keys.

Now that the problems of the real world are solved, let’s solve the digital
ones! In short, I am using **rsync daemon mode over ssh**.

## Use of rsync to perform the backups

[`rsync`](https://en.wikipedia.org/wiki/Rsync) is an ideal tool to perform 
backups of distant hosts. There are several ways to do so:

- With a password and directly over SSH. Secured but manual.
- With a public key and directly over SSH. Better for automation but whoever
  has the private key of the backup server can mess with the main server.
  (I am considering this possibility because my backup servers are away
  from me and I am not in full control, they can be stolen for instance.)
- Running rsync in daemon mode on the main server. Avoid the previous issue
  and has nice options like read-only mode. But it opens a TCP port that has
  filesystem access and is hard to secure.

The solution I am choosing is a combination of the previous ones: using rsync 
daemon mode over SSH.
This mode is explained in the man page of `rsync` under 
*USING RSYNC-DAEMON FEATURES VIA A REMOTE-SHELL CONNECTION*.

Here are the steps to set this up.

### Configure the `rsync` daemon

The basic use of `rsync` with a distant host is something like:

```shell
rsync source_dir host:dest_dir
```

Where `dest_dir` is a path. But when `rsync` runs as a daemon on the distant
host, it can manage `modules` which reference one or more paths with options,
and the command line becomes:

```shell
rsync source_dir host::module_name
```

Note that there are two double-dots between the host name and the module name.

The description of the available modules can be done in a configuration file
given as a parameter to the `rsync` command while starting the daemon:

```shell
rsync --daemon --config=rsyncd.conf
```

To learn everything about this configuration file, read `man rsyncd.conf`.
Here is the configuration file of my daemon to backup files from Nextcloud:

```ini
# /home/pi/.rsync/nextcloud_rsyncd.conf

log file = /home/pi/.rsync/nextcloud_backup.log
read only = true  # to avoid a compromised backup server to harm the main server.
exclude = lost+found/
fake super = yes  # important to keep users and groups of files untouched 
use chroot = no  # important or else it tries to chroot to path and fails because not root

[nextcloud_install]
comment = Files of the Nextcloud server installation. Does not contain user data.
path = /var/www/html/nextcloud  # user 'pi' must have read access on this directory!

[nextcloud_data]
comment = Data files of the Nextcloud server, including the database.
path = /path/to/nextcloud/data
```

As you can see there are two modules described in this configuration.
One is dedicated to the core files of Nextcloud, and the other to the
data files of all users of Nextcloud, including the database of Nextcloud.
This allows the backup servers to save these directories into different
locations and at different periods for instance.

The daemon will be run from user `pi`. To give this user read access to
Nextcloud files, `pi` has been added to the group `www-data`. In this way
`pi` user cannot write anything in Nextcloud directories, even without the
option `exclude`: double security!

### Configure the SSH connection

Each backup server must have access to the main server using SSH. To do so,
each backup server must have a SSH key pair, and give the public key to the
main server so that it can connect to it without giving a password.

First, I need to generate a ssh key pair on the backup server:

```shell
ssh-keygen -b 256 -t rsa -N my_passphrase -f ~/.ssh/id_rsa_nextcloud
```

And copy the content of `~/.ssh/id_rsa_nextcloud.pub` for the next step.

On the main server containing Nextcloud, I add this public key to the list
of authorized keys with a **forced command**. A forced command ensures that
a command, and only this one, will be run when a given key is recognized.
That ensures that the host connecting with this identity won’t be able to do
anything else than this written command.

Therefore, I force the backup server to run a daemon of rsync when connecting,
and disable a number of advance features, such as TCP and X11 forwarding.

```ini
# ~/.ssh/authorized_keys

# The following ssh key starts a rsync daemon when connecting, to perform backup
command="rsync --config=/home/pi/.rsync/nextcloud_rsyncd.conf --server --daemon . ",no-agent-forwarding,no-port-forwarding,no-pty,no-user-rc,no-X11-forwarding ssh-rsa <pub key> <name>
```

Where of course `<pub key>` and `<name>` are to be replaced with the public key
of the backup server and an arbitrary name.

One can see that the configuration file written above is given as an argument
in the command. Note that the path to this file cannot contain `~`, it is not
interpreted. One must use a full path or an environment variable like `$HOME`.

I must confess that I do not fully understand the use of the `--server` option.

### Perform the backup

To actually perform the backup, the `rsync` command must be run on the backup 
server. Two commands: one for the Nextcloud installation directory, and one for
data:

```shell
rsync -a -v -e "ssh -i ~/.ssh/id_rsa_nextcloud" pi@main_server_address::nextcloud_install backup/nextcloud_install
rsync -a -v -e "ssh -i ~/.ssh/id_rsa_nextcloud" pi@main_server_address::nextcloud_data backup/nextcloud_data
```

- The `-a` option is used to preserve the ownership of the files.
- The `-v` to make the command verbose.
- The `-e` to precise to use rsync over SSH with a given identity and SSH
  user name.

Attention: there are two usernames used here — ssh and rsync usernames:

```shell
rsync -e "ssh -l ssh_username" rsync_username@host::module dest/
```

I must highlight the fact that the IP addresses of the backup servers
are not known by the main server and they do not need to be static or
even public. Only the main server has these requirements. Therefore, it
is very easy to spread many backup servers anywhere as long as there is
a simple internet connection.

### Automate the backup

What I want is to run the commands written above periodically, like once
a week for instance. The use of [`cron`](https://en.wikipedia.org/wiki/Cron) 
is well-suited in this case.

I will detail the implementation if I have time later.

## In case of...

### Loss of the backup server

No need to worry, my mainserver is still on tracks and I did not lose anything.
I just need to replace/fix the broken/stolen/burnt backup server with a new
one.

If I am worried about the SSH key being in the hands of an evil person, I can
simply remove this key from the list of authorized keys on the main server,
and this key becomes useless.

### Loss of the main server

That is more annoying, but backup servers are there for it! I must bring back 
the hardware in place for the main server, then I use `rsync` on the other way
to restore the installation of Nextcloud and most of all, data. Nextcloud can 
easily rescan a whole data directory and run like nothing happened.

### Addition of a new backup server

I can multiply the number of backup servers simply by adding each new public 
key to the list of authorized keys on the main server. There can’t be any
conflict between them, and each new backup will be an additional safe copy.

