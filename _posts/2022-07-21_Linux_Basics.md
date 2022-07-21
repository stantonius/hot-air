---
title: "Linux Basics"
description: "I use Linux all the time now. I really don't know enough about it. This changes now"
layout: post
toc: true
comments: true
image: images/some_folder/your_image.png
hide: false
search_exclude: false
categories: [linux, computer_science]
metadata_key1: metadata_value1
metadata_key2: metadata_value2
---



# 2022-07-21_Linux_Basics

I use Linux every day now - whether building a business using GCP cloud resources, or at home building a home server and smart home applications on the Raspberry Pi. 

I have a confesion: I really don't fully understand it, and I overly rely on blind copy-pasta from Stackoverflow. 

This needs to change. Below are some of my ongoing notes on Linux fundamentals. Note that, simply due to the amount of time I spend on Linux machines, I have managed to pick up some of the basics. Therefore these are more for anyone who has some experience but could do with a reminder of other fundamentals.

## Commands

### General
* `man [command]`: (ie. `man ls`) shows you the **manual** of the command argument you supply
	* Note you press `q` to get rid of the manual info
* `ls -lha`: lists all files in current directory along with more info about those files with their file size in human-readable format, including all hidden files
* **Double tab** for autocomplete when there are multiple files with the same query you have entered, to see all of those files listed
* `mv [source] [target]`: the target argument can also rename the source file
	* `cp` works the same way (including renaming of the file)

### Navigation
* While `cd ..` moves up a directory, I wasn't aware that `cd -` moves you back to the directory where you just were.


## Directories
Instead of reading about this, you could watch [Christian at The Digital Life](https://youtu.be/85-o08kaje0) explain these, with video timestamps for each of the major directories.

Below are just some of the directories that exist but I have encountered recently and need to remember what they are.

### `/bin`
Contains essential binaries (programs) that are ciritical for the system to run. Think `/bin/bash`,  `cp`, `ls` - all of these would need to be available in a situation where we needed to restore the system; they are critical programs that when absent, make the system unusable since most likely, boot scripts depend on these commands.

Note, calling `ls` command actually calls the `/bin/ls` binary. This is because `/bin` is in the `$PATH` and therefore when you call `ls`, it searches all paths in `$PATH` to find the appropriate binary

Also note - more and more, `/bin` is being moved to `/usr/bin`. SOme Linux distros only have symlinks in `/bin` referencing `/usr/bin`. The same is true for `/sbin`, etc.

#### `/sbin`
Similar to `/bin`; binaries needed in superuser mode

### `/boot`
Just don't touch.

### `/dev`
Virtual directory that the kernel uses to communicates with **devices**, such as webcam, keyboard, hard drives etc. 

#### `/sys`
Contains information about the devices in the `/dev` directory. So you would use `/sys` to query whether a device is on, its model #, etc.

### `/etc`
Conatins system-wide configuration files (DNS clients, webservers). Also contains configuration files for **users** and **groups**


### `/home`
Every user has their own `/home` directory that is bespoke for them.
* Except the `root` user - their home directory is `/home`

### `/usr`
Doesn't actually stand for user (users have their `/home` directory, but rather Unix-system Shared Resources.

Contains apps and shared resources. Something like Firefox (an external, installed application) would be stored here.

### `/var`
Any files that are expected to grow in size. 

## Sudo
Special command that allows you to act as `root` without having to log in as them


## Users and Groups
* `cat /etc/passwd`: lists all users, along with their default shell (which helps you see which users can run shell commands)
	* `passwd` also works as a command to change passwords of a user
* `cat /etc/group`: lists all groups

### Commands
* `su [username]`: allows you to log in as a user
	* type `exit` to log out

* `usermod`: modify attributes of the user
	* `usermod -aG [groupname] [user]`: to add a user to a group. Note that without the *add* command `-a`, we would *replace* all groups that user already has.

## Files and Permissions
Take the following example

```bash
drwxr-xr-x  4 craig admingroup     4096 Jul 20 21:30 Documents
-rw-------  1 craig craig          1527 Jul 21 02:27 script.sh
-r-xr-----  4 root  craig            46 Jul 20 21:30 test.txt
```

* The `d` in the first column tells us this is a directory; if `-` then it is a file
* The 4th column, even though `craig` is in this column, this is the **group** `craig`
* Permissions are grouped in 3s. In this case, the first `Documents` directory has (in order):
	* `rwx`: the file owner *user*  `craig` has read, write, and execute permissions
	* `r-x`: the group `admingroup` only has read and execute
	* `r-x`: everyone else has only read and write
* In the third file, the owner is `root` but the permissions for the owner are `r-x`. It turns out that for the root, this doesn't matter. The root user always has all permissions by default
	* Also for this file, any member of the craig group can only read the file. Everyone else can't even open the file


### Changing Permissions

**`chmod`**

* `u`: user; aka file **owner**
* `g`: group
* `o`: all other users

For example, to:
* **replace** the group permissions of the `script.sh`: 
	* `chmod g=rwx script.sh`
* **update/add** the group permissions:
	* `chmod g+rw script.sh`
* **remove** the group permissions:
	* `chmod g-x script.sh`

#### Numeric Permissions
Allow you to change the file permissions for all users & groups in one command using the following key:
```bash
r=4
w=2
x=1
```

In order to change the permissions for the `script.sh` file, we could write:
```bash
chmod 761 script.sh
```

Using the key above, this means: `u` will have `rwx` (4+2+1=7), `g` will have `rw-` (4+2=6), and `o` will have `--x` 

### Change Owner
`chown [user] [file]` or `chown [user:group] [file]`

Can add `-R` to make these ownership changes recursively

## Resources
* [Awesome Linux beginner Youtube series](https://youtube.com/playlist?list=PLj-2elZxVPZ_E5cFhNrUa_BqWC8sfcH6q) by one of my favourite Youtubers Christian at [The DIgital Life](https://www.youtube.com/c/TheDigitalLifeTech)
