---
layout: post
title:  "Save Nextcloud data when Proxmox backup is too big to be restored"
summary: "Save Nextcloud data installed on Proxmox as LXC with Alpine Linux as OS. Installed via community-scripts"
author: maocypher
date: '2025-12-31 00:00:00 +0100'
category: homelab
thumbnail: /assets/img/posts/nextcloud.jpg
keywords: homelab, nextcloud, proxmox, lxc, alpine, recovery
permalink: /blog/2025-12-31-save-nextcloud-data-when-backup-too-big/
usemathjax: true
---

## What happend?

Until I had my update-day for my homelab I run Nextcloud on Alpine Linux without issues as VM on my Proxmox. I had daily backups that were created and stored in my Proxmox Backup Server. Everything was fine.
In the beginning of 2025 I wrote a guide "how to update Nextcloud on Alpine Linux". So, I took that guide and did everything as usual, but in the end I was not able to fully upgrade Nextcloud. At some point it was impossible to continue.

Smart as I was, I stopped to continue the installation and wanted to restore the backup I had. BUT I was not aware of the fact, that my backup was to big to be restored with the remaining disk space I had. What happend next, was that after triggering the restore of my backup, the existing VM was simply gone...

## The plan

As I wanted to update Nextcloud and my VM was now gone, I needed a new plan how to proceed from here. I didn't want to run into the same issue again, when I continue using Alpine Linux, so I decided to install a new VM using the [TurnKey VM](https://community-scripts.github.io/ProxmoxVE/scripts?id=nextcloud-vm) provided by the community-scripts. My disk with all the data still existed (btw. the disk size was 2TB and the compressed backup like 450GB), so I tried to create a brand new VM and load the data as additional disk into the VM.

## Recovering Nextcloud data

- I installed Nextcloud in a new VM (with a different ID) from the [community-scripts](https://community-scripts.github.io/ProxmoxVE/scripts?id=nextcloud-vm).
- After the successful installation I first updated to the latest Nextcloud version via the included updater
- I created all existing users (for me it was possible to create them manually again, because I only have 2 users at home)
- Then the interessting part started, recovering the user data
  - Shutdown Nextcloud
  - Open proxmox shell and add the old storage to the Nextcloud VM

```shell
qm set <new vm id> --scsi2 <your storage e.g. local>:vm-<old id>-disk-0
# qm set 102 --scsi2 local:vm-101-disk-0
```

- After adding the disk to the VM, it can be started again
- Next it is necessary to mount the disk in the right place

```shell
# create a folder to mount the disk temporarily
cd /mnt
mkdir -p nextcloud-legacy

# check the path of disk, for me it was /dev/sdb for you it might be different
lsblk
mount /dev/sdb /mnt/nextcloud-legacy

# copy current nextcloud data
cp -r /var/www/nextcloud-data/* /mnt/nextcloud-legacy

# check that all data were copied, if not copy them manually, e.g.
cp /var/www/nextcloud-data/.htaccess /mnt/nextcloud-legacy
cp /var/www/nextcloud-data/.ncdata /mnt/nextcloud-legacy

# change the owner to match the needed owner to access the data
chown -R www-data:www-data /mnt/nextcloud-legacy/*

# identify the disk for fstab usage
blkid /dev/sdb
umount /dev/sdb

nvim /etc/fstab
```

- With that we are now good to go and replace the current data with the recovered data
- Next we should mount the data automatically on reboot

_/etc/fstab_
```plaintext
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
/dev/mapper/turnkey-root /               ext4    errors=remount-ro 0       1
/dev/mapper/turnkey-swap_1 none            swap    sw              0       0
UUID="<your GUID from blkid>" /var/www/nextcloud-data ext4 defaults 0 2
```

- Now reboot your VM
- After rebooting you should see in `lsblk` that your disk is mounted to `/var/www/nextcloud-data`
- Data should be accessable via WebUi as before
- Now we restore the user data to be accessable

```shell
# Now you should know where the old data was stored. For me it was: /var/lib/nextcloud/data on the mounted disk
cd /var/www/nextcloud-data

# I backuped the current data temporarily
mv <username>/ <username>-backup

# Restore the old data
mv -r var/lib/nextcloud/data/<username> ./
chown -R www-data:www-data <username>

# Make nextcloud aware of all the data, these creates needed database entries
turnkey-occ files:scan -vv -p /<username>/files <username>
```

- The scan takes a while depending on the amount of data
- After a successful scan you should now see all your data again in the WebUi
- The calendar I restored from my exported .ics file via my mobile phone
- All data from the old Alpine Linux OS can be removed savely
