Hostname,IP,Role,Macros (shard/replica),Keeper ID
control01|172.16.1.101|Replica 1,"shard=1, replica=1",1
control02|172.16.1.102|Replica 2,"shard=1, replica=2",2
control03|172.16.1.103|Replica 3,"shard=1, replica=3",3

| Environment Variable | Default Value | Description |
| --- | --- | --- |
| `MONITOR_INTERFACE` | `eno1` | The name of the host network interface to attach eBPF filters to (e.g., `eth0`, `bond0`). |
| `MONITOR_SUBNETS` | `172.16.0.0/16,192.168.1.0/24` | Comma-separated list of IPv4 subnets to monitor. Traffic outside these subnets is ignored. |
| `REFRESH_INTERVAL` | `60` | The window interval (in seconds) at which the user-space daemon clears the eBPF map and updates Prometheus. |
| `EXPORTER_BIND_ADDR` | `0.0.0.0` | The host network IP address the Prometheus metric server should bind to. |
| `EXPORTER_PORT` | `8000` | The TCP port the Prometheus metrics engine will listen on. |
