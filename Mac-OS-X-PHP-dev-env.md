# Mac OS X - PHP developer environment.
How to:
- Homebrew, Composer, Symfony installer
- NGINX with autostart afret reboot on port *:80
- PHP-FPM (5.6+)
- NGINX + PHP-FPM + Symfony2 minimal config


## Preparation

### Install [Homebrew](http://brew.sh/)

    ruby -e "$(curl -fsSL https://raw.github.com/mxcl/homebrew/go/install)"

### Install (globally) [Composer](http://etcomposer.org/)

    curl -sS https://getcomposer.org/installer | php
    mv composer.phar /usr/local/bin/composer
>**Note**: If the above fails due to permissions, run the mv line again with sudo.  
>**Note**: In OSX Yosemite the /usr directory does not exist by default. If you receive the error "/usr/local/bin/composer: No such file or directory" then you must create /usr/local/bin/ manually before proceeding.

Then, just run `composer` in order to run Composer instead of `php composer.phar`.

### Install [Symfony installer](http://symfony.com/)

    sudo curl -LsS http://symfony.com/installer -o /usr/local/bin/symfony
    sudo chmod a+x /usr/local/bin/symfony

This will create a global `symfony` command in your system.

### Appearance
Optionally, you can change the appearance of the terminal.  
I prefer [Flat UI Terminal Theme](https://dribbble.com/shots/1021755-Flat-Terminal-Theme)


## NGINX

### Install

    brew install nginx

Start nginx with launchctl, when your Mac boots up

    sudo cp /usr/local/opt/nginx/*.plist /Library/LaunchDaemons
    sudo launchctl load -w /Library/LaunchDaemons/homebrew.mxcl.nginx.plist

>You need to put your plist file in `/Library/LaunchDaemons`, not in `~/Library/LaunchAgents` like the Homebrew instructions say to do. You also need to **copy the file over**, not just symlink it - again, going against the Homebrew instructions. Lastly, use the `-w` option with launchctl.

### Configuration

First prepare follow dirs

    mkdir /usr/local/etc/nginx/sites-available
    mkdir /usr/local/etc/nginx/sites-enabled

nginx configuration files can be found here `vi /usr/local/etc/nginx/nginx.conf`

Here is example `nginx.conf` file (do not forgot change root path):

    worker_processes  1;

    events {
        worker_connections  1024;
    }

    http {
        include       mime.types;
        include       sites-enabled/*; # load virtuals config
        sendfile        on;
        keepalive_timeout  65;

        server {
            listen       80;
            server_name  localhost;

            location / {
                root  /Users/pawelkolanowski/www;
                try_files  $uri  $uri/  /index.php?$args ;
                index index.html index.php;
            }

            # configure *.PHP requests

            location ~ \.php$ {
                root  /Users/pawelkolanowski/www;
                try_files  $uri  $uri/  /index.php?$args ;
                index  index.html index.htm index.php;
                fastcgi_param PATH_INFO $fastcgi_path_info;
                fastcgi_param PATH_TRANSLATED $document_root$fastcgi_path_info;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;

                fastcgi_pass 127.0.0.1:9000;
                fastcgi_index index.php;
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                fastcgi_intercept_errors on;
                include fastcgi_params;
            }
        }
    }

### Running

You can start nginx manually and check if running in browser

    sudo nginx            # start
    sudo nginx -s stop    # stop
    sudo nginx -s reload  # restart

Check if running `http://localhost`


## PHP-FPM

### Install

Start with taping formulas repositories:

    brew tap homebrew/dupes
    brew tap homebrew/versions
    brew tap homebrew/homebrew-php

Remove all PHP dependencies (it's only safe way how to compile PHP successfully)

    brew remove libtool freetype gettext icu4c jpeg libpng unixodbc zlib

Then install PHP

    brew install -v --with-fpm --with-mysql --disable-opcache php56
    
Launch after login

    ln -sfv /usr/local/opt/php56/*.plist ~/Library/LaunchAgents

### Install PHP extensions

    brew install php56-http
    brew install php56-mcrypt
    brew install php56-memcache
    brew install php56-memcached
    brew install php56-opcache
    brew install php56-propro
    brew install php56-raphf
    brew install php56-tidy
    brew install php56-xdebug
    # ...

add launch agent for memcached

    ln -sfv /usr/local/opt/memcached/*.plist ~/Library/LaunchAgents

or get others

    brew search php56

What about APC? See [stackoverflow](http://stackoverflow.com/questions/9611676/is-apc-compatible-with-php-5-4-or-php-5-5) - APC have some problems but you can install emulated APC

    brew install php56-apcu # APC

### Replace OS X PHP

change `~/.bash_profile`, add follow line:

    export PATH="/usr/local/bin:/usr/local/sbin:$PATH"

Restart Terminal and check if working `php -v` or `php-fpm -v`

### Configuration and php.ini

You can found basic php-fpm config file here `vi /usr/local/etc/php/5.6/php-fpm.conf`. Check especially `listen = 127.0.0.1:9000` everything else can be leave as is.

PHP config files can be found here `/usr/local/etc/php/5.6/conf.d/`. You can change `php.ini` but its more more easly keept change is spearate file:

    vi /usr/local/etc/php/5.6/conf.d/zzzz_custom.ini

See example configuration:

    short_open_tag = On
    display_errors = On
    display_startup_errors = On
    upload_max_filesize = 1024M
    post_max_size = 1024M
    date.timezone = "Europe/Warsaw"
    error_reporting = E_ALL
    memory_limit = 256M

    log_errors=On
    error_log=/tmp/php-error.log

    mysql.default_socket=/tmp/mysql.sock
    pdo_mysql.default_socket=/tmp/mysql.sock

    [opcache]
    opcache.revalidate_freq=1

    [xdebug]
    xdebug.remote_enable=1
    xdebug.remote_connect_back=On
    xdebug.remote_autostart=1
    xdebug.idekey=PHPSTORM
    xhprof.output_dir="/var/tmp/xhprof"

    xdebug.profiler_enable = 0;
    xdebug.profiler_output_name=cachegrind.out.%H.%t
    xdebug.profiler_enable_trigger = 1;
    xdebug.profiler_output_dir = /Users/pawelkolanowski/.Trash


## Symfony2 + NGINX + PHP-FPM

### Setup virtuals

Create first dev configuration:

    vi /usr/local/etc/nginx/sites-available/symfony.dev

Here is example configuration:

    server {
        listen 80;
        server_name symfony.dev;
        root /Users/pawelkolanowski/www/symfony/web;

        rewrite ^/app\.php/?(.*)$ /$1 permanent;

        try_files $uri @rewriteapp;

        location @rewriteapp {
            rewrite ^(.*)$ /app.php/$1 last;
        }

        location ~ /\. {
            deny all;
        }

        location ~ ^/(app|app_dev)\.php(/|$) {
            fastcgi_split_path_info ^(.+\.php)(/.*)$;
            include fastcgi_params;
            fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_index app.php;
            send_timeout 1800;
            fastcgi_read_timeout 1800;
            fastcgi_pass 127.0.0.1:9000;
        }

        location /(bundles|media) {
            access_log off;
            expires 30d;
                try_files $uri @rewriteapp;
        }
    }

Create symlink to sites-enabled:

    sudo ln -s /usr/local/etc/nginx/sites-available/symfony.dev /usr/local/etc/nginx/sites-enabled/symfony.dev

Update your `vi /etc/hosts` file with follow line:

    127.0.0.1   symfony.dev

Restart nginx `sudo nginx -s reload` and check if working `http://symfony.dev`


## Thanks

- [Roman Ožana](https://github.com/OzzyCzech) for a great work! 80% of the text which you read here comes from this [tutorial](https://github.com/OzzyCzech/dotfiles/blob/master/how-to-install-mac.md)
- [Derick Bailey](http://derickbailey.com/) for [finding answer](http://derickbailey.com/2014/12/27/how-to-start-nginx-on-port-80-at-mac-osx-boot-up-log-in/) how to autorun nginx on startup at port *:80
- [Leon Radley](https://github.com/leon) for [simple and clear](https://gist.github.com/leon/3019026) symfony nginx configutation.
