---
layout: default
title: "Run Chef Server Behind Nginx"
date: 2019-09-18
---

* create or edit chef-server.rb file

```
vi /etc/opscode/chef-server.rb
```

```
server_name = "chef.example.com"
api_fqdn server_name
bookshelf['vip'] = server_name
nginx['url'] = "https://#{server_name}"
nginx['server_name'] = server_name
nginx['client_max_body_size'] = "1000m"
nginx['enable_non_ssl']=true
```

* run chef server reconfigure

```
chef-server-ctl reconfigure
```

* Configure nginx like any virtual server with regular http traffic
