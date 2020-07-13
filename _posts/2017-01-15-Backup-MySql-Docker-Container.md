---
layout: post
title: "Backup MySql Docker Container"
date: 2018-11-01
---

Here is how you can make mysqldump on container that created from mariadb image

```
docker exec -it root_mysql_1 sh -c 'exec mysqldump -p"$MYSQL_ROOT_PASSWORD" wordpress' > /backup/wordpress-$(date +\%F).sql
```

This command does the following:  
1. run mysqldump command inside your mysql container
2. use root password from MYSQL_ROOT_PASSWORD env
3. dump to a backup file with the date of today
