# Plano de Configuração do Managed Link entre Azure SQL Managed Instance e SQL Server em VM

## Visão Geral
Este documento descreve o passo‑a‑passo para criar um **Managed Link** entre:
- **Origem**: Azure SQL Managed Instance chamada `sqlsource01` (banco `dbtest`)
- **Destino**: SQL Server rodando em uma Azure Virtual Machine chamada `sqltarget01`

Ambos os servidores estão na mesma VNet do Azure, permitindo comunicação privada.

O procedimento inclui:
1. Preparação dos servidores (certificados, endpoints, logins e usuários)
2. Criação do Managed Link via T‑SQL
3. Verificação e monitoramento do link
4. Consultas para acompanhar o progresso e a velocidade da cópia de dados
5. Sugestões de Extended Events para monitoramento detalhado

---

## Pré‑requisitos
- Acesso com privilégios de **sysadmin** ou equivalente em ambos os servidores.
- Conectividade de rede: as VNet peering ou VPN já configurada para que `sqlsource01` possa alcançar `sqltarget01` na porta 1433 (ou a porta configurada para o endpoint de dados).
- Ferramentas: SQL Server Management Studio (SSMS), Azure Data Studio ou `sqlcmd`.
- Poder de criação de credenciais e certificados nos dois ambientes.

---

## Etapa 1 – Preparar o Azure SQL Managed Instance (origem)

```sql
-- 1.1 Criar Master Key no banco master (se ainda não existir)
USE master;
GO
IF NOT EXISTS (SELECT * FROM sys.symmetric_keys WHERE name = '##MS_DatabaseMasterKey##')
BEGIN
    CREATE MASTER KEY ENCRYPTION BY PASSWORD = '<StrongMasterKeyPwd!>';
END
GO

-- 1.2 Criar certificado para o Managed Link
CREATE CERTIFICATE MI_Link_Cert
WITH SUBJECT = 'Certificate for Managed Link from sqlsource01';
GO

-- 1.3 Fazer backup do certificado (arquivo .cer) para ser importado no destino
BACKUP CERTIFICATE MI_Link_Cert
TO FILE = 'C:\temp\MI_Link_Cert.cer';   -- caminho acessível pela VM ou storage account
GO

-- 1.4 Criar endpoint de dados para o Managed Link (usando a porta 5022 ou outra livre)
CREATE ENDPOINT MI_Link_Endpoint
STATE = STARTED
AS TCP (LISTENER_PORT = 5022, LISTENER_IP = ALL)
FOR DATABASE_MIRRORING (
    AUTHENTICATION = CERTIFICATE MI_Link_Cert,
    ENCRYPTION = REQUIRED ALGORITHM AES,
    ROLE = ALL
);
GO

-- 1.5 Criar login para o certificado do destino (será criado depois)
-- Vamos criar um login temporário que será mapeado ao certificado do destino após troca
CREATE LOGIN MI_Link_Login
FROM CERTIFICATE MI_Link_Cert;
GO

-- 1.6 Criar usuário no banco dbtest e conceder permissões necessárias
USE dbtest;
GO
CREATE USER MI_Link_User FOR LOGIN MI_Link_Login;
GO
GRANT CONNECT, VIEW DEFINITION TO MI_Link_User;
GRANT SELECT, INSERT, UPDATE, DELETE ON SCHEMA::dbo TO MI_Link_User;  -- ajuste conforme necessário
GO
```

---

## Etapa 2 – Preparar o SQL Server na VM (destino)

> **Observação**: Repetir os mesmos passos no servidor `sqltarget01`, mas invertendo a direção (certificado do destino será usado para autenticar a origem).

```sql
-- 2.1 Master Key (se ainda não existir)
USE master;
GO
IF NOT EXISTS (SELECT * FROM sys.symmetric_keys WHERE name = '##MS_DatabaseMasterKey##')
BEGIN
    CREATE MASTER KEY ENCRYPTION BY PASSWORD = '<StrongMasterKeyPwd!>';
END
GO

-- 2.2 Criar certificado para o destino
CREATE CERTIFICATE Target_Link_Cert
WITH SUBJECT = 'Certificate for Managed Link to sqltarget01';
GO

-- 2.3 Backup do certificado do destino (para ser importado na origem)
BACKUP CERTIFICATE Target_Link_Cert
TO FILE = 'C:\temp\Target_Link_Cert.cer';
GO

-- 2.4 Criar endpoint de dados
CREATE ENDPOINT Target_Link_Endpoint
STATE = STARTED
AS TCP (LISTENER_PORT = 5022, LISTENER_IP = ALL)
FOR DATABASE_MIRRORING (
    AUTHENTICATION = CERTIFICATE Target_Link_Cert,
    ENCRYPTION = REQUIRED ALGORITHM AES,
    ROLE = ALL
);
GO

-- 2.5 Login para o certificado da origem (será criado após troca)
CREATE LOGIN Target_Link_Login
FROM CERTIFICATE Target_Link_Cert;
GO

-- 2.6 Usuário no banco destino (exemplo: dbtest_replica)
USE master; -- ou o banco onde você pretende receber a réplica
GO
CREATE USER Target_Link_User FOR LOGIN Target_Link_Login;
GO
GRANT CONNECT, VIEW DEFINITION TO Target_Link_User;
GRANT CREATE TABLE, INSERT, UPDATE, DELETE, SELECT TO Target_Link_User; -- permissões conforme necessidade
GO
```

---

## Etapa 3 – Troca de Certificados

1. Copie o arquivo `MI_Link_Cert.cer` (origem) para a VM do destino e importe-o:
   ```sql
   USE master;
   GO
   CREATE CERTIFICATE SourceMI_Cert
   FROM FILE = 'C:\temp\MI_Link_Cert.cer';
   GO
   ```

2. Copie o arquivo `Target_Link_Cert.cer` (destino) para a instância gerenciada e importe-o:
   ```sql
   USE master;
   GO
   CREATE CERTIFICATE TargetVM_Cert
   FROM FILE = 'C:\temp\Target_Link_Cert.cer';
   GO
   ```

3. Mapear os logins aos certificados opostos:
   ```sql
   -- Na origem (MI): mapear login do destino ao certificado importado
   ALTER LOGIN MI_Link_Login
   WITH DEFAULT_DATABASE = master;  -- ajuste se necessário
   GO
   -- Não é necessário mapear explicitamente; o login já está associado ao certificado original.
   -- Porém, para usar o certificado da VM, criamos um login separado se desejar:
   CREATE LOGIN MI_Link_VM_Login
   FROM CERTIFICATE TargetVM_Cert;
   GO

   -- No destino (VM): mapear login da origem ao certificado importado
   CREATE LOGIN VM_Link_MI_Login
   FROM CERTIFICATE SourceMI_Cert;
   GO
   ```

---

## Etapa 4 – Criar o Managed Link

> O Managed Link é criado usando a procedure sys.sp_addlinkedsrvlogin ou a sintaxe específica do recurso **Managed Instance Link** (disponível a partir de determinadas versões). Abaixo exemplos genéricos; ajuste conforme a versão e a documentação oficial.

### 4.1 No Azure SQL Managed Instance (origem) – apontar para o destino

```sql
USE master;
GO
-- Cria o linked server que representa o Managed Link
EXEC sp_addlinkedserver
    @server = N'SQltarget01_Link',
    @srvproduct = N'SQL Server',
    @provider = N'SQLNCLI',   -- ou 'MSOLEDBSQL' dependendo do driver
    @datasrc = N'sqltarget01.<vnet-domain>.database.windows.net,1433';  -- use o FQDN ou IP privado
GO

-- Define as credenciais de conexão (usando o login mapeado ao certificado do destino)
EXEC sp_addlinkedsrvlogin
    @rmtsrvname = N'SQltarget01_Link',
    @useself = N'False',
    @locallogin = NULL,
    @rmtuser = N'MI_Link_VM_Login',   -- login criado a partir do certificado da VM
    @rmtpassword = '<StrongPassword!>'; -- senha pode ser vazia se usar autenticação por certificado; ajuste conforme necessário
GO
```

### 4.2 No SQL Server da VM (destino) – apontar para a origem (opcional, para bidirecional)

```sql
USE master;
GO
EXEC sp_addlinkedserver
    @server = N'Sqlsource01_Link',
    @srvproduct = N'SQL Server',
    @provider = N'SQLNCLI',
    @datasrc = N'sqlsource01.<vnet-domain>.public.<region>.azuresql.com,1433';  -- endpoint público ou privado da MI
GO

EXEC sp_addlinkedsrvlogin
    @rmtsrvname = N'Sqlsource01_Link',
    @useself = N'False',
    @locallogin = NULL,
    @rmtuser = N'VM_Link_MI_Login',
    @rmtpassword = '<StrongPassword!>';
GO
```

> **Nota:** Se o ambiente suportar autenticação por certificado exclusivamente, você pode omitir a senha e confiar apenas no mapeamento de login ao certificado.

---

## Etapa 5 – Validar o Managed Link

```sql
-- Teste de conexão simples
SELECT * FROM [SQltarget01_Link].master.sys.databases WHERE name = 'master';
GO

-- Ou, se quiser listar objetos do banco destino
SELECT TABLE_NAME FROM [SQltarget01_Link].dbtest.INFORMATION_SCHEMA.TABLES;
GO
```

Se o retorno for esperado, o link está funcional.

---

## Etapa 6 – Monitoramento do Progresso e Velocidade da Cópia de Dados

### 6.1 Views de estado do Managed Link (disponíveis em versões recentes)

```sql
-- Status geral do link
SELECT * FROM sys.dm_managed_link_status;
GO

-- Detalhes de sessões de cópia
SELECT 
    session_id,
    start_time,
    last_action,
    percent_complete,
    estimated_completion_time,
    total_elapsed_time,
    total_bytes_copied,
    average_bytes_per_second
FROM sys.dm_managed_link_copies;
GO
```

### 6.2 Consultas genéricas de performance (úteis enquanto a cópia ocorre)

```sql
-- Verificar taxa de transferência em bytes por segundo na origem
SELECT 
    DB_NAME(database_id) AS DatabaseName,
    COUNT_BIG(*) AS page_count,
    8 * COUNT_BIG(*) / 1024.0 AS size_mb
FROM sys.dm_os_buffer_descriptors
WHERE database_id = DB_ID('dbtest')
GROUP BY database_id;
GO

-- Verificar solicitações ativas que envolvem o linked server
SELECT 
    session_id,
    status,
    command,
    database_id,
    wait_type,
    wait_time,
    last_wait_type,
    cpu_time,
    total_elapsed_time,
    reads,
    writes
FROM sys.dm_exec_requests
WHERE command LIKE '%distributed%' 
   OR sql_handle IN (
        SELECT sql_handle 
        FROM sys.dm_exec_sql_text 
        WHERE text LIKE '%SQltarget01_Link%'
   );
GO

-- Histórico de throughput via Performance Counters (se o SQL Server estiver configurado para coletar)
SELECT *
FROM sys.dm_os_performance_counters
WHERE object_name LIKE '%SQLServer:Databases%' 
  AND counter_name IN ('Log Bytes Flushed/sec', 'Data File(s) Size (KB)')
  AND instance_name = 'dbtest';
GO
```

### 6.3 Extended Events para monitoramento detalhado

Crie uma sessão XE que capture eventos de cópia e de espera do Managed Link:

```sql
-- Na origem (ou nos dois, conforme desejado)
CREATE EVENT SESSION [MonitorManagedLink] ON SERVER 
ADD EVENT sqlserver.rpc_starting(
    ACTION(sqlserver.sql_text, sqlserver.session_id, sqlserver.username)
    WHERE ([sqlserver].[like_i_sql_unicode_string]([sqlserver].[sql_text], N'%distributed%'))
),
ADD EVENT sqlserver.rpc_completed(
    ACTION(sqlserver.sql_text, sqlserver.session_id, sqlserver.username, sqlserver.duration)
    WHERE ([sqlserver].[like_i_sql_unicode_string]([sqlserver].[sql_text], N'%distributed%'))
),
ADD EVENT sqlserver.wait_info(
    ACTION(sqlserver.session_id, sqlserver.wait_type, sqlserver.duration)
    WHERE ([sqlserver].[wait_type] LIKE '%LINK%' OR [sqlserver].[wait_type] LIKE '%DTC%')
)
ADD TARGET package0.event_file(SET filename=N'C:\XE\MonitorManagedLink.xel', max_file_size=5, max_rollover_files=5)
WITH (MAX_MEMORY=5 MB, EVENT_RETENTION_MODE=ALLOW_SINGLE_EVENT_LOSS, MAX_DISPATCH_LATENCY=30 SECONDS);
GO

-- Iniciar a sessão
ALTER EVENT SESSION [MonitorManagedLink] ON SERVER STATE = START;
GO

-- Para ler os dados coletados (após algum tempo)
SELECT 
    event_data.value('(event/@name)[1]', 'varchar(50)') AS event_name,
    event_data.value('(event/@timestamp)[1]', 'datetime2') AS timestamp,
    event_data.value('(event/data[@name="sql_text"]/value)[1]', 'nvarchar(max)') AS sql_text,
    event_data.value('(event/action[@name="session_id"]/value)[1]', 'int') AS session_id,
    event_data.value('(event/action[@name="duration"]/value)[1]', 'bigint') AS duration_ms
FROM sys.fn_xe_file_target_read_file('C:\XE\MonitorManagedLink*.xel', NULL, NULL, NULL)
ORDER BY timestamp DESC;
GO
```

> Ajuste os caminhos de arquivo conforme o diretório disponível na sua VM ou use armazenamento de blob do Azure para persistir os arquivos `.xel`.

---

## Etapa 7 – Limpeza (se necessário)

```sql
-- Remover linked server
EXEC sp_dropserver 'SQltarget01_Link', 'droplogins';
GO

-- Remover endpoints (se quiser reutilizar as portas)
DROP ENDPOINT MI_Link_Endpoint;
DROP ENDPOINT Target_Link_Endpoint;
GO

-- Remover certificados e logins criados (cuidado com dependências)
-- Exemplo:
DROP USER MI_Link_User;
DROP LOGIN MI_Link_Login;
DROP CERTIFICATE MI_Link_Cert;
-- Repetir analogous no destino
```

---

## Próximos Passos / Perguntas Frequentes

- **Como lidar com atualizações de esquema?**  
  Replique o `ALTER TABLE` etc. usando o linked server ou ferramentas de sincronização (Data Sync, Transactional Replication).

- **Precisamos de criptografia em trânsito?**  
  O endpoint já está configurado com `ENCRYPTION = REQUIRED ALGORITHM AES`. Verifique se a política de rede permite a porta escolhida.

- **O linked server pode ser usado em transações distribuídas?**  
  Sim, mas pode gerar latência; avalie o uso de `sp_getbindtoken` ou de transações isoladas.

- **Quais permissões mínimas são necessárias?**  
  No mínimo: `CREATE ENDPOINT`, `CREATE CERTIFICATE`, `ALTER ANY LOGIN`, `CONNECT SQL` e permissões de leitura/escrita nos bancos envolvidos.

---

### Como usar este documento
1. Salve este arquivo como `SQLMI_managedlink.doc.md` no diretório do projeto.
2. À medida que avançar nas etapas, atualize o arquivo com:
   - Resultados dos comandos (copie a saída relevante).
   - Observações sobre ajustes de porta, nomes de certificados, ou caminhos.
   - Novas queries de monitoramento que você descobrir.
3. Quando precisar consultar o progresso, basta abrir o markdown e localizar a seção **Monitoramento do Progresso e Velocidade da Cópia de Dados**.

---

*Documento criado inicialmente em: <data atual>*