---
layout: post
title: "MySql: Get Database And Tables Size"
date: 2020-07-20
---

## Tested On

DB: MariaDB-10.2.12

If you have size issue with your mysql server and you want to understand which database or tables consume all the space you can use the following queries:

* List all databases with size in MB

```
SELECT table_schema AS "Database",
ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS "Size (MB)"
FROM information_schema.TABLES
GROUP BY table_schema;
```

* List all tables of a specific database and order them by size in MB

```
SELECT table_name AS "Table",
ROUND(((data_length + index_length) / 1024 / 1024), 2) AS "Size (MB)"
FROM information_schema.TABLES
WHERE table_schema = "database_name"
ORDER BY (data_length + index_length) DESC;
```

don't forget to change "database_name" with the correct database name

## Notes

* I got the above commands from <https://www.a2hosting.com/kb/developer-corner/mysql/determining-the-size-of-mysql-databases-and-tables>

* To see if a table has been use, you can use the following post to check when was the latest update in your table:
[Get-The-Latest-Record-MySQL]({% post_url 2020-07-20-Get-The-Latest-Record-MySQL %})
