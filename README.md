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
