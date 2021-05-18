---
title: "navigation-around-views-and-tables"
date: 2019-03-26T08:47:11+01:00
draft: true
---
##### Query to get table name from view name
 
```postgres-sql
-- referenced_table_name will contain the tablename
SELECT * FROM (SELECT u.view_schema AS schema_nameD,
u.view_name, u.table_schema AS referenced_table_schema,
u.table_name AS referenced_table_name, v.view_definition
FROM information_schema.view_table_usage u
JOIN information_schema.views v
ON u.view_schema = v.table_schema
AND u.view_name = v.table_name
WHERE u.table_schema NOT IN ('information_schema', 'pg_catalog')

--- CHANGE VIEW NAME HERE
 AND u.view_name= '{view_name}'

ORDER BY u.view_schema,
u.view_name) AS VIEW_DETAILS;

-- Source https://dataedo.com/kb/query/postgresql/list-tables-used-by-a-view
```

##### Query to get move around in partition table tree hierarchy
```postgres-sql
SELECT * FROM pg_partition_root(table_name);
SELECT * FROM pg_partition_ancestors(table_name);
SELECT * FROM pg_partition_tree(table_name);
```


