---
title: "db-design-use-case-problem-analysis"
date: 2019-03-26T08:47:11+01:00
draft: true
---
#### Project Use Cases 
The project is intended for creation and adoption of Datamarts in the org. 
As by definition data mart is a subset of the data warehouse(datalake) and is usually oriented to a specific business line or team.
We are moving towards a pattern where people will have access to data limited by tables,columns and even rows (by filtering).
For most requirements people won't have access to raw data instead they'll access to aggregated data with limited columns.

##### Type of Queries 
action_date
cs.sd

wbn - CS (client support) yeh waybill de dedo
calibration waybills - sorters (checking waybills)

join on bs, trip_id
Freshdesk / Escalation - specific wbn with daterange

#### APPLICATION HLD 
TODO:  

#### DB DESIGN
 
##### Partitioning Consideration 
* Prefer partitioning to small partitions size (say 1M to 5M records per partition) because smaller partitions will have smaller tables and indexes (faster lookups, low IO)   
* With smaller indexes it's possible for postgres to keep whole of index in-memory (probably recent ones/more queried) and better manage the memory for indexes, table cache   
* With postgres 12 going close to ~30-60 partitions is a safe bet. See [this](https://www.2ndquadrant.com/en/blog/postgresql-12-partitioning/)
* Partition pruning in extremely^2 important both for query planning time and query execution time. So our partitioning technique should make pruning happen in every query. See [this](https://www.2ndquadrant.com/en/blog/partitioning-improvements-pg11/)
* With smaller partitions, we have a chance for sequential reads only and no index scans. See [this](https://stackoverflow.com/questions/5203755/why-does-postgresql-perform-sequential-scan-on-indexed-column/5203827)

##### Why  Partitioning **only on** created_date is a not so good idea (eg. cd in package) ?
* With cd we won't actually be able to use that because, people can't write query on cd. They need something like action_Date, updated_date, ad etc.
* When we partition by cd for no real benefit(lets say), we are trading (loosing) something or the other. 
* With cd partitioning if people write query on action_Date, updated_date, ad (assuming we have index on that) we still miss the pruning. **Pruning happens only when we query data on partitioned column.**    

##### Then, How to partition table supporting upserts with partitioning pruning ??  
The ideal approach of this problem would have been to partition data on something like action_date, ad or updated_date and moving data between partitions when running UPSERTs. 
This cannot happen in postgres because - 
* There are no global indexes in postgres partitioned table yet. See -> [Postgres Proposal Link](https://www.postgresql.org/message-id/CALtqXTcurqy1PKXzP9XO%3DofLLA5wBSo77BnUnYVEZpmcA3V0ag%40mail.gmail.com)  
* Since there are no global indexes (i.e. indexes exist per partition ) we cannot create a ON CONFLICT condition globally, so cannot fire a UPDATE which moves data accross partitions. 

> Please note postgres does support partition movement in UPDATE query. Please don't confuse with limitation of UPSERT command as explained above. See [this](https://www.enterprisedb.com/blog/row-movement-across-postgresql-partitions-made-easy)

> [FAQs](https://docs.google.com/document/d/1dq03ttTeuVi_enk_YXPvxA_EWO8MXLpFktzjFimsZRA/edit#)
