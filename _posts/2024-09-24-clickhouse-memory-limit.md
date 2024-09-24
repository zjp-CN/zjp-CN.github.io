---
title: Clickhouse 解决内存受限
date: 2024-09-24 21:00:00 +0800
categories: [ubuntu, 开发日志, clickhouse]
tags: [ubuntu, 开发日志, clickhouse]
---

由于云服务器内存太小，在 Clickhouse 上遇到内存不足：

```bash
Received exception from server (version 24.9.1):
Code: 241. DB::Exception: Received from localhost:9000.
DB::Exception: Memory limit (total) exceeded:
would use 1.37 GiB (attempt to allocate chunk of 8371368 bytes),
current RSS 894.05 MiB, maximum: 1.36 GiB.
OvercommitTracker decision: Memory overcommit isn't used.
Waiting time or overcommit denominator are set to zero..
(MEMORY_LIMIT_EXCEEDED)
```

解决思路：在配置文件中控制内存使用大小和使用方式。

具体做法：

1. 在 `/etc/clickhouse-server/config.xml` 设置一个最大可用内存

```xml
<max_server_memory_usage>8G</max_server_memory_usage>
```

2. 在 `/etc/clickhouse-server/users.xml` 设置如下内容

```xml
<profiles>
    <default>
      <max_bytes_before_external_group_by>4G</max_bytes_before_external_group_by>
      <max_bytes_before_external_sort>4G</max_bytes_before_external_sort>
      <memory_overcommit_ratio_denominator>0</memory_overcommit_ratio_denominator>
      <memory_overcommit_ratio_denominator_for_user>0</memory_overcommit_ratio_denominator_for_user>
      <join_algorithm>partial_merge</join_algorithm>
    </default>
<profiles>
```

具体说明见文档：
* https://clickhouse.com/docs/en/operations/settings/query-complexity#settings-max_bytes_before_external_group_by
* https://clickhouse.com/docs/en/operations/settings/settings

---

此外，更改这些配置似乎需要重启 Clickhouse 才会生效。如果重启遇到失败，检查日志

```bash
less /var/log/clickhouse-server/clickhouse-server.err.log
```

在那里会看到原因。比如某个配置不应该写到 `config.xml`，而应该写到 `users.xml`。
