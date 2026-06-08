# Azure SQL Managed Instance Link — Guia de Configuração via T-SQL

## Visão Geral

Este documento descreve os passos para configurar o **Managed Instance Link** entre dois ambientes SQL usando T-SQL:

| Componente | Detalhes |
|---|---|
| **Origem (Primary)** | Azure SQL Managed Instance — servidor `sqlsource01`, banco `dbtest` |
| **Destino (Secondary)** | SQL Server em Azure VM — servidor `sqltarget01` |
| **Rede** | Ambos na mesma VNET no Azure |
| **Porta do endpoint** | TCP 5022 (database mirroring) |
| **Tecnologia** | Distributed Availability Group (Always On) |

> **Nota:** O Managed Instance Link utiliza a tecnologia de Distributed Availability Groups para replicar dados em near-real time entre as duas instâncias.

---

## Pré-Requisitos

### Requisitos do SQL Server na VM (`sqltarget01`)

- SQL Server 2022 (Enterprise ou Developer) com o service update mais recente instalado
- Always On Availability Groups **habilitado** no SQL Server Configuration Manager
- Serviço do SQL Server reiniciado após habilitar Always On
- Acesso `sysadmin` na instância

### Requisitos do Azure SQL Managed Instance (`sqlsource01`)

- Tier Business Critical ou General Purpose (vCore)
- Update policy configurada para **SQL Server 2022** (obrigatório para failover bidirecional)
- Permissões: membro do role **SQL Managed Instance Contributor** ou permissões customizadas equivalentes

### Requisitos de Rede

- Ambos os servidores na mesma VNET
- Porta **TCP 5022** aberta entre as instâncias (NSG rules)
- Resolução DNS funcional entre as instâncias — verificar com `nslookup`
- O FQDN do Managed Instance deve ser resolvível a partir da VM do SQL Server

### Ferramentas Necessárias

- Azure Cloud Shell (PowerShell) ou Az.SQL 6.0.0+ / Azure CLI 2.67.0+
- SQL Server Management Studio (SSMS) 19.x ou superior
- Acesso ao Azure Portal

---

## Etapa 1 — Preparar o Banco de Dados na Origem (`sqlsource01`)

O banco de dados `dbtest` no Managed Instance precisa estar em **FULL recovery model**. Como o MI gerencia backups automaticamente, esta etapa geralmente já está satisfeita.

Verificar o recovery model:

```sql
-- Executar no sqlsource01 (Managed Instance)
SELECT name, recovery_model_desc
FROM sys.databases
WHERE name = 'dbtest';
```

> **Resultado esperado:** `recovery_model_desc = FULL`

O MI já realiza backups automáticos, então não é necessário executar backup manual para o seeding inicial do link.

---

## Etapa 2 — Estabelecer Confiança entre as Instâncias (Certificados)

O Managed Instance Link usa **autenticação por certificados** para o endpoint de database mirroring. É necessário trocar certificados entre as duas instâncias.

### 2.1 — Criar Master Key e Certificado no SQL Server (`sqltarget01`)

```sql
-- Executar no sqltarget01 (SQL Server na VM)
USE master;
GO

-- Criar master key (se não existir)
IF NOT EXISTS (SELECT * FROM sys.symmetric_keys WHERE symmetric_key_id = 101)
BEGIN
    CREATE MASTER KEY ENCRYPTION BY PASSWORD = '<SenhaForte_MasterKey>';
    PRINT 'Master key criada com sucesso.';
END
ELSE
    PRINT 'Master key já existe.';
GO

-- Criar certificado para o endpoint do link
-- IMPORTANTE: Ajuste a data de expiração conforme necessário
DECLARE @cert_expiry_date VARCHAR(20) = '2027-06-30';
DECLARE @cert_name NVARCHAR(MAX) = N'Cert_sqltarget01_endpoint';
DECLARE @cert_subject NVARCHAR(MAX) = N'Certificate for sqltarget01 managed link endpoint';

IF NOT EXISTS (SELECT name FROM sys.certificates WHERE name = @cert_name)
BEGIN
    CREATE CERTIFICATE [Cert_sqltarget01_endpoint]
        WITH SUBJECT = 'Certificate for sqltarget01 managed link endpoint',
        EXPIRY_DATE = '2027-06-30';
    PRINT 'Certificado criado com sucesso.';
END
ELSE
    PRINT 'Certificado já existe.';
GO
```

### 2.2 — Exportar a Chave Pública do Certificado do SQL Server

```sql
-- Executar no sqltarget01
USE master;
GO

DECLARE @cert_name NVARCHAR(MAX) = N'Cert_sqltarget01_endpoint';
DECLARE @PUBLICKEYENC VARBINARY(MAX) = CERTENCODED(CERT_ID(@cert_name));

SELECT @cert_name AS SQLServerCertName;
SELECT @PUBLICKEYENC AS SQLServerPublicKey;
```

> **Ação:** Copie os valores de `SQLServerCertName` e `SQLServerPublicKey` — serão usados na próxima etapa.

### 2.3 — Importar o Certificado do SQL Server no Azure (PowerShell)

Executar no **Azure Cloud Shell (PowerShell)**:

```powershell
# Variáveis — ajustar conforme seu ambiente
$SubscriptionID     = "<SeuSubscriptionID>"
$ManagedInstanceName = "sqlsource01"
$CertificateName     = "Cert_sqltarget01_endpoint"
$PublicKeyEncoded    = "<ColarAquiAPublicKeyObtidaNoPassoAnterior>"  # Começa com 0x...

# Login e seleção de subscription
if ((Get-AzContext) -eq $null) { Login-AzAccount }
Select-AzSubscription -SubscriptionName $SubscriptionID

# Encontrar resource group
$ResourceGroup = (Get-AzSqlInstance -InstanceName $ManagedInstanceName).ResourceGroupName

# Upload do certificado público do SQL Server para o Azure
New-AzSqlInstanceServerTrustCertificate `
    -ResourceGroupName $ResourceGroup `
    -InstanceName $ManagedInstanceName `
    -Name $CertificateName `
    -PublicKey $PublicKeyEncoded
```

### 2.4 — Exportar o Certificado do Managed Instance e Importar no SQL Server

**No Azure Cloud Shell:**

```powershell
$ManagedInstanceName = "sqlsource01"
$ResourceGroup = (Get-AzSqlInstance -InstanceName $ManagedInstanceName).ResourceGroupName

# Obter a chave pública do certificado do MI
Get-AzSqlInstanceEndpointCertificate `
    -ResourceGroupName $ResourceGroup `
    -InstanceName $ManagedInstanceName `
    -EndpointType "DATABASE_MIRRORING" | Out-String
```

> **Ação:** Copie o valor completo de `PublicKey` (começa com `0x`).

**No SQL Server (`sqltarget01`):**

```sql
-- Executar no sqltarget01
-- Substituir <ManagedInstanceFQDN> pelo FQDN do MI
-- Exemplo: sqlsource01.abc123def456.database.windows.net
-- Substituir <PublicKey> pela chave obtida no passo anterior
USE master;
CREATE CERTIFICATE [sqlsource01.abc123def456.database.windows.net]
FROM BINARY = <PublicKey>;
GO
```

> **IMPORTANTE:** O nome do certificado **deve ser exatamente** o FQDN do Managed Instance. Não use nomes customizados.

### 2.5 — Importar Certificados Root CA do Azure no SQL Server

Baixe os certificados root CA do Azure e importe no SQL Server. No mínimo, importe:

1. **DigiCert Global Root G2**
2. **Microsoft RSA Root Certificate Authority 2017**

> **Recomendação:** Para links de longa duração, importe todos os 7 certificados listados em [Azure Root Certificate Authorities](https://learn.microsoft.com/en-us/azure/security/fundamentals/azure-ca-details#root-certificate-authorities).

Baixe os arquivos `.crt` e copie para o servidor SQL, por exemplo em `C:\Certs\`.

```sql
-- Executar no sqltarget01
-- Repetir para cada certificado root CA
USE master;

-- DigiCert Global Root G2
IF NOT EXISTS (SELECT name FROM sys.certificates WHERE name = N'DigiCert Global Root G2')
BEGIN
    CREATE CERTIFICATE [DigiCert Global Root G2]
    FROM FILE = 'C:\Certs\DigiCertGlobalRootG2.crt';

    DECLARE @CERTID_DG INT = CERT_ID('DigiCert Global Root G2');
    EXEC sp_certificate_add_issuer @CERTID_DG, N'*.database.windows.net';
    PRINT 'DigiCert Global Root G2 importado.';
END
GO

-- Microsoft RSA Root Certificate Authority 2017
IF NOT EXISTS (SELECT name FROM sys.certificates WHERE name = N'Microsoft RSA Root Certificate Authority 2017')
BEGIN
    CREATE CERTIFICATE [Microsoft RSA Root Certificate Authority 2017]
    FROM FILE = 'C:\Certs\MicrosoftRSARootCertificateAuthority2017.crt';

    DECLARE @CERTID_MS INT = CERT_ID('Microsoft RSA Root Certificate Authority 2017');
    EXEC sp_certificate_add_issuer @CERTID_MS, N'*.database.windows.net';
    PRINT 'Microsoft RSA Root CA 2017 importado.';
END
GO
```

### 2.6 — Verificar Todos os Certificados

```sql
-- Executar no sqltarget01
USE master;
SELECT name, subject, expiry_date, pvt_key_encryption_type_desc
FROM sys.certificates
ORDER BY expiry_date;
```

> **Validação:** Você deve ver pelo menos 3 certificados: o do sqltarget01, o do FQDN do MI, e os root CAs do Azure.

---

## Etapa 3 — Criar e Configurar o Endpoint de Database Mirroring no SQL Server

### 3.1 — Verificar se já existe um endpoint

```sql
-- Executar no sqltarget01
SELECT name, type_desc, state_desc, role_desc,
       connection_auth_desc, is_encryption_enabled, encryption_algorithm_desc
FROM sys.database_mirroring_endpoints
WHERE type_desc = 'DATABASE_MIRRORING';
```

### 3.2 — Criar o endpoint (se não existir)

```sql
-- Executar no sqltarget01
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

### 3.3 — Alterar endpoint existente (se necessário)

Se já existe um endpoint com autenticação Windows, adicione suporte a certificado:

```sql
-- Executar no sqltarget01
USE master;
ALTER ENDPOINT [<NomeDoEndpointExistente>]
    STATE = STARTED
    AS TCP (LISTENER_PORT = 5022, LISTENER_IP = ALL)
    FOR DATABASE_MIRRORING (
        ROLE = ALL,
        AUTHENTICATION = WINDOWS NEGOTIATE CERTIFICATE [Cert_sqltarget01_endpoint],
        ENCRYPTION = REQUIRED ALGORITHM AES
    );
GO
```

### 3.4 — Validar o endpoint

```sql
-- Executar no sqltarget01
SELECT name, type_desc, state_desc, role_desc,
       connection_auth_desc, is_encryption_enabled, encryption_algorithm_desc
FROM sys.database_mirroring_endpoints;
```

> **Resultado esperado:** `state_desc = STARTED`, `connection_auth_desc = CERTIFICATE`, `encryption_algorithm_desc = AES`

---

## Etapa 4 — Criar Availability Group no SQL Server (`sqltarget01`)

> **Nota:** Como a origem é o Managed Instance (`sqlsource01`), o AG é criado no SQL Server (destino) que receberá a réplica.

### 4.1 — Obter o nome do SQL Server

```sql
-- Executar no sqltarget01
SELECT @@SERVERNAME AS SQLServerName;
```

### 4.2 — Criar o Availability Group

```sql
-- Executar no sqltarget01
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

> **Nota:** Substitua `<sqltarget01_IP>` pelo IP do SQL Server na VM ou pelo hostname resolvível na VNET.

### 4.3 — Criar o Distributed Availability Group

```sql
-- Executar no sqltarget01
-- Substituir os valores conforme seu ambiente
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

> **IMPORTANTE:** Substitua o FQDN `sqlsource01.abc123def456.database.windows.net` pelo FQDN real do seu Managed Instance (visível no Azure Portal > Overview > Host name).

---

## Etapa 5 — Criar o Link no Managed Instance (Azure)

Use o **Azure Cloud Shell (PowerShell)** para criar o link do lado do Managed Instance:

```powershell
# Variáveis
$ManagedInstanceName = "sqlsource01"
$ResourceGroup       = (Get-AzSqlInstance -InstanceName $ManagedInstanceName).ResourceGroupName
$DAGName             = "DAG_dbtest"
$AGNameOnSQLServer   = "AG_dbtest"
$AGNameOnMI          = "AG_dbtest_MI"
$SQLServerIP         = "<sqltarget01_IP>"

# Criar o link
New-AzSqlInstanceLink `
    -ResourceGroupName $ResourceGroup `
    -InstanceName $ManagedInstanceName `
    -Name $DAGName `
    -PrimaryAvailabilityGroupName $AGNameOnMI `
    -SecondaryAvailabilityGroupName $AGNameOnSQLServer `
    -TargetDatabase "dbtest" `
    -SourceEndpoint "TCP://${SQLServerIP}:5022"
```

> **Alternativa:** Também é possível criar o link via SSMS (wizard) ou Azure CLI (`az sql mi link create`).

---

## Etapa 6 — Verificar o Status do Link

### 6.1 — No SQL Server (`sqltarget01`)

```sql
-- Verificar estado do Distributed AG
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

### 6.2 — No Managed Instance (`sqlsource01`)

```sql
-- Verificar estado das réplicas
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

## Etapa 7 — Monitoramento do Progresso da Replicação

### 7.1 — Query de Monitoramento Contínuo (Executar na Origem)

Esta query mostra o progresso do seeding inicial e da replicação contínua:

```sql
-- Executar no sqlsource01 (origem) ou sqltarget01 (destino)
-- Monitoramento em tempo real do progresso da replicação
SELECT
    DB_NAME(database_id) AS database_name,
    synchronization_state_desc AS sync_state,
    synchronization_health_desc AS sync_health,

    -- Dados pendentes de envio (KB)
    log_send_queue_size AS log_send_queue_kb,
    CAST(log_send_queue_size / 1024.0 AS DECIMAL(10,2)) AS log_send_queue_mb,

    -- Dados pendentes de aplicação no destino (KB)
    redo_queue_size AS redo_queue_kb,
    CAST(redo_queue_size / 1024.0 AS DECIMAL(10,2)) AS redo_queue_mb,

    -- Taxa de redo (KB/s)
    redo_rate AS redo_rate_kb_sec,
    CAST(redo_rate / 1024.0 AS DECIMAL(10,2)) AS redo_rate_mb_sec,

    -- Timestamps importantes
    last_sent_time,
    last_received_time,
    last_hardened_time,
    last_redone_time,
    last_commit_time,

    -- Estimativa de tempo para completar o redo (segundos)
    CASE
        WHEN redo_rate > 0
        THEN CAST(redo_queue_size / redo_rate AS DECIMAL(10,1))
        ELSE NULL
    END AS estimated_redo_completion_seconds,

    -- Lag de replicação
    DATEDIFF(SECOND, last_redone_time, last_commit_time) AS replication_lag_seconds

FROM sys.dm_hadr_database_replica_states
WHERE DB_NAME(database_id) = 'dbtest';
```

### 7.2 — Query de Velocidade de Cópia (Polling a cada 10 segundos)

Execute esta query repetidamente para calcular a velocidade real de transferência:

```sql
-- Executar no destino (sqltarget01)
-- Captura snapshot para cálculo de velocidade
DECLARE @t1_time DATETIME = GETDATE();
DECLARE @t1_redo BIGINT;
DECLARE @t1_log BIGINT;

SELECT
    @t1_redo = redo_queue_size,
    @t1_log  = log_send_queue_size
FROM sys.dm_hadr_database_replica_states
WHERE DB_NAME(database_id) = 'dbtest';

WAITFOR DELAY '00:00:10'; -- Esperar 10 segundos

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
    @t1_redo AS redo_queue_inicio_kb,
    @t2_redo AS redo_queue_fim_kb,
    (@t1_redo - @t2_redo) AS redo_processado_kb,
    CAST((@t1_redo - @t2_redo) / @elapsed_sec AS DECIMAL(10,2)) AS velocidade_redo_kb_sec,
    CAST((@t1_redo - @t2_redo) / @elapsed_sec / 1024.0 AS DECIMAL(10,2)) AS velocidade_redo_mb_sec,

    @t1_log AS log_send_queue_inicio_kb,
    @t2_log AS log_send_queue_fim_kb,
    (@t1_log - @t2_log) AS log_transferido_kb,
    CAST((@t1_log - @t2_log) / @elapsed_sec AS DECIMAL(10,2)) AS velocidade_log_kb_sec,
    CAST((@t1_log - @t2_log) / @elapsed_sec / 1024.0 AS DECIMAL(10,2)) AS velocidade_log_mb_sec,

    @elapsed_sec AS intervalo_medicao_sec;
```

### 7.3 — Monitorar o Seeding Automático

Durante o seeding inicial, o backup e restore acontecem automaticamente. Use estas queries para acompanhar:

```sql
-- Executar no sqltarget01 (destino)
-- Monitorar progresso do automatic seeding
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
-- Verificar progresso do restore durante seeding
-- Executar no destino (sqltarget01)
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

### 7.4 — Monitoramento por Tamanho do Banco

```sql
-- Executar na origem (sqlsource01) para saber o tamanho total
SELECT
    DB_NAME(database_id) AS database_name,
    type_desc,
    CAST(size * 8.0 / 1024 AS DECIMAL(10,2)) AS size_mb,
    CAST(size * 8.0 / 1024 / 1024 AS DECIMAL(10,2)) AS size_gb
FROM sys.master_files
WHERE DB_NAME(database_id) = 'dbtest';
```

---

## Etapa 8 — Extended Events para Monitoramento Detalhado

### 8.1 — Criar Sessão de Extended Events no SQL Server

```sql
-- Executar no sqltarget01
-- Criar sessão de Extended Events para monitorar o link
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

-- Iniciar a sessão
ALTER EVENT SESSION [ManagedLink_Monitor] ON SERVER STATE = START;
GO
```

### 8.2 — Consultar os Dados dos Extended Events

```sql
-- Executar no sqltarget01
-- Ler eventos capturados (últimos 100)
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

### 8.3 — Monitorar Especificamente o Progresso do Seeding via XEvents

```sql
-- Executar no sqltarget01
-- Filtrar apenas eventos de seeding para acompanhar progresso
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

### 8.4 — Limpar a Sessão de Extended Events (quando não mais necessária)

```sql
-- Executar no sqltarget01
ALTER EVENT SESSION [ManagedLink_Monitor] ON SERVER STATE = STOP;
DROP EVENT SESSION [ManagedLink_Monitor] ON SERVER;
```

---

## Etapa 9 — Explicação Detalhada do Processo de Cópia

### Como o Managed Instance Link Copia os Dados

O processo de replicação segue estas fases:

1. **Seeding Inicial (Backup + Restore automático)**
   - O Managed Instance (origem) cria um backup completo do banco `dbtest`
   - O backup é transferido automaticamente via rede (TCP 5022) para o SQL Server destino
   - O SQL Server restaura o banco em modo `NORECOVERY` automaticamente (automatic seeding)
   - Durante esta fase, `dm_hadr_automatic_seeding` mostra o progresso

2. **Sincronização de Log (Continuous log shipping)**
   - Após o seeding, a replicação muda para modo contínuo
   - Transaction log records são enviados do primary para o secondary em near-real time
   - O `log_send_queue_size` indica quanto log ainda precisa ser enviado
   - O `redo_queue_size` indica quanto log já foi recebido mas ainda não foi aplicado (redo)

3. **Hardening e Redo**
   - **Hardening:** O log recebido é gravado em disco no destino (durável)
   - **Redo:** Os log records são aplicados ao banco de dados para tornar os dados visíveis
   - A `redo_rate` mostra a velocidade (KB/s) de aplicação dos logs

### Indicadores de Progresso

| Métrica | Significado | Ideal |
|---|---|---|
| `log_send_queue_size` | Log pendente de envio (KB) | Próximo de 0 |
| `redo_queue_size` | Log recebido mas não aplicado (KB) | Próximo de 0 |
| `redo_rate` | Velocidade de aplicação (KB/s) | > 0, estável |
| `synchronization_state_desc` | Estado da sincronização | `SYNCHRONIZED` ou `SYNCHRONIZING` |
| `last_commit_time` gap | Lag entre origem e destino | < poucos segundos |

---

## Etapa 10 — Failover (Quando Necessário)

### 10.1 — Verificar se está pronto para failover

```sql
-- Executar na origem (sqlsource01)
-- Confirmar que a replicação está em dia
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

> **Pré-requisito para failover:** `log_send_queue_size` e `redo_queue_size` devem estar próximos de zero, e `synchronization_state_desc` deve ser `SYNCHRONIZED`.

### 10.2 — Executar Failover

O failover pode ser executado via **PowerShell**, **Azure CLI**, ou **SSMS**. Exemplo com PowerShell:

```powershell
# Azure Cloud Shell
$ManagedInstanceName = "sqlsource01"
$ResourceGroup = (Get-AzSqlInstance -InstanceName $ManagedInstanceName).ResourceGroupName
$LinkName = "DAG_dbtest"

# Failover planejado (com sincronização completa)
Set-AzSqlInstanceLink `
    -ResourceGroupName $ResourceGroup `
    -InstanceName $ManagedInstanceName `
    -Name $LinkName `
    -ReplicationMode "Sync"

# Aguardar sincronização completa, depois:
Remove-AzSqlInstanceLink `
    -ResourceGroupName $ResourceGroup `
    -InstanceName $ManagedInstanceName `
    -Name $LinkName `
    -AllowDataLoss:$false
```

---

## Troubleshooting

### Verificar Conectividade de Rede

```powershell
# Na VM do sqltarget01
Test-NetConnection -ComputerName "sqlsource01.abc123def456.database.windows.net" -Port 5022
```

### Verificar Estado dos Endpoints

```sql
-- Executar no sqltarget01
SELECT name, state_desc, port
FROM sys.tcp_endpoints
WHERE type_desc = 'DATABASE_MIRRORING';
```

### Verificar Erros Relacionados ao AG

```sql
-- Executar no sqltarget01
-- Erros recentes do Always On (error range 35200-35299)
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
-- Alternativa: consultar o error log
EXEC xp_readerrorlog 0, 1, N'availability';
```

### Collation Mismatch

```sql
-- Verificar collation em ambos os servidores
SELECT SERVERPROPERTY('Collation') AS ServerCollation;
```

> **IMPORTANTE:** A collation deve ser idêntica entre o SQL Server e o Managed Instance. Mismatch pode causar falha na conexão.

---

## Referências

- [Managed Instance Link — Visão Geral](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/managed-instance-link-feature-overview)
- [Configurar Link com Scripts (T-SQL/PowerShell/CLI)](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/managed-instance-link-configure-how-to-scripts)
- [Failover do Managed Instance Link](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/managed-instance-link-failover-how-to)
- [Best Practices para Managed Instance Link](https://learn.microsoft.com/en-us/azure/azure-sql/managed-instance/managed-instance-link-best-practices)
- [Azure Root Certificate Authorities](https://learn.microsoft.com/en-us/azure/security/fundamentals/azure-ca-details)

---

## Notas e Perguntas

> _Esta seção será atualizada com perguntas e respostas durante a execução dos passos._

---

*Documento criado em: 2026-06-07*
*Última atualização: 2026-06-07*
