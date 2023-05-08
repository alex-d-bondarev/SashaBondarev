---
title: Home file storage server on the Raspberry Pi
toc: true
---

# Background
On summer 2022 I have realized that I have very basic k8s skills and I need a way to improve them. 
I knew that taking k8s related course is not enough and I need to do something practical.
Meanwhile, I had a collection of unordered family videos and photos. My wife's phone was also running out of memory.
As a result I have decided to buy a single Raspberry Pi and to install "any cloud app" there using k8s. 

After a quick search I have stumbled on the [Nextcloud Files](https://nextcloud.com/files/) app. 
It can support desktop and mobile clients. The Nextcloud is also [opensource](https://github.com/nextcloud) 
and provides frequent releases.

My setup includes a 500GB external USB drive for main storage and another 2TB external USB drive for nightly backups.
The smallest case I found that could fit my Raspberry Pi and two external USB drives is [this one](https://thepihut.com/collections/raspberry-pi-cases/products/ssd-cluster-case-for-raspberry-pi). 
Please leave a comment if you know a smaller one. 
The built-in case fans are not too loud, but there was no way to make them turn on and off automatically.
I did not like that, so I have installed this [fan control module](https://thepihut.com/products/auto-fan-control-module-5v-breakout-for-raspberry-pi) 
which allows to use Raspberry Pi OS fan control.   

I did not want to put too much power pressure on my Raspberry Pi, so I bought [this powered USB hub](https://www.amazon.co.uk/Sabrent-Individual-Switches-included-HB-UMP3/dp/B00TPMEOYM?ref_=ast_sto_dp&th=1&psc=1).
It has enough power to run both USB drives and any other USB devices that I sometimes plug in.

I did not and still don't want to change default OS, since it does not entertain me. 
My Raspberry Pi came with the "Raspbian GNU/Linux 11 (bullseye)".

Bellow you can find links, steps and commands I combined to get the "Nextcloud Files" running. 
I'm still not an expert, so please leave a comment if there is a more efficient and/or easy way to perform any given step(s).

> **Please Note**: Some properties, such as "USB device names", on my Raspberry Pi can be different from 
> the ones on your machine. I will use tags such as `<usb-device-name>` to make the steps more reusable.

# Recovery mode
[This gist comment](https://gist.github.com/etes/aa76a6e9c80579872e5f?permalink_comment_id=2781906#gistcomment-2781906) 
saved me a few times from reinstalling OS. I hope you do not need it, but it's always better to be prepared.

# Setup
## Fresh Raspberry Pi
1. Connect a mouse, a keyboard and a monitor to a Raspberry Pi. Follow the prompts.
1. [Optional] Set up the VNC instead of using a mouse, a keyboard and a monitor. 
   I followed [this article](https://linuxhint.com/run-realvnc-raspberry-pi/).   
1. [Optional] Set up SSH connection. I followed [this article](https://linuxhint.com/enable-ssh-raspberry-pi/).
1. [Optional] Update security settings. I followed [this article](https://raspberrytips.com/security-tips-raspberry-pi/).

## Fans setup via GUI
1. Follow "fan control module" manual.
1. Open Raspberry > Preferences > Raspberry Pi Configuration > Performance tab
   1. Set `Fan = true`
   1. Set `Fan GPIO = 4`
   1. Set `Fan Temperature = 60`. This will determine when the fan turns on.
1. [Optional] I followed [this article](https://www.raspberrypi-spy.co.uk/2020/11/raspberry-pi-temperature-monitoring/) 
   to add CPU temperature gauge on the screen with the following colors:
   1. Normal color - `#163c8c`
   1. Warning1 color - `#d68549`
   1. Warning2 color - `#e3001f`


## USB drives
### Device names
You can get the device name like `dev/name` by executing `lsblk`. For example:
```bash
$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
...
sda           8:0    0 465.8G  0 disk
└─sda1        8:1    0 465.8G  0 part /media/500
sdb           8:16   0   1.8T  0 disk
├─sdb1        8:17   0   200M  0 part
└─sdb2        8:18   0   1.8T  0 part /media/2000
...
```
In my case I used `dev/sda1` and `dev/sdb2`. The `<main-usb>` and `<backup-usb>` tags wil be used next instead.

### Format USB
For compatibility reasons, I have decided to format USB drives in ext4:
```bash
sudo umount <main-usb>
sudo umount <backup-usb>
sudo mkfs.ext4 <main-usb>
sudo mkfs.ext4 <backup-usb>
```

