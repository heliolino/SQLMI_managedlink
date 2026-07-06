# Adding a New Database to an Existing Managed Instance Link

## Overview

This guide covers the steps to add a **new database** to an environment where a Managed Instance Link is already configured and operational between `sqlsource01` (Azure SQL Managed Instance) and `sqltarget01` (SQL Server on Azure VM).

> **Key concept:** The Managed Instance Link supports **one database per link**. To replicate a new database, you must create a **new link** (a new Availability Group and Distributed Availability Group). However, you can reuse the existing certificates, endpoint, and trust configuration — those are instance-level resources, not database-level.

### What You Can Reuse

| Component | Reuse? | Notes |
|---|---|---|
| Certificates | ✅ Yes | Already exchanged between instances |
| Database mirroring endpoint | ✅ Yes | One endpoint per instance, shared across all links |
| Azure root CA certificates | ✅ Yes | Already imported on SQL Server |
| Trust configuration | ✅ Yes | Already established |
| Availability Group | ❌ No | New AG required per database |
| Distributed Availability Group | ❌ No | New DAG required per link |

---

## Prerequisites

Before proceeding, confirm the existing link is healthy:

```sql
-- Run on sqltarget01
-- Verify existing link is operational
SELECT
    ag.name AS ag_name,
    drs.synchronization_state_desc,
    drs.synchronization_health_desc
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_groups ag ON drs.group_id = ag.group_id;
```

Also verify the endpoint is running:

```sql
-- Run on sqltarget01
SELECT name, state_desc, connection_auth_desc, encryption_algorithm_desc
FROM sys.database_mirroring_endpoints;
```

> **Expected:** `state_desc = STARTED`. If the endpoint is not started, review the main guide [Step 3](SQLMI_managedlink.doc.eng.md#step-3--create-and-configure-the-database-mirroring-endpoint-on-sql-server).

---

## Step 1 — Prepare the New Database on the Source (`sqlsource01`)

Replace `<NewDatabaseName>` with the actual name of the new database throughout this guide.

### 1.1 — Verify the database exists and is in FULL recovery model

```sql
-- Run on sqlsource01 (Managed Instance)
SELECT name, recovery_model_desc, state_desc
FROM sys.databases
WHERE name = '<NewDatabaseName>';
```

> **Expected:** `recovery_model_desc = FULL`, `state_desc = ONLINE`

### 1.2 — Check the database size (for seeding time estimation)

```sql
-- Run on sqlsource01
SELECT
    DB_NAME(database_id) AS database_name,
    type_desc,
    CAST(size * 8.0 / 1024 AS DECIMAL(10,2)) AS size_mb,
    CAST(size * 8.0 / 1024 / 1024 AS DECIMAL(10,2)) AS size_gb
FROM sys.master_files
WHERE DB_NAME(database_id) = '<NewDatabaseName>';
```

---

## Step 2 — Create a New Availability Group on SQL Server (`sqltarget01`)

Since each link requires its own AG, create a new one specifically for this database.

### 2.1 — Choose naming convention

Use a consistent naming pattern that reflects the database name:

| Component | Naming Pattern | Example (for `dborders`) |
|---|---|---|
| Availability Group (SQL Server) | `AG_<dbname>` | `AG_dborders` |
| Availability Group (MI) | `AG_<dbname>_MI` | `AG_dborders_MI` |
| Distributed Availability Group | `DAG_<dbname>` | `DAG_dborders` |

### 2.2 — Create the Availability Group

```sql
-- Run on sqltarget01
USE master;

CREATE AVAILABILITY GROUP [AG_<NewDatabaseName>]
WITH (CLUSTER_TYPE = NONE)
    FOR DATABASE [<NewDatabaseName>]
    REPLICA ON
        N'sqltarget01' WITH
        (
            ENDPOINT_URL = 'TCP://<sqltarget01_IP>:5022',
            AVAILABILITY_MODE = SYNCHRONOUS_COMMIT,
            FAILOVER_MODE = MANUAL,
            SEEDING_MODE = AUTOMATIC
        );
GO
```

> **Note:** Use the same `<sqltarget01_IP>` and port (5022) as your existing link. The endpoint is shared.

### 2.3 — Verify the AG was created

```sql
-- Run on sqltarget01
SELECT
    ag.name AS ag_name,
    ar.replica_server_name,
    ar.endpoint_url,
    ar.availability_mode_desc,
    ar.seeding_mode_desc
FROM sys.availability_groups ag
JOIN sys.availability_replicas ar ON ag.group_id = ar.group_id
ORDER BY ag.name;
```

---

## Step 3 — Create a New Distributed Availability Group on SQL Server

```sql
-- Run on sqltarget01
USE master;

CREATE AVAILABILITY GROUP [DAG_<NewDatabaseName>]
WITH (DISTRIBUTED)
    AVAILABILITY GROUP ON
    N'AG_<NewDatabaseName>' WITH
    (
        LISTENER_URL = 'TCP://<sqltarget01_IP>:5022',
        AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
        FAILOVER_MODE = MANUAL,
        SEEDING_MODE = AUTOMATIC,
        SESSION_TIMEOUT = 20
    ),
    N'AG_<NewDatabaseName>_MI' WITH
    (
        LISTENER_URL = 'tcp://<ManagedInstanceFQDN>:5022;Server=[sqlsource01]',
        AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
        FAILOVER_MODE = MANUAL,
        SEEDING_MODE = AUTOMATIC
    );
GO
```

> **IMPORTANT:** Replace `<ManagedInstanceFQDN>` with the actual FQDN of your Managed Instance (e.g., `sqlsource01.abc123def456.database.windows.net`). Use the same FQDN as in your existing link.

---

## Step 4 — Create the Link on the Managed Instance (Azure)

Use **Azure Cloud Shell (PowerShell)** to create the new link:

```powershell
# Variables
$ManagedInstanceName = "sqlsource01"
$ResourceGroup       = (Get-AzSqlInstance -InstanceName $ManagedInstanceName).ResourceGroupName
$DAGName             = "DAG_<NewDatabaseName>"
$AGNameOnSQLServer   = "AG_<NewDatabaseName>"
$AGNameOnMI          = "AG_<NewDatabaseName>_MI"
$SQLServerIP         = "<sqltarget01_IP>"

# Create the new link
New-AzSqlInstanceLink `
    -ResourceGroupName $ResourceGroup `
    -InstanceName $ManagedInstanceName `
    -Name $DAGName `
    -PrimaryAvailabilityGroupName $AGNameOnMI `
    -SecondaryAvailabilityGroupName $AGNameOnSQLServer `
    -TargetDatabase "<NewDatabaseName>" `
    -SourceEndpoint "TCP://${SQLServerIP}:5022"
```

**Alternative — Azure CLI:**

```bash
az sql mi link create \
    --resource-group "<ResourceGroup>" \
    --instance-name "sqlsource01" \
    --name "DAG_<NewDatabaseName>" \
    --primary-ag "AG_<NewDatabaseName>_MI" \
    --secondary-ag "AG_<NewDatabaseName>" \
    --target-database "<NewDatabaseName>" \
    --source-endpoint "TCP://<sqltarget01_IP>:5022"
```

---

## Step 5 — Monitor the New Link Seeding

The seeding process will start automatically. Monitor it the same way as the original database:

### 5.1 — Check automatic seeding progress

```sql
-- Run on sqltarget01
SELECT
    ag.name AS ag_name,
    asd.database_name,
    asd.start_time,
    asd.current_state,
    asd.performed_seeding,
    asd.failure_state_desc,
    asd.error_code,
    asd.number_of_attempts
FROM sys.dm_hadr_automatic_seeding AS asd
JOIN sys.availability_groups AS ag
    ON asd.ag_id = ag.group_id
WHERE asd.database_name = '<NewDatabaseName>';
```

### 5.2 — Check restore progress during seeding

```sql
-- Run on sqltarget01
SELECT
    r.session_id,
    r.command,
    r.status,
    r.percent_complete,
    r.estimated_completion_time / 1000 / 60 AS est_minutes_remaining,
    r.total_elapsed_time / 1000 / 60 AS elapsed_minutes,
    DB_NAME(r.database_id) AS database_name
FROM sys.dm_exec_requests r
WHERE r.command LIKE '%RESTORE%'
   OR r.command LIKE '%BACKUP%';
```

### 5.3 — Monitor replication state after seeding completes

```sql
-- Run on sqltarget01
-- Monitor all linked databases at once
SELECT
    ag.name AS ag_name,
    DB_NAME(drs.database_id) AS database_name,
    drs.synchronization_state_desc,
    drs.synchronization_health_desc,
    drs.log_send_queue_size AS log_send_queue_kb,
    drs.redo_queue_size AS redo_queue_kb,
    drs.redo_rate AS redo_rate_kb_sec,
    drs.last_commit_time,
    DATEDIFF(SECOND, drs.last_redone_time, drs.last_commit_time) AS replication_lag_seconds
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_groups ag ON drs.group_id = ag.group_id
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id
ORDER BY ag.name;
```

---

## Step 6 — Verify All Links Are Healthy

After the new link is established, run a comprehensive check across all linked databases:

```sql
-- Run on sqltarget01
-- Summary of all Availability Groups and their databases
SELECT
    ag.name AS ag_name,
    DB_NAME(drs.database_id) AS database_name,
    ar.replica_server_name,
    drs.is_primary_replica,
    drs.synchronization_state_desc AS sync_state,
    drs.synchronization_health_desc AS sync_health,
    drs.log_send_queue_size AS log_queue_kb,
    drs.redo_queue_size AS redo_queue_kb
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_groups ag ON drs.group_id = ag.group_id
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id
ORDER BY ag.name, ar.replica_server_name;
```

```sql
-- List all availability groups on the server
SELECT name, is_distributed
FROM sys.availability_groups
ORDER BY name;
```

---

## Quick Reference — Adding Each Subsequent Database

Once comfortable with the process, here is a condensed checklist for each additional database:

1. ✅ Verify database is in FULL recovery model on source
2. ✅ Create `AG_<dbname>` on `sqltarget01`
3. ✅ Create `DAG_<dbname>` on `sqltarget01` (distributed AG)
4. ✅ Create link via PowerShell/CLI on Azure (referencing `AG_<dbname>_MI`)
5. ✅ Monitor seeding via `sys.dm_hadr_automatic_seeding`
6. ✅ Verify sync state via `sys.dm_hadr_database_replica_states`

> **Remember:** No need to recreate certificates, endpoints, or trust configuration. Those are reused automatically.

---

## Troubleshooting

### Error 1475 — Need a new backup chain

If you see error 1475 on the source, create a full backup without `COPY_ONLY`:

```sql
-- Run on sqlsource01
BACKUP DATABASE [<NewDatabaseName>] TO DISK = N'NUL';
```

### Seeding fails repeatedly

```sql
-- Check seeding failure details
SELECT
    ag.name,
    asd.database_name,
    asd.failure_state_desc,
    asd.error_code,
    asd.number_of_attempts
FROM sys.dm_hadr_automatic_seeding asd
JOIN sys.availability_groups ag ON asd.ag_id = ag.group_id
WHERE asd.database_name = '<NewDatabaseName>';
```

Common causes:
- **Insufficient disk space** on the target for the new database
- **Collation mismatch** between instances
- **Network timeout** — check NSG rules and port 5022 connectivity

### AG creation fails with "database not found"

The database must exist on the server where the AG is created. If the source is the MI and the AG is created on SQL Server, the database will be seeded automatically — it does **not** need to pre-exist on SQL Server.

---

## References

- [Main configuration guide](SQLMI_managedlink.doc.eng.md)
- [Managed Instance Link — One database per link](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/managed-instance-link-feature-overview)
- [Configure Link with Scripts](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/managed-instance-link-configure-how-to-scripts)

---

*Document created: 2026-06-08*
*Last updated: 2026-07-06 16:05 CST*
