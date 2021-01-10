---
layout: post
title: "Cerbot DNS Authentication with GoDaddy"
date: 2021-01-10
---

## Tested On

OS: CentOS Linux release 7.4.1708 (Core)
Certbot: certbot 1.10.1

## Procedure

* download authenticator script

```
sudo wget -O /usr/local/authenticator.sh https://raw.githubusercontent.com/nachum234/scripts/master/godaddy/authenticator.sh
sudo chmod +x /usr/local/authenticator.sh
```

* change API_KEY and API_SECRET variables in authenticator script to your GoDaddy API.  
if you don't have API keys than go to https://developer.godaddy.com/ and generate API keys

```
sudo vi /usr/local/authenticator.sh
```

* run the following command to create or renew certificate (wildcard example)

```
certbot certonly --manual --preferred-challenges=dns --manual-auth-hook /usr/local/bin/authenticator.sh -d '*.example.com'
```
