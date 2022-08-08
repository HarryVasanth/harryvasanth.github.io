---
layout: post
title: "Linux - Let's Encrypt for Java Apps"
date: 2022-07-22 19:12:22 +01
categories: linux
tags:  hosting
---
## Intro

Install certbot as mentioned in my previous post on [Linux - Install Certbot](https://harryvasanth.github.io/posts/linux-install-certbot/).

By default, `certbot` retrieves a certificate and installs it immediately on the web server by adding an extra parameter, eg. `--apache` for Apache HTTP Server. For the case of [Docker - Demo CAS Server,](https://harryvasanth.github.io/posts/docker-demo-cas-server/) it is just enough to retrieve only a certificate. This is done by adding the `certonly` parameter to the command as follows:

```bash
certbot certonly
```
>You need to make sure that the `TCP` port `80` of the server is accessible from Let's Encrypt servers and no other application is utilizing it.
{: .prompt-info }

This will spin up a temporary server on `TCP` port `80` to validate the authority of the domain to issue the required certificates. The installed Let’s Encrypt certificates in the `/etc/letsencrypt/live` directory on the file system.

## Create Keystore file

To use a certificate in a Java application, the preferred way is to add it to a separate `.keystore` file. The Java Runtime Environment (JRE) ships with a tool called `keytool` to create certificates and manipulate key stores. Adding certificates to a keystore can be done by using OpenSSL and the `keytool`.

If the system doesn't have JRE installed, you can install it by running the following command:

```bash
sudo apt install default-jre jq
```

### Archive certificates to PKCS #12 file

Importing multiple public and private `.pem` certificates directly in a keystore is not supported by the `keytool`. Hence, it is necessary to add all `.pem` files to a PKCS 12 archive. This can be done by the OpenSSL tool with the following command:

```bash
openssl pkcs12 -export \
	 -in /etc/letsencrypt/live/cas.harryvasanth.com/cert.pem \
	 -inkey /etc/letsencrypt/live/cas.harryvasanth.com/privkey.pem \
	 -out /tmp/cas.harryvasanth.com.p12 \
	 -name cas.harryvasanth.com \
	 -CAfile /etc/letsencrypt/live/cas.harryvasanth.com/fullchain.pem \
	 -caname "Let's Encrypt Authority X3" \
	 -password pass:changeit
```

>Change `cas.harryvasanth.com` with the custom FQDN of the target CAS server (or any Java application server).
{: .prompt-info }

### Import PKCS #12 archive to the Keystore file

The next step is to import the certificates into a `.keystore` file.

```bash
keytool -importkeystore \
	-deststorepass changeit \
	-destkeypass changeit \
	-deststoretype pkcs12 \
	-srckeystore /tmp/cas.harryvasanth.com.p12 \
	-srcstoretype PKCS12 \
	-srcstorepass changeit \
	-destkeystore /tmp/cas.harryvasanth.com.keystore \
	-alias cas.harryvasanth.com
```

The keystore at location `/tmp/cas.harryvasanth.com.keystore` can now be used by the CAS server (or any Java application server).

## Automate Keystore creation

### Make the shell script to create keystore

Create a shell script `/home/<user.name>/renew-cert-keystore.sh`
Here is an example:

```shell
touch /home/harry.vasanth/renew-cert-keystore.sh && \
chmod +x /home/harry.vasanth/renew-cert-keystore.sh
```

Add the following content to the script:

```bash
#!/usr/bin/env bash

IFS='
'
cd -P "$(dirname "$0")"

# Create keystore
echo "Refreshing '$(pwd)/cas.harryvasanth.com.keystore'"

openssl pkcs12 -export \
	 -in /etc/letsencrypt/live/cas.harryvasanth.com/cert.pem \
	 -inkey /etc/letsencrypt/live/cas.harryvasanth.com/privkey.pem \
	 -out /tmp/cas.harryvasanth.com.p12 \
	 -name cas.harryvasanth.com \
	 -CAfile /etc/letsencrypt/live/cas.harryvasanth.com/fullchain.pem \
	 -caname "Let's Encrypt Authority X3" \
	 -password pass:changeit

# User Keytool to import P12 to Keystore format
keytool -importkeystore \
	-deststorepass changeit \
	-destkeypass changeit \
	-deststoretype pkcs12 \
	-srckeystore /tmp/cas.harryvasanth.com.p12 \
	-srcstoretype PKCS12 \
	-srcstorepass changeit \
	-destkeystore /tmp/cas.harryvasanth.com.keystore \
	-alias cas.harryvasanth.com

# Move certificates to the current directory
echo "Copy '/tmp/cas.harryvasanth.com.keystore' to '$(pwd)/cas.harryvasanth.com.keystore'"
cp /tmp/cas.harryvasanth.com.keystore $(pwd)/cas.harryvasanth.com.keystore
```

### Schedule the script to run periodically

The following line in `crontab` makes sure our script is executed every hour.

```bash
0 * * * * yes | bash /home/<user.name>/renew-cert-keystore.sh >> /var/log/renew-cert-keystore.log
```
