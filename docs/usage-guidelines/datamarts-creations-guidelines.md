---
title: "datamart creation guidelines"
date: 2019-03-26T08:47:11+01:00
draft: true
---
## DataMarts Creation/Updation in DMDB 

#### Why use datamarts and not access data directly with full SQL from application (the widely/mostly known acceptable way)
1. Reusability - With each metric anyone writes it is contributed to Org and will be available to use for others. 
Numerous users do not have to write same query again.    
1. Standard Of metrics definition - When a metric calculation in a single way Org wide, it will be easier to standardize the definition 
for that metric. 
1. Democratization -   
    1.  Security 
        1. Auditing - Who is currently accessing what. Is that allowed? 
        1. Logging - Logging of database errors with User level information to identify the wrongdoers fast 
        1. Logical isolation - Isolation of schemas (aka namespaces) shows the user only those datasets that the user is authorized to do so 
    2. Privilege Control - Who has access to what datamarts,  how many columns
         


### Schemas in Datamarts that users should be aware of
1. raw_u (aka **base datamart**)  - It contains the data at raw level, all columns, unfiltered. The _u means the columns names are separated by undescore. 
    
    Example cs_ud, cs_sd
    The columns which already has underscore in them stays the same. Eg. trip_uuid    
    
2. raw_d (aka **base datamart**)  - It contains the data at raw level, all columns, unfiltered. The _d means the columns names are separated by dot.
    
    Example cs.ud, cs.sd, action_date,
    The dot convention is usually for data which comes from mongoDB.
    
3. dm (aka **datamart / intermediate datamart**) - It contains all the datamarts that the dmdb has and is supposed to be the primary knowledgebase of datamart.
This is the schema which will be **visible in Data Catalog**.
Every datamart is either getting data from raw* or from another datamart from the same schema.

4. {user_schema} eg. stm, eye (aka **exposed datamart**) - It's the schema which is visible to the same named user. 
This schema is exposed to application/backend/dashboard and should be queried by connecting to database programmatically with the username and password. 
Eg The stm user will query data for stm dashboard and only stm schema is visible to that user.

    Why we need the {user_schema} - 
    
    1. It gives control on whom can data access how much data from dm schema. 
    1. Logical layer separation gives debuggability with better audit logging
    1. When the underlying datamart changes its definition, this layer protects the stm user query by providing backward compatibility, control on number of columns, order of columns etc. so no application code changes/releases.
    1. Reduces the error vulnerability since the stm user can only access datamarts under it's own schema only  
    1. Ability to customize or tweak the metric specifically for a particular dashboard 
    
5. metrics - This is schema which stores all the pre-calculated metrics of historical data. Only materialized historical metrics 
are suppposed to be stored.

### Datamart Access Interfaces in _dm_ schema and _{user_schema}_
For every widget there has to be **TWO** datamart access interfaces -

1.  **LIVE** datamart access interface -
    1. It is based on runtime calculation/computation of the logic/query. 
    1. Since this executed eachtime, it is supposed to be used for only a small window of data to get output in milliseconds.
    1. Metrics of all any time can be queries via this including yesterday, today, current_hour and even current minute.
         
1.  **MATERIALIZED** datamart access interface - 
    1. It is based on pre-calculated metrics stored in database.
    1. It queries already calculated metrics from a metric table by providing a range of metric interval. 
    1. There are no limitations on range of metrics. Eg. We can get a metric with with a range of 1 year to make a quarter-on-quarter trend
    1. Metrics are available of DAY BEFORE YESTERDAY and BEFORE i.e. (<T-1/<=T-2). In other words metrics of today and yesterday are not available in this.
        1. The metrics is calculated at day level of previous day. So each day, the previous day's metrics is calculated and stored. 
        We cannot the previous day metric too because, we don't know when that metric is scheduled. It could be at 1:00, 5:00, 11:00, 14:00 or whatever.
        1 full day is reserved for the metric to get calculated and get saved.     
     
####  Convention naming for _LIVE_ datamart interfaces
For dm schema - 
> **dm.{metric_name}\__for\_{entity}\__for\_{duration}**

Examples:
```postgres-psql
dm.fdds_cod__for_facility_ids__for_day('{IND148101AAA, INHPAADY}',CURRENT_DATE)
dm.fdds_cod__for_client_ids__for_day('{AMAZON, SNAPDEAL_SURFACE}','2020-02-25')
```

For {user_schema} schema - 
> **user_schema}.{user_schema}__{metric_name}\__for\_{entity}\__for\_{duration}**

Example:
```postgres-psql
stm.stm__fdds_cod__for_facility_ids__for_day('{IND148101AAA, INHPAADY}',CURRENT_DATE)
```

#### Convention naming for _MATERIALIZED_ datamart interfaces
For {user_schema} schema -
> **{user_schema}.{user_schema}__{metric_name}\__for\_{entity}\__for_range__at_day_level**
> **{user_schema}.{user_schema}__{metric_name}\__for\_{entity}\__for_range__for_granularity**

>Examples: 
```postgres-psql
stm.stm__fdds_cod__for_facility_ids__for_range__at_day_level({'IND148101AAA, INHPAADY'},'2020-03-25','2020-03-28')
stm.stm__fdds_cod__for_facility_ids__for_range__for_granularity({'IND148101AAA, INHPAADY'},'2020-03-25 12:00:00','2020-03-28 13:00:00','1m')
```
#### Creating the datamarts - (Working Examples)
##### LIVE 

Creating the datamart in _**dm**_ schema - 
```postgres-psql
--BOILER_PLATE_CODE
CREATE OR REPLACE FUNCTION dm.fdds_cod__for_facility_ids__for_day(facility_ids_filter text[],ist_date_filter DATE)  
RETURNS SETOF RECORD AS
$func$
DECLARE
    utc_timestamp_filter timestamp := ist_date_filter - interval '330' MINUTE;
BEGIN
RETURN  QUERY

-- YOUR_QUERY_STARTS_HERE
-- explain analyze
SELECT *
FROM (
	SELECT DATE (fadt) dt, "cs_slid", count(wbn) volume, sum(CASE 
				WHEN (
						ss = 'Delivered'
						AND DATE (fadt) = DATE (ldd)
						)
					THEN 1
				ELSE 0
				END) fdd
	FROM (
		SELECT cs_slid, wbn, mwn, date_fadt, dd_dct dct, cs_sd sd, dd_fdd, cs_ss ss, (date_fadt + interval '330' MINUTE) fadt, (ldd + interval '330' MINUTE) ldd, row_number() OVER (
				PARTITION BY wbn ORDER BY action_date DESC
				) AS ROW
		FROM (
			SELECT cs_slid, wbn, mwn, date_fadt, dd_dct, cs_sd, dd_fdd, cs_ss, ldd, pt, action_date, cnid, cl
			FROM raw_u.package_scan__wbn_cs_uid__latest_u
			WHERE action_date >= cast(extract(epoch FROM ((date_trunc('day', (utc_timestamp_filter + interval '330' MINUTE)) - interval '0' DAY) - interval '330' MINUTE)) * 1000000 AS BIGINT)
				AND action_date < cast(extract(epoch FROM ((date_trunc('day', (utc_timestamp_filter + interval '330' MINUTE)) + interval '1' DAY) - interval '330' MINUTE)) * 1000000 AS BIGINT)
				AND "cs_slid"  = any (facility_ids_filter)
			) p
		WHERE cnid = "cs_slid"
			AND cl NOT IN ('DELHIVERY INTERCHANGE B2B', 'DUMMY B2B', 'DLV Internal Test', 'WalletTest3', 'Delhivery E POD', 'Delhivery', 'Delhivery E POD', 'DEL LS', 'FTPL XB EXPRESS', 'SNAPDEALXB EXPRESS', 'URBANIC XB EXPRESS', 'PRICE DROP XB EXPRESS', 'ANSERX XB B2B', 'CHUMXB EXPRESS')
			AND cs_sd >= ((utc_timestamp_filter + interval '330' MINUTE)::DATE - interval '0' DAY) - interval '330' MINUTE
			AND pt IN ('COD')
		) AS REF
	WHERE ROW = 1
		AND (
			mwn IS NULL
			OR mwn = wbn
			)
		AND date_fadt >= ((utc_timestamp_filter + interval '330' MINUTE)::DATE - interval '0' DAY) - interval '330' MINUTE
	GROUP BY cs_slid, DATE (fadt)
	) fds;


--BOILER_PLATE_CODE
END;
$func$ LANGUAGE plpgsql VOLATILE;
```

Creating the exposed datamart in _**stm**_ schema by accessing the above datamart 
```postgres-psql
--BOILER_PLATE_CODE
CREATE OR REPLACE FUNCTION stm.stm__fdds_cod__for_facility_ids__for_day(facility_ids_filter text[],date_filter DATE)
RETURNS TABLE (dt DATE,cs_slid text,volume bigint,fdd bigint) AS
$func$ 
#variable_conflict use_column
BEGIN
RETURN QUERY

-- YOUR_QUERY_STARTS_HERE
SELECT dt,cs_slid,volume,fdd 
from dm.fdds_cod__for_facility_ids__for_day(facility_ids_filter,date_filter) f(dt DATE,cs_slid text,volume bigint,fdd bigint);

--BOILER_PLATE_CODE
END;
$func$ LANGUAGE plpgsql VOLATILE;
```

##### MATERIALIZED
For creation of datamarts based on materialized datamart access interface refer [metric_table-and-materialized-datamarts](./metric_table-design-and-usage.md)
 

#### Queries from Application Code with stm user -
LIVE
```postgres-sql
-- Accessing Today's Metric for facitlities - IND148101AAA, INHPAADY, IN152026A1C
SELECT dt,cs_slid,volume,fdd from stm__fdds_cod__for_facility_ids__for_day('{IND148101AAA, INHPAADY, IN152026A1C}',CURRENT_DATE)

-- Accessing Yesterday's Metric for facitlities - IND148101AAA, INHPAADY, IN152026A1C
SELECT dt,cs_slid,volume,fdd, trunc(((fdd::float4/volume::float4)*100)::numeric,2) as "fdd%" from stm__fdds_cod__for_facility_ids__for_day('{IND148101AAA, INHPAADY, IN152026A1C}',CURRENT_DATE-1);

-- Accessing 2021-04-07 Metric for facitlities - IND148101AAA, INHPAADY, IN152026A1C
SELECT dt,cs_slid,volume,fdd from stm__fdds_cod__for_facility_ids__for_day('{IND148101AAA, INHPAADY, IN152026A1C}','2021-04-07')

```
MATERIALIZED
```postgres-sql
-- Accessing Metric of range 2021-04-05 to 2021-04-07  for facitlities - IND148101AAA, INHPAADY, IN152026A1C
SELECT date_value,facility_id,total as volume,fdds as fdd from stm.stm__fdds_cod__for_facility_ids__for_range__at_day_level('{IND148101AAA, INHPAADY, IN152026A1C}',CURRENT_DATE-10,CURRENT_DATE-1)

```
COMBINED
```postgres-sql
-- Accessing Metric Last 10 days for facitlities - IND148101AAA, INHPAADY, IN152026A1C
SELECT date_value,facility_id,sum(volume) as volume,sum(fdd) as fdd, trunc(((sum(fdd)::float4/sum(volume)::float4)*100)::numeric,2) as "fdd%" from (
SELECT dt as date_value,cs_slid as facility_id,volume,fdd from stm__fdds_cod__for_facility_ids__for_day('{IND148101AAA, INHPAADY, IN152026A1C}',CURRENT_DATE-1)
 UNION ALL
--# TODO -- This typecast into bigint should not be required since function will already do it inside
SELECT date_value,facility_id,total::bigint as volume,fdds::bigint as fdd from stm.stm_fdds_cod__for_facility_ids__for_range__at_day_level('{IND148101AAA, INHPAADY, IN152026A1C}',CURRENT_DATE-10,CURRENT_DATE-1)
 ) fv
group by 1,2 
```

> Note: Schema name **stm.** is not required i.e. when the user access its own named schema 


#### Creating the datamart for Datacatalog (catalog.delhivery.com)
The datacatalog would require views for displaying sameple/dummy data from a datamart. 
So a view should be **created by datamart authors** to list it into datacatalog. 
The datamart authors also have to write datamart description, column description etc. in datacatalog. 
   
```postgres-sql
CREATE OR replace  view dm.b2b_center_connect as
select * from stm.b2b_center_connect_for_facility_ids((select array_agg(property_facility_facility_code) from raw_u.facility__facility_code__latest_u),CURRENT_DATE); 
```
##### Why FUNCTIONS? and NOT VIEWS for database ? 
1. Views are limited when passing filter arguments like facility_ids, client_ids in query while putting WHERE clause while querying the views.

    Since postgres re-writes every query it pushes this filter parameter inside and filters at the initial level. However, this doesn't happen everytime specially the queries in which we do a window function filter or complex aggregates.
    If postgres doesn't understand to put the filter in the initial level, then it calculates the result for all facilties/clients and then later on filters the output  leading to increased response times.
        
2. Increasing Vulnerability and decreasing enforcement on how to query datamart - 
    If we use views it is upto the user that it specifies the facilities_ids/clients_ids. A user is free to query data of all centers and take all data into application and then filter in application, which is anti-pattern and leads to high execution time.
    
3. Views makes more problems to update than functions. 
    Whenever we update the column type, order of columns, or number of columns views doesn't allow us to replace on the fly. 
    In functions, with datamarts written in dm schema, they are schemaless so updating them is easier.    
     
