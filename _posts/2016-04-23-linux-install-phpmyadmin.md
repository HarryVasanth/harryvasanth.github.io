---
layout: post
title: "Linux - Install phpMyAdmin"
date: 2016-04-23 08:49:01 +00
categories: linux
tags:  hosting
---

## Installation

Use `apt` to install phpMyAdmin and additional complimentary packages:

```bash
sudo apt update
sudo apt install phpmyadmin php-mbstring php-gettext
```

This will ask you a few questions in order to configure your installation correctly.

>Warning: When the prompt appears, **`apache2`** is highlighted, but not selected. If you do not hit `SPACE` to select **Apache**, the installer will not move the necessary files during installation. Hit `SPACE`, `TAB`, and then `ENTER` to select **Apache**.
{: .prompt-danger}

- For the server selection, choose `apache2`
- Select Yes when asked whether to use `dbconfig-common` to set up the database
- You will then be asked to choose and confirm a MySQL application password for phpMyAdmin

## Configuration

The installation process adds the phpMyAdmin Apache configuration file into the `/etc/apache2/conf-enabled/` directory, where it is read automatically. The only thing you need to do is explicitly enable the `mbstring` PHP extension, and restart Apache service:

```bash
sudo phpenmod mbstring
sudo systemctl restart apache2
```

Create dedicated user for phpMyAdmin:

```bash
mysql -u root -p
```

```sql
mysql> CREATE USER 'phpmyadmin'@'localhost' IDENTIFIED BY 'password';
mysql> GRANT ALL PRIVILEGES ON *.* TO 'phpmyadmin'@'localhost' WITH GRANT OPTION;
mysql> exit
```

## Verification

You can now access the web interface by visiting your server’s domain name or public IP address followed by `/phpmyadmin`:

`http://your_domain_or_IP/phpmyadmin`
