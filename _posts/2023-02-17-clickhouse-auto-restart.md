---
title: clickhouse 禁用自动重启
date: 2023-02-17 13:08:32 +0800
categories: [开发日志]
tags: [clickhouse, 开发日志]
---

clickhouse 的 GC 比较明显，每隔一段时间总是带来 CPU 峰值（目测是几秒钟一次），由于仅在需要的使用它，
不需要后台常驻，所以使用 `clickhouse stop` 关闭。

```shell
$ clickhouse stop
/var/run/clickhouse-server/clickhouse-server.pid file exists and contains pid = 1653591.
The process with pid = 1653591 is running.
Sent terminate signal to process with pid 1653591.
Waiting for server to stop
/var/run/clickhouse-server/clickhouse-server.pid file exists and contains pid = 1653591.
The process with pid = 1653591 is running.
Waiting for server to stop
/var/run/clickhouse-server/clickhouse-server.pid file exists and contains pid = 1653591.
The process with pid = 1653591 is running.
Waiting for server to stop
Now there is no clickhouse-server process.
Server stopped
```

但我发现这个命令不能完全关闭 clickhouse，然后根据 issue [#16254]，但最终用以下方式禁用重启：

[#16254]: https://github.com/ClickHouse/ClickHouse/issues/16254

1. 查看 systemctl 服务，得到加载文件的路径 `/lib/systemd/system/clickhouse-server.service`

```shell
$ systemctl status clickhouse-server.service
Warning: The unit file, source configuration file or drop-ins of clickhouse-server.service changed on disk. Run 'systemctl daemon-reload' to reload units.
● clickhouse-server.service - ClickHouse Server (analytic DBMS for big data)
     Loaded: loaded (/lib/systemd/system/clickhouse-server.service; disabled; vendor preset: enabled)
     Active: active (running) since Fri 2023-02-17 12:59:27 CST; 6min ago
   Main PID: 1686041 (clckhouse-watch)
      Tasks: 231 (limit: 2310)
     Memory: 351.9M
     CGroup: /system.slice/clickhouse-server.service
             ├─1686041 clickhouse-watchdog        --config=/etc/clickhouse-server/config.xml --pid-file=/run/clickhouse-server/clickhouse-server.pid
             └─1686055 /usr/bin/clickhouse-server --config=/etc/clickhouse-server/config.xml --pid-file=/run/clickhouse-server/clickhouse-server.pid

Feb 17 12:59:27 zjp clickhouse-server[1686041]: Processing configuration file '/etc/clickhouse-server/config.xml'.
Feb 17 12:59:27 zjp clickhouse-server[1686041]: Merging configuration file '/etc/clickhouse-server/config.d/default.xml'.
Feb 17 12:59:27 zjp clickhouse-server[1686041]: Logging trace to /var/log/clickhouse-server/clickhouse-server.log
Feb 17 12:59:27 zjp clickhouse-server[1686041]: Logging errors to /var/log/clickhouse-server/clickhouse-server.err.log
Feb 17 12:59:28 zjp clickhouse-server[1686055]: Processing configuration file '/etc/clickhouse-server/config.xml'.
Feb 17 12:59:28 zjp clickhouse-server[1686055]: Merging configuration file '/etc/clickhouse-server/config.d/default.xml'.
Feb 17 12:59:28 zjp clickhouse-server[1686055]: Saved preprocessed configuration to '/var/lib/clickhouse/preprocessed_configs/config.xml'.
Feb 17 12:59:28 zjp clickhouse-server[1686055]: Processing configuration file '/etc/clickhouse-server/users.xml'.
Feb 17 12:59:28 zjp clickhouse-server[1686055]: Merging configuration file '/etc/clickhouse-server/users.d/users.xml'.
Feb 17 12:59:28 zjp clickhouse-server[1686055]: Saved preprocessed configuration to '/var/lib/clickhouse/preprocessed_configs/users.xml'.
```

2. 修改那个文件的 `Restart=always` 部分，可选值有 `no, on-success, on-failure, on-abnormal, on-watchdog, on-abort, or always`，见 [systemd 文档][Restart]

[Restart]: https://www.freedesktop.org/software/systemd/man/systemd.service.html#Restart=

3. 关闭 clickhouse-server，会遇到需要重新加载配置文件的警告：

```console
$ systemctl stop clickhouse-server.service
Warning: The unit file, source configuration file or drop-ins of clickhouse-server.service changed on disk. Run 'systemctl daemon-reload' to reload units.
```

**所以应该先运行 `systemctl daemon-reload`，再运行 `systemctl stop clickhouse-server.service`**

4. 如果启动 clickhouse-server，可使用 `systemctl start clickhouse-server.service`
