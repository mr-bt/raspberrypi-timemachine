# 使用树莓派搭建 Time Machine 的正确姿势

群晖太贵了！若是不介意 USB 2.0 和 100 Mbps 带宽的话，用树莓派 + 移动硬盘做个低功耗小型 NAS 也是不错的选择，搭建 Samba 的教程到处都是，本篇将介绍如何以正确的姿势搭建 AFP 共享 + Time Machine。

The following step are the ones that enable Time Machine backups with Raspberry Pi plus a bit of polishing to my taste.

## 1. Format the hard drive
I had a hard-drive serving as Time Machine disk. However, I couldn't mount the disk due to Apple Core Storage:

```
Disk /dev/sda: 931.5 GiB, 1000204886016 bytes, 1953525168 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: DE07BD84-C4E1-4229-81CD-E146E04D46C6

Device         Start        End   Sectors   Size Type
/dev/sda1         40     409639    409600   200M EFI System
/dev/sda2     409640  975539735 975130096   465G Apple Core storage
/dev/sda3  975539736  975801879    262144   128M Apple boot
/dev/sda4  975802368 1953523711 977721344 466.2G Microsoft basic data
```

StackOverflow thread [mounting hfs partition on arch linux](https://superuser.com/questions/961401/mounting-hfs-partition-on-arch-linux/1088110#1088110) didn't work for me. Since the backups on that disk were a bit outdated I decided to format the partition and give it a go. Another alternative would be to use Disk Utility to get rid of Apple Core Storage but in my case not worth the effort.

So, format the HD on your Mac using Disk Utility. Settings used:
* Name: `Time Machine`
* Format: `Mac OS Extended (Journaled)`
* Scheme: `GUID Partition Map`

## 2. Ensure Pi has permissions to control the drive
Go to the Finder, then right-click the drive in the sidebar. Click “Get Info”.

Click the lock at bottom right, then enter your password. Next, check `Ignore ownership on this volume.` and give `Read & Write`permissions to `everyone`.

Linux cannot write data into `journaled hfs+` file system, so you also need to disable that:

```
diskutil list
```

Then find your drive and partition identifier (like: disk\*s\*), and run:

```
 diskutil disableJournal /dev/disk*s*
```

Dont forget to replace the asterisks. You'll see:

```
Journaling has been disabled for volume MyTimeMachine on disk*s*
```

## 3. Install tools for Apple-formatted drives
Go to Pi (ssh'ed it!) and run:
```
$ sudo apt-get update
$ sudo apt-get upgrade
$ sudo apt-get --assume-yes install hfsprogs hfsplus
```

## 4. Mount the drive

#### Find the drive:

```
$ sudo fdisk -l

...

Device         Boot   Start      End  Sectors  Size Id Type
/dev/mmcblk0p1         8192  3292968  3284777  1.6G  e W95 FAT16 (LBA)
/dev/mmcblk0p2      3292969 62333951 59040983 28.2G  5 Extended
/dev/mmcblk0p5      3293184  3358717    65534   32M 83 Linux
/dev/mmcblk0p6      3358720  3500031   141312   69M  c W95 FAT32 (LBA)
/dev/mmcblk0p7      3506176 62333951 58827776 28.1G 83 Linux


Disk /dev/sda: 931.5 GiB, 1000204886016 bytes, 1953525168 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: DE07BD84-C4E1-4229-81CD-E146E04D46C6

Device         Start        End   Sectors   Size Type
/dev/sda1         40     409639    409600   200M EFI System
/dev/sda2     409640  975539735 975130096   465G Apple HFS/HFS+
/dev/sda3  975802368 1953523711 977721344 466.2G Microsoft basic data
```

In my case my HD is connected to USB and the device is `/dev/sda2`. A good hint is the fs type `Apple HFS/HFS+` or on other tools `hfsx`. 

#### Create your mounting point:

```
$ sudo mkdir -p /media/time_machine
```

#### Check if Pi already auto-mounted your drive:

```
$ sudo mount
```

If it's mounted, you need to un-mount it or give it write permissions. In my case I didn't want to have it mounted on `/media/pi/Time\ Machine` so I un-mounted it:

```
$ sudo umount /dev/sda2
```

#### Mount drive using your editor of choice:

```
$ sudo nano /etc/fstab
```

Add to the end of the file:

```
/dev/sda2 /media/time_machine hfsplus force,rw,user,noauto,x-systemd.automount 0 0
```

Mount the drive

```
$ sudo mount -a
```

Check if it's mounted by finding the line like the bellow:

```
$ sudo mount
...
/dev/sda2 on /media/time_machine type hfsplus (rw,nosuid,nodev,noexec,relatime,umask=22,uid=0,gid=0,nls=utf8,user)
```

#### Install Netatalk
Netatalk simulates AFP, the network protocol Apple currently users for Time Machine backups.

Install dependencies

```
sudo aptitude install build-essential libevent-dev libssl-dev libgcrypt11-dev libkrb5-dev libpam0g-dev libwrap0-dev libdb-dev libtdb-dev avahi-daemon libavahi-client-dev libacl1-dev libldap2-dev libcrack2-dev systemtap-sdt-dev libdbus-1-dev libdbus-glib-1-dev libglib2.0-dev libio-socket-inet6-perl tracker libtracker-sparql-1.0-dev libtracker-miner-1.0-dev
```

Install Netatalk via apt-get

```
apt-get install netatalk
```
```
whereis netatalk
```

## 6. Configure Netatalk

First let's set up `nsswitch.conf` by adding to the end of `hosts:          files mdns4_minimal [NOTFOUND=return] dns` line ` mdns4 mdns`.

```
$ sudo nano /etc/nsswitch.conf
```

It should look like this:

```
# /etc/nsswitch.conf
#
# Example configuration of GNU Name Service Switch functionality.
# If you have the `glibc-doc-reference' and `info' packages installed, try:
# `info libc "Name Service Switch"' for information about this file.

passwd:         compat
group:          compat
shadow:         compat
gshadow:        files

hosts:          files mdns4_minimal [NOTFOUND=return] dns mdns4 mdns
networks:       files

protocols:      db files
services:       db files
ethers:         db files
rpc:            db files

netgroup:       nis
```

This means your Time Machine drive will show up in Finder’s sidebar.

Next set up `afp.conf`

```
$ sudo nano /usr/local/etc/afp.conf
```

add to the end

```
[Global]
  mimic model = TimeCapsule6,106

[Time Machine]
  path = /media/time_machine
  time machine = yes
```

At last set `AppleVolumes.default` (might not be necessary)! I did it anyway...

```
$ sudo nano /etc/netatalk/AppleVolumes.default
```

and add to the end of the file

```
/media/time_machine "Time Machine" 	options:tm
```

## 7. Launch network services
```
$ sudo service avahi-daemon start
$ sudo service netatalk start
```

```
$ sudo systemctl enable avahi-daemon
$ sudo systemctl enable netatalk
```

## 8. Give your Pi a static IP
Go to your router and assign a static IP to your Pi.

## 9. Connect to time machine
Go to your Mac Finder you should see your Raspberry Pi there.
Click on `Connect as` and insert your credentials (user: timemachine). If doesn't work, connect to your Pi through its static IP. Open Finder, then hit Command+K on your keyboard and insert:
```
afp://<you Pi ip>
```

## 10. Configure your Mac Time Machine
Go to `System Preferences > Time Machine` and clik on `Select Disk...`. Your Pi should show on the list. Select and use the settings that work best.

## Notice

- Make sure that `erveryone` has permissions to read & write your disk drive.

- If `aptitude install ...` at the last of step 5 shows an eror like `Package 'libmysqlclient-dev' has no installation candidate`, please try `default-libmysqlclient-dev`.

## Reference Articles
* [How to make a Mac Time Capsule with the Raspberry Pi](http://www.techradar.com/how-to/computing/how-to-make-a-mac-time-capsule-with-the-raspberry-pi-1319989)
* [How to Use a Raspberry Pi as a Networked Time Machine Drive For Your Mac](https://www.howtogeek.com/276468/how-to-use-a-raspberry-pi-as-a-networked-time-machine-drive-for-your-mac/)
* [Use rPi as a Time Capsule - another method](https://www.raspberrypi.org/forums/viewtopic.php?f=36&t=47029)
* [Mounting HFS+ drive (OS X Journaled)](https://www.raspberrypi.org/forums/viewtopic.php?t=18003)
* [Mounting HFS+ partition on Arch Linux](https://superuser.com/questions/961401/mounting-hfs-partition-on-arch-linux/1088110#1088110)
* [Time Machine on Raspberry Pi](http://lanjanitor.blogspot.co.uk/2013/12/time-machine-on-raspberry-pi.html)
* [Time Capsule / Time Machine through Raspberry Pi](https://www.bersling.com/2017/01/02/time-capsule-time-machine-through-raspberry-pi/)
* [DIY Time Capsule with a Raspberry Pi](http://www.calebwoods.com/2015/04/06/diy-time-capsule-raspberry-pi/)
* [fstab - ArchWiki](https://wiki.archlinux.org/index.php/Fstab#Automount_with_systemd)
