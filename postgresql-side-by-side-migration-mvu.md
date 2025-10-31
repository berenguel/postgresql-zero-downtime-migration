# ⏳ Minimizing Downtime: Azure PostgreSQL Major Version Side-by-Side Upgrade

Upgrading the major version of your Azure Database for PostgreSQL instance can sometimes require **unacceptable levels of downtime**.

This guide provides a structured, **side-by-side migration path** to drastically reduce service interruption. This method is crucial when architectural constraints—such as extensive replica sets, intricate network configurations, or specific extension requirements—make a standard in-place upgrade unfeasible due to time limitations.


## Pre Requisites

### Step 0 - Role Privileges (Source & Target Azure Database for PostgreSQL servers)

```sql
ALTER ROLE demo WITH REPLICATION;
GRANT azure_pg_admin TO demo;
```


### Step 1 - Check Prerequisites for Logical Replication (Source & Target Azure Database for PostgreSQL servers)
Enable the below server parameters - note that that this is the minimal recommendation
```sql
wal_level=logical
max_worker_processes=16
max_replication_slots=10
max_wal_senders=10
track_commit_timestamp=on
```

### Step 2 - Check that all tables are ready for logical replication

Tables *must have* one of the following:
- a Primary Key, or
- a unique index, or
- Replica identity full

## Let's start the Logical Replication & Data Copy

### Step 3 - Set Up Logical Replication (Source Server)

```sql
create publication logical_mig01;

alter publication logical_mig01 add table conditions;

SELECT pg_create_logical_replication_slot('logical_mig01', 'pgoutput');
```

### Step 4 - Generate Dump of Schema + Data (recommended: Azure VM)

```
pg_dump -U demo -W \
-h pg13blogtest.postgres.database.azure.com -p 5432 -Fc -v \
-f dump.bak postgres \
	-N pg_catalog \
	-N cron \
	-N information_schema
```


### Step 5 - Restore the data dump into the Target (recommended: Azure VM)
```
pg_restore -U demo -W  \
-h pg17blogtest.postgres.database.azure.com -p 5432 --no-owner \
-Fc -v -d postgres dump.bak --no-acl
```

### Step 6 - Create the target server as subscriber & Advance Replication Origin

```sql
-- On the target
CREATE SUBSCRIPTION logical_sub01 CONNECTION 'host=pg13blogtest.postgres.database.azure.com port=5432 dbname=postgres user=yyyy password=zzzzzzz' PUBLICATION logical_mig01
WITH (
	copy_data = false,
	create_slot = false,
	enabled = false,
	slot_name = 'logical_mig01'
);

SELECT roident, roname FROM pg_replication_origin;
-- Example output:
--  roident |  roname
-- ---------+----------
--        1 | pg_25910

-- On the source:
SELECT slot_name, restart_lsn FROM pg_replication_slots WHERE slot_name = 'logical_mig01';
-- Example output:
--  slot_name    | restart_lsn
-- -------------+-------------
--  logical_mig01 | 20/9300A690

-- On the target (replace 'pg_25910' and '20/9300A690' with your actual values):
SELECT pg_replication_origin_advance('pg_25910', '20/9300A690');
```

Step 7 - Enable the target server as a subscriver of source server
```sql
ALTER SUBSCRIPTION logical_sub01 ENABLE;
```
## Let's test that the replication works
Step 8 - Test Replication 
--@Source Server
```sql
insert into dummy_test values (1, 'London', 'Test', 66, 77, now());
select * from dummy_test;
```