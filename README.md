This guide walks you through setting up Clickhouse cluster with Docker Compose with 3 nodes, 3 Clickhouse-servers and 3 of them as Clickhouse-keeper at the same time.
We setup (3 nodes, 1 shard, 3 replicas) is simpler than the original 2-shard article and is a very common and solid HA configuration.

## Node Mapping
| Hostname | IP | Role | Macros (shard/replica) | Keeper ID |
| --- | --- | --- | --- | --- |
| control01 | 172.16.1.101 | Replica 1 | "shard=1\|replica=1" | 1 |
| control02 | 172.16.1.102 | Replica 2 | "shard=1\|replica=2" | 2 |
| control03 | 172.16.1.103 | Replica 3 | "shard=1\|replica=3" | 3 |

## 1. Prepare Directories (on all 3 nodes)
```bash
sudo mkdir -p /opt/clickhouse/config.d \
              /opt/clickhouse/users.d \
              /opt/clickhouse/clickhouse-data \
              /var/log/clickhouse-server \
              /var/log/clickhouse-keeper
```
## 2. Set Default User Password (on all nodes)
remember to change password

```bash
PASSWORD='YourStrongPasswordHere123!'
echo -n "$PASSWORD" | sha256sum | tr -d '-'
```
Copy the hash (e.g. `69ca9615...`) and replace it below by `PUT_YOUR_HASH_HERE`  
Create `/opt/clickhouse/users.xml` on all nodes:
```xml
<clickhouse>
    <profiles>
        <default>
            <max_memory_usage>16000000000</max_memory_usage>
            <max_distributed_depth>4000</max_distributed_depth>
            <distributed_connections_pool_size>4096</distributed_connections_pool_size>
            <max_distributed_connections>4096</max_distributed_connections>
        </default>
    </profiles>

    <users>
        <default>
            <password_sha256_hex>PUT_YOUR_HASH_HERE</password_sha256_hex>
            <networks>
                <ip>::/0</ip>
            </networks>
            <profile>default</profile>
            <quota>default</quota>
            <access_management>1</access_management>
        </default>
    </users>

    <quotas>
        <default>
            <interval>
                <duration>3600</duration>
                <queries>0</queries>
                <errors>0</errors>
                <result_rows>0</result_rows>
                <read_rows>0</read_rows>
                <execution_time>0</execution_time>
            </interval>
        </default>
    </quotas>
</clickhouse>
```
## 3. Main Cluster Config (remote_servers.xml)
Create `/opt/clickhouse/config.d/remote_servers.xml` on all nodes with the following content (only the `<macros>` section changes per node):
```xml
<clickhouse>
    <listen_host>0.0.0.0</listen_host>

    <remote_servers>
        <cluster_1S_3R>
            <shard>
                <internal_replication>True</internal_replication>
                <replica>
                    <host>control01</host>
                    <port>9000</port>
                    <user>default</user>
                    <password>YourStrongPasswordHere123!</password>
                </replica>
                <replica>
                    <host>control02</host>
                    <port>9000</port>
                    <user>default</user>
                    <password>YourStrongPasswordHere123!</password>
                </replica>
                <replica>
                    <host>control03</host>
                    <port>9000</port>
                    <user>default</user>
                    <password>YourStrongPasswordHere123!</password>
                </replica>
            </shard>
        </cluster_1S_3R>
    </remote_servers>

    <!-- Change these per node -->
    <macros>
        <shard>1</shard>
        <replica>1</replica>   <!-- 1 on control01, 2 on control02, 3 on control03 -->
    </macros>

    <zookeeper>
        <node>
            <host>control01</host>
            <port>9181</port>
        </node>
        <node>
            <host>control02</host>
            <port>9181</port>
        </node>
        <node>
            <host>control03</host>
            <port>9181</port>
        </node>
    </zookeeper>
</clickhouse>
```
> [!IMPORTANT]
> Important: Update the `<replica>` number in `<macros>` for each node.

## 4. ClickHouse Keeper Config
Create `/opt/clickhouse/config.d/keeper.xml` on all 3 nodes (only s`erver_id` changes):
# On control01:
```xml
<clickhouse>
    <keeper_server>
        <tcp_port>9181</tcp_port>
        <server_id>1</server_id>
        <log_storage_path>/var/lib/clickhouse/coordination/log</log_storage_path>
        <snapshot_storage_path>/var/lib/clickhouse/coordination/snapshots</snapshot_storage_path>

        <coordination_settings>
            <operation_timeout_ms>10000</operation_timeout_ms>
            <session_timeout_ms>30000</session_timeout_ms>
            <raft_logs_level>warning</raft_logs_level>
        </coordination_settings>

        <raft_configuration>
            <server>
                <id>1</id>
                <hostname>control01</hostname>
                <port>9234</port>
            </server>
            <server>
                <id>2</id>
                <hostname>control02</hostname>
                <port>9234</port>
            </server>
            <server>
                <id>3</id>
                <hostname>control03</hostname>
                <port>9234</port>
            </server>
        </raft_configuration>
    </keeper_server>
</clickhouse>
```
On **control02** → change `<server_id>2</server_id>`  
On **control03** → change `<server_id>3</server_id>`  

## 5. `/etc/hosts` (on all 3 nodes)
```bash
sudo tee -a /etc/hosts <<EOF
172.16.1.101 control01
172.16.1.102 control02
172.16.1.103 control03
EOF
```  

## 6. Create docker-compose file (on all 3 nodes)
```bash
cat > /opt/clickhouse/docker-compose.yml << 'EOF'
version: '3.8'

services:
  clickhouse:
    image: clickhouse/clickhouse-server:24.8.12
    container_name: clickhouse
    network_mode: host
    restart: unless-stopped
    
    volumes:
      - /opt/clickhouse/users.xml:/etc/clickhouse-server/users.xml:ro
      - /opt/clickhouse/config.d:/etc/clickhouse-server/config.d:ro
      - /opt/clickhouse/users.d:/etc/clickhouse-server/users.d:ro
      - /opt/clickhouse/clickhouse-data:/var/lib/clickhouse
      - /var/log/clickhouse-server:/var/log/clickhouse-server
      - /var/log/clickhouse-keeper:/var/log/clickhouse-keeper
    
    ulimits:
      nofile:
        soft: 262144
        hard: 262144

    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
EOF
```
## 7. tart the service (on all nodes):
```bash
cd /opt/clickhouse
docker compose up -d
```

## 8. Check the keeper and Clickhouse and database replication 
First Check the keeper and there shoud be a 2 follower and 1 leader  
run on all node:
```bash
echo srvr | nc 127.0.0.1 9181
```
<img width="1066" height="946" alt="Screenshot 2026-07-09 at 1 55 42 PM" src="https://github.com/user-attachments/assets/b1246496-999c-4b3e-8c09-dccdfb9a6dd2" />


Login into one of node to run commands  
```bash
docker exec -it clickhouse clickhouse-client --user default --password 'YourStrongPasswordHere123!'
```

```bash
SELECT * FROM system.clusters WHERE cluster = 'cluster_1S_3R';
```
```bash
SELECT *
FROM system.clusters
WHERE cluster = 'cluster_1S_3R'

Query id: 8177938d-7098-48e4-a81e-62196eee43be

   ┌─cluster───────┬─shard_num─┬─shard_weight─┬─internal_replication─┬─replica_num─┬─host_name─┬─host_address─┬─port─┬─is_local─┬─user────┬─default_database─┬─errors_count─┬─slowdowns_count─┬─estimated_recovery_time─┬─database_shard_name─┬─database_replica_name─┬─is_active─┬─replication_lag─┬─recovery_time─┐
1. │ cluster_1S_3R │         1 │            1 │                    1 │           1 │ control01 │ 172.16.1.101 │ 9000 │        1 │ default │                  │            0 │               0 │                       0 │                     │                       │      ᴺᵁᴸᴸ │            ᴺᵁᴸᴸ │          ᴺᵁᴸᴸ │
2. │ cluster_1S_3R │         1 │            1 │                    1 │           2 │ control02 │ 172.16.1.102 │ 9000 │        0 │ default │                  │            0 │               0 │                       0 │                     │                       │      ᴺᵁᴸᴸ │            ᴺᵁᴸᴸ │          ᴺᵁᴸᴸ │
3. │ cluster_1S_3R │         1 │            1 │                    1 │           3 │ control03 │ 172.16.1.103 │ 9000 │        0 │ default │                  │            0 │               0 │                       0 │                     │                       │      ᴺᵁᴸᴸ │            ᴺᵁᴸᴸ │          ᴺᵁᴸᴸ │
   └───────────────┴───────────┴──────────────┴──────────────────────┴─────────────┴───────────┴──────────────┴──────┴──────────┴─────────┴──────────────────┴──────────────┴─────────────────┴─────────────────────────┴─────────────────────┴───────────────────────┴───────────┴─────────────────┴───────────────┘

3 rows in set. Elapsed: 0.006 sec.
```

control01 :) SELECT 
    cluster,
    shard_num,
    replica_num,
    host_name,
    port,
    is_active 
FROM system.cluster 
WHERE cluster = 'cluster_1S_3R';

SELECT
    cluster,
    shard_num,
    replica_num,
    host_name,
    port,
    is_active
FROM system.cluster
WHERE cluster = 'cluster_1S_3R'

Query id: f936f298-b692-4cdb-a3d4-aefb4b5c52be


Elapsed: 0.003 sec. 

Received exception from server (version 24.8.12):
Code: 60. DB::Exception: Received from localhost:9000. DB::Exception: Unknown table expression identifier 'system.cluster' in scope SELECT cluster, shard_num, replica_num, host_name, port, is_active FROM system.cluster WHERE cluster = 'cluster_1S_3R'. (UNKNOWN_TABLE)

control01 :) SELECT
    hostName() AS node,
    count() AS row_count,
    min(created) AS oldest,
    max(created) AS newest
FROM testdb.sometable;
 
SELECT
    hostName() AS node,
    count() AS row_count,
    min(created) AS oldest,
    max(created) AS newest
FROM testdb.sometable
 
Query id: 102cb285-0405-44e1-af47-7b0c329b67e5
 
   ┌─node──────┬─row_count─┬──────────────oldest─┬──────────────newest─┐
1. │ control01 │         0 │ 1970-01-01 00:00:00 │ 1970-01-01 00:00:00 │
   └───────────┴───────────┴─────────────────────┴─────────────────────┘
 
1 row in set. Elapsed: 0.027 sec. 
 
control01 :) SELECT 
    database,
    table,
    is_leader,
    absolute_delay,
    replica_is_active
FROM system.replicas
ORDER BY table;
 
SELECT
    database,
    `table`,
    is_leader,
    absolute_delay,
    replica_is_active
FROM system.replicas
ORDER BY `table` ASC
 
Query id: a39bf23f-ba8f-4cc7-afdf-930e11f037f2
 
   ┌─database─┬─table─────┬─is_leader─┬─absolute_delay─┬─replica_is_active───┐
1. │ testdb   │ sometable │         1 │              0 │ {'1':1,'2':1,'3':1} │
   └──────────┴───────────┴───────────┴────────────────┴─────────────────────┘
 
1 row in set. Elapsed: 0.005 sec.
```
