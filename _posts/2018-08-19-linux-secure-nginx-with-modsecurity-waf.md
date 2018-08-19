---
layout: post
title: "Linux - Secure NGINX with ModSecurity Web Application Firewall"
date: 2018-08-19 11:55:39 +01
categories: linux hosting
tags: waf hosting nginx
---

## Intro

A Web Application Firewall (WAF) like ModSecurity, combined with Nginx, provides a robust defense against various web-based attacks. This guide provides an in-depth look at setting up and configuring ModSecurity with Nginx to protect your web server.

---

## Step 1: Install Nginx

If you don't already have Nginx installed, begin by installing it:

For Debian/Ubuntu:

```sh
sudo apt update
sudo apt install nginx
```

For CentOS/RHEL:

```sh
sudo yum install nginx
sudo systemctl start nginx
sudo systemctl enable nginx
```

Verify Nginx is running:

```sh
sudo systemctl status nginx
```

---

## Step 2: Install ModSecurity and its Dependencies

### Install the ModSecurity package

For Debian/Ubuntu:

```sh
sudo apt update
sudo apt install libapache2-mod-security2
```

For CentOS/RHEL, you might need to enable the EPEL repository first:

```sh
sudo yum install epel-release
sudo yum install mod_security
```

### Install the Core Rule Set (CRS)

The Core Rule Set provides a baseline set of rules to protect against common web application attacks.

```sh
sudo apt install modsecurity-crs # Debian/Ubuntu

#For CentOS/RHEL
sudo yum install modsecurity-crs
```

---

## Step 3: Configure ModSecurity with Nginx

### Create a ModSecurity Configuration File

Create a basic ModSecurity configuration file for Nginx:

```sh
sudo nano /etc/nginx/conf.d/modsecurity.conf
```

Add the following configuration:

```conf
# modsecurity.conf
load_module modules/ngx_http_modsecurity_module.so;

SecRuleEngine On

SecAuditEngine RelevantOnly
SecAuditLogRelevantStatus "^(?:5|4)\d\d"
SecAuditLogParts ABIJDEFGHZ
SecDataDir /var/cache/modsecurity
```

### Link ModSecurity Configuration to Nginx

Edit your Nginx site configuration file (e.g., `/etc/nginx/sites-available/default`):

```sh
sudo nano /etc/nginx/sites-available/default
```

Add the following inside the `server` block:

```conf
server {
    # Existing configuration...

    include /etc/nginx/conf.d/modsecurity.conf;

    location / {
        # Existing configuration...
        modsecurity_rules_file /etc/modsecurity/modsecurity.conf;
    }
}
```

### Configure CRS

Link the CRS rules in `/etc/modsecurity/modsecurity.conf`:

```sh
sudo nano /etc/modsecurity/modsecurity.conf
```

Add or modify:

```conf
Include /etc/modsecurity/crs-setup.conf
Include /etc/modsecurity/rules/*.conf
```

---

## Step 4: Testing the Setup

### Test Nginx Configuration

Ensure there are no syntax errors:

```sh
sudo nginx -t
```

### Restart Nginx

```sh
sudo systemctl restart nginx
```

### Trigger a ModSecurity Rule

Try triggering a rule, for example, by sending a request with a common SQL injection pattern:

```sh
curl "http://your_server_ip/?param=<script>alert('XSS')</script>"
```

Check the ModSecurity audit logs:

```sh
sudo tail /var/log/modsec_audit.log
```

---

## Step 5: Advanced Configuration and Tuning

### Understanding ModSecurity Rules

ModSecurity rules are based on a powerful rule language. Here's an example:

```conf
SecRule REQUEST_URI "@contains 'sql'" \
    "id:12345, \
    phase:2, \
    t:lowercase, \
    deny, \
    status:403, \
    msg:'Potential SQL Injection Attempt'"
```

- `SecRule`: Defines the rule.
- `REQUEST_URI`: Checks the requested URL.
- `@contains 'sql'`: Checks if the URL contains the string 'sql'.
- `id`: Unique ID for the rule.
- `phase`: The phase in which the rule is executed.
- `t:lowercase`: Transforms the input to lowercase.
- `deny`: Denies the request.
- `status`: Sets the HTTP status code.
- `msg`: Custom message for the log.

### Tuning CRS

The CRS comes with a lot of rules enabled by default. You might need to disable some rules to prevent false positives.

1.  **Examine the Audit Logs**:

    ```sh
    sudo tail /var/log/modsec_audit.log
    ```

2.  **Identify False Positives**: Look for requests that are wrongly flagged.
3.  **Disable Rules**: Disable rules by adding the rule ID to an exclusion file (e.g., `/etc/modsecurity/crs-setup.conf`):

```conf
SecRuleRemoveById 941100 942100 942440
```

### Using the Paranoia Level

The CRS has different paranoia levels that you can configure:

```conf
SecAction "id:900000, \
    phase:1, \
    nolog, \
    pass, \
    t:none, \
    setvar:'tx.paranoia_level=1'"
```

- Level 1: Recommended for most installations.
- Level 2: For stricter environments.
- Level 3 and 4: For highly sensitive environments.

---

## Step 6: Logging and Monitoring

### Centralized Logging

Set up centralized logging using tools like `rsyslog` or `Fluentd` to aggregate ModSecurity logs for analysis.

### Real-time Monitoring

Use tools like `GoAccess` or `Elasticsearch/Kibana` to monitor ModSecurity logs in real time.

---

## Step 7: Best Practices

- **Keep Rules Updated**: Regularly update the CRS to protect against new threats.
- **Monitor Logs**: Regularly review ModSecurity logs to identify and address potential issues.
- **Test Configuration**: Before deploying changes to production, test them thoroughly in a staging environment.
- **Tune Rules**: Adjust ModSecurity rules to minimize false positives and optimize performance.

---

## Conclusion

By following this guide, you've implemented a robust Web Application Firewall using ModSecurity and Nginx. Regularly updating and tuning your configuration will help protect your web applications from a wide range of attacks.
