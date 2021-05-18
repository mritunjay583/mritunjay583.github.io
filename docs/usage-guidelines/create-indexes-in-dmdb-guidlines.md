---
title: "create index in dmdb"
date: 2019-03-26T08:47:11+01:00
draft: true
---
###### Using the following doc requires you the name of the table. If you don't know the tablename here's a doc through which you can find out here -  
> [navigation-around-views-and-tables](navigation-around-views-and-tables.md) after reading this. 
 

#### List Already Existing Index on a tables 
```postgres-sql
-- For listing all indexes on all partititioned table name
select * from pg_indexes where schemaname = 'partitioned';

-- For listing all indexes on all standalone table name
select * from pg_indexes where schemaname = 'standalone';

-- For listing all indexes on package__wbn__latest_prtd partititioned table 
select * from pg_indexes where schemaname = 'partitioned' and tablename = 'package__wbn__latest_prtd';

-- For listing all indexes on package_scan__wbn_cs_uid__latest_prtd partititioned table 
select * from pg_indexes where schemaname = 'standalone' and tablename = 'facility__facility_code__latest_std';
```

#### To create index

**Convention 1 of index naming** [DEFAULT] :  
 {table_name_without_schema}\_{list_of_underscore_column_name_with_underscore_replacing_dot_if_col_name_contains_dot}\_{type_of_index}_idx
```postgres-sql
-- Example to create index on wbn in  partitioned table
CREATE INDEX package__wbn__latest_prtd_cs_sd_btree_idx ON partitioned.package__wbn__latest_prtd USING btree ("cs.sd")

-- Example to create index on wbn in  partitioned table
CREATE INDEX package_scan__wbn_cs_uid__latest_prtd_cs_sd_btree_idx ON partitioned.package_scan__wbn_cs_uid__latest_prtd USING btree ("cs.sd")

-- Example to create index on property.facility.facility_type in standalone table 
CREATE INDEX CONCURRENTLY facility__facility_code__latest_std_facility_type_btree_idx ON standalone.facility__facility_code__latest_std USING btree ("property.facility.facility_type")
```
**Convention 2 of index naming when names get longer than 63 characters** [>63 chars] :
 {table_name_without_schema_without_double_underscored_embraced_column_names}\_{list_of_underscore_column_name_with_underscore_replacing_dot_if_col_name_contains_dot}\_{type_of_index}_idx

```postgres-sql
-- Example when index name exceeds the allowed length and is being truncated. Use this convention
CREATE INDEX package_scan_v1_latest_prtd_action_date_cs_slid_btree_idx ON partitioned.package_scan_v1__wbn_cs_uid__latest_prtd USING btree ("action_date","cs.slid")
```

#### Notes
1. The index has to be created on the actual table and not on the raw_* view
2. Always try to create indexes  CONCURRENTLY first in case the table is in standalone schema, if the query fails then try removing the concurrently
3. For partitioned schema, do not use  CONCURRENTLY in index creation query
4. Use DatamartDB Special to create indexes. The only "DatamartDB" will not allow to create indexes. However, for all read queries, the only "DatamartDB" is mandatory.
5. Use the appropriate index type. The btree index is not good for all use-cases. Refer documentation https://www.postgresql.org/docs/current/indexes-types.html   
6. Only create index when you need and understand what/why you are doing.


