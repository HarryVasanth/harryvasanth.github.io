---
layout: post
title: "Linux - Install Certbot"
date: 2016-04-30 09:39:21 +00
categories: linux
tags:  hosting
---

## Installation

Install Certbot’s Apache package with apt:

```bash
sudo apt install certbot
```

## Configuration

Now use Certbot to obtain the SSL certificate:

```bash
sudo certbot --apache -d your_domain -d www.your_domain
```

This runs `certbot` with the `--apache` plugin, using `-d` to specify the domain names you’d like the certificate to be valid for.

If that’s successful, `certbot` will ask how you’d like to configure your HTTPS settings:

```console
Please choose whether or not to redirect HTTP traffic to HTTPS, removing HTTP access.
-------------------------------------------------------------------------------
1: No redirect - Make no further changes to the webserver configuration.
2: Redirect - Make all requests redirect to secure HTTPS access. Choose this for
new sites, or if you're confident your site works on HTTPS. You can undo this
change by editing your web server's configuration.
-------------------------------------------------------------------------------
Select the appropriate number [1-2] then [enter] (press 'c' to cancel):
```

Select your choice then hit `ENTER`. The configuration will be updated, and Apache will reload to pick up the new settings. `certbot` will wrap up with a message telling you the process was successful and where your certificates are stored:

```console
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/your_domain/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/your_domain/privkey.pem
   Your cert will expire on 2016-07-30. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot again
   with the "certonly" option. To non-interactively renew *all* of
   your certificates, run "certbot renew"
 - Your account credentials have been saved in your Certbot
   configuration directory at /etc/letsencrypt. You should make a
   secure backup of this folder now. This configuration directory will
   also contain certificates and private keys obtained by Certbot so
   making regular backups of this folder is ideal.
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

## Verification

Finally lets test the certificate renewal process by doing a dry run with certbot:

```bash
sudo certbot renew --dry-run
```

If you see no errors, you’re all set.
