---
title: Home file storage server on the Raspberry Pi
toc: true
---

# Introduction
Hi everyone! This post might be useful for you if you are planning to set up a home file storage server. 
Initially I tried to explain every step, but the post grew wordy pretty quickly. 
Instead, I have decided to minimize the number of comments and assume that you have a minimum bash knowledge.
Also, I am using `${FOO_BAR}`-s style variables a lot, cos I believe that you might need or want to have different values.
Finally, I believe that these steps are not limited to the Raspberry Pi, but I did not test this.

One more thing: [this gist comment](https://gist.github.com/etes/aa76a6e9c80579872e5f?permalink_comment_id=2781906#gistcomment-2781906) 
saved me a few times from reinstalling OS. I hope you do not need it, but it's always better to be prepared.

# Background
On summer 2022 I have realized that I have very basic k8s skills and I need a way to improve them.
I knew that taking k8s related course is not enough and I need to do something practical.
At the same time, I was looking for some way to sync 250GB of family photos and videos across family laptops and phones.
As a result I have decided to buy a single Raspberry Pi and to install "any cloud app" there using k8s.

After a quick search I have stumbled on the [Nextcloud Files](https://nextcloud.com/files/) app.
It can support desktop and mobile clients. The Nextcloud is also [opensource](https://github.com/nextcloud)
and provides frequent releases.

My setup includes 2 external USB drives. A smaller one for storing files and a bigger one for backups.
The smallest case I found that could fit my Raspberry Pi and two external USB drives is 
[this one](https://thepihut.com/collections/raspberry-pi-cases/products/ssd-cluster-case-for-raspberry-pi). 
Please leave a comment if you know a smaller one that could handle that.
The built-in case fans are not too loud, but there was no way to make them turn on and off automatically.
I have fixed this by using this [fan control module](https://thepihut.com/products/auto-fan-control-module-5v-breakout-for-raspberry-pi).

I did not want to put too much power pressure on my Raspberry Pi, so I bought this
[USB hub with power adapter](https://www.amazon.co.uk/Sabrent-Individual-Switches-included-HB-UMP3/dp/B00TPMEOYM?ref_=ast_sto_dp&th=1&psc=1).

I did not and still don't want to change the default "Raspbian GNU/Linux 11 (bullseye)" OS, 
since this process does not entertain me.

Bellow you can find links, steps and commands I combined to get the "Nextcloud Files" running. 
I'm not an expert in what I was doing, so please leave a comment if there is a better or easier way to perform any given step(s).

# Setup
## Fresh Raspberry Pi
1. Connect a mouse, a keyboard and a monitor to a Raspberry Pi. Follow the prompts.
1. [Optional] Set up the VNC instead of using Peripherals per [this article](https://linuxhint.com/run-realvnc-raspberry-pi/).   
1. [Optional] Set up SSH connection per [this article](https://linuxhint.com/enable-ssh-raspberry-pi/).
1. [Optional] Update security settings per [this article](https://raspberrytips.com/security-tips-raspberry-pi/).

## Fans setup via GUI
1. Follow "fan control module" manual.
1. Open Raspberry > Preferences > Raspberry Pi Configuration > Performance tab
   1. Set `Fan = true`
   1. Set `Fan GPIO = 4`
   1. Set `Fan Temperature = 60`. The fan will turn on when the temperature hits 60C.
1. [Optional] Add CPU temperature gauge per [this article](https://www.raspberrypi-spy.co.uk/2020/11/raspberry-pi-temperature-monitoring/) 
   with the following colors:
   1. Normal color `#163c8c`
   1. Warning1 color `#d68549`
   1. Warning2 color `#e3001f`

## USB drives
### Device names
Get the device names like `dev/name` via `lsblk`. For example:
```console
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
...
sda           8:0    0 465.8G  0 disk
└─sda1        8:1    0 465.8G  0 part /media/500
sdb           8:16   0   1.8T  0 disk
├─sdb1        8:17   0   200M  0 part
└─sdb2        8:18   0   1.8T  0 part /media/2000
...
# Please note that you might have different values
$ export MAIN_USB = "dev/sda1"
$ export BACKUP_USB = "dev/sdb2"
```

### Format USB
For compatibility reasons, I have decided to format USB drives in ext4:
```console
$ sudo umount ${MAIN_USB}
$ sudo mkfs.ext4 ${MAIN_USB}

$ sudo umount ${BACKUP_USB}
$ sudo mkfs.ext4 ${BACKUP_USB}
```

### Mount USB drives
Sometimes USB drives are not mounted after restart, so a few extra steps are required
> **Note**: First I tried to mount using fstab, the following way:
> ```console
> $ sudo cp /etc/fstab /etc/fstab.backup
> 
> # Add the following to "/etc/fstab":
> UUID="${MAIN_USB_UUID}" ${MAIN_USB_MOUNT} ext4 defaults,nofail,x-systemd.device-timeout=1,noatime  0       0
> UUID="${BACKUP_USB_UUID}" ${BACKUP_USB_MOUNT} ext4 defaults,nofail,x-systemd.device-timeout=1,noatime  0       0
> 
> $ sudo reboot
> ```
> But usb drives often failed to mount during boot, so I used the approach from bellow:

Create a `${USB_MOUNT_SCRIPT}` file with the following content:

```shell
#!/bin/sh

echo "mounting ${MAIN_USB} to ${MAIN_USB_MOUNT} folder:"
while ! lsblk | grep "${MAIN_USB_MOUNT}"; do
   echo "5 sec break..."; sleep 5
   sudo mount ${MAIN_USB} ${MAIN_USB_MOUNT}
done
echo "Done for the main USB"

echo "mounting ${BACKUP_USB} to ${BACKUP_USB_MOUNT} folder:"
while ! lsblk | grep "${BACKUP_USB_MOUNT}"; do
   echo "5 sec break..."; sleep 5
   sudo mount ${BACKUP_USB} ${BACKUP_USB_MOUNT}
done
echo "Done for 2T"

echo "Set 777 permissions for the ${MAIN_USB_MOUNT} folder to avoid NextCloud errors"
sudo chmod 777 -R ${MAIN_USB_MOUNT}
echo "Permissions were set"
```
Update crontab
```console
$ sudo chmod +x ${USB_MOUNT_SCRIPT}
$ sudo crontab -e
```
Add the following:
```shell
@reboot ${USB_MOUNT_SCRIPT}>/tmp/usb_mount.log
```
Inspired by [this comment](https://superuser.com/a/547124) and [this article](https://www.baeldung.com/linux/new-files-dirs-default-permission):

## Backup
I did not want to use [rsync](https://linux.die.net/man/1/rsync) 
and decided to use [borg](https://borgbackup.readthedocs.io) instead.


`${BACKUP_SOURCE}` is path to files and `${BACKUP_REPOSITORY}` is path to backups.

### Set up borg repository
```console
$ touch ${BACKUP_JOB_LOG}

$ sudo apt install borgbackup
$ sudo su -

# Setup
$ borg init --storage-quota 1.5T --encryption=none ${BACKUP_REPOSITORY} 2> ${BACKUP_JOB_LOG}
$ borg create --compression auto,zstd ${BACKUP_REPOSITORY}::${BACKUP_NAME}`date +%Y-%m-%d_%H:%M` ${BACKUP_SOURCE} 2> ${BACKUP_JOB_LOG}

# Restore
$ borg list ${BACKUP_REPOSITORY}  # See available backups
$ borg extract ${BACKUP_REPOSITORY}::${BACKUP_NAME} ${BACKUP_SOURCE} # Restore latest backup
$ borg extract --timestamp 2022-01-01T12:00:00 ${BACKUP_REPOSITORY}::${BACKUP_NAME} ${BACKUP_SOURCE} # Restore per timestamp

$ exit
```

### Set up periodic backups
Create a `${BACKUP_SCRIPT}` file with the following content:
```shell
#!/bin/sh

echo "Backup started on `date +%Y-%m-%d_%H:%M`" >> ${BACKUP_JOB_LOG} 2>&1
borg prune --keep-daily=7 --keep-weekly=4 --keep-monthly=6 ${BACKUP_REPOSITORY} >> ${BACKUP_JOB_LOG} 2>&1
borg create --compression auto,zstd ${BACKUP_REPOSITORY}::${BACKUP_NAME}`date +%Y-%m-%d_%H:%M` ${BACKUP_SOURCE} >> ${BACKUP_JOB_LOG} 2>&1
echo "Backup finished on `date +%Y-%m-%d_%H:%M`" >> ${BACKUP_JOB_LOG} 2>&1
```
Update crontab
```console
$ sudo chmod +x ${BACKUP_SCRIPT}
$ sudo crontab -e
```
Add the following:
```shell
0 0 * * * /usr/bin/sh ${BACKUP_SCRIPT}
```

## Install k8s
I did not plan to learn to [install vanilla k8s](https://kubernetes.io/docs/setup/). So I looked for alternatives.
