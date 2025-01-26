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

* Enable dependencies again and ensure they are upgraded

```shell
# enable dependencies again
occ app:enable <your dependency package>
occ upgrade
```

* Disable the maintenance mode again and try access nextcloud. Depending on the changes that were made by nextcloud you are not done with the upgrade yet and you will fail seeing the login page.

``` shell
occ maintenance:mode --off
```

* When you can login check on Admin-UI which error you should fix in the `/etc/nextcloud/config.php`, because an update of the configuration is needed.

## Nextcloud Hub 28 to 29

#### OCC
* With the update to `nextcloud29` *occ* has its own package in alpine linux
* From upgrade nextcloud 28 to 29 the php version changed and I had to update some more packages. Make sure to run *occ* from the target version.

```shell
apk add nextcloud29
apk add nextcloud29-occ

#ensure to use the new occ version
which occ
```

### Post-Upgrade

* Enable maintenance mode

#### PHP 8.3

Between `nextcloud28` and `nextcloud29` there were changes in the php version, so I had to install the following php packages

```shell
apk add php83 php83-xml libxml2 libxml2-dev php83-xmlreader php83-xmlwriter php83-simplexml php83-pecl-redis php83-pdo_mysql php83-fpm
```

Configure *php83-fpm* `nano /etc/php83/php-fpm.d/www.conf`

**/etc/php83/php-fpm.d/www.conf**
```config
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
rc-service nginx restart
```

#### Nginx

With nextcloud hub 29 a lot of changes were necessary in nginx config. Older nexcloud nginx config were much smaller. To solve a lot of issues I took the config from [nextcloud manual](https://docs.nextcloud.com/server/latest/admin_manual/installation/nginx.html)

**/etc/nginx/http.d/nextcloud.conf**
```config
upstream php-handler {
#    #server 127.0.0.1:9000;
    server unix:/run/nextcloud/fastcgi.sock;
}

# Set the `immutable` cache control options only for assets with a cache busting `v` argument
map $arg_v $asset_immutable {
    "" "";
    default ", immutable";
}

server {
    listen 80;
    listen [::]:80;
    server_name localhost;

    # Prevent nginx HTTP Server Detection
    server_tokens off;

    # Enforce HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    # With NGinx >= 1.25.1 you should use this instead:
    # listen 443      ssl;
    # listen [::]:443 ssl;
    # http2 on;
    server_name localhost;

    # Path to the root of your installation
    root /usr/share/webapps/nextcloud;

    # Use Mozilla's guidelines for SSL/TLS settings
    # https://mozilla.github.io/server-side-tls/ssl-config-generator/
    ssl_certificate     /etc/ssl/certs/nextcloud-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nextcloud-selfsigned.key;

    # Prevent nginx HTTP Server Detection
    server_tokens off;

    # HSTS settings
    # WARNING: Only add the preload option once you read about
    # the consequences in https://hstspreload.org/. This option
    # will add the domain to a hardcoded list that is shipped
    # in all major browsers and getting removed from this list
    # could take several months.
    add_header Strict-Transport-Security "max-age=15552000; preload" always;

    # set max upload size and increase upload timeout:
    client_max_body_size 512M;
    client_body_timeout 300s;
    fastcgi_buffers 64 4K;

    # Enable gzip but do not remove ETag headers
    gzip on;
    gzip_vary on;
    gzip_comp_level 4;
    gzip_min_length 256;
    gzip_proxied expired no-cache no-store private no_last_modified no_etag auth;
    gzip_types application/atom+xml text/javascript application/javascript application/json application/ld+json application/manifest+json application/rss+xml application/vnd.geo+json application/vnd.ms-fontobject application/wasm application/x-font-ttf application/x-web-app-manifest+json application/xhtml+xml application/xml font/opentype image/bmp image/svg+xml image/x-icon text/cache-manifest text/css text/plain text/vcard text/vnd.rim.location.xloc text/vtt text/x-component text/x-cross-domain-policy;

    # Pagespeed is not supported by Nextcloud, so if your server is built
    # with the `ngx_pagespeed` module, uncomment this line to disable it.
    #pagespeed off;

    # The settings allows you to optimize the HTTP2 bandwidth.
    # See https://blog.cloudflare.com/delivering-http-2-upload-speed-improvements/
    # for tuning hints
    client_body_buffer_size 512k;

    # HTTP response headers borrowed from Nextcloud `.htaccess`
    add_header Referrer-Policy                   "no-referrer"       always;
    add_header X-Content-Type-Options            "nosniff"           always;
    add_header X-Frame-Options                   "SAMEORIGIN"        always;
    add_header X-Permitted-Cross-Domain-Policies "none"              always;
    add_header X-Robots-Tag                      "noindex, nofollow" always;
    add_header X-XSS-Protection                  "1; mode=block"     always;

    # Remove X-Powered-By, which is an information leak
    fastcgi_hide_header X-Powered-By;

    # Set .mjs and .wasm MIME types
    # Either include it in the default mime.types list
    # and include that list explicitly or add the file extension
    # only for Nextcloud like below:
    include mime.types;
    types {
        text/javascript mjs;
        #application/wasm wasm;
    }

    # Specify how to handle directories -- specifying `/index.php$request_uri`
    # here as the fallback means that Nginx always exhibits the desired behaviour
    # when a client requests a path that corresponds to a directory that exists
    # on the server. In particular, if that directory contains an index.php file,
    # that file is correctly served; if it doesn't, then the request is passed to
    # the front-end controller. This consistent behaviour means that we don't need
    # to specify custom rules for certain paths (e.g. images and other assets,
    # `/updater`, `/ocs-provider`), and thus
    # `try_files $uri $uri/ /index.php$request_uri`
    # always provides the desired behaviour.
    index index.php index.html /index.php$request_uri;

    # Rule borrowed from `.htaccess` to handle Microsoft DAV clients
    location = / {
        if ( $http_user_agent ~ ^DavClnt ) {
            return 302 /remote.php/webdav/$is_args$args;
        }
    }

    location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    # Make a regex exception for `/.well-known` so that clients can still
    # access it despite the existence of the regex rule
    # `location ~ /(\.|autotest|...)` which would otherwise handle requests
    # for `/.well-known`.
    location ^~ /.well-known {
        # The rules in this block are an adaptation of the rules
        # in `.htaccess` that concern `/.well-known`.

        location = /.well-known/carddav { return 301 /remote.php/dav/; }
        location = /.well-known/caldav  { return 301 /remote.php/dav/; }

        location /.well-known/acme-challenge    { try_files $uri $uri/ =404; }
        location /.well-known/pki-validation    { try_files $uri $uri/ =404; }

        # Let Nextcloud's API for `/.well-known` URIs handle all other
        # requests by passing them to the front-end controller.
        return 301 /index.php$request_uri;
    }

    # Rules borrowed from `.htaccess` to hide certain paths from clients
    location ~ ^/(?:build|tests|config|lib|3rdparty|templates|data)(?:$|/)  { return 404; }
    location ~ ^/(?:\.|autotest|occ|issue|indie|db_|console)                { return 404; }

    # Ensure this block, which passes PHP files to the PHP process, is above the blocks
    # which handle static assets (as seen below). If this block is not declared first,
    # then Nginx will encounter an infinite rewriting loop when it prepends `/index.php`
    # to the URI, resulting in a HTTP 500 error response.
    location ~ \.php(?:$|/) {
        # Required for legacy support
        rewrite ^/(?!index|remote|public|cron|core\/ajax\/update|status|ocs\/v[12]|updater\/.+|ocs-provider\/.+|.+\/richdocumentscode(_arm64)?\/proxy) /index.php$request_uri;

        fastcgi_split_path_info ^(.+?\.php)(/.*)$;
        set $path_info $fastcgi_path_info;

        try_files $fastcgi_script_name =404;

        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $path_info;
        fastcgi_param HTTPS on;

        fastcgi_param modHeadersAvailable true;         # Avoid sending the security headers twice
        fastcgi_param front_controller_active true;     # Enable pretty urls
        fastcgi_pass php-handler;

        fastcgi_intercept_errors on;
        fastcgi_request_buffering off;

        fastcgi_max_temp_file_size 0;
    }

    # Serve static files
    location ~ \.(?:css|js|mjs|svg|gif|ico|jpg|png|webp|wasm|tflite|map|ogg|flac)$ {
        try_files $uri /index.php$request_uri;
        # HTTP response headers borrowed from Nextcloud `.htaccess`
        add_header Cache-Control                     "public, max-age=15778463$asset_immutable";
        add_header Referrer-Policy                   "no-referrer"       always;
        add_header X-Content-Type-Options            "nosniff"           always;
        add_header X-Frame-Options                   "SAMEORIGIN"        always;
        add_header X-Permitted-Cross-Domain-Policies "none"              always;
        add_header X-Robots-Tag                      "noindex, nofollow" always;
        add_header X-XSS-Protection                  "1; mode=block"     always;
        access_log off;     # Optional: Don't log access to assets
    }

    location ~ \.(otf|woff2?)$ {
        try_files $uri /index.php$request_uri;
        expires 7d;         # Cache-Control policy borrowed from `.htaccess`
        access_log off;     # Optional: Don't log access to assets
    }

    # Rule borrowed from `.htaccess`
    location /remote {
        return 301 /remote.php$request_uri;
    }

    location / {
        try_files $uri $uri/ /index.php$request_uri;
    }
}
```

After changing the config restart *nginx* and *php-fpm83*

```shell
rc-service php-fpm83 restart
rc-service nginx restart
```

#### Fix owner

For me it was necessary to update the owner of some folders that were changed to root user after the upgrade (or were root before the upgrade)

```shell
chown nginx:nginx /run/nextcloud
chown -R nextcloud:www-data /usr/share/webapps/nextcloud
chown nextcloud:www-data /etc/nextcloud/config.php

chown -R nginx:www-data /var/lib/nextcloud
```

## Nextcloud Hub 29 to 30

#### OCC

Ensure to use the occ from the latest version

```shell
apk add nextcloud30
apk add nextcloud30-occ

#ensure to use the new occ version
which occ
```

#### Additional php packages

When updating to `nextcloud30` additional php packages are needed

```shell
apk add php83-pecl-imagick php83-exif php83-sodium php83-sysvsem
rc-service php-fpm83 restar
```

#### Cron-Jobs not running

If you run into issues that the cron jobs are not running regularily after the update check your configured cron-tabs

```shell
crontab -e

# It should contain these entries
*/5  *  *  *  * sudo -u nextcloud php83 -f /usr/share/webapps/nextcloud/cron.php
0 2 * * * sudo -u nextcloud occ integrity:check-core > /var/log/nextcloud-integrity.log 2>&1
```