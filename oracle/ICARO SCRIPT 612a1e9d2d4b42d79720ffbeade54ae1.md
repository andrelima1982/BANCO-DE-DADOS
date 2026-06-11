# ICARO SCRIPT

Date Created: 18 de dezembro de 2023 16:30

SERVIDOR COMANDOS

- Controle
    
    Acesso IBMCLOUD - Z&485#4Wf999$9
    
- Mysql
    - Procedimento para corrigir o erro de acesso
        
        ```sql
        Procedimento para corrigir o erro de acesso negado para root e outros usuários nos bancos MYSQL 8 ou superior
        OBS: Se for possível acessar com ROOT aí o processo é apenas alterar a senha do usuário.
        
        systemctl stop mysqld
        
        systemctl set-environment MYSQLD_OPTS="--skip-grant-tables --skip-networking";
        
        systemctl start mysqld
        
        mysql -u root
        
        FLUSH PRIVILEGES;
        
        ALTER USER 'root'@'localhost' IDENTIFIED BY 'SENHAROOT';
        
        ou
        
        UPDATE mysql.user SET Password=PASSWORD('SENHAROOT') WHERE User='root';
        
        exit
        
        systemctl stop mysqld
        
        systemctl unset-environment MYSQLD_OPTS
        
        systemctl start mysqld
        
        mysql -u root -p
        
        alter user 'SEUUSUARIO'@'localhost' identified by 'SENHAUSUARIO';
        Se o comando acima falhar use o UPDATE abaixo
        
        Use o select abaixo para ver o tipo de autenticação e plugin utilizado
        
        SELECT user, host, authentication_string, plugin FROM mysql.user WHERE user LIKE 'usrdb';
        
        Select para testar a criação da string
        
        SELECT concat('*',upper(sha1(unhex(sha1('SENHAUSUARIO'))))) AS authentication_string;
        
        UPDATE mysql.user SET authentication_string=concat('*',upper(sha1(unhex(sha1('SENHAUSUARIO')))))  WHERE user='SEUUSUARIO' and plugin='mysql_native_password';
        
        FLUSH PRIVILEGES;
        
        exit
        
        Tentar fazer logon com o usuário e senha alterados
        ```
        
    - Verificar tabelas em uso
        
        ```sql
        --Verificar tabelas em uso
        SHOW OPEN TABLES WHERE In_use > 0;
        ```
        
    - Parar e subir um banco
        
        ```sql
        systemctl stop httpd ; systemctl stop mysqld
        ```
        
    - Verificar privilégios de admin
        
        ```sql
        SELECT 
            user,
            host,
            Super_priv,
            Grant_priv,
            Create_user_priv,
            Shutdown_priv
        FROM mysql.user
        WHERE Super_priv = 'Y'
           OR Grant_priv = 'Y'
           OR Create_user_priv = 'Y'
        ORDER BY user;
        ```
        
    - Verificar usuários e quais privilégios globais eles têm
        
        ```sql
        SELECT 
            GRANTEE AS 'Usuário',
            PRIVILEGE_TYPE AS 'Privilégio',
            IS_GRANTABLE AS 'Pode Conceder (Grant)'
        FROM information_schema.USER_PRIVILEGES
        ORDER BY GRANTEE, PRIVILEGE_TYPE;
        ```
        
    - Tamanho total dos bancos
        
        ```sql
        SELECT table_schema "Database", 
               ROUND(SUM(data_length + index_length) / 1024 / 1024, 1) "Size(MB)" 
        FROM information_schema.tables 
        GROUP BY table_schema;
        
        ----
        --Tamanho de um database específico
        SELECT
          table_schema AS database_name,
          ROUND(SUM(data_length + index_length) / 1024 / 1024, 2) AS size_mb
        FROM information_schema.tables
        WHERE table_schema = 'discovery';
        
        -----
        --Tamanho por tabela
        SELECT
          table_name,
          ROUND((data_length + index_length) / 1024 / 1024, 2) AS size_mb
        FROM information_schema.tables
        WHERE table_schema = 'discovery'
        ORDER BY size_mb DESC;
        
        ```
        
    - Tamanho das tablespaces
        
        ```sql
        SELECT
            SPACE,
            NAME,
            SPACE_TYPE,
            ROUND(FILE_SIZE / 1024 / 1024 / 1024, 2) AS FILE_SIZE_GB,
            ROUND(ALLOCATED_SIZE / 1024 / 1024 / 1024, 2) AS ALLOCATED_SIZE_GB,
            ROUND(AUTOEXTEND_SIZE / 1024 / 1024 / 1024, 2) AS AUTOEXTEND_SIZE_GB,
            PAGE_SIZE,
            ROW_FORMAT,
            ENCRYPTION,
            STATE
        FROM information_schema.INNODB_TABLESPACES
        ORDER BY FILE_SIZE DESC
        LIMIT 10;
        
        ******
        --Verificar fragmentação
        SELECT
          SPACE,
          NAME,
          SPACE_TYPE,
          ROUND(FILE_SIZE / 1024 / 1024 / 1024, 2) AS FILE_SIZE_GB,
          ROUND(ALLOCATED_SIZE / 1024 / 1024 / 1024, 2) AS ALLOCATED_SIZE_GB,
          ROUND( (CASE WHEN FILE_SIZE > ALLOCATED_SIZE THEN FILE_SIZE - ALLOCATED_SIZE ELSE 0 END) / 1024 / 1024 / 1024, 2) AS FRAGMENTATION_GB,
          ROUND(
            CASE 
              WHEN FILE_SIZE > 0 THEN (CASE WHEN FILE_SIZE > ALLOCATED_SIZE THEN (FILE_SIZE - ALLOCATED_SIZE) ELSE 0 END) / FILE_SIZE * 100
              ELSE NULL
            END
          , 2) AS FRAGMENTATION_PCT,
          PAGE_SIZE,
          ROW_FORMAT,
          ENCRYPTION,
          STATE
        FROM information_schema.INNODB_TABLESPACES
        ORDER BY FILE_SIZE DESC
        LIMIT 10;
        
        *******
        --Verifica tamanho por schema
        SELECT
          SUBSTRING_INDEX(NAME, '/', 1) AS SCHEMA_NAME,
          ROUND(SUM(FILE_SIZE) / 1024 / 1024 / 1024, 2) AS TOTAL_SIZE_GB,
          ROUND(SUM(ALLOCATED_SIZE) / 1024 / 1024 / 1024, 2) AS ALLOCATED_GB,
          ROUND(SUM(CASE WHEN FILE_SIZE > ALLOCATED_SIZE THEN (FILE_SIZE - ALLOCATED_SIZE) ELSE 0 END) / 1024 / 1024 / 1024, 2) AS FRAGMENTATION_GB,
          ROUND(
            CASE 
              WHEN SUM(FILE_SIZE) > 0 THEN SUM(CASE WHEN FILE_SIZE > ALLOCATED_SIZE THEN (FILE_SIZE - ALLOCATED_SIZE) ELSE 0 END) / SUM(FILE_SIZE) * 100
              ELSE NULL
            END
          , 2) AS FRAGMENTATION_PCT
        FROM information_schema.INNODB_TABLESPACES
        GROUP BY SCHEMA_NAME
        ORDER BY TOTAL_SIZE_GB DESC;
        
        ```
        
    - Verificar os bancos
        
        ```sql
        SHOW DATABASES;
        ```
        
    - Verificar o tamanho das tabelas
        
        ```sql
        --verificar tamanho das tabelas MySQL
        SELECT 
            table_schema AS database_name,
            table_name,
            ROUND(data_length / 1024 / 1024, 2) AS data_size_mb,
            ROUND(index_length / 1024 / 1024, 2) AS index_size_mb,
            ROUND((data_length + index_length) / 1024 / 1024, 2) AS total_size_mb
        FROM information_schema.tables
        WHERE table_schema IN ('operation')
        ORDER BY total_size_mb DESC;
        ```
        
    - Listar processos inativos
        
        ```sql
        --listar processos inativos
        SELECT 
            ID,
            USER,
            HOST,
            DB,
            TIME,
            STATE,
            INFO
        FROM INFORMATION_SCHEMA.PROCESSLIST
        WHERE COMMAND = 'Sleep'
        ORDER BY TIME DESC;
        ```
        
    - Verificar sessões ativas com status relacionados a LOCK
        
        ```sql
        --sessões ativas com estados relacionados a lock MySQL
        SELECT * 
        FROM information_schema.PROCESSLIST 
        WHERE COMMAND = 'Sleep' OR STATE LIKE '%lock%';
        ```
        
    - Consumo de CPU por sessão
        
        ```sql
        --Consumo de CPU por Sessão (Thread)
        SELECT 
            thd.PROCESSLIST_ID AS thread_id,
            thd.PROCESSLIST_USER AS user,
            thd.PROCESSLIST_HOST AS host,
            thd.PROCESSLIST_DB AS db,
            thd.PROCESSLIST_COMMAND AS command,
            thd.PROCESSLIST_TIME AS time,
            thd.PROCESSLIST_STATE AS state,
            es.TIMER_WAIT / 1000000000000 AS cpu_time_s
        FROM 
            performance_schema.threads thd
        JOIN 
            performance_schema.events_statements_current es 
            ON thd.THREAD_ID = es.THREAD_ID
        WHERE 
            thd.PROCESSLIST_DB = 'operation'
        ORDER BY 
            cpu_time_s DESC
        	\G;
        
        --Identificar a Query que está rodando nessa thread
        SELECT ID, USER, HOST, DB, COMMAND, TIME, STATE, INFO 
        FROM information_schema.processlist 
        WHERE ID = 37834;
        ```
        
    - Verificar processos em execução
        
        ```sql
        SHOW PROCESSLIST;
        ```
        
    - Verificar uso de espaço
        
        ```sql
        SELECT table_schema "Database", 
               SUM(data_length + index_length) / 1024 / 1024 "Size (MB)" 
        FROM information_schema.tables 
        GROUP BY table_schema;
        ```
        
    - Top 20 queries consumindo CPU
        
        ```sql
        --top 20 queries consumindo CPU MySQL
        SELECT 
            DIGEST_TEXT AS query,
            SCHEMA_NAME AS database_name,
            COUNT_STAR AS execution_count,
            ROUND(SUM_TIMER_WAIT / 1000000000000, 3) AS total_time_s,
            ROUND(AVG_TIMER_WAIT / 1000000000000, 3) AS avg_time_s,
            ROUND(SUM_LOCK_TIME / 1000000000000, 3) AS total_lock_time_s,
            ROUND(MAX_TIMER_WAIT / 1000000000000, 3) AS max_time_s,
            FORMAT(SUM_ROWS_SENT, 0) AS total_rows_sent,
            FORMAT(SUM_ROWS_EXAMINED, 0) AS total_rows_examined
        FROM 
            performance_schema.events_statements_summary_by_digest
        WHERE 
            SCHEMA_NAME = 'operation'
        ORDER BY 
            total_time_s DESC
        LIMIT 10
        \G; > top20.txt
        ```
        
    - Verificar se uma coluna possui index
        
        ```sql
        --verificar se uma coluna possui index
        
        SHOW INDEX FROM operation.alarmes_ericsson;
        
        ou
        
        SELECT
            TABLE_SCHEMA,
            TABLE_NAME,
            INDEX_NAME,
            COLUMN_NAME
        FROM
            information_schema.STATISTICS
        WHERE
            TABLE_SCHEMA = 'operation'
            AND TABLE_NAME = 'alarmes'
            AND COLUMN_NAME = 'NEName';
        ```
        
    - Verificar se é uma view ou uma tabela
        
        ```sql
        -- verificar se é uma view ou uma tabela
        
        SELECT TABLE_TYPE
        FROM information_schema.TABLES
        WHERE TABLE_SCHEMA = 'operation' 
          AND TABLE_NAME = 't_vandalismo';
        ```
        
    - Gerar o DDL de uma view
        
        ```sql
        --gerar o ddl da view
        
        SHOW CREATE VIEW operation.alarmes\G
        ```
        
    - Listar tabelas
        
        ```sql
        SHOW tables;
        ```
        
    - Descrever  uma tabela
        
        ```sql
        DESCRIBE nome_da_tabela;
        
        ou 
        
        SHOW COLUMNS FROM nome_da_tabela;
        ```
        
    - Verificar colunas de uma tabela
        
        ```sql
        SHOW COLUMNS FROM operation.t_alarmes;
        
        ou
        
        --verificar se uma coluna existe
        
        SELECT 
            COLUMN_NAME
        FROM 
            information_schema.COLUMNS
        WHERE 
            TABLE_SCHEMA = 'operation' 
            AND TABLE_NAME = 'alarmes' 
            AND COLUMN_NAME = 'NEName'; 
        ```
        
- POSTGRESQL
    - **Sessões bloqueadas e bloqueios não concedidos identificar se algum processo está travando outro**
        
        ```sql
        SELECT 
            blocked_locks.pid AS blocked_pid,
            blocked_activity.usename AS blocked_user,
            blocked_activity.query AS blocked_query,
            blocking_locks.pid AS blocking_pid,
            blocking_activity.usename AS blocking_user,
            blocking_activity.query AS blocking_query
        FROM 
            pg_catalog.pg_locks AS blocked_locks
        JOIN 
            pg_catalog.pg_stat_activity AS blocked_activity ON blocked_activity.pid = blocked_locks.pid
        JOIN 
            pg_catalog.pg_locks AS blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
            AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
            AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation
            AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
            AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple
            AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
            AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid
            AND blocking_locks.virtualtransaction IS NOT DISTINCT FROM blocked_locks.virtualtransaction
            AND blocking_locks.pid <> blocked_locks.pid
        JOIN 
            pg_catalog.pg_stat_activity AS blocking_activity ON blocking_activity.pid = blocking_locks.pid
        WHERE 
            NOT blocked_locks.granted;
        
        ```
        
    - Verificar sessões por database
        
        ```sql
        SELECT
            pid,
            usename,
            datname,
            state,
            now() - state_change AS tempo_no_estado,
            client_addr,
            query
        FROM pg_stat_activity
        WHERE datname = 'cemigdata'
        ORDER BY state, state_change;
        ```
        
    - Verificar permissões de um usuário
        
        ```sql
        SELECT
            grantee,
            table_schema,
            table_name,
            privilege_type
        FROM information_schema.table_privileges
        WHERE grantee = 'gabriela.matsunaga'
          AND table_name IN (
                'workflow_agv_protocolo_nota',
                'workflow_agv_historico'
          );
        
        ```
        
    - Verificar tamanho de todas as tabelas
        
        ```sql
        SELECT
            relname AS "Nome da Tabela",
            pg_size_pretty(pg_total_relation_size(relid) / (1024 * 1024)) AS "Tamanho Total (MB)"
        FROM
            pg_catalog.pg_statio_user_tables
        ORDER BY
            pg_total_relation_size(relid) DESC;
        
        ```
        
    - Verificar tamanho do banco
        
        ```sql
        SELECT
            pg_size_pretty(pg_database_size(current_database())) AS "Tamanho do Banco de Dados";
        
        ```
        
    - Agendador crontab
        
        ```sql
        select * from cron.job;public."BASE_MATERIAL"
        
        select * from cron.job_run_details;
        
        SELECT cron.schedule_in_database(
          'vacuum_full_basematerial_domingo_19h', 
          '0 19 * * 0',
          $$VACUUM FULL public."BASE_MATERIAL";$$,
          'ibmclouddb'
        );
        ```
        
    - Verificar tempo de atividade de uma instância
        
        ```sql
        SELECT
            inet_server_addr() AS server_ip,
            current_setting('server_version') AS postgres_version,
            pg_postmaster_start_time() AS start_time,
            NOW() - pg_postmaster_start_time() AS uptime;
            
        **********************
        --Com o hostname
        SELECT
            inet_server_addr() AS server_ip,
            pg_read_file('/etc/hostname') AS server_hostname,
            current_setting('server_version') AS postgres_version,
            pg_postmaster_start_time() AS start_time,
            NOW() - pg_postmaster_start_time() AS uptime;
        
        ```
        
    - Verificar quem pode alterar uma tabela
        
        ```sql
        SELECT 
            u.rolname AS usuario,
            c.relname AS tabela,
            CASE 
                WHEN u.rolsuper THEN 'PODE (SUPERUSER)'
                WHEN c.relowner = u.oid THEN 'PODE (OWNER)'
                ELSE 'NAO PODE'
            END AS pode_alterar
        FROM 
            pg_class c
        JOIN 
            pg_roles u ON u.rolname = 'gabriele_silva'
        WHERE 
            c.relname = 'workflow_agv_etapa_atual'
            AND c.relkind = 'r'; -- apenas tabelas
        
        ```
        
    - Verifica o número máximo de conexões e quantas tem no momento
        
        ```sql
        SELECT 
            (SELECT COUNT(*) FROM pg_stat_activity) AS current_connections,
            (SELECT setting::int FROM pg_settings WHERE name = 'max_connections') AS max_connections;
        ```
        
    - Verifica todas as conexões e identifica quem é
        
        ```sql
        SELECT
            pid,
            usename,
            datname,
            state,
            backend_start,
            client_addr
        FROM pg_stat_activity;
        ```
        
    - Verifica colunas de uma tabela e configurações
        
        ```sql
        SELECT 
            column_name,
            data_type,
            character_maximum_length,
            numeric_precision,
            is_nullable
        FROM 
            information_schema.columns
        WHERE 
            table_name = 'workflow_agv_etapa_atual'
        ORDER BY 
            ordinal_position;
        ```
        
    - Conexões ativas por banco e usuário
        
        ```sql
        SELECT datname, usename, count(*) AS conexoes_ativas
        FROM pg_stat_activity
        WHERE state = 'active'
        GROUP BY datname, usename
        ORDER BY datname, conexoes_ativas DESC;
        
        ```
        
    - Espaço usado por tablespaces e bancos
        
        ```sql
        SELECT pg_database.datname, 
               pg_size_pretty(pg_database_size(pg_database.datname)) AS tamanho
        FROM pg_database
        ORDER BY pg_database_size(pg_database.datname) DESC;
        
        ```
        
    - Conceder privilégios
        
        ```sql
        --Para conceder permissões em tabelas
        ALTER DEFAULT PRIVILEGES IN SCHEMA seu_schema GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO seu_usuario;
        
        --Para conceder permissões em sequences
        ALTER DEFAULT PRIVILEGES IN SCHEMA seu_schema GRANT ALL ON SEQUENCES TO seu_usuario;
        
        --Para conceder permissões em funções
        ALTER DEFAULT PRIVILEGES IN SCHEMA seu_schema GRANT EXECUTE ON FUNCTIONS TO seu_usuario;
        
        --Criar objetos nos schemas
        GRANT CREATE ON SCHEMA <schema> TO <user>;
        ```
        
    - GRANTS
        
        ```sql
        SELECT 'GRANT ' || privilege_type || ' ON FUNCTION ' || specific_schema || '.' || routine_name || 
               ' TO ibmvale;'
        FROM information_schema.role_routine_grants
        WHERE grantee = 'postgres';
        
        SELECT 'GRANT ' || privilege_type || ' ON ' || table_schema || '.' || table_name || 
               ' TO ibmvale;'
        FROM information_schema.role_table_grants
        WHERE grantee = 'postgres';
        
        ```
        
    - Atividades longas (queries demorando muito)
        
        ```sql
        SELECT pid, usename, now() - query_start AS duracao, query
        FROM pg_stat_activity
        WHERE state = 'active' AND (now() - query_start) > interval '5 minutes'
        ORDER BY duracao DESC;
        
        ```
        
    - Verificar a tablespace de um banco
        
        ```sql
        SELECT datname, pg_database.datdba, pg_database_size(datname), pg_tablespace.spcname, pg_tablespace_location(pg_database.dattablespace)
        FROM pg_database
        JOIN pg_tablespace ON pg_database.dattablespace = pg_tablespace.oid
        WHERE datname = 'platform_analytics_wh';
        
        ```
        
    - Finalizar uma sessão
        
        ```sql
        SELECT pg_terminate_backend(<blocking_pid>);
        ```
        
    - V**erificar bancos de dados**
        
        ```sql
        SELECT * FROM pg_database WHERE datname = 'cemigdata';
        ```
        
    - **Conectar ao bancos de dados**
        
        ```sql
        \connect <nome_do_banco>
        ```
        
    - Listar as tabelas do banco de dados
        
        ```sql
        \dt
        ```
        
- SQL SERVER
    - V**erificar tamanho dos logs**
        
        ```sql
        SELECT 
            DB_NAME(database_id) AS DatabaseName,
            Name AS LogicalName,
            Physical_Name AS PhysicalFileName,
            CONVERT(decimal(10,2), (size * 8.0) / 1024 / 1024) AS FileSizeGB
        FROM 
            sys.master_files
        WHERE 
            type_desc = 'LOG';
        ```
        
    - V**erificar versão do banco**
        
        ```sql
        SELECT @@VERSION AS SQLServerVersion;
        ```
        
- CIRION
    - V**erificar Sessões com muito tempo de execução**
        
        ```sql
        --sessões com muito tempo de execução
        select *,
          pid, 
          now() - pg_stat_activity.query_start AS duration,
          query,
          state
        FROM pg_stat_activity
        WHERE (now() - pg_stat_activity.query_start) > interval '15 hours'
        and query <> 'COMMIT'
        
        *** Ignorar querys com START_REPLICATION SLOT
        *** IDLE IN TRANSACTION (Warning)
        
        ```
        
    - Conexões
        
        ```sql
        para conectar no POD
        putty : 4.74.134.65
        login : camatoco
        senha : n3JowJq7bZYUSIjcms
        
        kubectl exec -it -n opennms timescaledb-backbone-1 -- bash (conectar no zero ou um)
        kubectl exec -it -n opennms timescaledb-main-1 -- bash
        kubectl exec -it -n opennms timescaledb-flow-1 -- bash
        
        Identificar Leader e réplica
        patronictl -c /etc/timescaledb/patroni.yaml list
        
        Percona
        -- Percona
        http://4.74.134.65:31683/
        admin
        5%qLc9f7<Cht
        ```
        
- DB2
    - **Troubleshooting Total Indoubt Transactions**
        
        ```sql
        Use o comando a seguir para se conectar ao banco de dados:
        
        db2 connect to NOMEDOBANCO
        
        Verificar Processos Problemáticos
        
        db2 list indoubt transactions with prompting
        
        --Evidência após tratativa
        db2 "
        SELECT
            CURRENT SERVER    AS DB_NAME,
            CURRENT DATE      AS DATA,
            CURRENT TIME      AS HORA,
            member,
            ((total_log_available/1024)/1024)/1024 AS Available_Logsize_in_GB,
            num_indoubt_trans,
            APPLID_HOLDING_OLDEST_XACT
        FROM TABLE(mon_get_transaction_log(-2)) AS t
        ORDER BY member ASC"
        
        ```
        
    - Verificar sessões ativas e idle
        
        ```sql
        db2 "SELECT 
          CASE 
            WHEN CLIENT_IDLE_WAIT_TIME < 10000000 THEN 'ATIVA'
            ELSE 'IDLE'
          END AS ESTADO_SESSAO,
          COUNT(*) AS QTD_SESSOES
        FROM TABLE(MON_GET_CONNECTION(CAST(NULL AS BIGINT), -1)) AS CONN
        GROUP BY 
          CASE 
            WHEN CLIENT_IDLE_WAIT_TIME < 10000000 THEN 'ATIVA'
            ELSE 'IDLE'
          END;"
        ```
        
    - Teste de cargas
        
        ```sql
        --Atividades criadas
        db2 "SELECT
                LP.NAME || ' - ' || T.ACTIVITY_NAME AS activity_name,
                COUNT(*) as baw_activity_created
            FROM
                BAWDBUSR.LSW_TASK T
            JOIN
                BAWDBUSR.LSW_BPD_INSTANCE BPD ON T.BPD_INSTANCE_ID = BPD.BPD_INSTANCE_ID
            JOIN
                BAWDBUSR.LSW_SNAPSHOT LS ON BPD.SNAPSHOT_ID = LS.SNAPSHOT_ID
            JOIN
                BAWDBUSR.LSW_PROJECT LP ON LS.PROJECT_ID = LP.PROJECT_ID
            WHERE
                TRUNC_TIMESTAMP(T.RCVD_DATETIME, 'MI') - MOD(MINUTE(T.RCVD_DATETIME), 5) MINUTES >= CURRENT TIMESTAMP - CURRENT TIMEZONE - 10 MINUTES
                AND TRUNC_TIMESTAMP(T.RCVD_DATETIME, 'MI') - MOD(MINUTE(T.RCVD_DATETIME), 5) MINUTES < CURRENT TIMESTAMP - CURRENT TIMEZONE - 5 MINUTES
            GROUP BY
                TRUNC_TIMESTAMP(T.RCVD_DATETIME, 'MI') - MOD(MINUTE(T.RCVD_DATETIME), 5) MINUTES,
                LP.name,
                BPD.BPD_NAME,
                T.ACTIVITY_NAME
            UNION ALL
            SELECT
                NULL AS activity_name,
                0 as baw_activity_created_count
            FROM
            SYSIBM.SYSDUMMY1
            WHERE NOT EXISTS (SELECT
                    1
                FROM
                    BAWDBUSR.LSW_TASK T
                JOIN
                    BAWDBUSR.LSW_BPD_INSTANCE BPD ON T.BPD_INSTANCE_ID = BPD.BPD_INSTANCE_ID
                JOIN
                    BAWDBUSR.LSW_SNAPSHOT LS ON BPD.SNAPSHOT_ID = LS.SNAPSHOT_ID
                JOIN
                    BAWDBUSR.LSW_PROJECT LP ON LS.PROJECT_ID = LP.PROJECT_ID
                WHERE
                    TRUNC_TIMESTAMP(T.RCVD_DATETIME, 'MI') - MOD(MINUTE(T.RCVD_DATETIME), 5) MINUTES >= CURRENT TIMESTAMP - CURRENT TIMEZONE - 10 MINUTES
                    AND TRUNC_TIMESTAMP(T.RCVD_DATETIME, 'MI') - MOD(MINUTE(T.RCVD_DATETIME), 5) MINUTES < CURRENT TIMESTAMP - CURRENT TIMEZONE - 5 MINUTES
                GROUP BY
                    TRUNC_TIMESTAMP(T.RCVD_DATETIME, 'MI') - MOD(MINUTE(T.RCVD_DATETIME), 5) MINUTES,
                    LP.name,
                    BPD.BPD_NAME,
                    T.ACTIVITY_NAME)"
                    
           ---Somatório das cargas criadas
           db2 "SELECT
                COUNT(*) AS baw_activity_created
            FROM
                BAWDBUSR.LSW_TASK T
            JOIN
                BAWDBUSR.LSW_BPD_INSTANCE BPD ON T.BPD_INSTANCE_ID = BPD.BPD_INSTANCE_ID
            JOIN
                BAWDBUSR.LSW_SNAPSHOT LS ON BPD.SNAPSHOT_ID = LS.SNAPSHOT_ID
            JOIN
                BAWDBUSR.LSW_PROJECT LP ON LS.PROJECT_ID = LP.PROJECT_ID
            WHERE
                TRUNC_TIMESTAMP(T.RCVD_DATETIME, 'MI') - MOD(MINUTE(T.RCVD_DATETIME), 5) MINUTES >= CURRENT TIMESTAMP - CURRENT TIMEZONE - 35 MINUTES
                AND TRUNC_TIMESTAMP(T.RCVD_DATETIME, 'MI') - MOD(MINUTE(T.RCVD_DATETIME), 5) MINUTES < CURRENT TIMESTAMP - CURRENT TIMEZONE - 5 MINUTES"
        ```
        
    - Verificar quantidade de sessões
        
        ```sql
        db2 "SELECT
            APPL_STATUS,
            COUNT(*) QTD
        FROM SYSIBMADM.APPLICATIONS
        GROUP BY APPL_STATUS;"
        ```
        
    - Verificar valor de memória em uso
        
        ```sql
        db2pd -dbptnmem
        ```
        
    - Identificar necessidade de criação de index
        
        ```sql
        --A query irá mostrar as 5 consultas que estão onerando
        SELECT HEX(EXECUTABLE_ID) AS EXECUTABLE_ID_HEX,
               NUM_EXEC_WITH_METRICS AS EXECUCOES,
               ROWS_READ / NUM_EXEC_WITH_METRICS AS MEDIA_LINHAS_LIDAS, 
               ROWS_RETURNED / NUM_EXEC_WITH_METRICS AS MEDIA_LINHAS_RETORNADAS, 
               SUBSTR(STMT_TEXT, 1, 100) AS SQL_TEXT 
        FROM TABLE(MON_GET_PKG_CACHE_STMT(NULL, NULL, NULL, -2)) 
        WHERE NUM_EXEC_WITH_METRICS > 0 
        ORDER BY MEDIA_LINHAS_LIDAS DESC 
        FETCH FIRST 5 ROWS ONLY;
        ```
        
    - Verificar Locks
        
        ```sql
        SELECT 
            APPLICATION_HANDLE AS handle_esperando,
            LOCK_NAME AS nome_do_lock,
            LOCK_MODE AS modo_solicitado,
            LOCK_OBJECT_TYPE AS tipo_objeto,
            LOCK_STATUS AS status
        FROM 
            TABLE(MON_GET_LOCKS(NULL, -2))
        WHERE 
            LOCK_STATUS = 'W';
        ```
        
    - Verificar todos os locks na memória
        
        ```sql
        SELECT 
            APPLICATION_HANDLE AS handle_conexao,
            LOCK_NAME AS nome_do_lock,
            LOCK_MODE AS modo,             -- ex: X (Exclusivo), S (Compartilhado)
            LOCK_STATUS AS status,         -- G (Granted/Concedido) ou W (Waiting/Esperando)
            LOCK_OBJECT_TYPE AS tipo_objeto
        FROM 
            TABLE(MON_GET_LOCKS(NULL, -2))
        ORDER BY 
            LOCK_STATUS DESC, APPLICATION_HANDLE;
            
        *****Interpretação*******
        
        1571 (Handle/Conexão): Este é o ID da aplicação ou processo que está rodando no DB2.
        
        S (Modo Share / Compartilhado): Significa que a conexão 1571 abriu uma trava de leitura. Locks do tipo S permitem que outros usuários também leiam os mesmos dados ao mesmo tempo.
        
        G (Status Granted / Concedido): Excelente sinal. Significa que o DB2 já liberou o acesso para essa conexão. Ela não está travada na fila, o banco deu o "ok" e ela está operando normalmente.
        
        VARIATION (Tipo de Objeto): No DB2, o lock do tipo Variation é uma trava interna de controle do sistema (geralmente ligada à validação de planos de execução de SQL, procedures ou alterações menores de ambiente). Não é um lock pesado de tabela ou linha de dados (Table ou Row).
        ```
        
    - Verificar se uma tabela existe
        
        ```sql
        SELECT 
            TABSCHEMA,
            TABNAME,
            TYPE,
            CREATE_TIME
        FROM SYSCAT.TABLES
        WHERE TABNAME = 'WAS_TRAN_LOG_RI_POD0';
        ```
        
    - Evidência agendamento crontab
        
        ```sql
        {
          echo "=============================================="
          echo "        STATUS TIMEZONE SERVIDOR"
          echo "=============================================="
          timedatectl
          echo
          echo "=============================================="
          echo "        AGENDAMENTO STOP/START DB2"
          echo "=============================================="
          crontab -l | grep -Ei "stop-DB2.sh|start-DB2.sh"
          echo
        }
        ```
        
    - Verificar expiração de senhas
        
        ```sql
        --Completo
        for u in db2inst1; do
          echo "========== $u =========="
          chage -l "$u"
          echo
        done
        
        --Somente as informações importantes
        for u in odmdbusr aaedbusr aeosdbusr awsdocsusr awsdbusr ; do
          echo "========== $u =========="
          chage -l "$u" | grep -E "Last password change|Password expires|Account expires"
          echo
        done
        
        --Com informação de hostname e IP
        echo "======================================"
        echo "Servidor: $(hostname)"
        echo "IP: $(hostname -I | awk '{print $1}')"
        echo "======================================"
        for u in odmdbusr aaedbusr aeosdbusr awsdocsusr awsdbusr; do
          echo "Usuário: $u"
          chage -l "$u" | grep -E "Last password change|Password expires|Account expires"
          echo
        done
        
        --Alterar o last password change
        for user in odmdbusr aaedbusr aeosdbusr awsdocsusr awsdbusr
        do
           echo "Renovando expiração para: $user"
           sudo chage -d $(date +%Y-%m-%d) $user
        done
        
        ```
        
    - Verificar senhas usuários
        
        ```sql
        --Verificar requisitos de senha
        chage -l db2inst1
        
        --Alterar senha
        passwd db2inst1
        
        --Evidenciar alterações
        awk -F: '$3 >= 1000 {print $1}' /etc/passwd | while read u; do
            echo "==================== $u ===================="
            chage -l "$u" | grep -E "Last password change|Password expires|Password inactive|Account expires"
            echo ""
        done
        
        ```
        
    - Scripts criação databases DB2
        
        ```sql
        --Dropar os bancos
        #!/bin/bash
        
        # Lista de bancos a dropar
        BANCOS="ADPDB BAADB BANDB AWSDB STDDB UMSDB"
        
        # Alternativamente, você pode ler de um arquivo:
        # BANCOS=$(cat bancos_para_dropar.txt)
        
        for DB in $BANCOS
        do
          echo "============================================"
          echo "🔄 Iniciando processo para dropar banco: $DB"
        
          echo "→ Resetando conexão..."
          db2 connect reset
        
          echo "→ Tentando dropar o banco $DB..."
          db2 drop database $DB
        
          if [ $? -eq 0 ]; then
            echo "✅ Banco $DB dropado com sucesso."
          else
            echo "❌ Falha ao dropar banco $DB. Verifique se ele existe ou se a instância está correta."
          fi
        
          echo "--------------------------------------------"
        done
        
        ************************************************************
        RECRIAR DATABASES
        
        #!/bin/bash
        
        BANCOS="ADPDB BAADB BANDB AWSDB STDDB UMSDB"
        
        for DB in $BANCOS
        do
          echo "==> Criando diretórios para $DB"
          mkdir -p /db2/data/$DB
          mkdir -p /db2/logs/$DB
          mkdir -p /db2/backup/$DB
        
          echo "==> Gerando script de criação para $DB"
          cat <<EOF > createDatabase_${DB}.sql
        create database $DB automatic storage yes on '/db2/data/$DB' using codeset UTF-8 territory US pagesize 32768 encrypt;
        
        connect to $DB;
        
        CREATE USER TEMPORARY TABLESPACE USRTMPSPC1;
        
        UPDATE DB CFG FOR $DB USING NEWLOGPATH '/db2/logs/$DB' DEFERRED;
        UPDATE DB CFG FOR $DB USING LOGFILSIZ 16384 DEFERRED;
        UPDATE DB CFG FOR $DB USING LOGSECOND 64 IMMEDIATE;
        
        connect reset;
        EOF
        
          echo "==> Executando script para $DB"
          db2 -tvf createDatabase_${DB}.sql
        
          echo "Banco $DB criado com sucesso."
          echo "----------------------------------"
        done
        
        **************************************************************************
        PERMISSÕES
        
        #!/bin/bash
        
        # Lista de bancos
        BANCOS="GCDDB TOSDB DOSDB"  # ou leia de um arquivo: BANCOS=$(cat lista.txt)
        
        for DB in $BANCOS
        do
          USER="${DB}usr"  # Nome do usuário no padrão <banco>usr (tudo minúsculo)
        
          echo "============================================"
          echo "🎯 Conectando ao banco: $DB"
          db2 connect to ${DB^^}  # Usa nome do banco em maiúsculo
        
          echo "🔐 Concedendo permissões ao usuário: $USER"
        
          db2 "GRANT USE OF TABLESPACE VWDATA_TS TO USER $USER"
          db2 "GRANT USE OF TABLESPACE DOCSDB_TMP_TBS TO USER $USER"
          db2 "GRANT SELECT ON SYSIBM.SYSVERSIONS TO USER $USER"
          db2 "GRANT SELECT ON SYSCAT.DATATYPES TO USER $USER"
          db2 "GRANT SELECT ON SYSCAT.INDEXES TO USER $USER"
          db2 "GRANT SELECT ON SYSIBM.SYSDUMMY1 TO USER $USER"
          db2 "GRANT USAGE ON WORKLOAD SYSDEFAULTUSERWORKLOAD TO USER $USER"
          db2 "GRANT IMPLICIT_SCHEMA ON DATABASE TO USER $USER"
          db2 "CREATE SCHEMA $USER AUTHORIZATION $USER"
        
          db2 connect reset
          echo "✅ Permissões aplicadas com sucesso para $USER em $DB"
          echo "--------------------------------------------"
        done
        
        ************************************************************
        AJUSTES
        
        #!/bin/bash
        
        # Lista de bancos recém-criados
        DB_LIST=("CHOSDB" "DOCSDB" "GCDDB" "DOSDB" "TOSDB")
        
        # Caminhos padrão
        LOG_BASE="/db2/logs"
        BACKUP_BASE="/db2/backup"
        SCRIPT_DIR="/home/db2inst1/scripts/scripts_sustain"
        
        # Loop para aplicar configurações
        for DB in "${DB_LIST[@]}"; do
          echo -e "\n--- Configurando banco: $DB ---"
        
          # Garante diretório de log
          mkdir -p "$LOG_BASE/$DB/archive_log"
        
          # Parametrizações principais
          db2 update db cfg for $DB using TRACKMOD YES
          db2 update db cfg for $DB using CATALOGCACHE_SZ 1024
          db2 update db cfg for $DB using LOGARCHMETH1 DISK:$LOG_BASE/$DB/archive_log
          db2 update db cfg for $DB using LOGFILSIZ 32768
          db2 update db cfg for $DB using LOGPRIMARY 23
          db2 update db cfg for $DB using LOGSECOND 12
          db2 update db cfg for $DB using LOGBUFSZ 8192
          db2 update db cfg for $DB using STMT_CONC OFF
          db2 update db cfg for $DB using LOCKTIMEOUT 30
          db2 update db cfg for $DB using UTIL_HEAP_SZ 100000
          db2 update db cfg for $DB using CODEUNITS32
        
          echo -e "[OK] Parametrização aplicada em $DB."
        done
        
        # Stop da instância
        echo -e "\n--- Parando instância DB2 ---"
        $SCRIPT_DIR/stop-DB2.sh
        if [[ $? -ne 0 ]]; then
          echo "[ERRO] Falha ao parar a instância. Abortando script."
          exit 1
        fi
        
        # Start da instância
        echo -e "\n--- Iniciando instância DB2 ---"
        $SCRIPT_DIR/start-DB2.sh
        if [[ $? -ne 0 ]]; then
          echo "[ERRO] Falha ao iniciar a instância. Abortando script."
          exit 1
        fi
        
        # Backup completo após restart
        for DB in "${DB_LIST[@]}"; do
          echo -e "\n--- Iniciando backup do banco: $DB ---"
          mkdir -p "$BACKUP_BASE/$DB"
          db2 backup db $DB to $BACKUP_BASE/$DB compress
        done
        
        echo -e "\n✅ Configuração finalizada com sucesso para todos os bancos."
        
        ```
        
    - Verificar se há alguma coleta de estatística em andamento
        
        ```sql
        SELECT 
            EXECUTABLE_ID, 
            UOW_ID, 
            ACTIVITY_TYPE, 
            ACTIVITY_STATE, 
            TOTAL_ACT_TIME, 
            TOTAL_CPU_TIME
        FROM 
            TABLE(MON_GET_ACTIVITY(NULL, -2)) AS T
        WHERE 
            ACTIVITY_TYPE = 'STATISTICS';
        ```
        
    - Verificar indexes de um schema específico
        
        ```sql
        db2 -x "SELECT INDSCHEMA, INDNAME, TABNAME, TABSCHEMA, UNIQUERULE 
             FROM SYSCAT.INDEXES 
             WHERE TABSCHEMA = 'DWP' 
             AND INDSCHEMA <> 'SYSIBM' 
             ORDER BY INDSCHEMA, INDNAME" > index.txt
        ```
        
    - Matar sesões
        
        ```sql
        CALL SYSPROC.ADMIN_CMD('FORCE APPLICATION ()');
        ```
        
    - Query que mostra os campos, tipos, tamanho etc
        
        ```sql
        Query que mostra os campos, tipos, tamanho etc
        
        db2 "SELECT COLNAME, TYPENAME, LENGTH, NULLS FROM SYSCAT.COLUMNS WHERE TABNAME = 'TABELA'   AND TABSCHEMA = 'SCHEMA';"
        ```
        
    - Verificar os maiores ofensores
        
        ```sql
        --Query DB2 para identificar os 10 maiores ofensores 
        SELECT 
            STMT_TEXT AS SQL_TEXT,                  
            NUM_EXECUTIONS AS EXECUTION_COUNT,     
            TOTAL_CPU_TIME AS TOTAL_CPU_TIME_MS,   
            ROWS_READ,                             
            ROWS_RETURNED,                         
            LOCK_WAIT_TIME AS TOTAL_LOCK_WAIT_MS   
        FROM 
            TABLE(MON_GET_PKG_CACHE_STMT(NULL, NULL, NULL, -2)) AS S
        ORDER BY 
            TOTAL_CPU_TIME_MS DESC                 
        FETCH FIRST 10 ROWS ONLY;      
        ```
        
    - Verificar tabelas associadas a uma view
        
        ```sql
        SELECT * 
        FROM SYSCAT.VIEWS 
        WHERE VIEWNAME IN ('VW_DIM_FG_SITE_VIP', 'VW_DIM_SPECIAL_CLASS', 'VW_DIM_SICI_UF_TT');
        ```
        
    - Verificar tabelas de um schema específico
        
        ```sql
        db2 "SELECT 
            TABNAME 
        FROM 
            SYSCAT.TABLES 
        WHERE 
            TABSCHEMA = 'DWP' 
        ORDER BY 
            TABNAME;" > tables.txt
        ```
        
    - Verificar grants aplicados a um usuário (DB2-CLOUD)
        
        ```sql
        SELECT 
            TABSCHEMA,
            TABNAME,
            GRANTEE,
            --GRANTEETYPE,
            --GRANTOR,
            SELECTAUTH,
            INSERTAUTH,
            UPDATEAUTH,
            DELETEAUTH
        FROM 
            SYSCAT.TABAUTH 
        WHERE 
            TABSCHEMA IN ('DWP', 'GWNP', 'GWNCP')
            AND GRANTEE IN ('T3043735');
            
            ------
            
            SELECT 
            GRANTEE, 
            TABNAME, 
            TABSCHEMA, 
            ALTERAUTH, 
            CONTROLAUTH, 
            UPDATEAUTH, 
            SELECTAUTH
        FROM SYSCAT.TABAUTH
        WHERE GRANTEE = 'T3043735'
          AND TABNAME IN ('REPORTER_STATUS_COL', 'REPORTER_STATUS')
          AND TABSCHEMA IN ('GWNCP','GWNP');
        ```
        
    - Conceder permissões a um usuário (DB2-CLOUD)
        
        ```sql
        GRANT SELECT, INSERT, UPDATE, DELETE ON TABLE DWP.CNTRL_RSE_TICKETS TO USER IMPP;
        ```
        
    - Verificar colunas de uma tabela
        
        ```sql
        db2 "DESCRIBE TABLE DWP.CNTRL_LOAD_FULL_CEL_HOR"
        ```
        
    - Identificar procedures ou functions
        
        ```sql
        --Identificar procedures ou functions	
        db2 "SELECT ROUTINENAME, ROUTINESCHEMA, ROUTINETYPE
        FROM SYSCAT.ROUTINES
        WHERE ROUTINENAME = 'UPD_QUALITY_DATA';"
        ```
        
    - Identificar locks
        
        ```sql
        --Verificar lock
        db2 "SELECT lock_name, 
               hld_member, 
               lock_status,
               hld_application_handle FROM 
               TABLE (MON_GET_APPL_LOCKWAIT(NULL, -2))"
        ```
        
    - Listar as tablespaces
        
        ```sql
        db2 "LIST TABLESPACES SHOW DETAIL"
        ```
        
    - REORG
        
        ```sql
        db2 "reorgchk on table all" > Reorgchk_3011_depois.txt
        
        ```
        
    - Tabela DB2TOP
        
        ```sql
        DB2 Interactive Snapshot Monitor V2.0
        Use these keys to navigate:
        d - Database            l - Sessions            a - Agent
        t - Tablespaces         b - Bufferpools         T - Tables
        D - Dynamic SQL         U - Locks               m - Memory
        s - Statements          p - Members             u - Utilities
        A - HADR                F - Federation          B - Bottlenecks
        J - Skew monitor        q - Quit
        
        ```
        
    - Log de erros
        
        ```sql
        log de erros -> db2diag.log
        
        find / -iname "*.log" -mtime 0
        
        tail -n 1000 /optibm/db2iadm1/db2diag.log > parcial.log
        
        db2 get db cfg for BAWDB | grep -i path
        ```
        
    - P**rincipais declarações por tempo de execução (”AWR”)**
        
        ```sql
        ~/sqllib/samples/perf/db2mon.sh BAWDB > db2mon11_11.out
        ```
        
    - Identificar consultas que mais consomem CPU
        
        ```sql
        db2 "SELECT 
            SUBSTR(STMT_TEXT, 1, 100) AS QUERY_TEXT,
            NUM_EXECUTIONS,
            TOTAL_CPU_TIME,
            TOTAL_CPU_TIME / NUM_EXECUTIONS AS AVG_CPU_TIME_PER_EXEC,
            TOTAL_SECTION_SORTS,
            TOTAL_ACT_WAIT_TIME
        FROM 
            TABLE(MON_GET_PKG_CACHE_STMT(NULL, NULL, NULL, -2)) AS M
        ORDER BY 
            TOTAL_CPU_TIME DESC
        FETCH FIRST 10 ROWS ONLY"
        ```
        
- VIEWS
    - Evidência view recompilada
        
        ```sql
        SELECT object_name, status, last_ddl_time
        FROM user_objects
        WHERE object_type = 'VIEW' AND object_name = 'NOME_DA_VIEW';
        ```
        
    - Evidência view materializada recompilada
        
        ```sql
        SELECT owner, object_name, object_type, status, last_ddl_time
        FROM all_objects
        WHERE object_type = 'MATERIALIZED VIEW'
          AND object_name = 'MV_INVENTARIO_NGNIS';
        
        ```
        
    - Verificar se uma view existe no banco de dados
        
        ```sql
        SELECT owner
        FROM all_views
        WHERE view_name = 'NOME_DA_VIEW';
        
        SELECT owner, view_name
        FROM all_views
        WHERE view_name LIKE 'NOME_DA_VIEW';
        
        ```
        
    - Top Query com alto tempo decorrido
        
        ```sql
        --- Queries in last 1 hour
        Select
        module,parsing_schema_name,sql_id,sql_fulltext,
        to_char(last_active_time,'DD/MM/YY HH24:MI:SS' ),executions, elapsed_time/executions/1000/1000,
        rows_processed from gv$sql where last_active_time>sysdate-1/24
        and executions <> 0 order by elapsed_time/executions desc;
        ```
        
- Evidências de chamados (SOX)
    - Versão do banco de dados com IP e Hostname
        
        ```sql
        SET LINESIZE 200 
        COLUMN DATABASE_NAME FORMAT A20
        COLUMN HOSTNAME FORMAT A30
        COLUMN IP_ADDRESS FORMAT A20
        COLUMN DB_VERSION FORMAT A50
        SELECT 
            SYS_CONTEXT('USERENV', 'DB_NAME') AS DATABASE_NAME,
            SYS_CONTEXT('USERENV', 'SERVER_HOST') AS HOSTNAME,
            UTL_INADDR.GET_HOST_ADDRESS(SYS_CONTEXT('USERENV', 'SERVER_HOST')) AS IP_ADDRESS,
            (SELECT BANNER FROM V$VERSION WHERE ROWNUM = 1) AS DB_VERSION
        FROM DUAL;
        
        ```
        
    - Evidência criação/status usuários (com data e horário de criação)
        
        ```sql
        WITH inst AS (
          SELECT host_name
          FROM v$instance
        )
        SELECT
          u.username,
          u.account_status,
          u.profile,
          TO_CHAR(u.created, 'YYYY-MM-DD HH24:MI:SS') AS created_ts,
          inst.host_name                     AS db_hostname,
          utl_inaddr.get_host_address(inst.host_name) AS db_ip
        FROM dba_users u
        CROSS JOIN inst
        WHERE u.username IN ('ZABBIX')
        ORDER BY u.username;
        
        ****************************
        
        WITH inst AS (
          SELECT host_name
          FROM v$instance
        )
        SELECT
          u.username,
          u.account_status,
          u.profile,
          u.default_tablespace,
          TO_CHAR(u.created, 'YYYY-MM-DD HH24:MI:SS') AS created_ts,
         inst.host_name                     AS db_hostname,
          utl_inaddr.get_host_address(inst.host_name) AS db_ip
        FROM dba_users u
        CROSS JOIN inst
        WHERE u.username IN ('PMO_TOOLS')
        ORDER BY u.username;
        
        ******************************
        --Com privilégio em role específica
        
        WITH inst AS (
          SELECT host_name
          FROM v$instance
        )
        SELECT
            u.username,
            u.account_status,
            u.profile,
            rp.granted_role,
            TO_CHAR(u.created, 'YYYY-MM-DD HH24:MI:SS') AS created_ts,
            inst.host_name AS db_hostname,
            utl_inaddr.get_host_address(inst.host_name) AS db_ip
        FROM dba_users u
        LEFT JOIN dba_role_privs rp
               ON rp.grantee = u.username
        CROSS JOIN inst
        WHERE u.username IN ('HARDENING_RAW')
        ORDER BY u.username, rp.granted_role;
        ```
        
    - Evidência criação/lock/status usuários
        
        ```sql
        set linesize 200
        set pagesize 10000
        col username format a20
        col account_status format a15
        col profile format a25
        col created format a10
        col expiry_date format a15
        col lock_date format a10
        SELECT username, account_status, profile, created, expiry_date, lock_date 
        FROM dba_users 
        WHERE username = 'F8004221';
        ```
        
    - Evidência COLISEU
        
        ```sql
        set markup html on spool on
        spool VERIFY_FUNCTION_12C_COLISEU.html
        set echo on
        --############ DADOS DO BANCO ############
        
        SELECT NAME BANCO FROM V$DATABASE;
        select name PDB from v$pdbs;
        SELECT to_char(systimestamp,'dd-mm-YYYY HH24:MM:SS') DATA ,UTL_INADDR.get_host_address IP_ADDRESS, UTL_INADDR.get_host_name HOSTNAME from dual;
        
        -- ############ funcao "VERIFY_FUNCTION_12C" ############
        
        SELECT TEXT FROM ALL_SOURCE WHERE TYPE = 'FUNCTION' AND NAME = 'VERIFY_FUNCTION_12C'
        ORDER BY LINE;
        
        --##FIM##
        spool off
        set markup html off
        ```
        
    - Evidência Netflow
        
        ```sql
        set markup html on spool on
        spool COLISEU.html
        set echo on
        --############ DADOS DO BANCO ############
        
        SELECT NAME BANCO FROM V$DATABASE;
        select name PDB from v$pdbs;
        SELECT to_char(systimestamp,'dd-mm-YYYY HH24:MM:SS') DATA ,UTL_INADDR.get_host_address IP_ADDRESS, UTL_INADDR.get_host_name HOSTNAME from dual;
        
        --############ Nome de todos os profiles ############
        
        SELECT DISTINCT PROFILE FROM dba_profiles
        ORDER BY PROFILE;
        
        -- ############ Todos os parametros de configuracao de senha que existem nos profiles ############
        
        SELECT DISTINCT RESOURCE_NAME
        FROM dba_profiles
        WHERE RESOURCE_TYPE = 'PASSWORD'
        ORDER BY RESOURCE_NAME;
        
        -- ############ Todos os parametros por profile ############
        
        SELECT PROFILE,RESOURCE_NAME, LIMIT FROM dba_profiles
        WHERE RESOURCE_TYPE = 'PASSWORD'
        ORDER BY PROFILE;
        
        -- ############ Relacao profile por users com status "OPEN" ############
        
        SELECT username, profile
        FROM dba_users
        WHERE account_status = 'OPEN'
        ORDER BY profile;
        
        -- ############ funcao "VERIFY_FUNCTION_12C" ############
        
        SELECT TEXT FROM ALL_SOURCE WHERE TYPE = 'FUNCTION' AND NAME = 'VERIFY_FUNCTION_12C'
        ORDER BY LINE;
        
        --##FIM##
        spool off
        set markup html off
        ```
        
    - Evidência tablespace
        
        ```sql
        
        set echo off
        set termout on
        set heading on
        set feedback off
        set verify off
        set pagesize 1000
        set linesize 180
        tti 'Tablespaces'
        col "TOTAL(GB)"      for 99999.999
        col "USAGE(GB)"      for 99999.999
        col "FREE(GB)"       for 99999.999
        col "EXTENSIBLE(GB)" for 9999999.999
        col "FREE PCT %"     for 999.99
        col "DTF_NAUTO"      for 3
        col "DTF_AUTO"       for 3
        col INFO             for a100
        SELECT
            d.tablespace_name "NAME",
            d.contents "TYPE",
            ROUND(NVL(a.bytes / 1024 / 1024 / 1024, 0), 3) "TOTAL(GB)",
            ROUND(NVL(f.bytes, 0) / 1024 / 1024 / 1024, 3) "FREE(GB)",
            ROUND(NVL(a.bytes - NVL(f.bytes, 0), 0) / 1024 / 1024 / 1024, 3) "USAGE(GB)",
            ROUND(NVL((NVL(f.bytes, 0) / a.bytes) * 100, 0), 3) "FREE PCT %",
            ROUND(NVL(a.ARTACAK, 0) / 1024 / 1024 / 1024, 3) "EXTENSIBLE(GB)",
            a.NOTO AS DTF_NAUTO,
            a.OTO  AS DTF_AUTO
        FROM sys.dba_tablespaces d
        LEFT JOIN (
            SELECT tablespace_name,
                   SUM(bytes) bytes,
                   SUM(DECODE(autoextensible, 'YES', MAXbytes - bytes, 0)) ARTACAK,
                   COUNT(DECODE(autoextensible, 'NO', 0)) NOTO,
                   COUNT(DECODE(autoextensible, 'YES', 0)) OTO
            FROM dba_data_files
            GROUP BY tablespace_name
        ) a ON d.tablespace_name = a.tablespace_name
        LEFT JOIN (
            SELECT tablespace_name, SUM(bytes) bytes
            FROM dba_free_space
            GROUP BY tablespace_name
        ) f ON d.tablespace_name = f.tablespace_name
        WHERE NOT (d.extent_management LIKE 'LOCAL' AND d.contents LIKE 'TEMPORARY')
        UNION ALL
        SELECT
            d.tablespace_name "NAME",
            d.contents "TYPE",
            ROUND(NVL(a.bytes / 1024 / 1024 / 1024, 0), 3) "TOTAL(GB)",
            ROUND(NVL(a.bytes - NVL(t.bytes, 0), 0) / 1024 / 1024 / 1024, 3) "FREE(GB)",
            ROUND(NVL(t.bytes, 0) / 1024 / 1024 / 1024, 3) "USAGE(GB)",
            ROUND(NVL((NVL(a.bytes - NVL(t.bytes, 0), 0) / a.bytes) * 100, 0), 3) "FREE PCT %",
            ROUND(NVL(a.ARTACAK, 0) / 1024 / 1024 / 1024, 3) "EXTENSIBLE(GB)",
            a.NOTO AS DTF_NAUTO,
            a.OTO  AS DTF_AUTO
        FROM sys.dba_tablespaces d
        LEFT JOIN (
            SELECT tablespace_name,
                   SUM(bytes) bytes,
                   SUM(DECODE(autoextensible, 'YES', MAXbytes - bytes, 0)) ARTACAK,
                   COUNT(DECODE(autoextensible, 'NO', 0)) NOTO,
                   COUNT(DECODE(autoextensible, 'YES', 0)) OTO
            FROM dba_temp_files
            GROUP BY tablespace_name
        ) a ON d.tablespace_name = a.tablespace_name
        LEFT JOIN (
            SELECT tablespace_name, SUM(bytes_used) bytes
            FROM v$temp_extent_pool
            GROUP BY tablespace_name
        ) t ON d.tablespace_name = t.tablespace_name
        WHERE d.extent_management LIKE 'LOCAL'
          AND d.contents LIKE 'TEMPORARY%'
        ORDER BY 6 ASC NULLS LAST;
        SELECT 'DATA/HORA : ' || TO_CHAR(SYSDATE, 'DD/MM/YYYY HH24:MI:SS') AS INFO FROM dual
        UNION ALL
        SELECT 'BANCO     : ' || SYS_CONTEXT('USERENV','DB_NAME') FROM dual
        UNION ALL
        SELECT 'HOSTNAME  : ' || SYS_CONTEXT('USERENV','HOST') FROM dual
        UNION ALL
        SELECT 'IP        : ' || NVL(SYS_CONTEXT('USERENV','IP_ADDRESS'),
                                    UTL_INADDR.get_host_address) FROM dual;
                                    
        ---------
        
        --Evidenciar tablespace específica
                                    
        set echo off
        set termout on
        set heading on
        set feedback off
        set verify off
        set pagesize 100
        set linesize 180
        col NAME        for a25
        col DB_NAME     for a12
        col HOSTNAME    for a25
        col IP          for a16
        col "TOTAL(GB)" for 99999.999
        col "FREE(GB)"  for 99999.999
        col "USAGE(GB)" for 99999.999
        col "FREE PCT%" for 999.99
        SELECT
            d.tablespace_name AS NAME,
            SYS_CONTEXT('USERENV','DB_NAME') AS DB_NAME,
            SYS_CONTEXT('USERENV','SERVER_HOST') AS HOSTNAME,
            UTL_INADDR.get_host_address(
                SYS_CONTEXT('USERENV','SERVER_HOST')
            ) AS IP,
            ROUND(a.total_bytes/1024/1024/1024,3) AS "TOTAL(GB)",
            ROUND(NVL(f.free_bytes,0)/1024/1024/1024,3) AS "FREE(GB)",
            ROUND((a.total_bytes-NVL(f.free_bytes,0))/1024/1024/1024,3)
                AS "USAGE(GB)",
            ROUND((NVL(f.free_bytes,0)/a.total_bytes)*100,2)
                AS "FREE PCT%"
        FROM dba_tablespaces d
        JOIN (
            SELECT tablespace_name,
                   SUM(bytes) total_bytes
            FROM dba_data_files
            GROUP BY tablespace_name
        ) a
        ON d.tablespace_name = a.tablespace_name
        LEFT JOIN (
            SELECT tablespace_name,
                   SUM(bytes) free_bytes
            FROM dba_free_space
            GROUP BY tablespace_name
        ) f
        ON d.tablespace_name = f.tablespace_name
        WHERE d.tablespace_name = 'NI_TBS';
        
        ```
        
    - Evidência usuários nativos
        
        ```sql
        set line 500
        COLUMN USERNAME FORMAT A25
        COLUMN ACCOUNT_STATUS FORMAT A20
        COLUMN PROFILE FORMAT A20
        COLUMN CREATED FORMAT A20
        COLUMN ORACLE_MAINTAINED FORMAT A30
        SELECT USERNAME, ACCOUNT_STATUS, PROFILE, CREATED, ORACLE_MAINTAINED
        FROM DBA_USERS
        WHERE ORACLE_MAINTAINED = 'Y';
        ```
        
    - Evidência bkps
        
        ```sql
        set markup html on spool on
        SPOOL EV_BKP_SPAZIO.html
        
        SET ECHO ON;
        SELECT NAME AS BANCO FROM V$DATABASE;
        SELECT NAME AS PDB FROM V$PDBS;
        SELECT TO_CHAR(SYSTIMESTAMP, 'dd-mm-YYYY HH24:MI:SS') AS DATA,
               UTL_INADDR.GET_HOST_ADDRESS AS IP_ADDRESS,
               UTL_INADDR.GET_HOST_NAME AS HOSTNAME
        FROM dual;
        SELECT session_key, input_type, jd.status,
               TO_CHAR(start_time, 'yyyy-mm-dd hh24:mi') AS start_time,
               TO_CHAR(end_time, 'yyyy-mm-dd hh24:mi') AS end_time,
               output_bytes_display, time_taken_display
        FROM v$rman_backup_job_details jd
        WHERE (start_time) > SYSDATE - 34
        ORDER BY session_key ASC;
        
        SPOOL OFF
        set markup html off
        ```
        
    - Evidência segurança usuários TestFactory
        
        [TIM_DB_Evidências Segurança SOX TestFactory.docx](ICARO%20SCRIPT/TIM_DB_Evidncias_Segurana_SOX_TestFactory.docx)
        
    - Evidência PROFILE
        
        ```jsx
        SELECT PROFILE, RESOURCE_NAME, RESOURCE_TYPE, LIMIT from dba_profiles where profile like '%DEFAULT_TIM%' and RESOURCE_NAME in ('IDLE_TIME','FAILED_LOGIN_ATTEMPTS','PASSWORD_LIFE_TIME','PASSWORD_REUSE_MAX','PASSWORD_VERIFY_FUNCTION','PASSWORD_LOCK_TIME','PASSWORD_GRACE_TIME','INACTIVE_ACCOUNT_TIME')
        ORDER BY PROFILE;
        ```
        
- AWR
    - Gerar relatório AWR e ADDM
        
        ```sql
        For Single Instance Environment: @$ORACLE_HOME/rdbms/admin/awrrpt.sql
        
        For Oracle RAC Environment : @$ORACLE_HOME/rdbms/admin/awrgrpt.sql
        
        Gerar ADDM
        
        @$ORACLE_HOME/rdbms/admin/addmrpt.sql
        
        ```
        
    - Identificar queries que mais geraram I/O
        
        ```sql
        SELECT *
        FROM (
          SELECT 
            sql_id,
            SUM(disk_reads_delta) disk_reads,
            SUM(executions_delta) execs,
            ROUND(SUM(elapsed_time_delta)/1000000,2) elapsed_sec
          FROM dba_hist_sqlstat
          WHERE snap_id BETWEEN 86375 AND 86378
          GROUP BY sql_id
          ORDER BY disk_reads DESC
        )
        WHERE ROWNUM <= 10;
        ```
        
- BANCO DE DADOS
    - Verificar se um servidor é primário ou standby
        
        ```sql
        
        SELECT name, database_role, open_mode
        FROM v$database;
        
        *******************************************
        
        SELECT 
            INSTANCE_NAME,
            STATUS,
            DATABASE_STATUS,
            DATABASE_ROLE
        FROM 
            V$INSTANCE, 
            V$DATABASE;
        ```
        
    - Verificar os principais agressores no banco de dados
        
        ```sql
        SET LINESIZE 200
        SET PAGESIZE 100
        SET TRIMOUT ON
        SET TRIMSPOOL ON
        SET WRAP OFF
        COLUMN "Aggressor Type"     FORMAT A20
        COLUMN sid                  FORMAT 99999
        COLUMN serial#              FORMAT 99999
        COLUMN user_id              FORMAT 99999
        COLUMN machine              FORMAT A20
        COLUMN program              FORMAT A25
        COLUMN event                FORMAT A30
        COLUMN sql_id               FORMAT A13
        COLUMN blocking_session     FORMAT 99999
        COLUMN "Metric"             FORMAT 999999
        WITH top_cpu AS (
            SELECT *
            FROM (
                SELECT
                    s.session_id AS sid,
                    s.session_serial# AS serial#,
                    s.user_id,
                    s.machine,
                    s.program,
                    s.event,
                    s.sql_id,
                    s.blocking_session,
                    COUNT(*) AS cpu_seconds
                FROM
                    v$active_session_history s
                WHERE
                    s.sample_time BETWEEN SYSDATE - INTERVAL '1' HOUR AND SYSDATE
                    AND s.session_state = 'ON CPU'
                    AND s.session_type = 'FOREGROUND'
                GROUP BY
                    s.session_id, s.session_serial#, s.user_id, s.machine, s.program, s.event, s.sql_id, s.blocking_session
                ORDER BY
                    cpu_seconds DESC
            )
            WHERE ROWNUM <= 10
        ),
        top_io AS (
            SELECT *
            FROM (
                SELECT
                    s.session_id AS sid,
                    s.session_serial# AS serial#,
                    s.user_id,
                    s.machine,
                    s.program,
                    s.event,
                    s.sql_id,
                    s.blocking_session,
                    COUNT(*) AS io_requests
                FROM
                    v$active_session_history s
                WHERE
                    s.sample_time BETWEEN SYSDATE - INTERVAL '1' HOUR AND SYSDATE
                    AND s.session_state = 'WAITING'
                    AND s.session_type = 'FOREGROUND'
                    AND s.event LIKE '%db file%'
                GROUP BY
                    s.session_id, s.session_serial#, s.user_id, s.machine, s.program, s.event, s.sql_id, s.blocking_session
                ORDER BY
                    io_requests DESC
            )
            WHERE ROWNUM <= 10
        ),
        top_wait AS (
            SELECT *
            FROM (
                SELECT
                    s.session_id AS sid,
                    s.session_serial# AS serial#,
                    s.user_id,
                    s.machine,
                    s.program,
                    s.event,
                    s.sql_id,
                    s.blocking_session,
                    COUNT(*) AS wait_seconds
                FROM
                    v$active_session_history s
                WHERE
                    s.sample_time BETWEEN SYSDATE - INTERVAL '1' HOUR AND SYSDATE
                    AND s.session_state = 'WAITING'
                    AND s.session_type = 'FOREGROUND'
                GROUP BY
                    s.session_id, s.session_serial#, s.user_id, s.machine, s.program, s.event, s.sql_id, s.blocking_session
                ORDER BY
                    wait_seconds DESC
            )
            WHERE ROWNUM <= 10
        )
        SELECT
            'CPU Aggressor' AS "Aggressor Type",
            tc.sid,
            tc.serial#,
            tc.user_id,
            tc.machine,
            tc.program,
            tc.event,
            tc.sql_id,
            tc.blocking_session,
            tc.cpu_seconds AS "Metric"
        FROM
            top_cpu tc
        UNION ALL
        SELECT
            'IO Aggressor' AS "Aggressor Type",
            ti.sid,
            ti.serial#,
            ti.user_id,
            ti.machine,
            ti.program,
            ti.event,
            ti.sql_id,
            ti.blocking_session,
            ti.io_requests AS "Metric"
        FROM
            top_io ti
        UNION ALL
        SELECT
            'Wait Time Aggressor' AS "Aggressor Type",
            tw.sid,
            tw.serial#,
            tw.user_id,
            tw.machine,
            tw.program,
            tw.event,
            tw.sql_id,
            tw.blocking_session,
            tw.wait_seconds AS "Metric"
        FROM
            top_wait tw
        ORDER BY
            "Metric" DESC;
        
        ```
        
    - Verificar jobs que envolve tabelas
        
        ```sql
        SELECT
            ash.sample_time,
            ash.session_id,
            ash.session_serial#,
            --ash.username,
            ash.program,
            ash.machine,
            ash.module,
            ash.action,
            ash.sql_id
        FROM gv$active_session_history ash
        WHERE ash.sql_opname IN ('INSERT', 'UPDATE')
          AND ash.current_obj# = (
                SELECT object_id
                FROM dba_objects
                WHERE owner = 'AUX_TABLES'
                  AND object_name = 'TB_NRM_MONITORACAO_CLASSIF'
                  AND object_type = 'TABLE'
        ```
        
    - Ver dados da sessão ofensora
        
        ```sql
        --Identificar a sessão
        SELECT s.sid,
               s.serial#,
               s.username,
               s.osuser,
               s.machine,
               s.program,
               s.status,
               s.logon_time
        FROM v$session s
        WHERE s.sid = &SID
          AND s.serial# = &SERIAL;
        
        --Ver o SQL em execução
        SELECT s.sid,
               s.serial#,
               q.sql_id,
               q.sql_text
        FROM v$session s
        JOIN v$sql q
             ON s.sql_id = q.sql_id
        WHERE s.sid = &SID
          AND s.serial# = &SERIAL;
          
        --Último SQL executado (mesmo que não esteja em execução)
         SELECT s.sid,
               s.serial#,
               q.sql_id,
               q.sql_text
        FROM v$session s
        JOIN v$sql q
             ON s.prev_sql_id = q.sql_id
        WHERE s.sid = &SID
          AND s.serial# = &SERIAL;
        
        ```
        
    - V**erificar tempo de atividade de uma instância**
        
        ```sql
        SET LINESIZE 360
        SET PAGESIZE 1000
        SET TRIMSPOOL ON
        COLUMN INSTANCE_NAME      FORMAT A20
        COLUMN PDB_NAME           FORMAT A20
        COLUMN PDB_OPEN_MODE      FORMAT A15
        COLUMN PDB_RESTRICTED     FORMAT A10
        COLUMN HOST_NAME          FORMAT A35
        COLUMN IP_ADDRESS         FORMAT A15
        COLUMN STATUS             FORMAT A10
        COLUMN DATABASE_STATUS    FORMAT A15
        COLUMN DATABASE_ROLE      FORMAT A12
        COLUMN LAST_RESTART       FORMAT A20
        SELECT 
            i.INSTANCE_NAME,
            p.NAME AS PDB_NAME,
            p.OPEN_MODE     AS PDB_OPEN_MODE,
            p.RESTRICTED    AS PDB_RESTRICTED,
            i.HOST_NAME,
            UTL_INADDR.get_host_address(i.HOST_NAME) AS IP_ADDRESS,
            i.STATUS,
            i.DATABASE_STATUS,
            d.DATABASE_ROLE,
            TO_CHAR(i.STARTUP_TIME, 'DD-MON-YYYY HH24:MI:SS') AS LAST_RESTART
        FROM 
            GV$INSTANCE i
            CROSS JOIN V$DATABASE d
            CROSS JOIN V$PDBS p
        ORDER BY 
            i.INSTANCE_NAME, p.NAME;
            
            
        ***************************
        
        Sem PDB
        
        SET LINESIZE 300
        SET PAGESIZE 1000
        COLUMN INSTANCE_NAME FORMAT A20
        COLUMN HOST_NAME     FORMAT A40
        COLUMN IP_ADDRESS    FORMAT A15
        COLUMN STATUS        FORMAT A12
        COLUMN DATABASE_STATUS FORMAT A18
        COLUMN DATABASE_ROLE FORMAT A20
        COLUMN OPEN_MODE     FORMAT A15
        COLUMN LAST_RESTART  FORMAT A20
        SELECT
            i.INSTANCE_NAME,
            NULL AS PDB_NAME,
            i.HOST_NAME,
            UTL_INADDR.get_host_address(i.HOST_NAME) AS IP_ADDRESS,
            i.STATUS,
            i.DATABASE_STATUS,
            d.DATABASE_ROLE,
            d.OPEN_MODE,
            TO_CHAR(i.STARTUP_TIME, 'DD-MON-YYYY HH24:MI:SS') AS LAST_RESTART
        FROM
            GV$INSTANCE i
            CROSS JOIN V$DATABASE d
        ORDER BY
            i.INSTANCE_NAME;
            
        
        ```
        
    - V**erificar se o banco está aberto**
        
        ```sql
        SELECT OPEN_MODE FROM V$DATABASE;
        
        ou 
        
        select database_role, name, open_mode from v$database;
        
        SELECT 
            d.NAME,
            d.DATABASE_ROLE,
            d.OPEN_MODE,
            i.HOST_NAME AS SERVER_HOSTNAME,
            UTL_INADDR.GET_HOST_ADDRESS AS SERVER_IP
        FROM 
            V$DATABASE d,
            V$INSTANCE i;
        ```
        
    - Verificar os principais agressores no banco de dados versão 11G
        
        ```sql
        --- Banco 11G agressores
        WITH top_cpu AS (
            SELECT *
            FROM (
                SELECT
                    s.session_id AS sid,
                    s.session_serial# AS serial#,
                    s.user_id,
                    s.machine,
                    s.program,
                    s.event,
                    s.sql_id,
                    s.blocking_session,
                    COUNT(*) AS cpu_seconds
                FROM
                    v$active_session_history s
                WHERE
                    s.sample_time BETWEEN SYSDATE - INTERVAL '1' HOUR AND SYSDATE
                    AND s.session_state = 'ON CPU'
                    AND s.session_type = 'FOREGROUND'  -- Exclui sessões de background
                GROUP BY
                    s.session_id, s.session_serial#, s.user_id, s.machine, s.program, s.event, s.sql_id, s.blocking_session
                ORDER BY
                    cpu_seconds DESC
            )
            WHERE ROWNUM <= 10  -- Limitar a 10 resultados
        ),
        top_io AS (
            SELECT *
            FROM (
                SELECT
                    s.session_id AS sid,
                    s.session_serial# AS serial#,
                    s.user_id,
                    s.machine,
                    s.program,
                    s.event,
                    s.sql_id,
                    s.blocking_session,
                    COUNT(*) AS io_requests
                FROM
                    v$active_session_history s
                WHERE
                    s.sample_time BETWEEN SYSDATE - INTERVAL '1' HOUR AND SYSDATE
                    AND s.session_state = 'WAITING'
                    AND s.session_type = 'FOREGROUND'  -- Exclui sessões de background
                    AND s.event LIKE '%db file%'
                GROUP BY
                    s.session_id, s.session_serial#, s.user_id, s.machine, s.program, s.event, s.sql_id, s.blocking_session
                ORDER BY
                    io_requests DESC
            )
            WHERE ROWNUM <= 10  -- Limitar a 10 resultados
        ),
        top_wait AS (
            SELECT *
            FROM (
                SELECT
                    s.session_id AS sid,
                    s.session_serial# AS serial#,
                    s.user_id,
                    s.machine,
                    s.program,
                    s.event,
                    s.sql_id,
                    s.blocking_session,
                    COUNT(*) AS wait_seconds
                FROM
                    v$active_session_history s
                WHERE
                    s.sample_time BETWEEN SYSDATE - INTERVAL '1' HOUR AND SYSDATE
                    AND s.session_state = 'WAITING'
                    AND s.session_type = 'FOREGROUND'  -- Exclui sessões de background
                GROUP BY
                    s.session_id, s.session_serial#, s.user_id, s.machine, s.program, s.event, s.sql_id, s.blocking_session
                ORDER BY
                    wait_seconds DESC
            )
            WHERE ROWNUM <= 10  -- Limitar a 10 resultados
        )
        SELECT
            'CPU Aggressor' AS "Aggressor Type",
            tc.sid,
            tc.serial#,
            tc.user_id,
            tc.machine,
            tc.program,
            tc.event,
            tc.sql_id,
            tc.blocking_session,
            tc.cpu_seconds AS "Metric"
        FROM
            top_cpu tc
        UNION ALL
        SELECT
            'IO Aggressor' AS "Aggressor Type",
            ti.sid,
            ti.serial#,
            ti.user_id,
            ti.machine,
            ti.program,
            ti.event,
            ti.sql_id,
            ti.blocking_session,
            ti.io_requests AS "Metric"
        FROM
            top_io ti
        UNION ALL
        SELECT
            'Wait Time Aggressor' AS "Aggressor Type",
            tw.sid,
            tw.serial#,
            tw.user_id,
            tw.machine,
            tw.program,
            tw.event,
            tw.sql_id,
            tw.blocking_session,
            tw.wait_seconds AS "Metric"
        FROM
            top_wait tw
        ORDER BY
            "Metric" DESC;
        ```
        
    - Verificar SCAN OSSDB
        
        ```sql
        srvctl status listener ; srvctl status scan ; crsctl check cluster - all
        ```
        
    - Verificar os logs do banco DLFALHAS
        
        ```sql
        --Top 10 erros mais frequentes nas últimas 24h (Oracle 11g)
        SELECT *
        FROM (
            SELECT 
                SUBSTR(MESSAGE_TEXT, 1, 1000) AS MENSAGEM,
                COUNT(*) AS OCORRENCIAS
            FROM 
                V$DIAG_ALERT_EXT
            WHERE 
                UPPER(MESSAGE_TEXT) LIKE '%ERROR%'
                AND ORIGINATING_TIMESTAMP > SYSDATE - 1
            GROUP BY 
                SUBSTR(MESSAGE_TEXT, 1, 1000)
            ORDER BY 
                OCORRENCIAS DESC
        )
        WHERE ROWNUM <= 10;
        
        --Verifica o último reinício da instância
        SELECT 
            TO_CHAR(ORIGINATING_TIMESTAMP, 'DD/MM/YYYY HH24:MI:SS') AS DATA_EVENTO,
            SUBSTR(MESSAGE_TEXT, 1, 1000) AS MESSAGE_TEXT
        FROM 
            V$DIAG_ALERT_EXT
        WHERE 
            UPPER(MESSAGE_TEXT) LIKE '%INSTANCE%'
            AND ORIGINATING_TIMESTAMP > SYSDATE - 3;
            
            -------------
            Demais bancos
            
            SELECT SUBSTR (MESSAGE_TEXT, 1, 300) MESSAGE_TEXT, COUNT (*) cnt
        FROM X$DBGALERTEXT
        WHERE MESSAGE_TEXT LIKE '%SHUTDOWN%'
        AND CAST (ORIGINATING_TIMESTAMP AS DATE) > SYSDATE - 1
        GROUP BY SUBSTR (MESSAGE_TEXT, 1, 300);
        
        SELECT SUBSTR(MESSAGE_TEXT, 1, 1000) AS MESSAGE_TEXT,
               COUNT(*) AS CNT
          FROM V$DIAG_ALERT_EXT
         WHERE (MESSAGE_TEXT LIKE '%ORA-%' OR UPPER(MESSAGE_TEXT) LIKE '%ERROR%')
           AND ORIGINATING_TIMESTAMP > SYSDATE - 1
         GROUP BY SUBSTR(MESSAGE_TEXT, 1, 1000);
        
        SELECT SUBSTR(MESSAGE_TEXT, 1, 1000) AS MESSAGE_TEXT,
               COUNT(*) AS CNT
          FROM V$DIAG_ALERT_EXT
         WHERE UPPER(MESSAGE_TEXT) LIKE '%SHUTDOWN%' 
           AND ORIGINATING_TIMESTAMP > SYSDATE - 1
         GROUP BY SUBSTR(MESSAGE_TEXT, 1, 1000);
        ```
        
    - Verificar quantidade de processos e sessões
        
        ```sql
        COLUMN resource_name FORMAT A15 HEADING "Resource Name"
        COLUMN current_utilization FORMAT 99999 HEADING "Current|Utilization"
        COLUMN max_utilization FORMAT 99999 HEADING "Max|Utilization"
        COLUMN limit_value FORMAT 99999 HEADING "Limit|Value"
        SELECT 
            resource_name, 
            current_utilization, 
            max_utilization, 
            limit_value
        FROM 
            v$resource_limit
        WHERE 
            resource_name IN ('processes', 'sessions');
        
        ```
        
    - Verificar versão do banco com hostname e IP
        
        ```sql
        SELECT
            d.name AS nome_banco,
            TO_CHAR(d.created, 'DD/MM/YYYY HH24:MI:SS') AS data_criacao,
            i.host_name AS nome_servidor,
            UTL_INADDR.GET_HOST_ADDRESS(i.host_name) AS ip_servidor,
            (
                SELECT banner_full
                FROM v$version
                WHERE banner_full LIKE 'Oracle Database%'
            ) || ' - ' ||
            (
                SELECT banner_full
                FROM v$version
                WHERE banner_full LIKE 'Version%'
            ) AS versao_oracle
        FROM v$database d
        JOIN v$instance i ON 1=1;
        
        ```
        
    - Verificar o texto sql baseado no SQL_ID
        
        ```sql
        SELECT
            sql_id,
            sql_fulltext
        FROM
            v$sql
        WHERE
            sql_id = '7vy0ft23afh5m';
        ```
        
    - Consulta detalhada PKG
        
        ```sql
        --Consulta detalhada PKG
        SELECT
          u.username              AS usuario,
          ash.sample_time         AS data_execucao,
          s.sql_text              AS texto_sql,
          ash.sql_id,
          CASE
            WHEN s.sql_text LIKE '%PKG_GENOMA.%' THEN 'UPDATEGERSITESADDRESSES'
            ELSE 'NAO_IDENTIFICADO'
          END                    AS procedimento_identificado,
          ash.program,
          ash.module,
          ash.machine,
          ash.instance_number
        FROM dba_hist_active_sess_history ash
        JOIN dba_hist_sqltext s ON ash.sql_id = s.sql_id
        JOIN dba_users u ON ash.user_id = u.user_id
        WHERE s.sql_text LIKE '%PKG_GENOMA%'
          AND ash.sample_time >= SYSDATE - 80
        ORDER BY ash.sample_time DESC;
        
        --Detalhes do SQL ID
        SELECT 
            SQL_FULLTEXT, 
            SQL_ID, 
            FIRST_LOAD_TIME, 
            PARSING_SCHEMA_NAME, 
            SERVICE, 
            MODULE, 
            ACTION
        FROM v$sql
        WHERE sql_id = '9adf2gy149sa3';
        
        --Quem mais usou essa procedure
        SELECT *
        FROM dba_hist_active_sess_history
        WHERE sql_id = '9adf2gy149sa3';
        
        --Verifica se há SQL recente que utilizaram o package
        SELECT *
        FROM v$sql
        WHERE sql_text LIKE '%PKG_GENOMA%';
        
        --Verifica dependências
        SELECT *
        FROM dba_dependencies
        WHERE referenced_name = 'PKG_GENOMA'
          AND referenced_type = 'PACKAGE';
        ```
        
    - Verificar os schemas excluindo SYS
        
        ```sql
        SELECT username
        FROM dba_users
        WHERE account_status = 'OPEN'
          AND default_tablespace NOT IN ('SYSTEM','SYSAUX')
          AND username NOT IN (
            'SYS', 'SYSTEM', 'OUTLN', 'DBSNMP', 'MGMT_VIEW', 'SYSMAN',
            'FLOWS_FILES', 'APEX_PUBLIC_USER', 'APPQOSSYS', 'AUDSYS', 'CTXSYS',
            'DBSFWUSER', 'DBSNMP', 'DIP', 'DVF', 'DVSYS', 'GSMADMIN_INTERNAL',
            'GSMCATUSER', 'GSMUSER', 'LBACSYS', 'MDDATA', 'MDSYS', 'OJVMSYS',
            'OLAPSYS', 'ORDDATA', 'ORDPLUGINS', 'ORDSYS', 'REMOTE_SCHEDULER_AGENT',
            'SI_INFORMTN_SCHEMA', 'SPATIAL_CSW_ADMIN_USR', 'SPATIAL_WFS_ADMIN_USR',
            'SYS', 'SYSBACKUP', 'SYSDG', 'SYSKM', 'SYSTEM', 'WMSYS', 'XDB', 'XS$NULL'
          )
        ORDER BY username;
        ```
        
    - Verificar os cursores em uso por sessão
        
        ```sql
        SELECT 
            S.SID,
            S.SERIAL#,
            S.USERNAME,
            S.PROGRAM,
            S.MODULE,
            S.STATUS,
            COUNT(*) AS open_cursors
        FROM 
            V$OPEN_CURSOR C
        JOIN 
            V$SESSION S ON C.SID = S.SID
        GROUP BY 
            S.SID, S.SERIAL#, S.USERNAME, S.PROGRAM, S.MODULE, S.STATUS
        ORDER BY 
            open_cursors DESC;
        ```
        
    - Verificar os 10 maiores consumidores de memória
        
        ```sql
        --Verificar processos
        ps -eo pid,cmd,rsz --sort=-rsz | head -n 10
        
        --Verificar processos com maior consumo de SWAP
        grep VmSwap /proc/<PID>/status
        ```
        
    - Verificar consumo de CPU
        
        ```sql
        --Verificar processos
        ps -eo pid,cmd,%cpu --sort=-%cpu | head -n 10
        
        ps -eo pid,user,cmd,%cpu,%mem --sort=-%cpu | head -n 10
        
        --Verificar processos com maior consumo de SWAP
        grep VmSwap /proc/<PID>/status
        ```
        
    - Verificar o uso de cursores
        
        ```sql
        select
        'session_cached_cursors' parameter,
        lpad(value, 5) value,
        decode(value, 0, ' n/a', to_char(100 * used / value, '990') || '%') usage
        from
        ( select
        max(s.value) used
        from
        v$statname n,
        v$sesstat s
        where
        n.name = 'session cursor cache count' and
        s.statistic# = n.statistic#
        ),
        ( select
        value
        from
        v$parameter
        where
        name = 'session_cached_cursors'
        )
        union all
        select
        'open_cursors',
        lpad(value, 5),
        to_char(100 * used / value, '990') || '%'
        from
        ( select
        max(sum(s.value)) used
        from
        v$statname n,
        v$sesstat s
        where
        n.name in ('opened cursors current') and
        s.statistic# = n.statistic#
        group by
        s.sid
        ),
        ( select
        value
        from
        v$parameter
        where
        name = 'open_cursors');
        ```
        
    - Verificar todos os schemas do banco
        
        ```sql
        SELECT username 
        FROM dba_users 
        WHERE oracle_maintained = 'N'  -- Exclui schemas nativos do Oracle
        AND username NOT IN (
            'SYS', 'SYSTEM', 'OUTLN', 'DBSNMP', 'APPQOSSYS', 'AUDSYS', 
            'CTXSYS', 'DVSYS', 'GGSYS', 'LBACSYS', 'MDSYS', 'OLAPSYS', 
            'ORDDATA', 'ORDPLUGINS', 'ORDSYS', 'SI_INFORMTN_SCHEMA', 
            'WMSYS', 'XDB', 'OJVMSYS', 'GSMADMIN_INTERNAL'
        ) 
        ORDER BY username;
        ```
        
    - Identificar ocorrências recentes do erro ORA-00600
        
        ```sql
        SELECT MESSAGE_TEXT, ORIGINATING_TIMESTAMP
        FROM V$DIAG_ALERT_EXT
        WHERE MESSAGE_TEXT LIKE '%ORA-00600%'
        ORDER BY ORIGINATING_TIMESTAMP DESC;
        ```
        
    - Identificar consultas com alto número de parse calls
        
        ```sql
        SELECT 
            PARSE_CALLS,
            EXECUTIONS,
            SQL_TEXT,
            PARSE_CALLS / (EXECUTIONS + 1) AS parse_ratio
        FROM 
            V$SQL
        WHERE 
            PARSE_CALLS > 100
        ORDER BY 
            PARSE_CALLS DESC;
        ```
        
    - Identificar consultas que causam hard parses
        
        ```sql
        SELECT 
            EXECUTIONS,
            PARSE_CALLS,
            SQL_TEXT,
            PARSE_CALLS - EXECUTIONS AS hard_parses
        FROM 
            V$SQL
        WHERE 
            (PARSE_CALLS - EXECUTIONS) > 0
        ORDER BY 
            hard_parses DESC;
        ```
        
    - Análise das sessões com open cursor elevado
        
        ```sql
        --identificar a sessão pelo SQL_ID
        SELECT 
            S.SID,
            S.SERIAL#,
            S.USERNAME,
            S.MACHINE,
            S.PROGRAM,
            S.MODULE,
            S.ACTION,
            S.LOGON_TIME,
            P.SQL_ID,
            P.SQL_TEXT
        FROM 
            V$SESSION S
        JOIN 
            V$SQL P ON S.SQL_ID = P.SQL_ID
        WHERE 
            P.SQL_ID = 'fdawsrqyuap79';
            
          
        ```
        
    - Verificar o tamanho do banco de dados
        
        ```sql
        col "Database Size" format a20
        col "Free space" format a20
        col "Used space" format a20
        select round(sum(used.bytes) / 1024 / 1024 / 1024 ) || ' GB' "Database Size"
        , round(sum(used.bytes) / 1024 / 1024 / 1024 ) -
        round(free.p / 1024 / 1024 / 1024) || ' GB' "Used space"
        , round(free.p / 1024 / 1024 / 1024) || ' GB' "Free space"
        from (select bytes
        from v$datafile
        union all
        select bytes
        from v$tempfile
        union all
        select bytes
        from v$log) used
        , (select sum(bytes) as p
        from dba_free_space) free
        group by free.p
        /
        ```
        
    - Verificar os logfiles
        
        ```sql
        SET LINESIZE 200
        SET PAGESIZE 100
        COL "GROUP" FORMAT 999
        COL "STATUS" FORMAT A10
        COL "NAME" FORMAT A50
        COL "SIZE_MB" FORMAT 9999
        
        -- Consulta
        SELECT 
            L.GROUP# AS "GROUP",
            L.STATUS AS "STATUS",
            LF.MEMBER AS "NAME",
            L.BYTES / 1024 / 1024 AS "SIZE_MB"
        FROM 
            V$LOG L
        JOIN 
            V$LOGFILE LF ON L.GROUP# = LF.GROUP#
        UNION ALL
        SELECT 
            SL.GROUP# AS "GROUP",
            SL.STATUS AS "STATUS",
            LF.MEMBER AS "NAME",
            SL.BYTES / 1024 / 1024 AS "SIZE_MB"
        FROM 
            V$STANDBY_LOG SL
        JOIN 
            V$LOGFILE LF ON SL.GROUP# = LF.GROUP#
        ORDER BY "GROUP";
        
        --Essa query mostra os online e os standby
        
        set lines 200 pages 200
        col l.group# format a5
        col l.sequence# format a5
        col l.THREAD# format a5
        col mbytes format 9999999999
        col l.members format a3
        col l.archived format a5
        col l.status# format a10
        col first_time format a20
        col redo_filename format a80
        select l.group#,  l.sequence#, l.THREAD#,
              (l.bytes/1024/1024) mbytes, l.members,
              l.archived, l.status,       
              to_char(l.first_time,'dd/mm/yyyy hh24:mi:ss') first_time,
              f.member as redo_filename
        from  V$LOG l
        join  V$LOGFILE f
          on  l.group# = f.group#
        order by 1;
        ```
        
    - Verificar processos em espera (COLISEU)
        
        ```sql
        Para ver os processos em espera:
        ps -eo state,pid,cmd | grep "^D"
        
        para matar automaticamente:
        OBS: Use com cuidado!!
        ps -eo state,pid --no-headers | awk '$1=="D" {print $2}' | xargs -r kill -9
        ```
        
    - Restore banco de dados
        
        ```sql
        
        CAT_TIM =
          (DESCRIPTION =
            (ADDRESS = (PROTOCOL = TCP)(HOST = 10.192.67.211)(PORT = 1521))
            (CONNECT_DATA =
              (SERVER = DEDICATED)
              (SERVICE_NAME = cat_TIM)
            )
          )
        
        MSMTPD =
          (DESCRIPTION =
            (ADDRESS = (PROTOCOL = TCP)(HOST = 10.192.12.13)(PORT = 1521))
            (CONNECT_DATA =
              (SERVER = DEDICATED)
              (SERVICE_NAME = MSMTPD)
            )
          )
        
        run{
        set DBID=1925848344;
        allocate channel 'ch1' DEVICE TYPE 'SBT_TAPE' PARMS  'ENV=(BLKSIZE=1048576,SBT_LIBRARY=/opt/dpsapps/dbappagent/lib/amd64/libddboostora.so,CONFIG_FILE=/opt/dpsapps/dbappagent/config/oracle_ddbda.cfg)';
        restore controlfile;
        }
        
        *.db_name='MSMTPD'
        *.cluster_database=FALSE
        *.use_large_pages=TRUE
        *.audit_file_dest='/u01/app/oracle/admin/MSMTPD/adump'
        *.pga_aggregate_target=2G
        *.pga_aggregate_limit=4G
        *.sga_max_size=12G
        *.sga_target=12G
        *.undo_tablespace=UNDOTBS1
        *.db_create_file_dest='/u03/oradata'
        *.db_create_online_log_dest_1='/u03/oradata/MSMTPD/onlinelog'
        *.db_create_online_log_dest_2='/arch/oracle/fast_recovery_area/MSMTPD/onlinelog'
        *.db_recovery_file_dest='/arch/oracle/fast_recovery_area'
        *.db_recovery_file_dest_size=140G
        
        DBID=1925848344)
        
        output file name=/u03/oradata/MSMTPD/onlinelog/MSMTPD/controlfile/o1_mf_mo16yv2m_.ctl
        output file name=/arch/oracle/fast_recovery_area/MSMTPD/onlinelog/MSMTPD/controlfile/o1_mf_mo16ywyk_.ctl
        
        *.control_files='/u03/oradata/MSMTPD/onlinelog/MSMTPD/controlfile/o1_mf_mo16yv2m_.ctl','/arch/oracle/fast_recovery_area/MSMTPD/onlinelog/MSMTPD/controlfile/o1_mf_mo16ywyk_.ctl'
        
        run{
        allocate channel 'ch1' DEVICE TYPE 'SBT_TAPE' PARMS  'ENV=(BLKSIZE=1048576,SBT_LIBRARY=/opt/dpsapps/dbappagent/lib/amd64/libddboostora.so,CONFIG_FILE=/opt/dpsapps/dbappagent/config/oracle_ddbda.cfg)';
        allocate channel 'ch2' DEVICE TYPE 'SBT_TAPE' PARMS  'ENV=(BLKSIZE=1048576,SBT_LIBRARY=/opt/dpsapps/dbappagent/lib/amd64/libddboostora.so,CONFIG_FILE=/opt/dpsapps/dbappagent/config/oracle_ddbda.cfg)';
        allocate channel 'ch3' DEVICE TYPE 'SBT_TAPE' PARMS  'ENV=(BLKSIZE=1048576,SBT_LIBRARY=/opt/dpsapps/dbappagent/lib/amd64/libddboostora.so,CONFIG_FILE=/opt/dpsapps/dbappagent/config/oracle_ddbda.cfg)';
        allocate channel 'ch4' DEVICE TYPE 'SBT_TAPE' PARMS  'ENV=(BLKSIZE=1048576,SBT_LIBRARY=/opt/dpsapps/dbappagent/lib/amd64/libddboostora.so,CONFIG_FILE=/opt/dpsapps/dbappagent/config/oracle_ddbda.cfg)';
        restore database;
        SWITCH DATAFILE ALL;
        SWITCH TEMPFILE ALL;
        RECOVER database;
        }
        
        create spfile from pfile='/home/oracle/init.ora';
        
        startup mount
        
        SET LINESIZE 200
        SET PAGESIZE 50
        COLUMN Operation_Name FORMAT A40
        COLUMN Target_Object FORMAT A30
        COLUMN Percentage_Complete FORMAT 999.99
        COLUMN Start_Time FORMAT A20
        COLUMN Max_Time_Remaining_In_Min FORMAT 9999
        COLUMN Time_Spent_In_Min FORMAT 9999
        SELECT 
            opname AS Operation_Name,
            target AS Target_Object,
            ROUND((sofar / totalwork) * 100, 2) AS Percentage_Complete,
            TO_CHAR(start_time, 'YYYY-MM-DD HH24:MI:SS') AS Start_Time,
            CEIL(time_remaining / 60) AS Max_Time_Remaining_In_Min,
            FLOOR(elapsed_seconds / 60) AS Time_Spent_In_Min
        FROM 
            v$session_longops
        WHERE 
            sofar != totalwork
            AND totalwork <> 0
        ORDER BY 
            start_time DESC;
        ```
        
    - Count de schemas
        
        ```sql
        SELECT DECODE(GROUPING(a.owner),1,'All Owners',a.owner) OWNER
              ,COUNT(CASE WHEN a.object_type = 'TABLE'             THEN 1 ELSE NULL END) "TABLE"
              ,COUNT(CASE WHEN a.object_type = 'TABLE PARTITION'   THEN 1 ELSE NULL END) "PTABLE"
              ,COUNT(CASE WHEN a.object_type = 'INDEX'             THEN 1 ELSE NULL END) "INDEX"
              ,COUNT(CASE WHEN a.object_type = 'INDEX PARTITION'   THEN 1 ELSE NULL END) "PINDEX"
              ,COUNT(CASE WHEN a.object_type = 'PACKAGE'           THEN 1 ELSE NULL END) "PACKAGE"
              ,COUNT(CASE WHEN a.object_type = 'PACKAGE BODY'      THEN 1 ELSE NULL END) "PCK_BODY"
              ,COUNT(CASE WHEN a.object_type = 'SEQUENCE'          THEN 1 ELSE NULL END) "SEQ"
              ,COUNT(CASE WHEN a.object_type = 'TRIGGER'           THEN 1 ELSE NULL END) "TRIG"
              ,COUNT(CASE WHEN a.object_type = 'PROCEDURE'         THEN 1 ELSE NULL END) "PROC"
              ,COUNT(CASE WHEN a.object_type = 'FUNCTION'          THEN 1 ELSE NULL END) "FUNC"
              ,COUNT(CASE WHEN a.object_type = 'VIEW'              THEN 1 ELSE NULL END) "VIEW"
              ,COUNT(CASE WHEN a.object_type = 'TYPE'              THEN 1 ELSE NULL END) "TYPE"
              ,COUNT(CASE WHEN a.object_type = 'SYNONYM'           THEN 1 ELSE NULL END) "SYN"
              ,COUNT(CASE WHEN a.object_type = 'LOB'               THEN 1 ELSE NULL END) "LOB"
              ,COUNT(CASE WHEN a.object_type = 'JOB'               THEN 1 ELSE NULL END) "JOB"
              ,COUNT(CASE WHEN a.object_type = 'MATERIALIZED VIEW' THEN 1 ELSE NULL END) "MTVIEW"
              ,COUNT(CASE WHEN a.object_type = 'DATABASE LINK'     THEN 1 ELSE NULL END) "DBLINK"
              ,COUNT(CASE
                     WHEN a.object_type NOT IN ('PACKAGE','TABLE','INDEX','SEQUENCE','TRIGGER','PACKAGE BODY','PROCEDURE','FUNCTION','VIEW','TYPE','SYNONYM','LOB','JOB', 'DATABASE LINK','MATERIALIZED VIEW') THEN 1
                     ELSE NULL END) "Other"
              ,COUNT(CASE WHEN 1 = 1                               THEN 1 ELSE NULL END) "Total"
        FROM dba_objects a
        where owner in
                   ('ODMDCNW','ODMRES','ODMRESHML')
        GROUP BY rollup( a.owner);
        ```
        
    - Verificar tamanho do schema
        
        ```sql
        --Count objetos
        SELECT object_type, COUNT(*) AS total
        FROM dba_objects
        WHERE owner = 'MSTR_USER'
        GROUP BY object_type
        ORDER BY object_type;
        
        --Tamanho total do schema
        SELECT 
            segment_type,
            COUNT(*) AS total_objetos,
            ROUND(SUM(bytes)/1024/1024/1024, 2) AS tamanho_gb
        FROM dba_segments
        WHERE owner = 'MJOLNIR'
        GROUP BY segment_type
        ORDER BY segment_type;
        
        --Tamanho por tipo de objeto (TABLE, INDEX, LOB, etc.)
        SELECT 
            segment_type,
            COUNT(*) AS total_objetos,
            ROUND(SUM(bytes)/1024/1024, 2) AS tamanho_mb
        FROM dba_segments
        WHERE owner = 'MSTR_USER'
        GROUP BY segment_type
        ORDER BY segment_type;
        
        --Contagem + tamanho
        SELECT 
            o.object_type,
            COUNT(DISTINCT o.object_name) AS total_objetos,
            ROUND(SUM(s.bytes)/1024/1024/1024, 2) AS tamanho_gb
        FROM dba_objects o
        LEFT JOIN dba_segments s
               ON o.owner = s.owner
              AND o.object_name = s.segment_name
        WHERE o.owner = 'MJOLNIR'
        GROUP BY o.object_type
        ORDER BY o.object_type;
        
        --Top 10 maiores tabelas (somente dados)
        SELECT *
        FROM (
            SELECT 
                segment_name AS tabela,
                ROUND(SUM(bytes)/1024/1024/1024,2) AS tamanho_gb
            FROM dba_segments
            WHERE owner = 'MJOLNIR'
              AND segment_type = 'TABLE'
            GROUP BY segment_name
            ORDER BY tamanho_gb DESC
        )
        WHERE ROWNUM <= 10;
        
        --Dados vs Índices
        SELECT 
            CASE 
                WHEN segment_type LIKE 'TABLE%' THEN 'DADOS'
                WHEN segment_type LIKE 'INDEX%' THEN 'INDICES'
                WHEN segment_type LIKE 'LOB%' THEN 'LOB'
                ELSE 'OUTROS'
            END AS categoria,
            ROUND(SUM(bytes)/1024/1024/1024,2) AS tamanho_gb
        FROM dba_segments
        WHERE owner = 'MJOLNIR'
        GROUP BY 
            CASE 
                WHEN segment_type LIKE 'TABLE%' THEN 'DADOS'
                WHEN segment_type LIKE 'INDEX%' THEN 'INDICES'
                WHEN segment_type LIKE 'LOB%' THEN 'LOB'
                ELSE 'OUTROS'
            END
        ORDER BY tamanho_gb DESC;
        
        --Crescimento mensal por segmento (últimos 12 meses)
        SELECT
            o.owner,
            o.object_name,
            o.subobject_name,
            TO_CHAR(s.begin_interval_time, 'YYYY-MM') AS mes,
            ROUND(SUM(ss.space_used_delta)/1024/1024,2) AS crescimento_mb
        FROM dba_hist_seg_stat ss
        JOIN dba_hist_seg_stat_obj o 
             ON ss.obj# = o.obj#
        JOIN dba_hist_snapshot s 
             ON ss.snap_id = s.snap_id
            AND ss.dbid = s.dbid
            AND ss.instance_number = s.instance_number
        WHERE o.owner = 'MJOLNIR'
          AND s.begin_interval_time >= ADD_MONTHS(TRUNC(SYSDATE,'MM'), -12)
        GROUP BY 
            o.owner,
            o.object_name,
            o.subobject_name,
            TO_CHAR(s.begin_interval_time, 'YYYY-MM')
        HAVING SUM(ss.space_used_delta) > 0
        ORDER BY mes, crescimento_mb DESC;
        ```
        
    - Criar relação de confiança RAC
        
        ```sql
        ssh-keygen -t rsa
        
        ssh-copy-id grid@vaoracleraclab01
        ```
        
    - Verificar tabelas dos schemas
        
        ```sql
        SELECT 
            segment_name AS table_name,
            SUM(bytes) / 1024 / 1024 AS size_mb
        FROM 
            dba_segments
        WHERE 
            owner = 'EXEMPLO' 
            AND segment_type = 'TABLE'
        GROUP BY 
            segment_name
        ORDER BY 
            size_mb DESC;
        ```
        
    - Verificar cursors
        
        ```sql
        select
        'session_cached_cursors' parameter,
        lpad(value, 5) value,
        decode(value, 0, ' n/a', to_char(100 * used / value, '990') || '%') usage
        from
        ( select
        max(s.value) used
        from
        v$statname n,
        v$sesstat s
        where
        n.name = 'session cursor cache count' and
        s.statistic# = n.statistic#
        ),
        ( select
        value
        from
        v$parameter
        where
        name = 'session_cached_cursors'
        )
        union all
        select
        'open_cursors',
        lpad(value, 5),
        to_char(100 * used / value, '990') || '%'
        from
        ( select
        max(sum(s.value)) used
        from
        v$statname n,
        v$sesstat s
        where
        n.name in ('opened cursors current') and
        s.statistic# = n.statistic#
        group by
        s.sid
        ),
        ( select
        value
        from
        v$parameter
        where
        name = 'open_cursors');
        ```
        
    - Verificação de processos consumindo memória do sistema
        
        ```sql
        ps aux --sort=-%mem | head -n 10
        ```
        
    - Export de dados
        
        ```sql
        Verificar volume para criar o diretório
        df -h
        
        Criar caminho do diretório 
        mkdir -p /u01/app/oracle/exports (exemplo)
        
        Conceder permissões (grant)
        chmod 777 /u01/app/oracle/exports
        chown oracle:oinstall /u01/app/oracle/exports
        
        Criar o diretório e conceder permissão
        
        CREATE OR REPLACE DIRECTORY EXPORT AS '/u01/app/oracle/exports';
        
        GRANT READ, WRITE ON DIRECTORY EXPORT TO SYSTEM;
        
        Verificar se tem ou não PDB
        Verificar conexão com PDB (caso tenha) - tnsping nomepdb (caso não tenha criar conexão)
        Após configurar conexão efetuar teste (sqlplus system/senha@nomepdb
        
        (sem pdb) nohup expdp \"/ as sysdba\" directory=EXPORT dumpfile=EXP%U.dmp logfile=exp.log schemas=ODMDCNW,ODMRES,ODMRESHML exclude=statistics parallel=8 &
        
        (com pdb) nohup impdp system/"senha"@nomepdb directory=IMPORT dumpfile=EXP%U.dmp logfile=imp.log schemas=ODMDCNW,ODMRES,ODMRESHML &
        
        nohup expdp "C##MDOXDBA/angryDragon#87"@GOVERNANCAPMO directory=DUMP dumpfile=EXP%U.dmp logfile=exp.log schemas=STATUS_REPORT exclude=statistics parallel=8 &
        
        --Meio alternativo (PDB/SYSDBA)
        impdp \"SYS/"267wEVqU-kr3Hq"@MSTOLTH1 AS SYSDBA\" DIRECTORY=DATAPUMP DUMPFILE=MSTR_USR%U.dmp LOGFILE=MSTR_IMPORT.log SCHEMAS=MSTR_USER REMAP_SCHEMA=MSTR_USER:MSTR_HOM PARALLEL=1 CLUSTER=N
        
        ```
        
    - Verificação rápida do estado da memória do sistema
        
        ```sql
        free -h; ps -e -o pid,comm,vsz,rss --sort -size | head -n 11 | awk '{print $1, $2, $3/1024 " MB (VSZ)", $4/1024 " MB (RSS)"}'
        ```
        
    - Criar um diretório no Oracle
        
        ```sql
        ---- Criar diretorio no ORACLE
        
        select * from DBA_DIRECTORIES where directory_name='EXPDP_DIR';
        
        CREATE DIRECTORY IMPDP_DIR AS '/dump';
        
        GRANT READ, WRITE ON DIRECTORY EXPDP_DIR TO SYSTEM;
        ```
        
    - Executar restore via script
        
        ```sql
        ---- Criar arquivo 
        
        vi restore_stdby.rcv
        
        run{
        restore database;
        recover database;
        }
        
        Exemplo comando: 
        nohup rman target sys/"senha"@service_name CATALOG rman/senha@nome catálogo cmdfile=/home/oracle/restorestdby.rcv log=/home/oracle/restorestdby.log &
        
        ```
        
    - Verificar coleta de estatísticas Oracle
        
        ```sql
        -> Objetos do banco
        EXEC dbms_stats.gather_fixed_objects_stats();
        exec dbms_stats.gather_system_stats();
        EXEC DBMS_STATS.GATHER_DICTIONARY_STATS;
        
        -> Executar de acordo com a necessidade
        EXEC DBMS_STATS.gather_schema_stats('SYS', estimate_percent => 20, cascade => TRUE, degree =>4);
        EXEC DBMS_STATS.gather_schema_stats('SYSTEM', estimate_percent => 20, cascade => TRUE, degree =>4);
        select systimestamp from dual;
        
        -> Dados de todo o banco
        EXEC DBMS_STATS.GATHER_DATABASE_STATS(cascade => TRUE, method_opt => 'FOR ALL COLUMNS SIZE AUTO' );
        select systimestamp from dual;
        ```
        
    - Para alterar o conjunto de caracteres NLS_CHARACTERSET
        
        ```sql
        -- 1. Verificar o conjunto de caracteres atual
        SELECT value FROM NLS_DATABASE_PARAMETERS WHERE parameter = 'NLS_CHARACTERSET';
        
        -- 2. Realizar backup (usando expdp como exemplo)
        expdp system/your_password full=Y directory=backup_dir dumpfile=backup_before_charset_change.dmp logfile=backup_before_charset_change.log
        
        -- 3. Executar CSSCAN
        csscan system/your_password FULL=Y TOCHAR=AL32UTF8 LOG=csscan.log
        
        ***Antes de efetuar a troca verificar: show parameter JOB_QUEUE_PROCESSES; e show parameter AQ_TM_PROCESSES;***
        
        -- 4. Alterar o conjunto de caracteres
        SHUTDOWN IMMEDIATE;
        STARTUP MOUNT;
        ALTER SYSTEM ENABLE RESTRICTED SESSION;
        ALTER SYSTEM SET JOB_QUEUE_PROCESSES=0;
        ALTER SYSTEM SET AQ_TM_PROCESSES=0;
        ALTER DATABASE OPEN;
        ALTER DATABASE CHARACTER SET USE WE8MSWIN1252;-- (Caso dê erro utilizar INTERNAL_USE)
        ALTER SYSTEM DISABLE RESTRICTED SESSION;
        
        -- 5. Reiniciar o banco de dados
        SHUTDOWN IMMEDIATE;
        STARTUP;
        
        ```
        
    - V**erificar sessões ativas**
        
        ```sql
        SELECT
            COUNT(*) AS active_sessions
        FROM
            V$SESSION;
        ```
        
    - V**erificar tnsnames**
        
        ```sql
        vi $ORACLE_HOME/network/admin/tnsnames.ora
        
        ```
        
    - V**erificar hora atual do banco de dados**
        
        ```sql
        SET LINESIZE 200
        SET PAGESIZE 500
        COLUMN "Data e Hora" FORMAT A80
        COLUMN "Hostname" FORMAT A30
        COLUMN "IP" FORMAT A20
        SELECT 
            'Data e Hora Atual: ' || TO_CHAR(SYSDATE, 'DD/MM/YYYY HH24:MI:SS') AS "Data e Hora",
            'Hostname: ' || SYS_CONTEXT('USERENV', 'HOST') AS "Hostname",
            'IP: ' || UTL_INADDR.get_host_address(SYS_CONTEXT('USERENV', 'HOST')) AS "IP"
        FROM 
            dual;
        ```
        
    - V**erificar caminho alert log**
        
        ```sql
        SELECT * FROM V$DIAG_INFO WHERE NAME = 'Diag Alert';
        ```
        
    - V**erificar sessões bloqueadas**
        
        ```sql
        select s1.username || '@' || s1.machine
        || ' ( SID=' || s1.sid || ' ) is blocking '
        || s2.username || '@' || s2.machine || ' ( SID=' || s2.sid || ' ) ' AS blocking_status
        from gv$lock l1, gv$session s1, gv$lock l2, gv$session s2
        where s1.sid=l1.sid and s2.sid=l2.sid
        and l1.BLOCK=1 and l2.request > 0
        and l1.id1 = l2.id1
        and l2.id2 = l2.id2 ;
        
        ///
        
        Verificar sessões por snapshot
        
        select count(*), sql_id,sql_child_number,session_state,blocking_session_status,event,wait_class
          from DBA_HIST_ACTIVE_SESS_HISTORY 
          where snap_id between 98241 and 98243
          and event like '%cursor: pin S%'
          group by sql_id,sql_child_number,session_state,blocking_session_status,event,wait_class;
          
        ///
        
        select p2raw, p2/power(16,8) blocking_sid, p1 mutex_id, sid blocked_sid  
           from v$session   
           where event like 'cursor:%'   
           and state='WAITING';
        ```
        
    - V**erificar tamanhos físico e lógico**
        
        ```sql
        Select (select round(sum(bytes)/1024/1024/1024,2)  from dba_data_files) as "Tamanho fisico (GB)"
                ,(select round(sum(bytes)/1024/1024/1024,2)  from dba_segments) as "Tamanho Logico(GB)"
        from dual;
        
        ----
        SELECT 
            (SELECT ROUND(SUM(bytes)/1024/1024/1024, 2) FROM dba_data_files) AS "Tamanho Físico (GB)",
            (SELECT ROUND(SUM(bytes)/1024/1024/1024, 2) FROM dba_segments) AS "Tamanho Lógico (GB)",
            (SELECT ROUND(SUM(bytes)/1024/1024/1024, 2) FROM dba_data_files) +
            (SELECT ROUND(SUM(bytes)/1024/1024/1024, 2) FROM dba_segments) AS "Tamanho Total (GB)"
        FROM dual;
        
        ----
        SET LINESIZE 200
        SET PAGESIZE 50
        COLUMN "Tamanho Fisico (GB)" FORMAT 999999990.00 JUSTIFY RIGHT
        COLUMN "Tamanho Logico (GB)" FORMAT 999999990.00 JUSTIFY RIGHT
        COLUMN "Tamanho Total (GB)" FORMAT 999999990.00 JUSTIFY RIGHT
        COLUMN "Hostname do Servidor" FORMAT A29
        COLUMN "IP do Servidor" FORMAT A20
        SELECT
            (SELECT ROUND(SUM(bytes)/1024/1024/1024, 2) FROM dba_data_files) AS "Tamanho Fisico (GB)",
            (SELECT ROUND(SUM(bytes)/1024/1024/1024, 2) FROM dba_segments) AS "Tamanho Logico (GB)",
            (SELECT ROUND(SUM(bytes)/1024/1024/1024, 2) FROM dba_data_files) +
            (SELECT ROUND(SUM(bytes)/1024/1024/1024, 2) FROM dba_segments) AS "Tamanho Total (GB)",
            (SELECT host_name FROM v$instance) AS "Hostname do Servidor",
            (SELECT UTL_INADDR.get_host_address FROM dual) AS "IP do Servidor"
        FROM dual;
        
        ----- Com a versão do banco
        
        SELECT
            ROUND((SELECT SUM(bytes)/1024/1024/1024 FROM dba_data_files), 2) AS "Tamanho Fisico (GB)",
            ROUND((SELECT SUM(bytes)/1024/1024/1024 FROM dba_segments), 2) AS "Tamanho Logico (GB)",
            ROUND((SELECT SUM(bytes)/1024/1024/1024 FROM dba_data_files), 2) +
            ROUND((SELECT SUM(bytes)/1024/1024/1024 FROM dba_segments), 2) AS "Tamanho Total (GB)",
            i.host_name AS "Hostname do Servidor",
            i.instance_name AS "Instância",
            i.version AS "Versão",
            UTL_INADDR.get_host_address AS "IP do Servidor"
        FROM 
            v$instance i;
        ```
        
    - V**erificar profiles do banco**
        
        ```sql
        SELECT PROFILE, RESOURCE_NAME, RESOURCE_TYPE, LIMIT
        FROM DBA_PROFILES
        ORDER BY LIMIT DESC;
        ```
        
    - Verificar uso atual de SGA
        
        ```sql
        select round(used.bytes /1024/1024 ,2) used_mb
        , round(free.bytes /1024/1024 ,2) free_mb
        , round(tot.bytes /1024/1024 ,2) total_mb
        from (select sum(bytes) bytes
        from v$sgastat
        where name != 'free memory') used
        , (select sum(bytes) bytes
        from v$sgastat
        where name = 'free memory') free
        , (select sum(bytes) bytes
        from v$sgastat) tot
        /
        ```
        
    - Verificar uso de PGA por sessão
        
        ```sql
        set lines 2000
        SELECT SID, b.NAME, ROUND(a.VALUE/(1024*1024),2) MB FROM
        v$sesstat a, v$statname b
        WHERE (NAME LIKE '%session uga memory%' OR NAME LIKE '%session pga memory%')
        AND a.statistic# = b.statistic# order by ROUND(a.VALUE/(1024*1024),2) desc
        ```
        
    - Configuração e Consulta de Estimativas de Desempenho da SGA e PGA
        
        ```sql
        set lines 200 pages 200
        col  estd_physical_reads for 99999999999999999999999
        SELECT sga_size, sga_size_factor, estd_db_time, estd_physical_reads FROM   V$SGA_TARGET_ADVICE;
        
        set lines 200 pages 200
        col  estd_physical_reads for 99999999999999999999999
        col  estd_extra_bytes_rw for 99999999999999999999999
        SELECT pga_target_for_estimate/1024/1024 "PGA Target Estimado", pga_target_factor, estd_time, estd_extra_bytes_rw
        FROM   V$PGA_TARGET_ADVICE;
        ```
        
    - V**erificar consumo de CPU por sessão (MACROS)**
        
        ```sql
        SELECT 
            s.sql_id,
            s.sql_text,
            s.executions,
            s.buffer_gets,
            s.cpu_time,
            s.elapsed_time,
            (SELECT username FROM v$session WHERE sql_id = s.sql_id AND ROWNUM = 1) AS username
        FROM 
            (SELECT 
                sql_id,
                sql_text,
                executions,
                buffer_gets,
                cpu_time,
                elapsed_time,
                ROW_NUMBER() OVER (ORDER BY cpu_time DESC) AS rn
             FROM 
                v$sql
             WHERE 
                cpu_time > 0
                AND sql_text IS NOT NULL
            ) s
        WHERE 
            rn <= 10;
            
            Após retornar o resultado, salvar o maior SQL_ID ofensor.
            Verificar os SNAPID do AWR
            
            Ainda na linha de comando, executar os passos abaixo.
            
            Rodar macro Tuning Task 1 - Iniciar (preencher o que for solicitado)
            Rodar macro Tuning Task 2
            Rodar macro Tuning Task 3
            Rodar macro Tuning Task 4 - (não rodar a DMS_SQLTUNE.PX_PROFILE)
            
            Após toda a análise
            Rodar macro Tuning Task 5
            
             
        ```
        
    - V**erificar informações de memória da CPU do servidor de banco de dados**
        
        ```sql
        set pagesize 200
        set lines 200
        col name for a21
        col stat_name for a25
        col value for a13
        col comments for a56
        select STAT_NAME,to_char(VALUE) as VALUE ,COMMENTS from v$osstat where
        stat_name IN ('NUM_CPUS','NUM_CPU_CORES','NUM_CPU_SOCKETS')
        union
        select STAT_NAME,VALUE/1024/1024/1024 || ' GB' ,COMMENTS from
        v$osstat where stat_name IN ('PHYSICAL_MEMORY_BYTES');
        ```
        
    - V**erificar a última CPU aplicada em um banco de dados**
        
        ```sql
        col VERSION for a15;
        col COMMENTS for a50;
        col ACTION for a10;
        set lines 500;
        
        select ACTION,VERSION,COMMENTS,BUNDLE_SERIES from registry$history;
        
        ```
        
    - V**erificar se o banco é RAC**
        
        ```sql
        set linesize 200 pagesize 10000
        
        SELECT NAME,VERSION,CURRENTLY_USED,FEATURE_INFO
        FROM DBA_FEATURE_USAGE_STATISTICS
        WHERE NAME = 'Real Application Clusters (RAC)';
        ```
        
    - Comando para parar / subir o banco full ( em RAC ) (SRVCTL)
        
        ```sql
        srvctl stop database -d <database_name> -o immediate
        srvctl start database -d <database_name>
        srvctl status database -d <database_name>
        ```
        
    - V**erificar se o banco é DATAGUARD**
        
        ```sql
        SELECT NAME,VERSION,CURRENTLY_USED,FEATURE_INFO
        FROM DBA_FEATURE_USAGE_STATISTICS
        WHERE NAME = 'Data Guard';
        ```
        
    - V**erificar tamanho de um schema**
        
        ```sql
        select owner Schema, sum(bytes/1024/1024/1024) size_in_GB
        from dba_segments
        where OWNER = 'NOME DO SCHEMA'
        group by owner
        order by owner desc;
        
        /////
        
        Agrupado por segmentos
        
        SELECT owner, 
               segment_type, 
               tablespace_name, 
               SUM(bytes) / 1024 / 1024 AS size_in_MB
        FROM dba_segments
        WHERE owner = 'STATUS_REPORT'
        GROUP BY owner, segment_type, tablespace_name
        ORDER BY segment_type;
        
        ```
        
    - V**erificar as 10 maiores tabelas**
        
        ```sql
        SELECT * FROM
        (select
        SEGMENT_NAME,
        SEGMENT_TYPE,
        BYTES/1024/1024/1024 GB,
        TABLESPACE_NAME
        from
        dba_segments
        order by 3 desc ) WHERE
        ROWNUM <= 10
        ```
        
    - V**erificar o status do backup do banco de dados**
        
        ```sql
        set linesize 500
        col BACKUP_SIZE for a20
        SELECT
        INPUT_TYPE "BACKUP_TYPE",
        VL(INPUT_BYTES/(1024*1024),0)"INPUT_BYTES(MB)",
        NVL(OUTPUT_BYTES/(1024*1024),0) "OUTPUT_BYTES(MB)",
        STATUS,
        TO_CHAR(START_TIME,'MM/DD/YYYY:hh24:mi:ss') as START_TIME,
        TO_CHAR(END_TIME,'MM/DD/YYYY:hh24:mi:ss') as END_TIME,
        TRUNC((ELAPSED_SECONDS/60),2) "ELAPSED_TIME(Min)",
        ROUND(COMPRESSION_RATIO,3)"COMPRESSION_RATIO",
        ROUND(INPUT_BYTES_PER_SEC/(1024*1024),2) "INPUT_BYTES_PER_SEC(MB)",
        ROUND(OUTPUT_BYTES_PER_SEC/(1024*1024),2) "OUTPUT_BYTES_PER_SEC(MB)",
        INPUT_BYTES_DISPLAY "INPUT_BYTES_DISPLAY",
        OUTPUT_BYTES_DISPLAY "BACKUP_SIZE",
        OUTPUT_DEVICE_TYPE "OUTPUT_DEVICE"
        INPUT_BYTES_PER_SEC_DISPLAY "INPUT_BYTES_PER_SEC_DIS",
        OUTPUT_BYTES_PER_SEC_DISPLAY "OUTPUT_BYTES_PER_SEC_DIS"
        FROM V$RMAN_BACKUP_JOB_DETAILS
        where start_time > SYSDATE -1
        and INPUT_TYPE != 'ARC'
        ORDER BY END_TIME DESC;
        ```
        
    - V**erificar LAG Golden Gate**
        
        ```sql
        Diretórios
        
        CLARIFY: /u05/app/oragg/dirdat/cl																				
        CLARIFY2: /u05/app/oragg/dirdat/al
        SIEBELPRE2: /u05/app/oragg/dirdat/DP
        RDSBEVA1:/u05/app/oragg/dirdat/Lv*
        SBELPOS3:/u05/app/oracle/product/12.3.0.1/dirdat/ip21/DS*
        
        Arquivo de configuração prm: vi /u05/app/oracle/product/12.3.0.1/dirprm/ <arquivo>.prm
        Log golden gate: tail -f /u05/app/oracle/product/12.3.0.1/ggserr.log
        
        "VERIFICAR SERVIÇO GOLDENGATE
        1: verificar se o oracle esta up
        #su - oracle
        #dba
        Se não estiver up (Se aparecer idle)
        #startup;
        
        2: Verificar os processos do GoldenGate
        #su - oragg
        #gg2
        #Start mgr!
        #start replicat RDSBEVA1
        #start replicat SBELPOS3
        stop replicat
        #info all
        #exit
        #gg
        #Start mgr!
        #start replicat CLARIFY
        #start replicat SBELPRE2
        #info all
        "						
        
        select count(*) from  SENSOR.TABLE_CLOSE_CASE;
        SELECT count(*) FROM SENSOR.TABLE_CASE;
        
        select MAX(CREATE_DATE) from  SENSOR.TABLE_INTERACT; 
        select MAX(LAST_UPD) from  SENSOR.TABLE_INTERACT; 
        select max(create_Date) from SENSOR.TABLE_INTERACT
        select max(close_Date) from SENSOR.TABLE_CLOSE_CASE
        select max(creation_time) from SENSOR.TABLE_CASE
        select max(created) from SENSOR.S_SRV_REQ_SBELPOS2
        
        SELECT 'SENSOR.TABLE_INTERACT - CREATE_DATE' AS TABELA, MAX(CREATE_DATE) AS DATA FROM SENSOR.TABLE_INTERACT
        UNION ALL
        SELECT 'SENSOR.TABLE_CLOSE_CASE - CLOSE_DATE' AS TABELA, MAX(CLOSE_DATE) AS DATA FROM SENSOR.TABLE_CLOSE_CASE
        UNION ALL
        SELECT 'SENSOR.TABLE_CASE - CREATION_TIME' AS TABELA, MAX(CREATION_TIME) AS DATA FROM SENSOR.TABLE_CASE
        UNION ALL
        SELECT 'SENSOR.S_SRV_REQ_SBELPOS2 - CREATED' AS TABELA, MAX(CREATED) AS DATA FROM SENSOR.S_SRV_REQ_SBELPOS2
        
        Verificar se há sessões com bloqueio.
        
        select s1.username || '@' || s1.machine
        || ' ( SID=' || s1.sid || ' ) is blocking '
        || s2.username || '@' || s2.machine || ' ( SID=' || s2.sid || ' ) ' AS blocking_status
        from gv$lock l1, gv$session s1, gv$lock l2, gv$session s2
        where s1.sid=l1.sid and s2.sid=l2.sid
        and l1.BLOCK=1 and l2.request > 0
        and l1.id1 = l2.id1
        and l2.id2 = l2.id2 ;
        
        select 'ALTER SYSTEM DISCONNECT SESSION '''||sid||','||serial#||',' || '@'|| INST_ID || ''' IMMEDIATE;'
        from Gv$session
        where  sid in (SID);
        ```
        
    - V**erificar os jobs em execução no banco de dados**
        
        ```sql
        SELECT * FROM ALL_SCHEDULER_RUNNING_JOBS;
        ```
        
    - V**erificar detalhes dos logs dos jobs**
        
        ```sql
        select log_id, log_date, owner, job_name
        from ALL_SCHEDULER_JOB_LOG
        where job_name like 'RMAN_B%' and log_date > sysdate-1;
        select log_id,log_date, owner, job_name, status, ADDITIONAL_INFO
        from ALL_SCHEDULER_JOB_LOG;
        --where log_id=113708;
        ```
        
    - V**erificar detalhes do percentual de exportação de dados (DATAPUMP)**
        
        ```sql
        SELECT SID, SERIAL#, USERNAME, CONTEXT, SOFAR, TOTALWORK,
        ROUND(SOFAR/TOTALWORK*100,2) "%_COMPLETE"
        FROM V$SESSION_LONGOPS WHERE TOTALWORK != 0 AND SOFAR <> TOTALWORK;
        DBA
        ```
        
    - V**erificar as principais classes de espera no banco de dados Oracle**
        
        ```sql
        Select wait_class, sum(time_waited), sum(time_waited)/sum(total_waits)
        Sum_Waits
        From v$system_wait_class
        Group by wait_class
        Order by 3 desc;
        
        -> A partir da consulta acima, forneça cada classe de espera na consulta abaixo para obter o
        principais eventos de espera no banco de dados em relação a uma classe de espera específica.
        
        Select a.event, a.total_waits, a.time_waited, a.average_wait
        From v$system_event a, v$event_name b, v$system_wait_class c
        Where a.event_id=b.event_id
        And b.wait_class#=c.wait_class#
        And c.wait_class = 'Idle'
        order by average_wait desc;
        
        ```
        
    - Script para eliminar todos os objetos pertencentes a um schema
        
        ```sql
        SET SERVEROUTPUT ON SIZE 1000000
        set verify off
        BEGIN
        FOR c1 IN (SELECT OWNER,table_name, constraint_name FROM dba_constraints
        WHERE constraint_type = 'R' and owner=upper('&shema_name')) LOOP
        EXECUTE IMMEDIATE
        'ALTER TABLE '||' "'||c1.owner||'"."'||c1.table_name||'" DROP CONSTRAINT ' || c1.constraint_name;
        END LOOP;
        FOR c1 IN (SELECT owner,object_name,object_type FROM dba_objects
        where owner=upper('&shema_name')) LOOP
        BEGIN
        IF c1.object_type = 'TYPE' THEN
        EXECUTE IMMEDIATE 'DROP '||c1.object_type||' "'||c1.owner||'"."'||c1.object_name||'" FORCE';
        END IF;
        IF c1.object_type != 'DATABASE LINK' THEN
        EXECUTE IMMEDIATE 'DROP '||c1.object_type||' "'||c1.owner||'"."'||c1.object_name||'"';
        END IF;
        EXCEPTION
        WHEN OTHERS THEN
        NULL;
        END;
        END LOOP;
        EXECUTE IMMEDIATE('purge dba_recyclebin');
        END;
        /
        ```
        
    - Verificar SGA e PGA utilizada
        
        ```sql
        set lines 200 pages 200
        col  estd_physical_reads for 99999999999999999999999
        SELECT sga_size, sga_size_factor, estd_db_time, estd_physical_reads FROM   V$SGA_TARGET_ADVICE;
        
        set lines 200 pages 200
        col  estd_physical_reads for 99999999999999999999999
        col  estd_extra_bytes_rw for 99999999999999999999999
        SELECT pga_target_for_estimate/1024/1024 "PGA Target Estimado", pga_target_factor, estd_time, estd_extra_bytes_rw
        FROM   V$PGA_TARGET_ADVICE;
        ```
        
    - Verificar versão do SO
        
        ```sql
        cat /etc/os-release
        
        cat /etc/*release
        
        hostnamectl
        ```
        
    - Verificar tamanho dos arquivos e limpeza
        
        ```sql
        du -chx /u01 | grep [0-9]G
        
        du -chx /u02/* | grep -E '[0-9]G' | sort -h
        
        du -hx --max-depth=3 /u01 /u02 2>/dev/null | sort -hr | head -30
        
        find /u02 -xdev -type f -size +100M -exec du -ch {} + | sort -h
        
        --10 maiores arquivos com extensões .aud, .xml, .xml* e .tr*
        find / -type f \( -name "*.aud" -o -name "*.xml" -o -name "*.xml*" -o -name "*.tr*" \) -exec du -h {} + 2>/dev/null | sort -hr | head -n 10
        
         df -h && hostname -i && date
          date && hostname -i
         
         Exclui arquivos com mais de 30 minutos
        
        find /u02/app/diag/asm/+asm/+ASM/trace -iname "*.tr*" -cmin +30 -delete
        
        Exclui arquivos com mais de 1 dia
        
        find /u02/app/diag/asm/+asm/+ASM/trace -iname "*.aud" -ctime +1 -delete
        
        find /u02/diag/rdbms/cdbtim01/cdbtim01/trace -iname "cdmp*" -cmin +30 -type d -exec rm -rf {} +
        
        --Verifica arquivos por intervalo de tempo
        find /u02/app/11.2.0/grid/rdbms/audit -iname "*.aud" -newermt "2023-01-01" ! -newermt "2024-01-01"
        
        tar -czf logs_old.tar.gz listener_*.log
        
        rm -f listener_*.log
        
        tar -czf logs_old.tar.gz log_*.xml
        
        rm -f log_*.xml
        ```
        
- JOBS
    - Descobrir por Dependência de Objetos
        
        ```sql
        SELECT 
            owner AS schema_rotina,
            name AS nome_rotina,
            type AS tipo_rotina,
            referenced_name AS nome_tabela
        FROM 
            all_dependencies
        WHERE 
            referenced_name = 'TB_CONTROLE_EXECUCAO'
            AND referenced_type = 'TABLE'
            -- AND owner = 'NOME_DO_SCHEMA'
        ORDER BY 
            type, name;
        ```
        
    - Descobrir processo agendado (Job)
        
        ```sql
        SELECT 
            owner AS schema_job,
            job_name,
            job_action, 
            start_date,
            repeat_interval,
            enabled
        FROM 
            all_scheduler_jobs
        WHERE 
            UPPER(job_action) LIKE '%TB_CONTROLE_EXECUCAO%'
            OR UPPER(job_action) LIKE '%NOME_DA_ROTINA_QUE_VOCE_ACHOU%';
        ```
        
    - Verificar as próximas execuções
        
        ```sql
        SELECT 
            job_name,
            last_start_date AS ultima_vez_que_rodou,
            next_run_date AS proxima_vez_que_vai_rodar,
            state AS status_atual_do_job
        FROM 
            all_scheduler_jobs
        WHERE 
            job_name = 'JOB_ATUALIZA_EXEC_URSA_MAIOR';
        ```
        
    - Verificar histórico detalhado
        
        ```sql
        SELECT 
            log_date AS data_hora_execucao,
            job_name,
            status, 
            run_duration AS tempo_que_demorou_rodando
        FROM 
            all_scheduler_job_run_details
        WHERE 
            job_name = 'JOB_ATUALIZA_EXEC_URSA_MAIOR'
        ORDER BY 
            log_date DESC;
        ```
        
- RESOLUÇÃO ERROS ORA
    - ORA-02292 - Restrição de integridade
        
        ```sql
        ---Descobrir qual é a tabela filha
        SELECT owner,
               constraint_name,
               table_name
        FROM dba_constraints
        WHERE constraint_name = 'ICT_AFC_CUSTOMER_FK';
        
        ---Descobrir a coluna de relacionamento da FK
        SELECT owner,
               table_name,
               column_name
        FROM dba_cons_columns
        WHERE constraint_name = 'ICT_AFC_CUSTOMER_FK';
        
        ---Ver quais registros estão bloqueando o DELETE
        SELECT *
        FROM ICT_TABLES.ICT_AFC
        WHERE CUSTOMER_ID = 106;
        
        ---Descobrir todas as tabelas que dependem de ICT_CUSTOMERS
        SELECT a.table_name filha,
               a.constraint_name,
               c.column_name
        FROM dba_constraints a
        JOIN dba_cons_columns c
             ON a.constraint_name = c.constraint_name
        WHERE a.r_constraint_name = (
              SELECT constraint_name
              FROM dba_constraints
              WHERE table_name = 'ICT_CUSTOMERS'
              AND constraint_type = 'P'
        );
        ```
        
- COMANDO FIND
    - Find alert log
        
        ```sql
        find / -iname "*alert.*log"
        
        find / -iname "init.*"
        
        find /u01/* -type f \( -name "*.aud" -o -name "*.tr" -o -name "*.xml" \)
        
        find /NOME_DO_DIRETÓRIO -xdev -type f -size +1M -exec du -sh {} ';' | sort -rh | more
        ```
        
    - Filtrar e deletar diretórios
        
        ```sql
        find /u01/app/oracle/diag/rdbms/cicop/CICOP/trace -iname ".tr" -exec rm -rf {} \; ← apaga tudo
        
        find /u03/admin/cdbtim01/audit/admin/CDBTIM01/adump -iname "*.aud" -ctime +4   -exec rm -rf {} \;
        
        find /u01/app/oracle/diag/rdbms/arsdb/ARSDB1/trace -iname "*.out" -cmin +180 -exec rm -rf {} \;
        ```
        
    - Procurar arquivos e deletar filtro com range de 3 dias
        
        ```sql
        find /u02/app/diag/asm/+asm/+ASM/trace -iname "*.trc" -ctime +1 -delete
        
        find /u01/app/oracle/diag/rdbms/icdblink/ICDBLINK2/trace -iname "*.trc" -ctime +0 -delete
        
        find /u02/app/diag/asm/+asm/+ASM/trace -iname "*.tr**" -cmin +1 -exec rm -rf {} \;
        
        find /u02/app/diag/asm/+asm/+ASM/trace -iname "*.tr**" -exec rm -rf {} \;
        
        rm -f /u02/app/diag/tnslsnr/dockerdbhombfc01/listener/alert/log_*
        ```
        
- BACKUPS e ROTINAS CRONTAB
    - Alterar CATALYST
        
        ```sql
        editar o arquivo 
        /u01/app/oracle/hpe/HPE-Catalyst-RMAN-Plugin/config/plugin.conf
        ou
        /opt/dpsapps/dbappagent/config/oracle_ddbda.cfg
        
        comentar as linhas (para HPE-Catalyst)
        #CATALYST_STORE_ADDRESS:10.221.57.32
        #CATALYST_STORE_NAME:SC_CTL1_ORACLE_DATABASE
        
        vide exemplo acima
        e alterar o destino
        
        --Lista catalyst disponíveis
        CATALYST_STORE_ADDRESS:10.221.57.32 - CATALYST_STORE_NAME:SC_CTL1_ORACLE_DATABASE
        CATALYST_STORE_ADDRESS:10.221.57.34 - CATALYST_STORE_NAME:SC_CTL2_ORACLE_DATABASE
        CATALYST_STORE_ADDRESS:10.221.57.36 - CATALYST_STORE_NAME:SC_CTL3_ORACLE_DATABASE
        CATALYST_STORE_ADDRESS:10.221.57.38 - CATALYST_STORE_NAME:SC_CTL4_ORACLE_DATABASE
        
        SC_CTL1-2_ORACLE_ARCHIVE|10.221.57.32
        SC_CTL1-2_ORACLE_ARCHIVE|10.221.57.34
        SC_CTL4_NFVI_ORACLE|10.221.57.38
        SC_CTL4_ORACLE_ARCHIVE|10.221.57.38
        
        ```
        
    - Comando para alterar várias palavras de uma vez
        
        ```sql
        :%s/deleteold.rcv/deleteoldarc.rcv/g
        ```
        
    - Comando para acompanhar o backup
        
        ```sql
        alter session set nls_date_format='DD-MON-YYYY HH24:MI:SS';
        set line 2222;
        set pages 2222;
        set long 6666;
        select sl.sid, sl.opname,
        to_char(100*(sofar/totalwork), '990.9')||'%' pct_done,
        sysdate+(TIME_REMAINING/60/60/24) done_by
        from v$session_longops sl, v$session s
        where sl.sid = s.sid
        and sl.serial# = s.serial#
        and sl.sid in (select sid from v$session where module like 'backup%' or module like 'restore%' or module like 'rman%')
        and sofar != totalwork
        and totalwork > 0
        order by sl.opname desc
        /
        
        --
        
        ALTER SESSION SET NLS_DATE_FORMAT = 'DD-MON-YYYY HH24:MI:SS';
        SET LINESIZE 400;
        SET PAGESIZE 800;
        COLUMN SID FORMAT 99999 HEADING 'SID';
        COLUMN SERIAL# FORMAT 99999 HEADING 'SERIAL';
        COLUMN ELAPSED_SECONDS FORMAT 9999999 HEADING 'DECORRIDO (s)';
        COLUMN TIME_REMAINING FORMAT 99999999 HEADING 'RESTANTE (s)';  
        COLUMN PCT_DONE FORMAT A30 HEADING 'CONCLUÍDO'; 
        COLUMN START_TIME FORMAT A40 HEADING 'INICIADO EM';
        COLUMN DONE_BY FORMAT A20 HEADING 'FINALIZAÇÃO PREVISTA';
        COLUMN OPNAME FORMAT A35 HEADING 'OPERAÇÃO'; 
        SELECT 
            sl.sid, 
            sl.serial#, 
            sl.elapsed_seconds,
            sl.time_remaining,
            TO_CHAR(ROUND(100 * (sofar / totalwork), 1), '90.0') || '%' AS pct_done, 
            sl.start_time,
            TO_CHAR(SYSDATE + (sl.time_remaining / 86400), 'DD-MON-YYYY HH24:MI:SS') AS done_by,
            sl.opname
        FROM v$session_longops sl
        WHERE sofar != totalwork
        AND totalwork > 0
        ORDER BY sl.start_time DESC;
        
        ```
        
    - Query para verificar processos que bloqueiam rotina bkp RMAN
        
        ```sql
        **Verificar quem está bloqueando
        
        SET LINESIZE 200
        SET PAGESIZE 50
        COLUMN SID FORMAT 99999
        COLUMN USERNAME FORMAT A15
        COLUMN PROGRAM FORMAT A40
        COLUMN MODULE FORMAT A30
        COLUMN ACTION FORMAT A30
        COLUMN LOGON_TIME FORMAT A19
        COLUMN TYPE FORMAT A5
        COLUMN ID1 FORMAT 99999
        COLUMN ID2 FORMAT 99999
        COLUMN LMODE FORMAT 999
        COLUMN REQUEST FORMAT 999
        COLUMN CTIME FORMAT 99999
        COLUMN BLOCK FORMAT 999
        
        SELECT 
            s.sid,
            NVL(s.username, '(BACKGROUND)') AS username,
            s.program,
            s.module,
            s.action,
            TO_CHAR(s.logon_time, 'YYYY-MM-DD HH24:MI:SS') AS logon_time,
            l.type,
            l.id1,
            l.id2,
            l.lmode,
            l.request,
            l.ctime,
            l.block
        FROM 
            v$session s
        JOIN 
            v$enqueue_lock l ON l.sid = s.sid
        WHERE 
            l.type = 'CF' 
            AND l.id1 = 0 
            AND l.id2 = 2;
        
        **Verificar o SERIAL#
        
        SELECT SID, SERIAL#, USERNAME, STATUS, PROGRAM
        FROM V$SESSION
        WHERE SID = 133;
        
        **Matar sessão
        
        ALTER SYSTEM disconnect SESSION '133,55205' IMMEDIATE;
        
        **************************************
        set lines 200 pages 200 
        col s.sid for 9999999999
        col username for a30
        col program for a30
        col module for 9999999999
        col action for a30
        col logon_time for 9999999999
        SELECT s.sid, username, program, module, action, logon_time, l.*
        FROM v$session s, v$enqueue_lock l
        WHERE l.sid = s.sid and l.type = 'CF' AND l.id1 = 0 and l.id2 = 2;
        ```
        
    - Query monitoramento bkp full zabbix
        
        ```sql
        select count(*) from V$RMAN_BACKUP_JOB_DETAILS@DBLINK_BACKUP 
        where END_TIME>=sysdate-2 and STATUS in ('COMPLETED','COMPLETED WITH WARNINGS') and (INPUT_TYPE='DB INCR' or INPUT_TYPE='DB FULL');
        ```
        
    - Excluir arquivos de backup antigo (Após crosscheck backup)
        
        ```sql
        run{
        DELETE EXPIRED BACKUP;
        
        DELETE  force noprompt OBSOLETE RECOVERY WINDOW OF 30 DAYS;
        }
        ```
        
    - Verificar backups
        
        ```sql
        SELECT
            HOSTNAME,
            IP_DB,
            DB NAME,
            dbid,
            NVL(TO_CHAR(max(BACKUPTYPE_DB), 'DD/MM/YYYY HH24:MI'), '01/01/0001:00:00') AS DBBKP,
            NVL(TO_CHAR(max(BACKUPTYPE_ARCH), 'DD/MM/YYYY HH24:MI'), '01/01/0001:00:00') AS ARCBKP
        FROM
            (SELECT
                c.nome_db AS "DB",
                a.dbid,
                IP_DB,
                HOSTNAME,
                DECODE(b.bck_type, 'D', MAX(b.completion_time), 'I', MAX(b.completion_time)) AS BACKUPTYPE_DB,
                DECODE(b.bck_type, 'L', MAX(b.completion_time)) AS BACKUPTYPE_ARCH
            FROM
                rc_database a,
                bs b,
                dados_banco c
            WHERE
                a.db_key = b.db_key
                AND a.dbid = c.dbid
                AND b.bck_type IS NOT NULL
                AND b.bs_key NOT IN (
                    SELECT bs_key
                    FROM rc_backup_controlfile
                    WHERE AUTOBACKUP_DATE IS NOT NULL OR AUTOBACKUP_SEQUENCE IS NOT NULL
                )
                AND b.bs_key NOT IN (
                    SELECT bs_key
                    FROM rc_backup_spfile
                )
            GROUP BY
                c.nome_db,
                a.dbid,
                b.bck_type,
                IP_DB,
                HOSTNAME
            )
        GROUP BY
            DB,
            dbid,
            IP_DB,
            HOSTNAME
        ORDER BY
            TO_DATE(ARCBKP, 'DD/MM/YYYY HH24:MI'),
            LEAST(TO_DATE(DBBKP, 'DD/MM/YYYY HH24:MI'));
        ```
        
    - Verificar tamanho dos backups a partir do tamanho dos datafiles
        
        ```sql
        select ((select sum(bytes) from v$datafile) - (select sum(bytes) from dba_free_space)) /1024/1024/1024 as tam_gb from dual;
        ```
        
    - Configurar rotinas de limpeza
        
        ```sql
        su - oracle
        
        vi /home/oracle/backup/scripts/clean_audit_tr.sh -- verificar diretório certo
        
        New file
        
        ##Colocar rotina de limpeza
        
        #!/bin/bash
        
        find /u01/app/oracle/diag/rdbms/ossrpa/OSSRPA1/trace -iname "*tr*" -cmin +30 -delete
        
        find /u01/app/oracle/diag/rdbms/ossrpa/OSSRPA1/alert -iname "log_*" -mtime +1 -delete
        
        find /u01/app/oracle/admin/OSSRPA/adump -iname "*.aud" -cmin +30 -exec rm -rf {} \;
        
        find /u02/app/grid/diag/tnslsnr/rparedbddsne01/asmnet1lsnr_asm/trace -iname "*.tr*" -cmin +30 -exec rm -rf {} \;
        
        find /u02/app/grid/diag/tnslsnr/rparedbddsne01/listener/trace -iname "*.tr*" -cmin +30 -exec rm -rf {} \;
        
        Conceder permissão de execução: chmod u+x /home/oracle/backup/scripts/clean_audit_tr.sh
        
        Agendar no crontab
        
        #Rotina de limpeza de audits antigos, todos os dias as 23h
        0 23 * * * /home/oracle/backup/scripts/cleanup_listener.sh
        ```
        
    - Configurar rotinas de compactação de arquivos xml
        
        ```sql
        su - oracle
        
        vi /home/oracle/backup/scripts/clean_audit_tr.sh -- verificar diretório certo
        
        New file
        
        ##Colocar rotina de limpeza
        
        #!/bin/bash
        
        # Diretório onde estão os logs
        LOG_DIR="/u02/app/diag/tnslsnr/projectdblap001/listener/alert" --verificar diretório
        
        # Navega até o diretório
        cd "$LOG_DIR" || exit 1
        
        # Encontra os arquivos log_*.xml com mais de 30 dias
        FILES=$(find . -maxdepth 1 -name "log_*.xml" -mtime +30) --verificar nome dos logs
        
        # Verifica se há arquivos a processar
        if [ -n "$FILES" ]; then
            # Cria o arquivo tar.gz com data no nome
            tar -czf "logs_old_$(date +%Y%m%d).tar.gz" $FILES
        
            # Remove os arquivos após compactar
            rm -f $FILES
        fi
        
        Conceder permissão de execução: chmod u+x /home/oracle/backup/scripts/clean_audit_tr.sh
        
        Agendar no crontab
        
        #Rotina de limpeza de audits antigos, todos os dias as 23h
        0 23 * * * /home/oracle/backup/scripts/clean_audit_tr.sh
        ```
        
    - Configurar rotina de compactação dos logs
        
        ```sql
        vi /home/oracle/backup/scripts/compact_log_backup.sh
        
        #!/bin/bash
        
        # Diretório onde os logs estão localizados
        LOG_DIR="/home/oracle/backup/log"
        
        # Definindo o mês anterior
        ANO=$(date -d "last month" +%Y)
        MES=$(date -d "last month" +%m)
        
        # Arquivo ZIP de destino
        ZIP_FILE="backup_log_${MES}_${ANO}.zip"
        
        # Mudando para o diretório onde estão os arquivos de log
        cd "$LOG_DIR" || { echo "Diretório $LOG_DIR não encontrado"; exit 1; }
        
        # Encontrando e compactando arquivos .log criados há mais de um mês
        find . -name "*.log" -type f -mtime +30 -print | zip "$ZIP_FILE" -@
        
        # Verificando se o arquivo ZIP foi criado com sucesso
        if [ -f "$ZIP_FILE" ]; then
            #echo "Arquivos compactados com sucesso em $ZIP_FILE."
        
            # Removendo arquivos .log após compactação
            find . -name "*.log" -type f -mtime +30 -exec rm -f {} \;
           find . -name "*.zip" -type f -ctime +365 -exec rm -f {} \;
        # echo "Arquivos .log antigos removidos com sucesso."
        else
            echo "Erro ao compactar arquivos."
        fi
        
        chmod +x /home/oracle/backup/scripts/compact_log_backup.sh
        
        #Compactar e remover arquivos de log de backup com mais de 30 dias (ficara apenas o arquivo compactado)
        0 6 1 * * /home/oracle/backup/scripts/compact_log_backup.sh
        
        ./compact_log_backup.sh
        
        ```
        
    - Configurar rotinas de BKP
        
        ```sql
        Verificar banco com rotina ativa e fazer a transferência conforme modelo
        
        Diretório de configuração das rotinas: cd /home/oracle/backup/scripts/ 
        
        MÁQUINA QUE VAI RECEBER BKP
        
        $ sqlplus / as sysdba
        
        SQL> alter system set control_file_record_keep_time=40 scope=both;
        
        SQL> ALTER DATABASE ENABLE BLOCK CHANGE TRACKING;
        
        cd /home/oracle
        mkdir -p backup/bin backup/log backup/scripts
        
        cat /etc/oratab
        
        Verificar a home
        PIDBPROD:/u01/app/oracle/product/19.0.0/dbhome_1:N     # line added by Agent
        
        echo $ORACLE_SID
        PIDBPROD
        
        vi $ORACLE_HOME/network/admin/tnsnames.ora
        
        #Entrada Catalogo RMAN
        CAT_TIM =
          (DESCRIPTION =
            (ADDRESS = (PROTOCOL = TCP)(HOST = 10.192.67.211)(PORT = 1521))
            (CONNECT_DATA =
              (SERVER = DEDICATED)
              (SERVICE_NAME = cat_TIM)
            )
          )
        
        rman target sys/"267wEVqU-kr3Hq" CATALOG rman/Mu2t4ng@CAT_TIM
        
        RMAN> register database;
        
        CONFIGURAR SBT_LIBRARY
        
        RUN{
        CONFIGURE RETENTION POLICY TO RECOVERY WINDOW OF 30 DAYS;
        CONFIGURE BACKUP OPTIMIZATION OFF;
        CONFIGURE DEFAULT DEVICE TYPE TO 'SBT_TAPE';
        CONFIGURE DEVICE TYPE 'SBT_TAPE' PARALLELISM 4 BACKUP TYPE TO BACKUPSET; 
        CONFIGURE CHANNEL DEVICE TYPE 'SBT_TAPE' PARMS  'ENV=(BLKSIZE=1048576,SBT_LIBRARY=/u01/app/oracle/hpe/HPE-Catalyst-RMAN-Plugin/bin/libisvsupport_rman.so,CONFIG_FILE=/u01/app/oracle/hpe/HPE-Catalyst-RMAN-Plugin/config/plugin.conf)';
        CONFIGURE ARCHIVELOG DELETION POLICY TO BACKED UP 1 TIMES TO 'SBT_TAPE';
        CONFIGURE SNAPSHOT CONTROLFILE NAME TO '+DATA/snapcf_BBOTM.f';
        }
        
        ---Verificar alocação de canais
        
        RUN {
          ALLOCATE CHANNEL ch1 DEVICE TYPE SBT_TAPE
            PARMS 'SBT_LIBRARY=/opt/dpsapps/dbappagent/lib/lib64/libddboostora.so';
        }
        
        Ajustar os scripts
        
        backup_oracle_NEXUS_L0.sh
        backup_oracle_NEXUS_L1.sh
        delete_old_backup_ARC_NEXUS.sh
        delete_old_backup_NEXUS.sh
        
        ROTINAS CRONTAB (Exemplo)
        
        #======================================================================
        # Crontab Legend :
        #
        # |   |  |  |  |-> Day of week (0=Sunday 6=Saturday)
        # |   |  |  |-> Month (1-12)
        # |   |  |-> Day (1-31)
        # |   |-> Hours (0-23)
        # |-> Minutes (0-59)
        #
        #======================================================================
        
        #############################
        # BACKUPS DATABASE
        #############################
        
        # Backup FULL (Level 0)-Execucao: Quartas-feiras as 20:00
        #0 20 * * 3 /home/oracle/backup/scripts/backup_oracle_NSP_L0.sh CDBNSP
        
        # Backup Incremental (Level 1)-Execução: Diario exceto quarta-feira as 20:00
        #0 20 * * 0,1,2,4,5,6 /home/oracle/backup/scripts/backup_oracle_NSP_L1.sh CDBNSP
        
        # Backup Archivelog-Execucao: A cada hora no minuto 30
        #30 * * * * /home/oracle/backup/scripts/backup_oracle_NSP_ARC.sh CDBNSP
        
        #############################
        # LIMPEZA DE BACKUPS
        #############################
        
        # Remocao de archivelogs antigos-Execucao: Diario as 23:00
        #0 23 * * * /home/oracle/backup/scripts/delete_old_backup_ARC_NSP.sh CDBNSP
        
        # Remocao de backups antigos-Execucao: Diario as 01:00
        #0 1 * * * /home/oracle/backup/scripts/delete_old_backup_NSP.sh CDBNSP
        
        #############################
        # MANUTENCAO DE LOGS
        #############################
        
        # Compactar logs de backup com mais de 30 dias mantem apenas arquivos compactados-Execucao: Dia 01 de cada mes as 06:00
        #0 6 1 * * /home/oracle/backup/scripts/compact_log_backup.sh
        
        Conceder permissão para todas as rotinas
        
        Exemplo:
        
        chmod +x /home/oracle/backup/scripts/backup_oracle_PIDBPROD_L1.sh
        
        ```
        
    - Configurar rotinas de BKP (2) - código oracle e root
        
        ```bash
        Parte 2 - Arquivos: Parâmetros do banco, tnsnames.ora, hosts.
        obs: somente para quem for RAC vai ter um quinto Arquivo scan
        
        su - oracle
        mkdir /home/oracle/script_bkp_part2/
        cd /home/oracle/script_bkp_part2/
        vi /home/oracle/script_bkp_part2/script_info_backup_parte2.sql
        
        ##SQL Arquivo Parâmetros do banco
        
        set markup html on spool on
        spool /tmp/BACKUP_DOS_Parametros_BANCO_INFO.html
        --INFO DO BANCO
        TTITLE CENTER 'Ambiente de coleta do relatorio'
        SELECT to_char(systimestamp,'dd-mm-YYYY HH24:MM:SS') DATA ,UTL_INADDR.get_host_ad
        dress IP_ADDRESS, UTL_INADDR.get_host_name HOSTNAME from dual;
        select d.dbid dbid, d.name db_name, i.instance_number inst_num, i.instance_name
        inst_name from v$database d, v$instance i;
        select name,open_mode from v$database;
        TTITLE OFF
        TTITLE CENTER 'Informacoes gv$instance'
        SELECT INSTANCE_NAME, HOST_NAME, STATUS FROM gv$instance;
        TTITLE CENTER 'Informacoes v$pdbs'
        select name,open_mode from v$pdbs;
        TTITLE CENTER 'Informacoes v$database'
        select database_role, name, open_mode from v$database;
        TTITLE CENTER 'Informacoes gerais'
        SELECT * FROM V$DATABASE;
        SELECT * FROM gv$instance;
        --#####FEATURES ORACLE
        TTITLE CENTER 'FEATURES ORACLE'
        set linesize 200 pagesize 10000
        SELECT NAME,VERSION,CURRENTLY_USED,FEATURE_INFO
        FROM DBA_FEATURE_USAGE_STATISTICS
        WHERE NAME = 'Real Application Clusters (RAC)';
        SELECT NAME,VERSION,CURRENTLY_USED,FEATURE_INFO
        FROM DBA_FEATURE_USAGE_STATISTICS
        WHERE NAME = 'Data Guard';
        TTITLE CENTER 'Informacoes Oracle Database Vault'
        SELECT value
        FROM v$option
        WHERE parameter = 'Oracle Database Vault'
        AND value = 'TRUE';
        set pages 200 lines 1000
        SELECT * FROM DVSYS.DBA_DV_STATUS;
        --ASM_DISKGROUP
        TTITLE CENTER 'Informacoes Diskgroup'
        SELECT name, type, ceil (total_mb/1024) "TOTAL(GB)" , ceil (free_mb/1024) "FREE(G
        B)", CEIL((total_mb - free_mb) / 1024) AS "USED(GB)" ,round((free_mb) /(total_mb)
        * 100, 2) "FREE PCT %" FROM V$ASM_DISKGROUP;
        TTITLE CENTER 'Tamanhos Fisico e logico do banco'
        column TAMANHO_GB format a16
        Select 'Tamanho total fisico' DESCRICAO ,TO_CHAR(sum(bytes)/1024/1024/1024,'99999
        99999999.99') || ' GB' TAMANHO from dba_data_files
        union all
        Select 'Tamanho total logico' DESCRICAO ,TO_CHAR(sum(bytes)/1024/1024/1024,'99999
        99999999.99') || ' GB' TAMANHO from dba_segments;
        --#Tamanho Total Fisico:
        --Descricao: Refere-se ao espaco fisico ocupado pelos arquivos de dados do banco
        de dados no disco.
        --Calculo: Obtido somando o tamanho de todos os arquivos de dados fisicos (por ex
        emplo, arquivos de dados, tablespaces) no banco de dados.
        --Origem dos Dados na Query: A consulta utiliza a visao dba_data_files, que conte
        m informacões sobre os arquivos de dados fisicos.
        --#Tamanho Total Logico:
        --Descricao: Refere-se ao espaco logico ocupado pelos segmentos dentro do banco d
        e dados. Isso inclui os segmentos de tabelas, indices, entre outros.
        --Calculo: Obtido somando o tamanho de todos os segmentos logicos no banco de dad
        os.
        --Origem dos Dados na Query: A consulta utiliza a visao dba_segments, que contem
        informacões sobre os segmentos logicos do banco de dados.
        -- ########### Tablespaces
        TTITLE CENTER 'Informacoes das Tablespaces'
        BTITLE LEFT 'EXTENSIBLE(GB): A quantidade de espaco que pode ser automaticamente
        estendida, caso tenha um Data File Autoextend - DTF_AUTO )'
        set pagesize 1000 linesize 180
        col "TOTAL(GB)" for 99999.999
        col "USAGE(GB)" for 99999.999
        col "FREE(GB)" for 99999.999
        col "EXTENSIBLE(GB)" for 999.999
        col "FREE PCT %" for 999.99
        SELECT
        d.tablespace_name "NAME",
        d.contents "TYPE",
        ROUND(NVL(a.bytes / 1024 / 1024 / 1024, 0), 3) "TOTAL(GB)",
        ROUND(NVL(f.bytes, 0) / 1024 / 1024 / 1024, 3) "FREE(GB)",
        ROUND(NVL(a.bytes - NVL(f.bytes, 0), 0) / 1024 / 1024 / 1024, 3) "USAGE(GB)",
        ROUND(NVL((NVL(f.bytes, 0) / a.bytes) * 100, 0), 3) "FREE PCT %",
        ROUND(NVL(a.ARTACAK, 0) / 1024 / 1024 / 1024, 3) "EXTENSIBLE(GB)",
        a.NOTO AS "DTF_NAUTO",
        a.OTO AS "
        DTF_AUTO"
        FROM
        sys.dba_tablespaces d
        LEFT JOIN
        (
        SELECT
        tablespace_name,
        SUM(bytes) bytes,
        SUM(DECODE(autoextensible, 'YES', MAXbytes - bytes, 0)) ARTACAK,
        COUNT(DECODE(autoextensible, 'NO', 0)) NOTO,
        COUNT(DECODE(autoextensible, 'YES', 0)) OTO
        FROM
        dba_data_files
        GROUP BY
        tablespace_name
        ) a ON d.tablespace_name = a.tablespace_name
        LEFT JOIN
        (
        SELECT
        tablespace_name,
        SUM(bytes) bytes
        FROM
        dba_free_space
        GROUP BY
        tablespace_name
        ) f ON d.tablespace_name = f.tablespace_name
        WHERE
        NOT (d.extent_management LIKE 'LOCAL' AND d.contents LIKE 'TEMPORARY')
        UNION ALL
        SELECT
        d.tablespace_name "NAME",
        d.contents "TYPE",
        ROUND(NVL(a.bytes / 1024 / 1024 / 1024, 0), 3) "TOTAL(GB)",
        ROUND(NVL(a.bytes - NVL(t.bytes, 0), 0) / 1024 / 1024 / 1024, 3) "FREE(GB)",
        ROUND(NVL(t.bytes, 0) / 1024 / 1024 / 1024, 3) "USAGE(GB)",
        ROUND(NVL((NVL(a.bytes - NVL(t.bytes, 0), 0) / a.bytes) * 100, 0), 3) "FREE PCT
        %",
        ROUND(NVL(a.ARTACAK, 0) / 1024 / 1024 / 1024, 3) "EXTENSIBLE(GB)",
        a.NOTO AS DTF_NAUTO,
        a.OTO AS DTF_AUTO
        FROM
        sys.dba_tablespaces d
        LEFT JOIN
        (
        SELECT
        tablespace_name,
        SUM(bytes) bytes,
        SUM(DECODE(autoextensible, 'YES', MAXbytes - bytes, 0)) ARTACAK,
        COUNT(DECODE(autoextensible, 'NO', 0)) NOTO,
        COUNT(DECODE(autoextensible, 'YES', 0)) OTO
        FROM
        dba_temp_files
        GROUP BY
        tablespace_name
        ) a ON d.tablespace_name = a.tablespace_name
        LEFT JOIN
        (
        SELECT
        tablespace_name,
        SUM(bytes_used) bytes
        FROM
        v$temp_extent_pool
        GROUP BY
        tablespace_name
        ) t ON d.tablespace_name = t.tablespace_name
        WHERE
        d.extent_management LIKE 'LOCAL'
        AND d.contents LIKE 'TEMPORARY%'
        ORDER BY
        "FREE PCT %" ASC;
        --NAME (NOME): O nome da tablespace.
        --TYPE (TIPO): O tipo da tablespace (permanent ou temporary).
        --TOTAL(GB) (TOTAL(GB)): O tamanho total da tablespace em gigabytes.
        --FREE(GB) (LIVRE(GB)): O espaco livre atual na tablespace em gigabytes.
        --USAGE(GB) (USO(GB)): O espaco utilizado na tablespace em gigabytes.
        --FREE PCT % (PORCENTAGEM LIVRE %): A porcentagem de espaco livre em relacao ao t
        amanho total da tablespace.
        --EXTENSIBLE(GB) (EXTENSIVEL(GB)): A quantidade de espaco que pode ser automatica
        mente estendida (caso o autoextensible esteja habilitado) em gigabytes.
        --DTF_NAUTO (DTF_NAUTO): Numero de arquivos de dados nao autoextensiveis.
        --DTF_AUTO (DTF_AUTO): Numero de arquivos de dados autoextensiveis.
        TTITLE OFF
        BTITLE OFF
        --'LOCAL DOS DBF'
        TTITLE CENTER 'LOCAL DOS DBF'
        select name,checkpoint_change#,last_change# from v$datafile;
        TTITLE OFF
        --######## TEMFILES
        TTITLE CENTER 'TEMFILES'
        set pages 999
        set lines 400
        col FILE_NAME format a75
        select d.TABLESPACE_NAME, d.FILE_NAME, d.BYTES/1024/1024 SIZE_MB, d.AUTOEXTENSIBL
        E, d.MAXBYTES/1024/1024 MAXSIZE_MB, d.INCREMENT_BY*(v.BLOCK_SIZE/1024)/1024 INCRE
        MENT_BY_MB
        from dba_temp_files d, v$tempfile v
        where d.FILE_ID = v.FILE#
        order by d.TABLESPACE_NAME, d.FILE_NAME;
        --######## Datafiles
        TTITLE CENTER 'Datafiles'
        SELECT
        TABLESPACE_NAME,
        FILE_NAME,
        FILE_ID,
        TO_NUMBER(USER_BYTES) / 1024 / 1024 / 1024 AS USAGE_GB,
        TO_NUMBER(MAXBYTES) / 1024 / 1024 / 1024 AS TOTAL_GB,
        AUTOEXTENSIBLE,
        TO_NUMBER(INCREMENT_BY) / 1024 / 1024 AS INCREMENT_MB
        FROM DBA_DATA_FILES
        ORDER BY TABLESPACE_NAME;
        TTITLE OFF
        --######## REDOLOGS
        TTITLE CENTER 'Redologs'
        select l.group#, l.sequence#, l.thread#,
        (l.bytes/1024/1024) mbytes, l.members,
        l.archived, l.status,
        to_char(l.first_time,'dd/mm/yyyy hh24:mi:ss') first_time,
        f.member as redo_filename
        from GV$LOG l
        join GV$LOGFILE f
        on l.group# = f.group#
        order by 1;
        TTITLE OFF
        TTITLE CENTER ' v$controlfile'
        SELECT * FROM v$controlfile;
        TTITLE OFF
        --##############
        -- Definir formato das colunas para a saída da consulta
        COLUMN name FORMAT A30
        COLUMN value FORMAT A50
        -- Informações do SPFILE
        VARIABLE spfile VARCHAR2(1000);
        BEGIN
        SELECT value
        INTO :spfile
        FROM v$parameter
        WHERE name = 'spfile';
        END;
        /
        -- Informações do DBIID (Database Instance ID)
        VARIABLE dbiid NUMBER;
        BEGIN
        SELECT dbid
        INTO :dbiid
        FROM v$database;
        END;
        /
        -- Versão do Oracle
        VARIABLE oracle_version VARCHAR2(100);
        BEGIN
        SELECT version
        INTO :oracle_version
        FROM v$instance;
        END;
        /
        --##############
        -- Exibir os resultados
        PRINT dbiid
        --##
        PRINT oracle_version
        --##
        PRINT spfile
        --##
        -- Informações do parâmetro do SPFILE
        set linesize 20000
        set pagesize 20000
        SELECT name, value
        FROM v$spparameter;
        --
        TTITLE CENTER 'dba_hist_osstat'
        select
        (select max(value) from dba_hist_osstat
        where stat_name = 'NUM_CPUS') NUM_CPUS,
        (select max(value) from dba_hist_osstat
        where stat_name = 'NUM_CPU_CORES') NUM_CPU_CORES,
        (select max(value) from dba_hist_osstat
        where stat_name = 'NUM_CPU_SOCKETS') NUM_CPU_SOCKETS
        from dual;
        TTITLE CENTER 'NLS_DATABASE_PARAMETERS'
        select *from NLS_DATABASE_PARAMETERS;
        TTITLE CENTER 'Redologs'
        show parameter config;
        TTITLE CENTER 'PGA E SGA TARGET'
        SHOW PARAMETERS TARGET;
        TTITLE CENTER 'LIMIT'
        SHOW PARAMETERS LIMIT;
        TTITLE OFF
        archive log list
        TTITLE CENTER 'Parameter'
        SELECT name, value
        FROM v$parameter
        WHERE value IS NOT NULL;
        --###############FINAL
        spool off
        set markup html off
        exit
        
        ************************Código shell 1 de 2 - user oracle
        
        vi /home/oracle/script_bkp_part2/script_info_backup_parte2_shell.sh
        
        #!/bin/bash
        $ORACLE_HOME/bin/sqlplus SYS/267wEVqU-kr3Hq as sysdba @/home/oracle/script_bkp_part2/script_info_backup_parte2.sql
        echo
        echo
        echo
        cp $ORACLE_HOME/network/admin/tnsnames.ora /tmp/tnsnames.ora
        echo
        #caso for um Rac
        #srvctl config scan > /tmp/scan.txt
        echo
        
        Obs: Tem que conceder permissão de execução ao script. (chmod u+x)
        
        ****************************Crontab - user oracle
        
        #Rotina de backups dos parametros do banco parte 1 de 2
        0 8 * * * /home/oracle/script_bkp_part2/script_info_backup_parte2_shell.sh > /home/oracle/script_bkp_part2/log_script_info_backup_parte2.txt
        
        ****************************Codigo shell 2 de 2 - user root
        
        cd /root/script_bkp
        
        vi script_info_backup_parte2_shell.sh
        
        #!/bin/bash
        
        #Procura por "BACKUP_DOS_Parametros_BANCO_INFO.html" em todo o sistema e armazena o caminho completo na variavel
        caminhoarquivo=$(find / -type f -name "BACKUP_DOS_Parametros_BANCO_INFO.html" 2>/dev/null)
        
        if [ -n "$caminhoarquivo" ]; then
        #Obtem a localizacao do arquivo
        localizacao=$(realpath "$caminhoarquivo")
        diretorio=$(dirname "$localizacao")
        echo "localização completa do arquivo encontrado: $caminhoarquivo"
        echo
        echo "Caminho do arquivo: $diretorio"
        echo
        
        echo "scp para coletorsne02-10.192.26.248 BACKUP_DOS_Parametros_BANCO_INFO"
        sshpass -p 'coletor' scp "$caminhoarquivo" coletor@10.192.26.248:/files/SPO/info_backup/$(hostname)_$(hostname -i)
        
        echo "scp para coletortrj02-10.221.113.26 BACKUP_DOS_Parametros_BANCO_INFO"
        sshpass -p 'coletor' scp "$caminhoarquivo" coletor@10.221.113.26:/files/RJO/info_backup/$(hostname)_$(hostname -i)
        
        #esse cd e obrigatorio
        cd $diretorio
        if [ -e "BACKUP_DOS_Parametros_BANCO_INFO.html" ]; then
        arquivo="BACKUP_DOS_Parametros_BANCO_INFO.html"
        else
        echo "Nenhum arquivo encontrado com o nome 'BACKUP_DOS_Parametros_BANCO_INFO.html'."
        exit 1
        fi
        
        #Obtem a data de criacao do arquivo
        data_modificacao_conteudo=$(stat -c "Modify - %y" "$arquivo")
        data_alteracao_permissao_arquivo=$(stat -c "Change - %z" "$arquivo")
        
        #Exibe informacoes sobre o arquivo
        echo "Nome do arquivo: $arquivo"
        echo
        #echo "Localização: $localizacao"
        #echo
        echo "Data de modificação do conteudo do arquivo: $data_modificacao_conteudo"
        echo
        echo "Data de modificação da permissão do arquivo: $data_alteracao_permissao_arquivo"
        echo
        hostname -i
        echo
        hostname
        echo
        date
        echo
        echo
        
        else
        echo "Nenhum arquivo encontrado com o nome 'BACKUP_DOS_Parametros_BANCO_INFO.html'."
        exit 1
        fi
        
        echo
        echo
        echo
        #inicio
        echo "scp para coletorsne02-10.192.26.248 tnsnames.ora"
        sshpass -p 'coletor' scp /tmp/tnsnames.ora coletor@10.192.26.248:/files/SPO/info_backup/$(hostname)_$(hostname -i)
        
        echo "scp para coletortrj02-10.221.113.26 tnsnames.ora"
        sshpass -p 'coletor' scp /tmp/tnsnames.ora coletor@10.221.113.26:/files/RJO/info_backup/$(hostname)_$(hostname -i)
        #fim
        echo
        echo "scp para coletorsne02-10.192.26.248 /etc/hosts"
        sshpass -p 'coletor' scp /etc/hosts coletor@10.192.26.248:/files/SPO/info_backup/$(hostname)_$(hostname -i)
        
        echo "scp para coletortrj02-10.221.113.26 /etc/hosts"
        sshpass -p 'coletor' scp /etc/hosts coletor@10.221.113.26:/files/RJO/info_backup/$(hostname)_$(hostname -i)
        
        echo
        #Se for RAC
        #echo "scp para coletorsne02-10.192.26.248 /tmp/scan.txt"
        #sshpass -p 'coletor' scp /tmp/scan.txt coletor@10.192.26.248:/files/SPO/info_backup/$(hostname)_$(hostname -i)
        
        #echo "scp para coletortrj02-10.221.113.26 /tmp/scan.txt"
        #sshpass -p 'coletor' scp /tmp/scan.txt coletor@10.221.113.26:/files/RJO/info_backup/$(hostname)_$(hostname -i)
        echo
        
        ****************************Crontab - user root
        
        #Rotina de backups dos parametros do banco parte 2 de 2
        0 14 * * * /root/script_bkp/script_info_backup_parte2_shell.sh >> /root/script_bkp/log_script_info_backup_parte2.txt
        
        VALIDAÇÃO COLETORES
        
        {
          find /files -type f \( -name "BACKUP_DOS_Parametros_BANCO_INFO.html" -o -name "hosts" -o -name "tnsnames.ora" -o -name "scan.txt" \) 2>/dev/null -exec stat --format="%y %n" {} \; | sed 's/-0300//' | tee >(wc -l | sed 's/^/Número total de arquivos: /')
          echo
          echo "Quantidade total de BACKUP_DOS_Parametros_BANCO_INFO.html: $(find /files -type f -name "BACKUP_DOS_Parametros_BANCO_INFO.html" 2>/dev/null | wc -l)"
          echo
          echo "Quantidade total de hosts: $(find /files -type f -name "hosts" 2>/dev/null | wc -l)"
          echo
          echo "Quantidade total de tnsnames.ora: $(find /files -type f -name "tnsnames.ora" 2>/dev/null | wc -l)"
          echo
          echo "Quantidade total de scan.txt: $(find /files -type f -name "scan.txt" 2>/dev/null | wc -l)"
        } | sed 's/\(.*\)/\1/'
        
        ```
        
    - Verificar últimos backups realizados (linux)
        
        ```sql
        find / -type f \( -name "*.full.*" -o -name "*.incr.*" -o -name "*.arch.*" -o -name "*.DELETE.*" \) -exec stat --format="%y %n" {} \; 2>/dev/null | sed 's/-0300//' | sort -r | head -n 5
        
        find / -type f \( -name "*.full.*" -o -name "*.incr.*" -o -name "*.DELETE.*" \) -exec stat --format="%y %n" {} \; 2>/dev/null | sed 's/-0300//' | sort -r | head -n 15
        ```
        
    - Verificar servidores com backup ativo
        
        ```sql
        select * from dados_banco order by 3;
        ```
        
    - Comando para incluir/ alterar /retirar monitoramento do banco
        
        ```sql
        *****NO SQL DEVELOPER (INSERIR NO MONITORAMENTO)
          
        INSERT ALL
        INTO DADOS_BANCO (DBID,NOME_DB,IP_DB,HOSTNAME) VALUES ('DBIDNOVO',' Nomedobanconovo','IPNOVO','HOTNAMENOVO')
        SELECT * FROM dual;
        commit;
        
        *****Alterar um registro
        
        SELECT * FROM DADOS_BANCO WHERE DBID = <    >;
        
        UPDATE DADOS_BANCO
        SET NOME_DB = '<  >',
            IP_DB = '<  >',
            HOSTNAME = '<  >'
        WHERE DBID = <  >;
        
        commit;
        
        *****Excluir um registro
        DELETE FROM DADOS_BANCO
        WHERE DBID = '313284281';
        COMMIT;
        ```
        
    - Verificar configurações do ARCHIVELOG
        
        ```sql
        Verificar Status do archivelog
        	
        	ARCHIVE LOG LIST;
        
        Configurações de arquivo de parâmetros
        	
        	SHOW PARAMETER LOG_ARCHIVE_DEST;
        	SHOW PARAMETER LOG_ARCHIVE_FORMAT;
        
        ```
        
- DBLINKS
    - Verificar DBLINKS
        
        ```sql
        select * from all_db_links 
        ou 
        select * from dba_db_links
        ```
        
    - Testar DBLINK
        
        ```sql
        SELECT * FROM dual@DBLINK_PPM_PRD_HML;
        ```
        
    - Verificar  se existem procedures, functions ou views utilizando DBLINK
        
        ```sql
        -- ver se existem procedures ou functions usando determinado dblink
        SELECT      o.owner,
                    o.object_type,
                    o.object_name,
                    o.status,
                    s.text
        FROM        dba_objects o
        INNER JOIN  dba_source s
            ON      s.owner = o.owner
            AND     s.type = o.object_type
            AND     s.name = o.object_name
        WHERE       UPPER(s.TEXT) LIKE UPPER('%&db_link_name%')
        ORDER BY    1,2,3;
        
        -- ver se existem visoes usando determinado db_link
        SELECT * FROM DBA_VIEWS WHERE ADMBASECOMUM.FC_CONVERTE_LONG_TO_CHAR('DBA_VIEWS','TEXT','VIEW_NAME',VIEW_NAME) LIKE '%&db_link_name%';
        ```
        
    - Consulta dblinks em view, mview
        
        ```sql
        SELECT *
        FROM (
        
        SELECT 
        'VIEW' TIPO_OBJETO,
        OWNER,
        VIEW_NAME OBJETO,
        NULL TIPO,
        NULL DB_LINK
        FROM DBA_VIEWS
        WHERE TEXT_VC LIKE '%@%'
        
        UNION ALL
        
        SELECT
        'MATERIALIZED VIEW',
        OWNER,
        MVIEW_NAME,
        NULL,
        MASTER_LINK
        FROM DBA_MVIEWS
        WHERE MASTER_LINK IS NOT NULL
        
        UNION ALL
        
        SELECT
        'SOURCE',
        OWNER,
        NAME,
        TYPE,
        NULL
        FROM DBA_SOURCE
        WHERE TEXT LIKE '%@%'
        AND OWNER NOT IN ('SYS','SYSTEM')
        
        UNION ALL
        
        SELECT
        'SYNONYM',
        OWNER,
        SYNONYM_NAME,
        NULL,
        DB_LINK
        FROM DBA_SYNONYMS
        WHERE DB_LINK IS NOT NULL
        
        UNION ALL
        
        SELECT
        'DB_LINK',
        OWNER,
        DB_LINK,
        NULL,
        DB_LINK
        FROM DBA_DB_LINKS
        
        )
        ORDER BY OWNER, TIPO_OBJETO, OBJETO;
        ```
        
    - Criar DBLINKS
        
        ```sql
        CREATE PUBLIC DATABASE LINK "DBLINK_BACKUP"
           CONNECT TO "USERNAME" IDENTIFIED BY "PASSWORD"
           USING 'HOSTNAME';
           
           CREATE DATABASE LINK "SGFHML_DBLINK"
           CONNECT TO "SGFHML" IDENTIFIED BY "7PQuWXwU2#Ft"
           USING 'DEVARM3';
        
         //  
           
           
        CREATE PUBLIC DATABASE LINK DBLINK_BI_FAM_DG CONNECT TO BI_FAM IDENTIFIED BY B141_D0_F4m USING '
        (DESCRIPTION =
        (ADDRESS = (PROTOCOL = TCP)(HOST = 10.192.67.27)(PORT = 1521))
        (CONNECT_DATA =
        (SERVER = DEDICATED)
        (SERVICE_NAME = timauddb)
        )
        )';
        
        SELECT * FROM dual@DBLINK_MYSQL;
           
        ```
        
    - Exportar todos os DBLINKS
        
        ```sql
        full=y
        INCLUDE=DB_LINK:"IN(SELECT db_link FROM dba_db_links)"
        ```
        
- USUÁRIOS
    - Evidência criação de usuário
        
        ```sql
        SELECT 
            u.username, 
            u.account_status, 
            u.created 
            rp.granted_role
        FROM 
            dba_users u
        LEFT JOIN 
            dba_role_privs rp 
            ON u.username = rp.grantee
        WHERE 
            u.username IN (
                'T3752385',
                'T3746511',
                'T3746513',
                'T3749947')
        ORDER BY 
            u.username, 
            rp.granted_role;
            
            
            ****************************
            --Evidencia o schema de acesso da role
        WITH user_roles AS (
            SELECT 
                grantee,
                granted_role,
                ROW_NUMBER() OVER (PARTITION BY grantee ORDER BY granted_role) AS rn
            FROM dba_role_privs
            WHERE grantee = 'NTW_MABI_HQ'
        ),
        
        roles_pivot AS (
            SELECT 
                grantee,
                MAX(CASE WHEN rn = 1 THEN granted_role END) AS granted_role,
                MAX(CASE WHEN rn = 2 THEN granted_role END) AS granted_role2
            FROM user_roles
            GROUP BY grantee
        ),
        
        role_schemas AS (
            SELECT 
                role,
                LISTAGG(owner, ', ') WITHIN GROUP (ORDER BY owner) AS schemas
            FROM (
                SELECT DISTINCT role, owner
                FROM role_tab_privs
            )
            GROUP BY role
        )
        
        SELECT 
            u.username,
            u.account_status,
            u.default_tablespace,
            u.created,
            r.granted_role,
            rs1.schemas AS role_schemas,
            r.granted_role2,
            rs2.schemas AS role2_schemas
        FROM dba_users u
        JOIN roles_pivot r ON u.username = r.grantee
        LEFT JOIN role_schemas rs1 ON r.granted_role = rs1.role
        LEFT JOIN role_schemas rs2 ON r.granted_role2 = rs2.role
        WHERE u.username = 'NTW_MABI_HQ';
            
            **************************
            --Com quota de tablespace
            
        SELECT
            u.username,
            u.account_status,
            u.created,
            u.default_tablespace,
            --q.tablespace_name AS quota_tablespace,
            --q.max_bytes / 1024 / 1024 / 1024 AS quota_gb,
            MAX(DECODE(sp.privilege, 'CREATE TABLE', 'YES', 'NO')) AS create_table,
            MAX(DECODE(sp.privilege, 'CREATE PROCEDURE', 'YES', 'NO')) AS create_procedure,
            MAX(DECODE(sp.privilege, 'CREATE TRIGGER', 'YES', 'NO')) AS create_trigger,
            MAX(DECODE(sp.privilege, 'CREATE VIEW', 'YES', 'NO')) AS create_view,
            MAX(DECODE(sp.privilege, 'CREATE MATERIALIZED VIEW', 'YES', 'NO')) AS create_materialized_view,
            MAX(DECODE(sp.privilege, 'CREATE SEQUENCE', 'YES', 'NO')) AS create_sequence
        FROM
            dba_users u
        LEFT JOIN
            dba_ts_quotas q
            ON u.username = q.username
        LEFT JOIN
            dba_sys_privs sp
            ON u.username = sp.grantee
        WHERE
            u.username = 'ODM_HML'
        GROUP BY
            u.username,
            u.account_status,
            u.created,
            u.default_tablespace,
            q.tablespace_name,
            q.max_bytes
        ORDER BY
            u.username;
            
        
        ```
        
    - **Verificar se um usuário existe no banco**
        
        ```sql
        SELECT USERNAME FROM DBA_USERS
        WHERE USERNAME='USER';
        ```
        
    - Evidência de que um usuário não existe no banco de dados
        
        ```sql
        SET LINESIZE 200
        SET PAGESIZE 100
        COLUMN database_name FORMAT A15
        COLUMN host_name     FORMAT A30
        COLUMN ip            FORMAT A15
        COLUMN username      FORMAT A20
        SELECT d.name AS database_name,
               i.host_name,
               utl_inaddr.get_host_address(i.host_name) AS ip,
               u.username
          FROM v$database d
               JOIN v$instance i ON 1=1
               LEFT JOIN dba_users u ON u.username = 'ACCENTURE';
        
        ```
        
    - **Verificar a data de criação de um usuário a partir de data específica**
        
        ```sql
        SET LINESIZE 200 PAGESIZE 10000
        COL username FORMAT A20
        COL created FORMAT A10
        COL expiry_date FORMAT A10
        COL account_status FORMAT A15
        COL profile FORMAT A30
        COL DEFAULT_TABLESPACE FORMAT A30
        COL last_login FORMAT A20
        COL TEMPORARY_TABLESPACE FORMAT A30
        
        SELECT
            username,
            created,
            account_status,
            TO_CHAR(last_login, 'DD-MON-YY HH24:MI:SS') AS last_login,
            profile,
            expiry_date
        FROM
            dba_users
        WHERE
            created > TO_DATE('01-12-2023', 'DD-MM-YYYY');
        ```
        
    - **Verifica qual usuário tem permissão no DBVault**
        
        ```sql
        col grantee for a30
        col granted_role for a30
        select grantee, granted_role 
        from dba_role_privs 
        where granted_role in ('DV_OWNER','DV_ACCTMGR');
        ```
        
    - **Verificar usuários em banco multitenant direto do CDB**
        
        ```sql
        --verifica usuários criados no PDB a partir do CDB
        SELECT 
            cu.con_id,
            vc.name AS pdb_name,
            cu.username,
            cu.account_status,
            cu.created AS creation_date,
            cu.expiry_date,
            cu.common,
            cu.oracle_maintained
        FROM cdb_users cu
        JOIN v$containers vc ON cu.con_id = vc.con_id
        WHERE vc.name = 'NOME_PDB'
          AND cu.oracle_maintained = 'N'
          AND cu.username NOT LIKE 'SYS%'
          AND cu.username NOT LIKE 'SYSTEM'
          AND cu.username NOT LIKE 'APEX%'
          AND cu.username NOT LIKE 'DBA%'
          AND cu.username NOT LIKE 'C##%'
        ORDER BY cu.username;
        
        ```
        
    - **Verificar usuários que tem acesso ao banco de dados**
        
        ```sql
        COLUMN username FORMAT A30
        COLUMN account_status FORMAT A20
        COLUMN current_date FORMAT 999999
        SELECT username, account_status, TO_CHAR(SYSDATE, 'DD/MM/YYYY HH24:MI:SS') AS current_date
        FROM dba_users
        WHERE account_status = 'OPEN';
        ```
        
    - Evidências de criação de usuários e permissões
        
        ```sql
        --data de criação e status de um usuário
        SELECT username, account_status, created
          FROM dba_users
         WHERE  USERNAME='SINFO001';
         
        --privilégios de sistema de um usuário
        SET LINESIZE 200
        SET PAGESIZE 50
        COLUMN username FORMAT A20
        COLUMN account_status FORMAT A20
        COLUMN created FORMAT A12
        COLUMN privilege FORMAT A50
        SELECT u.username,
               u.account_status,
               u.created,
               p.privilege AS granted_privileges
          FROM dba_users u
          LEFT JOIN dba_sys_privs p
            ON u.username = p.grantee
         WHERE u.username = 'F8011907';
        
         
        --privilégios concedidos a uma role
        SET LINESIZE 200
        SET PAGESIZE 50
        COLUMN username FORMAT A20
        COLUMN account_status FORMAT A20
        COLUMN created FORMAT A12
        COLUMN granted_role FORMAT A50
         SELECT u.username,
               u.account_status,
               u.created,
               rp.granted_role AS roles
          FROM dba_users u
          LEFT JOIN dba_role_privs rp
            ON u.username = rp.grantee
         WHERE u.username = 'F8011907';
        
         
        --privilégios de objeto de um usuário
        SET LINESIZE 200
        SET PAGESIZE 50
        COLUMN username FORMAT A20
        COLUMN account_status FORMAT A20
        COLUMN created FORMAT A12
        COLUMN granted_object_privileges FORMAT A50
        SELECT u.username,
               u.account_status,
               u.created,
               tp.privilege || ' ON ' || tp.owner || '.' || tp.table_name AS granted_object_privileges
          FROM dba_users u
          LEFT JOIN dba_tab_privs tp
            ON u.username = tp.grantee
         WHERE u.username = 'ZABBIX';
        ```
        
    - **Verificar usuários conectados**
        
        ```sql
        SELECT s.sid, s.serial#, s.username, s.OSUSER, s.MACHINE
        FROM v$session s
        WHERE s.username = 'ODMRESHML';
        ```
        
    - **Verificar configurações de um profile**
        
        ```sql
        SELECT PROFILE, RESOURCE_NAME, RESOURCE_TYPE, LIMIT
        FROM DBA_PROFILES
        WHERE PROFILE = 'DEFAULT_TIM_SYSTEM'
        ORDER BY RESOURCE_NAME;
        ```
        
    - **Privilégios concedidos a uma role que é concedida a um usuário**
        
        ```sql
        SELECT * FROM DBA_TAB_PRIVS WHERE GRANTEE IN
        (SELECT granted_role FROM DBA_ROLE_PRIVS WHERE GRANTEE = 'AUTOMACAO_GM') order by 3;
        
        SELECT *
        FROM DBA_TAB_PRIVS
        WHERE GRANTEE IN
            (SELECT granted_role
             FROM DBA_ROLE_PRIVS
             WHERE GRANTEE = 'T3526985')
             AND TABLE_NAME NOT LIKE 'BIN%'
        ORDER BY 3;
        ```
        
    - **Verificar último acesso de um usuário ao banco**
        
        ```sql
        select username, last_login from dba_users where username in ('T3326992','T3526985');
        ```
        
    - **Verificar qual aplicação está bloqueando a senha do usuário**
        
        ```sql
        SELECT OS_USERNAME, TIMESTAMP, userhost, RETURNCODE 
        FROM DBA_AUDIT_SESSION 
        WHERE USERNAME = 'F8083408' 
        ORDER BY TIMESTAMP DESC;
        ```
        
    - **Verificar se a senha de um usuário está expirada**
        
        ```sql
        SELECT username, profile, account_status, expiry_date
        FROM dba_users
        WHERE username = 'NTW_MABE';
        
        ***********************
        SELECT 
            u.username,
            u.profile,
            u.account_status,
            u.expiry_date,
            i.host_name AS server_host,
            UTL_INADDR.get_host_address(i.host_name) AS db_ip,
            CURRENT_TIMESTAMP AS data_hora_execucao
        FROM dba_users u,
             v$instance i
        WHERE u.username = 'F8004221';
        ```
        
    - **Verificar se um usuário está bloqueado.**
        
        ```sql
        SELECT username, account_status, created, lock_date, expiry_date
          FROM dba_users
         WHERE  USERNAME='DEVOPS';
        ```
        
    - **Verificar e alterar parâmetros de um profile**
        
        ```sql
        SELECT profile, resource_name, limit
        FROM dba_profiles
        WHERE profile IN 'C##DEFAULT_TIM_SYSTEM';
        AND LIMIT='VERIFY_FUNCTION_12C';
        
        ALTER PROFILE C##DEFAULT_TIM_SYSTEM LIMIT PASSWORD_VERIFY_FUNCTION VERIFY_FUNCTION_12C;
        ```
        
    - Mostra data de criação e bloqueio de um usuário
        
        ```sql
        set linesize 200
        set pagesize 10000
        col username format a20
        col account_status format a15
        col created format a10
        col lock_date format a10
        
        SELECT username, profile, account_status, created, lock_date 
        FROM dba_users 
        --WHERE account_status = 'locked' 
        AND username = ('C##SVC_BACKUP');
        
        ```
        
    - **Verificar os privilégios concedidos a um usuário.**
        
        ```sql
        SELECT * FROM DBA_TAB_PRIVS WHERE GRANTEE = 'ZABBIX';
        
        -----------------
        
        SELECT grantee,
               owner,
               table_name,
               privilege,
               grantable
        FROM dba_tab_privs
        WHERE grantee = 'AUTOMACAO_GM'
          AND owner = 'CUSTOMIZATION'
          AND table_name = 'VW_MONITORACAO_CLASSIFICACAO';
        ```
        
    - **Verificar os privilégios concedidos em tabelas específicas.**
        
        ```sql
        SELECT grantee, owner, table_name, privilege
        FROM dba_tab_privs
        WHERE table_name IN ('OFFER_OCS', 'OMS_CATALOG_PLAN')
        and GRANTEE = 'ZABBIX';
        
        ***
        
        --verificar se o usuário tem privilégio de execução sobre um procedimento
        SELECT grantee, privilege, owner, table_name
        FROM dba_tab_privs
        WHERE grantee = 'BI_NETFLOW'
          AND owner = 'MLOG_HOM'
          and PRIVILEGE = 'SELECT';
          AND table_name = 'VWTB_TASK_INFO';
        ```
        
    - **Verificar as roles concedidas a um usuário.**
        
        ```sql
        SELECT *
        FROM DBA_ROLE_PRIVS
        WHERE GRANTEE = 'USER'
        ```
        
    - **Verificar o tamanho dos schemas**
        
        ```sql
        SELECT owner, trunc(sum(bytes)/1024/1024/1024,2) "SIZE GB" 
        FROM dba_segments 
        where owner not in 
            ( 'SYSTEM', 'XDB', 'SYS', 'TSMSYS', 'MDSYS', 
            'EXFSYS', 'WMSYS', 'ORDSYS', 'OUTLN', 'DBSNMP') 
        group by owner
        order by 2 desc;
        ```
        
    - Comando para verificar o profile de um usuário
        
        ```sql
        SELECT username, profile FROM dba_users WHERE username = 'USER';
        ```
        
    - Comando para verificar os profiles do banco de dados
        
        ```sql
        SELECT DISTINCT profile FROM dba_profiles;
        ```
        
    - Comando para bloquear um usuário
        
        ```sql
        ALTER USER <USERNAME> ACCOUNT LOCK;
        ```
        
    - Comando para alterar o profile de um usuário
        
        ```sql
        ALTER USER ZABBIX PROFILE DEFAULT_TIM_SYSTEM;
        ```
        
- SESSÕES
    - Verificar sessões na v$session e v$process
        
        ```sql
        SELECT  S.SID, S.SERIAL#, P.SPID, S.USERNAME, 
                       S.STATUS, S.OSUSER, S.MACHINE, 
                       S.PROGRAM, S.MODULE, 
                       TO_CHAR(S.LOGON_TIME, 'dd/mm/yyyy hh24:mi:ss') LOGON_TIME, 
                       S.blocking_session -- id da sessao bloqueadora (qdo for o caso)
               FROM    V$SESSION S
               JOIN    V$PROCESS P
                 ON    P.addr = S.paddr
               WHERE   S.TYPE = 'USER';
               --and S.OSUSER='ldorighe';
        ```
        
    - Verificar sessões bloqueadas e desconectar
        
        ```sql
        select 'ALTER SYSTEM DISCONNECT SESSION '''||s1.sid||','||s1.serial#||',' || '@'|| s1.INST_ID || ''' IMMEDIATE;' cmd, s1.username || '@' || s1.machine
        || ' ( SID=' || s1.sid || ' ) is blocking '
        || s2.username || '@' || s2.machine || ' ( SID=' || s2.sid || ' ) ' AS blocking_status
        from gv$lock l1, gv$session s1, gv$lock l2, gv$session s2
        where s1.sid=l1.sid and s2.sid=l2.sid
        and l1.BLOCK=1 and l2.request > 0
        and l1.id1 = l2.id1
        and l2.id2 = l2.id2;
        ```
        
    - Verificar sessões que mais consumiram espaço na TEMP
        
        ```sql
        SELECT
            t.sql_id,
            t.max_temp_gb,
            t.total_temp_gb,
            st.sql_text
        FROM (
            SELECT
                ash.sql_id,
                ROUND(MAX(ash.temp_space_allocated) / 1024 / 1024 / 1024, 2) AS max_temp_gb,
                ROUND(SUM(ash.temp_space_allocated) / 1024 / 1024 / 1024, 2) AS total_temp_gb
            FROM dba_hist_active_sess_history ash
            WHERE ash.sample_time >= (SYSDATE - INTERVAL '3' HOUR)
              AND ash.temp_space_allocated > 0
            GROUP BY ash.sql_id
        ) t
        JOIN dba_hist_sqltext st
          ON st.sql_id = t.sql_id
        ORDER BY t.max_temp_gb DESC
        FETCH FIRST 10 ROWS ONLY;
        ```
        
    - Verificar sessões mais pesadas TOP 20
        
        ```sql
        SELECT * FROM (
            SELECT parsing_schema_name,
                   sql_id,
                   executions,
                   ROUND(elapsed_time/1000000,2) total_seconds,
                   ROUND(elapsed_time/DECODE(executions,0,1,executions)/1000000,2) avg_seconds,
                   buffer_gets,
                   disk_reads
            FROM v$sql
            WHERE parsing_schema_name NOT IN ('SYS','SYSTEM')
            ORDER BY elapsed_time DESC
        )
        WHERE ROWNUM <= 20;
        ```
        
    - Verificar quais usuários geram maior impacto
        
        ```sql
        SELECT s.username,
               COUNT(*) sessoes,
               SUM(q.buffer_gets) total_buffer_gets,
               SUM(q.disk_reads) total_disk_reads,
               ROUND(SUM(q.elapsed_time)/1000000,2) total_seconds
        FROM v$session s
        JOIN v$sql q ON s.sql_id = q.sql_id
        WHERE s.type = 'USER'
        AND s.username IS NOT NULL
        GROUP BY s.username
        ORDER BY total_seconds DESC;
        ```
        
    - Verificar utilização de UNDO
        
        ```sql
        WITH
        undo_size AS
        (
            SELECT SUM(bytes)/1024/1024 undo_mb
            FROM dba_data_files
            WHERE tablespace_name =
                  (SELECT value FROM v$parameter WHERE name='undo_tablespace')
        ),
        undo_rate AS
        (
            SELECT
               SUM(undoblks) /
               SUM((end_time - begin_time) * 86400) undo_blk_per_sec
            FROM v$undostat
        ),
        params AS
        (
            SELECT
               (SELECT value FROM v$parameter WHERE name='db_block_size') block_size,
               (SELECT value FROM v$parameter WHERE name='undo_retention') undo_retention
            FROM dual
        )
        SELECT
            ROUND(u.undo_mb,2) AS UNDO_ATUAL_MB,
            ROUND(u.undo_mb/1024,2) AS UNDO_ATUAL_GB,
            ROUND((r.undo_blk_per_sec * p.block_size * p.undo_retention)/1024/1024,2) AS UNDO_RECOMENDADO_MB,
            ROUND((r.undo_blk_per_sec * p.block_size * p.undo_retention)/1024/1024/1024,2) AS UNDO_RECOMENDADO_GB,
            ROUND((u.undo_mb/1024) -
                  ((r.undo_blk_per_sec * p.block_size * p.undo_retention)/1024/1024/1024),2) DIFERENCA_GB
        FROM undo_size u,
             undo_rate r,
             params p;
             
             
             
             
         --ALTER DATABASE DATAFILE '+DATA/PRDARM/DATAFILE/undotbs2.343.1155910837'  AUTOEXTEND ON NEXT 1G MAXSIZE 800G;    
        ```
        
    - Verificar em tempo real, quem está gerando o maior volume de dados de UNDO
        
        ```sql
        SELECT
        s.sid,
        s.serial#,
        s.username,
        s.program,
        t.used_ublk * p.value /1024/1024 MB_UNDO
        FROM v$transaction t
        JOIN v$session s ON s.saddr=t.ses_addr
        JOIN v$parameter p ON p.name='db_block_size'
        ORDER BY MB_UNDO DESC;
        ```
        
    - Verificar uso de memória
        
        ```sql
        --Verificação rápida do uso de memória
        SELECT s.sid,
               s.serial#,
               s.username,
               s.program,
               p.spid AS os_pid,
               ROUND(SUM(v.value)/1024/1024,2) AS memoria_mb
        FROM v$session s
        JOIN v$process p ON s.paddr = p.addr
        JOIN v$sesstat v ON s.sid = v.sid
        JOIN v$statname n ON v.statistic# = n.statistic#
        WHERE n.name = 'session pga memory'
        GROUP BY s.sid, s.serial#, s.username, s.program, p.spid
        ORDER BY memoria_mb DESC
        FETCH FIRST 10 ROWS ONLY;
        
        --Verificação completa do uso de memória com o sql_text
        SELECT s.sid,
               s.serial#,
               s.username,
               s.program,
               p.spid AS os_pid,
               ROUND(SUM(v.value)/1024/1024,2) AS memoria_mb,
               s.status,
               s.logon_time,
               ROUND((SYSDATE - s.logon_time)*24*60,2) AS tempo_minutos,
               q.sql_text
        FROM v$session s
        JOIN v$process p ON s.paddr = p.addr
        JOIN v$sesstat v ON s.sid = v.sid
        JOIN v$statname n ON v.statistic# = n.statistic#
        LEFT JOIN v$sql q ON s.sql_id = q.sql_id
        WHERE n.name = 'session pga memory'
        GROUP BY s.sid, s.serial#, s.username, s.program, p.spid, q.sql_text, s.status, s.logon_time
        ORDER BY memoria_mb DESC
        FETCH FIRST 10 ROWS ONLY;
        ```
        
    - Verificar detalhes da sessão através do SID e SERIAL
        
        ```sql
        SET LINES 300
        SET PAGES 100
        SET WRAP OFF
        SET TRIMSPOOL ON
        COLUMN username   FORMAT A14
        COLUMN status     FORMAT A8
        COLUMN osuser     FORMAT A10
        COLUMN machine    FORMAT A18
        COLUMN program    FORMAT A20
        COLUMN module     FORMAT A22
        COLUMN action     FORMAT A10
        COLUMN event      FORMAT A30
        COLUMN state      FORMAT A10
        COLUMN logon_time FORMAT A19
        SELECT
            sid,
            serial#,
            username,
            status,
            osuser,
            machine,
            program,
            module,
            action,
            TO_CHAR(logon_time,'DD/MM/YYYY HH24:MI:SS') AS logon_time,
            last_call_et  AS last_call_sec,
            event,
            state
        FROM v$session
        WHERE sid = 154
          AND serial# = 29486;
        
        ```
        
    - Verificar em qual instância está sendo executada através do SID e SERIAL (RAC)
        
        ```sql
        --Oracle RAC
        SELECT inst_id
        FROM gv$session
        WHERE sid = 1391
        AND serial# = 40504;
        
        ```
        
    - Verificar sessões e sessões bloqueadas
        
        ```sql
        SELECT s.sid,
               s.serial#,
               s.username,
               s.status,
               s.event,
               s.sql_id,
               s.sql_hash_value,
               q.sql_text,
               s.blocking_session,
               s.wait_class,
               s.seconds_in_wait
          FROM v$session s
          JOIN v$sql q ON s.sql_id = q.sql_id
         --WHERE q.sql_text LIKE '%DROP%'
           where q.sql_text LIKE '%STG_STFC_ROTA_CLASSIFICADA%'
           AND s.status = 'ACTIVE';
           
           
           
        SELECT
            s.sid                  AS sessao_bloqueada,
            s.serial#              AS serial_bloqueada,
            s.username             AS usuario_bloqueado,
            s.status,
            s.event,
            s.blocking_session     AS sessao_bloqueadora,
            s.seconds_in_wait,
            s.wait_class,
            bs.username            AS usuario_bloqueador,
            s.sql_id,
            q.sql_text             AS sql_bloqueado,
            bq.sql_text            AS sql_bloqueador
        FROM v$session s
        LEFT JOIN v$sql q  ON s.sql_id = q.sql_id
        LEFT JOIN v$session bs ON s.blocking_session = bs.sid
        LEFT JOIN v$sql bq ON bs.sql_id = bq.sql_id
        WHERE s.blocking_session IS NOT NULL
          AND s.status = 'ACTIVE'
          AND (q.sql_text LIKE '%STG_STFC_ROTA_CLASSIFICADA%' OR bq.sql_text LIKE '%STG_STFC_ROTA_CLASSIFICADA%');
        
        SELECT s.sid,
               s.serial#,
               s.username,
               s.status,
               s.event,
               s.sql_id,
               q.sql_text,
               s.blocking_session,
               s.seconds_in_wait,
               s.state
          FROM v$session s
          LEFT JOIN v$sql q ON s.sql_id = q.sql_id
         WHERE s.status = 'ACTIVE'
           AND s.wait_class != 'Idle'
           AND q.sql_text LIKE '%STG_STFC_ROTA_CLASSIFICADA%';
        ```
        
    - Verificar sessões bloqueadas
        
        ```sql
        select s1.username || '@' || s1.machine
        || ' ( SID=' || s1.sid || ' ) is blocking '
        || s2.username || '@' || s2.machine || ' ( SID=' || s2.sid || ' ) ' AS blocking_status
        from gv$lock l1, gv$session s1, gv$lock l2, gv$session s2
        where s1.sid=l1.sid and s2.sid=l2.sid
        and l1.BLOCK=1 and l2.request > 0
        and l1.id1 = l2.id1
        and l2.id2 = l2.id2 ;
        ```
        
    - Verificar sessões ativas com SQL_TEXT
        
        ```sql
        SELECT s.sid,
               s.serial#,
               s.username,
               s.status,
               s.event,
               s.sql_id,
               s.sql_hash_value,
               q.sql_text,
               s.blocking_session,
               s.wait_class,
               s.seconds_in_wait
          FROM v$session s
          JOIN v$sql q ON s.sql_id = q.sql_id
         --WHERE q.sql_text LIKE '%DROP%'
         --where q.sql_text LIKE '%STG_STFC_ROTA_CLASSIFICADA%'
           where s.status = 'ACTIVE';
        ```
        
    - Verificação de processos com percentual
        
        ```sql
        SELECT 
            p.inst_id AS node,
            COUNT(*) AS processos_ativos,
            param.value AS limite_processos,
            ROUND((COUNT(*) / param.value) * 100, 2) AS percentual_utilizado
        FROM 
            gv$process p,
            (SELECT value FROM v$parameter WHERE name = 'processes') param
        GROUP BY 
            p.inst_id, param.value
        ORDER BY 
            p.inst_id;
            
            ----
            
            SELECT
            s.sid,
            s.serial#,
            s.username,
            s.status,
            s.osuser,
            s.machine,
            s.program,
            s.sql_id,
            q.sql_text
        FROM
            v$session s
        LEFT JOIN
            v$sql q ON s.sql_id = q.sql_id
        WHERE
            s.username IS NOT NULL
            AND s.status = 'INACTIVE'
        ORDER BY
            s.sid;
        ```
        
    - Verificar sessões bloqueadas (consulta completa)
        
        ```sql
        SELECT 
          s1.username || '@' || s1.machine
            || ' ( SID=' || s1.sid || ' ) is blocking '
            || s2.username || '@' || s2.machine || ' ( SID=' || s2.sid || ' ) ' AS blocking_status,
          'ALTER SYSTEM DISCONNECT SESSION ''' || s1.sid || ',' || s1.serial# || '@' || s1.inst_id || ''' IMMEDIATE;' AS disconnect_command,
          s1.status,
          s1.inst_id,
          s1.sid,
          s1.serial#,
          s1.sql_id,
          s1.last_call_et AS blocking_seconds,
          q.sql_text AS blocking_sql_text
        FROM gv$lock l1
        JOIN gv$session s1 ON s1.sid = l1.sid AND s1.inst_id = l1.inst_id
        JOIN gv$lock l2 ON l1.id1 = l2.id1 AND l1.id2 = l2.id2
        JOIN gv$session s2 ON s2.sid = l2.sid AND s2.inst_id = l2.inst_id
        LEFT JOIN gv$sql q ON q.sql_id = s1.sql_id AND q.inst_id = s1.inst_id
        WHERE l1.block = 1 AND l2.request > 0;
        ```
        
    - Bloco PL/SQL para localizar e matar as sessões
        
        ```sql
        BEGIN
            FOR r IN (
                SELECT s.sid, s.serial#, s.inst_id
                FROM   gv$session s
                JOIN   gv$access a ON s.inst_id = a.inst_id AND s.sid = a.sid
                WHERE  a.owner = 'ARADMIN'
                AND    a.object = 'T5411'
            ) LOOP
                EXECUTE IMMEDIATE
                    'ALTER SYSTEM DISCONNECT SESSION ''' || r.sid || ',' || r.serial# || ',@' || r.inst_id || ''' IMMEDIATE';
            END LOOP;
        END;
        /
        ```
        
    - Verificar sessões e sessões bloqueadas  (CLARO)
        
        ```sql
        --Verifica sessões bloqueadas e bloqueadores
        SET LINESIZE 200
        SET PAGESIZE 100
        COLUMN sample_time        FORMAT A20      HEADING 'SAMPLE_TIME'
        COLUMN sid                FORMAT 99999    HEADING 'SID'
        COLUMN serial#            FORMAT 999999   HEADING 'SERIAL#'
        COLUMN sql_id             FORMAT A13      HEADING 'SQL_ID'
        COLUMN sql_text           FORMAT A90      HEADING 'SQL_TEXT'
        COLUMN event              FORMAT A30      HEADING 'EVENT'
        COLUMN blocker_sid        FORMAT 99999    HEADING 'BLOCKER_SID'
        COLUMN session_state      FORMAT A12      HEADING 'SESSION'
        
        SELECT 
            TO_CHAR(ash.sample_time, 'DD/MM/YYYY HH24:MI:SS') AS sample_time,
            ash.session_id                  AS sid,
            ash.session_serial#             AS serial#,
            ash.sql_id,
            SUBSTR(s.sql_text, 1, 100)      AS sql_text,
            ash.event,
            ash.blocking_session            AS blocker_sid,
            ash.session_state
        FROM 
            dba_hist_active_sess_history ash
        JOIN 
            dba_hist_sqltext s ON ash.sql_id = s.sql_id
        WHERE 
            ash.session_id IN (1923)
            AND ash.sample_time BETWEEN TO_DATE('02/07/2025 18:00','DD/MM/YYYY HH24:MI') 
                                    AND TO_DATE('02/07/2025 18:30','DD/MM/YYYY HH24:MI')
        ORDER BY 
            ash.sample_time, ash.session_id;
        
        --Verificar SQLTEXT
        SET LINESIZE 320
        SET PAGESIZE 100
        SET LONG 10000
        COLUMN sample_time FORMAT A20 HEADING 'SAMPLE_TIME'
        COLUMN session_id  FORMAT 99999 HEADING 'SID'
        COLUMN sql_id      FORMAT A13 HEADING 'SQL_ID'
        COLUMN sql_text    FORMAT A100 HEADING 'SQL_TEXT'
        
        SELECT TO_CHAR(ash.sample_time, 'DD/MM/YYYY HH24:MI:SS') AS sample_time,
               ash.session_id,
               ash.sql_id,
               sq.sql_text
        FROM   dba_hist_active_sess_history ash
        LEFT JOIN dba_hist_sqltext sq ON ash.sql_id = sq.sql_id
        WHERE  ash.session_id = 1923
        AND    ash.sample_time BETWEEN TO_TIMESTAMP('2025-07-02 18:00:00', 'YYYY-MM-DD HH24:MI:SS')
                                  AND TO_TIMESTAMP('2025-07-03 18:10:00', 'YYYY-MM-DD HH24:MI:SS')
        ORDER BY ash.sample_time;
        
        ```
        
    - Troubleshooting sessões em lock
        
        ```sql
        --Identificar sessões que estão com locks em tabela específica
        SELECT
            l.inst_id,
            l.session_id AS sid,
            s.serial#,
            s.username,
            s.osuser,
            s.machine,
            s.status,
            l.locked_mode,
            o.owner,
            o.object_name,
            s.sql_id,
            sql.sql_text
        FROM
            gv$locked_object l
            JOIN gv$session s ON l.inst_id = s.inst_id AND l.session_id = s.sid
            JOIN dba_objects o ON l.object_id = o.object_id
            LEFT JOIN gv$sql sql ON s.sql_id = sql.sql_id
        WHERE
            o.owner = 'BROKENW'
            AND o.object_name = 'SN_ETL_IIAS_T_COLUMNS'
        ORDER BY
            l.inst_id, l.session_id;
            
            
        --Mostrar sessões bloqueadas e bloqueadoras (quem está travado e quem está travando)
        SELECT
            l.inst_id,
            l.session_id AS sid,
            s.serial#,
            s.username,
            s.osuser,
            s.machine,
            s.status,
            s.blocking_session,        
            s.blocking_instance,       
            l.locked_mode,
            o.owner,
            o.object_name,
            s.sql_id,
            sql.sql_text
        FROM
            gv$locked_object l
            JOIN gv$session s
                ON l.inst_id = s.inst_id
               AND l.session_id = s.sid
            JOIN dba_objects o
                ON l.object_id = o.object_id
            LEFT JOIN gv$sql sql
                ON s.sql_id = sql.sql_id
               AND s.inst_id = sql.inst_id
        WHERE
            o.owner = 'BROKENW'
            AND o.object_name = 'SN_ETL_IIAS_T_COLUMNS'
        ORDER BY
            l.inst_id, l.session_id;
            
        
        --Buscar sessões executando ações nessa tabela
        SELECT
            s.inst_id,
            s.sid,
            s.serial#,
            s.username,
            s.status,
            sql.sql_text
        FROM
            gv$session s
            JOIN gv$sql sql ON s.sql_id = sql.sql_id
        WHERE
            UPPER(sql.sql_text) LIKE '%INSERT%'
            AND UPPER(sql.sql_text) LIKE '%BROKENW.SN_ETL_IIAS_T_COLUMNS%'
        ORDER BY
            s.inst_id, s.sid;    
         
        ```
        
    - Verificação de sessões inativas com mais de 24h com alter disconnect (RAC)
        
        ```sql
        SELECT
            s.inst_id,
            s.sid,
            s.serial#,
            s.username,
            s.program,
            s.status,
            s.osuser,
            s.machine,
            s.module,
            s.logon_time,
            ROUND(s.last_call_et / 3600, 2) AS horas_inativas,
            'ALTER SYSTEM DISCONNECT SESSION ''' || s.sid || ',' || s.serial# || ',@' || s.inst_id || ''' IMMEDIATE;' AS comando_kill
        FROM
            gv$session s
        WHERE
            s.status = 'ACTIVE'
            AND s.username IS NOT NULL
            AND s.username <> 'SYS'
            AND s.type = 'USER'
            AND s.last_call_et > 86400  -- 24 horas
        ORDER BY
            s.last_call_et DESC;
        ```
        
    - Contagem de sessões (OSSDB)
        
        ```sql
        SELECT 
            username,
            SUM(CASE WHEN status = 'ACTIVE' THEN 1 ELSE 0 END) AS sessoes_ativas,
            SUM(CASE WHEN status = 'INACTIVE' THEN 1 ELSE 0 END) AS sessoes_inativas
        FROM v$session
        --WHERE username = 'REPORTER'
        GROUP BY username;
        
        --**--
        --Contagem nos dois nós
        SET LINESIZE 200
        SET PAGESIZE 100
        COLUMN node                  HEADING 'Instância'         FORMAT 999
        COLUMN processos_ativos     HEADING 'Processos_Ativos'  FORMAT 999999
        COLUMN limite_processos     HEADING 'Limite_Processos'  FORMAT 999999
        COLUMN percentual_utilizado HEADING 'Percentual(%)'     FORMAT 990.00
        SELECT 
            p.inst_id AS node,
            COUNT(*) AS processos_ativos,
            TO_NUMBER(param.value) AS limite_processos,
            ROUND((COUNT(*) / TO_NUMBER(param.value)) * 100, 2) AS percentual_utilizado
        FROM 
            gv$process p,
            (SELECT value FROM v$parameter WHERE name = 'processes') param
        GROUP BY 
            p.inst_id, param.value
        ORDER BY 
            p.inst_id;
            
           --**--
           
           SELECT 
            username,
            status,
            COUNT(*) AS total_sessoes
        FROM gv$session
        WHERE username IS NOT NULL 
        AND process IS NOT NULL
        GROUP BY username, status
        ORDER BY username, status;
        ```
        
    - Verificação de sessões v$session+v$process
        
        ```sql
        SELECT 
            s.inst_id AS node,
            NVL(s.machine, 'UNKNOWN') AS  maquina_origem,
            SUM(CASE WHEN s.status = 'ACTIVE' THEN 1 ELSE 0 END) AS sessoes_ativas,
            SUM(CASE WHEN s.status = 'INACTIVE' THEN 1 ELSE 0 END) AS sessoes_inativas,
            COUNT(*) AS total_sessoes
        FROM 
            gv$session s,
               gv$process p
        WHERE  s.paddr = p.addr
            and s.type <> 'BACKGROUND'  --and s.machine ='netprodsne001'-- apenas sessões de usuário
        GROUP BY 
            s.inst_id, s.machine
        ORDER BY 
            s.inst_id, total_sessoes DESC;
        ```
        
    - Verificar sessões na v$session e v$SQL
        
        ```sql
        SELECT  S.SID, S.SERIAL#, P.SPID, S.USERNAME,
                       S.STATUS, S.OSUSER, S.MACHINE,
                       S.PROGRAM, S.MODULE,
                       TO_CHAR(S.LOGON_TIME, 'dd/mm/yyyy hh24:mi:ss') LOGON_TIME,
                       S.blocking_session, -- id da sessao bloqueadora (qdo for o caso)
                       DBMS_LOB.SUBSTR(a.SQL_FULLTEXT, 4000,1) sql_text
                 FROM    V$SESSION S
               JOIN    V$PROCESS P
                 ON    P.addr = S.paddr
                 JOIN    V$SQLAREA A
                  ON   s.sql_hash_value = a.hash_value
                 WHERE   TYPE = 'USER';
        ```
        
    - Verificar status das sessões
        
        ```sql
        SELECT 
            s.sid, 
            s.serial#, 
            s.username, 
            s.status, 
            s.osuser, 
            s.machine, 
            s.program, 
            s.module, 
            s.action, 
            s.event, 
            s.wait_class, 
            s.seconds_in_wait, 
            s.state, 
            s.blocking_session, 
            s.sql_id, 
            q.sql_text, 
            s.last_call_et AS tempo_execucao_segundos
        FROM v$session s
        LEFT JOIN v$sql q ON s.sql_id = q.sql_id
        WHERE s.status = 'ACTIVE' 
        AND s.username IS NOT NULL  -- Exclui sessões internas do Oracle
        AND s.program NOT LIKE '%GRID%'  -- Exclui processos do Grid Control
        AND s.username NOT IN ('SYS', 'SYSTEM') -- Exclui usuários do sistema
        ORDER BY s.seconds_in_wait DESC;
        ```
        
    - Verificar consumo de cpu por sessão
        
        ```sql
        SET LINESIZE 200
        SET PAGESIZE 100
        COLUMN USERNAME FORMAT A20
        COLUMN SID_SESSION FORMAT A15
        COLUMN CPU_USED_PERCENTAGE FORMAT 999.99
        
        SELECT    v.username, 
                  v.sid || ',' || v.serial# AS sid_session, 
                  ROUND((s.value/y.value)*100,2) AS cpu_used_percentage
        FROM      v$session v
        JOIN      v$sesstat s ON v.sid = s.sid
        JOIN      v$sysstat y ON s.statistic# = y.statistic#
        WHERE     v.username IS NOT NULL
        AND       y.name = 'CPU used by this session'
        ORDER BY  3 DESC;
        
        ```
        
    - Verificar sessões que estão utilizando tabela específica
        
        ```sql
        SELECT
            s.SID,
            s.SERIAL#,
            s.USERNAME,
            s.STATUS,
            s.OSUSER,
            s.MACHINE,
            s.PROGRAM,
            l.TYPE,
            l.LMODE,
            l.REQUEST,
            l.BLOCK
        FROM
            V$LOCK l
        JOIN
            V$SESSION s ON l.SID = s.SID
        WHERE
            l.ID1 = (SELECT OBJECT_ID FROM DBA_OBJECTS WHERE OBJECT_NAME = 'ICT_SITESDOWN' AND OWNER = 'ICT_TABLES');
            
            *****************************************************  
            
        SELECT
            s.SID,
            s.SERIAL#,
            p.SPID AS OS_PROCESS_ID,
            s.USERNAME,
            s.STATUS,
            s.OSUSER,
            s.MACHINE,
            s.PROGRAM,
            l.TYPE,
            l.LMODE,
            l.REQUEST,
            l.BLOCK
        FROM
            V$LOCK l
        JOIN
            V$SESSION s ON l.SID = s.SID
        JOIN
            V$PROCESS p ON s.PADDR = p.ADDR
        WHERE
            l.ID1 = (SELECT OBJECT_ID FROM DBA_OBJECTS WHERE OBJECT_NAME = 'TABLE_NAME' AND OWNER = 'SCHEMA');
            
        ```
        
    - Verificar sessões com maior consumo de CPU (OSSDB)
        
        ```sql
        --Verificar sessões com maior consumo de CPU    
        SELECT s.sid, s.serial#, s.username, s.program, s.osuser, s.type,
               t.value AS cpu_time
        FROM gv$session s
        JOIN gv$sesstat t ON s.sid = t.sid
        JOIN gv$statname n ON t.statistic# = n.statistic#
        WHERE n.name = 'CPU used by this session'
        and s.type <> 'BACKGROUND'
        ORDER BY t.value DESC;
        
        --Verificar sessões com maior consumo de CPU (mais detalhada)
        SELECT s.inst_id, s.sid, s.serial#, s.username, s.schemaname, s.program, 
               s.osuser, s.machine, s.status, s.event, s.type, t.value AS cpu_time,
               q.sql_text
        FROM gv$session s
        JOIN gv$sesstat t ON s.sid = t.sid AND s.inst_id = t.inst_id
        JOIN gv$statname n ON t.statistic# = n.statistic#
        LEFT JOIN gv$sqlarea q ON s.sql_id = q.sql_id AND s.inst_id = q.inst_id
        WHERE n.name = 'CPU used by this session'
        AND s.type <> 'BACKGROUND'
        ORDER BY t.value DESC;
        
        --Verificar SQL_TEXT da consulta ofensora
        SELECT s.sid, s.serial#, s.username, s.program, q.sql_text
        FROM v$session s
        JOIN v$sql q ON s.sql_id = q.sql_id
        WHERE s.sid = 2470 AND s.serial# = 2001;
        ```
        
    - Verificar processos
        
        ```sql
        SELECT RESOURCE_NAME, CURRENT_UTILIZATION, MAX_UTILIZATION, LIMIT_VALUE
        FROM GV$RESOURCE_LIMIT
        WHERE RESOURCE_NAME IN ('sessions','processes');
        ```
        
    - Verificar  informações sobre sessões de usuários
        
        ```sql
        SELECT NVL(s.username, '(oracle)') AS username,
               s.sql_id,
               s.osuser,
               s.sid,
               s.serial#,
               p.spid,
               ROUND(p.pga_used_mem/1024/1024,2) AS pga_used_mem_mb,
               ROUND(p.pga_alloc_mem/1024/1024,2) AS pga_alloc_mem_mb,
               ROUND(p.pga_freeable_mem/1024/1024,2) AS pga_freeable_mem_mb,
               ROUND(p.pga_max_mem/1024/1024,2) AS pga_max_mem_mb,
               s.lockwait,
               s.status,
               s.service_name,
               s.module,
               s.machine,
               s.program,
               TO_CHAR(s.logon_Time,'DD-MON-YYYY HH24:MI:SS') AS logon_time,
               s.last_call_et AS last_call_et_secs
        FROM   gv$session s,
               gv$process p
        WHERE  s.paddr = p.addr and s.type <> 'BACKGROUND' and s.username not in ('ORDS','SYS','DBSNMP')
        ORDER BY s.username, s.osuser;
        ```
        
    - Verificar  transações ativas no banco de dados
        
        ```sql
        col name format a10
        col username format a8
        col osuser format a8
        col start_time format a17
        col status format a12
        tti 'Active transactions'
        select s.sid,username,t.start_time, r.name, t.used_ublk "USED BLKS",
        decode(t.space, 'YES', 'SPACE TX',
        decode(t.recursive, 'YES', 'RECURSIVE TX',
        decode(t.noundo, 'YES', 'NO UNDO TX', t.status)
        )) status
        from sys.v_$transaction t, sys.v_$rollname r, sys.v_$session s
        where t.xidusn = r.usn
        and t.ses_addr = s.saddr
        /
        ```
        
    - Relatório diário histórico de sessões ativas
        
        ```sql
        --Relatório_Diário_Histórico_Sessões_Ativas  
        SELECT                            
           SESSION_ID,
           SAMPLE_TIME,
           PROGRAM,
           MACHINE,
           EVENT,
           WAIT_CLASS,
           TIME_WAITED / 100 AS TIME_WAITED_SECONDS, 
           PGA_ALLOCATED / 1024 / 1024 AS PGA_ALLOCATED_MB
        FROM V$ACTIVE_SESSION_HISTORY
        WHERE SAMPLE_TIME BETWEEN TO_DATE('2024-10-04 17:00:00', 'YYYY-MM-DD HH24:MI:SS') 
                              AND TO_DATE('2024-10-04 23:00:00', 'YYYY-MM-DD HH24:MI:SS')
        ORDER BY SAMPLE_TIME ASC;
        ```
        
    - Verificar o PID com base no SID e SERIAL
        
        ```sql
        SELECT
            s.SID,
            s.SERIAL#,
            s. USERNAME,
            s.STATE,
            p.SPID AS OS_PROCESS_ID
        FROM
            V$SESSION s
            JOIN V$PROCESS p ON s.PADDR = p.ADDR
        WHERE
            s.SID = 1645
            AND s.SERIAL# = 48199;
            
            
            
            SELECT s.sid, s.serial#, s.username, s.program, s.sql_id, s.status
        FROM v$session s
        JOIN v$process p ON s.paddr = p.addr
        WHERE p.spid = '102371';
        ```
        
    - Verificar o SERIAL com base no SID
        
        ```sql
        SELECT sid, serial# 
        FROM v$session 
        WHERE sid = <SID>;
        
        ```
        
    - Identificar sessões aguardando recursos
        
        ```sql
        --Identificar sessões aguardando recursos
        SELECT
            s.SID,
            s.SERIAL#,
            s.USERNAME,
            s.STATUS,
            s.MACHINE,
            s.PROGRAM,
            sw.EVENT AS wait_event,
            sw.WAIT_TIME AS wait_time,
            sw.SECONDS_IN_WAIT AS seconds_in_wait
        FROM
            V$SESSION s
            JOIN V$SESSION_WAIT sw ON s.SID = sw.SID
        WHERE
            s.STATUS = 'ACTIVE'
            AND sw.EVENT IS NOT NULL
            AND sw.WAIT_TIME > 0
        ORDER BY
            sw.SECONDS_IN_WAIT DESC;
        
        ```
        
    - Identificar sessões com alto consumo de memória
        
        ```sql
        SELECT
            p.spid,
            p.pid,
            s.sid,                  
            s.serial#,              
            s.username,
            p.program,
            ROUND(p.pga_used_mem / (1024 * 1024 * 1024), 2) AS pga_used_mem_GB,  
            ROUND(p.pga_alloc_mem / (1024 * 1024 * 1024), 2) AS pga_alloc_mem_GB,  
            q.sql_text             
        FROM
            v$process p
            JOIN v$session s ON p.addr = s.paddr
            LEFT JOIN v$sql q ON s.sql_id = q.sql_id  
        WHERE
            p.pga_used_mem > 1000000  -- (Exemplo para filtrar processos com uso de memória maior que 1MB)
        ORDER BY 
            pga_used_mem DESC;
        
        ```
        
    - Verificar sessões ativas com alter disconnect RAC
        
        ```sql
        SELECT * FROM (
            SELECT 
                TO_CHAR(s.logon_time, 'DD/MM HH24:MI:SS') AS loggedon,
                s.sid, 
                s.serial#,
                s.inst_id AS node, 
                s.status,
                FLOOR(s.last_call_et/60) AS "Inativo_Ha_Quantos_Minutos",
                s.username, 
                s.osuser,
                s.sql_id AS last_sql_id, -- Último SQL ID que ela executou antes de dormir
                p.spid AS linux_pid, 
                NVL(s.module, s.program) AS uprogram,
                s.machine, 
                -- Comando pronto para ver o plano do ÚLTIMO SQL (se ainda estiver no cache)
                CASE WHEN s.sql_id IS NOT NULL THEN 
                    'SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY_CURSOR('''|| s.sql_id ||''', NULL, ''ADVANCED''));'
                ELSE 
                    '-- Sem SQL registrado no momento'
                END AS "VER_ULTIMO_CURSOR",
                -- Comando pronto para matar a sessão na instância CORRETA (RAC SAFE)
                'ALTER SYSTEM DISCONNECT SESSION ''' || s.sid || ',' || s.serial# || ',@' || s.inst_id || ''' IMMEDIATE;' AS "COMANDO_KILL"
            FROM 
                gv$session s
            JOIN 
                gv$process p ON (p.addr = s.paddr AND p.inst_id = s.inst_id) -- Correção do JOIN em RAC
            WHERE 
                s.type = 'USER'
                AND s.status = 'INACTIVE'
                AND s.last_call_et >= 7200 -- 2 horas ou mais de inatividade
                AND s.username = 'ICT_TABLES'
            ORDER BY 
                s.last_call_et DESC -- Ordena por quem está inativo há mais tempo
        )
        WHERE ROWNUM < 2000;
        ```
        
    - Verificar sessões ativas OSSDB
        
        ```sql
        SELECT NVL(s.username, '(oracle)') AS username,
               s.sql_id,
               s.osuser,
               s.sid,
               s.serial#,
               p.spid,
               s.lockwait,
               s.status,
               s.service_name,
               s.module,
               s.machine,
               s.program,
               TO_CHAR(s.logon_Time,'DD-MON-YYYY HH24:MI:SS') AS logon_time,
               s.last_call_et AS last_call_et_secs
        FROM   gv$session s,
               gv$process p
        WHERE  s.paddr = p.addr and s.type <> 'BACKGROUND' 
        and s.username not in ('ORDS','SYS','DBSNMP')
        --and s.last_call_et >=3600
        --and s.sql_id is not NULL
        and s.status = 'INACTIVE'
        ORDER BY logon_time desc;
        ```
        
    - Verificar sessões
        
        ```sql
        SELECT NVL(s.username, '(oracle)') AS username,
               s.sql_id,
               s.osuser,
               s.sid,
               s.serial#,
               p.spid,
               ROUND(p.pga_used_mem/1024/1024,2) AS pga_used_mem_mb,
               ROUND(p.pga_alloc_mem/1024/1024,2) AS pga_alloc_mem_mb,
               ROUND(p.pga_freeable_mem/1024/1024,2) AS pga_freeable_mem_mb,
               ROUND(p.pga_max_mem/1024/1024,2) AS pga_max_mem_mb,
               s.lockwait,
               s.status,
               s.service_name,
               s.module,
               s.machine,
               s.program,
               TO_CHAR(s.logon_Time,'DD-MON-YYYY HH24:MI:SS') AS logon_time,
               s.last_call_et AS last_call_et_secs
        FROM   gv$session s,
               gv$process p
        WHERE  s.paddr = p.addr and s.type <> 'BACKGROUND' and s.username not in ('ORDS','SYS','DBSNMP')
        ORDER BY s.username, s.osuser;
        ```
        
    - Verificar sessões II (TOP CPU)
        
        ```sql
        WITH top_cpu AS (
            SELECT
                *
            FROM
                (
                    SELECT
                        s.session_id      AS sid,
                        s.session_serial# AS serial#,
                        s.user_id,
                        s.machine,
                        s.program,
                        s.event,
                        s.sql_id,
                        s.blocking_session,
                        COUNT(*)          AS cpu_seconds
                    FROM
                        v$active_session_history s
                    WHERE
                        s.sample_time BETWEEN sysdate - INTERVAL '1' HOUR AND sysdate
                        AND s.session_state = 'ON CPU'
                        AND s.session_type = 'FOREGROUND'  -- Exclui sessões de background
                    GROUP BY
                        s.session_id,
                        s.session_serial#,
                        s.user_id,
                        s.machine,
                        s.program,
                        s.event,
                        s.sql_id,
                        s.blocking_session
                    ORDER BY
                        cpu_seconds DESC
                )
            WHERE
                ROWNUM <= 10  -- Limitar a 10 resultados
        ), top_io AS (
            SELECT
                *
            FROM
                (
                    SELECT
                        s.session_id      AS sid,
                        s.session_serial# AS serial#,
                        s.user_id,
                        s.machine,
                        s.program,
                        s.event,
                        s.sql_id,
                        s.blocking_session,
                        COUNT(*)          AS io_requests
                    FROM
                        v$active_session_history s
                    WHERE
                        s.sample_time BETWEEN sysdate - INTERVAL '1' HOUR AND sysdate
                        AND s.session_state = 'WAITING'
                        AND s.session_type = 'FOREGROUND'  -- Exclui sessões de background
                        AND s.event LIKE '%db file%'
                    GROUP BY
                        s.session_id,
                        s.session_serial#,
                        s.user_id,
                        s.machine,
                        s.program,
                        s.event,
                        s.sql_id,
                        s.blocking_session
                    ORDER BY
                        io_requests DESC
                )
            WHERE
                ROWNUM <= 10  -- Limitar a 10 resultados
        ), top_wait AS (
            SELECT
                *
            FROM
                (
                    SELECT
                        s.session_id      AS sid,
                        s.session_serial# AS serial#,
                        s.user_id,
                        s.machine,
                        s.program,
                        s.event,
                        s.sql_id,
                        s.blocking_session,
                        COUNT(*)          AS wait_seconds
                    FROM
                        v$active_session_history s
                    WHERE
                        s.sample_time BETWEEN sysdate - INTERVAL '1' HOUR AND sysdate
                        AND s.session_state = 'WAITING'
                        AND s.session_type = 'FOREGROUND'  -- Exclui sessões de background
                    GROUP BY
                        s.session_id,
                        s.session_serial#,
                        s.user_id,
                        s.machine,
                        s.program,
                        s.event,
                        s.sql_id,
                        s.blocking_session
                    ORDER BY
                        wait_seconds DESC
                )
            WHERE
                ROWNUM <= 10  -- Limitar a 10 resultados
        )
        SELECT
            'CPU Aggressor' AS "Aggressor Type",
            tc.sid,
            tc.serial#,
            tc.user_id,
            tc.machine,
            tc.program,
            tc.event,
            tc.sql_id,
            tc.blocking_session,
            tc.cpu_seconds  AS "Metric"
        FROM
            top_cpu tc
        UNION ALL
        SELECT
            'IO Aggressor' AS "Aggressor Type",
            ti.sid,
            ti.serial#,
            ti.user_id,
            ti.machine,
            ti.program,
            ti.event,
            ti.sql_id,
            ti.blocking_session,
            ti.io_requests AS "Metric"
        FROM
            top_io ti
        UNION ALL
        SELECT
            'Wait Time Aggressor' AS "Aggressor Type",
            tw.sid,
            tw.serial#,
            tw.user_id,
            tw.machine,
            tw.program,
            tw.event,
            tw.sql_id,
            tw.blocking_session,
            tw.wait_seconds       AS "Metric"
        FROM
            top_wait tw
        ORDER BY
            "Metric" DESC;
        ```
        
    - Identificar bloqueios específicos
        
        ```sql
        --Identificar bloqueios específicos com status das sessões    
        SELECT
            s1.SID AS waiting_sid,
            s1.SERIAL# AS waiting_serial,
            s1.STATUS AS waiting_status,
            s2.SID AS holding_sid,
            s2.SERIAL# AS holding_serial,
            s2.STATUS AS holding_status,
            l1.TYPE AS lock_type,
            l2.TYPE AS held_lock_type
        FROM
            V$LOCK l1
            JOIN V$SESSION s1 ON l1.SID = s1.SID
            JOIN V$LOCK l2 ON l1.BLOCK = l2.SID
            JOIN V$SESSION s2 ON l2.SID = s2.SID
        WHERE
            l1.SID <> l2.SID
        ORDER BY
            s1.SID, s2.SID;
            
            
            
        --Siglas coluna LOCK_TYPE
          
        AE (Automatic Exponential Backoff): Este tipo de bloqueio usa um mecanismo de retrocesso exponencial automático. Se um bloqueio é solicitado e não pode ser imediatamente adquirido, o sistema tenta novamente após um intervalo de tempo que aumenta exponencialmente com cada tentativa falhada.
        
        KT (Kernel Transaction): Refere-se a um bloqueio associado a uma transação do kernel. Em geral, isso está relacionado a bloqueios internos do Oracle que são usados para gerenciar o acesso a recursos críticos.
        
        MR (Multi-Row): Indica um bloqueio que afeta múltiplas linhas. Esses bloqueios são usados para garantir a consistência dos dados quando várias linhas de uma tabela são manipuladas em uma única operação.
        
        RT (Row Table): Relacionado a bloqueios de nível de linha em uma tabela. Esse bloqueio garante que uma linha específica em uma tabela não seja modificada por outra transação enquanto está sendo processada.
        
        TM (Table Manipulation): Refere-se a bloqueios que são aplicados quando uma tabela está sendo manipulada. Isso pode incluir operações como inserções, atualizações ou exclusões que afetam a estrutura da tabela.
        
        TX (Transaction): Representa bloqueios de transação. Esses bloqueios garantem a integridade das transações, impedindo que duas transações diferentes façam alterações conflitantes nos mesmos dados simultaneamente.
        
        XR (Exclusive Row): Refere-se a bloqueios exclusivos em uma linha. Esse tipo de bloqueio impede que outras transações acessem ou modifiquem uma linha específica enquanto o bloqueio estiver em vigor.  
        ```
        
    - Consultar objetos bloqueados por usuários
        
        ```sql
        -- consultar objetos bloqueados por usuario
        SELECT    LO.SESSION_ID, LO.PROCESS, LO.ORACLE_USERNAME, O.OWNER, O.OBJECT_NAME
        FROM      V$LOCKED_OBJECT LO
        JOIN      DBA_OBJECTS O
          ON      O.OBJECT_ID = LO.OBJECT_ID;
        
        -- consultar locks que estao bloqueando outras sessoes
        SELECT  L.SESSION_ID, L.LOCK_TYPE, L.MODE_HELD, L.LOCK_ID1, L.LOCK_ID2, L.BLOCKING_OTHERS
        FROM    DBA_LOCKS L
        WHERE   L.BLOCKING_OTHERS <> 'Not Blocking';
        
        -- consultar sessoes bloqueadoras
        select * from dba_blockers;
        
        -- consultar sessoes bloqueadas
        select * from dba_waiters;
        
        select      /*+ ALL_ROWS */
                    b.sid, 
                    c.username, 
                    c.osuser, 
                    c.terminal, 
                    c.status, 
                    a.owner, 
                    decode (NVL (b.id2, 0), 0, a.object_name, 'Trans-'||to_char(b.id1)) object_name, 
                    b.type,
                    decode (NVL (b.lmode, 0), 0, '--Waiting--',
                                      1, 'Null',
                                      2, 'Row Share',
                                      3, 'Row Excl',
                                      4, 'Share',
                                      5, 'Sha Row Exc',
                                      6, 'Exclusive',
                                      'Other') "Lock Mode",
                    decode(NVL (b.request, 0), 0, ' - ',
                                      1, 'Null',
                                      2, 'Row Share',
                                      3, 'Row Excl',
                                      4, 'Share',
                                      5, 'Sha Row Exc',
                                      6, 'Exclusive',
                                      'Other') "Req Mode"
        from          dba_objects a
        right join    v$lock b
            on        a.object_id = b.id1
        inner join    v$session c
            on        b.sid = c.sid
        where         c.username is not null;
        -- order by b.sid, b.id2 
        ```
        
    - Verificar sessões inativas e desconectar
        
        ```sql
        select * from (
        select to_char(s.logon_time, 'mm/dd hh:mi:ssAM') loggedon,
        s.sid, s.status,
        floor(last_call_et/60) "LastCallET",
        s.username, s.osuser,s.sql_id,
        p.spid, s.module || ' ' || s.program uprogram,
        s.machine, s.sql_hash_value, 'SELECT * FROM TABLE (DBMS_XPLAN.display_cursor('''
        || s.sql_id
        || ''','
        || 'NULL,''ADVANCED +ALLSTATS LAST +MEMSTATS LAST''));'
        "CURSOR"
        ,'ALTER SYSTEM DISCONNECT SESSION '||''''||s.sid||','||s.serial#||''' IMMEDIATE;'
        from gv$session s, gv$process p
        where p.addr = s.paddr
        and s.type = 'USER'
        and module is not null
        and s.status = 'INACTIVE'
        and s.last_call_et >=3600
        -- and s.username = 'BROKERNW'
        --and SID = '395'
        --and SCHEMANAME = 'ARADMIN'
        --and p.spid =41572
        --and SQL_ID = 'gb1uu85cm9art'
        --and MACHINE = 'servicenowsne12'
        order by 3 desc)
        where rownum < 2000;
        ```
        
    - Verificar se as sessões foram desativadas
        
        ```sql
        SELECT inst_id, sid, serial#, status, username, program, machine
        FROM gv$session
        WHERE (sid, serial#) IN ((1893, 48063), (381, 57220));
        ```
        
    - Verificar sessões inativas (por tempo de inatividade)
        
        ```sql
        --sessões de usuários no banco de dados que estão inativas há mais de X horas / com disconnect--
        SELECT
            s.sid,
            s.serial#,
            s.username,
            s.osuser,
            s.machine,
            s.type,
            s.status,
            s.program,
            ROUND(last_call_et/60) as minutes_since_last_call,
            'ALTER SYSTEM DISCONNECT SESSION ' || s.sid || ',' || s.serial# || ' IMMEDIATE;' as disconnect_command
        FROM
            gv$session s
        WHERE
            s.type = 'USER' -- Sessões de usuários
            AND s.status = 'INACTIVE' -- Sessões inativas
            AND last_call_et >= 7200 -- Tempo de inatividade maior ou igual a X horas (em segundos)
        ORDER BY
            last_call_et DESC;
        ```
        
    - Encontrar a sessão pelo SQL_ID
        
        ```sql
        select INST_ID,sid,username,program,status,machine, to_char(logon_time,'DD_MON_YYYY HH24:MI:SS') LOGON_TIME,sql_id from gv$session where sql_ID='d86r1gv6yzaup';
        
        ou
        
        select SQL_ID,sql_text FROM V$SQL WHERE SQL_ID='d86r1gv6yzaup';
        
        ou
        
        select * from v$undostat order by begin_time;
        
        ou
        
        select a.sid, a.serial#, a.username, b.used_urec used_undo_record, b.used_ublk used_undo_blocks
        from v$session a, v$transaction b
        where a.saddr=b.ses_addr ;
        
        ou
        
        select USER_ID from V$ACTIVE_SESSION_HISTORY where SQL_ID = 'brtbr31x31k9x';
        
        ou
        
        select USER_ID from DBA_HIST_ACTIVE_SESS_HISTORY where SQL_ID = 'brtbr31x31k9x';
        ```
        
    - Matar sessões
        
        ```sql
        Localizar a sessão:
        select sid, serial#, username from v$session where username='BD2COP';
        
        Matar sessão:
        alter system kill session '582,33757';
        
        alter system disconnect session '3664,50800' immediate; Utilizar sempre esse
        
        Meios alternativos:
        
        -> O select abaixo identifica as sessões em ambiente single instance:
        
        SELECT s.sid,
        s.serial#,
        p.spid,
        s.username,
        s.program
        FROM   v$session s
        JOIN v$process p
        ON p.addr = s.paddr
        WHERE  s.type != ‘BACKGROUND’;
        
        -> O select abaixo identifica as sessões em ambiente RAC:
        
        SELECT s.inst_id,
        s.sid,
        s.serial#,
        p.spid,
        s.username,
        s.program
        FROM   gv$session s
        JOIN gv$process p
        ON p.addr = s.paddr
        AND p.inst_id = s.inst_id
        WHERE  s.type != ‘BACKGROUND’;
        
        -> Essa opção permite que se remova uma sessão de um Oracle RAC mesmo estando conectado num nó diferente.
        
        ALTER SYSTEM KILL SESSION ‘SID,SERIAL#,@INST_ID’;
        
        -> Segue abaixo o exemplo da sintaxe ALTER … KILL SESSION IMMEDIATE:
        
        ALTER SYSTEM KILL SESSION ‘SID,SERIAL#’  IMMEDIATE; – para ambiente   single instance
        ALTER SYSTEM KILL SESSION ‘SID,SERIAL#,@INST_ID’  IMMEDIATE;  para ambiente RAC
        
        -> Usando o comando comando ALTER SYSTEM KILL DISCONNECT SESSION
        
        ALTER SYSTEM KILL DISCONNECT SESSION ‘SID,SERIAL#’  IMMEDIATE;
        ALTER SYSTEM KILL DISCONNECT SESSION ‘SID,SERIAL#’  POST_TRANSACTION;
        
        para ambiente RAC executamos o seguinte commando:
        
        ALTER SYSTEM KILL DISCONNECT SESSION ‘SID,SERIAL#,@INST_ID’ IMMEDIATE;
        
        ```
        
    - C**onsulta para obter os 5 principais SQL com uso intensivo de recursos**
        
        ```sql
        SET PAGESIZE 1000
        SET LINESIZE 400
        COLUMN "SQL" FOR A120
        COLUMN "Rank" FOR 99999
        COLUMN "Snap Day" FOR A10
        COLUMN "Execs" FOR 999999999
        SELECT *
          FROM (
                SELECT RANK() OVER (
                         PARTITION BY "Snap Day"
                         ORDER BY "Buffer Gets" + "Disk Reads" DESC
                       ) AS "Rank",
                       i1.*
                  FROM (
                        SELECT TO_CHAR(hs.begin_interval_time, 'MM/DD/YY') AS "Snap Day",
                               SUM(shs.executions_delta) AS "Execs",
                               SUM(shs.buffer_gets_delta) AS "Buffer Gets",
                               SUM(shs.disk_reads_delta) AS "Disk Reads",
                               ROUND(SUM(shs.buffer_gets_delta) / NULLIF(SUM(shs.executions_delta), 0), 1) AS "Gets/Exec",
                               ROUND(SUM(shs.cpu_time_delta) / 1000000 / NULLIF(SUM(shs.executions_delta), 0), 1) AS "CPU/Exec(S)",
                               ROUND(SUM(shs.iowait_delta) / 1000000 / NULLIF(SUM(shs.executions_delta), 0), 1) AS "IO/Exec(S)",
                               shs.sql_id AS "Sql id",
                               REPLACE(DBMS_LOB.SUBSTR(sht.sql_text, 120, 1), CHR(10), ' ') AS "SQL"
                          FROM dba_hist_sqlstat shs
                          JOIN dba_hist_sqltext sht ON sht.sql_id = shs.sql_id
                          JOIN dba_hist_snapshot hs ON shs.snap_id = hs.snap_id
                         WHERE shs.executions_delta > 0
                      GROUP BY shs.sql_id,
                               TO_CHAR(hs.begin_interval_time, 'MM/DD/YY'),
                               DBMS_LOB.SUBSTR(sht.sql_text, 120, 1)
                     ) i1
               )
         WHERE "Rank" <= 10
           AND "Snap Day" = TO_CHAR(SYSDATE, 'MM/DD/YY');
        
        ```
        
    - Gerar plano de execução de uma query
        
        ```sql
        SELECT * FROM table(DBMS_XPLAN.DISPLAY_CURSOR('bf75j1pm2swqh',0));
        ```
        
- PDB
    - COMANDOS PDB
        
        ```sql
        SE O PDB ESTIVER EM ESTADO MOUNTED
        	ALTER PLUGGABLE DATABASE TIMAUDDB OPEN;
        	ALTER PLUGGABLE DATABASE TIMAUDDB CLOSE IMMEDIATE;
        	DROP PLUGGABLE DATABASE NFLOWHOMOLD INCLUDING DATAFILES;
        
        ATIVAR MAIS DE UM PDB
        	ALTER PLUGGABLE ALL DATABASES OPEN;
        
        CONECTAR AO PDB
        	ALTER SESSION SET CONTAINER=<NOME DO PDB>;
        ```
        
    - Verificar ID e nome do PDB
        
        ```sql
        select con_id, name from v$containers;
        ```
        
    - Verificar o estado do PDB
        
        ```sql
        set lines 200 pages 200
        select name, open_mode from v$pdbs;
        
        https://oraclehome.com.br/2017/03/21/renomeando-um-pluggable-database-pdb/
        ```
        
    - Verificar informações do PDB e tamanho
        
        ```sql
        SELECT 
          a.con_id, 
          a.name, 
          b.status, 
          a.open_mode, 
          ROUND(a.total_size / POWER(1024, 3), 2) AS total_size_gb
        FROM 
          v$pdbs a, 
          dba_pdbs b 
        WHERE 
          a.con_id = b.pdb_id;
        ```
        
- ARCHIVELOG
    - SCRIPT PARA VERIFICAR GERAÇÃO DE ARCHIVES POR HORA
        
        ```sql
        --verifica geração de archives por hora (30 dias)
        set lines 299
        SELECT TO_CHAR(TRUNC(FIRST_TIME),'Mon DD') "Date",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'00',1,0)),'9999') "12AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'01',1,0)),'9999') "01AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'02',1,0)),'9999') "02AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'03',1,0)),'9999') "03AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'04',1,0)),'9999') "04AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'05',1,0)),'9999') "05AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'06',1,0)),'9999') "06AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'07',1,0)),'9999') "07AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'08',1,0)),'9999') "08AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'09',1,0)),'9999') "09AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'10',1,0)),'9999') "10AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'11',1,0)),'9999') "11AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'12',1,0)),'9999') "12PM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'13',1,0)),'9999') "1PM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'14',1,0)),'9999') "2PM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'15',1,0)),'9999') "3PM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'16',1,0)),'9999') "4PM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'17',1,0)),'9999') "5PM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'18',1,0)),'9999') "6PM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'19',1,0)),'9999') "7PM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'20',1,0)),'9999') "8PM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'21',1,0)),'9999') "9PM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'22',1,0)),'9999') "10PM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'23',1,0)),'9999') "11PM"
        FROM V$LOG_HISTORY
        GROUP BY TRUNC(FIRST_TIME)
        ORDER BY TRUNC(FIRST_TIME) DESC
        /
        
        *******
        
        --verifica geração de archives por hora (últimos 10 dias)
        SET LINES 299
        SELECT TO_CHAR(TRUNC(FIRST_TIME),'Mon DD') "Date",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'00',1,0)),'9999') "12AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'01',1,0)),'9999') "01AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'02',1,0)),'9999') "02AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'03',1,0)),'9999') "03AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'04',1,0)),'9999') "04AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'05',1,0)),'9999') "05AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'06',1,0)),'9999') "06AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'07',1,0)),'9999') "07AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'08',1,0)),'9999') "08AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'09',1,0)),'9999') "09AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'10',1,0)),'9999') "10AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'11',1,0)),'9999') "11AM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'12',1,0)),'9999') "12PM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'13',1,0)),'9999') "1PM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'14',1,0)),'9999') "2PM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'15',1,0)),'9999') "3PM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'16',1,0)),'9999') "4PM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'17',1,0)),'9999') "5PM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'18',1,0)),'9999') "6PM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'19',1,0)),'9999') "7PM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'20',1,0)),'9999') "8PM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'21',1,0)),'9999') "9PM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'22',1,0)),'9999') "10PM",
        TO_CHAR(SUM(DECODE(TO_CHAR(FIRST_TIME,'HH24'),'23',1,0)),'9999') "11PM"
        FROM V$LOG_HISTORY
        WHERE TRUNC(FIRST_TIME) >= TRUNC(SYSDATE) - 9
        GROUP BY TRUNC(FIRST_TIME)
        ORDER BY TRUNC(FIRST_TIME) DESC
        /
        
        ```
        
    - SCRIPT PARA VERIFICAR O TAMANHO DOS ARCHIVES
        
        ```sql
        SELECT 
            SEQUENCE#, 
            NAME AS "Arquivo", 
            DEST_ID, 
            TO_CHAR(COMPLETION_TIME, 'DD/MM/YYYY HH24:MI:SS') AS "Data de Conclusão",
            BLOCKS * (SELECT value FROM v$parameter WHERE name = 'db_block_size') / 1024 / 1024 / 1024 AS "Tamanho (GB)"
        FROM 
            V$ARCHIVED_LOG
        WHERE 
            NAME IS NOT NULL;
        ```
        
    - Alterar parâmetro DB_RECOVERY_FILE_DEST_SIZE
        
        ```sql
        alter system set db_recovery_file_dest_size=75G scope=both;
        ```
        
    - Verificar último archivelog aplicado (Primário e standby)
        
        ```sql
        --No primário
        COLUMN HOST_NAME       FORMAT A25 HEADING 'Servidor'
        COLUMN THREAD#         FORMAT 999 HEADING 'Thread'
        COLUMN LAST_GENERATED  FORMAT 9999999 HEADING 'Último|Gerado'
        SELECT i.HOST_NAME,
               a.THREAD#,
               MAX(a.SEQUENCE#) AS LAST_GENERATED
        FROM   V$ARCHIVED_LOG a,
               V$INSTANCE i
        WHERE  a.RESETLOGS_CHANGE# = (
                  SELECT MAX(RESETLOGS_CHANGE#)
                  FROM V$ARCHIVED_LOG
               )
        GROUP  BY i.HOST_NAME, a.THREAD#
        ORDER  BY a.THREAD#;
        
        --No standby
        
        COLUMN HOST_NAME    FORMAT A25 HEADING 'Servidor'
        COLUMN THREAD#      FORMAT 999 HEADING 'Thread'
        COLUMN LAST_APPLIED FORMAT 9999999 HEADING 'Último|Aplicado'
        SELECT i.HOST_NAME,
               a.THREAD#,
               MAX(a.SEQUENCE#) AS LAST_APPLIED
        FROM   V$ARCHIVED_LOG a,
               V$INSTANCE i
        WHERE  a.APPLIED = 'YES'
        GROUP  BY i.HOST_NAME, a.THREAD#
        ORDER  BY a.THREAD#;
        
        ```
        
- TABELA
    - S**tatus de todos os índices da tabela**
        
        ```sql
        select index_name,status from dba_indexes where table_name like ‘&table_name’;
        ```
        
    - Verificar estatísticas de uma tabela
        
        ```sql
        SELECT
            owner,
            table_name,
            num_rows,
            blocks,
            last_analyzed
        FROM
            all_tables
        WHERE
            owner = 'INFOACESSO'
            AND table_name = 'USER_SEARCH_PLATFORM';
        ```
        
    - Verificar estatísticas de index
        
        ```sql
        SELECT
            a.index_name,
            b.blevel,
            b.leaf_blocks,
            b.clustering_factor
        FROM
            all_indexes a,
            all_ind_statistics b
        WHERE
            a.owner = 'INFOACESSO'
            AND a.table_name = 'USER_SEARCH_PLATFORM'
            AND a.owner = b.owner
            AND a.index_name = b.index_name;
        ```
        
    - Acompanhar uso da TEMP em um rebuild de index
        
        ```sql
        --Verificar o uso da TEMP
        SELECT s.sid, s.serial#, s.username, t.blocks, t.segtype, t.contents
        FROM v$sort_usage t
        JOIN v$session s ON t.session_addr = s.saddr
        ORDER BY t.blocks DESC;
        
        -- progresso do rebuild (v$session_longops)
        SELECT sid, serial#, opname, sofar, totalwork,
               ROUND(sofar/NULLIF(totalwork,0)*100,2) pct
        FROM v$session_longops
        WHERE sid = 10161;
        ```
        
    - Criar procedure para apagar dados
        
        ```sql
        CREATE OR REPLACE PROCEDURE expurgar_tabela (
            p_tabela IN VARCHAR2,
            p_data_inicio IN DATE,
            p_data_fim IN DATE
        )
        IS
            v_mes_atual DATE := TRUNC(p_data_inicio, 'MM'); -- Primeiro dia do mês de p_data_inicio
            v_data_final_mes DATE;
        BEGIN
            WHILE v_mes_atual <= p_data_fim LOOP
                -- Determinar o último dia do mês
                v_data_final_mes := LAST_DAY(v_mes_atual);
                
                -- Excluir dados da tabela para o mês atual
                EXECUTE IMMEDIATE 'DELETE FROM ' || p_tabela ||
                                  ' WHERE data_coluna >= :data_inicio AND data_coluna <= :data_fim' -- Definir data_coluna de acordo com a tabela
                                  USING v_mes_atual, v_data_final_mes;
                
                -- Avançar para o próximo mês
                v_mes_atual := ADD_MONTHS(v_mes_atual, 1);
            END LOOP;
        END;
        /
        ```
        
    - Verificar modificações nas tabelas
        
        ```sql
        SELECT 
            TABLE_NAME,
            (INSERTS + UPDATES + DELETES) AS TOTAL_MODIFICACOES
        FROM 
            DBA_TAB_MODIFICATIONS;
            
        *************************************    
         SELECT 
            TABLE_NAME,
            INSERTS,
            UPDATES,
            DELETES
        FROM 
            DBA_TAB_MODIFICATIONS;
        ```
        
    - Verificar quantidade de tabelas que contém HML ou DEV
        
        ```sql
        --Quantidade de tabelas cujo nome contém "HML" ou "DEV"
        SELECT owner, table_name, 
               ROUND((num_rows * avg_row_len / 1024 / 1024), 2) AS size_MB, 
               num_rows
        FROM all_tables
        WHERE table_name LIKE '%HML%' OR table_name LIKE '%DEV%'
        ORDER BY owner, table_name;
        
        --Quantidade de tabelas por schema cujo nome contém "HML" ou "DEV", ordenadas pela maior contagem.
        SELECT owner, COUNT(*) AS table_count
        FROM all_tables
        WHERE table_name LIKE '%HML%' OR table_name LIKE '%DEV%'
        GROUP BY owner
        ORDER BY table_count DESC;
        
        SELECT owner, table_name, 
               ROUND((num_rows * avg_row_len / 1024 / 1024), 2) AS size_MB, 
               num_rows
        FROM all_tables
        WHERE REGEXP_LIKE(table_name, 'HML(_OLD)?|_H')
        ORDER BY owner, table_name;
        
        SELECT owner, table_name, 
               ROUND((num_rows * avg_row_len / 1024 / 1024), 2) AS size_MB, 
               num_rows
        FROM all_tables
        WHERE REGEXP_LIKE(table_name, 'HML(_OLD)?|_H$')
        --AND OWNER = 'BROKERNW'
        ORDER BY owner, table_name;
        ```
        
    - Verificar dados de auditoria (DBA_AUDIT_TRAIL)
        
        ```sql
        SELECT 
            TO_CHAR(s.logon_time, 'MM/DD HH:MI:SSAM') loggedon,
            s.inst_id,
            s.sid, 
            s.osuser,
            s.sql_id,
            p.spid,
            s.machine,
            da.SQL_TEXT
          FROM 
            gv$session s, 
            gv$process p,
            DBA_AUDIT_TRAIL da
          WHERE 
            da.OS_PROCESS = p.spid
            and
            p.addr = s.paddr
            AND s.type = 'USER'
          ORDER BY 
            3 DESC;
        ```
        
    - Listar as tabelas para REVOKE
        
        ```sql
        SELECT 'REVOKE ' || privilege || ' ON ' || owner || '.' || table_name || ' FROM ' || grantee || ';' AS revogar,
            grantee, owner, table_name, privilege
        FROM dba_tab_privs
        WHERE table_name LIKE '%HML'
           OR table_name LIKE '%_HML'
           OR table_name LIKE '%_H'
           OR table_name LIKE '%_DEV'
           OR table_name LIKE '%_D'
        ORDER BY grantee, owner, table_name;
        
        --seleciona tabelas que terminam com os prefixos informados
        SELECT 'REVOKE ' || privilege || ' ON ' || owner || '.' || table_name || ' FROM ' || grantee || ';' AS revogar,
               grantee, owner, table_name, privilege
        FROM dba_tab_privs
        WHERE REGEXP_LIKE(table_name, '(HML|_HML|_H|_DEV|_D)$')
        ORDER BY grantee, owner, table_name;
        ```
        
    - Criar INDEX
        
        ```sql
        CREATE INDEX INTERMITENT.IDX_ALARM_SERVERSERIAL ON INTERMITENT.ICT_INTERMITTENT_ALARMS_HIST (SERVERSERIAL) ONLINE COMPUTE STATISTICS NOLOGGING;
        ```
        
    - Monitoramento de INDEX
        
        ```sql
        Ativar monitoramento de todos os INDEX
        
        DECLARE
            v_table_name  VARCHAR2(30) := 'MY_TABLE';
            v_table_owner VARCHAR2(30) := 'MY_OWNER';
        BEGIN
            FOR idx IN (
                SELECT INDEX_NAME, TABLE_NAME, TABLE_OWNER
                FROM ALL_INDEXES
                WHERE TABLE_NAME = v_table_name
                  AND TABLE_OWNER = v_table_owner
            ) LOOP
                EXECUTE IMMEDIATE 'ALTER INDEX ' || idx.TABLE_OWNER || '.' || idx.INDEX_NAME || ' MONITORING USAGE';
            END LOOP;
        END;
        /
        
        ****Ativar monitoramento de INDEX específico: ALTER INDEX schema.nome_do_index MONITORING USAGE;
        
        O bloco abaixo é para cancelar o monitoramento de todos os INDEX
        
        DECLARE
            v_table_name  VARCHAR2(30) := 'MY_TABLE';
            v_table_owner VARCHAR2(30) := 'MY_OWNER';
        BEGIN
            FOR idx IN (
                SELECT INDEX_NAME, TABLE_NAME, TABLE_OWNER
                FROM ALL_INDEXES
                WHERE TABLE_NAME = v_table_name
                  AND TABLE_OWNER = v_table_owner
            ) LOOP
                EXECUTE IMMEDIATE 'ALTER INDEX ' || idx.TABLE_OWNER || '.' || idx.INDEX_NAME || ' NOMONITORING USAGE';
            END LOOP;
        END;
        /
        
        ****Cancelar monitoramento de INDEX específico: ALTER INDEX schema.nome_do_index NOMONITORING USAGE;
        
        ```
        
    - Verificar INDEX com problema
        
        ```sql
        Ativar monitoramento de todos os INDEX
        
        DECLARE
            v_table_name  VARCHAR2(30) := 'MY_TABLE';
            v_table_owner VARCHAR2(30) := 'MY_OWNER';
        BEGIN
            FOR idx IN (
                SELECT INDEX_NAME, TABLE_NAME, TABLE_OWNER
                FROM ALL_INDEXES
                WHERE TABLE_NAME = v_table_name
                  AND TABLE_OWNER = v_table_owner
            ) LOOP
                EXECUTE IMMEDIATE 'ALTER INDEX ' || idx.TABLE_OWNER || '.' || idx.INDEX_NAME || ' MONITORING USAGE';
            END LOOP;
        END;
        /
        
        ****Ativar monitoramento de INDEX específico: ALTER INDEX schema.nome_do_index MONITORING USAGE;
        
        O bloco abaixo é para cancelar o monitoramento de todos os INDEX
        
        DECLARE
            v_table_name  VARCHAR2(30) := 'MY_TABLE';
            v_table_owner VARCHAR2(30) := 'MY_OWNER';
        BEGIN
            FOR idx IN (
                SELECT INDEX_NAME, TABLE_NAME, TABLE_OWNER
                FROM ALL_INDEXES
                WHERE TABLE_NAME = v_table_name
                  AND TABLE_OWNER = v_table_owner
            ) LOOP
                EXECUTE IMMEDIATE 'ALTER INDEX ' || idx.TABLE_OWNER || '.' || idx.INDEX_NAME || ' NOMONITORING USAGE';
            END LOOP;
        END;
        /
        
        ****Cancelar monitoramento de INDEX específico: ALTER INDEX schema.nome_do_index NOMONITORING USAGE;
        
        ```
        
    - Verificar qual a tabela que o INDEX pertence
        
        ```sql
        --Verificar a qual tabela que o index pertence
        SELECT 
            index_name,
            tablespace_name,
            owner,
            table_name,
            uniqueness
        FROM 
            dba_indexes
        WHERE 
            index_name IN ('IDX_CELLNAME', 
                           'IDX_ALARM_TYPE_UPPER');
                           
        --Verificar qual schema que o index pertence
        SELECT 
            owner, 
            table_name
        FROM 
            dba_tables
        WHERE 
            table_name in ('PURGE_ALTAIA',
            'ICT_ALARM_FILTERS');
        ```
        
    - Consulta retorna os dados de uma tabela baseado por tempo
        
        ```sql
        select * from REPORTER.REPORTER_STATUS netcool0_ where netcool0_.ICTCREATIONTIMESTAMP>= SYSDATE -(5/1440);
        ```
        
    - Apagar registros de uma tabela específica (por período)
        
        ```sql
        DELETE FROM <owner>.<tabela>
        WHERE CREATED >= TO_DATE('2019-12-01', 'YYYY-MM-DD')--INÍCIO
        AND CREATED <= TO_DATE('2019-12-31', 'YYYY-MM-DD');
        
        commit;
        ```
        
    - Script para coletar estatísticas de todas as tabelas e index de um schema
        
        ```sql
        DECLARE
            ownname VARCHAR2(30) := 'SENSOR';
        BEGIN
            -- Coleta de estatísticas para todas as tabelas e índices do schema
            DBMS_STATS.GATHER_SCHEMA_STATS(
                ownname => ownname,
                estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE,
                method_opt => 'FOR ALL COLUMNS SIZE AUTO',
                degree => DBMS_STATS.DEFAULT_DEGREE,
                granularity => 'ALL',
                cascade => TRUE
            );
        END;
        /
        ```
        
    - Script para justificar um shrink
        
        ```sql
        SELECT segment_name,
                round(allocated_space/1024/1024,1) alloc_mb,
                round( used_space/1024/1024, 1 ) used_mb,
                    round( reclaimable_space/1024/1024) reclaim_mb,
                    round(reclaimable_space/allocated_space*100,0) pctsave,
                    recommendations
           FROM TABLE(dbms_space.asa_recommendations())
          where segment_owner = 'SENSOR'
          order by PCTSAVE DESC;
        ```
        
    - Verificar a quantidade de tabelas em um schema
        
        ```sql
        SELECT COUNT(*) FROM all_tables WHERE owner = 'MLOG_PROD';
        ```
        
    - Consultar index conforme estatísticas do BD
        
        ```sql
        -- consulta TAMANHO dos indices conforme estatisticas do BD
        SELECT i.owner,
               i.index_name AS "Index Name",
               nvl(i.num_rows, 0) AS "Rows",
               ROUND((nvl(i.leaf_blocks, 0) * p.value) / 1024 / 1024, 2) AS "Size MB",
               i.last_analyzed AS "Last Analyzed",
               s.tablespace_name AS "Tablespace Name"
        FROM   dba_indexes i
               JOIN dba_segments s
               ON i.index_name = s.segment_name
               AND i.owner = s.owner
               JOIN v$parameter p
               ON p.name = 'db_block_size'
        WHERE  i.owner = UPPER(NVL('SENSOR', i.owner))
        UNION ALL
        SELECT i.owner,
               NULL,
               NULL,
               SUM(ROUND((nvl(i.leaf_blocks, 0) * p.value) / 1024 / 1024, 2)) AS "Size MB",
               NULL,
               s.tablespace_name AS "Tablespace Name"
        FROM   dba_indexes i
               JOIN dba_segments s
               ON i.index_name = s.segment_name
               AND i.owner = s.owner
               JOIN v$parameter p
               ON p.name = 'db_block_size'
        WHERE  i.owner = UPPER(NVL('SENSOR', i.owner))
        GROUP BY i.owner, s.tablespace_name
        ORDER BY "Size MB" DESC;
        ```
        
    - Verificar tamanho de um INDEX
        
        ```sql
        SELECT
            INDEX_NAME,
            ROUND(SUM(BYTES) / 1024 / 1024, 2) AS index_size_mb
        FROM
            DBA_SEGMENTS
        WHERE
            SEGMENT_TYPE = 'INDEX'
            AND INDEX_NAME = 'IDX_YOUR_INDEX_NAME'
        GROUP BY
            INDEX_NAME;
        ```
        
    - Evidência criação de index
        
        ```sql
        SELECT
            i.owner AS index_owner,
            i.index_name,
            i.table_name,
            c.column_name
        FROM
            all_ind_columns c
        JOIN
            all_indexes i ON c.index_name = i.index_name AND c.index_owner = i.owner
        WHERE
            i.table_name = 'REPORTER_STATUS'
            AND i.index_name = 'IDX_NENAME';
        ```
        
    - Consultar index/coluna
        
        ```sql
        select        I.OWNER, 
                      I.TABLE_NAME,
                      C.COLUMN_NAME,
                      i.index_name,
                      I.index_type
        FROM          DBA_INDEXES I
        INNER JOIN    dba_ind_columns C
            ON        I.OWNER = C.index_owner
            AND       I.table_name = c.table_name
            AND       I.index_name = C.index_name
        WHERE         I.OWNER = UPPER('INTERMITENT');
        ```
        
    - Consultar index na mesma coluna
        
        ```sql
        select      TABLE_OWNER,
                    TABLE_NAME,
                    COLUMN_NAME,
                    COUNT(1) QTDE_INDEXES
        from        all_ind_columns 
        where       TABLE_OWNER not in ('SYS','SYSTEM')
        group by    TABLE_OWNER, TABLE_NAME, COLUMN_NAME
        HAVING      COUNT(1) > 1
        ORDER BY    1, 2;
        ```
        
    - Verificar INDEXES que foram utilizados
        
        ```sql
         --verificar indexes que foram utilizados 
          SELECT 
            INDEX_NAME, 
            TABLE_NAME, 
            MONITORING, 
            USED, 
            START_MONITORING, 
            END_MONITORING 
        FROM 
            V$OBJECT_USAGE 
        WHERE 
            INDEX_NAME = 'S_EVT_ACT_M55';
        ```
        
    - Verificar espaço ocupado pelos INDEXES
        
        ```sql
        --verificar espaço ocupado pelos indexes
        SELECT s.segment_name,
               s.segment_type,
               s.tablespace_name,
               s.bytes / 1024 / 1024 / 1024 AS size_in_gb,
               i.index_type
        FROM dba_segments s
        JOIN all_indexes i
        ON s.segment_name = i.index_name
        AND s.owner = i.owner
        WHERE s.segment_type = 'INDEX'
        AND s.segment_name IN (
            SELECT index_name
            FROM all_indexes
            WHERE table_name = 'S_EVT_ACT_SBELPOS2'
            AND tablespace_name = 'SIEBEL_DATA'
            AND owner = 'SENSOR'
        );
        ```
        
    - Verificar tabelas em lock
        
        ```sql
        SELECT l.session_id||','||v.serial# sid_serial,
         l.ORACLE_USERNAME ora_user,
         l.OS_USER_NAME,
         o.object_name, 
         o.object_type, 
         DECODE(l.locked_mode,
         0, 'None',
         1, 'Null',
         2, 'Row-S (SS)',
         3, 'Row-X (SX)',
         4, 'Share',
         5, 'S/Row-X (SSX)',
         6, 'Exclusive', 
         TO_CHAR(l.locked_mode)
         ) lock_mode,
         o.status,  
         to_char(o.last_ddl_time,'dd.mm.yy') last_ddl
        FROM dba_objects o, gv$locked_object l, v$session v
        WHERE o.object_id = l.object_id
         and l.SESSION_ID=v.sid
        order by 2,3;
        
        ```
        
    - Verificar o tamanho das tabelas
        
        ```sql
        select 
          table_name, 
          decode(
            partitioned, '/', 'NO', partitioned
          ) partitioned, 
          num_rows, 
          data_mb, 
          indx_mb, 
          lob_mb, 
          total_mb 
        from 
          (
            select 
              data.table_name, 
              partitioning_type || decode (
                subpartitioning_type, 'none', null, 
                '/' || subpartitioning_type
              ) partitioned, 
              num_rows, 
              nvl(data_mb, 0) data_mb, 
              nvl(indx_mb, 0) indx_mb, 
              nvl(lob_mb, 0) lob_mb, 
              nvl(data_mb, 0) + nvl(indx_mb, 0) + nvl(lob_mb, 0) total_mb 
            from 
              (
                select 
                  table_name, 
                  nvl(
                    min(num_rows), 
                    0
                  ) num_rows, 
                  round(
                    sum(data_mb), 
                    2
                  ) data_mb 
                from 
                  (
                    select 
                      table_name, 
                      num_rows, 
                      data_mb 
                    from 
                      (
                        select 
                          a.table_name, 
                          a.num_rows, 
                          b.bytes / 1024 / 1024 as data_mb 
                        from 
                          dba_tables a, 
                          dba_segments b 
                        where 
                          a.table_name = b.segment_name 
                          and b.owner = 'REPORTER'
                      )
                  ) 
                group by 
                  table_name
              ) data, 
              (
                select 
                  a.table_name, 
                  round(
                    sum(b.bytes / 1024 / 1024), 
                    2
                  ) as indx_mb 
                from 
                  dba_indexes a, 
                  dba_segments b 
                where 
                  a.index_name = b.segment_name 
                  and b.owner = 'REPORTER' 
                group by 
                  a.table_name
              ) indx, 
              (
                select 
                  a.table_name, 
                  round(
                    sum(b.bytes / 1024 / 1024), 
                    2
                  ) as lob_mb 
                from 
                  dba_lobs a, 
                  dba_segments b 
                where 
                  a.segment_name = b.segment_name 
                  and b.owner = 'REPORTER' 
                group by 
                  a.table_name
              ) lob, 
              user_part_tables part 
            where 
              data.table_name = indx.table_name(+) 
              and data.table_name = lob.table_name(+) 
              and data.table_name = part.table_name(+)
          ) 
        order by 
          7 desc;
        
        ```
        
    - Verificar o tamanho das tabelas 2
        
        ```sql
        --Essa query mostra o tamanho total, o tamanho usado e o espaço desperdiçado.
        --Ex.: Reservou 1TB e esta usando 100gb.
        
        select owner
            , table_name,round((blocks*8)/1024,2) "size (mb)" 
            , round(((num_rows*avg_row_len)/1024/1024),2) "actual_data (mb)"
            , (round((blocks*8),2)/1024 - round(((num_rows*avg_row_len)/1024/1024),2)) "wasted_space (mb)" 
            from dba_tables 
            where (round((blocks*8),2)/1024 > round(((num_rows*avg_row_len)/1024/1024),2)) and owner = 'SENSOR'
            order by 5 desc;
        ```
        
    - Verificar o tamanho das tabelas 3
        
        ```sql
        SELECT /*+ rule */
               d.OWNER,
               d.table_name,
               DECODE(d.partitioned, '/', 'NO', d.partitioned) AS partitioned,
               d.num_rows,
               d.data_mb,
               COALESCE(i.indx_mb, 0) AS indx_mb,
               COALESCE(l.lob_mb, 0) AS lob_mb,
               d.data_mb + COALESCE(i.indx_mb, 0) + COALESCE(l.lob_mb, 0) AS total_mb,
               s.tablespace_name
          FROM (SELECT a.table_name,
                       a.OWNER,
                       'NO' AS partitioned,  -- Simplificado, pois não temos acesso a PARTITIONING_TYPE
                       NVL(MIN(a.num_rows), 0) AS num_rows,
                       ROUND(SUM(b.bytes / 1024 / 1024), 2) AS data_mb
                  FROM DBA_TABLES a
                  JOIN DBA_SEGMENTS b
                    ON a.table_name = b.segment_name
                   AND a.owner = b.owner
                 GROUP BY a.table_name, a.OWNER) d
          LEFT JOIN (SELECT a.table_name,
                             ROUND(SUM(b.bytes / 1024 / 1024), 2) AS indx_mb
                        FROM DBA_INDEXES a
                        JOIN DBA_SEGMENTS b
                          ON a.index_name = b.segment_name
                         AND a.owner = b.owner
                     GROUP BY a.table_name) i
            ON d.table_name = i.table_name
          LEFT JOIN (SELECT a.table_name,
                             ROUND(SUM(b.bytes / 1024 / 1024), 2) AS lob_mb
                        FROM DBA_LOBS a
                        JOIN DBA_SEGMENTS b
                          ON a.segment_name = b.segment_name
                         AND a.owner = b.owner
                     GROUP BY a.table_name) l
            ON d.table_name = l.table_name
          LEFT JOIN (SELECT a.segment_name AS table_name,
                             MAX(a.tablespace_name) AS tablespace_name
                        FROM DBA_SEGMENTS a
                       GROUP BY a.segment_name) s
            ON d.table_name = s.table_name
         ORDER BY total_mb DESC, d.table_name;
        ```
        
    - Verificar tabelas associadas a uma tablespace
        
        ```sql
        SELECT s.segment_name AS table_name,
               s.bytes/1024/1024/1024 AS size_gb
        FROM dba_segments s
        JOIN dba_tables t ON s.segment_name = t.table_name
        WHERE s.tablespace_name = 'ICT_AUX_DAT'
        ORDER BY s.bytes DESC;
        ```
        
    - **Verificar se há fragmentação na tabela**
        
        ```sql
        set pages 50000 lines 32767
        
        select owner,table_name,round((blocks*8),2)||’kb’ “Fragmented size”, round((num_rows*avg_row_len/1024),2)||’kb’ 
        “Actual size”, round((blocks*8),2)-round((num_rows*avg_row_len/1024),2)||’kb’,
        ((round((blocks*8),2)-round((num_rows*avg_row_len/1024),2))/round((blocks*8),2))*100 -10 “reclaimable space % ” 
        from dba_tables where table_name =’&table_Name’ AND OWNER LIKE ‘&schema_name’
         /
        ```
        
    - Análise de dados em uma tabela específica
        
        ```sql
        --Verificar os campos da tabela que possuem datas
        desc MLOG_PROD.TBPROJECT_ATTACHMENT;
        
        --Verifica as datas de registros em uma tabela específica
        select update_dt from MLOG_PROD.TBPROJECT_ATTACHMENT 
        group by update_dt 
        order by update_dt asc;
        
        -- Verifica a quantidade de registros por ano na tabela
        SELECT EXTRACT(YEAR FROM update_dt) AS ano,
               COUNT(*) AS quantidade_registros
        FROM MLOG_PROD.TBPROJECT_ATTACHMENT
        GROUP BY EXTRACT(YEAR FROM update_dt)
        ORDER BY ano ASC;
        ```
        
- DISKGROUPS
    - Procedimento para alocar disco oracleasm
        
        ```sql
        Logar como root
        
        oracleasm scandisks
        oracleasm listdisks
        oracleasm querydisk /dev/sd*1
        lsblk -o NAME,LABEL,SIZE,MOUNTPOINT,FSTYPE
        lsblk -o NAME,LABEL,SIZE,MOUNTPOINT,FSTYPE | grep -E 'DISCO|DATA'
        oracleasm createdisk DSK_OCR_01 /dev/sdc1
        
        su - grid
        
        Interface gráfica
        
        export DISPLAY=10.221.113.212:0.0 - Alteração do diskgroup na interface
        asmca
        
        ou
        
        Linha de comando
        
        sqlplus / as sysasm
        
        ALTER DISKGROUP DATA ADD DISK '<caminho e nome do disco>' NAME DATA07 REBALANCE POWER 8;
        
        Listar todos os discos do diskgroup
        
        set pagesize 1000 linesize 180
        col "TOTAL(GB)" for 99999.999
        col "USAGE(GB)" for 99999.99
        col "FREE(GB)" for 99999.999
        	 SELECT group_number,
           disk_number,
           mount_status,
           header_status,
           mode_status,
           state,
           path,
           total_mb,
           free_mb
        	   FROM v$asm_disk
        		 WHERE group_number = (SELECT group_number FROM v$asm_diskgroup WHERE name = 'DATA');
        
        ```
        
    - Procedimento para alocar disco AFD
        
        ```sql
        Logar com root
         
         export ORACLE_HOME=/u02/app/19.27.0.0/grid -- Verificar caminho
         export ORACLE_BASE=/tmp
         $ORACLE_HOME/bin/asmcmd afd_refresh
         $ORACLE_HOME/bin/asmcmd afd_scan
         $ORACLE_HOME/bin/asmcmd afd_lslbl
         
         Montar disco
         $ORACLE_HOME/bin/asmcmd afd_label DISCO16 /dev/sdad1
         
         su - grid
         
         sqlplus / as sysasm
         
        Adicionar disco ao DATA
        
        Linha de comando 
        
        ALTER DISKGROUP DATA ADD DISK 'AFD:DISCO15' NAME DISCO15 REBALANCE POWER 8; 
        
        ou
        
        Interface gráfica
        
        export DISPLAY=<IP_JUMP>:0.0 
        asmca
        
        ```
        
    - Verificar diskgroup
        
        ```sql
        Verificar tamanho dos discos diskgroup
        
        set pages 200 lines 999;
        col diskgroup for a30
        col diskname for a15
        col path for a35
        select a.name DiskGroup,b.name DiskName, b.total_mb, (b.total_mb-b.free_mb) Used_MB, b.free_mb,b.path,b.header_status
        from v$asm_disk b, v$asm_diskgroup a
        where a.group_number (+) =b.group_number
        order by b.group_number,b.name;
        
        Verificar montagem
        
        select name, mount_status, header_status, mode_status, label, path, total_mb, free_mb from v$asm_disk
        	where mount_status != 'IGNORED'
        	order by label, path;
        ```
        
    - Query diskgroup
        
        ```sql
        select name Diskgroup
                ,total_mb "TamanhoTT(MB)"
                ,free_mb "Disponivel(MB)"
                ,round(free_mb/total_mb*100,2) percent_livre
                ,state
        from v$asm_diskgroup, v$instance 
        order by name;
        
        *****************************************************
        SELECT name Diskgroup,
               total_mb "TamanhoTT(MB)",
               free_mb "Disponivel(MB)",
               CASE 
                   WHEN total_mb > 0 THEN ROUND(free_mb/total_mb*100, 2)
                   ELSE 0
               END percent_livre,
               state
        FROM v$asm_diskgroup, v$instance
        ORDER BY name;
        
        ****************************************************************
        --Alocação dos discos
        set pages 200 lines 999;
        col diskgroup for a30
        col diskname for a15
        col path for a35
        select a.name DiskGroup,b.name DiskName, b.total_mb, (b.total_mb-b.free_mb) Used_MB, b.free_mb,b.path,b.header_status
        from v$asm_disk b, v$asm_diskgroup a
        where a.group_number (+) =b.group_number
        order by b.group_number,b.name;
        *****************************************************************
        --Verificar o rebalance
        select INST_ID, OPERATION, STATE, POWER, SOFAR, EST_WORK, EST_RATE, EST_MINUTES from GV$ASM_OPERATION;
        ```
        
- TABLESPACE
    - Verificar tablespace
        
        ```sql
        set pagesize 1000 linesize 180
        tti 'Tablespaces'
        col "TOTAL(GB)" for 99999.999
        col "USAGE(GB)" for 99999.999
        col "FREE(GB)" for 99999.999
        col "EXTENSIBLE(GB)" for 9999999.999
        col "FREE PCT %" for 999.99
        col "DTF_NAUTO" for 3
        col "DTF_AUTO" for 3
        SELECT
        d.tablespace_name "NAME",
        d.contents "TYPE",
        ROUND(NVL(a.bytes / 1024 / 1024 / 1024, 0), 3) "TOTAL(GB)",
        ROUND(NVL(f.bytes, 0) / 1024 / 1024 / 1024, 3) "FREE(GB)",
        ROUND(NVL(a.bytes - NVL(f.bytes, 0), 0) / 1024 / 1024 / 1024, 3) "USAGE(GB)",
        ROUND(NVL((NVL(f.bytes, 0) / a.bytes) * 100, 0), 3) "FREE PCT %",
        ROUND(NVL(a.ARTACAK, 0) / 1024 / 1024 / 1024, 3) "EXTENSIBLE(GB)",
        a.NOTO AS DTF_NAUTO,
        a.OTO AS DTF_AUTO
        FROM
        sys.dba_tablespaces d
        LEFT JOIN
        (
        SELECT
        tablespace_name,
        SUM(bytes) bytes,
        SUM(DECODE(autoextensible, 'YES', MAXbytes - bytes, 0)) ARTACAK,
        COUNT(DECODE(autoextensible, 'NO', 0)) NOTO,
        COUNT(DECODE(autoextensible, 'YES', 0)) OTO
        FROM
        dba_data_files
        GROUP BY
        tablespace_name
        ) a ON d.tablespace_name = a.tablespace_name
        LEFT JOIN
        (
        SELECT
        tablespace_name,
        SUM(bytes) bytes
        FROM
        dba_free_space
        GROUP BY
        tablespace_name
        ) f ON d.tablespace_name = f.tablespace_name
        WHERE
        NOT (d.extent_management LIKE 'LOCAL' AND d.contents LIKE 'TEMPORARY')
        UNION ALL
        SELECT
        d.tablespace_name "NAME",
        d.contents "TYPE",
        ROUND(NVL(a.bytes / 1024 / 1024 / 1024, 0), 3) "TOTAL(GB)",
        ROUND(NVL(a.bytes - NVL(t.bytes, 0), 0) / 1024 / 1024 / 1024, 3) "FREE(GB)",
        ROUND(NVL(t.bytes, 0) / 1024 / 1024 / 1024, 3) "USAGE(GB)",
        ROUND(NVL((NVL(a.bytes - NVL(t.bytes, 0), 0) / a.bytes) * 100, 0), 3) "FREE PCT %",
        ROUND(NVL(a.ARTACAK, 0) / 1024 / 1024 / 1024, 3) "EXTENSIBLE(GB)",
        a.NOTO AS DTF_NAUTO,
        a.OTO AS DTF_AUTO
        FROM
        sys.dba_tablespaces d
        LEFT JOIN
        (
        SELECT
        tablespace_name,
        SUM(bytes) bytes,
        SUM(DECODE(autoextensible, 'YES', MAXbytes - bytes, 0)) ARTACAK,
        COUNT(DECODE(autoextensible, 'NO', 0)) NOTO,
        COUNT(DECODE(autoextensible, 'YES', 0)) OTO
        FROM
        dba_temp_files
        GROUP BY
        tablespace_name
        ) a ON d.tablespace_name = a.tablespace_name
        LEFT JOIN
        (
        SELECT
        tablespace_name,
        SUM(bytes_used) bytes
        FROM
        v$temp_extent_pool
        GROUP BY
        tablespace_name
        ) t ON d.tablespace_name = t.tablespace_name
        WHERE
        d.extent_management LIKE 'LOCAL'
        AND d.contents LIKE 'TEMPORARY%'
        ORDER BY
        "FREE PCT %" ASC;
        ```
        
    - Verificar tablespace de uma tabela específica
        
        ```sql
        SELECT TABLE_NAME, TABLESPACE_NAME
        FROM dba_tables
        WHERE table_name = 'TB_FT_ECQ_DAILY_FULL'
          AND owner = 'NTW_OP';
        ```
        
    - Verificar tablespace padrão de um usuário
        
        ```sql
        SET LINES 200
        SET PAGES 100
        COL USERNAME            FORMAT A15
        COL DEFAULT_TABLESPACE  FORMAT A25
        COL TEMPORARY_TABLESPACE FORMAT A25
        SELECT
            username,
            default_tablespace,
            temporary_tablespace
        FROM
            dba_users
        WHERE
            username IN ('NSR_PRD', 'NTW_RES')
        ORDER BY
            username;
        
        ```
        
    - Verificação completa das tabelas consumidoras na tablespace
        
        ```sql
        select table_name,   
           decode(partitioned,'/','NO',partitioned) partitioned,  
           num_rows,  
           data_mb,
           indx_mb,  
           lob_mb,   
           total_mb  
            from (select data.table_name,   
                    partitioning_type 
                     || decode (subpartitioning_type,   
                                'none', null,   
                                '/' || subpartitioning_type)   
                            partitioned,   
                     num_rows,   
                     nvl(data_mb,0) data_mb,  
                     nvl(indx_mb,0) indx_mb,  
                     nvl(lob_mb,0) lob_mb,   
                     nvl(data_mb,0) + nvl(indx_mb,0) + nvl(lob_mb,0) total_mb  
                     from (  select table_name,  
                           nvl(min(num_rows),0) num_rows,  
                           round(sum(data_mb),2) data_mb   
                              from (select table_name, num_rows, data_mb  
                                  from (select a.table_name,  
                                        a.num_rows,  
                                        b.bytes/1024/1024 as data_mb  
                                          from dba_tables a, dba_segments b  
                                          where a.table_name = b.segment_name and b.owner='MLOG_PROD')) 
                         group by table_name) data,   
                         (  select a.table_name,   
                                round(sum(b.bytes/1024/1024),2) as indx_mb  
                             from dba_indexes a, dba_segments b  
                               where a.index_name = b.segment_name and b.owner='MLOG_PROD'
                            group by a.table_name) indx, 
                         (  select a.table_name,  
                               round(sum(b.bytes/1024/1024),2) as lob_mb 
                            from dba_lobs a, dba_segments b  
                           where a.segment_name = b.segment_name and b.owner='MLOG_PROD'
                            group by a.table_name) lob,  
                           user_part_tables part  
                     where     data.table_name = indx.table_name(+)  
                           and data.table_name = lob.table_name(+)   
                           and data.table_name = part.table_name(+)) 
          order by 7 desc;
        ```
        
    - Verificação informações detalhadas sobre os datafiles
        
        ```sql
        -- consultar informacoes detalhadas sobre os datafiles: status, autoincremento, tamanho do autoincremento, espaco alocado etc.
        SET LINESIZE 200
        SET PAGESIZE 100
        COLUMN TABLESPACE FORMAT A20
        COLUMN FILE_NAME FORMAT A50
        COLUMN STATUS FORMAT A10
        COLUMN AUTO_EXT FORMAT A10
        COLUMN INCREMENT_BY_GB FORMAT 999999.99
        COLUMN TAM_GB FORMAT 999999999.99
        COLUMN ALOCADO_GB FORMAT 999999999.99
        COLUMN USER_BYTES_GB FORMAT 999999999.99
        COLUMN PCT_OCUP FORMAT 999.99
        COLUMN MAX_GB FORMAT 999999.99
        
        SELECT  
            F.TABLESPACE_NAME                    AS "TABLESPACE",
            F.FILE_NAME                           AS "FILE_NAME",
            F.STATUS                              AS "STATUS",
            F.AUTOEXTENSIBLE                      AS "AUTO_EXT",
            DECODE(F.AUTOEXTENSIBLE, 'NO', 0, 
                ROUND((F.INCREMENT_BY * (F.BYTES / F.BLOCKS)) / 1073741824, 2)
            )                                     AS "INCREMENT_BY_GB",
            ROUND(F.BYTES / 1073741824, 2)        AS "TAM_GB",
            ROUND((F.BYTES - NVL(S.LIVRES, 0)) / 1073741824, 2) 
                                                  AS "ALOCADO_GB",
            ROUND(F.USER_BYTES / 1073741824, 2)   AS "USER_BYTES_GB",
            ROUND(
                DECODE(
                    (F.BYTES - NVL(S.LIVRES, 0)) / 1073741824, 
                    0, 0, 
                    (((F.BYTES - NVL(S.LIVRES, 0)) / 1073741824) / (F.BYTES / 1073741824)) * 100
                ), 
                2
            )                                     AS "PCT_OCUP",
            ROUND(F.MAXBYTES / 1073741824, 2)     AS "MAX_GB"
        FROM DBA_DATA_FILES F
        LEFT JOIN (
            SELECT 
                FILE_ID, 
                SUM(BYTES) AS LIVRES 
            FROM DBA_FREE_SPACE 
            GROUP BY FILE_ID
        ) S
        ON F.FILE_ID = S.FILE_ID
        ORDER BY F.TABLESPACE_NAME, F.FILE_NAME;
        
        ```
        
    - Bloco PL/SQL para adicionar vários datafiles
        
        ```sql
        DECLARE
          v_tbs_name     VARCHAR2(30) := 'INFOACESSO_TBS';
          v_total_mb     NUMBER;
          v_used_mb      NUMBER;
          v_pct_used     NUMBER;
          v_prev_total   NUMBER := 0;
          v_max_adds     NUMBER := 5;  -- limite de tentativas para evitar loop infinito
          v_add_count    NUMBER := 0;
        BEGIN
          LOOP
            -- Coleta o uso da tablespace usando DBA_TABLESPACE_USAGE_METRICS
            SELECT 
              ROUND((DTUM.TABLESPACE_SIZE*pa.value)/1024/1024,2) AS tamanho_max_mb,
              ROUND((DTUM.USED_SPACE*pa.value)/1024/1024,2) AS espaco_usado_mb,
              ROUND(DTUM.USED_PERCENT) AS pct_usado
            INTO 
              v_total_mb, v_used_mb, v_pct_used
            FROM 
              DBA_TABLESPACE_USAGE_METRICS DTUM,
              v$parameter pa
            WHERE 
              pa.name ='db_block_size'
              AND DTUM.TABLESPACE_NAME = v_tbs_name;
        
            DBMS_OUTPUT.PUT_LINE('Total: ' || v_total_mb || ' MB | Usado: ' || v_used_mb || ' MB | Uso: ' || v_pct_used || '%');
        
            -- Condição de parada
            IF v_pct_used < 88 THEN
              DBMS_OUTPUT.PUT_LINE('Uso abaixo de 88%. Encerrando o processo.');
              EXIT;
            END IF;
        
            -- Impede adições infinitas
            IF v_add_count >= v_max_adds THEN
              DBMS_OUTPUT.PUT_LINE('Limite máximo de adições alcançado. Abortando para evitar esgotar o diskgroup.');
              EXIT;
            END IF;
        
            -- Verifica se adicionar espaço está realmente surtindo efeito
            IF v_total_mb = v_prev_total THEN
              DBMS_OUTPUT.PUT_LINE('O tamanho total não aumentou desde a última execução. Abortando.');
              EXIT;
            END IF;
        
            -- Adiciona novo datafile
            EXECUTE IMMEDIATE 'ALTER TABLESPACE ' || v_tbs_name || ' ADD DATAFILE SIZE 32767M';
            DBMS_OUTPUT.PUT_LINE('Datafile de 32GB adicionado.');
        
            v_prev_total := v_total_mb;
            v_add_count := v_add_count + 1;
        
            -- Aguarda antes da próxima verificação
            DBMS_LOCK.SLEEP(5);
          END LOOP;
        EXCEPTION
          WHEN OTHERS THEN
            DBMS_OUTPUT.PUT_LINE('Erro: ' || SQLERRM);
        END;
        /
        ```
        
    - Bloco PL/SQL para adicionar vários datafiles II
        
        ```sql
        BEGIN
           FOR i IN 1..25 LOOP
              EXECUTE IMMEDIATE
                 'ALTER TABLESPACE MLOGTBS ADD DATAFILE SIZE 32767M AUTOEXTEND OFF';
           END LOOP;
        END;
        /
        ```
        
    - Alterar TEMP
        
        ```sql
        alter tablespace temp add TEMPFILE size 32767M autoextend ON next 128M maxsize 32767M;
        
        Alterar um tempfile existente:
        alter database tempfile '+DATA/CDBTIM01/21AED9DBB10A5AE3E053573FDD0A4C19/TEMPFILE/temp.2039.1158080485' autoextend ON;
        
        alter tablespace UNDOTBS2 add datafile size 32767m autoextend on next 100m maxsize UNLIMITED;
        ```
        
    - Verificar tablespace de um schema
        
        ```sql
        SELECT DISTINCT t.tablespace_name, u.username AS owner
        FROM dba_tablespaces t
        INNER JOIN dba_segments s ON t.tablespace_name = s.tablespace_name
        INNER JOIN dba_users u ON s.owner = u.username;
        ```
        
    - Verificar se é ASM ou Filesystem
        
        ```sql
        SELECT tablespace_name,
               SUM(bytes)/1024/1024/1024 AS "Size_GB",
               CASE WHEN tablespace_name LIKE 'SYS%' THEN 'ASM'
                    ELSE 'File System' END AS "Usage"
          FROM dba_data_files
        GROUP BY tablespace_name;
        ```
        
    - Resize datafile
        
        ```sql
        ALTER DATABASE DATAFILE '/caminho/para/datafile.dbf' RESIZE novo_tamanho;
        ```
        
    - Comando para redimensionar datafiles menores que 32G
        
        ```sql
        SELECT 
            'ALTER DATABASE DATAFILE ''' || file_name ||
            ''' RESIZE 32767M;' AS comando
        FROM dba_data_files
        WHERE tablespace_name = 'INF_DW_TBS'
        AND bytes/1024/1024 < 32767;
        ```
        
    - Limpeza SYSAUX
        
        ```sql
        create tablespace AUDTBS datafile size 32767M autoextend ON;
        
        --Isso move a tabela AUD$
        BEGIN
        DBMS_AUDIT_MGMT.set_audit_trail_location(
        audit_trail_type => DBMS_AUDIT_MGMT.AUDIT_TRAIL_AUD_STD,
        audit_trail_location_value => 'AUDTBS');
        END;
        /
        
        --Esse move a tabela FGA_LOG$
        BEGIN
        DBMS_AUDIT_MGMT.set_audit_trail_location(
        audit_trail_type => DBMS_AUDIT_MGMT.AUDIT_TRAIL_FGA_STD,--this moves table FGA_LOG$
        audit_trail_location_value => 'AUDTBS');
        END;
        /
        
        *****************************************************************
        
        Procedimento para limpar SYSAUX
        
        select occupant_name "NOME", occupant_desc "DESCRICAO", round(space_usage_kbytes/1024,2) "USADO(MB)" from v$sysaux_occupants;
        
        select systimestamp - min(savtime) from sys.wri$_optstat_histgrm_history;
        
        exec DBMS_STATS.PURGE_STATS(DBMS_STATS.PURGE_ALL);
        
        BEGIN
        DBMS_AUDIT_MGMT.CLEAN_AUDIT_TRAIL(
        AUDIT_TRAIL_TYPE => DBMS_AUDIT_MGMT.AUDIT_TRAIL_UNIFIED,
        USE_LAST_ARCH_TIMESTAMP => FALSE,
        CONTAINER => dbms_audit_mgmt.container_current);
        END;
        /
        ***********************************************
        BEGIN
          DBMS_AUDIT_MGMT.CLEAN_AUDIT_TRAIL(
            AUDIT_TRAIL_TYPE       => DBMS_AUDIT_MGMT.AUDIT_TRAIL_UNIFIED,
            USE_LAST_ARCH_TIMESTAMP => FALSE,
            CLEANUP_TILL           => SYSTIMESTAMP - INTERVAL '15' DAY,
            CONTAINER              => DBMS_AUDIT_MGMT.CONTAINER_CURRENT
          );
        END;
        /
        
        **********************************************************************
        select 'alter database datafile '''  || file_name ||  ''' resize ' ||
               ceil( (nvl(hwm,1)*8192*1.2)/1024/1024 )  || 'm;' cmd
         from dba_data_files a,
             ( select file_id, max(block_id+blocks-1) hwm
                 from dba_extents
                group by file_id ) b
         where a.file_id = b.file_id(+)
          and ceil( (nvl(hwm,1)*8192)/1024/1024 ) < ceil( blocks*8192/1024/1024)
          and ceil( (nvl(hwm,1)*8192)/1024/1024 ) > 1
          and a.TABLESPACE_NAME='AUDTBS';
        ```
        
    - Limpeza SYSAUX  (NOVO)
        
        ```sql
        --Verificar retenção
        SELECT retention, snap_interval FROM dba_hist_wr_control;
        
        --Verificar quantidade de snapshots
        SELECT COUNT(*) snapshots
        FROM dba_hist_snapshot;
        
        --Crescimento diário do AWR
        SELECT
          TRUNC(begin_interval_time) dia,
          COUNT(*) snapshots
        FROM dba_hist_snapshot
        GROUP BY TRUNC(begin_interval_time)
        ORDER BY dia;
        
        --Ver intervalo de snapshots existentes
        SELECT
          MIN(snap_id) min_snap,
          MAX(snap_id) max_snap
        FROM dba_hist_snapshot;
        
        --Descobrir o SNAP_ID limite (30 dias)
        SELECT snap_id,
               begin_interval_time
        FROM dba_hist_snapshot
        WHERE begin_interval_time <= SYSDATE - 30
        ORDER BY snap_id;
        
        --Purge 30 dias mais antigos
        BEGIN
          DBMS_WORKLOAD_REPOSITORY.drop_snapshot_range(
            low_snap_id  => 105311,
            high_snap_id => 106760,
            dbid         => 1680616902
          );
        END;
        /
        ```
        
    - Limpeza AUDTBS
        
        ```sql
        --Verificar o que está utilizando a AUDTBS
        SELECT
            obj$creator AS owner,
            obj$name AS objeto,
            COUNT(*) AS qtd
        FROM sys.aud$
        WHERE action# IN (2,3)  -- 2 = INSERT, 3 = UPDATE
        GROUP BY obj$creator, obj$name
        ORDER BY qtd DESC;
        
        --Uso de espaço em disco por owner
        SELECT OWNER, SUM(BYTES)/1024/1024 AS MB_USED
        FROM DBA_SEGMENTS
        WHERE TABLESPACE_NAME = 'AUDTBS'
        GROUP BY OWNER
        ORDER BY MB_USED DESC;
        
        --Detalhamento do Uso de Espaço em Disco no Tablespace AUDTBS
        SET LINESIZE 200
        SET PAGESIZE 100
        COLUMN OWNER FORMAT A20
        COLUMN SEGMENT_NAME FORMAT A30
        COLUMN SEGMENT_TYPE FORMAT A15
        COLUMN MB_USED FORMAT 9999999
        SELECT 
            OWNER, 
            SEGMENT_NAME, 
            SEGMENT_TYPE, 
            ROUND(BYTES / 1024 / 1024, 2) AS MB_USED
        FROM 
            DBA_SEGMENTS
        WHERE 
            TABLESPACE_NAME = 'AUDTBS';
        
        --Inicialização da Limpeza da Trilha de Auditoria Padrão (agendamento)
        BEGIN
          DBMS_AUDIT_MGMT.INIT_CLEANUP(
            AUDIT_TRAIL_TYPE => DBMS_AUDIT_MGMT.AUDIT_TRAIL_AUD_STD, -- Define o tipo da trilha de auditoria
            DEFAULT_CLEANUP_INTERVAL => 30 -- Defina o intervalo de limpeza conforme sua necessidade
          );
        END;
        /
        
        --Executar a limpeza manualmente
        BEGIN
          DBMS_AUDIT_MGMT.CLEAN_AUDIT_TRAIL(
            AUDIT_TRAIL_TYPE => DBMS_AUDIT_MGMT.AUDIT_TRAIL_AUD_STD,
            USE_LAST_ARCH_TIMESTAMP => TRUE
          );
        END;
        /
        
        truncate table sys.aud$;
        
        SELECT COUNT(*) FROM SYS.AUD$;
        
        SELECT *FROM SYS.AUD$;
        ```
        
    - Script para limpar snapshots antigos AWR SYSAUX
        
        ```sql
        
        -- 1. Mostra intervalo de SNAP_IDs que serão excluídos
        PROMPT ===========================================================
        PROMPT Passo 1: Verificando SNAP_IDs com mais de 45 dias
        PROMPT ===========================================================
        
        SELECT MIN(snap_id) AS low_snap_id,
               MAX(snap_id) AS high_snap_id,
               COUNT(*)     AS total_snapshots
          FROM dba_hist_snapshot
         WHERE begin_interval_time < SYSDATE - 45;
        
        -- 2. Mostra detalhes dos snapshots para conferência (limite de 10 para não poluir)
        PROMPT ===========================================================
        PROMPT Passo 2: Visualizando amostra de snapshots antigos (10 linhas)
        PROMPT ===========================================================
        
        SELECT snap_id, begin_interval_time
          FROM dba_hist_snapshot
         WHERE begin_interval_time < SYSDATE - 45
         ORDER BY begin_interval_time
         FETCH FIRST 10 ROWS ONLY;
        
        -- 3. Comando de exclusão dos snapshots antigos
        PROMPT ===========================================================
        PROMPT Passo 3: Excluindo snapshots no intervalo [41603 - 42735]
        PROMPT ===========================================================
        
        BEGIN
          DBMS_WORKLOAD_REPOSITORY.DROP_SNAPSHOT_RANGE (
            low_snap_id  => 41603,
            high_snap_id => 42735
          );
        END;
        /
        
        -- 4. Verificação após exclusão (deve retornar zero)
        PROMPT ===========================================================
        PROMPT Passo 4: Verificando se os snapshots antigos foram excluídos
        PROMPT ===========================================================
        
        SELECT COUNT(*) AS remaining_old_snapshots
          FROM dba_hist_snapshot
         WHERE snap_id BETWEEN 41603 AND 42735;
        
        -- 5. Espaço usado na SYSAUX (pode não cair imediatamente, mas será reduzido)
        PROMPT ===========================================================
        PROMPT Passo 5: Verificando uso atual da SYSAUX
        PROMPT ===========================================================
        
        SELECT occupant_name "NOME",
               occupant_desc "DESCRICAO",
               ROUND(space_usage_kbytes/1024,2) "USADO(MB)"
          FROM v$sysaux_occupants
        ORDER BY 3 DESC;
        
        -- =========================================================================
        PROMPT Concluído! Snapshots AWR antigos foram excluídos.
        -- =========================================================================
        
        ```
        
    - Purge de objetos AUDSYS Coliseu
        
        ```sql
        BEGIN
          DBMS_AUDIT_MGMT.SET_LAST_ARCHIVE_TIMESTAMP(
            AUDIT_TRAIL_TYPE => DBMS_AUDIT_MGMT.AUDIT_TRAIL_UNIFIED,
            LAST_ARCHIVE_TIME => SYSTIMESTAMP - INTERVAL '30' DAY
          );
          DBMS_AUDIT_MGMT.CLEAN_AUDIT_TRAIL(
            AUDIT_TRAIL_TYPE => DBMS_AUDIT_MGMT.AUDIT_TRAIL_UNIFIED,
            USE_LAST_ARCH_TIMESTAMP => TRUE
          );
        END;
        /
        
        ```
        
    - Criar tablespace
        
        ```sql
        create tablespace BASECLONE_IDX_TBS datafile size 32767M autoextend off 
        extent management LOCAL 
        autoallocate segment space management AUTO LOGGING;
        
        CREATE BIGFILE TABLESPACE PMOINTERFACE_TBS DATAFILE SIZE 192G AUTOEXTEND OFF;
        ```
        
    - Adicionar datafile
        
        ```sql
        ALTER TABLESPACE NTW_OP_TBS ADD DATAFILE SIZE 32767M AUTOEXTEND OFF;
        ```
        
    - Verificar SM/AWR (SYSAUX)
        
        ```sql
        * O script abaixo irá listar os objetos e seus respectivos tamanhos dentro da tablespace SYSAUX
        
        set lines 200
        col occupant_name  for a20
        col schema_name    for a20
        col move_procedure for a35
        SELECT occupant_name, round(space_usage_kbytes/1024) "Space (M)",
        schema_name, move_procedure
        FROM v$sysaux_occupants
        ORDER BY 1;
        
        * Com essa consulta é possível ver o menor e o maior intervalo de informações do AWR armazenadas na base de dados.
        
        col begin_interval_time for a30
        col end_interval_time   for a30
        SELECT 
          snap_id, 
          instance_number, 
          begin_interval_time, 
          end_interval_time 
        FROM 
          SYS.WRM$_SNAPSHOT 
        WHERE 
          snap_id = (
            SELECT 
              MIN (snap_id) 
            FROM 
              SYS.WRM$_SNAPSHOT
          ) 
        UNION 
        SELECT 
          snap_id, 
          INSTANCE_NUMBER, 
          begin_interval_time, 
          end_interval_time 
        FROM 
          SYS.WRM$_SNAPSHOT 
        WHERE 
          snap_id = (
            SELECT 
              MAX (snap_id) 
            FROM 
              SYS.WRM$_SNAPSHOT
          ) 
        ORDER BY 
          2;
          
        * Para removermos as informações desejadas podemos verificar ainda os detalhes dos intervalos desejados executando a consulta
          
        SELECT snap_id, instance_number, begin_interval_time
        FROM SYS.WRM$_SNAPSHOT
        WHERE instance_number = 1
        AND begin_interval_time like '%JUN%';
        
         * Para remover as informações mais antigas, executar a package DBMS_WORKLOAD_REPOSITORY
           
         execute dbms_workload_repository.drop_snapshot_range( low_snap_id => 1, high_snap_id=>9840);
        
        ```
        
    - Verificar INDEX consumidores em tablespace específica (2 consultas)
        
        ```sql
        --Verificar consumo total
        SET LINESIZE 300;
        SET PAGESIZE 100;
        COLUMN OWNER FORMAT A20;
        COLUMN SEGMENT_NAME FORMAT A40;
        COLUMN SEGMENT_TYPE FORMAT A20;
        COLUMN SIZE_MB FORMAT 9999999.99;
        SELECT owner, 
               segment_name, 
               segment_type, 
               bytes/1024/1024 AS size_mb
        FROM dba_segments
        WHERE tablespace_name = 'SYSTEM'
        ORDER BY size_mb DESC
        FETCH FIRST 20 ROWS ONLY;
        
        --Visualizar segmentos específicos
        SELECT 
            segment_name, 
            segment_type,
            tablespace_name, 
            SUM(bytes) / 1024 / 1024 AS used_mb
        FROM 
            dba_segments
        WHERE 
            tablespace_name = 'SYSTEM'
            AND segment_name LIKE 'ID%'  -- Filtro para segmentos que começam com "IDX"
        GROUP BY 
            segment_name, segment_type, tablespace_name
        ORDER BY 
            used_mb DESC;
        
        ```
        
    - Shrink Tablespace
        
        ```sql
        select 'alter database datafile '''  || file_name ||  ''' resize ' ||
               ceil( (nvl(hwm,1)*8192*1.2)/1024/1024 )  || 'm;' cmd
         from dba_data_files a,
             ( select file_id, max(block_id+blocks-1) hwm
                 from dba_extents
                group by file_id ) b
         where a.file_id = b.file_id(+)
          and ceil( (nvl(hwm,1)*8192)/1024/1024 ) < ceil( blocks*8192/1024/1024)
          and ceil( (nvl(hwm,1)*8192)/1024/1024 ) > 1
          and a.TABLESPACE_NAME='AUDITORIAAPPUSER_TBS';
        ```
        
    - Verificar todos os objetos na tablespace (DBA_SEGMENTS)
        
        ```sql
        set lines 200 pages 200 
        col owner for a30
        col segment_name for a60
        col segment_type for a30
        col bytes for 9999999999
        SELECT owner, segment_name, segment_type, bytes/(1024*1024) as tamanho
        FROM dba_segments
        WHERE tablespace_name = 'MLOGTBS'
        ORDER BY owner, segment_type, segment_name;
        
        ```
        
    - Consulta para exibir os 20 maiores segmentos
        
        ```sql
        SET LINESIZE 200
        COLUMN owner FORMAT A30
        COLUMN segment_name FORMAT A30
        COLUMN tablespace_name FORMAT A30
        COLUMN size_mb FORMAT 99999999.00
        
        SELECT *
        FROM   (SELECT owner,
                       segment_name,
                       segment_type,
                       tablespace_name,
                       ROUND(bytes/1024/1024,2) size_mb
                FROM   dba_segments
                ORDER BY 5 DESC)
        WHERE  ROWNUM <= 20;
        ```
        
    - Verificar possibilidade de resize da tablespace
        
        ```sql
        select 'alter database datafile '''  || file_name ||  ''' resize ' ||
               ceil( (nvl(hwm,1)*8192*1.2)/1024/1024 )  || 'm;' cmd
         from dba_data_files a,
             ( select file_id, max(block_id+blocks-1) hwm
                 from dba_extents
                group by file_id ) b
         where a.file_id = b.file_id(+)
          and ceil( (nvl(hwm,1)*8192)/1024/1024 ) < ceil( blocks*8192/1024/1024)
          and ceil( (nvl(hwm,1)*8192)/1024/1024 ) > 1
          and a.TABLESPACE_NAME='USR';
        ```
        
    - Dropar uma tablespace junto com seus datafiles e constraints
        
        ```sql
        DROP TABLESPACE <TS_NAME> INCLUDING CONTENTS AND DATAFILES CASCADE CONSTRAINTS;
        ```
        
    - Verificar se a tablespace é BigFile
        
        ```sql
        SELECT tablespace_name, bigfile
        FROM dba_tablespaces
        WHERE tablespace_name = 'PMOINTERFACE_TBS';
        ```
        
    - Alterar tablespace
        
        ```sql
        Resize
        ALTER TABLESPACE SYSAUX RESIZE 2764800M;
        
        alter tablespace XXXX add datafile size 32767M;
        
        select file_id,(USER_BYTES/1024/1024/1024) ,(MAXBYTES/1024/1024/1024) from dba_data_files where tablespace_name='SYSAUX';
        ```
        
    - Conceder quota em uma tablespace
        
        ```sql
        -- Conceder quota no tablespace USERS para o usuário DEVOPS
        ALTER USER <usuário> QUOTA UNLIMITED ON USERS;
        
        -- Verificar os privilégios de quota
        SELECT USERNAME, TABLESPACE_NAME, BYTES
        FROM DBA_TS_QUOTAS
        WHERE USERNAME = 'USUÁRIO' AND TABLESPACE_NAME = 'USERS';
        ```
        
    - Análise tablespace
        
        ```sql
        Tamanho físico e livre das tablespaces do banco
        
        select tablespace_name, sum(bytes)/1024/1024/1024 as "TAMANHO(GB)"
        from dba_data_files
        group by tablespace_name
        order by sum(bytes);
        
        Tamanho livre por tablespaces
        
        Select tablespace_name, sum(bytes)/1024/1024/1024 as "TAMANHO(GB)"
        from dba_free_space
        group by tablespace_name
        order by sum(bytes);
        
        Tablespace que dá para fazer resize
        
        SELECT 'ALTER DATABASE DATAFILE ''' || file_name || ''' RESIZE ' || 
               CEIL((NVL(hwm,1)*8192*1.2)/1024/1024) || 'M;' AS cmd
        FROM dba_data_files a
        LEFT JOIN (
            SELECT file_id, MAX(block_id+blocks-1) AS hwm
            FROM dba_extents
            GROUP BY file_id
        ) b ON a.file_id = b.file_id
        WHERE CEIL((NVL(hwm,1)*8192*1.2)/1024/1024) < CEIL(blocks*8192/1024/1024)
        AND CEIL((NVL(hwm,1)*8192*1.2)/1024/1024) > 100;
        ```
        
    - Verificar datafiles de uma tablespace específica
        
        ```sql
        SELECT 
            file_name,
            tablespace_name,
            autoextensible,
            bytes / (1024 * 1024 * 1024) AS "Tamanho (GB)",
            (bytes - (SELECT NVL(SUM(bytes),0) FROM dba_free_space WHERE tablespace_name = 'CGR')) / (1024 * 1024 * 1024) AS "Espaço Livre (GB)"
        FROM 
            dba_data_files
        WHERE 
            tablespace_name = 'UNDOTBS1';
            
        ```
        
    - Alterar datafile
        
        ```sql
        Alterar autoextend
        
        ALTER DATABASE DATAFILE 'caminho do datafile' AUTOEXTEND OFF NEXT 32767M MAXSIZE;
        
        Resize datafile
        
        ALTER DATABASE DATAFILE '+DATA/PRDARM/DATAFILE/users_02' RESIZE 10G;
        ```
        
    - Verificar datafiles de uma tablespace
        
        ```sql
        SELECT tablespace_name, file_id, bytes, blocks, maxbytes, maxblocks
        FROM dba_data_files
        WHERE tablespace_name = 'GIS';
        ```
        
    - Verificar tamanho tablespace específica
        
        ```sql
        select * from dba_tablespace_usage_metrics
        where tablespace_name in ('NTW_OP_TBS')
        order by 1;
        ```
        
    - Verificar tamanho todas as tablespaces e espaço livre
        
        ```sql
        
        select a.tablespace_name,
        	   a.status,
        	   b.tamanho,
        	   c.livre,
        	   b.tamanho - c.livre usado
        from dba_tablespaces a, (select tablespace_name,
                                 sum(bytes)/1024/1024/1024 tamanho
        						 from dba_data_files
        						 group by tablespace_name) b,
        						(select tablespace_name,
        						 sum(bytes)/1024/1024/1024 livre
        						 from dba_free_space
        						 group by tablespace_name
        						 ) c
        where a.tablespace_name = b.tablespace_name
        and a.tablespace_name = c.tablespace_name;
        ```
        
    - Verificar quais tabelas utilizam a tablespace
        
        ```sql
        SELECT owner, table_name
        FROM dba_tables
        WHERE tablespace_name = 'AUDTBS';
        ```
        
    - Verificar quais usuários utilizam a mesma tablespace
        
        ```sql
        SELECT DISTINCT owner
        FROM dba_segments
        WHERE tablespace_name = 'NOME_DA_TABLESPACE';
        ```
        
    - Verificar tablespace específica
        
        ```sql
        SELECT tablespace_name, SUM(bytes)/1024/1024/1024 AS total_GB
        FROM dba_data_files
        WHERE tablespace_name = 'nome_da_tablespace'
        GROUP BY tablespace_name;
        
        ```
        
    - Verificar utilização dos datafiles de uma tablespace específica
        
        ```sql
        SELECT file_name,
               tablespace_name,
               round(bytes/1024/1024/1024,2) as size_gb,
               round((bytes-free_space)/1024/1024/1024,2) as used_gb,
               round((free_space)/1024/1024/1024,2) as free_gb,
               round((free_space/bytes)*100,2) as percent_free
        FROM
          (SELECT file_name, tablespace_name, bytes,
                  (SELECT sum(bytes) FROM dba_free_space WHERE tablespace_name = df.tablespace_name AND file_id = df.file_id) as free_space
           FROM dba_data_files df
           WHERE tablespace_name = 'UNDOTBS1');
        
        ```
        
    - Verificar configurações tablespace
        
        ```sql
        set pages 999
        set lines 400
        col FILE_NAME format a75
        select d.TABLESPACE_NAME, d.FILE_NAME, d.BYTES/1024/1024 SIZE_MB, d.AUTOEXTENSIBLE, d.MAXBYTES/1024/1024 MAXSIZE_MB, d.INCREMENT_BY*(v.BLOCK_SIZE/1024)/1024 INCREMENT_BY_MB
        from dba_temp_files d, v$tempfile v
        where d.FILE_ID = v.FILE#
        order by d.TABLESPACE_NAME, d.FILE_NAME;
        ```
        
    - Verificar tamanho tablespace
        
        ```sql
        Select tablespace_name, sum(bytes)/1024/1024 as "TAMANHO(MB)"
        from dba_free_space
        group by tablespace_name
        order by sum(bytes);
        ```
        
    - Criação TEMP
        
        ```sql
        TEMP
        
        CREATE TEMPORARY TABLESPACE "TEMP" TEMPFILE 
        SIZE 991952896
        AUTOEXTEND ON NEXT 67108864 MAXSIZE 3072M
        EXTENT MANAGEMENT LOCAL UNIFORM SIZE 1048576;
        ```
        
    - V**erificar uso da TEMP por sessão**
        
        ```sql
        SELECT   S.sid || ',' || S.serial# sid_serial, S.username, S.osuser, P.spid, S.module, 
        P.program, SUM (T.blocks) * TBS.block_size / 1024 / 1024 mb_used, T.tablespace, 
        COUNT(*) statements 
        FROM v$sort_usage T, v$session S, dba_tablespaces TBS, v$process P 
        WHERE T.session_addr = S.saddr AND S.paddr = P.addr 
        AND T.tablespace = TBS.tablespace_name 
        GROUP BY S.sid, S.serial#, S.username, S.osuser, P.spid, S.module, P.program, TBS.block_size, T.tablespace ORDER BY mb_used;
        ```
        
    - V**erificar uso da TEMP por instrução**
        
        ```sql
        SELECT  S.sid || ',' || S.serial# sid_serial, S.username, Q.hash_value, Q.sql_text,
        T.blocks * TBS.block_size / 1024 / 1024 mb_used, T.tablespace
        FROM    v$sort_usage T, v$session S, v$sqlarea Q, dba_tablespaces TBS
        WHERE   T.session_addr = S.saddr
        AND     T.sqladdr = Q.address
        AND     T.tablespace = TBS.tablespace_name ORDER BY mb_used;
        ```
        
    - V**erificar uso da TEMP**
        
        ```sql
        select TABLESPACE_NAME, BYTES_USED, BYTES_FREE from V$TEMP_SPACE_HEADER;
        ```
        
    - **Consulta para verificar a utilização do tablespace**
        
        ```sql
        set pages 50000 lines 32767
        col tablespace_name format a30
        col TABLESPACE_NAME heading “Tablespace|Name”
        col Allocated_size heading “Allocated|Size(GB)” form 99999999.99
        col Current_size heading “Current|Size(GB)” form 99999999.99
        col Used_size heading “Used|Size(GB)” form 99999999.99
        col Available_size heading “Available|Size(GB)” form 99999999.99
        col Pct_used heading “%Used (vs)|(Allocated)” form 99999999.99
        select a.tablespace_name
                ,a.alloc_size/1024/1024/1024 Allocated_size
                ,a.cur_size/1024/1024/1024 Current_Size
                ,(u.used+a.file_count*65536)/1024/1024/1024 Used_size
                ,(a.alloc_size-(u.used+a.file_count*65536))/1024/1024/1024 Available_size
                ,((u.used+a.file_count*65536)*100)/a.alloc_size Pct_used
        from     dba_tablespaces t
                ,(select t1.tablespace_name
                ,nvl(sum(s.bytes),0) used
                from  dba_segments s
                ,dba_tablespaces t1
                 where t1.tablespace_name=s.tablespace_name(+)
                 group by t1.tablespace_name) u
                ,(select d.tablespace_name
                ,sum(greatest(d.bytes,nvl(d.maxbytes,0))) alloc_size
                ,sum(d.bytes) cur_size
                ,count(*) file_count
                from dba_data_files d
                group by d.tablespace_name) a
        where t.tablespace_name=u.tablespace_name and t.tablespace_name=a.tablespace_name order by t.tablespace_name;
        ```
        
    - Verificar bloco do datafile corrompido
        
        ```sql
        SELECT DISTINCT owner, segment_name
        FROM   v$database_block_corruption dbc
               JOIN dba_extents e ON dbc.file# = e.file_id AND dbc.block# BETWEEN e.block_id and e.block_id+e.blocks-1
        ORDER BY 1,2;
        
        ou
        
        select OWNER, segment_name, segment_type, tablespace_name, block_id from dba_extents 
        where file_id = 1 and 81408 between block_id and block_id + blocks-1;
        
        //
        
        SELECT
            dbc.file# AS FILE_NUMBER,
            dbc.blocks AS BLOCKS,
            e.owner,
            e.segment_name
        FROM 
            v$database_block_corruption dbc
        JOIN 
            dba_extents e ON dbc.file# = e.file_id 
            AND dbc.block# BETWEEN e.block_id AND e.block_id + e.blocks - 1
        WHERE 
            e.segment_name IN ('BRK_LOG_CREATE_AUTOMATED_EVENT');
            
         //
         
         SELECT
            dbc.file# AS FILE_NUMBER,
            dbc.BLOCK# AS BLOCK,
            dbc.blocks AS BLOCKS,
            e.owner,
            e.segment_name
        FROM
                 v$database_block_corruption dbc
            JOIN dba_extents e ON dbc.file# = e.file_id
                                  AND dbc.block# BETWEEN e.block_id AND e.block_id + e.blocks - 1
        ORDER BY
            e.owner;
            
         //
         
         SELECT
            dbc.file# AS FILE_NUMBER,
            e.blocks AS BLOCKS,
            e.owner,
            e.segment_name
        FROM 
            v$database_block_corruption dbc
        JOIN 
            dba_extents e ON dbc.file# = e.file_id 
            AND dbc.block# BETWEEN e.block_id AND e.block_id + e.blocks - 1
        WHERE 
            e.segment_name IN (
                'BRK_LOG_CREATE_AUTOMATED_EVENT',
                'ICT_MASSIVAS_LIVE_ALM',
                'ICT_SN_BLOCK_NRML_ALARMS_PRD',
                'ICT_METRICS_ODM_REQS_PRD',
                'ALARMUPDATESN',
                'ICT_BBN_ALARMES_ROTAS',
                'VW_ALL_ACTIVE_SITES_NRM_CELL',
                'MV_CTRL_PHYSICAL_ACCES',
                'ICT_ELOC_COMUNICADO',
                'ICT_TABLES.ICT_ELOC_EVENTOS_INPUT',
                'ICT_TABLES.VW_ELOC_EVENTOS_AD',
                'MV_ICT_SITESDOWN_MASS_REG'
            );
            
            //
            
            SELECT owner, table_name, ROUND((num_rows * avg_row_len / 1024 / 1024 ), 2) AS size_MB, num_rows
        FROM all_tables
        WHERE table_name IN ('BRK_LOG_CREATE_AUTOMATED_EVENT')
        AND owner IN ('BROKERNW');
            
            
            //
            
            
            BEGIN
        DBMS_REPAIR.SKIP_CORRUPT_BLOCKS [
        SCHEMA NAME => 'BROKERNW',
        OBJECT_NAME => 'BRK_LOG_CREATE_AUTOMATED_EVENT',
        OBJECT_TYPE => DBMS_REPAIR.TABLE_OBJECT) ;
        END;
        
        ```
        
    - Monitoramento bloco do datafile corrompido
        
        ```sql
        SELECT COALESCE(SUM(corruption_count), 0) AS total_corruption FROM (SELECT COUNT(*) AS corruption_count FROM v$database_block_corruption dbc JOIN dba_data_files df ON df.file_id = dbc.file# WHERE df.file_id NOT IN (28, 23) GROUP BY df.tablespace_name);
        ```
        
    - Bloco corrompido se for LOB, descobrir a tabela
        
        ```sql
        --SE LOB, DESCOBRIR A TABELA
        SELECT table_name, column_name
        FROM dba_lobs
        WHERE segment_name = 'SYS_LOB0006846994C00012$$';
        ```
        
    - Bloco corrompido  descobrir segmento
        
        ```sql
        --DESCOBRIR SEGMENTO (EXEMPLO)
        SELECT s.owner, s.segment_name, s.segment_type
        FROM dba_segments s
        JOIN dba_extents e ON s.segment_name = e.segment_name
        WHERE e.file_id = 138
          AND e.block_id <= 85367
          AND e.block_id + e.blocks > 85367;
        ```
        
- UNDO
    - Alterar UNDO
        
        ```sql
        SELECT 'ALTER DATABASE DATAFILE ''' || d.name || ''' AUTOEXTEND OFF;' AS statement
        FROM v$datafile d, v$tablespace t
        WHERE d.TS# = t.TS#
        AND t.name = 'UNDOTBS1';
        ```
        
    - Criar UNDO
        
        ```sql
        UNDOTBS1
        
        CREATE UNDO TABLESPACE "UNDOTBS1" DATAFILE
        SIZE 26214400
        BLOCKSIZE 8192
        EXTENT MANAGEMENT LOCAL AUTOALLOCATE;
        ```
        
    - Recriar UNDO
        
        ```sql
        --Verificar espaço disponível em disco para executar o procedimento
        
        --Criar uma nova tablespace
        CREATE BIGFILE UNDO TABLESPACE <NOME_DA_UNDO> DATAFILE SIZE 32G AUTOEXTEND ON NEXT 1G MAXSIZE 64G;
        
        --Alterar undo 
        ALTER SYSTEM SET UNDO_TABLESPACE =<NOME_DA_UNDO>scope=both;
        
        Verificar se ocorreu a mudança: show parameter undo;
        
        --Verificar se há alguma sessão na UNDO antiga
        SELECT s.sid, s.serial#, s.username, t.start_time, t.used_urec
        FROM v$transaction t, v$session s
        WHERE t.ses_addr = s.saddr
          AND t.xidusn IN (SELECT ts# FROM v$tablespace WHERE name = 'NOME_DA_UNDO');
        
        Caso haja alguma sessão ativa na undo matar a sessão:
        alter system disconnect session 'SID,SERIAL' immediate;
          
        --Dropar undo antiga
        DROP TABLESPACE UNDOTBS2 INCLUDING CONTENTS AND DATAFILES;
        
        Após dropar a UNDO repetir todo o processo para que a mesma volte a ser UNDOTBS1;
        Se houver sessões que estejam ocupando a UNDO e não seja possível desconectar essas sessões, terá que ser reiniciado o banco. 
        ```
        
    - Mostra as queries de maior duração no banco de dados
        
        ```sql
        # Mostra as queries de maior duração no banco de dados.
        select begin_time, end_time, undotsn, undoblks, maxquerylen, maxqueryid, activeblks, unexpiredblks, expiredblks, tuned_undoretention from v$undostat 
        where trunc(begin_time)=trunc(sysdate)-1 order by maxquerylen DESC;
        
        # Pega o valor da query com maior duração
        select dbms_undo_adv.longest_query(sysdate-1,sysdate) from dual;
        
        #Use a query abaixo para definir o tamanho da UNDO em Megabytes
        select dbms_undo_adv.required_undo_size(12898,sysdate-2,sysdate) from dual;
        
        *************
        Últimas 24h
        
        set pagesize 1000 linesize 180
        col "begin_time" for 99999.999
        col "end_time" for 99999.999
        col "undotsn" for 9
        col "undoblks" for 9999999
        col "maxquerylen" for 9
        col "maxqueryid" for 999.99
        col "activeblks" for 9
        col "unexpiredblks" for 9
        col "expiredblks" for 9
        COL "tuned_undoretention" for 99999
        SELECT begin_time, end_time, undotsn, undoblks, maxquerylen, maxqueryid, activeblks, unexpiredblks, expiredblks, tuned_undoretention 
        FROM v$undostat 
        WHERE begin_time >= SYSDATE - INTERVAL '1' DAY 
        ORDER BY maxquerylen DESC;
        ```
        
    - Análise consumo de UNDO
        
        ```sql
        --Verificar transações ativas segurando UNDO
        SELECT s.sid,
               s.serial#,
               s.username,
               t.used_ublk,
               t.used_urec,
               s.status
        FROM v$transaction t
        JOIN v$session s ON t.ses_addr = s.saddr
        ORDER BY t.used_ublk DESC;
        
        --Diagnóstico real da causa (utilização em MB)
        SELECT s.sid,
               s.serial#,
               s.username,
               s.program,
               t.used_ublk,
               ROUND(t.used_ublk * 8 / 1024,2) AS undo_mb
        FROM v$transaction t
        JOIN v$session s ON t.ses_addr = s.saddr
        ORDER BY t.used_ublk DESC;
        
        --Verificar active, unexpired e expired
        SELECT 
          ROUND(SUM(CASE WHEN status='ACTIVE' THEN bytes END)/1024/1024/1024,2) ACTIVE_GB,
          ROUND(SUM(CASE WHEN status='UNEXPIRED' THEN bytes END)/1024/1024/1024,2) UNEXPIRED_GB,
          ROUND(SUM(CASE WHEN status='EXPIRED' THEN bytes END)/1024/1024/1024,2) EXPIRED_GB
        FROM dba_undo_extents;
        
        --Ver consumo histórico de UNDO (por intervalo)
        SELECT 
            TO_CHAR(begin_time,'DD/MM HH24:MI') begin_time,
            TO_CHAR(end_time,'DD/MM HH24:MI') end_time,
            undoblks,
            txncount,
            MAXQUERYLEN
        FROM dba_hist_undostat
        ORDER BY begin_time DESC
        FETCH FIRST 48 ROWS ONLY;
        
        --Identificar qual SQL estava pesado por intevalo de horário
        SELECT 
            s.snap_id,
            s.begin_interval_time,
            ss.sql_id,
            ss.executions_delta,
            ROUND(ss.rows_processed_delta) rows_processed,
            ROUND(ss.buffer_gets_delta/1000000,2) buffer_gets_milhoes
        FROM dba_hist_sqlstat ss
        JOIN dba_hist_snapshot s ON ss.snap_id = s.snap_id
        WHERE s.begin_interval_time BETWEEN 
              TO_DATE('03/03/2026 14:00','DD/MM/YYYY HH24:MI')
          AND TO_DATE('03/03/2026 16:00','DD/MM/YYYY HH24:MI')
        ORDER BY ss.buffer_gets_delta DESC;
        
        --Pegar o SQL_text
        SELECT sql_text
        FROM dba_hist_sqltext
        WHERE sql_id = 'SQL_ID';
        
        --Descobrir qual SQL foi responsável
        SELECT 
            ss.sql_id,
            ss.executions_delta,
            ROUND(ss.rows_processed_delta) rows_processed,
            ROUND(ss.buffer_gets_delta/1000000,2) buffer_gets_milhoes
        FROM dba_hist_sqlstat ss
        JOIN dba_hist_snapshot s ON ss.snap_id = s.snap_id
        WHERE s.begin_interval_time BETWEEN 
              TO_DATE('28/02/2026 19:40','DD/MM/YYYY HH24:MI')
          AND TO_DATE('28/02/2026 20:10','DD/MM/YYYY HH24:MI')
        ORDER BY ss.buffer_gets_delta DESC
        FETCH FIRST 15 ROWS ONLY;
        
        --Identificar o usuário responsável
        SELECT parsing_schema_name
        FROM dba_hist_sqlstat
        WHERE sql_id = 'f1mpttm8bz9zq'
        FETCH FIRST 1 ROW ONLY;
        ```
        
    - Análise UNDO
        
        ```sql
        COLUMN sid FORMAT 99999
        COLUMN serial# FORMAT 999999
        COLUMN username FORMAT A15
        COLUMN undo_bytes FORMAT 9999999999
        COLUMN osuser FORMAT A20
        COLUMN machine FORMAT A30
        COLUMN program FORMAT A40
        
        SELECT 
            s.sid, 
            s.serial#, 
            s.username, 
            t.used_ublk * TO_NUMBER(p.value) AS undo_bytes, 
            s.osuser, 
            s.machine, 
            s.program
        FROM 
            v$transaction t
        JOIN 
            v$session s 
            ON t.ses_addr = s.saddr
        JOIN 
            v$parameter p 
            ON p.name = 'db_block_size'
        ORDER BY 
            undo_bytes DESC;
        
        SELECT s.sid, s.serial#, s.username, s.status, s.osuser, s.program, s.machine
        FROM v$session s
        WHERE s.sid = 154 AND s.serial# = 29486;
        
        SELECT s.sid, s.serial#, s.username, s.status, w.event, w.seconds_in_wait
        FROM v$session s
        JOIN v$session_wait w ON s.sid = w.sid
        WHERE s.sid = 2563 AND s.serial# = 50757;
        ```
        
    - Verificar consumo atual
        
        ```sql
        select 
          a.sid, 
          a.serial#, a.username, b.used_urec used_undo_record, b.used_ublk used_undo_blocks
        from 
          v$session a, 
          v$transaction b 
        where 
          a.saddr = b.ses_addr;
        ```
        
    - Verificar quais querys estão na UNDO
        
        ```sql
        SELECT a.name,b.status , d.username , d.sid , d.serial#
        FROM   v$rollname a,v$rollstat b, v$transaction c , v$session d
        WHERE  a.usn = b.usn
        AND    a.usn = c.xidusn
        AND    c.ses_addr = d.saddr
        AND    a.name IN ( 
          SELECT segment_name
          FROM dba_segments 
         WHERE tablespace_name = 'UNDOTBS2');
        ```
        
    - Verificar ofensores
        
        ```sql
        SET LINESIZE 120
        SET PAGESIZE 100
        TTITLE CENTER 'Monitoramento de Operações Longas no Banco de Dados'
        
        COLUMN "SID" FORMAT A10
        COLUMN "Serial#" FORMAT A10
        COLUMN "Target" FORMAT A20
        COLUMN "Username" FORMAT A15
        COLUMN "SQL ID" FORMAT A15
        COLUMN "Opname" FORMAT A20
        COLUMN "Start Time" FORMAT A20
        COLUMN "SQL Plan Options" FORMAT A20
        COLUMN "SQL Plan Operation" FORMAT A25
        COLUMN "Percent Complete" FORMAT 999.99
        COLUMN "Time Remaining" FORMAT A20
        COLUMN "Message" FORMAT A40
        
        SELECT 
          SID AS "SID",
          SERIAL# AS "Serial#",
          target AS "Target",
          username AS "Username",
          sql_id AS "SQL ID",
          opname AS "Opname",
          START_TIME AS "Start Time",
          sql_plan_options AS "SQL Plan Options",
          sql_plan_operation AS "SQL Plan Operation",
          ROUND((SOFAR / TOTALWORK) * 100, 2) AS "Percent Complete",
          TIME_REMAINING/60 AS "Time Remaining (Min)",
          MESSAGE AS "Message"
        FROM 
          V$SESSION_LONGOPS
        WHERE 
          TIME_REMAINING > 0 
        ORDER BY 
          START_TIME;
        ```
        
    - Status da UNDO
        
        ```sql
        SELECT DISTINCT STATUS, SUM(BYTES)/1024/1024/1024 sum_in_GB, COUNT(*) FROM DBA_UNDO_EXTENTS GROUP BY STATUS;
        ```
        
        ```sql
        select tablespace_name tablespace, status, sum(bytes)/1024/1024/1024 sum_in_GB, count(*) counts
        from dba_undo_extents
        group by tablespace_name, status order by 1,2;
        ```
        
    - Verificar segmentos que estão ocupando a UNDO
        
        ```sql
        --verificar quais segmentos ainda estão ocupando espaço na UNDO
        SELECT tablespace_name, status, sum(bytes)/1024/1024 AS tamanho_MB
        FROM dba_undo_extents
        GROUP BY tablespace_name, status;
        ```
        
    - Forçar limpeza da UNDO
        
        ```sql
        ALTER SYSTEM SET UNDO_RETENTION = 600 SCOPE=BOTH;
        ```
        
    - Verificar qual a capacidade de resize da undo
        
        ```sql
        select file_id,(USER_BYTES/1024/1024/1024) ,(MAXBYTES/1024/1024/1024) from dba_data_files where tablespace_name='UNDOTBS1';
        
        ou
        
        set pagesize 1000 linesize 180
        select file_id, USER_BYTES ,MAXBYTES from dba_data_files where tablespace_name='UNDOTBS1';
        ```
        
    - Query para determinar o tamanho ideal da UNDO
        
        ```sql
        # Mostra as queries de maior duração no banco de dados.
        
        select begin_time, end_time, undotsn, undoblks, maxquerylen, maxqueryid, activeblks, unexpiredblks, expiredblks, tuned_undoretention from v$undostat 
        where trunc(begin_time)=trunc(sysdate)-1 order by maxquerylen DESC;
        
        # Pegue o valor da query com maior duração
        select dbms_undo_adv.longest_query(sysdate-1,sysdate) from dual;
        
        #Use a query abaixo para definir o tamanho da UNDO em Megabytes
        select dbms_undo_adv.required_undo_size(1133887,sysdate-1/12,sysdate) from dual;
        
        **O valor 153873 é um valor de exemplo, o valor a ser colocado é o valor da query com maior duração
        ```
        
    - Resize UNDO
        
        ```sql
        ALTER TABLESPACE UNDOTBS2 RESIZE 358400M;
        ```
        
    - Verificar sessões ativas na UNDO
        
        ```sql
        --Consulta para identificar uso de UNDO por sessões ativas--
        SELECT s.sid, s.serial#, s.username, s.status, s.osuser, s.machine, s.program, s.logon_time,
               u.used_ublk "Undo Blocks Used",
               u.used_urec "Undo Records Used"
        FROM gv$session s
        JOIN gv$transaction t ON s.saddr = t.ses_addr
        JOIN gv$transaction u ON s.saddr = u.ses_addr
        WHERE s.type <> 'BACKGROUND'
        ORDER BY u.used_ublk DESC;
        ********************************
        
        set pagesize 1000
        set linesize 180
        
        col SID for 99999
        col SERIAL# for 99999
        col USERNAME for a30
        col SQL_ID for a13
        
        SELECT s.SID, 
               s.SERIAL#, 
               s.USERNAME, 
               s.SQL_ID
        FROM V$SESSION s
        WHERE s.SQL_ID IN (SELECT SQL_ID FROM V$SQL WHERE SQL_TEXT LIKE '%UNDO%')
        AND s.STATUS = 'ACTIVE';
        
        select tablespace_name, status, count(*) from dba_rollback_segs group by tablespace_name, status;
        ```
        
    - Verificar sessões que estão utilizando a UNDO
        
        ```sql
        --obter informações sobre sessões que estão utilizando undo--
        SELECT a.name,b.status , d.username , d.sid , d.serial#
        FROM   v$rollname a,v$rollstat b, v$transaction c , v$session d
        WHERE  a.usn = b.usn
        AND    a.usn = c.xidusn
        AND    c.ses_addr = d.saddr
        AND    a.name IN (
        SELECT segment_name
        FROM dba_segments
        WHERE tablespace_name = 'UNDOTBS1');
        ```
        
    - Tamanho máximo, atual e espaço livre
        
        ```sql
        SELECT tablespace_name,
        ROUND(MAX(bytes) / 1024 / 1024) AS max_size_mb,
        ROUND(SUM(bytes) / 1024 / 1024) AS current_size_mb,
        ROUND(MAX(bytes) / 1024 / 1024 - SUM(bytes) / 1024 / 1024) AS free_space_mb
        FROM dba_undo_extents
        GROUP BY tablespace_name;
        
        -> No SQLPLUS
        
        COLUMN tablespace_name FORMAT A20
        COLUMN max_size_mb FORMAT 999,999
        COLUMN current_size_mb FORMAT 999,999
        COLUMN free_space_mb FORMAT 999,999
        
        SELECT tablespace_name,
               ROUND(MAX(bytes) / 1024 / 1024) AS max_size_mb,
               ROUND(SUM(bytes) / 1024 / 1024) AS current_size_mb,
               ROUND(MAX(bytes) / 1024 / 1024 - SUM(bytes) / 1024 / 1024) AS free_space_mb
        FROM dba_undo_extents
        GROUP BY tablespace_name;
        ```
        
    - DROP UNDO
        
        ```sql
        DROP TABLESPACE antiga_undo_data INCLUDING CONTENTS AND DATAFILES;
        ```
        
    - Definir UNDO padrão
        
        ```sql
        ALTER SYSTEM SET UNDO_TABLESPACE = nova_undo_data;
        ```
        
    - Verificar UNDO online
        
        ```sql
        SELECT tablespace_name
        FROM dba_tablespaces
        WHERE contents = 'UNDO' AND status = 'ONLINE';
        ```
        
    - Undo RETENTION
        
        ```sql
        SELECT VALUE
        FROM V$PARAMETER
        WHERE NAME = 'undo_retention';
        ```
        
    - Verificar parâmetros UNDO
        
        ```sql
        SELECT VALUE
        FROM V$PARAMETER
        WHERE NAME = 'undo_management';
        ```
        
    - Tamanho alocação, tamanho usado
        
        ```sql
        SELECT 
          size_allocated.tablespace_name, 
          size_allocated.size_allocated_mb, 
          size_used.size_used_mb, 
          ROUND (
            size_used.size_used_mb / size_allocated.size_allocated_mb * 100, 
            2
          ) pct_size_used_mb 
        FROM 
          (
            SELECT 
              due.tablespace_name, 
              SUM (due.bytes) / 1024 / 1024 AS size_used_mb 
            FROM 
              dba_undo_extents due 
            GROUP BY 
              due.tablespace_name
          ) size_used, 
          (
            SELECT 
              dt.tablespace_name, 
              SUM (ddf.bytes) / 1024 / 1024 size_allocated_mb 
            FROM 
              dba_tablespaces dt, 
              dba_data_files ddf 
            WHERE 
              dt.tablespace_name = ddf.tablespace_name 
              AND dt.contents = 'UNDOTBS02' 
            GROUP BY 
              dt.tablespace_name
          ) size_allocated 
        WHERE 
          size_allocated.tablespace_name = size_used.tablespace_name(+) 
        ORDER BY 
          tablespace_name;
        
        ```
        
- FRA
    - Script verificar FRA
        
        ```sql
        set lines 200 pages 200 
        col db_name for a15
        col host for a20
        col ip_address for a15
        col name for a30
        col used_gb for 9999999999
        col max_gb for 9999999999
        col number_of_files for 9999999999
        col data_coleta for a20
        select 
            TO_CHAR(SYSDATE, 'DD/MM/YYYY HH24:MI:SS') as data_coleta,
            sys_context('userenv','DB_NAME') as db_name,
            sys_context('userenv','HOST') as host,
            utl_inaddr.get_host_address as ip_address,
            name as "NAME", 
            round(space_limit/1024/1024/1024,2) as MaxGB, 
            round(space_used/1024/1024/1024,2) as UsedGB, 
            round((space_used/space_limit)*100,2) as "USED %",
            number_of_files 
        from v$recovery_file_dest;
        
        ```
        
    - Configuração FRA
        
        ```sql
        Configurando a Fast Recovery Área
        
        Verificar parâmetros atuais
        show parameter db_recovery_file;
        
        Alterar tamanho
        alter system set db_recovery_file_dest_size=70G scope=both;
        ```
        
    - Verificar quem está consumindo espaço na FRA
        
        ```sql
        SELECT 
            FILE_TYPE AS "Tipo de Arquivo",
            PERCENT_SPACE_USED AS "% Espaço Usado",
            PERCENT_SPACE_RECLAIMABLE AS "% Espaço Recup.",
            NUMBER_OF_FILES AS "Qtd Arquivos"
        FROM 
            V$FLASH_RECOVERY_AREA_USAGE
        WHERE 
            PERCENT_SPACE_USED > 0;
        ```
        
- REDOLOG
    - Redolog
        
        ```sql
        Alter system set log_file_name_convert= '+REDO1/ARSDB/ONLINELOG', '+REDO1/ARSDBSTB/ONLINELOG', '+DATA/ARSDB/ONLINELOG', '+DATA/ARSDBSTB/ONLINELOG' SCOPE=SPFILE;
        
        SELECT name, path, state, mount_status 
        FROM v$asm_disk 
        WHERE group_number = (SELECT group_number FROM v$asm_diskgroup WHERE name = 'REDO');
        ```
        
    - Verificar tamanho dos Redologs
        
        ```sql
        set lines 200 pages 200
        col l.group# format a5
        col l.sequence# format a5
        col l.THREAD# format a5
        col mbytes format 9999999999
        col l.members format a3
        col l.archived format a5
        col l.status# format a10
        col first_time format a20
        col redo_filename format a80
        select l.group#,  l.sequence#, l.THREAD#,
              (l.bytes/1024/1024) mbytes, l.members,
              l.archived, l.status,       
              to_char(l.first_time,'dd/mm/yyyy hh24:mi:ss') first_time,
              f.member as redo_filename
        from  V$LOG l
        join  V$LOGFILE f
          on  l.group# = f.group#
        order by 1;
        ```
        
    - Verificar quanto o banco está pedindo de redologs
        
        ```sql
        select  OPTIMAL_LOGFILE_SIZE from V$INSTANCE_RECOVERY;
        ```
        
    - Verificar quantidade de Redologs
        
        ```sql
        SELECT
            TO_CHAR(FIRST_TIME, 'YYYY-MM-DD HH24') AS HORA,
            COUNT(*) AS QUANTIDADE_DE_REDOLOGS
        FROM
            V$LOG_HISTORY
        GROUP BY
            TO_CHAR(FIRST_TIME, 'YYYY-MM-DD HH24')
        ORDER BY
            HORA;
        ```
        
    - Verificar standby Redologs
        
        ```sql
        SET LINESIZE 200
        SET PAGESIZE 100
        COL "GROUP" FORMAT 999999999999
        COL "STATUS" FORMAT A10
        COL "NAME" FORMAT A50
        COL "SIZE_MB" FORMAT 9999
        -- Consulta
        SELECT 
            L.GROUP# AS "GROUP",
            L.STATUS AS "STATUS",
            LF.MEMBER AS "NAME",
            L.BYTES / 1024 / 1024 AS "SIZE_MB"
        FROM 
            V$LOG L
        JOIN 
            V$LOGFILE LF ON L.GROUP# = LF.GROUP#
        UNION ALL
        SELECT 
            SL.GROUP# AS "GROUP",
            SL.STATUS AS "STATUS",
            LF.MEMBER AS "NAME",
            SL.BYTES / 1024 / 1024 AS "SIZE_MB"
        FROM 
            V$STANDBY_LOG SL
        JOIN 
            V$LOGFILE LF ON SL.GROUP# = LF.GROUP#
        ORDER BY "GROUP";
        ```
        
    - Adicionar Redologs
        
        ```sql
        ALTER DATABASE ADD LOGFILE THREAD 1
        GROUP 101 ('+REDO1','+REDO2') SIZE 20G,
        GROUP 102 ('+REDO1','+REDO2') SIZE 20G,
        GROUP 103 ('+REDO1','+REDO2') SIZE 20G,
        GROUP 104 ('+REDO1','+REDO2') SIZE 20G;
        
        ALTER DATABASE ADD STANDBY LOGFILE THREAD 1
        GROUP 201 ('+REDO1','+REDO2') SIZE 20G,
        GROUP 202 ('+REDO1','+REDO2') SIZE 20G,
        GROUP 203 ('+REDO1','+REDO2') SIZE 20G,
        GROUP 204 ('+REDO1','+REDO2') SIZE 20G;
        ```
        
    - Comando para soltar os redologs
        
        ```sql
        ALTER SYSTEM SWITCH LOGFILE;
        alter system checkpoint global;
        ```
        
    - Comando para dropar os redologs
        
        ```sql
        alter database drop logfile group 1;
        alter database drop logfile group 2;
        alter database drop logfile group 3;
        alter database drop logfile group 4;
        ```
        
- COLETA DE ESTATÍSTICAS
    - Verificar estatísticas
        
        ```sql
        select t.owner, t.table_name, t.last_analyzed 
        from all_all_tables t 
        where t.last_analyzed < (SYSDATE-30) and t.owner not in ('SYS'
                                                                    ,'AUDSYS'
                                                                    ,'APPQOSSYS'
                                                                    ,'CTXSYS'
                                                                    ,'DBSNMP'
                                                                    ,'DBFW'
                                                                    ,'DBSFWUSER'
                                                                    ,'DVSYS'
                                                                    ,'GSMADMIN_INTERNAL'
                                                                    ,'LBACSYS'
                                                                    ,'MDSYS'
                                                                    ,'OJVMSYS','OLAPSYS','ORDDATA','ORDSYS','OUTLN','SYSTEM','WMSYS','XDB') 
        order by t.owner, t.last_analyzed;
        
        ******
        
        --Verificar estatísticas de um schema específico
        set pagesize 1000 linesize 180
        COLUMN table_name      FORMAT A30
        COLUMN last_analyzed   FORMAT A20
        COLUMN num_rows        FORMAT 99999999
        COLUMN blocks          FORMAT 99999999
        COLUMN empty_blocks    FORMAT 9999999
        COLUMN avg_space       FORMAT 999999
        SELECT table_name,
               last_analyzed,
               num_rows,
               blocks,
               empty_blocks,
               avg_space
        FROM   all_tables
        WHERE  owner = 'SGF_PRODUCAO'
        ORDER BY last_analyzed;
        
        -- Verificar estatísticas dos indexes
        set pagesize 1000 linesize 180
        COLUMN index_name      FORMAT A30
        COLUMN table_name      FORMAT A30
        COLUMN last_analyzed   FORMAT A20
        COLUMN leaf_blocks     FORMAT 999999999
        COLUMN distinct_keys   FORMAT 999999999
        SELECT index_name,
               table_name,
               last_analyzed,
               leaf_blocks,
               distinct_keys
        FROM   all_indexes
        WHERE  owner = 'MLOG_HOM'
        ORDER BY last_analyzed;
        ```
        
    - Resumo Histórico Job Optimizer
        
        ```sql
        SET LINES 200 PAGES 200
        TTITLE CENTER 'Auto Optimizer Stats Collection History'
        COL job_name FOR a40
        COL window_name FOR a30
        COL job_start_time FOR a20
        COL job_duration FOR a15
        COL status FOR a20
        
        SELECT 
        *
        FROM 
            DBA_AUTOTASK_JOB_HISTORY
        WHERE 
            Client_name = 'auto optimizer stats collection'
        ORDER BY 
            job_start_time;
        ```
        
    - Atualizar estatísticas do schema
        
        ```sql
        EXEC DBMS_STATS.gather_schema_stats('MLOGHOM', estimate_percent => 20, cascade => TRUE, degree =>4);
        ```
        
    - Atualizar estatísticas do banco I
        
        ```sql
        EXEC dbms_stats.gather_fixed_objects_stats();
        exec dbms_stats.gather_system_stats();
        EXEC DBMS_STATS.GATHER_DICTIONARY_STATS;
        EXEC DBMS_STATS.GATHER_DATABASE_STATS(options => 'GATHER', estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE, degree => DBMS_STATS.AUTO_DEGREE, no_invalidate => TRUE, cascade => TRUE, method_opt => 'FOR ALL COLUMNS SIZE AUTO', granularity => 'AUTO' );
        ```
        
    - Atualizar estatísticas do banco
        
        ```sql
        EXEC dbms_stats.gather_fixed_objects_stats();
        exec dbms_stats.gather_system_stats();
        EXEC DBMS_STATS.GATHER_DICTIONARY_STATS;
        EXEC DBMS_STATS.gather_schema_stats('SYS', estimate_percent => 20, cascade => TRUE, degree =>4);
        EXEC DBMS_STATS.gather_schema_stats('SYSTEM', estimate_percent => 20, cascade => TRUE, degree =>4);
        EXEC DBMS_STATS.GATHER_DATABASE_STATS(options => 'GATHER STALE', estimate_percent => DBMS_STATS.AUTO_SAMPLE_SIZE, degree => DBMS_STATS.AUTO_DEGREE, no_invalidate => TRUE, cascade => TRUE, method_opt => 'FOR ALL COLUMNS SIZE AUTO', granularity => 'AUTO' );
        ```
        
    - Atualizar estatísticas de tabela
        
        ```sql
        EXEC DBMS_STATS.gather_table_stats(
            'INTERMITENT',                         -- Nome do esquema (schema)
            'ICT_INTERMITTENT_ALARMS_HIST_OLD',     -- Nome da tabela para a qual as estatísticas serão coletadas
            estimate_percent => 20,                 -- Percentual de amostragem dos dados para estimar as estatísticas
            cascade => TRUE,                        -- Se TRUE, as estatísticas das colunas e índices serão coletadas também
            degree => 3                             -- Número de threads paralelas para realizar o trabalho
        );
        ```
        
    - Relatório tamanho tabelas desatualizadas
        
        ```sql
        SET LINES 200 PAGES 200
        TTITLE CENTER 'Relatorio de Tabelas nao atualizadas com Tamanho em GB'
        COL owner FOR a20
        COL table_name FOR a30
        COL last_analyzed FOR a20
        COL table_size_gb FOR 9999999999.99
        COL num_rows FOR 99999999999
        
        SELECT
            t.owner AS "OWNER",
            t.table_name AS "TABLE_NAME",
            TO_CHAR(t.last_analyzed, 'YYYY-MM-DD HH24:MI:SS') AS "LAST_ANALYZED",
            ROUND(SUM(s.bytes) / (1024 * 1024 * 1024), 2) AS "TABLE_SIZE_GB",
            t.num_rows AS "NUM_ROWS"
        FROM
            dba_tables t
        JOIN
            dba_segments s ON t.owner = s.owner AND t.table_name = s.segment_name
        WHERE
            t.last_analyzed < (SYSDATE - 30)
            AND t.owner NOT IN (
                SELECT username 
                FROM dba_users 
                WHERE oracle_maintained = 'Y'
            )
        GROUP BY
            t.owner,
            t.table_name,
            t.last_analyzed,
            t.num_rows
        ORDER BY
            t.owner,
            t.last_analyzed;
        ```
        
    - Relatório tabelas desatualizadas
        
        ```sql
        SPOOL log.txt
        
        SELECT
            t.owner,
            t.table_name,
            t.last_analyzed
        FROM
            all_all_tables t
        WHERE
                t.last_analyzed < ( sysdate - 30 )
            AND t.owner NOT IN ( 'SYS', 'AUDSYS', 'APPQOSSYS', 'CTXSYS', 'DBSNMP',
                                 'DBFW', 'DBSFWUSER', 'DVSYS', 'GSMADMIN_INTERNAL', 'LBACSYS',
                                 'MDSYS', 'OJVMSYS', 'OLAPSYS', 'ORDDATA', 'ORDSYS',
                                 'OUTLN', 'SYSTEM', 'WMSYS', 'XDB' )
        ORDER BY
            t.owner,
            t.last_analyzed;
        
        SPOOL OFF;
        ```
        
    - Consultar objetos sem estatísticas
        
        ```sql
        SELECT 
          'TABLE' object_type, 
          owner, 
          table_name object_name, 
          last_analyzed, 
          stattype_locked, 
          stale_stats 
        FROM 
          all_tab_statistics 
        WHERE 
          (
            last_analyzed IS NULL 
            OR stale_stats = 'YES'
          ) 
          and stattype_locked IS NULL 
          and owner NOT IN (
            'ANONYMOUS', 'CTXSYS', 'DBSNMP', 'EXFSYS', 
            'LBACSYS', 'MDSYS', 'MGMT_VIEW', 
            'OLAPSYS', 'OWBSYS', 'ORDPLUGINS', 
            'ORDSYS', 'OUTLN', 'SI_INFORMTN_SCHEMA', 
            'SYS', 'SYSMAN', 'SYSTEM', 'TSMSYS', 
            'WK_TEST', 'WKSYS', 'WKPROXY', 'WMSYS', 
            'XDB'
          ) 
          AND owner NOT LIKE 'FLOW%' 
        UNION ALL 
        SELECT 
          'INDEX' object_type, 
          owner, 
          index_name object_name, 
          last_analyzed, 
          stattype_locked, 
          stale_stats 
        FROM 
          all_ind_statistics 
        WHERE 
          (
            last_analyzed IS NULL 
            OR stale_stats = 'YES'
          ) 
          and stattype_locked IS NULL 
          AND owner NOT IN (
            'ANONYMOUS', 'CTXSYS', 'DBSNMP', 'EXFSYS', 
            'LBACSYS', 'MDSYS', 'MGMT_VIEW', 
            'OLAPSYS', 'OWBSYS', 'ORDPLUGINS', 
            'ORDSYS', 'OUTLN', 'SI_INFORMTN_SCHEMA', 
            'SYS', 'SYSMAN', 'SYSTEM', 'TSMSYS', 
            'WK_TEST', 'WKSYS', 'WKPROXY', 'WMSYS', 
            'XDB'
          ) 
          AND owner NOT LIKE 'FLOW%' 
        ORDER BY 
          object_type desc, 
          owner, 
          object_name;
        
        ```
        
- FLASHBACK RESTORE CHECKPOINT
    
    ```sql
    archive log list;
    
    show parameter db_recovery_file;
    
    show parameter retention;
    
    ALTER DATABASE FLASHBACK ON;
    
    CREATE RESTORE POINT CHG0054013 guarantee flashback database;
    
    -- Apagar e desabilitar FLASHBACK
    
    SELECT NAME, STORAGE_SIZE FROM V$RESTORE_POINT WHERE GUARANTEE_FLASHBACK_DATABASE = 'YES';
    
    DROP RESTORE POINT ANTESCHG;
    
    ALTER DATABASE FLASHBACK OFF;
    
    --Restaurar para o ponto criado
    select current_scn from v$database;
    
    SHUTDOWN IMMEDIATE;
    
    startup mount;
    
    select * from v$restore_point;
    
    flashback database to restore point restpointcriado;
    
    alter database open resetlogs;
    
    alter database open;
    
    Tenho uma teoria de como esse processo é feito na categoria banco
    
    http://www.dba-oracle.com/t_expdp_flashback_time.htm
    
    http://rodrigo-oracle.blogspot.com/2010/06/trabalhando-com-restore-points-em-um.html
    ```
    
- GRANTS
    - Conceder permissões (DBA_SYS_PRIVS, DBA_TAB_PRIVS, DBA_ROLE_PRIVS):
        
        ```sql
        
        Privilégios:
        achar tabelas e viiews dos owners:
        
        SELECT owner, object_name, object_type, created
        FROM all_objects
        WHERE owner IN ('TELCOMANAGER', 'CONSULTA_NETCOOL', 'ZBXUSER', 'MONITORAÇÃO_ZABBIX')
        AND object_type IN ('TABLE', 'VIEW')
        ORDER BY owner, object_name;
        
        Tabelas:
        
        SELECT 'GRANT '||PRIVILEGE||' ON '||owner||'.'||table_name||' to ' || 'F8073010;'  FROM dba_tab_privs WHERE GRANTEE = 'F8075426';
        
        SELECT 'GRANT SELECT, UPDATE ON '||owner||'.'||table_name||' to ' || 'F8075426;'  FROM all_tables WHERE owner = 'NTW_MABE';
        
        REVOKE
        
        SELECT 'REVOKE UPDATE ON '||owner||'.'||table_name||' from ' || 'MSAUTO;'  FROM all_tables WHERE owner = 'AUDITORIAAPPUSER';
        
        SELECT 'GRANT SELECT ON ' || owner || '.' || view_name || ' TO GRAFANA;' FROM all_views WHERE owner IN ('AUX_TABLES', 'ICT_TABLES');
        
        select 'grant select on '|| OWNER || '.' ||view_name || ' to ' || '<USERNAME>;' from dba_views;
        
        SELECT 'GRANT select ON ' || owner || '.' || view_name || ' TO RAN_SHARING_VIEW_READ;'
        FROM all_views 
        WHERE owner = 'RAN_SHARING';
        ```
        
    - Bloco PL/SQL para conceder privilégios
        
        ```sql
        CREATE ROLE ACC_REPORTER;
        
        -- Concede o privilégio
        BEGIN
           FOR r IN (SELECT table_name FROM dba_tables WHERE owner = 'REPORTER')
           LOOP
              EXECUTE IMMEDIATE 'GRANT SELECT, INSERT, UPDATE, DELETE ON REPORTER.' || r.table_name || ' TO ACC_REPORTER';
           END LOOP;
        END;
        /
        
        -- Associa a role ao usuário
        GRANT ACC_REPORTER TO "accenture";
        
        SELECT grantee, privilege, owner, table_name
          FROM dba_tab_privs
         WHERE grantee = 'ACC_REPORTER';
        
        SELECT granted_role, admin_option, default_role
          FROM dba_role_privs
         WHERE grantee = 'accenture';
        ```
        
    - Verificar grants de um usuário
        
        ```sql
        SELECT * FROM DBA_TAB_PRIVS WHERE GRANTEE = 'nome_do_usuario';
        
        Consulta grants de usuário para uma tabela específica
        
        SELECT * 
        FROM DBA_TAB_PRIVS 
        WHERE GRANTEE = 'nome_do_usuario' 
        AND TABLE_NAME = 'nome_da_tabela';
        
        Consulta mostrará as concessões de privilégios para a tabela específica
        
        SELECT * 
        FROM ALL_TAB_PRIVS 
        WHERE GRANTEE = 'nome_do_usuario' 
        AND TABLE_NAME = 'nome_da_tabela';
        ```
        
    - Verificar se o GRANT foi aplicado corretamente
        
        ```sql
        SELECT OWNER,
               TABLE_NAME,
               GRANTEE,
               PRIVILEGE
        FROM DBA_TAB_PRIVS
        WHERE TABLE_NAME = 'TB_LOOKUP_USER_INFO'
          AND OWNER = 'AUX_TABLES'
          AND GRANTEE = 'ICT_TABLES'
          AND PRIVILEGE = 'INSERT';
        ```
        
    - Verificar todos os grants
        
        ```sql
        select 
          owner, 
          table_name, 
          grantee, 
          grantor, 
          max(
            case when privilege = 'SELECT' then 'sel' end
          ) sel, 
          max(
            case when privilege = 'INSERT' then 'ins' end
          ) ins, 
          max(
            case when privilege = 'UPDATE' then 'upd' end
          ) upd, 
          max(
            case when privilege = 'DELETE' then 'del' end
          ) del, 
          max(
            case when privilege = 'ALTER' then 'alt' end
          ) alt, 
          max(
            case when privilege = 'READ' then 'rea' end
          ) rea, 
          max(
            case when privilege = 'QUERY REWRITE' then 'qrr' end
          ) qrr, 
          max(
            case when privilege = 'DEBUG' then 'dbg' end
          ) dbg, 
          max(
            case when privilege = 'ON COMMIT REFRESH' then 'ocr' end
          ) ocr, 
          max(
            case when privilege = 'FLASHBACK' then 'flb' end
          ) flb 
        from 
          dba_tab_privs 
        where 
          owner not in (
            'SYS', 'SYSTEM', 'APPQOSSYS', 'DBSNMP', 
            'GSMADMIN_INTERNAL', 'XDB', 'WMSYS'
          ) 
        group by 
          owner, 
          table_name, 
          grantee, 
          grantor 
        order by 
          owner, 
          table_name, 
          grantee, 
          grantor;
        
        ```
        
    - Verificar privilégios de sistema
        
        ```sql
        SELECT privilege
        FROM dba_sys_privs
        WHERE grantee = 'MSTG_STS_PRD';
        ```
        
    - Verificar privilégios de objeto
        
        ```sql
        SELECT privilege, table_name
        FROM dba_tab_privs
        WHERE grantee = 'NOME_DO_USUÁRIO';
        ```
        
    - Verificar privilégios de objeto e sistema
        
        ```sql
        SELECT privilege, admin_option, granted_role
        FROM dba_role_privs
        WHERE grantee = 'NOME_DO_USUÁRIO';
        ```
        
    - Verificar privilégios de objeto e sistema c**oncedidos por papéis hierárquicos**
        
        ```sql
        SELECT granted_role, admin_option
        FROM dba_role_privs
        WHERE grantee = 'NOME_DO_USUÁRIO'
        AND grantee IN (SELECT granted_role
                        FROM dba_role_privs
                        WHERE grantee = 'NOME_DO_USUÁRIO');
        ```
        
- INSTÂNCIAS
    - Verificar quais instâncias estão on-line
        
        ```sql
        ps -ef | grep smon
        ```
        
        ![Untitled](ICARO%20SCRIPT/Untitled.png)
        
    - Verificar se a instância está open
        
        ```sql
        sqlplus / as sysdba
        
        SELECT INSTANCE_NAME, HOST_NAME, STATUS FROM V$INSTANCE
        ```
        
- RMAN
    - Verificar sessões bloqueadoras (RMAN)
        
        ```sql
        SELECT p.SPID, s.EVENT
            ,s.SECONDS_IN_WAIT AS SEC_WAIT
            ,s.STATE
            ,s.CLIENT_INFO
        FROM V$SESSION_WAIT sw
            , V$SESSION s
            , V$PROCESS p
        WHERE s.sid=8 and s.SID=sw.SID
              AND s.PADDR=p.ADDR;
        
        SELECT s.sid, username , program, module, action, logon_time, l.*   FROM v$session s, v$enqueue_lock l   WHERE l.sid = s.sid and l.type = 'CF' AND l.id1 = 0 and l.id2 = 2;
        
        select 'ALTER SYSTEM DISCONNECT SESSION '''||s.sid||','||s.serial#||',' || '@'|| s.INST_ID || ''' IMMEDIATE;'
        FROM gv$session s
        inner join gv$process p
        on s.paddr = p.addr
        WHERE s.sid = 8;
        ```
        
    - Matar sessões bloqueadoras (RMAN)
        
        ```sql
        ps -ef | grep -i rman
        
        kill -9 15466 18105 20754 22077 23371 24596 25989 28595 30994 31207
        ```
        
    - **Excluir logs de arquivo expirado (RMAN)**
        
        ```sql
        RMAN> crosscheck archivelog all; 
        
        RMAN> delete noprompt expired archivelog all;
        ```
        
    - Comandos RMAN
        
        ```sql
        RMAN> show all;
        
        Os parâmetros de configuração RMAN para banco de dados com db_unique_name ELPISYS são:
        
        CONFIGURE RETENTION POLICY TO REDUNDANCY 1; # default
        CONFIGURE BACKUP OPTIMIZATION OFF; # default
        CONFIGURE DEFAULT DEVICE TYPE TO DISK; # default
        CONFIGURE CONTROLFILE AUTOBACKUP ON;
        CONFIGURE CONTROLFILE AUTOBACKUP FORMAT FOR DEVICE TYPE DISK TO '%F'; # default
        CONFIGURE DEVICE TYPE DISK PARALLELISM 1 BACKUP TYPE TO BACKUPSET; # default
        CONFIGURE DATAFILE BACKUP COPIES FOR DEVICE TYPE DISK TO 1; # default
        CONFIGURE ARCHIVELOG BACKUP COPIES FOR DEVICE TYPE DISK TO 1; # default
        CONFIGURE MAXSETSIZE TO UNLIMITED; # default
        CONFIGURE ENCRYPTION FOR DATABASE OFF; # default
        CONFIGURE ENCRYPTION ALGORITHM 'AES128'; # default
        CONFIGURE COMPRESSION ALGORITHM 'BASIC' AS OF RELEASE 'DEFAULT' OPTIMIZE FOR LOAD TRUE ; # default
        CONFIGURE RMAN OUTPUT TO KEEP FOR 7 DAYS; # default
        CONFIGURE ARCHIVELOG DELETION POLICY TO NONE; # default
        CONFIGURE SNAPSHOT CONTROLFILE NAME TO '/d01/oracle/product/v.19.13.0.0/dbs/snapcf_ELPISYS.f'; # default
        ```
        
    - **Backup RMAN usando SCN**
        
        ```sql
        connect target /
        
        BACKUP DEVICE TYPE DISK INCREMENTAL FROM SCN 1828103625 DATABASE FORMAT '/RMAN/backup/bkup_%U';
        ```
        
    - **Excluir logs de arquivo mantendo “n” dias de logs de arquivo (RMAN)**
        
        ```sql
        run{ 
        allocate channel C1 type disk; 
        
        delete noprompt archivelog until time 'SYSDATE-1'; (n should be replaced by an integer, for which days you want to retain the archive logs)
        release channel C1;
        }
        ```
        
    - V**erificar detalhes do JOB RMAN**
        
        ```sql
        set line 200 
        set pages 2000
        col start_time for a25 
        col end_time for a25 
        col status for a30
         
        select SESSION_KEY, INPUT_TYPE, STATUS, to_char(START_TIME,'mm/dd/yy hh24:mi') start_time, to_char(END_TIME,'mm/dd/yy hh24:mi') end_time,elapsed_seconds/3600 hrs from V$RMAN_BACKUP_JOB_DETAILS order by session_key;
        ```
        
    - F**azer backup dos logs de arquivo entre sequência específica**
        
        ```sql
        RMAN> BACKUP ARCHIVELOG FROM SEQUENCE 288 UNTIL SEQUENCE 388 DELETE INPUT;
        ```
        
    - **Resolvendo erro RMAN - 08591: Warning invalid archived log retention policy**
        
        ```sql
        Logar no RMAN
        	rman target /
        
        Rodar comando: crosscheck archivelog all;
        
        Após crosscheck, rodar comando: delete FORCE noprompt archivelog all completed before 'SYSDATE-2';
        ```
        
- PGA
    - Verificar estatísticas PGA
        
        ```sql
        select * from v$pga_target_advice;
        
        select * from v$pgastat;
        
        show parameter pga
        ```
        
    - Verificar todas as sessões (PGA)
        
        ```sql
        select name, value
        from v$statname n, v$sesstat t
        where n.statistic# = t.statistic#
        and t.sid = ( select sid from v$mystat where rownum = 1 )
        and n.name in ( 'session pga memory', 'session pga memory max','session uga memory', 'session uga memory max')
        ```
        
    - Verificar PGA e SGA estimado
        
        ```sql
        ---- Verificar PGA e SGA estimado
        set lines 200 pages 200
        col  estd_physical_reads for 99999999999999999999999
        SELECT sga_size, sga_size_factor, estd_db_time, estd_physical_reads FROM   V$SGA_TARGET_ADVICE;
        
        set lines 200 pages 200
        col  estd_physical_reads for 99999999999999999999999
        col  estd_extra_bytes_rw for 99999999999999999999999
        SELECT pga_target_for_estimate/1024/1024 "PGA Target Estimado", pga_target_factor, estd_time, estd_extra_bytes_rw
        FROM   V$PGA_TARGET_ADVICE;
        ```
        
    - Verificar **o uso da PGA por todas as sessões**
        
        ```sql
        select s.osuser osuser,s.serial# serial,se.sid,n.name,
        max(se.value) maxmem
        from v$sesstat se,
        v$statname n
        ,v$session s
        where n.statistic# = se.statistic#
        and n.name in ('session pga memory','session pga memory max',
        'session uga memory','session uga memory max')
        and s.sid=se.sid
        group by n.name,se.sid,s.osuser,s.serial#
        order by 2
        ;
        ```
        
    - Verificar **o uso da PGA**
        
        ```sql
        SET LINESIZE 200
        SET PAGESIZE 100
        COLUMN SID           FORMAT 99999       HEADING "SID"
        COLUMN SERIAL#       FORMAT 99999       HEADING "SERIAL#"
        COLUMN USUARIO       FORMAT A12         HEADING "USUARIO"
        COLUMN PROGRAMA      FORMAT A25         HEADING "PROGRAMA"
        COLUMN PID_SO        FORMAT A8          HEADING "PID_SO"
        COLUMN PGA_USADA_MB  FORMAT 9990.99     HEADING "PGA_MB"
        COLUMN SQL_ID        FORMAT A13         HEADING "SQL_ID"
        COLUMN SQL_TEXT_RESUMIDA FORMAT A80     HEADING "SQL_TEXT_RESUMIDA"
        SELECT
            s.sid                               AS SID,
            s.serial#                           AS SERIAL#,
            s.username                          AS USUARIO,
            s.program                           AS PROGRAMA,
            p.spid                              AS PID_SO,
            ROUND(st.value / 1024 / 1024, 2)    AS PGA_USADA_MB,
            sql.sql_id                          AS SQL_ID,
            SUBSTR(sql.sql_text, 1, 80)         AS SQL_TEXT_RESUMIDA
        FROM
            v$session s
        JOIN
            v$process p ON s.paddr = p.addr
        JOIN
            v$sesstat st ON s.sid = st.sid
        JOIN
            v$statname sn ON st.statistic# = sn.statistic#
        LEFT JOIN
            v$sql sql ON s.sql_id = sql.sql_id
        WHERE
            sn.name = 'session pga memory'
            AND s.username IS NOT NULL
        ORDER BY
            PGA_USADA_MB DESC
        FETCH FIRST 10 ROWS ONLY;
        
        ```
        
    - Aumentar PGA sem reinicializar o banco de dados
        
        ```sql
        ALTER SYSTEM SET pga_aggregate_target = 3500M SCOPE=BOTH;
        
        Nota: é bom usar o valor 3 conforme definido em pga_Aggregate_target
        
        alter system set pga_aggregate_limit=pga_aggregate_target*3 Scope=both;
        ```
        
    - Verificar utilização de recursos e quantidade de memória necessária para PGA
        
        ```sql
        select resource_name, current_utilization, max_utilization, limit_value
        from v$resource_limit
        where resource_name in ('sessions', 'processes');
        
        ou
        
        SELECT MAx_utilization_session*(2048576+P1.VALUE+P2.VALUE)/(1024*1024) YOU_NEED_PGA_MB
        FROM V$PARAMETER P1, V$PARAMETER P2
        WHERE P1.NAME = 'sort_area_size'
        AND P2.NAME = 'hash_area_size';
        
        Obs.: a utilização máxima é obtida do número de consulta acima da sessão
        ```
        
    - **Verificar alocação de PGA para cada processo**
        
        ```sql
        SELECT spid, program,
        pga_max_mem max,
        pga_alloc_mem alloc,
        pga_used_mem used,
        pga_freeable_mem free
        FROM V$PROCESS;
        ```
        
    - **PGA usou memória para todos os processos**
        
        ```sql
        SELECT ROUND(SUM(pga_used_mem)/(1024*1024),2) PGA_USED_MB FROM v$process;
        ```
        
    - **PGA usou memória máxima para todos os processos**
        
        ```sql
        select ROUND(SUM(pga_max_mem)/(1024*1024),2) PGA_USED_MB FROM v$process;
        ```
        
    - T**otal alocação de memória por processo**
        
        ```sql
        SELECT ROUND(SUM(pga_used_mem)/(1024*1024),2) max,
        ROUND(SUM(pga_alloc_mem)/(1024*1024),2) alloc,
        ROUND(SUM(pga_used_mem)/(1024*1024),2) used,
        ROUND(SUM(pga_freeable_mem)/(1024*1024),2) free
        FROM V$PROCESS;
        ```
        
- RECOVERY FILE DEST
    - Query recovery file dest
        
        ```sql
        select (SELECT sys_context ('userenv','DB_NAME') DB_NAME FROM DUAL) DB_NAME,name as "NAME", round(space_limit/1024/1024/1024,2) MaxGB , round(space_used/1024/1024/1024,2) UsedGB, round((space_used/space_limit)*100,2) "USED %", number_of_files 
        from v$recovery_file_dest
        ```
        
- LOGS
    - **Restaurar logs de arquivo da sequência**
        
        ```sql
        run
        {
        ALLOCATE CHANNEL c1 DEVICE TYPE SBT_TAPE PARMS 'ENV=(NB_ORA_POLICY=,NB_ORA_CLIENT=),BLKSIZE=524288'; 
        SET ARCHIVELOG DESTINATION TO '/RMAN/arch';
        restore archivelog from logseq=196735 until logseq=196749 thread=1;
        }
        
        Observação: NB_ORA_POLICY e NB_ORA_CLIENT são detalhes específicos do ambiente que você pode obter nos arquivos de configuração.
        ```
        
- RECYCLEBIN
    - Verificar quais os objetos do usuário estão na lixeira
        
        ```sql
        SELECT * FROM RECYCLEBIN;
        
        ou SQL*PLUS
        
        show recyclebin
        ```
        
    - Estrutura da lixeira
        
        ```sql
        desc RECYCLEBIN;
        ```
        
    - Limpeza da lixeira
        
        ```sql
        PURGE TABLE "BIN$yrMKlZaVMhfgNAgAIMenRA==$0";
        PURGE TABLE employees;
        PURGE INDEX "BIN$yrQIJMkVDlopOAlAIMenRA==$1";
        PURGE INDEX idx_emp;
        
        Um usuário também pode limpar toda a sua lixeira através do comando:
        
        PURGE RECYCLEBIN;
        ```
        
    - Recuperando objetos da lixeira
        
        ```sql
        FLASHBACK TABLE employees TO BEFORE DROP;
        
        -> A cláusula RENAME TO permite que a tabela receba um novo nome na recuperação. (opcional)
        
        FLASHBACK TABLE employees TO BEFORE DROP RENAME TO old_emp;
        
        -> Caso exista mais de um objeto com o mesmo nome original, o excluído mais recentemente será recuperado. 
        Porém se a intenção do usuário for recuperar uma determinada versão do objeto que foi excluído que não seja
        a mais recente deve ser especificado o novo nome atribuído ao objeto pela lixeira.
        
        FLASHBACK TABLE "BIN$yrMKlZaVMhfgNAgAIMenRA==$0"
        TO BEFORE DROP RENAME TO old_old_emp;
        ```
        
- GAP
    - Verificar GAP
        
        ```sql
        SQL>  alter pluggable database TIMAUDDB open;
        
        Pluggable database altered.
        
        --------------------------------------------------------------------------------
        Rodar sempre no standby
        --------------------------------------------------------------------------------
        
        SET LINES 800
        SET PAGESIZE 10000
        BREAK ON REPORT
        COMPUTE SUM LABEL TOTAL OF GAP ON REPORT
        select primary.thread#,
              primary.maxsequence primaryseq,
              standby.maxsequence standbyseq,
              primary.maxsequence - standby.maxsequence gap
        		  from ( select thread#, max(sequence#) maxsequence
              from v$archived_log
              where archived = 'YES'
              and resetlogs_change# = ( select d.resetlogs_change# from v$database d )
        		  group by thread# order by thread# ) primary,
              ( select thread#, max(sequence#) maxsequence
              from v$archived_log
              where applied = 'YES'
              and resetlogs_change# = ( select d.resetlogs_change# from v$database d )
              group by thread# order by thread# ) standby
        where primary.thread# = standby.thread#; 
        
         
         
        SQL> ALTER DATABASE RECOVER MANAGED STANDBY DATABASE CANCEL;
        
        Database altered.
        
        SQL> ALTER DATABASE RECOVER MANAGED STANDBY DATABASE DISCONNECT FROM SESSION;
        
        Database altered.
        
        Explicando a Diferença nos Valores de Sequence
        Threads Diferentes (THREAD#): Cada instância do Oracle RAC possui sua própria thread de redo log. As sequências de redo logs (números de sequência) são geradas de forma independente para cada thread. Isso significa que uma instância pode estar mais avançada no processamento de transações em comparação com outra.
        Independência de Instâncias: As instâncias do RAC podem estar em níveis diferentes de atividade de transação, o que resulta em diferentes números de sequência para os redo logs. Uma instância com mais atividade de escrita de dados avançará mais rapidamente no número de sequência do redo log do que uma instância com menos atividade.
        Sincronização e Replicação de Logs: Em um ambiente com replicação de dados, como em uma configuração de Data Guard com standby, o número de sequência também reflete o ponto até o qual os redo logs foram aplicados na instância standby. Os números de sequência PRIMARYSEQ e STANDBYSEQ indicam o progresso do envio e aplicação dos redo logs.
        Gaps nos Números de Sequência: Os gaps podem ocorrer devido a várias razões, como falhas de comunicação, downtime de uma instância, ou atrasos na aplicação dos logs na instância standby. No exemplo, o GAP é 0, indicando que não há diferença entre os números de sequência aplicados nas instâncias primária e standby.
        Distribuição de Carga: Em um ambiente RAC, a carga de trabalho é distribuída entre as instâncias, o que pode resultar em diferentes níveis de avanço nos redo logs para cada instância.
        Conclusão
        A diferença nos valores das sequências nas threads de um banco de dados Oracle RAC é normal e esperada devido à natureza distribuída do ambiente. Cada instância do RAC opera com sua própria thread de redo log, gerando números de sequência de forma independente, de acordo com a atividade de transações que processa. A sincronização com instâncias standby (como em Data Guard) e outros fatores, como falhas ou atrasos na comunicação, também podem influenciar os números de sequência observados. 
        ```
        
    - Verificar SEQUENCE
        
        ```sql
        select max(SEQUENCE#)  from   v$archived_log where applied='YES';
        
        ///
        
        SELECT ARCH.THREAD# "Thread", ARCH.SEQUENCE# "Last Sequence Received", APPL.SEQUENCE# "Last Sequence Applied", (ARCH.SEQUENCE# - APPL.SEQUENCE#) "Difference" 
        FROM (SELECT THREAD# ,SEQUENCE# FROM V$ARCHIVED_LOG WHERE (THREAD#,FIRST_TIME ) IN (SELECT THREAD#,MAX(FIRST_TIME) FROM V$ARCHIVED_LOG GROUP BY THREAD#)) ARCH,(SELECT THREAD# ,SEQUENCE# FROM V$LOG_HISTORY WHERE (THREAD#,FIRST_TIME ) IN (SELECT THREAD#,MAX(FIRST_TIME) FROM V$LOG_HISTORY GROUP BY THREAD#)) APPL 
        WHERE ARCH.THREAD# = APPL.THREAD# ORDER BY 1;
        
        ///
        
        Rodar no primário
        
        SELECT  CASE when ARCH.NAME  like '+FRA%' then (select instance_name from v$instance) when ARCH.NAME  like '/u%' then (select instance_name from v$instance) ELSE ARCH.NAME END NOME, ARCH.THREAD# "Thread", ARCH.SEQUENCE# "Ultima sequencia recebida", APPL.SEQUENCE# "Ultima sequencia aplicada", (ARCH.SEQUENCE# - APPL.SEQUENCE#) "Diferenca" 
        FROM (SELECT NAME, THREAD# ,SEQUENCE# FROM V$ARCHIVED_LOG WHERE (THREAD#,FIRST_TIME ) IN (SELECT THREAD#,MAX(FIRST_TIME) FROM V$ARCHIVED_LOG GROUP BY THREAD#)) ARCH,(SELECT THREAD# ,SEQUENCE# FROM V$LOG_HISTORY WHERE (THREAD#,FIRST_TIME ) IN (SELECT THREAD#,MAX(FIRST_TIME) FROM V$LOG_HISTORY GROUP BY THREAD#)) APPL 
        WHERE ARCH.THREAD# = APPL.THREAD# ORDER BY 1;
        ```
        
    - Verificar aplicação de SEQUENCE
        
        ```sql
        SELECT process, status, sequence# FROM v$managed_standby;
        
        select max(SEQUENCE#)  from   v$archived_log where applied='YES';
        
        ///
        
        SELECT ARCH.THREAD# "Thread", ARCH.SEQUENCE# "Last Sequence Received", APPL.SEQUENCE# "Last Sequence Applied", (ARCH.SEQUENCE# - APPL.SEQUENCE#) "Difference" 
        FROM (SELECT THREAD# ,SEQUENCE# FROM V$ARCHIVED_LOG WHERE (THREAD#,FIRST_TIME ) IN (SELECT THREAD#,MAX(FIRST_TIME) FROM V$ARCHIVED_LOG GROUP BY THREAD#)) ARCH,(SELECT THREAD# ,SEQUENCE# FROM V$LOG_HISTORY WHERE (THREAD#,FIRST_TIME ) IN (SELECT THREAD#,MAX(FIRST_TIME) FROM V$LOG_HISTORY GROUP BY THREAD#)) APPL 
        WHERE ARCH.THREAD# = APPL.THREAD# ORDER BY 1;
        
        ///
        
        Rodar no primário
        
        SELECT  CASE when ARCH.NAME  like '+FRA%' then (select instance_name from v$instance) when ARCH.NAME  like '/u%' then (select instance_name from v$instance) ELSE ARCH.NAME END NOME, ARCH.THREAD# "Thread", ARCH.SEQUENCE# "Ultima sequencia recebida", APPL.SEQUENCE# "Ultima sequencia aplicada", (ARCH.SEQUENCE# - APPL.SEQUENCE#) "Diferenca" 
        FROM (SELECT NAME, THREAD# ,SEQUENCE# FROM V$ARCHIVED_LOG WHERE (THREAD#,FIRST_TIME ) IN (SELECT THREAD#,MAX(FIRST_TIME) FROM V$ARCHIVED_LOG GROUP BY THREAD#)) ARCH,(SELECT THREAD# ,SEQUENCE# FROM V$LOG_HISTORY WHERE (THREAD#,FIRST_TIME ) IN (SELECT THREAD#,MAX(FIRST_TIME) FROM V$LOG_HISTORY GROUP BY THREAD#)) APPL 
        WHERE ARCH.THREAD# = APPL.THREAD# ORDER BY 1;
        ```
        
    - Verificar MRP
        
        ```sql
        SELECT INST_ID, PROCESS, STATUS, THREAD#, SEQUENCE#, BLOCK#, BLOCKS,
               decode(nvl(BLOCKS,0), 0, 0, round((BLOCK#/BLOCKS)*100,2)) PERC, systimestamp dthora
        FROM GV$MANAGED_STANDBY
        WHERE PROCESS like 'MRP%';
        
        ```