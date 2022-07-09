---
layout: post
title: "Linux - Install WordPress"
date: 2016-05-07 13:37:00 +00
categories: linux
tags:  hosting
---

## Installation

Lets start by creating MySQL user and a database for WordPress:

```bash
mysql -u root -p
```

```sql
mysql> CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;

mysql> GRANT ALL ON wordpress.* TO 'wordpressuser'@'localhost' IDENTIFIED BY 'password';

mysql> FLUSH PRIVILEGES;

mysql> EXIT;
```

>Warning: Every MySQL statement must end in a semi-colon (`;`). Check to make sure this is present if you are running into any issues.
{: .prompt-warning}

Install additional PHP Plugins and restart Apache service:

```bash
sudo apt update
sudo apt install php-curl php-gd php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip
sudo systemctl restart apache2
```

## Configuration

Now let’s enable `.htaccess` overrides:

```bash
sudo vim /etc/apache2/sites-available/harryvasanth.com.conf
```

```conf
<VirtualHost *:80>
    #
    # DocumentRoot: The directory out of which you will serve your
    # documents. By default, all requests are taken from this directory, but
    # symbolic links and aliases may be used to point to other locations.
    #
    DocumentRoot "/var/www/harryvasanth.com/htdocs"
    ServerName  harryvasanth.com
    ServerAlias  www.harryvasanth.com
    ServerAdmin hello@harryvasanth.com
    ErrorLog /var/log/apache2/harryvasanth.com-error.log
    CustomLog /var/log/apache2/harryvasanth.com-access.log combined
    # This should be changed to whatever you set DocumentRoot to.
    #
    <Directory "/var/www/harryvasanth.com/htdocs">
        #
        # Possible values for the Options directive are "None", "All",
        # or any combination of:
        #   Indexes Includes FollowSymLinks SymLinksifOwnerMatch ExecCGI MultiViews
        #
        # Note that "MultiViews" must be named *explicitly* --- "Options All"
        # doesn't give it to you.
        #
        # The Options directive is both complicated and important.  Please see
        # http://httpd.apache.org/docs-2.0/mod/core.html#options
        # for more information.
        #
        Options Indexes FollowSymLinks MultiViews

        #
        # AllowOverride controls what directives may be placed in .htaccess files.
        # It can be "All", "None", or any combination of the keywords:
        #   Options FileInfo AuthConfig Limit
        #
        AllowOverride All

        #
        # Controls who can get stuff from this server.
        #
        Order allow,deny
        Allow from all
    </Directory>
</VirtualHost>
```

Download WordPress and extract it to `/tmp` and do basic preparations:

```bash
cd /tmp
```

```bash
# Download and extract the latest WordPress version
curl -O https://wordpress.org/latest.tar.gz
tar xzvf latest.tar.gz

# Create .htaccess
touch /tmp/wordpress/.htaccess

# Copy sample config to get started
cp /tmp/wordpress/wp-config-sample.php /tmp/wordpress/wp-config.php

# Create Upgrade folder for future updates
mkdir /tmp/wordpress/wp-content/upgrade

# Move the directory to our hosting directory
sudo cp -a /tmp/wordpress/. /var/www/wordpress
Fix file permission for the WordPress directory:

sudo chown -R www-data:www-data /var/www/wordpress
sudo find /var/www/wordpress/ -type d -exec chmod 750 {} \;
sudo find /var/www/wordpress/ -type f -exec chmod 640 {} \;
```

Now let’s start changing change the configurations by generating salted keys:

```bash
curl -s https://api.wordpress.org/secret-key/1.1/salt/
```

Copy the output. It should look something like this:

```php
define('AUTH_KEY',         '6y9T>=`^-3NaDywbj,~0,8!/$1m.uvadJ++N;-yoL^8eaSe%V(]VX<2%wxbxgLw&');
define('SECURE_AUTH_KEY',  'lz>$g?@!+<gmQJukfRtti#Dkk~;:+?wQY,7K!D30W&=U;]-3}2CS>D8;+~%Kv}v9');
define('LOGGED_IN_KEY',    'p}> nuTwbA~%ueeM#LuU7.WzV1[;3VA bo,Pm6s_f$HY^uzqJ6.$Y|bVbZV1jMm3');
define('NONCE_KEY',        '4oS4i:KwkOz:/$Z)&e]y@7m5B8FR<+zs+do=BmG~d<<EV@NQ:9h-GNnk*k/z77@K');
define('AUTH_SALT',        '68|hC4*`4x-ehSRF][-(fR5S8-R|sQt]O}Lt(6&DJCU}v5#|]uh:)$iM+)h|?fR~');
define('SECURE_AUTH_SALT', 'rW)-xV{c_BeO]x-G!,e$TBE(Fz|)9P0MhV+l/N.&80@O2GC:Fkd(?7Ft;M<;nMH4');
define('LOGGED_IN_SALT',   'XU.w FA[A{n#-$T[$+ OTm3gOo+AG`Z`HH<l&|QjTmk)@4v|-I;|SEccx9>ay$<x');
define('NONCE_SALT',       'AV_?7B/oX;mq-/,.h_eX-Dl-xm_zbG8.B]ySK~VUyVa>Rbx dNqd!&m-Fez}HiPq');
```

Now copy these values and replace it in the `wp-config.php` file:

```bash
sudo vim /var/www/wordpress/wp-config.php
```

```php
define('AUTH_KEY',         'put your unique phrase here');
define('SECURE_AUTH_KEY',  'put your unique phrase here');
define('LOGGED_IN_KEY',    'put your unique phrase here');
define('NONCE_KEY',        'put your unique phrase here');
define('AUTH_SALT',        'put your unique phrase here');
define('SECURE_AUTH_SALT', 'put your unique phrase here');
define('LOGGED_IN_SALT',   'put your unique phrase here');
define('NONCE_SALT',       'put your unique phrase here');
```

Delete those lines and paste in the values you copied:

```php
define('AUTH_KEY',         '6y9T>=`^-3NaDywbj,~0,8!/$1m.uvadJ++N;-yoL^8eaSe%V(]VX<2%wxbxgLw&');
define('SECURE_AUTH_KEY',  'lz>$g?@!+<gmQJukfRtti#Dkk~;:+?wQY,7K!D30W&=U;]-3}2CS>D8;+~%Kv}v9');
define('LOGGED_IN_KEY',    'p}> nuTwbA~%ueeM#LuU7.WzV1[;3VA bo,Pm6s_f$HY^uzqJ6.$Y|bVbZV1jMm3');
define('NONCE_KEY',        '4oS4i:KwkOz:/$Z)&e]y@7m5B8FR<+zs+do=BmG~d<<EV@NQ:9h-GNnk*k/z77@K');
define('AUTH_SALT',        '68|hC4*`4x-ehSRF][-(fR5S8-R|sQt]O}Lt(6&DJCU}v5#|]uh:)$iM+)h|?fR~');
define('SECURE_AUTH_SALT', 'rW)-xV{c_BeO]x-G!,e$TBE(Fz|)9P0MhV+l/N.&80@O2GC:Fkd(?7Ft;M<;nMH4');
define('LOGGED_IN_SALT',   'XU.w FA[A{n#-$T[$+ OTm3gOo+AG`Z`HH<l&|QjTmk)@4v|-I;|SEccx9>ay$<x');
define('NONCE_SALT',       'AV_?7B/oX;mq-/,.h_eX-Dl-xm_zbG8.B]ySK~VUyVa>Rbx dNqd!&m-Fez}HiPq');
```

Now change the MySQL database details:

```php
. . .

define('DB_NAME', 'wordpress');

/** MySQL database username */
define('DB_USER', 'wordpressuser');

/** MySQL database password */
define('DB_PASSWORD', 'password');

. . .
```

Then add the following line at the end to prevent WordPress from promting for FTP Server.

```php
. . .

define('FS_METHOD', 'direct');
```

## Verification

Finally visit your using your web browser to finish setting up WordPress:

```console
http://harryvasanth.com
```
