---
layout: post
title: "Get The Latest Record in MySql"
date: 2020-07-20
---

In order to see when was your latest update in your table, you can use the following sql query to get the latest record:

```
SELECT * FROM table_name
WHERE date = (SELECT MAX(date) FROM table_name);
```

Just change table_name and date field with the correct table name and the field that include the date modification in your table

## Notes

* To get the current size of all tables in your Database or get the size of all databases you can use the following:
[Mysql-Get-Database-And-Tables-Size]({% post_url 2020-07-20-Mysql-Get-Database-And-Tables-Size %})
