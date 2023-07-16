https://shockerli.net/post/mysql-source-macos-vscode-debug-5-7/

环境
macOS Big Sur 11.5.2
VSCode 1.59.1
MySQL 5.7.35
依赖
如果以下软件未安装或版本不符合，使用 brew 安装即可

CMake
g++
openssl 1.1
VSCode 插件
在 VSCode 应用商店搜索并安装以下插件

C/C++
C/C++ Clang Command Adapter
CodeLLDB
CMake Tools
下载源码
从官网下载携带 boost 版本源码

下载链接：https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-boost-5.7.35.tar.gz

也可以从 GitHub 上克隆代码，切换到指定 TAG 或分支。

解压源码：

1
2
tar -zxf mysql-boost-5.7.35.tar.gz
cd mysql-5.7.35
Patch 源码
如果 MySQL <= 8.0.21，则需要执行以下脚本 Patch 源码：

1
2
mv VERSION MYSQL_VERSION
sed -i '' 's|${CMAKE_SOURCE_DIR}/VERSION|${CMAKE_SOURCE_DIR}/MYSQL_VERSION|g' cmake/mysql_version.cmake
具体原因，可参考文章：MySQL 源码 —— 问题 expanded from macro MYSQL_VERSION_MAJOR

创建目录
编译、安装、数据、配置等都统一到 cmake-build-debug 目录，方便管理、不与其他冲突

1
mkdir -p cmake-build-debug/{data,etc}
编译
有 2 种方式，命令行、CMake Tools

推荐 CMake Tools，因为后续 Debug 都在 VSCode 中，重新编译或切换编译目标也方便

CMake Tools
关于 CMake Tools 的详细配置和用法，参考官方文档：CMake Tools for Visual Studio Code documentation

配置
创建文件 .vscode/settings.json，并写入配置：

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
{
"cmake.buildBeforeRun": true,
"cmake.buildDirectory": "${workspaceFolder}/cmake-build-debug/build",
"cmake.configureSettings": {
"WITH_DEBUG": "1",
"CMAKE_INSTALL_PREFIX": "${workspaceFolder}/cmake-build-debug",
"MYSQL_DATADIR": "${workspaceFolder}/cmake-build-debug/data",
"SYSCONFDIR": "${workspaceFolder}/cmake-build-debug/etc",
"MYSQL_TCP_PORT": "3307",
"MYSQL_UNIX_ADDR": "${workspaceFolder}/cmake-build-debug/data/mysql-debug.sock",
"WITH_BOOST": "${workspaceFolder}/boost",
"DOWNLOAD_BOOST": "1",
"DOWNLOAD_BOOST_TIMEOUT": "600"
}
}
查看 > 命令面板，搜索CMake，找到CMake: Configure并回车运行： cmake-configure

等待配置完成，输出以下日志：

1
2
3
4
5
6
7
8
[main] Configuring folder: mysql-5.7.35
[proc] Executing command: /usr/local/bin/cmake --no-warn-unused-cli -DWITH_DEBUG:STRING=1 -DCMAKE_INSTALL_PREFIX:STRING=/path/to/mysql-5.7.35/cmake-build-debug -DMYSQL_DATADIR:STRING=/path/to/mysql-5.7.35/cmake-build-debug/data -DSYSCONFDIR:STRING=/path/to/mysql-5.7.35/cmake-build-debug/etc -DMYSQL_TCP_PORT:STRING=3307 -DMYSQL_UNIX_ADDR:STRING=/path/to/mysql-5.7.35/cmake-build-debug/data/mysql-debug.sock -DWITH_BOOST:STRING=/path/to/mysql-5.7.35/boost -DDOWNLOAD_BOOST:STRING=1 -DDOWNLOAD_BOOST_TIMEOUT:STRING=600 -DCMAKE_EXPORT_COMPILE_COMMANDS:BOOL=TRUE -DCMAKE_BUILD_TYPE:STRING=Debug -H/path/to/mysql-5.7.35 -B/path/to/mysql-5.7.35/cmake-build-debug/build -G "Unix Makefiles"

···中间日志省略···

[cmake] -- Configuring done
[cmake] -- Generating done
[cmake] -- Build files have been written to: /path/to/mysql-5.7.35/cmake-build-debug/build
编译
查看 > 命令面板，搜索CMake，找到CMake: Build Target并回车运行： 构建目标

查看 > 命令面板，搜索mysqld，找到mysqld EXECUTABLE并回车运行： 选择目标

然后，就开始编译了，最终输出日志：

1
2
3
4
5
6
7
8
[main] Building folder: mysql-5.7.35 mysqld
[build] Starting build
[proc] Executing command: /usr/local/bin/cmake --build /path/to/mysql-5.7.35/cmake-build-debug/build --config Debug --target mysqld -j 6 --

···中间日志省略···

[build] [100%] Built target mysqld
[build] Build finished with exit code 0
命令行
切换到 cmake-build-debug 目录：

1
cd cmake-build-debug
配置
1
2
3
4
5
6
7
8
9
10
11
cmake .. \
-B./build \
-DWITH_DEBUG=1 \
-DCMAKE_INSTALL_PREFIX=. \
-DMYSQL_DATADIR=./data \
-DSYSCONFDIR=./etc \
-DMYSQL_TCP_PORT=3307 \
-DMYSQL_UNIX_ADDR=mysql-debug.sock \
-DWITH_BOOST=../boost \
-DDOWNLOAD_BOOST=1 \
-DDOWNLOAD_BOOST_TIMEOUT=60000
编译
-j 3 表示3个线程，提高速度

--build build 表示构建build目录，此处与-B./build保持一致

--target mysqld 表示只构建mysqld

1
cmake --build build --target mysqld -j 3
Tips: make 命令会编译所有目标

初始化数据库
以下步骤切换到 cmake-build-debug 目录下执行：

MySQL 配置文件
1
2
3
4
5
6
cat > etc/my.cnf <<EOF
[mysqld]
port=3307
socket=mysql.sock
innodb_file_per_table=1
EOF
修改 port 和 socket 是为了避免与本机数据库冲突。

初始化数据库
1
2
3
4
5
# 不携带配置文件，因为CMake时指定了SYSCONFDIR
build/sql/mysqld --initialize-insecure

# 也可携带文件路径
build/sql/mysqld --defaults-file=etc/my.cnf --initialize-insecure
配置调试参数
创建文件 .vscode/launch.json，并写入配置：

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
{
// 使用 IntelliSense 了解相关属性。
// 悬停以查看现有属性的描述。
// 欲了解更多信息，请访问: https://go.microsoft.com/fwlink/?linkid=830387
"version": "0.2.0",
"configurations": [
{
"type": "lldb",
"request": "launch",
"name": "Debug mysqld",
"program": "${workspaceFolder}/cmake-build-debug/build/sql/mysqld",
"args": [
"--defaults-file=${workspaceFolder}/cmake-build-debug/etc/my.cnf"
],
"cwd": "${workspaceFolder}"
},
{
"type": "lldb",
"request": "launch",
"name": "Debug mysql",
"program": "${workspaceFolder}/cmake-build-debug/build/client/mysql",
"args": [
"-uroot",
"-P3307",
"-h127.0.0.1"
],
"cwd": "${workspaceFolder}"
}
]
}
启动调试
启动 mysqld
左侧导航栏 > 运行与调试，选择 Debug mysqld，开始调试。 选择mysqld调试

终端中会输出 mysqld 服务器的启动日志：

1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
[Warning] TIMESTAMP with implicit DEFAULT value is deprecated. Please use amp server option (see documentation for more details).
[Note] --secure-file-priv is set to NULL. Operations related to importing and exporting data
[Note] /path/to/mysql-5.7.35/cmake-build-debug/build/sql/mysqld (mysqld ess 70489 ...
[ERROR] Can't find error-message file '/path/to/mysql-5.7.35/cmake-build-debug/shareage file location and 'lc-messages-dir' configuration directive.
[Warning] Setting lower_case_table_names=2 because file system for /path/to/data/ is case insensitive
[Note] InnoDB: !!!!!!!! UNIV_DEBUG switched on !!!!!!!!!
[Note] InnoDB: Mutexes and rw_locks use GCC atomic builtins
[Note] InnoDB: Uses event mutexes
[Note] InnoDB: GCC builtin __atomic_thread_fence() is used for memory barrier
[Note] InnoDB: Compressed tables use zlib 1.2.11
[Note] InnoDB: Number of pools: 1
[Note] InnoDB: Using CPU crc32 instructions
[Note] InnoDB: Initializing buffer pool, total size = 128M, instances = 1, chunk size = 128M
[Note] InnoDB: Completed initialization of buffer pool
[Note] InnoDB: Highest supported file format is Barracuda.
[Note] InnoDB: Log scan progressed past the checkpoint lsn 2748488
[Note] InnoDB: Doing recovery: scanned up to log sequence number 2748497
[Note] InnoDB: Database was not shutdown normally!
[Note] InnoDB: Starting crash recovery.
[Note] InnoDB: Removed temporary tablespace data file: "ibtmp1"
[Note] InnoDB: Creating shared tablespace for temporary tables
[Note] InnoDB: Setting file './ibtmp1' size to 12 MB. Physically writing the file full;
[Note] InnoDB: File './ibtmp1' size is now 12 MB.
[Note] InnoDB: 96 redo rollback segment(s) found. 96 redo rollback segment(s) are active.
[Note] InnoDB: 32 non-redo rollback segment(s) are active.
[Note] InnoDB: Waiting for purge to start
[Note] InnoDB: 5.7.35 started; log sequence number 2748497
[Note] InnoDB: Loading buffer pool(s) from /path/to/mysql-5.7.35/cmake-build-debug/
[Note] Plugin 'FEDERATED' is disabled.
[Note] InnoDB: Buffer pool(s) load completed at 210826 21:45:37
[Note] Found ca.pem, server-cert.pem and server-key.pem in data directory. Trying to enable
[Note] Skipping generation of SSL certificates as certificate files are present in data
[Warning]
[Warning]
[Warning] CA certificate ca.pem is self signed.
[Note] Skipping generation of RSA key pair as key files are present in data directory.
[Note] Server hostname (bind-address): '*'; port: 3307
[Note] IPv6 is available.
[Note]   - '::' resolves to '::';
[Note] Server socket created on IP: '::'.
[Note] Event Scheduler: Loaded 0 events
[Note]
连接数据库
以同样的方式，先编译 mysql，然后启动 mysql 客户端，连接到刚刚启动的服务端 mysqld： 选择mysql连接服务端

这样就可以在一个窗口下，既 Debug 源码，也能看到输出。

当然，也可以通过其他 mysql 客户端，或 Navicat 连接：

1
mysql -uroot -P3307 -h127.0.0.1
Debug 界面
debug-window

小技巧
代码蒙灰
C/C++ 扩展有个 C_Cpp.dimInactiveRegions 参数，控制非活动区域代码变灰。 C_Cpp.dimInactiveRegions

参考资料
MySQL 源码阅读 —— macOS CLion 编译调试 MySQL 5.7
MySQL 源码阅读 —— 问题 expanded from macro MYSQL_VERSION_MAJOR
CMake Tools for Visual Studio Code documentation
文章作者 Jioby

发布日期 2021-08-27

上次更新 2021-08-27

许可协议 CC BY-NC-ND 4.0（请看转载要求）

原文链接 https://shockerli.net/post/mysql-source-macos-vscode-debug-5-7/

MySQL MySQL源码阅读
如何不靠运气致富MySQL 源码阅读 —— 问题 expanded from macro MYSQL_VERSION_MAJOR 
