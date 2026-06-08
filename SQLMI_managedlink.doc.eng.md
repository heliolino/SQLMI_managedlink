# Azure SQL Managed Instance Link — Configuration Guide via T-SQL

## Overview

This document describes the steps to configure a **Managed Instance Link** between two SQL environments using T-SQL:

| Component | Details |
|---|---|
| **Source (Primary)** | Azure SQL Managed Instance — server `sqlsource01`, database `dbtest` |
| **Target (Secondary)** | SQL Server on Azure VM — server `sqltarget01` |
| **Network** | Both on the same VNET in Azure |
| **Endpoint Port** | TCP 5022 (database mirroring) |
| **Technology** | Distributed Availability Group (Always On) |

> **Note:** The Managed Instance Link uses Distributed Availability Groups technology to replicate data in near-real time between the two instances.

---

## Prerequisites

### SQL Server on VM Requirements (`sqltarget01`)

- SQL Server 2022 (Enterprise or Developer) with the latest service update installed
- Always On Availability Groups **enabled** in SQL Server Configuration Manager
- SQL Server service restarted after enabling Always On
- `sysadmin` access on the instance

### Azure SQL Managed Instance Requirements (`sqlsource01`)

- Business Critical or General Purpose tier (vCore)
- Update policy configured for **SQL Server 2022** (required for bidirectional failover)
- Permissions: member of the **SQL Managed Instance Contributor** role or equivalent custom permissions

### Network Requirements

- Both servers on the same VNET
- Port **TCP 5022** open between instances (NSG rules)
- Functional DNS resolution between instances — verify with `nslookup`
- The Managed Instance FQDN must be resolvable from the SQL Server VM

### Required Tools

- Azure Cloud Shell (PowerShell) or Az.SQL 6.0.0+ / Azure CLI 2.67.0+
- SQL Server Management Studio (SSMS) 19.x or higher
- Access to the Azure Portal

---

## Step 1 — Prepare the Database on the Source (`sqlsource01`)

The `dbtest` database on the Managed Instance needs to be in **FULL recovery model**. Since MI manages backups automatically, this requirement is usually already met.

Verify the recovery model:

```sql
-- Run on sqlsource01 (Managed Instance)
SELECT name, recovery_model_desc
FROM sys.databases
WHERE name = 'dbtest';
```

> **Expected result:** `recovery_model_desc = FULL`

MI already performs automatic backups, so no manual backup is needed for the initial link seeding.

---

## Step 2 — Establish Trust Between Instances (Certificates)

The Managed Instance Link uses **certificate-based authentication** for the database mirroring endpoint. Certificates must be exchanged between the two instances.

### 2.1 — Create Master Key and Certificate on SQL Server (`sqltarget01`)

```sql
-- Run on sqltarget01 (SQL Server on VM)
USE master;
GO

-- Create master key (if it doesn't exist)
IF NOT EXISTS (SELECT * FROM sys.symmetric_keys WHERE symmetric_key_id = 101)
BEGIN
    CREATE MASTER KEY ENCRYPTION BY PASSWORD = '<StrongPassword_MasterKey>';
    PRINT 'Master key created successfully.';
END
ELSE
    PRINT 'Master key already exists.';
GO

-- Create certificate for the link endpoint
-- IMPORTANT: Adjust the expiration date as needed
DECLARE @cert_expiry_date VARCHAR(20) = '2027-06-30';
DECLARE @cert_name NVARCHAR(MAX) = N'Cert_sqltarget01_endpoint';
DECLARE @cert_subject NVARCHAR(MAX) = N'Certificate for sqltarget01 managed link endpoint';

IF NOT EXISTS (SELECT name FROM sys.certificates WHERE name = @cert_name)
BEGIN
    CREATE CERTIFICATE [Cert_sqltarget01_endpoint]
        WITH SUBJECT = 'Certificate for sqltarget01 managed link endpoint',
        EXPIRY_DATE = '2027-06-30';
    PRINT 'Certificate created successfully.';
END
ELSE
    PRINT 'Certificate already exists.';
GO
```

### 2.2 — Export the SQL Server Certificate Public Key

```sql
-- Run on sqltarget01
USE master;
GO

DECLARE @cert_name NVARCHAR(MAX) = N'Cert_sqltarget01_endpoint';
DECLARE @PUBLICKEYENC VARBINARY(MAX) = CERTENCODED(CERT_ID(@cert_name));

SELECT @cert_name AS SQLServerCertName;
SELECT @PUBLICKEYENC AS SQLServerPublicKey;
```

> **Action:** Copy the values of `SQLServerCertName` and `SQLServerPublicKey` — they will be used in the next step.

### 2.3 — Import the SQL Server Certificate to Azure (PowerShell)

Run in **Azure Cloud Shell (PowerShell)**:

```powershell
# Variables — adjust according to your environment
$SubscriptionID     = "<YourSubscriptionID>"
$ManagedInstanceName = "sqlsource01"
$CertificateName     = "Cert_sqltarget01_endpoint"
$PublicKeyEncoded    = "<PasteThePublicKeyFromPreviousStep>"  # Starts with 0x...

# Login and subscription selection
if ((Get-AzContext) -eq $null) { Login-AzAccount }
Select-AzSubscription -SubscriptionName $SubscriptionID

# Find resource group
$ResourceGroup = (Get-AzSqlInstance -InstanceName $ManagedInstanceName).ResourceGroupName

# Upload the SQL Server public certificate to Azure
New-AzSqlInstanceServerTrustCertificate `
    -ResourceGroupName $ResourceGroup `
    -InstanceName $ManagedInstanceName `
    -Name $CertificateName `
    -PublicKey $PublicKeyEncoded
```

### 2.4 — Export the Managed Instance Certificate and Import to SQL Server

**In Azure Cloud Shell:**

```powershell
$ManagedInstanceName = "sqlsource01"
$ResourceGroup = (Get-AzSqlInstance -InstanceName $ManagedInstanceName).ResourceGroupName

# Get the MI certificate public key
Get-AzSqlInstanceEndpointCertificate `
    -ResourceGroupName $ResourceGroup `
    -InstanceName $ManagedInstanceName `
    -EndpointType "DATABASE_MIRRORING" | Out-String
```

> **Action:** Copy the full `PublicKey` value (starts with `0x`).

**On SQL Server (`sqltarget01`):**

```sql
-- Run on sqltarget01
-- Replace <ManagedInstanceFQDN> with the MI FQDN
-- Example: sqlsource01.abc123def456.database.windows.net
-- Replace <PublicKey> with the key obtained in the previous step
USE master;
CREATE CERTIFICATE [sqlsource01.abc123def456.database.windows.net]
FROM BINARY = <PublicKey>;
GO
```

> **IMPORTANT:** The certificate name **must be exactly** the Managed Instance FQDN. Do not use custom names.

### 2.5 — Import Azure Root CA Certificates to SQL Server

Download the Azure root CA certificates and import them to SQL Server. At minimum, import:

1. **DigiCert Global Root G2**
2. **Microsoft RSA Root Certificate Authority 2017**

> **Recommendation:** For long-running links, import all 7 certificates listed in [Azure Root Certificate Authorities](https://learn.microsoft.com/en-us/azure/security/fundamentals/azure-ca-details#root-certificate-authorities).

Download the `.crt` files and copy them to the SQL Server, for example in `C:\Certs\`.

```sql
-- Run on sqltarget01
-- Repeat for each root CA certificate
USE master;

-- DigiCert Global Root G2
IF NOT EXISTS (SELECT name FROM sys.certificates WHERE name = N'DigiCert Global Root G2')
BEGIN
    CREATE CERTIFICATE [DigiCert Global Root G2]
    FROM FILE = 'C:\Certs\DigiCertGlobalRootG2.crt';

    DECLARE @CERTID_DG INT = CERT_ID('DigiCert Global Root G2');
    EXEC sp_certificate_add_issuer @CERTID_DG, N'*.database.windows.net';
    PRINT 'DigiCert Global Root G2 imported.';
END
GO

-- Microsoft RSA Root Certificate Authority 2017
IF NOT EXISTS (SELECT name FROM sys.certificates WHERE name = N'Microsoft RSA Root Certificate Authority 2017')
BEGIN
    CREATE CERTIFICATE [Microsoft RSA Root Certificate Authority 2017]
    FROM FILE = 'C:\Certs\MicrosoftRSARootCertificateAuthority2017.crt';

    DECLARE @CERTID_MS INT = CERT_ID('Microsoft RSA Root Certificate Authority 2017');
    EXEC sp_certificate_add_issuer @CERTID_MS, N'*.database.windows.net';
    PRINT 'Microsoft RSA Root CA 2017 imported.';
END
GO
```

### 2.6 — Verify All Certificates

```sql
-- Run on sqltarget01
USE master;
SELECT name, subject, expiry_date, pvt_key_encryption_type_desc
FROM sys.certificates
ORDER BY expiry_date;
```

> **Validation:** You should see at least 3 certificates: the sqltarget01 certificate, the MI FQDN certificate, and the Azure root CAs.

---

## Step 3 — Create and Configure the Database Mirroring Endpoint on SQL Server

### 3.1 — Check if an endpoint already exists

```sql
-- Run on sqltarget01
SELECT name, type_desc, state_desc, role_desc,
       connection_auth_desc, is_encryption_enabled, encryption_algorithm_desc
FROM sys.database_mirroring_endpoints
WHERE type_desc = 'DATABASE_MIRRORING';
```

### 3.2 — Create the endpoint (if it doesn't exist)

```sql
-- Run on sqltarget01
USE master;
CREATE ENDPOINT database_mirroring_endpoint
    STATE = STARTED
    AS TCP (LISTENER_PORT = 5022, LISTENER_IP = ALL)
    FOR DATABASE_MIRRORING (
        ROLE = ALL,
        AUTHENTICATION = CERTIFICATE [Cert_sqltarget01_endpoint],
        ENCRYPTION = REQUIRED ALGORITHM AES
    );
GO
```

### 3.3 — Alter an existing endpoint (if necessary)

If an endpoint with Windows authentication already exists, add certificate support:

```sql
-- Run on sqltarget01
USE master;
ALTER ENDPOINT [<ExistingEndpointName>]
    STATE = STARTED
    AS TCP (LISTENER_PORT = 5022, LISTENER_IP = ALL)
    FOR DATABASE_MIRRORING (
        ROLE = ALL,
        AUTHENTICATION = WINDOWS NEGOTIATE CERTIFICATE [Cert_sqltarget01_endpoint],
        ENCRYPTION = REQUIRED ALGORITHM AES
    );
GO
```

### 3.4 — Validate the endpoint

```sql
-- Run on sqltarget01
SELECT name, type_desc, state_desc, role_desc,
       connection_auth_desc, is_encryption_enabled, encryption_algorithm_desc
FROM sys.database_mirroring_endpoints;
```

> **Expected result:** `state_desc = STARTED`, `connection_auth_desc = CERTIFICATE`, `encryption_algorithm_desc = AES`

---

## Step 4 — Create Availability Group on SQL Server (`sqltarget01`)

> **Note:** Since the source is the Managed Instance (`sqlsource01`), the AG is created on the SQL Server (target) that will receive the replica.

### 4.1 — Get the SQL Server name

```sql
-- Run on sqltarget01
SELECT @@SERVERNAME AS SQLServerName;
```

### 4.2 — Create the Availability Group

```sql
-- Run on sqltarget01
USE master;
CREATE AVAILABILITY GROUP [AG_dbtest]
WITH (CLUSTER_TYPE = NONE)
    FOR DATABASE [dbtest]
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

> **Note:** Replace `<sqltarget01_IP>` with the SQL Server VM IP address or a resolvable hostname on the VNET.

### 4.3 — Create the Distributed Availability Group

```sql
-- Run on sqltarget01
-- Replace values according to your environment
USE master;
CREATE AVAILABILITY GROUP [DAG_dbtest]
WITH (DISTRIBUTED)
    AVAILABILITY GROUP ON
    N'AG_dbtest' WITH
    (
        LISTENER_URL = 'TCP://<sqltarget01_IP>:5022',
        AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
        FAILOVER_MODE = MANUAL,
        SEEDING_MODE = AUTOMATIC,
        SESSION_TIMEOUT = 20
    ),
    N'AG_dbtest_MI' WITH
    (
        LISTENER_URL = 'tcp://sqlsource01.abc123def456.database.windows.net:5022;Server=[sqlsource01]',
        AVAILABILITY_MODE = ASYNCHRONOUS_COMMIT,
        FAILOVER_MODE = MANUAL,
        SEEDING_MODE = AUTOMATIC
    );
GO
```

> **IMPORTANT:** Replace the FQDN `sqlsource01.abc123def456.database.windows.net` with your actual Managed Instance FQDN (visible in Azure Portal > Overview > Host name).

---

## Step 5 — Create the Link on the Managed Instance (Azure)

Use **Azure Cloud Shell (PowerShell)** to create the link from the Managed Instance side:

```powershell
# Variables
$ManagedInstanceName = "sqlsource01"
$ResourceGroup       = (Get-AzSqlInstance -InstanceName $ManagedInstanceName).ResourceGroupName
$DAGName             = "DAG_dbtest"
$AGNameOnSQLServer   = "AG_dbtest"
$AGNameOnMI          = "AG_dbtest_MI"
$SQLServerIP         = "<sqltarget01_IP>"

# Create the link
New-AzSqlInstanceLink `
    -ResourceGroupName $ResourceGroup `
    -InstanceName $ManagedInstanceName `
    -Name $DAGName `
    -PrimaryAvailabilityGroupName $AGNameOnMI `
    -SecondaryAvailabilityGroupName $AGNameOnSQLServer `
    -TargetDatabase "dbtest" `
    -SourceEndpoint "TCP://${SQLServerIP}:5022"
```

> **Alternative:** You can also create the link via SSMS (wizard) or Azure CLI (`az sql mi link create`).

---

## Step 6 — Verify Link Status

### 6.1 — On SQL Server (`sqltarget01`)

```sql
-- Check Distributed AG state
SELECT
    ag.name AS ag_name,
    ar.replica_server_name,
    ar.availability_mode_desc,
    drs.synchronization_state_desc,
    drs.synchronization_health_desc,
    drs.is_primary_replica,
    drs.log_send_queue_size,
    drs.redo_queue_size,
    drs.last_sent_time,
    drs.last_received_time,
    drs.last_hardened_time,
    drs.last_redone_time,
    drs.last_commit_time
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_groups ag ON drs.group_id = ag.group_id
JOIN sys.availability_replicas ar ON drs.replica_id = ar.replica_id
ORDER BY ag.name, ar.replica_server_name;
```

### 6.2 — On the Managed Instance (`sqlsource01`)

```sql
-- Check replica states
SELECT
    ag.name AS ag_name,
    drs.synchronization_state_desc,
    drs.synchronization_health_desc,
    drs.log_send_queue_size AS log_send_queue_kb,
    drs.redo_queue_size AS redo_queue_kb,
    drs.last_sent_time,
    drs.last_received_time,
    drs.last_hardened_time,
    drs.last_redone_time
FROM sys.dm_hadr_database_replica_states drs
JOIN sys.availability_groups ag ON drs.group_id = ag.group_id;
```

---

## Step 7 — Replication Progress Monitoring

### 7.1 — Continuous Monitoring Query (Run on Source)

This query shows the initial seeding and continuous replication progress:

```sql
-- Run on sqlsource01 (source) or sqltarget01 (target)
-- Real-time replication progress monitoring
SELECT
    DB_NAME(database_id) AS database_name,
    synchronization_state_desc AS sync_state,
    synchronization_health_desc AS sync_health,

    -- Pending data to send (KB)
    log_send_queue_size AS log_send_queue_kb,
    CAST(log_send_queue_size / 1024.0 AS DECIMAL(10,2)) AS log_send_queue_mb,

    -- Pending data to apply on target (KB)
    redo_queue_size AS redo_queue_kb,
    CAST(redo_queue_size / 1024.0 AS DECIMAL(10,2)) AS redo_queue_mb,

    -- Redo rate (KB/s)
    redo_rate AS redo_rate_kb_sec,
    CAST(redo_rate / 1024.0 AS DECIMAL(10,2)) AS redo_rate_mb_sec,

    -- Important timestamps
    last_sent_time,
    last_received_time,
    last_hardened_time,
    last_redone_time,
    last_commit_time,

    -- Estimated time to complete redo (seconds)
    CASE
        WHEN redo_rate > 0
        THEN CAST(redo_queue_size / redo_rate AS DECIMAL(10,1))
        ELSE NULL
    END AS estimated_redo_completion_seconds,

    -- Replication lag
    DATEDIFF(SECOND, last_redone_time, last_commit_time) AS replication_lag_seconds

FROM sys.dm_hadr_database_replica_states
WHERE DB_NAME(database_id) = 'dbtest';
```

### 7.2 — Copy Speed Query (Poll every 10 seconds)

Run this query repeatedly to calculate the actual transfer speed:

```sql
-- Run on target (sqltarget01)
-- Capture snapshot for speed calculation
DECLARE @t1_time DATETIME = GETDATE();
DECLARE @t1_redo BIGINT;
DECLARE @t1_log BIGINT;

SELECT
    @t1_redo = redo_queue_size,
    @t1_log  = log_send_queue_size
FROM sys.dm_hadr_database_replica_states
WHERE DB_NAME(database_id) = 'dbtest';

WAITFOR DELAY '00:00:10'; -- Wait 10 seconds

DECLARE @t2_time DATETIME = GETDATE();
DECLARE @t2_redo BIGINT;
DECLARE @t2_log BIGINT;

SELECT
    @t2_redo = redo_queue_size,
    @t2_log  = log_send_queue_size
FROM sys.dm_hadr_database_replica_states
WHERE DB_NAME(database_id) = 'dbtest';

DECLARE @elapsed_sec DECIMAL(10,2) = DATEDIFF(MILLISECOND, @t1_time, @t2_time) / 1000.0;

SELECT
    @t1_redo AS redo_queue_start_kb,
    @t2_redo AS redo_queue_end_kb,
    (@t1_redo - @t2_redo) AS redo_processed_kb,
    CAST((@t1_redo - @t2_redo) / @elapsed_sec AS DECIMAL(10,2)) AS redo_speed_kb_sec,
    CAST((@t1_redo - @t2_redo) / @elapsed_sec / 1024.0 AS DECIMAL(10,2)) AS redo_speed_mb_sec,

    @t1_log AS log_send_queue_start_kb,
    @t2_log AS log_send_queue_end_kb,
    (@t1_log - @t2_log) AS log_transferred_kb,
    CAST((@t1_log - @t2_log) / @elapsed_sec AS DECIMAL(10,2)) AS log_speed_kb_sec,
    CAST((@t1_log - @t2_log) / @elapsed_sec / 1024.0 AS DECIMAL(10,2)) AS log_speed_mb_sec,

    @elapsed_sec AS measurement_interval_sec;
```

### 7.3 — Monitor Automatic Seeding

During initial seeding, backup and restore happen automatically. Use these queries to track progress:

```sql
-- Run on sqltarget01 (target)
-- Monitor automatic seeding progress
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
    ON asd.ag_id = ag.group_id;
```

```sql
-- Check restore progress during seeding
-- Run on target (sqltarget01)
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

### 7.4 — Database Size Monitoring

```sql
-- Run on source (sqlsource01) to get total size
SELECT
    DB_NAME(database_id) AS database_name,
    type_desc,
    CAST(size * 8.0 / 1024 AS DECIMAL(10,2)) AS size_mb,
    CAST(size * 8.0 / 1024 / 1024 AS DECIMAL(10,2)) AS size_gb
FROM sys.master_files
WHERE DB_NAME(database_id) = 'dbtest';
```

---

## Step 8 — Extended Events for Detailed Monitoring

### 8.1 — Create Extended Events Session on SQL Server

```sql
-- Run on sqltarget01
-- Create Extended Events session to monitor the link
CREATE EVENT SESSION [ManagedLink_Monitor] ON SERVER
ADD EVENT sqlserver.hadr_physical_seeding_progress
    (WHERE ([database_id] = DB_ID('dbtest'))),
ADD EVENT sqlserver.hadr_physical_seeding_submit_callback
    (WHERE ([database_id] = DB_ID('dbtest'))),
ADD EVENT sqlserver.hadr_physical_seeding_restore_state_change,
ADD EVENT sqlserver.hadr_automatic_seeding_state_transition,
ADD EVENT sqlserver.hadr_db_commit_mgr_update_harden,
ADD EVENT sqlserver.hadr_log_block_send_complete,
ADD EVENT sqlserver.hadr_transport_receive_log_block_message,
ADD EVENT sqlserver.hadr_database_flow_control_action,
ADD EVENT sqlserver.availability_replica_state_change,
ADD EVENT sqlserver.error_reported
    (WHERE ([error_number] >= 35200 AND [error_number] <= 35299))
ADD TARGET package0.event_file
    (SET filename = N'ManagedLink_Monitor.xel',
         max_file_size = 100,  -- MB
         max_rollover_files = 5)
WITH (
    MAX_MEMORY = 4096 KB,
    EVENT_RETENTION_MODE = ALLOW_SINGLE_EVENT_LOSS,
    MAX_DISPATCH_LATENCY = 10 SECONDS,
    STARTUP_STATE = ON
);
GO

-- Start the session
ALTER EVENT SESSION [ManagedLink_Monitor] ON SERVER STATE = START;
GO
```

### 8.2 — Query Extended Events Data

```sql
-- Run on sqltarget01
-- Read captured events (last 100)
SELECT TOP 100
    event_data.value('(event/@name)', 'VARCHAR(100)') AS event_name,
    event_data.value('(event/@timestamp)', 'DATETIME2') AS event_time,
    event_data.value('(event/data[@name="database_id"]/value)', 'INT') AS database_id,
    event_data.value('(event/data[@name="error_number"]/value)', 'INT') AS error_number,
    event_data.value('(event/data[@name="message"]/value)', 'VARCHAR(MAX)') AS message,
    event_data.query('.') AS full_event_xml
FROM (
    SELECT CAST(event_data AS XML) AS event_data
    FROM sys.fn_xe_file_target_read_file(
        'ManagedLink_Monitor*.xel', NULL, NULL, NULL)
) AS events
ORDER BY event_time DESC;
```

### 8.3 — Monitor Seeding Progress Specifically via XEvents

```sql
-- Run on sqltarget01
-- Filter only seeding events to track progress
SELECT
    event_data.value('(event/@name)', 'VARCHAR(100)') AS event_name,
    event_data.value('(event/@timestamp)', 'DATETIME2') AS event_time,
    event_data.value('(event/data[@name="transferred_size_bytes"]/value)', 'BIGINT') AS transferred_bytes,
    event_data.value('(event/data[@name="database_size_bytes"]/value)', 'BIGINT') AS total_size_bytes,
    CASE
        WHEN event_data.value('(event/data[@name="database_size_bytes"]/value)', 'BIGINT') > 0
        THEN CAST(
            event_data.value('(event/data[@name="transferred_size_bytes"]/value)', 'FLOAT') /
            event_data.value('(event/data[@name="database_size_bytes"]/value)', 'FLOAT') * 100
            AS DECIMAL(5,2))
        ELSE 0
    END AS percent_complete
FROM (
    SELECT CAST(event_data AS XML) AS event_data
    FROM sys.fn_xe_file_target_read_file(
        'ManagedLink_Monitor*.xel', NULL, NULL, NULL)
) AS events
WHERE event_data.value('(event/@name)', 'VARCHAR(100)')
    LIKE '%seeding%'
ORDER BY event_time DESC;
```

### 8.4 — Clean Up Extended Events Session (when no longer needed)

```sql
-- Run on sqltarget01
ALTER EVENT SESSION [ManagedLink_Monitor] ON SERVER STATE = STOP;
DROP EVENT SESSION [ManagedLink_Monitor] ON SERVER;
```

---

## Step 9 — Detailed Explanation of the Copy Process

### How the Managed Instance Link Copies Data

The replication process follows these phases:

1. **Initial Seeding (Automatic Backup + Restore)**
   - The Managed Instance (source) creates a full backup of the `dbtest` database
   - The backup is automatically transferred over the network (TCP 5022) to the target SQL Server
   - SQL Server restores the database in `NORECOVERY` mode automatically (automatic seeding)
   - During this phase, `dm_hadr_automatic_seeding` shows the progress

2. **Log Synchronization (Continuous log shipping)**
   - After seeding, replication switches to continuous mode
   - Transaction log records are sent from primary to secondary in near-real time
   - `log_send_queue_size` indicates how much log still needs to be sent
   - `redo_queue_size` indicates how much log has been received but not yet applied (redo)

3. **Hardening and Redo**
   - **Hardening:** The received log is written to disk on the target (durable)
   - **Redo:** The log records are applied to the database to make the data visible
   - The `redo_rate` shows the speed (KB/s) of log application

### Progress Indicators

| Metric | Meaning | Ideal |
|---|---|---|
| `log_send_queue_size` | Pending log to send (KB) | Close to 0 |
| `redo_queue_size` | Received but unapplied log (KB) | Close to 0 |
| `redo_rate` | Application speed (KB/s) | > 0, stable |
| `synchronization_state_desc` | Synchronization state | `SYNCHRONIZED` or `SYNCHRONIZING` |
| `last_commit_time` gap | Lag between source and target | < a few seconds |

---

## Step 10 — Failover (When Required)

### 10.1 — Verify readiness for failover

```sql
-- Run on source (sqlsource01)
-- Confirm that replication is up to date
SELECT
    DB_NAME(database_id) AS database_name,
    synchronization_state_desc,
    synchronization_health_desc,
    log_send_queue_size,
    redo_queue_size,
    DATEDIFF(SECOND, last_redone_time, last_commit_time) AS lag_seconds
FROM sys.dm_hadr_database_replica_states
WHERE DB_NAME(database_id) = 'dbtest';
```

> **Failover prerequisite:** `log_send_queue_size` and `redo_queue_size` should be close to zero, and `synchronization_state_desc` should be `SYNCHRONIZED`.

### 10.2 — Execute Failover

Failover can be performed via **PowerShell**, **Azure CLI**, or **SSMS**. PowerShell example:

```powershell
# Azure Cloud Shell
$ManagedInstanceName = "sqlsource01"
$ResourceGroup = (Get-AzSqlInstance -InstanceName $ManagedInstanceName).ResourceGroupName
$LinkName = "DAG_dbtest"

# Planned failover (with full synchronization)
Set-AzSqlInstanceLink `
    -ResourceGroupName $ResourceGroup `
    -InstanceName $ManagedInstanceName `
    -Name $LinkName `
    -ReplicationMode "Sync"

# Wait for full synchronization, then:
Remove-AzSqlInstanceLink `
    -ResourceGroupName $ResourceGroup `
    -InstanceName $ManagedInstanceName `
    -Name $LinkName `
    -AllowDataLoss:$false
```

---

## Troubleshooting

### Verify Network Connectivity

```powershell
# On the sqltarget01 VM
Test-NetConnection -ComputerName "sqlsource01.abc123def456.database.windows.net" -Port 5022
```

### Verify Endpoint State

```sql
-- Run on sqltarget01
SELECT name, state_desc, port
FROM sys.tcp_endpoints
WHERE type_desc = 'DATABASE_MIRRORING';
```

### Check AG-Related Errors

```sql
-- Run on sqltarget01
-- Recent Always On errors (error range 35200-35299)
SELECT TOP 20
    log_date, process_info, text
FROM sys.dm_os_ring_buffers
CROSS APPLY (
    SELECT
        CAST(record AS XML).value('(Record/Error/ErrorNumber)[1]', 'INT') AS error_number
) err
WHERE error_number BETWEEN 35200 AND 35299;
```

```sql
-- Alternative: query the error log
EXEC xp_readerrorlog 0, 1, N'availability';
```

### Collation Mismatch

```sql
-- Check collation on both servers
SELECT SERVERPROPERTY('Collation') AS ServerCollation;
```

> **IMPORTANT:** The collation must be identical between SQL Server and the Managed Instance. A mismatch can cause connection failure.

---

## References

- [Managed Instance Link — Overview](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/managed-instance-link-feature-overview)
- [Configure Link with Scripts (T-SQL/PowerShell/CLI)](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/managed-instance-link-configure-how-to-scripts)
- [Managed Instance Link Failover](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/managed-instance-link-failover-how-to)
- [Managed Instance Link Best Practices](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/managed-instance-link-best-practices)
- [Azure Root Certificate Authorities](https://learn.microsoft.com/en-us/azure/security/fundamentals/azure-ca-details)

---

## Notes and Questions

> _This section will be updated with questions and answers during the execution of the steps._

---

*Document created: 2026-06-07*
*Last updated: 2026-06-07*
