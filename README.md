# Overall Architecture 

<img width="1098" alt="image" src="https://github.com/YCat33/cdc_streaming_ingest/assets/115039992/89d23ea3-dec8-400d-b8be-479339cd8157">



## Pre-Requisites 

  - PostgreSQL Database
  - Starburst Galaxy Domain
  - Available AWS S3 Bucket
  - Docker installed and configured

### PostgreSQL AWS RDS Instance Preperation

  - Set the rds.logical_replication to 1
  - Set shared_preload_libraries to pg_stat_statements,pg_hint_plan,pgaudit
ssl=0
  - Verify that the wal_level parameter is set to logical by running Show wal_level as the database RDS master user. 
  - Set the Debezium plugin.name parameter to pgoutput.
  - Configure new user with replication access
          
          CREATE USER <username> password '<password>';
          GRANT CONNECT ON DATABASE postgres  TO <username>;
          GRANT USAGE ON SCHEMA burst_bank  TO <username>;
          GRANT all privileges on table <table> to <username>;
          GRANT USAGE ON SCHEMA burst_bank TO <username>;
          GRANT SELECT ON ALL TABLES IN SCHEMA <schema_name> TO <username>;
          ALTER DEFAULT PRIVILEGES IN SCHEMA burst_bank GRANT SELECT ON TABLES TO <username>;
          GRANT rds_replication to <username>;

## Tutorial

**Troubleshooting: Once the container is up, you should be able to review the logs to understand what is occurring within each connector at (localhost:3030/logs/connect-distributed.log)

1. `docker-compose up kafka-cluster -d`

     This Docker Compose file defines a Kafka-cluster service that sets up a Kafka cluster and other related components. As part of this process, we are leveraging the "zachmartinkr/fast-data-dev" which provides us with the ability to utilize the Landoop UI to configure the Kafka connectors (along with a few other services (e.g. Zookeeper/SchemaRegistry/etc.). Once the docker container is up and running, you should be able to view the Landoop UI (at localhost:3030) to better understand Connectors, Topics and even view detailed Log information (see screenshot below for an example).

   <img width="1134" alt="image" src="https://github.com/YCat33/cdc_streaming_ingest/assets/115039992/220f803a-829f-4bac-83cf-774450d88a56">

         
   
2. Now that the container is up and running we can configure the PostgreSQL Source, by running the below command:
  `curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @debezium.json`


3. Next, we can configure the s3 Sink by executing the following:
   `curl -i -X POST -H "Accept:application/json" -H  "Content-Type:application/json" http://localhost:8083/connectors/ -d @s3_sink.json`

4. Now that the connection has been setup, you can start making changes to the underlying table(s) in Postgres and you should see data start to populate within in s3.

<img width="1193" alt="image" src="https://github.com/YCat33/cdc_streaming_ingest/assets/115039992/f66829ab-425f-41c9-ad20-495e62ca8df2">

5. From there you can leverage schema discovery within Starburst Galaxy to create the "changes" table for you (which will look something like the below).

<img width="1279" alt="image" src="https://github.com/YCat33/cdc_streaming_ingest/assets/115039992/f905dfd8-0667-4277-9c12-16a30fab19f3">

6. Next you can create the "consume" table in s3, that your end-users/applications will query from:

```
create table <table_name> (
     custkey VARCHAR(3),
    first_name VARCHAR(50),
    city VARCHAR(50),
    state VARCHAR(2),
    fico INT,
    registration_date DATE,
    ts_ms BIGINT
) WITH (type='iceberg',
      format='parquet',
      location='<s3_bucket_location>');
```
7. Finally you can leverage the following MERGE statement to update records in the "consume" table from your "changes" table that is getting populated with records from Kafka.


```
----Merge Statement
MERGE INTO <consume_table> as a
USING (
    SELECT
        coalesce(before.custkey, after.custkey) as custkey,
        after.first_name,
        after.city,
        after.state,
        after.fico,
        after.registration_date,
        ts_ms,
        op,
        ROW_NUMBER() OVER (PARTITION BY coalesce(before.custkey, after.custkey) ORDER BY ts_ms DESC) as row_num,
        source.lsn
    FROM
        <staging_table>
) latest_changes
ON a.custkey = latest_changes.custkey AND latest_changes.row_num = 1
WHEN MATCHED and latest_changes.op = 'd' THEN DELETE
WHEN MATCHED AND a.ts_ms < latest_changes.ts_ms THEN UPDATE
    SET
        custkey = latest_changes.custkey,
        first_name = latest_changes.first_name,
        city = latest_changes.city,
        state = latest_changes.state,
        fico = latest_changes.fico,
        registration_date = latest_changes.registration_date,
        ts_ms = latest_changes.ts_ms
WHEN NOT MATCHED and latest_changes.row_num = 1 and latest_changes.op != 'd' THEN
    INSERT (custkey, first_name, city, state, fico, registration_date, ts_ms)
    VALUES (
        latest_changes.custkey,
        latest_changes.first_name,
        latest_changes.city,
        latest_changes.state,
        latest_changes.fico,
        latest_changes.registration_date,
        latest_changes.ts_ms
    );
```

**The above is just an example merge statement with a few key points I want to highlight:

1. Source Data Transformation:

In the USING clause, a subquery is employed to transform the source data.
The coalesce(before.custkey, after.custkey) ensures that the custkey is taken from either the before or after depending on which is not null. This is because if a new record is inserted into the table (op = c) then the before column will be blank and only the after column will be populated. Conversely, if a record is deleted from the table (op = d) then only the before column will be populated and the after column will be blank. Lastly, the row_number() window function ensures that we are only working with the most recently updated change for a given custkey and not dealing with multiple different rows. 

2. Matching and Actions:

The ON a.custkey = latest_changes.custkey AND latest_changes.row_num = 1 clause ensures only the latest change (based on ts_ms) for each custkey is considered.
WHEN MATCHED and latest_changes.op = 'd' THEN DELETE: If the latest change in the source indicates a delete (op = 'd'), then delete the corresponding row in the target.
WHEN MATCHED AND a.ts_ms < latest_changes.ts_ms THEN UPDATE: If there is a match, and the timestamp in the target is older than the timestamp in the source, update the target row with values from the source.

3. Handling Inserts:

The WHEN NOT MATCHED and latest_changes.row_num = 1 and latest_changes.op != 'd' condition, ensures that if there is no match in the target, and the latest change in the source is not a delete, then insert a new row into the target with values from the source.

*If interested, you can learn more about the merge functionality ([here](https://www.starburst.io/blog/apache-iceberg-dml-update-delete-merge-maintenance-in-trino/))
