---
title: Home file storage server on the Raspberry Pi
toc: true
---

# Introduction
Hi everyone! If you want to set up a home file storage server, this post can help. 
I've left out some details assuming you know basic bash, and I use `${FOO_BAR}` style variables for flexibility. 
Although I haven't tested these steps on other systems, I believe they'll work beyond the Raspberry Pi.

Also, I want to highlight [this gist comment](https://gist.github.com/etes/aa76a6e9c80579872e5f?permalink_comment_id=2781906#gistcomment-2781906)
which has saved me from having to reinstall my operating system several times. 
Hopefully, you won't need it, but it's always good to be prepared.

# Background
In the summer of 2022, I aimed to improve my basic k8s skills while syncing 250GB of family photos and videos across 
devices. I discovered the [Nextcloud Files](https://nextcloud.com/files/) app, which is 
[open-source](https://github.com/nextcloud), regularly updated, and supports desktop and mobile clients. 
Though it offers office collaboration features, they weren't a priority for me.

For my setup, I used two external USB drives — a smaller one for storing files and a larger one for backups. 
I found [this compact](https://thepihut.com/collections/raspberry-pi-cases/products/ssd-cluster-case-for-raspberry-pi) 
case that accommodated my Raspberry Pi and both drives. If you know of a smaller case that can 
handle this setup, please let me know. To control the case fans, I utilized this 
[fan control module](https://thepihut.com/products/auto-fan-control-module-5v-breakout-for-raspberry-pi).

To avoid overburdening my Raspberry Pi, I purchased [this USB hub with a power adapter](https://www.amazon.co.uk/Sabrent-Individual-Switches-included-HB-UMP3/dp/B00TPMEOYM?ref_=ast_sto_dp&th=1&psc=1). 
I opted to stick with the default "Raspbian GNU/Linux 11 (bullseye)" OS, as changing it wasn't appealing to me.

Below, you'll find the steps I took to get "Nextcloud Files" up and running. 
I'm not an expert, so if you have any tips for a better or easier process, please share them in the comments.

# Installation
## Fresh Raspberry Pi
1. Connect a mouse, a keyboard and a monitor to a Raspberry Pi. Follow the prompts.
1. Install [helm](https://helm.sh/docs/intro/install/).
1. [Optional] Set up the VNC instead of using peripherals per [this article](https://linuxhint.com/run-realvnc-raspberry-pi/).   
1. [Optional] Set up SSH connection per [this article](https://linuxhint.com/enable-ssh-raspberry-pi/).
1. [Optional] Update security settings per [this article](https://raspberrytips.com/security-tips-raspberry-pi/).

## Fans setup via GUI
1. Follow the "fan control module" manual.
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
```
**Note** that you might have different values  
${MAIN_USB} = "dev/sda1"  
${BACKUP_USB} = "dev/sdb2"
{: .notice--info}

### Format USB
I have decided to format USB drives in `ext4`:
```console
$ sudo umount ${MAIN_USB}
$ sudo mkfs.ext4 ${MAIN_USB}

$ sudo umount ${BACKUP_USB}
$ sudo mkfs.ext4 ${BACKUP_USB}
```

### Mount USB drives
**New variables**  
${BACKUP_USB_MOUNT} - path to the folder with the backup USB  
${BACKUP_USB_UUID} - UUID from the `blkid ${BACKUP_USB}`  
${MAIN_USB_MOUNT} - path to the folder with the main USB  
${MAIN_USB_UUID} - UUID from the `blkid ${MAIN_USB}`  
${USB_MOUNT_SCRIPT} - path to file with mount script  
{: .notice--info}

Sometimes USB drives did not mount after a restart, so I took a few extra steps:
> **Note**: First, I tried to mount using fstab, the following way:
> ```console
> $ sudo cp /etc/fstab /etc/fstab.backup
> ```
> Add the following to "/etc/fstab":
> ```shell
> UUID="${MAIN_USB_UUID}" ${MAIN_USB_MOUNT} ext4 defaults,nofail,x-systemd.device-timeout=1,noatime  0       0
> UUID="${BACKUP_USB_UUID}" ${BACKUP_USB_MOUNT} ext4 defaults,nofail,x-systemd.device-timeout=1,noatime  0       0
> ```
> ```console
> $ sudo reboot
> ```
> But USB drives often failed to mount during boot, so I used the approach from below:

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
I have decided to use [borg](https://borgbackup.readthedocs.io) for backups.

**New variables**  
${BACKUP_LOG} - path to file with backup logs  
${BACKUP_NAME} - the name of the backup to use in `borg`  
${BACKUP_PATH} - path to the folder that will contain backups  
${BACKUP_SCRIPT} - path to file with the backup script  
${NEXTCLOUD_PATH} - path to the folder that will contain NextCloud and all related files  
{: .notice--info}

`${NEXTCLOUD_PATH}` is path to files and `${BACKUP_PATH}` is path to backups.

### Set up borg repository
```console
$ touch ${BACKUP_LOG}

$ sudo apt install borgbackup
$ sudo su -

# Setup
$ borg init --storage-quota 1.5T --encryption=none ${BACKUP_PATH} 2> ${BACKUP_LOG}
$ borg create --compression auto,zstd ${BACKUP_PATH}::${BACKUP_NAME}`date +%Y-%m-%d_%H:%M` ${NEXTCLOUD_PATH} 2> ${BACKUP_LOG}

# To restore a backup:
$ borg list ${BACKUP_PATH}  # See available backups
$ borg extract ${BACKUP_PATH}::${BACKUP_NAME} ${NEXTCLOUD_PATH} # Restore latest backup
$ borg extract --timestamp 2022-01-01T12:00:00 ${BACKUP_PATH}::${BACKUP_NAME} ${NEXTCLOUD_PATH} # Restore per timestamp

$ exit
```

### Set up periodic backups
Create a `${BACKUP_SCRIPT}` file with the following content:
```shell
#!/bin/sh

echo "Backup started on `date +%Y-%m-%d_%H:%M`" >> ${BACKUP_LOG} 2>&1
borg prune --keep-daily=7 --keep-weekly=4 --keep-monthly=6 ${BACKUP_PATH} >> ${BACKUP_LOG} 2>&1
borg create --compression auto,zstd ${BACKUP_PATH}::${BACKUP_NAME}`date +%Y-%m-%d_%H:%M` ${NEXTCLOUD_PATH} >> ${BACKUP_LOG} 2>&1
echo "Backup finished on `date +%Y-%m-%d_%H:%M`" >> ${BACKUP_LOG} 2>&1
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
**New variables**  
${NAMESPACE} - Kubernetes namespace for NextCloud  
{: .notice--info}

I did not plan to learn to [install vanilla k8s](https://kubernetes.io/docs/setup/). So I looked for alternatives:
1. [Minikube](https://minikube.sigs.k8s.io/docs/) - is not supporting mounts with > 600 files by default 
   ([link](https://minikube.sigs.k8s.io/docs/handbook/mount/)). That was too few for me.
1. I have got the error when trying to install [MicroK8s](https://microk8s.io) and failed to fix it:
   ```shell
   # error: snap "microk8s" is not available on 1.21/stable for this architecture (armhf) but exists on
   #        other architectures (amd64, arm64).
   ```
1. I did not face issues with [k3s](https://k3s.io):
   ```console
   $ curl -sfL https://get.k3s.io | sh -
   ```

### Configure k3s
1. Add `cgroup_memory=1 cgroup_enable=memory` in the end of `/boot/cmdline.txt` without adding any new lines
1. Reboot the Raspberry Pi.
1. Confirm install was successful:
   ```console
   $ sudo kubectl get nodes
   # Expect the output similar to
   NAME          STATUS   ROLES                  AGE   VERSION
   raspberrypi   Ready    control-plane,master   56d   v1.25.7+k3s1
   ```
1. Add the following to the `/etc/rancher/k3s/config.yaml` to allow running `kubectl` without sudo:
   ```shell
   write-kubeconfig-mode: "0644"
   ```
1. Reboot the Raspberry Pi.
1. Confirm the change was successful:
   ```console
   $ kubectl get nodes
   ```
1. Create a namespace for the NextCloud:
   ```console
   $ kubectl create namespace ${NAMESPACE}
   $ kubectl get namespace
   NAME              STATUS   AGE
   default           Active   56d
   kube-system       Active   56d
   kube-public       Active   56d
   kube-node-lease   Active   56d
   ${NAMESPACE}      Active   48d
   ```

### Storage
**New variables**  
${PERSISTENT_VOLUME_CLAIM_NAME} - PersistentVolumeClaim name. For example `next-cloud-volume-claim`  
${PERSISTENT_VOLUME_CLAIM_YAML} - Path to `.yaml` file for PersistentVolumeClaim  
${PERSISTENT_VOLUME_NAME} - PersistentVolume name. For example `next-cloud-volume`  
${PERSISTENT_VOLUME_YAML} - Path to `.yaml` file for PersistentVolume  
${STORAGE_SIZE} - NextCloud storage size. For example `123Gi`  
{: .notice--info}

You will need to create `PersistentVolume` and `PersistentVolumeClaim` for the NextCloud

#### PersistentVolume
Create `${PERSISTENT_VOLUME_YAML}.yml` file with the following contents:
```yaml
---
apiVersion: v1
kind: PersistentVolume
metadata:
  namespace: "${NAMESPACE}"
  name: "${PERSISTENT_VOLUME_NAME}"
  labels:
    type: "local"
spec:
  storageClassName: "manual"
  capacity:
    storage: "${STORAGE_SIZE}"
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "${NEXTCLOUD_PATH}"
---
```
Apply the file:
```console
$ kubectl apply -f ${PERSISTENT_VOLUME_YAML}
$ kubectl get pv
```

#### PersistentVolumeClaim
Create `${PERSISTENT_VOLUME_CLAIM_YAML}` file with the following contents:
```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  namespace: "${NAMESPACE}"
  name: "${PERSISTENT_VOLUME_CLAIM_NAME}"
spec:
  storageClassName: "manual"
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: "${STORAGE_SIZE}"
---
```
Apply the file:
```console
$ kubectl apply -f ${PERSISTENT_VOLUME_CLAIM_YAML}
$ kubectl get pvc -n "${NAMESPACE}"
```

## NextCloud
**New variables**  
${ADMIN_NAME} - Nextcloud admin name.  
${ADMIN_PASSWORD} - Nextcloud admin password.  
${NEXTCLOUD_POD_NAME} - k8s NextCloud pod name. Get it like `kubectl get pods -n ${NAMESPACE}`  
${NEXTCLOUD_VALUES_YAML} - Path to `.yaml` file for NextCloud config values.  
${RASPBERRY_PI_IP} - IP of your Raspberry Pi in the local network.  
{: .notice--info}
### Install

Prepare for the NextCloud installation:
```console
$ helm repo add nextcloud https://nextcloud.github.io/helm/
$ helm show values nextcloud/nextcloud >> ${NEXTCLOUD_VALUES_YAML}
```

Update the following lines in the `${NEXTCLOUD_VALUES_YAML}`:
```yaml
---
...
nextcloud:
  username: ${ADMIN_NAME}
  password: ${ADMIN_PASSWORD}
...
persistence:
  enabled: true # Change to true
  existingClaim: "${PERSISTENT_VOLUME_CLAIM_NAME}"
  accessMode: ReadWriteMany
  size: "${STORAGE_SIZE}"
...
---
```

Install NextCloud:
```console
$ export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
$ helm install nextcloud nextcloud/nextcloud \
  --namespace ${NAMESPACE} \
  --values ${NEXTCLOUD_VALUES_YAML}
```

Check that the installation was successful:
```console
$ kubectl get services -n next-cloud
```
Open `http://<CLUSTER-IP>:<PORT(S)>` using the Raspberry Pi web browser locally or via VNC.

### Access
#### Config
Append Raspberry Pi IP to `trusted_domains` in the `${NEXTCLOUD_PATH}/config/config.php` the following way:
```php
...
'trusted_domains' =>
  array (
    0 => 'localhost',
    1 => 'nextcloud.kube.home',
    2 => '${RASPBERRY_PI_IP}',
  ),
...
```
#### Expose locally
```console
$ kubectl expose service nextcloud \
  --target-port 80 \
  --port 8080 \
  --name nextcloud-exp \
  --type=LoadBalancer \
  -n ${NAMESPACE}
```

## Final Tweaks
1. Open `http://${RASPBERRY_PI_IP}:8080`
1. Login with `${ADMIN_NAME}` and `${ADMIN_PASSWORD}`
1. Create non-admin users
1. Done

## Maintenance
I have run into some issues when I was uploading and deleting many files at once.
To fix them I ran some commands on the `nextcloud` pod:
```console
$ kubectl exec --stdin --tty ${NEXTCLOUD_POD_NAME} -n ${NAMESPACE} -- bash
$ su -s /bin/bash www-data
$ cd /var/www/html
```
### File Locks
Sometimes I could not view, update or delete files and/or folders. I have fixed this by running:
```console
$ php occ files:scan --all
$ php occ files:cleanup
```

### Deleted Files Errors
Sometimes error was thrown for me when I opened the "Deleted files" page. I have fixed this by running:
```console
$ php occ trashbin:cleanup --all-users
```
