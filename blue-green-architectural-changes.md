# 🎨 Blue/Green Deployment for Architectural Changes: Minimizing downtime: The PITR Approach

Modifying the fundamental architecture of your database—such as downsizing hardware, changing networking topology, or migrating to a new storage technology—often requires database recreation, which can lead to extended service outages.

This guide details a structured Blue/Green deployment strategy utilizing Azure Database for PostgreSQL's Point-in-Time Restore (PITR) feature to facilitate these critical architectural shifts with near-zero downtime. 
This method is ideal when in-place modifications are impossible, allowing you to provision and validate a new environment (Green) before instantly cutting over from the old (Blue) environment.

## ⚙️ I. Setup and Provisioning 

### Step 0 - Role Privileges (Source Azure Database for PostgreSQL)

The user role designated for the migration must have the necessary permissions to manage replication and slots.
```sql
ALTER ROLE demo WITH REPLICATION;
GRANT azure_pg_admin TO demo;
```
> The user role designated for the migration must have the necessary permissions to manage replication and slots. The "demo" user was created at server creation. In case you want to create a dedicated replication user, find my guide [here](https://github.com/berenguel/bi-directional-replication-in-Flexible-Server/blob/main/configuring_replication_user.sql)

### Step 1 - Check Prerequisites for Logical Replication (Source Azure Database for PostgreSQL)
Set these server parameters to at least the minimum recommended values shown below to enable and support the features required for logical replication.

```sql
wal_level=logical
max_worker_processes=16
max_replication_slots=10
max_wal_senders=10
track_commit_timestamp=on
```

### Step 2 - Ensure tables are ready for logical replication

For PostgreSQL to accurately track row-level changes (updates, deletes), every table you plan to replicate must have a unique identifier.

Tables *must have* one of the following:
- a Primary Key, or
- a unique index, or
- Replica identity full (less efficient)

## ➡️ II. Migration and Synchronization (Hybrid Data Transfer)

This section details the core process of establishing the synchronization mechanism before creating the new environment via PITR.

### Step 3 - Set Up Logical Replication (Source Azure Database for PostgreSQL)

A Publication defines the logical grouping of tables on the source server that will be replicated

```sql
create publication logical_mig01;
alter publication logical_mig01 add table dummy_test;

SELECT pg_create_logical_replication_slot('logical_mig01', 'pgoutput');
```
> The Safety Net: The slot created in step 4 immediately begins tracking all changes in the WAL. This guarantees that all transactions occurring from this moment forward are recorded and preserved, preventing data loss during the initial dump/restore process

Step 4 - Choose the Restore Point
Determine the latest possible point in time (PIT) to restore the data from the source server. This Point in Time must be, necessarily, after you created the replication slot in Step 3.


### Step 5 - Provision the New Target Environment (Green Environment) via PITR

Go to the Azure Portal & trigger a Point in Time Restore. Alternatively, use Azure CLI.

Action: Provision the new Azure Database for PostgreSQL instance by restoring a snapshot of the Blue server's data state.

> Key Concept: You are provisioning the Green environment based on the Blue server's backup capabilities. This Green environment will initially be a copy of the Blue environment's data state at a specific point in time.

### Step 6 - Create the target server as subscriber & Advance Replication Origin

This is the crucial step that connects the target (subscriber) to the source and manually tells the target where in the WAL log stream to begin reading changes, skipping the data already restored.

```sql
-- On the target
CREATE SUBSCRIPTION logical_sub01 CONNECTION 'host=pgprimary6666.postgres.database.azure.com port=5432 dbname=postgres user=yyyy password=zzzzzzz' PUBLICATION logical_mig01
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

> Manual Synchronization: By advancing the origin, you instruct the subscriber to ignore all WAL records before the specified restart_lsn. This aligns the subscription with the exact point in time when the initial slot was created, effectively forcing the Green environment to catch up on everything that happened since the slot's creation.

Step 7 - Enable the target server as a subscriber of the source server

With the target server populated and the replication origin advanced, you can start the synchronization.

```sql
ALTER SUBSCRIPTION logical_sub01 ENABLE;
```
> Result: The target server now starts consuming the WAL entries from the source, rapidly closing the gap on all transactions that occurred between the slot creation (Step 3) and the completion of the PITR (Step 5).

## ✅ III. Post-Migration & Cutover (The Blue/Green Switch)

Step 8 - Test Replication works
Confirm that the synchronization is working by inserting a record on the source and immediately verifying its presence on the target.
```
--@Source Server
insert into dummy_test values (1, 'London', 'Test', 66, 77, now());
```
```
--@Target Server
select * from dummy_test;
```

Step 9 - The Cutover
Result: The target server now starts consuming the WAL entries from the source, rapidly closing the gap on all transactions that occurred between the slot creation (Step 3) and the completion of the PITR (Step 5).
