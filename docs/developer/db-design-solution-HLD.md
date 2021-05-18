---
title: "Db-Design-Solution"
date: 2019-03-26T08:47:11+01:00
draft: true
---
> Please read [USE CASES and PROBLEM ANALYSIS of DB DESIGN ](db-design-use-case-problem-analysis.md) before reading this.

## Partitioning Db Design Solutioning HLD

###### Solution in 1 liner: Global Indexes in a combination to hash-partitioned tables
  
### Terminologies 
> *partitioned* -> The partitioned refers to the table is partitioned, mostly with hash based partitioning  
 
> *global*  -> Global means non-partitioned table primarily made to access it's indexes  

Every pipeline will require 2 tables namely {pipeline_name}._prtd and {pipeline_name}._gbl

### Sample DB Design and Explanation

#### Create the schemas
```postgres-sql
-- Trying not to create anything in public schema. Trying to keep things clean, and controlled from start 
create schema partitioned;
create schema partitions;
create schema subpartitions;
create schema global;
create schema raw; -- aka base datamart 
create schema dm; -- aka Intermediate data marts
-- create  schema eye; -- aka exposed datamart. This should be mapped with username one to one 
```

#### Creating partitioned table of pipeline   

```postgres-psql
-- Creating a separate schema to distinguish b/w normal tables and partitioned 
-- Also adds scope for better privelege control

create TABLE partitioned.package_prtd (
wbn text, 
"cs.st" text,
"cs.ss" text,
cl text,
"cs.uid" text,
action_date bigint,
"cs.sd" timestamp,
cd timestamp,
event_type text
) PARTITION BY RANGE (action_date);

-- Used to get the epochs in microseconds
--select (extract(epoch from '2020-12-01'::timestamp)*1000000)::bigint; 

CREATE TABLE partitions.package_2020_12_01_to_2020_12_05 PARTITION OF partitioned.package_prtd
FOR VALUES FROM (1606780800000000) TO (1607126400000000) PARTITION BY HASH(cl);

CREATE TABLE partitions.package_2020_12_05_to_2020_12_10 PARTITION OF partitioned.package_prtd
FOR VALUES FROM (1607126400000000) TO (1607558400000000)  PARTITION BY HASH(cl);
 
-- Make default partition for safety. Ideally when partitioned on action_date this should not be used
CREATE TABLE partitions.package_default_partition PARTITION OF partitioned.package_prtd DEFAULT;

-- Creating a separate schema for partitions to make sure we don't clutter up the schema user is seeing
-- This is supposed to be an internal schema and no user should be interested in seeing this  

create TABLE subpartitions.package_2020_12_01_to_2020_12_05__00 PARTITION OF partitions.package_2020_12_01_to_2020_12_05
    FOR VALUES with (MODULUS 3, REMAINDER 0);
create TABLE subpartitions.package_2020_12_01_to_2020_12_05__01 PARTITION OF partitions.package_2020_12_01_to_2020_12_05
    FOR VALUES with (MODULUS 3, REMAINDER 1);
create TABLE subpartitions.package_2020_12_01_to_2020_12_05__02 PARTITION OF partitions.package_2020_12_01_to_2020_12_05
    FOR VALUES with (MODULUS 3, REMAINDER 2);


create TABLE subpartitions.package_2020_12_05_to_2020_12_10__00 PARTITION OF partitions.package_2020_12_05_to_2020_12_10
    FOR VALUES with (MODULUS 3, REMAINDER 0);
create TABLE subpartitions.package_2020_12_05_to_2020_12_10__01 PARTITION OF partitions.package_2020_12_05_to_2020_12_10
    FOR VALUES with (MODULUS 3, REMAINDER 1);
create TABLE subpartitions.package_2020_12_05_to_2020_12_10__02 PARTITION OF partitions.package_2020_12_05_to_2020_12_10
    FOR VALUES with (MODULUS 3, REMAINDER 2);

--Change Note: #TODO Improve This
-- Partitioning Key => (action_date, cl, wbn) in above example but on prod the Partitioning Key is => (action_date,cl,cs.slid,wbn)
-- Primary Key cannot be made because of partitioning key can be null. 
--In above case we are using action_date , cl and wbn out of which no key is null but on prod, since we are have cs.slid it can be null
-- So we didn't create any primary key.
--Though THe primary key constraint is covered by global table constraint

  -- required for unique index which is eligible for onflict arg

-- Multicolumn partitioning of cl and wbn is thought but requires both to be in query to get pruned. 
-- Refer [this](https://www.enterprisedb.com/postgres-tutorials/what-multi-column-partitioning-postgresql-and-how-pruning-occurs) 
```
> Do not attempt to play on client prefix id as they are broken with aramax integrations. Aramax uses 20-25 client prefix instead of 1 
#### Creating the global table for pipeline (For global indexes)
 
```postgres-psql
-- This schema shall also contain the function too which is maintaining the sync
CREATE TABLE global.package_gbl (
wbn text, 
cl text,
action_date bigint,
"cs.sd" timestamp,
cd timestamp,
PRIMARY KEY(wbn)
);

-- Not creating Primary Key Constraint on Global Table since it's not required. 
-- Assuming global table will always be in sync with partitioned table
-- There's also no need for an index because the other indexes should always contain wbn within it
-- i.e. always create compound (multicolumn) index eg. (cd_sd,cl,wbn) , (action_date, cl,wbn)

-- Sync Functiona 
CREATE OR REPLACE FUNCTION global.package_prtd_function()
  RETURNS trigger AS
$func$
BEGIN
   CASE TG_OP
   WHEN 'INSERT' THEN
      INSERT INTO global.package_gbl (
      wbn,cl,action_date,"cs.sd",cd)
      VALUES (
      NEW.wbn,NEW.cl,NEW.action_date,NEW."cs.sd",NEW.cd
      );
      RETURN NEW;
   WHEN 'DELETE' THEN
      DELETE FROM global.package_gbl
      WHERE  wbn = OLD.wbn;
      RETURN OLD;
   WHEN 'UPDATE' THEN 
      UPDATE global.package_gbl 
      SET (cl,action_date,"cs.sd",cd) = (NEW.cl,NEW.action_date,NEW."cs.sd",NEW.cd)
      WHERE  package_gbl.wbn = OLD.wbn;
      RETURN NEW;
   ELSE
        RAISE EXCEPTION 'Unhandled Case in Trigger NEW --> % OLD --> %', NEW, OLD;
   END CASE;  
END
$func$  LANGUAGE plpgsql VOLATILE;

-- Trigger which is attached on partitioned table
-- Note: Putting a trigger before since insertion should happen in global table to check on constraint
-- This is done as a PRIMARY(action_date,cl,wbn) is vulnerable as action_date is a running key. So there's a 
-- chance that a new action_date with same wbn could get enter in case of an application level bug
-- To stop that we'll insert in global table first since it'll have a unique constraint of wbn     
-- Note: Trigger have to be on leaf partitions as postgres doesn't support BEFORE partitions on parent partitioned table 
-- Need postgres 13 for creating triggers on partitioned table    
CREATE TRIGGER package_2020_12_01_to_2020_12_05__00_trigger
   BEFORE INSERT OR UPDATE OR DELETE on subpartitions.package_2020_12_01_to_2020_12_05__00
   FOR EACH  ROW 
       EXECUTE PROCEDURE global.package_prtd_function();

CREATE TRIGGER package_2020_12_01_to_2020_12_05__01_trigger
   BEFORE INSERT OR UPDATE OR DELETE on subpartitions.package_2020_12_01_to_2020_12_05__01
   FOR EACH  ROW 
       EXECUTE PROCEDURE global.package_prtd_function();

CREATE TRIGGER package_2020_12_01_to_2020_12_05__02_trigger
   BEFORE INSERT OR UPDATE OR DELETE on subpartitions.package_2020_12_01_to_2020_12_05__02
   FOR EACH  ROW 
       EXECUTE PROCEDURE global.package_prtd_function();

CREATE TRIGGER package_2020_12_05_to_2020_12_10__00_trigger
   BEFORE INSERT OR UPDATE OR DELETE on subpartitions.package_2020_12_05_to_2020_12_10__00
   FOR EACH  ROW 
       EXECUTE PROCEDURE global.package_prtd_function();

CREATE TRIGGER package_2020_12_05_to_2020_12_10__01_trigger
   BEFORE INSERT OR UPDATE OR DELETE on subpartitions.package_2020_12_05_to_2020_12_10__01
   FOR EACH  ROW 
       EXECUTE PROCEDURE global.package_prtd_function();

CREATE TRIGGER package_2020_12_05_to_2020_12_10__02_trigger
   BEFORE INSERT OR UPDATE OR DELETE on subpartitions.package_2020_12_05_to_2020_12_10__02
   FOR EACH  ROW 
       EXECUTE PROCEDURE global.package_prtd_function();

-- Trigger of default partition 
CREATE TRIGGER package_default_partition_trigger
   BEFORE INSERT OR UPDATE OR DELETE on partitions.package_default_partition
   FOR EACH  ROW 
       EXECUTE PROCEDURE global.package_prtd_function();

```

####   Bulk Data Generation 
```postgres-sql
-- Bulk Data Insertion in Partitioned table
INSERT INTO  partitioned.package_prtd select g+5961810374542,
(array['PP','UD','DL','PU','RT'])[floor(random() * 5 + 1)],
(array['Manifested','Not Picked','In Transit','Dispatched','Scheduled','Pending','Delivered'])[floor(random() * 7 + 1)],
md5((g % 100)::text),
md5(g::text),
cast (extract ( epoch from ('2020-12-01 14:26:02.960718+00'::timestamp + ( g || ' second' ) :: interval))*1000000 as bigint),
'2020-12-01 14:26:02.960718+00'::timestamp + ( g || ' milliseconds' ) :: interval,
'2020-12-01 14:26:02.960718+00'::timestamp + ( g || ' second' ):: interval
from generate_series(1,1000000) as g;

-- This is bulk insertion for package_gbl. Usually not required since 
-- trigger will make the data available in global table too.
-- INSERT INTO  global.package_gbl select g+5961810374542,md5(g::text),g*999,CURRENT_TIMESTAMP - ( g || ' second' ) :: interval,CURRENT_TIMESTAMP - ( g || ' second' ) :: interval from generate_series(1,1000000) as g;

--Sample INSERT on conflict query 
--INSERT INTO partitioned.package_prtd values (
-- '7269364714286','DL','RTO','SNAPDEAL SURFACE','scan::1f7d9f0f-670a-48e2-9143-480ed932e3c4','1606309512002468','2020-11-25 13:05:12.407','2020-11-13 12:49:54.19'
-- )
--ON CONFLICT (cd,cl,wbn)
--DO UPDATE SET
--wbn = EXCLUDED.wbn,
--"cs.st" = EXCLUDED."cs.st",
--"cs.ss"= EXCLUDED."cs.ss",
--cl = EXCLUDED.cl,
--"cs.uid" = EXCLUDED."cs.uid",
--action_date = EXCLUDED.action_date,
--"cs.sd" = EXCLUDED."cs.sd",
--cd = EXCLUDED.cd;
```

#### Creating the Index on partitioned table
```postgres-sql
-- Creating index on wbn only because the partitioning already know the path to right partition so
-- this index will be used after reaching the partition in case of update / delete / select
create INDEX package_prtd_wbn_btree_idx ON partitioned.package_prtd using btree(wbn) ;

-- Another list of index - make more as per needs - but since we know action_Date, cl and cs.slid are most queried 
-- We'll start with those
create INDEX package_prtd_action_date_btree_idx ON partitioned.package_prtd using btree(action_date) ;
create INDEX package_prtd_cl_btree_idx ON partitioned.package_prtd using btree(cl) ;
create INDEX package_prtd_cs_slid_btree_idx ON partitioned.package_prtd using btree("cs.slid") ;
```

Test Queries ->
 
`explain analyze select * from partitioned.package_prtd where action_date > 1606832764960717 and action_date  < 1606832764960720 and cl='c81e728d9d4c2f636f067f89cc14862c';`
 
`explain analyze select * from partitioned.package_prtd where action_date > 1606832764960717 and action_date  < 1606832764960720 and cl='c81e728d9d4c2f636f067f89cc14862c' and wbn ='5961810374544';`

> While it is known, hash indexes are faster for lookups than btree, refer [this](https://www.enterprisedb.com/blog/postgresql-indexes-hash-indexes-are-faster-btree-indexes) but they cannot be used in neither in cl nor in wbn because people query somtimes on pattern like for __cl__ _'%AMAZON%'_ and for __wbn__ _'1337%'_
 
#### Indexes on Global Table
```postgres-sql
-- We need indexes for lookups of wbn to pick based on range
-- Each time user needs a new column as a primary filter it need 
-- to have in global table and create a global index on it.

-- Always create index containing the action_date,cl,wbn so we never go the global table actually

-- This index is used for update query 
create index package_gbl_action_date_cl_wbn_btree_idx on global.package_gbl using btree ( action_date,cl,wbn);

-- These index are used primarily for select when action_Date is not the first level priliminary filter
create index package_gbl_cs_sd_action_date_cl_wbn_btree_idx on global.package_gbl using btree ( "cs.sd",action_date,cl,wbn);
-- TODO Check if wbn is required
create index package_gbl_cd_action_date_cl_wbn_btree_idx on global.package_gbl using btree ( cd,action_date,cl,wbn);
create index package_gbl_wbn_action_date_cl_btree_idx on global.package_gbl using btree ( wbn,action_date,cl);

create index package_gbl_wbn_action_date_btree_idx on global.package_gbl using btree ( wbn,action_date);
```
> Why Not Brin. See [this](https://info.crunchydata.com/blog/avoiding-the-pitfalls-of-brin-indexes-in-postgres)
> Not creating a single index on cl because there seems to be no use-case for people to query all data of a cl. Even if it's there they should use the cs.sd / action_date since default partition has always some old records. 

##### Test Queries of global table(not including plans to limit doc size)
* `explain analyze SELECT wbn,cd,cl from global.package_gbl where wbn = '7269364714286';`  
* `explain analyze select wbn, "cs.sd" from global.package_gbl gp where gp."cs.sd" > '2020-12-01 14:16:52.488192' and  gp."cs.sd" < '2020-12-01 14:26:02.988192';`

#### Inserting/Updating Data 

##### Sample queries - Plain (Not Used Actually - Just to create some data)
```postgres-sql 
-- Use this to view data with partition name and delete 
-- select tableoid::regclass,* from package_prtd ;

explain analyze INSERT INTO partitioned.package_prtd values (
 '7269364714286','DL','RTO','SNAPDEAL SURFACE','scan::1f7d9f0f-670a-48e2-9143-480ed932e3c4','1606780800000006','2020-11-02 13:05:12.407','2020-11-01 12:49:54.19','CREATE'
);
INSERT INTO partitioned.package_prtd values (
 '7269364714287','DL','RTO','AMAZON','scan::2f7f9ffdf-670a-48e2-9143-6534gdfgfd','1607126400000000','2020-11-03 13:05:12.407','2020-11-01 12:49:54.19','CREATE'
);

```

##### Real Queries which will run from application 
```postgres-sql
**INSERT** -
-- We'll insert every record after checking if there's no record exist 
-- Though on-conflict with do nothing covers idempotency but there are other cases too described below     
explain analyze insert into partitioned.package_prtd (col1,col2,col3)
select 
     '7269364714286','DL','RTO','SNAPDEAL SURFACE','scan::1f7d9f0f-670a-48e2-9143-480ed932e3c4','1606780800000006','2020-11-02 13:05:12.407','2020-11-01 12:49:54.19','CREATE'
where not exists (
    select wbn from global.package_gbl where wbn = '7269364714286' 
);



**SELECT**
-- Getting the current action_date corresponding to wbn which we'll fire update for
SELECT wbn,action_date,"cs.slid" from global.package_gbl 
where wbn in( '7269364714286', '7269364714286');  

  
**UPDATE**
-- Update query specifiying the right and exact action_Date with equals so that we prune the partition right
 update partitioned.package_prtd as p 
SET "cs.st" = 'RT', action_date = 1611659129000104   
WHERE p.action_date = 1611659129000103
-- idompotency and unordered data from source handling
and 1611659129000103 < 1611659129000104
and "cs.slid" = 'Gurgaon Central'
and p.cl = 'SNAPDEAL SURFACE'
and p.wbn = '7269364714286';


-- Using Global Table join  (not used in prod) -DEPRECATED 
explain analyze update partitioned.package_prtd as p 
SET "cs.st" = 'RT' 
FROM 
(SELECT action_date as action_Date from global.package_gbl 
where  
 wbn = '7269364714286'
and cl = 'SNAPDEAL SURFACE'
and action_date < 1611659129000103
) gp 
WHERE
 p.action_date = gp.action_date 
and p.action_date < 1611659129000103
and p.cl = 'SNAPDEAL SURFACE'
and p.wbn = '7269364714286';


-- Using only partitioned table ( Scannning knowing all 1st level partitions) (not used in prod) DEPRECATED
explain analyze update partitioned.package_prtd as p 
SET "cs.st" = 'RT' 
WHERE p.action_date < 1611659129000103
and p.cl = 'SNAPDEAL SURFACE'
and p.wbn = '7269364714286';


-- Using subquery (not used in prod)  DEPRECATED
explain analyze update partitioned.package_prtd as p 
SET "cs.st" = 'RT' 
WHERE
 p.action_date = (SELECT action_date from global.package_gbl 
where  
action_date <= 1607558400000000
and cl = 'SNAPDEAL SURFACE'
and wbn = '7269364714286'
)   
and p.cl = 'SNAPDEAL SURFACE'
and p.wbn = '7269364714286';
```

###### Cases Covered - 
1. Replay Cases (How to achieve **idempotency**) and Case when data is unordered from the source itself -

    In case of insert we are checking if the record exists in global table or not. If not then only we proceed for insertion.
    In the case of update we are updating the record only when a new action_Date i.e. new action_date greater than existing 
    action_date comes, then only we are updating. 

2. Case when data older than 6 months -
 
    In the case when we receive a update after 6 months there's a possibility that record is deleted.
    Because of this case, we always need to run a INSERT before running a UPDATE for every record. 
    That is 2 query sets for 1 record.

3.  Case when we don't have event_type key -

    We don't care about event_type much because we're inserting every record first and then updating it. 
    Though there can be a optimization in future for sources which does send event_type. 

#### Combining the partitioned table and global table | Creating the view
```postgres-sql
-- Tried merge Join (by using sort by) and using in operator. 
--The below uses Nested Loop and offers best performance. See [this](https://www.cybertec-postgresql.com/en/join-strategies-and-performance-in-postgresql/)
DROP VIEW IF EXISTS raw.package_multidimensional;
CREATE OR REPLACE view raw.package_multidimensional as 
SELECT gp.wbn,gp."cs.sd",gp.cd,pp.action_date,pp.cl,pp."cs.uid",pp."cs.st",pp."cs.ss" from global.package_gbl gp
JOIN 
 partitioned.package_prtd pp 
ON 
gp.action_date= pp.action_date
AND gp.cl = pp.cl 
AND
 gp.wbn = pp.wbn
;

DROP VIEW IF EXISTS raw.package_meta;
-- TODO remove sensitive columns;  
CREATE OR REPLACE view raw.package_meta as 
SELECT * from partitioned.package_prtd; 


--TODO Make a view with underscore convention for compatibility
```

#### Selection of columns between partitioned and global table

* **cl** -> From partitioned table 
    - We are using partitioned table cl and not global table cl because it's slightly faster in query test plans,
the smaller partitioned pkey index is used and in case of direct queries we also have a dedicated index on cl
as mentioned above.
    - There's no dedicated index on cl in global table

* **wbn** -> From global table
    - specified above combines index of (wbn,cd,cl) was slightly slower than index on just wbn with a sequential scan on cd,cl.
    - Since we have a global wbn index this would give enablement to query only on wbn without any action_date / cd.sd filter
   
* **action_Date** -> From partitioned table 
    - Similar to cl we'll prefer cd from partitioned table because it uses partitioned index which will save IO.
    - There's no requirement of dedicated cd index because it has one (package_gbl_cl_wbn_btree_idx) which has cd as first key
    - In partitioned table also we have the main primary key index which has cd in first position
       
* **cd** and **cs.sd** -> From global table
     - Both the columns have compound indexes starting these column in order
     - The cd and cs.sd in partitioned_table is not indexed and no need to NOT be.
     
#### Final Query Test with Partition Pruning 
Queries: - >
  
  `explain analyze select * from raw.package_multidimensional where action_date > 1606939513960000 and action_Date < 1606939676960718 and cl = 'aab3238922bcc25a6f606eb525ffdc56' ;`                                                                                                                                                                                                                  
    
Try same query to pruning off to confirm
```postgres-sql
set enable_partition_pruning TO false;
```
                                                                                                                                                                    
> There's only a slight increase because of very low number of partitions with very substantially lesser data.

#### Clean Everything 
```postgres-psql
drop schema partitioned cascade;
drop schema partitions cascade;  
drop schema subpartitions cascade;
drop schema global cascade;
drop schema raw cascade;
drop schema dm cascade;
```

#### Extra SQL for management / debugging 
```postgres-psql
-- Add all schemas to search path for use
alter role datawarehouse set search_path = "$user", public, partitioned, subpartitions, partitions, global, raw, dm;

-- Show all schemas
select schema_name from information_schema.schemata;

-- Hierarchy
SELECT * FROM pg_partition_root('package_2020_12_01_to_2020_12_05__01');
SELECT * FROM pg_partition_ancestors('package_2020_12_01_to_2020_12_05__01');
SELECT * FROM pg_partition_tree('package_prtd');


https://www.postgresql.org/docs/current/pageinspect.html
https://www.postgresql.org/docs/12/pgbuffercache.html

can use the enable_seqscan and enable_indexscan parameters
```

#### Tuning Configs
```postgres-sql
* SET enable_partitionwise_aggregate=on;
* SET enable_partitionwise_join=on;
* SET enable_partition_pruning=on; -- Default is on

wal_compression= on
https://wiki.postgresql.org/wiki/Tuning_Your_PostgreSQL_Server
https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Appendix.PostgreSQL.CommonDBATasks.html#Appendix.PostgreSQL.CommonDBATasks.Autovacuum


#### Possible Optimizations
* Slid FIlter in update quuery to prune exactly instead of 3 tables. Blocked by null case
* Do SELECT once for INSERT queries and don't send INSERT queries which are not required
* 

```
TODO -> REview this https://www.enterprisedb.com/blog/row-movement-across-postgresql-partitions-made-easy

> Please read [OUTCOMES, LIMITATIONS, SUPPORT](outcomes-limitations-support.md) after reading this. 
