# 指标的采集方式

本节介绍指标的常见采集方式。如果你碰到指标采集不到的问题，需要从源头开始排查，了解各类指标的采集方式就至关重要了。

## 操作系统指标

以 Linux 为例，操作系统层面的大部分指标，都是通过读取 `/proc` 文件系统来采集的。`/proc` 文件系统是 Linux 内核提供的一个虚拟文件系统，包含了大量关于系统运行状态的信息。

比如大家最为熟悉的内存相关的指标，是读取自 `/proc/meminfo` 文件：

```bash
$ cat /proc/meminfo
MemTotal:        7675216 kB
MemFree:          213560 kB
MemAvailable:    2022384 kB
Buffers:          808268 kB
Cached:          1138168 kB
SwapCached:            0 kB
Active:          1191004 kB
Inactive:        5692612 kB
Active(anon):       4852 kB
Inactive(anon):  5011608 kB
Active(file):    1186152 kB
Inactive(file):   681004 kB
... (more) ...
```

有时只靠 `/proc` 的内容是不够的，还需要做系统调用来获取数据，比如磁盘利用率相关的指标，是通过 `statfs` 系统调用来获取的。

## 存储系统指标

各类存储系统，比如 MySQL、PostgreSQL、Redis、MongoDB、Memcached 等，通常会通过查询语句来获取指标数据。比如 MySQL 提供了 `SHOW STATUS` 命令，可以获取大量的运行状态指标：

```sql
MariaDB [(none)]> SHOW STATUS LIKE '%conn%';
+-----------------------------------------------+---------+
| Variable_name                                 | Value   |
+-----------------------------------------------+---------+
| Aborted_connects                              | 7748054 |
| Connection_errors_accept                      | 0       |
| Connection_errors_internal                    | 0       |
| Connection_errors_max_connections             | 0       |
| Connection_errors_peer_address                | 0       |
| Connection_errors_select                      | 0       |
| Connection_errors_tcpwrap                     | 0       |
| Connections                                   | 8533229 |
| Max_used_connections                          | 130     |
| Performance_schema_session_connect_attrs_lost | 0       |
| Slave_connections                             | 0       |
| Slaves_connected                              | 0       |
| Ssl_client_connects                           | 0       |
| Ssl_connect_renegotiates                      | 0       |
| Ssl_finished_connects                         | 0       |
| Threads_connected                             | 33      |
| wsrep_connected                               | OFF     |
+-----------------------------------------------+---------+
17 rows in set (0.000 sec)
```

类似地，Redis 通过 `INFO` 命令来获取指标数据：

```shell
[root@iZ2ze4oi71k3qgdxwsyn07Z ~]# redis-cli -c info cpu
# CPU
used_cpu_sys:4832.231900
used_cpu_user:7311.381816
used_cpu_sys_children:101.789726
used_cpu_user_children:895.400558
[root@iZ2ze4oi71k3qgdxwsyn07Z ~]# redis-cli -c info memory
# Memory
used_memory:5287416
used_memory_human:5.04M
used_memory_rss:27045888
used_memory_rss_human:25.79M
used_memory_peak:17602640
... (more) ...
```

这些存储系统都是非常成熟的软件，具有良好的可观测性，为了方便用户了解软件运行的情况，自然要暴露各类指标。亲爱的读者朋友，你们自己写的软件，也要注意这一点呀！

## Prometheus 指标

Prometheus 是一个非常流行的开源监控系统，很多现代化的软件都会内置 Prometheus 指标的采集端点，通常是 `/metrics`。采集器会定期访问这个 URL，获取最新的指标数据。访问 `/metrics` 的示例输出如下：

```
# HELP http_requests_total The total number of HTTP requests.
# TYPE http_requests_total counter
http_requests_total{method="post",code="200"} 1027
http_requests_total{method="post",code="400"} 3
```

Kubernetes 生态系统中的大部分组件，比如 kube-apiserver、kube-scheduler、kube-controller-manager、kubelet 等，都会内置 Prometheus 指标端点，方便用户采集指标数据进行监控和告警。

ETCD、MinIO、Nginx vts、新版的 RabbitMQ、HAProxy 等也都可以暴露 Prometheus 协议的指标，可见 Prometheus 协议的影响力之大。如果是自研的业务程序，也尽量通过 Prometheus 协议来暴露指标，方便后续的采集和监控。

## SNMP 指标

SNMP（Simple Network Management Protocol，简单网络管理协议），广泛用于网络设备的管理和监控。各类路由器、交换机、防火墙等，通常会使用 SNMP 协议来采集指标数据。

对于某个设备，会暴露很多监控数据，那就需要一个机制来标识这些数据，数据使用 OID 来标识。比如，获取设备的启动时间：

```
snmpget -v2c -c public <ip_address> 1.3.6.1.2.1.1.3.0
```

其中：

- `-v2c` 指定使用 SNMP v2c 协议
- `-c public` 指定使用的团体名（community string），是一种认证信息
- `<ip_address>` 是设备的 IP 地址
- `1.3.6.1.2.1.1.3.0` 是要查询的 OID

SNMP OID 非常多，有一些是标准化的，所有网络设备都会支持，还有一些是厂商自定义的，不同厂商的设备可能会有不同的 OID 定义。要了解某个设备支持哪些 OID，通常需要参考该设备的 MIB（Management Information Base，管理信息库）文件。

比如设备的 CPU 利用率，就是私有 OID，即不同的品牌设备，表示 CPU 利用率的 OID 不同，对用户来说，梳理这些 OID 是一项巨大的挑战。

## 总结

本节介绍了常见的指标采集方式，涵盖操作系统指标、存储系统指标、Prometheus 指标和 SNMP 指标。可以看出，但凡是一些比较成熟的软件、设备，都会暴露衡量自身健康状况的指标，只是指标暴露方式不同而已。
