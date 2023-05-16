---
title: Home file storage server on the Raspberry Pi
toc: true
date: 2023-05-15 22:00:00 +0100
categories: [Raspberry Pi, NextCloud, k8s, k3s, borg, file storage]
---

# Introduction
Hi everyone! Setting up a home file server that syncs with family devices and backs up regularly can take an evening. 
All you need is basic bash and k8s skills, and knowledge of the related software. I created a list of steps for myself 
and am sharing it here to help others with similar goals. 

I've only tested these steps on a Raspberry Pi with the default "Raspbian GNU/Linux 11 (bullseye)" OS, but I think they 
should work on most Linux-based systems with only a few tweaks. Please leave a comment if you have any tips on how to 
simplify or improve the steps. 

# Installation
## Notes
1. To make things clearer for your specific setup, I used `${FOO_BAR}` style variables to indicate places where you 
   might need to use different devices, folder names, etc.
1. [This gist comment](https://gist.github.com/etes/aa76a6e9c80579872e5f?permalink_comment_id=2781906#gistcomment-2781906)
   has saved me from having to reinstall the OS several times. 
1. I am using the [Nextcloud Files](https://nextcloud.com/files/) software. It is [open-source](https://github.com/nextcloud), 
   regularly updated, and supports desktop and mobile clients. It also offers office collaboration features and other addons.
1. I am using two external USB drives — a smaller one for storing files and a larger one for backups. 
1. I am using [this](https://thepihut.com/collections/raspberry-pi-cases/products/ssd-cluster-case-for-raspberry-pi),
   more or less, compact case that accommodated my Raspberry Pi and both USB drives. Please share a link to a smaller 
   case that can fit them, plus a fan if you know one. 
1. I am using this [fan control module](https://thepihut.com/products/auto-fan-control-module-5v-breakout-for-raspberry-pi) 
   to enable fan only when my Raspberry Pi becomes too hot.
1. I am using this [USB hub with a power adapter](https://www.amazon.co.uk/Sabrent-Individual-Switches-included-HB-UMP3/dp/B00TPMEOYM?ref_=ast_sto_dp&th=1&psc=1)
   to avoid Raspberry Pi power overburdening by external USB drives.

## Fresh Raspberry Pi
1. Connect a mouse, a keyboard and a monitor to a Raspberry Pi. Follow the prompts.
1. Install a [helm](https://helm.sh/docs/intro/install/).
1. [Optional] Set up a VNC instead of using peripherals per [this article](https://linuxhint.com/run-realvnc-raspberry-pi/).   
1. [Optional] Set up an SSH connection per [this article](https://linuxhint.com/enable-ssh-raspberry-pi/).
1. [Optional] Update security settings per [this article](https://raspberrytips.com/security-tips-raspberry-pi/).

## Fans setup via GUI
1. Follow the [fan control module manual](https://cdn.shopify.com/s/files/1/0176/3274/files/HATL01F-2.pdf?v=1629391085).
1. Open Raspberry > Preferences > Raspberry Pi Configuration > Performance tab
   1. Set `Fan = true`
   1. Set `Fan GPIO = 4`
   1. Set `Fan Temperature = 60`. In this case the fan will turn on when the temperature hits 60C. You can set higher temperature if needed.
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
**New variables**  (you will probably have different values)  
`${MAIN_USB}` = "dev/sda1"  
`${BACKUP_USB}` = "dev/sdb2"  
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
`${BACKUP_USB_MOUNT}` - path to the folder with the backup USB.  
`${BACKUP_USB_UUID}` - UUID from the `blkid ${BACKUP_USB}`.  
`${MAIN_USB_MOUNT}` - path to the folder with the main USB.  
`${MAIN_USB_UUID}` - UUID from the `blkid ${MAIN_USB}`.  
`${USB_MOUNT_SCRIPT}` - path to file with mount script.  
{: .notice--info}

Sometimes USB drives did not mount after a restart, so I took a few extra steps:
> **Note**: First, I tried to mount using fstab, the following way:
> ```console
> $ sudo cp /etc/fstab /etc/fstab.backup
> ```
> Add the following to the `/etc/fstab`:
> ```shell
> UUID="${MAIN_USB_UUID}" ${MAIN_USB_MOUNT} ext4 defaults,nofail,x-systemd.device-timeout=1,noatime  0       0
> UUID="${BACKUP_USB_UUID}" ${BACKUP_USB_MOUNT} ext4 defaults,nofail,x-systemd.device-timeout=1,noatime  0       0
> ```
> ```console
> $ sudo reboot
> ```
> But USB drives often failed to mount during the boot. So, inspired by [this comment](https://superuser.com/a/547124) 
> and [this article](https://www.baeldung.com/linux/new-files-dirs-default-permission) I used the following approach:

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
Update a crontab
```console
$ sudo chmod +x ${USB_MOUNT_SCRIPT}
$ sudo crontab -e
```
Add the following:
```shell
@reboot ${USB_MOUNT_SCRIPT}>/tmp/usb_mount.log
```

## Backup
I have decided to use [borg](https://borgbackup.readthedocs.io) for backups.

**New variables**  
`${BACKUP_LOG}` - path to file with backup logs.  
`${BACKUP_NAME}` - the name of the backup to use in `borg`.  
`${BACKUP_PATH}` - path to the folder that will contain backups (you should create an empty folder).  
`${BACKUP_SCRIPT}` - path to file with the backup script.  
`${MAX_BACKUP_SPACE}` - How much space to allocate for backups. This number should at least equal, 
but preferably be several times larger than the main storage. For example `1.5T` when main storage is `500G`.   
`${NEXTCLOUD_PATH}` - path to the folder that will contain NextCloud and all related files (you should create an empty folder).  
{: .notice--info}

### Set up borg repository
```console
$ touch ${BACKUP_LOG}

$ sudo apt install borgbackup
$ sudo su -

# Setup
$ borg init --storage-quota ${MAX_BACKUP_SPACE} --encryption=none ${BACKUP_PATH} 2> ${BACKUP_LOG}
$ borg create --compression auto,zstd ${BACKUP_PATH}::${BACKUP_NAME}`date +%Y-%m-%d_%H:%M` ${NEXTCLOUD_PATH} 2> ${BACKUP_LOG}

# To restore a backup:
$ borg list ${BACKUP_PATH}  # See available backups
$ borg extract ${BACKUP_PATH}::${BACKUP_NAME} ${NEXTCLOUD_PATH} # Restore latest backup
$ borg extract --timestamp 2022-01-01T12:00:00 ${BACKUP_PATH}::${BACKUP_NAME} ${NEXTCLOUD_PATH} # Restore per timestamp

$ exit
```

**Note:** I have decided to set up backups without encryption. But borg recommends to use one. You can find more 
details [here](https://borgbackup.readthedocs.io/en/stable/usage/init.html#encryption-mode-tldr) 
and [here](https://github.com/borgbackup/borg/issues/5285).  

### Set up periodic backups
Create a `${BACKUP_SCRIPT}` file with the following content:
```shell
#!/bin/sh

echo "Backup started on `date +%Y-%m-%d_%H:%M`" >> ${BACKUP_LOG} 2>&1
borg prune --keep-daily=7 --keep-weekly=4 --keep-monthly=6 ${BACKUP_PATH} >> ${BACKUP_LOG} 2>&1
borg create --compression auto,zstd ${BACKUP_PATH}::${BACKUP_NAME}`date +%Y-%m-%d_%H:%M` ${NEXTCLOUD_PATH} >> ${BACKUP_LOG} 2>&1
echo "Backup finished on `date +%Y-%m-%d_%H:%M`" >> ${BACKUP_LOG} 2>&1
```
Update the crontab
```console
$ sudo chmod +x ${BACKUP_SCRIPT}
$ sudo crontab -e
```
Add the following:
```shell
0 0 * * * /usr/bin/sh ${BACKUP_SCRIPT}
```

## Install the k8s
**New variables**  
`${NAMESPACE}` - Kubernetes namespace for NextCloud.  
{: .notice--info}

I did not plan to learn to [install vanilla k8s](https://kubernetes.io/docs/setup/). So I looked for alternatives:
1. [Minikube](https://minikube.sigs.k8s.io/docs/) - is not supporting mounts with > 600 files by default 
   ([link](https://minikube.sigs.k8s.io/docs/handbook/mount/)). That was too few for me.
1. I have got the error when trying to install [MicroK8s](https://microk8s.io) and failed to fix it:
   ```shell
   # error: snap "microk8s" is not available on 1.21/stable for this architecture (armhf) but exists on
   #        other architectures (amd64, arm64).
   ```
1. I did not face any blocking issues with [k3s](https://k3s.io):
   ```console
   $ curl -sfL https://get.k3s.io | sh -
   ```

### Configure k3s
1. Add `cgroup_memory=1 cgroup_enable=memory` in the end of the `/boot/cmdline.txt` without adding any new lines
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

#### High CPU Usage
Apply [this fix](https://github.com/k3s-io/k3s/issues/4593#issuecomment-1409260954) in case you see high CPU usage:
```console
$ sudo update-alternatives --set iptables /usr/sbin/iptables-legacy
$ sudo update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
```

### Storage
You will need to create `PersistentVolume` and `PersistentVolumeClaim` for the NextCloud

**New variables**  
`${PERSISTENT_VOLUME_CLAIM_NAME}` - PersistentVolumeClaim name. For example `next-cloud-volume-claim`.  
`${PERSISTENT_VOLUME_CLAIM_YAML}` - Path to `.yaml` file for PersistentVolumeClaim.  
`${PERSISTENT_VOLUME_NAME}` - PersistentVolume name. For example `next-cloud-volume`.  
`${PERSISTENT_VOLUME_YAML}` - Path to `.yaml` file for PersistentVolume.  
`${STORAGE_SIZE}` - NextCloud (main) storage size. For example `123Gi`. This is a reminder that it should be equal or 
less than `${MAX_BACKUP_SPACE}`.  
{: .notice--info}

#### PersistentVolume
Create a `${PERSISTENT_VOLUME_YAML}` file with the following content:
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
Create a `${PERSISTENT_VOLUME_CLAIM_YAML}` file with the following content:
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
`${ADMIN_NAME}` - Nextcloud admin name.  
`${ADMIN_PASSWORD}` - Nextcloud admin password.  
`${NEXTCLOUD_VALUES_YAML}` - Path to `.yaml` file for NextCloud config values.  
`${RASPBERRY_PI_IP}` - IP of your Raspberry Pi in the local network.  
{: .notice--info}
### Install the NextCloud

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
  enabled: true
  existingClaim: "${PERSISTENT_VOLUME_CLAIM_NAME}"
  accessMode: ReadWriteMany
  size: "${STORAGE_SIZE}"
...
---
```

Install the NextCloud:
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
Append Raspberry Pi IP to the `trusted_domains` in the `${NEXTCLOUD_PATH}/config/config.php` the following way:
```php
...
'trusted_domains' =>
  array (
    0 => 'localhost',
    1 => 'nextcloud.kube.home', # You might have a different default value in this line
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

## Create Users
1. Open `http://${RASPBERRY_PI_IP}:8080`
1. Login with `${ADMIN_NAME}` and `${ADMIN_PASSWORD}`
1. Create non-admin users
1. Now, at home, you can log in to the NextCloud via browser using `http://${RASPBERRY_PI_IP}:8080` or via 
   [supported clients](https://nextcloud.com/clients). 

## Maintenance
**New variables**  
`${NEXTCLOUD_POD_NAME}` - k8s NextCloud pod name. Get it like `kubectl get pods -n ${NAMESPACE}`.  
{: .notice--info}

I have run into some issues when I was uploading and deleting many files at once.
I fixed them by running some commands on the `nextcloud` pod:
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

# Other References
This is it. Here are a few more references that I used:
- [Advanced Networking with Kubernetes on AWS](https://learn.acloud.guru/course/advanced-networking-with-kubernetes-for-aws/dashboard)
- [Deploy NextCloud on Kubernetes. The self-hosted Dropbox](https://greg.jeanmart.me/2020/04/13/deploy-nextcloud-on-kuberbetes--the-self-hos/)
- [How to build a Raspberry Pi Kubernetes Cluster with k3s](https://medium.com/thinkport/how-to-build-a-raspberry-pi-kubernetes-cluster-with-k3s-76224788576c)
- [How to share storage between Kubernetes pods?](https://stackoverflow.com/a/36524584)
- [Introduction to K3s](https://learn.acloud.guru/course/introduction-to-k3s/dashboard)
- [Introduction to Kubernetes](https://learn.acloud.guru/course/introduction-to-kubernetes)
- [Introduction to Rancher](https://learn.acloud.guru/course/introduction-to-rancher/dashboard)
- [Kubernetes Deep Dive](https://learn.acloud.guru/course/kubernetes-deep-dive/dashboard)
- [Kubernetes Essentials](https://learn.acloud.guru/course/2e0bad96-a602-4c91-9da2-e757d32abb8f/dashboard)
- [Kubernetes Quick Start](https://learn.acloud.guru/course/9a0082c5-5331-492d-a677-173c393a85f7/dashboard)
- [Simplest way to access internal resources in a K3S installation](https://stackoverflow.com/a/72554140)
- [k3s configuration options](https://docs.k3s.io/installation/configuration)
