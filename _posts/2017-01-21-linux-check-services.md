---
layout: post
title: "Linux - Check Services"
date: 2017-01-21 00:00:00 +01
categories: linux
tags: general
---

This script is to check if a service is running. If not, restarts the service and send an email to a specific recipient.

## Preparing the script

First of make an `executable`  bash script:

```bash
touch check_service.sh
chmod +x check_service.sh
```

Now edit the file with the following script:

```bash
 #!/bin/bash
 
 #---- edit the following ----#
 service="service_name"
 email="hello@harryvasanth.com"
 #------- stop editing -------#
 
 host=`hostname -f`
 if (( $(ps -ef | grep -v grep | grep $service | wc -l) > 0 ))
 then
 echo "$service is running"

 else
 /etc/init.d/$service restart

 if (( $(ps -ef | grep -v grep | grep $service | wc -l) > 0 ))
 then
 subject="$service at $host has been started"
 echo "$service at $host wasn't running and has been started" | mail -s "$subject" $email

 else
 subject="$service at $host is not running"
 echo "$service at $host is stopped and cannot be started!!!" | mail -s "$subject" $email

 fi
 fi

 ```

## Scheduling

The go ahead and create a `cron` job to run periodically.
>You can use the following site to quickly configure a cronjob: [cron.help](https://cron.help/).
{: .prompt-tip }

This is an example to run this script `every day` at `01:00`:

```bash
0 1 * * * /home/harry.vasanth/scripts/check_service.sh
```
