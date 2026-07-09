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
```bash
PASSWORD='YourStrongPasswordHere123!'
echo -n "$PASSWORD" | sha256sum | tr -d '-'
```
Copy the hash (e.g. `69ca9615...`) and replace it below by `PUT_YOUR_HASH_HERE`.
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
