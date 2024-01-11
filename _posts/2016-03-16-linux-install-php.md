---
layout: post
title: "Linux - Install PHP"
date: 2016-03-16 11:09:12 +01
categories: linux
tags:  hosting
---

## Installation

Use `apt` to install PHP. In addition, include some helper packages this time so that PHP code can run under the Apache server and talk to your MySQL database:

```bash
sudo apt install php libapache2-mod-php php-mysql
```

Install PHP Command Line Interface:

```bash
sudo apt install php-cli
```

## Configuration

Open the `dir.conf` file to prioritize PHP extensions over standard ones:

```bash
sudo vim /etc/apache2/mods-available/dir.conf
```

It should look like this:

```conf
<IfModule mod_dir.c>
    DirectoryIndex index.php index.html index.cgi index.pl index.xhtml index.htm
</IfModule>
```

Lets restart Apache to apply the changes and also check the status of Apache service:

```bash
sudo systemctl restart apache2 && sudo systemctl status apache2

● apache2.service - LSB: Apache2 web server
   Loaded: loaded (/etc/init.d/apache2; bad; vendor preset: enabled)
  Drop-In: /lib/systemd/system/apache2.service.d
           └─apache2-systemd.conf
   Active: active (running) since Tue 2018-04-23 14:28:43 EDT; 45s ago
     Docs: man:systemd-sysv-generator(8)
  Process: 13581 ExecStop=/etc/init.d/apache2 stop (code=exited, status=0/SUCCESS)
  Process: 13605 ExecStart=/etc/init.d/apache2 start (code=exited, status=0/SUCCESS)
    Tasks: 6 (limit: 512)
   CGroup: /system.slice/apache2.service
           ├─13623 /usr/sbin/apache2 -k start
           ├─13626 /usr/sbin/apache2 -k start
           ├─13627 /usr/sbin/apache2 -k start
           ├─13628 /usr/sbin/apache2 -k start
           ├─13629 /usr/sbin/apache2 -k start
           └─13630 /usr/sbin/apache2 -k start
```

## Verification

Now let’s check if everything is working correctly by creating a PHP file under the web hosting directory we created earlier on the [Apache](../linux-install-apache) Installation section:

```bash
sudo vim /var/www/your_domain/phpinfo.php
```

Add the following lines to the phpinfo.php:

```php
<?php
phpinfo();
?>
```


Now we can check the ouput via the browser by going to your domain/ip:

`http://your_domain/phpinfo.php` or `http://your_ip/phpinfo.ph`
If it successful go ahead and remove the file to avoid giving away security information:

```bash
sudo rm /var/www/your_domain/phpinfo.php
```
