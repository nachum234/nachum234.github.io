---
layout: default
title: "Reset Pivotal User in CHEF"
date: 2019-12-29
---

These are low level commands in chef, please backup and make sure you can restore before running these commands

if you see the following error message in chef like I did:

```
chef-server-ctl user-list
 ERROR: Failed to authenticate to http://127.0.0.1:80 as pivotal with key /tmp/latovip20191029-14643-1na6z6n
 Response:  Invalid signature for user or client 'pivotal'
```

These commands may help you solve the problem:

* Create public key for pivotal user

```
openssl rsa -in /etc/opscode/pivotal.pem -pubout > /var/opt/opscode/postgresql/9.2/data/pivotal.pub
```

* Get pivotal authz id

```
echo "SELECT authz_id FROM auth_actor WHERE id = 1" | su -l opscode-pgsql -c 'psql bifrost -tA' | tr -d '\n' > /var/opt/opscode/postgresql/9.2/data/pivotal.authz_id
```

* Delete pivotal user from postgresdb

```
echo "DELETE FROM users WHERE authz_id = pg_read_file('pivotal.authz_id');" | su -l opscode-pgsql -c 'psql opscode_chef'
```

* Insert new pivotal user to postgresdb

```
echo "INSERT INTO users (id, authz_id, username, email, pubkey_version, public_key, serialized_object, last_updated_by, created_at, updated_at) VALUES (md5(random()::text), pg_read_file('pivotal.authz_id'), 'pivotal', 'kryptonite@opscode.com', 0, pg_read_file('pivotal.pub'), '{\"first_name\":\"Clark\",\"last_name\":\"Kent\",\"display_name\":\"Clark Kent\"}', pg_read_file('pivotal.authz_id'), LOCALTIMESTAMP, LOCALTIMESTAMP);" | su -l opscode-pgsql -c 'psql opscode_chef'
```

I took these commands from the following issue: https://github.com/chef/chef-server/issues/544
