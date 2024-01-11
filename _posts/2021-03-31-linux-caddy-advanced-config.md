---
layout: post
title: "Linux - Caddy: Advanced Configuration"
date: 2021-03-31 12:03:44 +01
categories: linux
tags: caddy hosting
---

## Introduction

[Caddy](https://caddyserver.com/) is a modern, flexible, and easy-to-use web server that comes with powerful features out of the box. In this guide, we'll explore advanced configurations for Caddy, focusing on rate limiting and other essential features.

### Install Caddy

Before diving into configurations, let's ensure you have Caddy installed. You can follow the official installation instructions on the [Caddy website](https://caddyserver.com/docs/download).

## Rate Limiting Configuration

### Basic Rate Limiting

Caddy makes rate limiting easy with its intuitive syntax. To add a basic rate limit to your site, edit your Caddyfile:

```plaintext
your_domain.com {
    route / {
        limit_req {
            rate 2r/s
        }
        respond "Rate Limited" 429
    }
    # ... other configurations
}
```

This configuration limits requests to 2 requests per second and returns a 429 status when the limit is exceeded.

### Customizing Rate Limiting

You can customize rate limiting based on specific paths or conditions. For example, limiting API requests:

```plaintext
your_domain.com {
    route /api* {
        limit_req {
            rate 10r/s
            burst 20
        }
        respond "Rate Limited" 429
    }
    # ... other configurations
}
```

This limits requests to paths starting with `/api` to 10 requests per second with a burst of 20.

## HTTPS and Automatic SSL

Caddy excels in providing easy SSL/TLS configuration:

```plaintext
your_domain.com {
    tls your_email@example.com
    # ... other configurations
}
```

This enables HTTPS with automatic SSL certificates from Let's Encrypt.

## Reverse Proxy

Caddy can act as a reverse proxy for your backend services:

```plaintext
your_domain.com {
    reverse_proxy localhost:8080
    # ... other configurations
}
```

Replace `localhost:8080` with your backend server's address.

## Gzip Compression

Improve website performance by enabling gzip compression:

```plaintext
your_domain.com {
    encode gzip
    # ... other configurations
}
```

Caddy will automatically compress content for browsers that support it.

## Error Handling

Customize error pages for a better user experience:

```plaintext
your_domain.com {
    handle_errors {
        404 /errors/not_found.html
        500 /errors/internal_server_error.html
    }
    # ... other configurations
}
```

Create the specified HTML error pages in a directory called `errors`.

## Conclusion

Caddy simplifies complex web server configurations with its user-friendly syntax and powerful features. Experiment with the provided examples to tailor Caddy to your specific needs, whether it's rate limiting, SSL, reverse proxy, or error handling.
