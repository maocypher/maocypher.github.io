---
layout: post
title:  "Updating Nextcloud on Proxmox LXC with Alpine Linux"
summary: "Updating Nextcloud installed on Proxmox as LXC with Alpine Linux as OS. Installed via community-scripts"
author: maocypher
date: '2025-01-26 00:00:00 +0100'
category: homelab
thumbnail: /assets/img/posts/nextcloud.jpg
keywords: homelab, nextcloud, proxmox, lxc, alpine
permalink: /blog/2025-01-26-update-nextcloud-proxmox/
usemathjax: true
---

## Installing Nextcloud on Proxmox

For installing Nextcloud on my Proxmox VE as LXC with Alpine Linux I used the [Proxmox helper scripts from tteck](https://github.com/tteck/Proxmox) (now continued in the [community scripts](https://community-scripts.github.io/ProxmoxVE/))

I followed `How-to-install` for Alpine-Linux with the [community script](https://community-scripts.github.io/ProxmoxVE/scripts?id=nextcloudpi) and was happy until I did my first try updating Nextcloud to the next major version.

## Upgrading Nextcloud

First I tried to follow the manual from [Nextcloud](https://docs.nextcloud.com/server/latest/admin_manual/maintenance/upgrade.html) to upgrade to the next major version, but it failed completely and I wasn't able to get it running again. So I reverted my changes and installed my backup.

> **CAUTION**
> Make sure you have a valid backup before starting with the upgrade

* Identify the version of your nextcloud hub. You can find this on the Admin UI `Administrator Settings -> Overview -> Version`
* Set nextcloud into `Maintenance mode` by calling `occ maintenance:mode --on`. Unlike on other installation of nextcloud on alpine it wasn't necessary for me to call it like this `sudo -u www-data php occ maintenance:mode --on`. On the one hand `sudo` isn't installed and on the other hand the users are either `nextcloud` or `nginx` depending on the installed nextcloud version.
* With `apk list | grep nextcloud | grep installed` you can list all nextcloud packages that are installed

```shell
# enable maintenance mode
occ maintenance:mode --on

# list current installed packages (here you can already see possible dependency packages that must also be updated)
apk list | grep nextcloud | grep installed
```

> **Hint**
> In my case I only had nextcloud packages (without the version in the package name) which was nextcloud hub 27

> **Tip**
> On [alpine packages](https://pkgs.alpinelinux.org/packages?name=nextcloud*&branch=edge&repo=&arch=x86_64&origin=&flagged=&maintainer=) you can find current available nextcloud packages 

* Install the next major version of nextcloud hub e.g. `apk add nextcloud28` if this runs into issues because `nextcloud` is blocking files you can try running `apk add nextcloud28 --force-overwrite`
* Run `occ upgrade` to start the upgrade process. During this step it might happen that occ cannot finish the update, because apps are not compatible the new version. In that case I disabled all apps by using this command e.g. `occ app:disable files_sharing`
* All the packages you disabled are probably available as alpine package and should be installed manually e.g. `apk add nextcloud29-files_sharing`. Also here when nextcloud was blocking I installed it with `apk add nextcloud29-files_sharing --force-overwrite`
* When you updated all dependencies try running `occ upgrade` it should be successful now.

```shell
# install nextcloud hub; optional use --force-overwrite
apk add nextcloud28

# the first run of this will fail very likely due to dependencies
occ upgrade

# disable dependency packages and upgrade them manually; optional use --force-overwrite if necessary
# repeat this for all of your dependencies
occ app:disable <your dependency package>
apk add nextcloud28-<your dependency package>

# should be successful now
occ upgrade
```

* Finalize the upgrade scripts from nextcloud

```shell
occ db:add-missing-columns
occ db:add-missing-indices
occ db:add-missing-primary-keys
```

* Remove old nextcloud packages e.g.

```shell
# First steps depends on your nextcloud version and installed dependencies
apk del nextcloud-federation nextcloud-notifications nextcloud-files_trashbin nextcloud-dashboard nextcloud-mysql nextcloud-comments nextcloud-admin_audit nextcloud-files_external nextcloud-support nextcloud-weather_status nextcloud-systemtags nextcloud-default-apps nextcloud-files_sharing nextcloud-sharebymail nextcloud-user_status nextcloud-files_versions nextcloud-encryption nextcloud-initscript nextcloud-suspicious_login
apk del nextcloud
```

* Disable the maintenance mode again and try access nextcloud. Depending on the changes that were made by nextcloud you are not done with the upgrade yet and you will fail seeing the login page.

``` shell
occ maintenance:mode --off
```

* When you can login check on Admin-UI which error you should fix in the `/etc/nextcloud/config.php`, because an update of the configuration is needed.

## Nextcloud Hub 28 -> 29

### Specialties
* With the update to nextcloud29 occ has its own package in alpine linux
* From upgrade nextcloud 28 to 29 the php version changed and I had to update some more packages. Make sure to run occ from the target version.

```shell
apk add nextcloud29
apk add nextcloud29-occ

#ensure to use the new occ version
which occ
```

### Post-Upgrade

* Enable maintenance mode

#### PHP 8.3

* Between nextcloud28 and nextcloud29 there were changes in the php version, so I had to install the following php packages

```shell
apk add php83 php83-xml libxml2 libxml2-dev php83-xmlreader php83-xmlwriter php83-simplexml php83-pecl-redis php83-pdo_mysql php83-fpm
```

* Configure php83-fpm `nano /etc/php83/php-fpm.d/www.conf`

```shell
[www]
user = nginx
group = nginx
listen = /run/nextcloud/fastcgi.sock
listen.owner = nginx
listen.group = nginx
listen.mode = 0660
pm = dynamic
pm.max_children = 5
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
env[PATH] = /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
```

* Add to auto start

```shell
rc-service php-fpm83 start
rc-update add php-fpm83 default
```
