---
title: "programmatic access and Integration"
date: 2019-03-26T08:47:11+01:00
draft: true
---
##### Guidelines for programmatic Integration with DMDB
1. Use only the user provided to you for your specific application. Even if you have any other credentials like developers credentials, some other application's credentials, etc.; dont use them. Ask for new credentials for each new application/dashboard.
1. Connection Timeout (300ms) -  Use this while making the connection to DB. If the connection fail within the timeout move further, log error (raise alarms). The default is unlimited time, so application can get stuck when DB is down if this is not specified.
    1. Retry continuously for 10 times for connection if you received ~ConnectionExhaust, ~DNSNotResolvedException, ~AddressUnavailable, ~ConnectionRefused etc. without any backoff. 
1. Query Timeout (500ms) - Use this while querying data from DB. Ideally the queries should run in around ~150ms so this is a much safer value. 
    1.  Retry Logic - If the query takes >500ms , then because of timeout you should get ~QueryTimeoutException. Handle this and retry the query now with a timeout of 
    **3000ms**. Retry only **once**.
1. General Retry in case of other exception - In case of exceptions like ~ConnectionExhaust, ~DBObjectNotAvailable, ~DBNotAvailable, ~DBShuttingDown which could be transient in nature,
use retry mechanism with a backoff. The configuration of this depends on your use-case and situations. The suggested values is 3 times retry with a increasing backoff starting from 1000ms.    
Please don't retry for every Exception(blanket) eg. ~SyntaxIssueException, ~PermissionException etc. because these are not transient and cannot be fixed without manual intervention.
1. If your application needs to fire multiple queries for a single page load, then start executing all queries in parallel. The database serves can serve of several concurrent requests at once.
    1. So when the dashboard page starts to open it fires all the queries for the selected facilities/group of facilities for a day/range of days.
       
       For example,
       
       if there are 5 metrics being shown on a page on the dashboard, and from DB th e respective query execution time is 120ms, 150ms, 30ms, 68ms, 190ms
       then, the total dashboard page loading time will be approx equal to  =  maximum query execution time among all queries - 190ms in this case + API compute time + network latency = ~220ms 

1. Always use datamart-db.delhivery.com endpoint for connecting to DB. This is a replica machine and all read queries from application/dashboard backend should be fired to this endpoint only.

1. For connection pooling, use connection pooling if possible. There are libraries available for pooling at application level in most languages.
Each user is alloted a 25 connections to start with. THis can be increased on case-to-case basis.
   
> To get credentials for a new dashboard/application backend please contact dg-tech@delhivery.com 
