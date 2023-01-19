# Snapshots for Home Directories on BTRFS

Snapshot support for home directories on [btrfs](https://en.wikipedia.org/wiki/Btrfs). Btrfs is a file system that is based on b-trees and copy-on-write (COW). It supports snapshots which are created instantly and with virtually no additional disk space. This `snapshot` script implements these snapshots for regular home directories and also for home directories encrypted with ecryptfs.

The `snapshot` script creates and deletes read-only snapshots of the home directory. These snapshots are stored either in the `~/.snapshots/` directory or a subdirectory like `hourly` or `monthly` of `~/.snapshots/`. Each snapshot contains all the files and directories in the home directory from the time the snapshot was taken. These read-only snapshots provide access to files that may have been modified or deleted since the snapshot was taken. Snapshots are not a replacement for backups; they do not protect against hardware failures. Unlike btrfs snapshots taken with [timeshift](https://github.com/teejee2008/timeshift), the purpose is not to later restore the system later. Home directory snapshots are mainly used to recover a set of files or directories that have since been changed or deleted.

For these scripts to work, the `btrfs` filesystem must be used for home directories. Different Linux distributions differ in the file systems they install by default; many also support btrfs as an option. Fedora 33 already uses btrfs as the default file system. Ubuntu allows to select btrfs during installation.

Btrfs snapshots can only be created from btrfs subvolumes. In order for the
`snapshot` command to work, the home directory must be a btrfs
subvolume. Script `home2subvolume` converts home directories into subvolumes.
Fedora 33 supports home directories in btrfs subvolumes right out of the
box: The command `useradd --btrfs-subvolume-home ...` adds one for a new user.

The `snapshot` and `home2subvolume` scripts have been written for Ubuntu
20.04 and also tested on Fedora 33. They will work on other Linux distributions.

## Usage

The first step is to convert one or more home directories into btrfs subvolumes. The `home2subvolume` script is called with one or more usernames and it performs this conversion.
The `home2subvolume` script must be run as root and only when the home directory
to be converted is not in use. A backup of the home directory is a good idea.

This is an example. We add two test users, `test` and `etest`. User `test` has a regular home directory and `etest`'s home directory is encrypted with ecryptfs.

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
# useradd -G ecryptfs etest
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

Btrfs allows subvolumes and snapshots to be created by normal users but only root is allowed to delete them. `snapshot` calls `sudo` when non-root users delete snapshots with `snapshot -d` or `snapshot --delete`. `sudo` will ask for the password. Users must to be allowed to invoke `sudo` to delete their snapshots. Example:

```
etest@server:~$ snapshot -d 2020-11-26_20:59:07
[sudo] password for etest: 
etest is not in the sudoers file.  This incident will be reported.
```

After adding etest to the `sudo` group, the snapshot can be deleted:

```
etest@server:~$ snapshot -d 2020-11-26_20:59:07
[sudo] password for etest: 
Snapshot /home/etest/.snapshots/2020-11-26_20:59:07 deleted.
```
