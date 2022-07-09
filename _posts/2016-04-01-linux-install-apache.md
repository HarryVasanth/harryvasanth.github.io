---
layout: post
title: "Linux - Install Apache"
date: 2016-04-01 02:32:03 +00
categories: linux
tags:  hosting
---

## Installation

Install Apache using Ubuntu’s package manager using `apt`:

```bash
sudo apt update
sudo apt install apache2
```

Check and adjust the Firewall rules to allow all web traffic to Apache using `ufw`:

```bash
sudo ufw allow in "Apache Full"
```

When you check your server’s IP on the browser, you should see a default landing page of Apache.

## Configuration

Now lets start setting virtual hosts:

```bash
sudo mkdir /var/www/your_domain
sudo chown -R www-data:www-data /var/www/your_domain
sudo chmod -R 755 /var/www/your_domain
```

Let’s create simple HTML page:

```bash
sudo vim /var/www/your_domain/index.html
```

```html
<html>
    <head>
        <title>Welcome to Your_domain!</title>
    </head>
    <body>
        <h1>Success!  The your_domain server block is working!</h1>
    </body>
</html>
```
{: file=' /var/www/your_domain/index.html'}

Now let’s create a new virtual host file under Apache’s available sites:

```bash
sudo vim /etc/apache2/sites-available/your_domain.conf
```

```conf
<VirtualHost *:80>
    ServerAdmin     webmaster@localhost
    ServerName      your_domain
    ServerAlias     www.your_domain
    DocumentRoot    /var/www/your_domain
    ErrorLog        ${APACHE_LOG_DIR}/error.log
    CustomLog       ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```
{: file='/etc/apache2/sites-available/your_domain.conf'}

Let’s enable out host file and disable the default Apache landing page:

```bash
sudo a2ensite your_domain.conf
sudo a2dissite 000-default.conf
```

## Verification

Now we can check if the configurations are working properly and restart Apache service for changes to take effect:

```bash
sudo apache2ctl configtest
    Syntax OK

sudo systemctl restart apache2
```

Finally, when you check your server’s IP on the browser, you should see `“The your_domain server block is working!”`.
