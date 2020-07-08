---
layout: post
title: "Backup MySql Docker Container"
date: 2018-11-01
---

Here is how you can make mysqldump on container that created from mariadb image

```
docker run -it --link db_1:mysql --rm mariadb sh -c 'exec mysqldump -h"$MYSQL_PORT_3306_TCP_ADDR" -P"$MYSQL_PORT_3306_TCP_PORT" -uroot -p"$MYSQL_ENV_MYSQL_ROOT_PASSWORD" wordpress' > /backup/wordpress-$(date +\%F).sql
```

This command does the following:  
1. creates new container from mariadb image
2. configure a link to your db container (db_1)
3. run the mysqldump command inside the new container
4. save the output of mysqldump command to a file
5. remove the new container
