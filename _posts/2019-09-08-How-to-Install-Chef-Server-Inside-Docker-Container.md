---
layout: default
title: "How to install Chef-Server Inside Docker Container"
date: 2019-09-08
---

* Create volumes for chef logs and data

```
docker volume create chef-log
docker volume create chef-data
```

* Run centos/systemd container with privileged permissions and ipv6 disabled

```
docker run --sysctl net.ipv6.conf.all.disable_ipv6=1 --privileged --name chef-server-core -d -v chef-data:/var/opt/opscode -v chef-log:/var/log/opscode -p 80:80 -p 443:443 centos/systemd
```

* Connect to your new container

```
docker exec -it chef-server-core bash
```

* Download and install chef

```
curl https://packages.chef.io/files/stable/chef-server/13.0.17/el/7/chef-server-core-13.0.17-1.el7.x86_64.rpm -o chef-server-core.rpm
rpm -Uvh chef-server-core.rpm
yum install rsync crontabs which net-tools less -y
systemctl enable crond
systemctl start crond
chef-server-ctl reconfigure
```

* **Note:** if you want to restore from a backup run the following command:

```
chef-server-ctl restore -t 60000 /tmp/chef-backup-date.tgz
```

* **Note:** When I install it I got the following error after running chef-server-ctl reconfigure:

```
================================================================================
Recipe Compile Error in /var/opt/opscode/local-mode-cache/cookbooks/private-chef/attributes/default.rb
================================================================================

NoMethodError
-------------
undefined method `[]' for nil:NilClass

Cookbook Trace:
---------------
  /var/opt/opscode/local-mode-cache/cookbooks/private-chef/attributes/default.rb:616:in `from_file'

Relevant File Content:
----------------------
/var/opt/opscode/local-mode-cache/cookbooks/private-chef/attributes/default.rb:

609:  default['private_chef']['postgresql']['db_superuser'] = 'opscode-pgsql'
610:  default['private_chef']['postgresql']['shell'] = "/bin/sh"
611:  default['private_chef']['postgresql']['home'] = "/var/opt/opscode/postgresql"
612:  default['private_chef']['postgresql']['user_path'] = "/opt/opscode/embedded/bin:/opt/opscode/bin:$PATH"
613:  default['private_chef']['postgresql']['vip'] = "127.0.0.1"
614:  default['private_chef']['postgresql']['port'] = 5432
615:  # We want to listen on all the loopback addresses, because we can't control which one localhost resolves to.
616>> default['private_chef']['postgresql']['listen_address'] = node['network']['interfaces']['lo']['addresses'].keys.join(',')
617:  default['private_chef']['postgresql']['max_connections'] = 350
618:  default['private_chef']['postgresql']['keepalives_idle'] = 60
619:  default['private_chef']['postgresql']['keepalives_interval'] = 15
620:  default['private_chef']['postgresql']['keepalives_count'] = 2
621:  default['private_chef']['postgresql']['md5_auth_cidr_addresses'] = [ '127.0.0.1/32', '::1/128' ]
622:  default['private_chef']['postgresql']['wal_level'] = "minimal"
623:  default['private_chef']['postgresql']['archive_mode'] = "off" # "cannot be enabled when wal_level is set to minimal"
624:  default['private_chef']['postgresql']['archive_command'] = ""
625:  default['private_chef']['postgresql']['archive_timeout'] = 0 # 0 is disabled.

System Info:
------------
chef_version=15.0.300
platform=centos
platform_version=7.6.1810
ruby=ruby 2.5.5p157 (2019-03-15 revision 67260) [x86_64-linux]
program_name=/opt/opscode/embedded/bin/chef-client
executable=/opt/opscode/embedded/bin/chef-client


Running handlers:
Running handlers complete
Chef Infra Client failed. 0 resources updated in 03 seconds
[2019-08-08T05:39:53+00:00] FATAL: Stacktrace dumped to /var/opt/opscode/local-mode-cache/chef-stacktrace.out
[2019-08-08T05:39:53+00:00] FATAL: Please provide the contents of the stacktrace.out file if you file a bug report
[2019-08-08T05:39:53+00:00] FATAL: NoMethodError: undefined method `[]' for nil:NilClass
```

* If you get the same error you need to configure default[‘private_chef’][‘postgresql’][‘listen_address’]

```
vi /opt/opscode/embedded/cookbooks/private-chef/attributes/default.rb
```

```
...
default['private_chef']['postgresql']['listen_address'] = "localhost"
...
```

```
chef-server-ctl reconfigure
```

* **Note:** In chef-server-core 12.15.7 I got the following error:

```
-----------------------------------------------------------------------
Your system has IPv6 enabled but its loopback interface has no IPv6
address.

You must either pass `ipv6.disable=1` to your kernel command line,
to completely disable IPv6, or ensure the loopback interface has an
`::1` address by running

    sysctl net.ipv6.conf.lo.disable_ipv6=0
-----------------------------------------------------------------------
```

* To fix this you need to run the following

```
echo 0 > /proc/sys/net/ipv6/conf/all/disable_ipv6
chef-server-ctl reconfigure
```

* Install chef-manage

```
chef-server-ctl install chef-manage
chef-server-ctl reconfigure
chef-manage-ctl reconfigure --accept-license
```

* Creat chef admin

```
chef-server-ctl user-create USER_NAME FIRST_NAME LAST_NAME EMAIL 'PASSWORD' --filename FILE_NAME
```

* Create organization

```
chef-server-ctl org-create short_name 'full_organization_name' --association_user user_name --filename ORGANIZATION-validator.pem
```

* you can browse to your server ip address to see chef-manage. https://server_ip
