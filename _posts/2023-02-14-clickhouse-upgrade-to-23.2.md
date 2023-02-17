---
title: clickhouse 从 22.4 更新到 23.2
date: 2023-02-14 23:51:07 +0800
categories: [开发日志]
tags: [开发日志, clickhouse]
img_path: /assets/img/2023-02-14-clickhouse-upgrade-to-23.2/
---

[clickhouse] 是一个开源的列式数据库，这两天在云服务器上对它进行了更新，记录一些问题和解决方式。

[clickhouse]: https://clickhouse.com/docs/en/intro

# 安装问题

根据文档 [install] 章节，以前的 deb 更新方式已经不行（距离上次更新已经过去十个月了）：

![](clickhouse-deprecated-installation.png)
_官网推荐的迁移方式_

无论按照迁移方式还是新的安装方式，都出现以下 404 问题：

```shell
~ $ apt-get install -y clickhouse-server clickhouse-client
Reading package lists... Done
Building dependency tree
Reading state information... Done
The following additional packages will be installed:
  clickhouse-common-static
Suggested packages:
  clickhouse-common-static-dbg
The following packages will be upgraded:
  clickhouse-client clickhouse-common-static clickhouse-server
3 upgraded, 0 newly installed, 0 to remove and 191 not upgraded.
Need to get 268 MB of archives.
After this operation, 33.8 MB of additional disk space will be used.
Err:1 https://packages.clickhouse.com/deb stable/main amd64 clickhouse-server amd64 22.7.1.2484
  404  Not Found [IP: 172.66.43.7 443]
Err:2 https://packages.clickhouse.com/deb stable/main amd64 clickhouse-client amd64 22.7.1.2484
  404  Not Found [IP: 172.66.43.7 443]
Err:3 https://packages.clickhouse.com/deb stable/main amd64 clickhouse-common-static amd64 22.7.1.2484
  404  Not Found [IP: 172.66.43.7 443]
E: Failed to fetch https://packages.clickhouse.com/deb/pool/stable/clickhouse-server_22.7.1.2484_amd64.deb  404  Not Found [IP: 172.66.43.7 443]
E: Failed to fetch https://packages.clickhouse.com/deb/pool/stable/clickhouse-client_22.7.1.2484_amd64.deb  404  Not Found [IP: 172.66.43.7 443]
E: Failed to fetch https://packages.clickhouse.com/deb/pool/stable/clickhouse-common-static_22.7.1.2484_amd64.deb  404  Not Found [IP: 172.66.43.7 443]
E: Unable to fetch some archives, maybe run apt-get update or try with --fix-missing?
```

[install]: https://clickhouse.com/docs/en/install#install-from-deb-packages

搜了一下 issue，[#46327] 这个目前最新的问题下面推荐了两种安装方式：

[#46327]: https://github.com/ClickHouse/ClickHouse/issues/46327

* `curl https://clickhouse.com/ | sh`：即文档中的 [Self-Managed Install](https://clickhouse.com/docs/en/install#self-managed-install)，
  我用这种方式安装成功。（可能需要 FQ）下载几百 M 的静态二进制文件之后会命令行提示怎么安装（与文档写的不太一样，不过应该都可以装成功）。
  期间会自动链接一系列命令到一个二进制文件：

```shell
total 2.3G
-rw-r--r-- 1 root root 604K Feb 13 22:31 clickhouse
-rwxr-xr-x 1 root root 2.3G Feb 13 22:38 clickhouse.0*
-rw-r--r-- 1 root root  722 Feb 13 20:38 clickhouse.sh

~ $ ll $(which clickhouse-server)
lrwxrwxrwx 1 root root 19 May 13  2022 /usr/bin/clickhouse-server -> /usr/bin/clickhouse*

~ $ ll /usr/bin/clickhouse*
-r-xr-xr-x 1 root              root              2.3G Feb 13 22:39 /usr/bin/clickhouse*
lrwxrwxrwx 1 root              root                19 May 13  2022 /usr/bin/clickhouse-benchmark -> /usr/bin/clickhouse*
lrwxrwxrwx 1 root              root                19 May 13  2022 /usr/bin/clickhouse-client -> /usr/bin/clickhouse*
lrwxrwxrwx 1 root              root                19 May 13  2022 /usr/bin/clickhouse-compressor -> /usr/bin/clickhouse*
lrwxrwxrwx 1 root              root                19 May 13  2022 /usr/bin/clickhouse-copier -> /usr/bin/clickhouse*
lrwxrwxrwx 1 root              root                19 Feb 13 22:39 /usr/bin/clickhouse-disks -> /usr/bin/clickhouse*
lrwxrwxrwx 1 root              root                19 May 13  2022 /usr/bin/clickhouse-extract-from-config -> /usr/bin/clickhouse*
lrwxrwxrwx 1 root              root                19 May 13  2022 /usr/bin/clickhouse-format -> /usr/bin/clickhouse*
lrwxrwxrwx 1 root              root                19 Apr 12  2022 /usr/bin/clickhouse-git-import -> /usr/bin/clickhouse*
lrwxrwxrwx 1 root              root                19 May 13  2022 /usr/bin/clickhouse-keeper -> /usr/bin/clickhouse*
lrwxrwxrwx 1 root              root                19 Apr 12  2022 /usr/bin/clickhouse-keeper-converter -> /usr/bin/clickhouse*
-rwxr-xr-x 1 clickhouse-bridge clickhouse-bridge 140M May  6  2022 /usr/bin/clickhouse-library-bridge*
lrwxrwxrwx 1 root              root                19 May 13  2022 /usr/bin/clickhouse-local -> /usr/bin/clickhouse*
lrwxrwxrwx 1 root              root                19 May 13  2022 /usr/bin/clickhouse-obfuscator -> /usr/bin/clickhouse*
-rwxr-xr-x 1 clickhouse-bridge clickhouse-bridge 141M May  6  2022 /usr/bin/clickhouse-odbc-bridge*
-r-xr-xr-x 1 root              root              461M May  6  2022 /usr/bin/clickhouse.old*
-rwxr-xr-x 1 root              root              2.0K May  6  2022 /usr/bin/clickhouse-report*
lrwxrwxrwx 1 root              root                19 May 13  2022 /usr/bin/clickhouse-server -> /usr/bin/clickhouse*

~ $ ll /usr/bin/clickhouse^C

~ $ ll /usr/bin/clickhouse
-r-xr-xr-x 1 root root 2.3G Feb 13 22:39 /usr/bin/clickhouse*

~ $ clickhouse-client --version
ClickHouse client version 23.2.1.1451 (official build).

```

* github 项目 [releases 页面](https://github.com/ClickHouse/ClickHouse/releases/) 发布了 stable/lts 版本的静态二进制文件。
  而上面的方式我安装到的是最新版本。

# 配置文件问题

首先我在阅读和尝试 [配置文件] 时，发现 `/etc/clickhouse-server/config.d/` 目录下的文件生效了，而 `/etc/clickhouse-server/users.d/`
文件下的文件没有正常覆盖掉 `/etc/clickhouse-server/users.xml`。

于是尝试重启 clickhouse-server（使用 `clickhouse restart`），但没有成功启动，尝试 `clickhouse start` 也是失败的。

```shell
$ clickhouse start
...省略正常内容
Cannot start server. You can execute sudo -u 'clickhouse' /usr/bin/clickhouse-server --config-file /etc/clickhouse-server/config.xml --pid-file /var/run/clickhouse-server/clickhouse-server.pid --daemon without --daemon option to run manually.
Look for logs at /var/log/clickhouse-server/ and for /var/log/clickhouse-server/stderr.log.

$ sudo -u 'clickhouse' /usr/bin/clickhouse-server --config-file /etc/clickhouse-server/config.xml --pid-file /var/run/clickhouse-server/clickhouse-server.pid
Processing configuration file '/etc/clickhouse-server/config.xml'.
Merging configuration file '/etc/clickhouse-server/config.d/default.xml'.
Logging trace to /var/log/clickhouse-server/clickhouse-server.log
Logging errors to /var/log/clickhouse-server/clickhouse-server.err.log
Code: 76. DB::ErrnoException: Cannot open file /var/run/clickhouse-server/clickhouse-server.pid, errno: 2, strerror: No such file or directory. (CANNOT_OPEN_FILE), Stack trace (when copying this message, always include the lines below):

0. ./build_docker/../src/Common/Exception.cpp:91: DB::Exception::Exception(DB::Exception::MessageMasked&&, int, bool) @ 0xdc7b0b5 in /usr/bin/clickhouse
1. ./build_docker/../contrib/llvm-project/libcxx/include/string:1499: DB::ErrnoException::ErrnoException(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, int, int, std::__1::optional<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>>> const&) @ 0xdc7bdf3 in /usr/bin/clickhouse
2. ./build_docker/../src/Common/Exception.cpp:0: DB::throwFromErrnoWithPath(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, int, int) @ 0xdc7c131 in /usr/bin/clickhouse
3. ./build_docker/../src/Common/StatusFile.cpp:0: DB::StatusFile::StatusFile(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>>, std::__1::function<void (DB::WriteBuffer&)>) @ 0xdd74444 in /usr/bin/clickhouse
4. ./build_docker/../contrib/llvm-project/libcxx/include/__functional/function.h:818: BaseDaemon::initialize(Poco::Util::Application&) @ 0xdebed71 in /usr/bin/clickhouse
5. ./build_docker/../programs/server/Server.cpp:482: DB::Server::initialize(Poco::Util::Application&) @ 0xdcfc883 in /usr/bin/clickhouse
6. ./build_docker/../base/poco/Util/src/Application.cpp:334: Poco::Util::Application::run() @ 0x1716f857 in /usr/bin/clickhouse
7. ./build_docker/../programs/server/Server.cpp:477: DB::Server::run() @ 0xdcfc6dc in /usr/bin/clickhouse
8. ./build_docker/../base/poco/Util/src/ServerApplication.cpp:612: Poco::Util::ServerApplication::run(int, char**) @ 0x1718335a in /usr/bin/clickhouse
9. ./build_docker/../programs/server/Server.cpp:0: mainEntryClickHouseServer(int, char**) @ 0xdcf9286 in /usr/bin/clickhouse
10. ./build_docker/../programs/main.cpp:0: main @ 0x885d433 in /usr/bin/clickhouse
11. __libc_start_main @ 0x7ffbbffa3083 in ?
12. _start @ 0x7eb976e in /usr/bin/clickhouse
 (version 23.2.1.1451 (official build))

# 这段错误是我写文章的时候才看到
$ sudo -u 'clickhouse' /usr/bin/clickhouse-server --config-file /etc/clickhouse-server/config.xml
2023.02.14 21:37:35.076257 [ 1188370 ] {} <Error> CertificateReloader: Cannot obtain modification time for certificate file /etc/clickhouse-server/server.crt, skipping update. errno: 2, strerror: No such file or directory
2023.02.14 21:37:35.076470 [ 1188370 ] {} <Error> CertificateReloader: Cannot obtain modification time for key file /etc/clickhouse-server/server.key, skipping update. errno: 2, strerror: No such file or directory
2023.02.14 21:37:35.076573 [ 1188370 ] {} <Debug> CertificateReloader: Initializing certificate reloader.
2023.02.14 21:37:35.077154 [ 1188370 ] {} <Error> CertificateReloader: Poco::Exception. Code: 1000, e.code() = 0, SSL context exception: Error loading private key from file /etc/clickhouse-server/server.key: error:02000002:system library:OPENSSL_internal:No such file or directory (version 23.2.1.1451 (official build))
2023.02.14 21:37:35.077325 [ 1188370 ] {} <Debug> ConfigReloader: Loaded config '/etc/clickhouse-server/config.xml', performed update on configuration
2023.02.14 21:37:35.080954 [ 1188370 ] {} <Debug> ConfigReloader: Loading config '/etc/clickhouse-server/users.xml'
Processing configuration file '/etc/clickhouse-server/users.xml'.
Merging configuration file '/etc/clickhouse-server/users.d/users.xml'.
2023.02.14 21:37:35.090311 [ 1188370 ] {} <Error> Application: Caught exception while setting up access control.: Poco::Exception. Code: 1000, e.code() = 0, Exception: Failed to merge config with '/etc/clickhouse-server/users.d/users.xml': Access to file denied: /etc/clickhouse-server/users.d/users.xml, Stack trace (when copying this message, always include the lines below):

0. ./build_docker/../src/Common/Config/ConfigProcessor.cpp:0: DB::ConfigProcessor::processConfig(bool*, zkutil::ZooKeeperNodeCache*, std::__1::shared_ptr<Poco::Event> const&) @ 0x1479c9f1 in /usr/bin/clickhouse
1. ./build_docker/../contrib/llvm-project/libcxx/include/__memory/shared_ptr.h:701: DB::ConfigProcessor::loadConfig(bool) @ 0x1479ccd4 in /usr/bin/clickhouse
2. ./build_docker/../base/poco/Foundation/include/Poco/AutoPtr.h:118: DB::ConfigReloader::reloadIfNewer(bool, bool, bool, bool) @ 0x147a2a7a in /usr/bin/clickhouse
3. ./build_docker/../src/Common/Config/ConfigReloader.cpp:32: DB::ConfigReloader::ConfigReloader(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, zkutil::ZooKeeperNodeCache&&, std::__1::shared_ptr<Poco::Event> const&, std::__1::function<void (Poco::AutoPtr<Poco::Util::AbstractConfiguration>, bool)>&&, bool) @ 0x147a1b4a in /usr/bin/clickhouse
4. ./build_docker/../contrib/llvm-project/libcxx/include/__memory/unique_ptr.h:714: DB::UsersConfigAccessStorage::load(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::function<std::__1::shared_ptr<zkutil::ZooKeeper> ()> const&) @ 0x121196e6 in /usr/bin/clickhouse
5. ./build_docker/../contrib/llvm-project/libcxx/include/__memory/shared_ptr.h:603: DB::AccessControl::addUsersConfigStorage(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::function<std::__1::shared_ptr<zkutil::ZooKeeper> ()> const&, bool) @ 0x11fcaff2 in /usr/bin/clickhouse
6. ./build_docker/../contrib/llvm-project/libcxx/include/string:1499: DB::AccessControl::addStoragesFromUserDirectoriesConfig(Poco::Util::AbstractConfiguration const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::function<std::__1::shared_ptr<zkutil::ZooKeeper> ()> const&) @ 0x11fcfd97 in /usr/bin/clickhouse
7. ./build_docker/../contrib/llvm-project/libcxx/include/string:1499: DB::AccessControl::addStoragesFromMainConfig(Poco::Util::AbstractConfiguration const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::function<std::__1::shared_ptr<zkutil::ZooKeeper> ()> const&) @ 0x11fc9c26 in /usr/bin/clickhouse
8. ./build_docker/../src/Access/AccessControl.cpp:285: DB::AccessControl::setUpFromMainConfig(Poco::Util::AbstractConfiguration const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::function<std::__1::shared_ptr<zkutil::ZooKeeper> ()> const&) @ 0x11fc897e in /usr/bin/clickhouse
9. ./build_docker/../contrib/llvm-project/libcxx/include/__functional/function.h:818: DB::Server::main(std::__1::vector<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>>, std::__1::allocator<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>>>> const&) @ 0xdd0af0b in /usr/bin/clickhouse
10. ./build_docker/../base/poco/Util/src/Application.cpp:0: Poco::Util::Application::run() @ 0x1716f862 in /usr/bin/clickhouse
11. ./build_docker/../programs/server/Server.cpp:477: DB::Server::run() @ 0xdcfc6dc in /usr/bin/clickhouse
12. ./build_docker/../base/poco/Util/src/ServerApplication.cpp:612: Poco::Util::ServerApplication::run(int, char**) @ 0x1718335a in /usr/bin/clickhouse
13. ./build_docker/../programs/server/Server.cpp:0: mainEntryClickHouseServer(int, char**) @ 0xdcf9286 in /usr/bin/clickhouse
14. ./build_docker/../programs/main.cpp:0: main @ 0x885d433 in /usr/bin/clickhouse
15. __libc_start_main @ 0x7f9ceb7e7083 in ?
16. _start @ 0x7eb976e in /usr/bin/clickhouse
 (version 23.2.1.1451 (official build))
2023.02.14 21:37:35.094238 [ 1188370 ] {} <Error> Application: Poco::Exception. Code: 1000, e.code() = 0, Exception: Failed to merge config with '/etc/clickhouse-server/users.d/users.xml': Access to file denied: /etc/clickhouse-server/users.d/users.xml, Stack trace (when copying this message, always include the lines below):

0. ./build_docker/../src/Common/Config/ConfigProcessor.cpp:0: DB::ConfigProcessor::processConfig(bool*, zkutil::ZooKeeperNodeCache*, std::__1::shared_ptr<Poco::Event> const&) @ 0x1479c9f1 in /usr/bin/clickhouse
1. ./build_docker/../contrib/llvm-project/libcxx/include/__memory/shared_ptr.h:701: DB::ConfigProcessor::loadConfig(bool) @ 0x1479ccd4 in /usr/bin/clickhouse
2. ./build_docker/../base/poco/Foundation/include/Poco/AutoPtr.h:118: DB::ConfigReloader::reloadIfNewer(bool, bool, bool, bool) @ 0x147a2a7a in /usr/bin/clickhouse
3. ./build_docker/../src/Common/Config/ConfigReloader.cpp:32: DB::ConfigReloader::ConfigReloader(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, zkutil::ZooKeeperNodeCache&&, std::__1::shared_ptr<Poco::Event> const&, std::__1::function<void (Poco::AutoPtr<Poco::Util::AbstractConfiguration>, bool)>&&, bool) @ 0x147a1b4a in /usr/bin/clickhouse
4. ./build_docker/../contrib/llvm-project/libcxx/include/__memory/unique_ptr.h:714: DB::UsersConfigAccessStorage::load(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::function<std::__1::shared_ptr<zkutil::ZooKeeper> ()> const&) @ 0x121196e6 in /usr/bin/clickhouse
5. ./build_docker/../contrib/llvm-project/libcxx/include/__memory/shared_ptr.h:603: DB::AccessControl::addUsersConfigStorage(std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::function<std::__1::shared_ptr<zkutil::ZooKeeper> ()> const&, bool) @ 0x11fcaff2 in /usr/bin/clickhouse
6. ./build_docker/../contrib/llvm-project/libcxx/include/string:1499: DB::AccessControl::addStoragesFromUserDirectoriesConfig(Poco::Util::AbstractConfiguration const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::function<std::__1::shared_ptr<zkutil::ZooKeeper> ()> const&) @ 0x11fcfd97 in /usr/bin/clickhouse
7. ./build_docker/../contrib/llvm-project/libcxx/include/string:1499: DB::AccessControl::addStoragesFromMainConfig(Poco::Util::AbstractConfiguration const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::function<std::__1::shared_ptr<zkutil::ZooKeeper> ()> const&) @ 0x11fc9c26 in /usr/bin/clickhouse
8. ./build_docker/../src/Access/AccessControl.cpp:285: DB::AccessControl::setUpFromMainConfig(Poco::Util::AbstractConfiguration const&, std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>> const&, std::__1::function<std::__1::shared_ptr<zkutil::ZooKeeper> ()> const&) @ 0x11fc897e in /usr/bin/clickhouse
9. ./build_docker/../contrib/llvm-project/libcxx/include/__functional/function.h:818: DB::Server::main(std::__1::vector<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>>, std::__1::allocator<std::__1::basic_string<char, std::__1::char_traits<char>, std::__1::allocator<char>>>> const&) @ 0xdd0af0b in /usr/bin/clickhouse
10. ./build_docker/../base/poco/Util/src/Application.cpp:0: Poco::Util::Application::run() @ 0x1716f862 in /usr/bin/clickhouse
11. ./build_docker/../programs/server/Server.cpp:477: DB::Server::run() @ 0xdcfc6dc in /usr/bin/clickhouse
12. ./build_docker/../base/poco/Util/src/ServerApplication.cpp:612: Poco::Util::ServerApplication::run(int, char**) @ 0x1718335a in /usr/bin/clickhouse
13. ./build_docker/../programs/server/Server.cpp:0: mainEntryClickHouseServer(int, char**) @ 0xdcf9286 in /usr/bin/clickhouse
14. ./build_docker/../programs/main.cpp:0: main @ 0x885d433 in /usr/bin/clickhouse
15. __libc_start_main @ 0x7f9ceb7e7083 in ?
16. _start @ 0x7eb976e in /usr/bin/clickhouse
 (version 23.2.1.1451 (official build))
2023.02.14 21:37:35.095173 [ 1188370 ] {} <Error> Application: Exception: Failed to merge config with '/etc/clickhouse-server/users.d/users.xml': Access to file denied: /etc/clickhouse-server/users.d/users.xml
2023.02.14 21:37:35.095255 [ 1188370 ] {} <Information> Application: shutting down
2023.02.14 21:37:35.095313 [ 1188370 ] {} <Debug> Application: Uninitializing subsystem: Logging Subsystem
2023.02.14 21:37:35.096778 [ 1188372 ] {} <Trace> BaseDaemon: Received signal -2
2023.02.14 21:37:35.096887 [ 1188372 ] {} <Information> BaseDaemon: Stop SignalListener thread
```


使用 `clickhouse-server` 可以启动，但原来的数据库的内容没有了（升级不会丢失数据）。

搜索一番之后发现遇到和 [#41819] 一样的问题，根据那里的回复，使用以下命令解决。它隐藏在 [config.xml] 文件中。

[config.xml]: https://github.com/ClickHouse/ClickHouse/blob/master/programs/server/config.xml#L271

```shell
$ openssl req -subj "/CN=localhost" -new -newkey rsa:2048 -days 365 -nodes -x509 -keyout /etc/clickhouse-server/server.key -out /etc/clickhouse-server/server.crt
Generating a RSA private key
..........................................................+++++
...........................................+++++
writing new private key to '/etc/clickhouse-server/server.key'
-----
```

注意：我使用了 root 创建配置文件和生成秘钥文件，因此还遇到权限问题（但未在日志中显示），所以需要
`chown clickhouse: /etc/clickhouse-server/ -R`

```shell
2023.02.15 21:51:28.888353 [ 1369548 ] {} <Error> Application: Poco::Exception. Code: 1000, e.code() = 0, I/O error: ECKeyImpl, cannot open file: /etc/clickhouse-server/server.key, Stack trace (when copying this message, always include the lines below):
```

[配置文件]: https://clickhouse.com/docs/en/operations/configuration-files/
[#41819]: https://github.com/ClickHouse/ClickHouse/issues/41819
