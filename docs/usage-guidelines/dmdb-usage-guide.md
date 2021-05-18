---
title: "dmdb usage"
date: 2019-03-26T08:47:11+01:00
draft: true
---
#### Definition of Datamart (dm)
Datamart is a subset / aggregation / filtered / limited by columns of Raw. 


#### Is DMDB right for you? (Use-cases)


#### Understanding the raw dataset naming convention
Convention: 
1. Latest Based Table - {schema_name}_{naming_convention}.{table_name}__{primary_key_column_list_on_which_the_latest_is_taken_on}__{method}_{column_name_convention}
eg. Package Scan Latest - raw_u.package_scan__wbn_cs_uid__latest_u
2. Append Only Table - {schema_name}_{naming_convention}.{table_name}_{column_name_convention}
eg. Audit Scan Append - raw_u.audit_scan_u

#### Quering data Guidelines 
1. Always action_date as the first filter and never forget to put typecast of bigint in the end. action_date is the epoch in microseconds.
    Eg. 
    ```postgres-sql
   -- Get today's (IST) data and avoid scanning future partitions
    WHERE action_date >= cast(extract(epoch FROM ((date_trunc('day',(CURRENT_TIMESTAMP + interval '330' MINUTE)) - interval '0' DAY) - interval '330' MINUTE)) * 1000000 AS BIGINT) 
    AND action_date < cast(extract(epoch FROM ((date_trunc('day',(CURRENT_TIMESTAMP + interval '330' MINUTE)) + interval '1' DAY) - interval '330' MINUTE)) * 1000000 AS  BIGINT)
    ```
2. Never continue running query which is taking more than 1 minute on DMDB. Expect results within seconds. 
Always run `explain {your_query}` to check whether you are scanning only the required partitions if your query is running long. 

3. Check for Indexes on tables - Refer doc   create-indexes-in-dmdb-guidlines.md 

4. Follow the optimization techniques mentioned in
https://docs.google.com/presentation/d/1hrtB0cJ3t98CyhoTRKk5EQARXtvAFU38vGdjTgBsEQ0/edit?usp=sharing
 
