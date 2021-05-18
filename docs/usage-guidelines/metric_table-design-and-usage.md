---
title: "metric table design"
date: 2019-03-26T08:47:11+01:00
draft: true
---
#### Problem Statement (In RTDB) 
* The current problem in RTDB is that every metric (i.e. data of every widget) is served by a dedicated table
    * Maintenance overhead of these tables like bloat, deletion etc
* Privelege control management of these metrics tables are not done. Everything is in under public schema and everyone
can access every metric
* Redundancy of metrics i.e. same (similar/overlapping metrics) are calcualated from the very start
* No structure of creating aggregations over aggregations 

#### Structure of Serving Metrics
* We have categorized the metrics in 2 parts - 
    * Day before Yesterday Metrics and even Before - MATERIALIZED in a table (aka Historical) 
    * CURRENT_DAY and PREVIOUS_DAY (IST) of Metrics - NOT MATERIALIZED - LIVE QUERY ONLY (aka recent) - 

##### Why materialization of metrics is required?
The metrics on historical data (immutable) needs to materialized in a table for the sole purpose of not running the original query 
again, because we know the isn't going  change. 
For example We calculate a metric at the end of the day for the previous day, we know that since the day is passed
no more data can be written for that date so the output of the query will never change.
The intent is to reduce the data scan overall. 

#### The Materialized Metric Table Schema
We are creating the metrics per client per facility level to open scope for more diverse aggregations on aggregated metrics.
This means we would require to put group by client_id and facility_id in each other (if the columns exist in the table)

```postgres-sql
CREATE SCHEMA metrics;

CREATE TABLE metrics.metric_registry ( 
id UUID PRIMARY KEY DEFAULT gen_random_uuid(), 
created_at timestamp default CURRENT_TIMESTAMP,
metric_name text unique not null,
created_by_email text not null,
created_for text not null,
used_by text[] not null,
stakeholder_emails text[] not null, -- ## #TODO make this infor available in Atlan
description text not null,
scheduling_reference_link text,
should_run bool
);


CREATE TABLE metrics.mat_metrics_per_client_per_facility 
(id BIGSERIAL PRIMARY KEY,
created_at timestamp default CURRENT_TIMESTAMP,
metric_registry_uuid UUID references metrics.metric_registry(id), -- FK of meta_info_table text  -- UUID Is used instead of metric_name unique because user can insert
-- value in metric table of other metric by mistake which would lead to wrong calulation of existing as well as new metric
metric_id text not null, -- Unique name to each metric  combination of metric_registry_uuid+client+facility+start_date+Granularity
metric_column_id text not null, -- Unique name to each metric column  combination of metric_registry_uuid_client+facility+start_date+end_date+metric_column_name+Granularity
date_type text not null,
date_value timestamp not null,
start_date timestamp not null, 
end_date timestamp not null, -- // #TODO Could be replaced by granularity
granularity interval not null, -- 1h, 2h, 1d, 7d
client_id text,
facility_id text, 
metric_column_name text not null,  -- ALWAYS PUT METRIC COLUMN NAME IN lower_case_with_underscore
metric_column_value text, 
metric_group text -- Entity/Group/Whatever the metric is calculating ,usually the main table entity it queris 
);


-- TODO To be partioned later on start_date, facility_id and client_id

```

#### Create a new metric definition [One Time Registration]
```postgres-sql
-- This query returns the metric_uuid for the registered metric. It's very important and will be used in below queries. 
INSERT INTO 
metrics.metric_registry (metric_name,created_by_email,created_for,used_by,stakeholder_emails,description,reporting_query_id,should_run)
VALUES ('b2b-center-connect2','kartik.modi@delhivery.com','test_usage','{stm}','{stm@delhivery.com}','test description','101','t')
RETURNING id;
```

#### Insert Data in Metric Table - The query to be Scheduled by query authors
Inserting the result of the metric in mat_metric table
```postgres-psql
WITH tmp_variables
AS (
	SELECT CURRENT_DATE + interval '1' DAY AS dt
	)
INSERT INTO  metrics.mat_metrics_per_client_per_facility(
	metric_registry_uuid
	,metric_id
    ,metric_column_id
	,date_type
	,date_value
	,start_date
	,end_date
	,granularity
	,client_id
	,facility_id
	,metric_column_name
	,metric_column_value
	,metric_group
	)
-- 4d5beeef-f1a3-4263-b044-695a307f2370 is the metric_uuid which is received by above query
SELECT '4d5beeef-f1a3-4263-b044-695a307f2370'
	,'4d5beeef-f1a3-4263-b044-695a307f2370' || '|' || COALESCE(center_code,'NULL') || '|' || CURRENT_DATE || '|' || CURRENT_DATE || '|' ||'1d'
    ,'4d5beeef-f1a3-4263-b044-695a307f2370' || '|' || COALESCE(center_code,'NULL') || '|' || CURRENT_DATE || '|' || CURRENT_DATE || '|'|| metric_column_name ||'|'||'1d'	
    ,'cd'
	,CURRENT_DATE
	,CURRENT_DATE
	,CURRENT_DATE
  ,'1d'
	,NULL
	,center_Code
	,metric_column_name
	,metric_column_value
	,'b2b'
FROM (
SELECT *
    -- ALWAYS PUT metric_column_name in lower case ONLY
	,unnest(array ['status','total','_sample_wbn','max']) AS metric_column_name
	,unnest(array [status,total::text, _sample_wbn, max::text]) AS metric_column_value
 from (


--- YOUR STANDARD SELECT QUERY 
	SELECT ocid AS center_code
		,(
			CASE 
				WHEN "cs.act" IN (
						'+L'
						,'+C'
						,'<L'
						,'<C'
						) --(<L & <C) included to account for short shipments in RT status
					THEN 'connected'
				ELSE 'not_connected'
				END
			) AS STATUS
		,CURRENT_DATE
		--         - INTERVAL '1' day AS dt
		,COUNT('*') AS total
		,max(t4.wbn) AS _sample_wbn
		,max(last_dispatched) AS "max"
	FROM (
		SELECT action_date
			,ocid
			,oc
			,"cs.act"
			,"cs.sd"
			,wbn
			,last_dispatched
			,action_center
		FROM (
			SELECT action_date
				,wbn
				,"cs.ss"
				,oc
				,ocid
				,"cs.sd"
				,"cs.act"
				,"cs.nsl"
			FROM (
				SELECT action_date
					,wbn
					,"cs.ss"
					,"cs.slid"
					,oc
					,ocid
					,"cs.sd"
					,"cs.act"
					,"cs.dwbn"
					,"cs.st"
					,"cs.nsl"
				--                                , row_number() OVER ( PARTITION BY wbn ORDER BY action_date DESC ) AS rn
				FROM raw_d.package__wbn__latest_d
				WHERE 1 = 1
					--                        and ad >= date_format((date_trunc('day', current_timestamp) - interval '1' day) - interval '00' minute, '%Y-%m-%d-%H')
					AND action_date >= (date_part('epoch'::TEXT, date_trunc('day', current_timestamp) - '330 minute'::interval) * 1000::DOUBLE PRECISION * 1000::DOUBLE PRECISION)::BIGINT
					-- and ad <= date_format((date_trunc('day', current_timestamp) - interval '0' day) - interval '00' minute, '%Y-%m-%d-%H')
					AND "date.mnd" >= (date_trunc('day', current_timestamp) - interval '0' day) - interval '330' minute
					-- and date_mnd <= (date_trunc('day', current_timestamp) - interval '0' day) - interval '330' minute
					AND cl = 'Delhivery E POD'
				) AS t1
				--                WHERE rn = 1
			) t2
		LEFT JOIN (
			SELECT action_center
				,max(TO_DATE('1970-01-01 00:00:00', 'YYYY-MM-DD HH24:MI:SS') + (action_date / 1000000) * INTERVAL '1' second) AS last_dispatched
			FROM raw_d.trips_d
			WHERE 1 = 1
				--                and ad >= date_format((date_trunc('day', current_timestamp) - interval '1' day) - interval '00' minute, '%Y-%m-%d')
				AND action_date >= (date_part('epoch'::TEXT, date_trunc('day', current_timestamp) - '330 minute'::interval) * 1000::DOUBLE PRECISION * 1000::DOUBLE PRECISION)::BIGINT
				-- and ad <= date_format((date_trunc('day', current_timestamp) - interval '0' day) - interval '00' minute, '%Y-%m-%d')
				--                AND action_date > cast(to_unixtime((date_trunc('day', current_timestamp) - interval '0' day) - interval '330' minute) as bigint)*1000000.0
				-- AND action_date > cast(to_unixtime((date_trunc('day', current_timestamp) - interval '0' day) - interval '330' minute) as bigint)*1000000.0
				AND action_performed = 'Departed'
			GROUP BY action_center
			) t3 ON ocid = action_center
		) t4
	WHERE (
			action_date < extract(epoch FROM (
				CASE 
					WHEN last_dispatched IS NULL
						OR last_dispatched > (
							(
								SELECT dt
								FROM tmp_variables
								) - INTERVAL '10' hour - INTERVAL '30' minute
							)
						THEN (
								SELECT dt
								FROM tmp_variables
								) - INTERVAL '13' hour - INTERVAL '30' minute
					WHEN last_dispatched < (
							(
								SELECT dt
								FROM tmp_variables
								) - INTERVAL '18' hour - INTERVAL '30' minute
							)
						THEN (
								SELECT dt
								FROM tmp_variables
								) - INTERVAL '37' hour - INTERVAL '30' minute
					ELSE last_dispatched - INTERVAL '180' minute
					END
				)) * 1000000
			)
	GROUP BY 1
		,2

--- YOUR STANDARD SELECT QUERY END

	) A
)B;
```

#### SELECTing Data from Mat Metric Table - (To be created for every metric)
The structure of getting metric from mat_metrics_per_client_per_facility is gonna be the same for all metric with some changes like column names output, metric_uuid, filters etc. 

> For convention naming see [datamarts-creation-guidlines](datamarts-creations-guidelines.md#convention-naming-for-_materialized_-datamart-interfaces)

```postgres-psql
-- change output column, function name & arguments 
CREATE OR REPLACE FUNCTION stm.stm__b2b_center_connect_for_facility_ids__for_range__at_day_level(facility_ids_filter text[],start_date_filter date, end_date_filter date) RETURNS 
TABLE(metric_id text ,facility_id text,date_type text,date_value timestamp,sum bigint,total bigint,_sample_wbn text,max bigint) AS

-- BOILER_PLATE_CODE
$func$
BEGIN
RETURN QUERY
EXECUTE format($execute$

-- typecast the output as per user expectation. The database returns metric column value in text only
-- if there's LIVE metric datamart, output of both LIVE and MATERIALIZED should be compatible so the user can do UNION 
select metric_id,facility_id ,date_type,date_value,sum::bigint,total::bigint,_sample_wbn text,max::bigint from crosstab(

$$
 SELECT metric_id,facility_id,date_type,date_value,metric_column_name,metric_column_value from (
 select metric_id,facility_id,date_type,date_value,
 metric_column_name,metric_column_value,
 row_number() over ( partition by metric_column_id order by created_at desc) as rnum
  from metrics.mat_metrics_per_client_per_facility 
-- Use the metric_id created in above
 where metric_registry_uuid = '4d5beeef-f1a3-4263-b044-695a307f2370'

-- Use filters passed from enduser to get needed metric only
 and (facility_id = any (%L) ) 
and start_date >= %L
start_date < %L

-- Getting latest version of that metric. Incase the same metric is calculated multiple times in the same time frame
 ) a where rnum = 1
$$

--Second argument crosstab function starts
, $$VALUES ('status'), ('total'),('_sample_wbn'),('max') $$) AS

-- Defining type
 ct(metric_id text, facility_id text, date_type text, date_value timestamp, 
status text, total text, _sample_wbn text, max text);

-- Format function ends here
$execute$

-- Passing filters of end-user to the query before running
,facility_ids_filter, start_date_filter, end_date_filter);

-- BOILER_PLATE_CODE
END;
$func$ LANGUAGE plpgsql VOLATILE;
```

#### Usage : To be queried from application code
```postgres-sql
select * from stm.stm__b2b_center_connect_for_facility_ids__for_range__at_day_level('{IN110051A1C}','2021-03-23','2021-03-28');
```

