---
layout: post
title: "Linux - Install Caddy Web Server"
date: 2021-02-14 01:42:22 +01
categories: linux
tags: caddy hosting
---

## Introduction

Caddy is a modern web server that is easy to configure and comes with built-in HTTPS support. In this guide, we'll walk through the process of installing and configuring Caddy on a Linux server.

### Installation

Install Caddy using the official script:

```bash
curl -fsSL https://getcaddy.com | bash
```

This script automatically detects your system and installs the latest version of Caddy.

### Basic Configuration

Create a new directory to store your Caddy configuration files:

```bash
sudo mkdir /etc/caddy
```

Now, let's create a basic Caddyfile to define our web server settings. Use your preferred text editor to create the file:

```bash
sudo nano /etc/caddy/Caddyfile
```

Add the following basic configuration, replacing "your_domain" with your actual domain name:

```plaintext
your_domain {
    root * /var/www/your_domain
    encode gzip
    file_server
}
```

Save and exit the text editor.

### Serving Static Content

Create the web root directory and a simple HTML file:

```bash
sudo mkdir -p /var/www/your_domain
sudo nano /var/www/your_domain/index.html
```

Add some content to the HTML file:

```html
<html>
    <head>
        <title>Welcome to Your_domain!</title>
    </head>
    <body>
        <h1>Success! The Caddy server is up and running!</h1>
    </body>
</html>
```

Save and exit the text editor.

### Enable and Start Caddy

Now, enable and start the Caddy service:

```bash
sudo systemctl enable caddy
sudo systemctl start caddy
```

### Automatic HTTPS

One of Caddy's powerful features is its automatic HTTPS support. The first time you access your site, Caddy will automatically obtain and configure SSL certificates for you.

### Advanced Configuration

For more advanced configurations, such as setting up reverse proxies, rewriting URLs, or adding additional sites, refer to the [Caddy documentation](https://caddyserver.com/docs/).

## Conclusion

Congratulations! You've successfully installed and configured the Caddy web server on your Linux machine. Caddy's simplicity and automatic HTTPS make it an excellent choice for serving your web applications.

Remember to check for updates regularly and review the Caddy documentation for more advanced configurations.
