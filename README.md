# Snapshots for Home Directories on BTRFS

Snapshot support for home directories on [btrfs](https://en.wikipedia.org/wiki/Btrfs). Btrfs is a filesystem that is based on b-trees and copy-on-write (COW). It supports snapshots which are created instantaneously and with virtually no additional disk space. Script `snapshot` implements these snapshots for regular home directories and also for home directories encrypted with ecryptfs.

The script `snapshot` creates and deletes read-only snapshots of the home directory. These snapshots are stored either in directory `~/.snapshots/` or a subdirectory like `hourly` or `monthly` of `~/.snapshots/`. Each snapshot contains all the files and directories of the home directory from the time the snapshot was created. These read-only snapshots provide access to files which may have been modified or deleted after the snapshot was created. Snapshots are not a replacement for backups; they do not guard against hardware failures. Unlike btrfs snapshots taken with [timeshift](https://github.com/teejee2008/timeshift), the purpose is not to later restore the system with the snapshots. Home directory snapshots are mainly used to retrieve a bunch of files or directories that have since been changed or deleted. 

In order for these scripts to work, the `btrfs` file system needs to used for home directories. Different Linux distros differ in the file systems they install by default; many optionally support btrfs as well. Fedora 33 already uses btrfs as its default file system. Ubuntu allows to select btrfs at the time of installation.

Btrfs snapshots can only be created from btrfs subvolumes. In order for the
`snapshot` command to be usable, the home directory thus needs to be a btrfs
subvolume. Script `home2subvolume` turns home directories into subvolumes.
Fedora 33 supports home directories in btrfs subvolumes right out of the
box: command `useradd --btrfs-subvolume-home ...` adds one for a new user.

These scripts `snapshot` and `home2subvolume` have been written for Ubuntu
20.04 and also tested on Fedora 33. They work on other Linux distros as well.

## Usage

The first step is to turn one or more home directories into btrfs subvolumes. Script `home2subvolume` is called with one or more userids and it performs this conversion.
Script `home2subvolume` needs to be run as root when the home directory to be converted is not in use. A backup of the home directory is a good idea.

This is an example. We add two test users, `test` and `etest`. Userid `test` has a regular home directory and `etest`'s home directory is encrypted with ecryptfs.

```
$ sudo -i
# cd /
# adduser test
 ...
## On Ubuntu create user with encrypted home as this:
# apt-get install -y ecryptfs-utils
# adduser --encrypt-home etest
 ...
# On Fedora create user with encrypted home as this:
# dnf install -y ecryptfs-utils
# useradd -G ecryptfs -etest
# passwd etest # set password, it will be needed by next command
# ecryptfs-migrate-home etest # enter password
## login as etest right away, BEFORE NEXT REBOOT
```

We then turn these home directories of `test` and `etest` as well as root's home directory `/root` into btrfs subvolumes.

```
# home2subvolume root test etest
Turning /root into a btrfs subvolume...
Directory /root converted to btrfs subvolume.

Turning /home/test into a btrfs subvolume...
Directory /home/test converted to btrfs subvolume.

Turning /home/.ecryptfs/etest/.Private into a btrfs subvolume...
Directory /home/.ecryptfs/etest/.Private converted to btrfs subvolume.
```
We list the subvolumes:
```
# btrfs subvolume list /
ID 256 gen 13158 top level 5 path @
ID 258 gen 13154 top level 5 path @home
ID 820 gen 13152 top level 256 path root
ID 821 gen 13153 top level 258 path @home/test
ID 822 gen 13154 top level 258 path @home/.ecryptfs/etest/.Private
```

Snapshots can be created in the home directories that have been converted to subvolumes.

```
# snapshot
Created snapshot /root/.snapshots/2020-11-26_20:48:14.
```
We can also give the snapshot a name.
```
# snapshot my_first_snapshot
Created snapshot /root/.snapshots/my_first_snapshot.
```

Snapshots are subvolumes and the list of subvolumes includes them:
```
# btrfs subvolume list /
ID 256 gen 13158 top level 5 path @
ID 258 gen 13154 top level 5 path @home
ID 820 gen 13152 top level 256 path root
ID 821 gen 13153 top level 258 path @home/test
ID 822 gen 13154 top level 258 path @home/.ecryptfs/etest/.Private
ID 823 gen 13168 top level 820 path @/root/.snapshots/2020-11-26_20:48:14
ID 824 gen 13171 top level 820 path @/root/.snapshots/my_first_snapshot
```

Snapshots can be deleted by calling `snapshot -d` or `snapshot --delete` and a list of the snapshots to be deleted:
```
# snapshot -d my_first_snapshot 2020-11-26_20:48:14
Snapshot /root/.snapshots/my_first_snapshot deleted.
Snapshot /root/.snapshots/2020-11-26_20:48:14 deleted.
```

When we create snapshots for encrypted home directories the subvolume names are much less readable.

```
$ rlogin -l etest localhost
  ...
etest@server:~$ snapshot
Created snapshot /home/etest/.snapshots/2020-11-26_20:59:07.
```

When we list the subvolumes after creating snapshots of encrypted home directories, there will be a name like this:

```
# btrfs subvolume list /
ID 256 gen 13158 top level 5 path @
ID 258 gen 13154 top level 5 path @home
ID 820 gen 13152 top level 256 path root
ID 821 gen 13153 top level 258 path @home/test
ID 822 gen 13154 top level 258 path @home/.ecryptfs/etest/.Private
ID 825 gen 13186 top level 822 path @home/.ecryptfs/etest/.Private/ECRYPTFS_FNEK_ENCRYPTED.FWZHHDIj65-YcEQ24AWIa.TUpMVwJSRXW4kjQoHVf3vtQVr3IS-UEIK70k--/ECRYPTFS_FNEK_ENCRYPTED.FXZHHDIj65-YcEQ24AWIa.TUpMVwJSRXW4kjFENwnx-2mB0VweDXI2xabtFDOdIi7qcyMxGF1AYXxa--
```

## Caveats

Btrfs allows subvolumes and snapshots to be created by regular users but only root is permitted to delete them. `snapshot` calls `sudo` when non-root users delete snapshots with `snapshot -d` or `snapshot --delete`. `sudo` which asks for the password. Users need to be allowed to invoke `sudo` to delete their snapshots. Example:

```
etest@server:~$ snapshot -d 2020-11-26_20:59:07
[sudo] password for etest: 
etest is not in the sudoers file.  This incident will be reported.
```

After etest has been added to group `sudo`, the snapshot can be deleted:
```
etest@server:~$ snapshot -d 2020-11-26_20:59:07
[sudo] password for etest: 
Snapshot /home/etest/.snapshots/2020-11-26_20:59:07 deleted.
```
