---
title: "outcomes limitations support"
date: 2019-03-26T08:47:11+01:00
draft: true
---
#### Outcomes
1. **package_meta** - Standard package data 
    - Used when filter on action_date (epoch in microseconds)
    > Use of action_date column as a preliminary first-level basic filter is **required & mandatory**.    
2. **package_multidimensional** - Multi-dimensional package data which is supposed to be used only **when action_date filter is NOT possible**. 
Used to query package data using **ANY BUT AT LEAST** one of the following - 
    - **cs.sd** -> scan timestamp of package scan
    - **cd** -> package's created timestamp 
    - **wbn** -> waybillnumber which was either created / updated in last 6 months
        > All the wbns of a particular client ends up a in a single partition so querying for list of wbns of a client it's faster.  
    > This is first level preliminary basic filter. As of now other data querying sources support only action_date / ad.
3. When query uses **client key (cl) as a filter**, planning and execution time is significantly reduced.
4. Can add more first level preliminary filters at a trade-off to space / IO if highly needed. eg. cs.ud, date_mnd, cs_uid, ad

#### Query level use-cases
1. ANY query which is **NOT I/O bound**. What that means is the query should return a certainly limited number of records. A ballpark number is **<1000**.
    - Please don't get confuse with number of records. The query can scope hundreds of thousands of records but with proper filters.     
2. Since it's still a RDBMS, it can serve hundreds of concurrent (but limited parallel) query executions. 

#### Scalability  
* Able to server data of packages created in last **6 months** at **3x** the 2020's peak package scale (though limited by storage cost) 
 
#### Caveats and Limitations
* Significant Increased maintenance cost ->  Have to main 2 different tables for 1 pipeline and update the view also.  
* Keep the sync b/w 2 tables i.e. pipeline partitioned table and it's global table.  
* We cannot use detech and drop partitions. Cleanups has to be still on DELETEs.
* Probably some rewrite in dbcleanup 
* For every table need to write some implementations, brainstorming, use-case evaluations.   
  
#### Bloating
* The bloating is concern in mainly global table indexes which needs proper cleanup regularly 
* The partitioned table and it's indexes will have significantly low bloating

#### Probable Concerns  
* Implications of Trigger and impact on scale

#### Perks and Benefits 
* With postgres we get a posix compliant SQL with extremely wide library of functions. 
* People are already comfortable in querying from postgres 

#### What about other pipeline integration? 
The bag table will be first level partitioned by created date and then second level by bs. The reason is that bag is joined to other tables based on bs.
Similarly, trips will also be partitioned on created date first and then on trip_uuid at second level.    

#### Infrastructure limitations as of 24 Dec 2020
* We are heavily dependent on pg v12. Partitioning improvements(pruning, join, aggregates) and number of partitions chosen.  
* We are looking forward to upgrade for pg v13 too because of more partitioning features. See [this](https://www.postgresql.org/docs/13/release-13.html#id-1.11.6.6.5) and v13 is in preview on RDS 
* Aurora is still on v11.x so no scope for converting to aurora for a long time. 
* pg_partman and pg_cron will be available in RDS with pgv13 soon but no commitments for aurora
* No disk compression extension of postgres in RDS neither zfs FS is supported.
* No multiple disk support in RDS. So all partitions have same storage IO bandwidth.

### Resource Allocation Strategies 
* [Priorities](https://wiki.postgresql.org/wiki/Priorities) set work mem at user level
* ALTER USER johndoe WITH CONNECTION LIMIT 2;
* https://towardsdatascience.com/how-to-handle-privileges-in-postgresql-with-specific-use-case-and-code-458fbdb67a73


#### Future Scope
* Mandate usage of any of preliminary first level filter columns (seems doesn't exist natively)
* Parallel_workers in postgres create table for new partitions (exploration)  
* Cluster based on action_date [this](https://www.postgresql.org/docs/9.1/sql-cluster.html)
* Explore privelegs and control in depth
* Incorporate Platform use-cases
* RDS Proxy for pooling
* Cluster by on action_Date / cs_ud 
* Explore compaction to latest event in the batch before inserting/updating
* Splitting of table into multiple tables based on priority of columns (kind of columnar storage) - Suggested by AWS
* Since we only need partitioned table for query on action_date filter we can avoid replicating global table to niche replicas
 
# Exploration
* https://pghintplan.osdn.jp/pg_hint_plan.html
* pg_proctab
* pg_repack or better 
* pg_stat_statements
* pglogical 2
* pgrowlocks 
* pgstattuple
* plv8
* tablefunc
* tsm_system_rows
* tsm_system_time
* uuid-ossp
* amcheck
* postgresql-hll    

#### Costing
Refer [this](https://docs.google.com/spreadsheets/d/1W31iDfLGPRYvIAdFYs-zIhc-KtpYe98hVidLaNGYSKI/edit#gid=0)
