---
layout: post
title: "Docker - Host a Demo CAS Server"
date: 2022-07-23 09:27:59 +01
categories: docker
tags:  cas
---

## Intro

### Obtain keystore file

In order to host a Demo CAS server, first of all we will need to generate a trusted authority signed certificate which can be obtained by following my pervious post on [Linux - Let's Encrypt for Java Apps](https://harryvasanth.github.io/posts/linux-lets-encrypt-java-apps/).

### Install docker

Furthermore, we will be running the Demo CAS Server in docker. Hence, we will be needing a working docker installation. You can follow another one of my posts on [Docker - Install and Configure](https://harryvasanth.github.io/posts/docker-install-configure/).

### Obtain the CAS docker image

In this case, we are configuring our Demo CAS server to run on v6.4.0. In order to obtain this version you can use the following command:

```bash
docker pull apereo/cas:6.4.0
```

If you wish to run the latest version of the CAS server you can use the following command:

```bash
docker pull apereo/cas:latest
```

## Make the shell script to run the docker container

```bash
#!/usr/bin/env bash

IFS='
'
cd -P "$(dirname "$0")"

properties='{
      "management": {
        "endpoints": {
            "enabled-by-default": true,
            "web": {
                "exposure": {
                    "include": "*"
                }
            }
        }
    },
    "cas": {
        "authn": {
            "accept": {
                "users": "casuser::Mellon,casadmin::Mellon,harry::Vasanth"
            },
            "attribute-repository": {
                "core": {
                        "default-attributes-to-release": [ "first-name", "last-name", "email" ]
                },
                "stub": {
                    "attributes": {
                        "cn": "CAS",
                        "display-name": "Harry Vasanth",
                        "uid": "harry",
                        "first-name": "Harry",
                        "last-name": "Vasanth",
                        "email": "harry@harryvasanth.com"
                    },
                    "id": "Static",
                    "order": "0"
                }
            }
        },
        "events": {
            "core": {
                "track-configuration-modifications": false
            }
        },
        "monitor": {
            "endpoints": {
                "endpoint": {
                    "defaults": {
                        "access": "ANONYMOUS"
                    }
                }
            }
        },
        "server": {
            "name": "https://cas.harryvasanth.com:8444",
            "prefix": "https://cas.harryvasanth.com:8444/cas",
            "tomcat": {
                "http": {
                    "enabled": false
                }
            }
        },
        "service-registry": {
            "core": {
                "init-from-json": true
            }
        }
    }
}'
properties=$(echo "$properties" | tr -d '[:space:]')
echo -e "*******************************\nCAS Server properties\n*******************************"
echo "${properties}" | jq

docker stop demo-cas-server || true && docker rm demo-cas-server || true

echo -e "Mapping CAS keystore in Docker container to '$(pwd)/cas.harryvasanth.com.keystore'"

docker run --rm -d \
        --mount type=bind,source="$(pwd)/cas.harryvasanth.com.keystore",target=/etc/cas/thekeystore
 \
  -e SPRING_APPLICATION_JSON="${properties}" \
  -p 8444:8443 --name demo-cas-server apereo/cas:6.4.0

docker logs -f demo-cas-server &
echo -e "Waiting for CAS Server to Initialize..."
until curl -k -L --output /dev/null --silent --fail https://cas.harryvasanth.com:8444/cas/login; do
  echo -n .
  sleep 1
done
echo -e "\nCAS Server is running on port 8444"
echo -e "\n\n CAS Server is Ready!"
```
>Change `cas.harryvasanth.com` with the FQDN of the desired CAS server.
{: .prompt-info }
