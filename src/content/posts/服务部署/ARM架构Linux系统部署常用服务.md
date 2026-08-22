---
title: ARM架构Linux系统部署常用服务
published: 2026-08-22
description: 在ARM架构Linux上源码编译部署nginx、mysql、docker等常用服务,创建systemd服务实现开机自启。
category: 服务部署
tags:
  - ARM
  - Linux
  - 服务部署
slug: arm-linux-deploy-common-services
---

# ARM架构Linux系统部署常用服务

## nginx

安装必要工具

`yum install gcc pcre pcre-devel zlib zlib-devel openssl openssl-devel`

下载源码并解压到任意目录，如 `/root/nginx`，不要将源文件放到 `/usr/local/nginx` 下，除非你指定输出目录，编译时会自动在 `/usr/local/nginx` 创建目录，
配置和编译软件并添加 SSL module

```shell
./configure --with-http_ssl_module --with-http_gzip_static_module
make
make install

#可选。配置www用户管理，需要www用户和用户组，可参考下面mysql添加。
./configure --user=www --group=www --with-http_ssl_module --with-http_gzip_static_module
```

添加环境变量。

```shell
nano /etc/profile
export PATH=$PATH:/usr/local/nginx/sbin
source /etc/profile
```

创建 nginx 系统服务

创建 `nginx.service` 文件并添加如下内容

`nano /lib/systemd/system/nginx.service`

```shell
[Unit]
Description=nginx service
After=network.target

[Service]
Type=forking
ExecStart=/usr/local/nginx/sbin/nginx //nginx的可执行路径,依据实际情况修改
ExecReload=/usr/local/nginx/sbin/nginx -s reload
ExecStop=/usr/local/nginx/sbin/nginx -s stop
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

开机自启动

`systemctl enable nginx`

## mysql

**注意：MYSQL8.0 开始才支持 arm 架构，我们可以去第三方下载编译好的安装包，或者可以采取 docker 安装.**

系统自带 mariadb 和 mysql 冲突，请先卸载 mariadb

`yum remove mariadb-server`

以下步骤采用华为镜像站编译好的 mysql 5.7.27 arm 版本,[下载链接](https://obs.cn-north-4.myhuaweicloud.com/obs-mirror-ftp4/database/mysql-5.7.27-aarch64.tar.gz)。

关闭 SELINUX

编辑 `nano /etc/sysconfig/selinux`

将 `SELINUX=enforcing` 改为 `SELINUX=disabled`

创建 mysql 用户和组

mysql 用户不能登录系统选项，不创建用户的主目录。

`groupadd -r mysql && useradd -r -g mysql -s /sbin/nologin -M mysql`

安装必要依赖

`yum install gcc gcc-c++ ncurses-devel bison cmake openssl-devel libtirpc-devel libaio-devel`

下载 Mysql 文件

[下载链接](https://obs.cn-north-4.myhuaweicloud.com/obs-mirror-ftp4/database/mysql-5.7.27-aarch64.tar.gz)

解压到 `/usr/local/mysql`

创建 mysql 配置文件软链接

`ln -sf /usr/local/mysql/my.cnf /etc/my.cnf`

覆盖依赖包

```shell
cp -rf /usr/local/mysql/extra/lib /usr/lib64/
mv /usr/lib64/libstdc++.so.6 /usr/lib64/libstdc++.so.6.old
ln -s /usr/lib64/libstdc++.so.6.0.24 /usr/lib64/libstdc++.so.6
```

备份并按需修改 Mysql 配置文件

```shell
cd /usr/local/mysql
cp my.cnf my.cnf.bak
nano my.cnf
```

按实际情况修改配置文件。

```shell
[client]
# 客户端连接 MySQL 服务器的端口号，通常是 3306。
port = 3306
# MySQL 服务器的套接字文件路径，用于本地连接。
socket = /dev/shm/mysql.sock

[mysqld]
# MySQL 服务器监听的端口号，通常也是 3306。
port = 3306
# MySQL 服务器的套接字文件路径，用于本地连接。
socket = /dev/shm/mysql.sock
# MySQL 的根目录路径，通常用于安装 MySQL 的根目录。
basedir = /user/local/mysql
# 存放数据库文件的目录路径。
datadir = /user/local/mysql/data
# 启用binglog日志文件，可以指定目录，如果不指定则放在数据目录下面
log_bin = mysql-bin
#存放 MySQL 进程 ID 的文件路径。
pid-file = /user/local/mysql/data/mysql.pid
#错误日志路径
log_error = /user/local/mysql/logs/mysql-error.log
#慢查询sql日志路径
slow_query_log_file = /user/local/mysql/logs/mysql-slow.log
#临时数据路径
tmpdir=/user/local/mysql/mysql_tmp
#MySQL 服务器运行时使用的用户（通常是 "mysql" 用户）
user = mysql
#用于指定 MySQL 服务器绑定的 IP 地址，0.0.0.0 表示绑定到所有可用的 IP 地址。
bind-address = 0.0.0.0
# MySQL 服务器的唯一标识符，用于主从复制等。
server-id = 1
# 连接到 MySQL 服务器时初始化 SQL 命令。
init-connect = 'SET NAMES utf8mb4'
# 服务器默认的字符集。
character-set-server = utf8mb4
#skip-name-resolve
#skip-networking
#允许在内核中等待的连接数量
back_log = 300
# 允许的最大并发连接数。
max_connections = 1000
# 最大连接错误数
max_connect_errors = 6000
# 打开的文件数限制。
open_files_limit = 65535
# 表缓存大小。
table_open_cache = 128
# 单个查询的最大允许数据包大小
max_allowed_packet = 4M
# 二进制日志缓存大小
binlog_cache_size = 1M
# 最大堆表大小
max_heap_table_size = 8M
# 临时表大小
tmp_table_size = 16M
# 读取缓冲区大小
read_buffer_size = 2M
# 随机读取缓冲区大小
read_rnd_buffer_size = 8M
# 排序缓冲区大小
sort_buffer_size = 8M
# 连接缓冲区大小
join_buffer_size = 8M
# 键缓冲区大小
key_buffer_size = 4M
# 线程缓存大小
thread_cache_size = 8
# 查询缓存类型 (1 表示启用)
query_cache_type = 1
# 查询缓存大小
query_cache_size = 8M
# 查询缓存限制
query_cache_limit = 2M
# 全文索引最小词长度
ft_min_word_len = 4
# 二进制日志文件的格式
binlog_format = mixed
# 二进制日志文件自动清理天数
expire_logs_days = 30
# 启用慢查询日志 (1 表示启用)
slow_query_log = 1
# 定义慢查询的阈值时间
long_query_time = 1
# 性能模式 (0 表示禁用)
performance_schema = 0
# 明确指定 MySQL 是否应该使用严格的模式来检查日期和时间值
explicit_defaults_for_timestamp
# 表名大小写不敏感 (1 表示启用)
lower_case_table_names = 1
# 禁用外部锁定，用于控制表级锁定
skip-external-locking
# 默认存储引擎 (InnoDB)
default_storage_engine = InnoDB
# 默认存储引擎 (MyISAM)
#default-storage-engine = MyISAM
# 每个表使用单独的 InnoDB 文件
innodb_file_per_table = 1
# InnoDB 可以打开的最大文件数
innodb_open_files = 500
# InnoDB 缓冲池大小
innodb_buffer_pool_size = 64M
# InnoDB 写 I/O 线程数
innodb_write_io_threads = 4
# InnoDB 读 I/O 线程数
innodb_read_io_threads = 4
# InnoDB 线程并发度
innodb_thread_concurrency = 0
# InnoDB 清理线程数
innodb_purge_threads = 1
# InnoDB 日志刷新行为
innodb_flush_log_at_trx_commit = 2
# InnoDB 日志缓冲大小
innodb_log_buffer_size = 2M
# InnoDB 日志文件大小
innodb_log_file_size = 32M
# InnoDB 日志文件组数
innodb_log_files_in_group = 3
# InnoDB 最大脏页百分比
innodb_max_dirty_pages_pct = 90
# InnoDB 锁等待超时时间
innodb_lock_wait_timeout = 120
# 批量插入缓冲区大小
bulk_insert_buffer_size = 8M
# MyISAM 排序缓冲区大小
myisam_sort_buffer_size = 8M
# MyISAM 最大排序文件大小
myisam_max_sort_file_size = 10G
# MyISAM 修复线程数
myisam_repair_threads = 1
# 交互超时时间
interactive_timeout = 28800
# 等待超时时间
wait_timeout = 28800
[mysqldump]
quick
# mysqldump 最大允许数据包大小
max_allowed_packet = 100M

[myisamchk]
# MyISAM 检查工具的键缓冲区大小
key_buffer_size = 8M
# MyISAM 检查工具的排序缓冲区大小
sort_buffer_size = 8M
# 读缓存大小
read_buffer = 4M
# 写缓存大小
write_buffer = 4M
```

创建系统服务和环境变量并设置开机自启

环境变量

```shell
nano /etc/profile
export MYSQL_HOME=/usr/local/mysql
export PATH=$PATH:$MYSQL_HOME/bin
source /etc/profile #刷新环境变量
```

创建系统服务

`cp -rf /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld`

`chmod +x /etc/init.d/mysqld`

开机自启

`systemctl enable mysqld`

无密码初始化

先启动服务

`systemctl start mysqld`

查看服务状态

`systemctl status mysqld`

如果服务没起来，参考状态里的报错信息，可能会缺少 logs 目录，在 mysql 路径下创建缺少的目录，并修改权限。

`chown -R mysql:mysql /usr/local/mysql`

初始化

`mysqld --initialize-insecure`

初始化成功后可以使用 root 用户登录并设置密码。

```shell
#使用root用户免密登录
mysql -u root
#使用mysql库
use mysql;
#更新root密码
update user set authentication_string=password("你的密码") where user="root";
#赋予所有IP都可以使用root用户远程连接的权限
grant all privileges on *.* to root@'%' identified by "你的密码";
#刷新权限配置
flush privileges;
#退出mysql
exit
```

## tomcat

下载并解压 tomcat 源文件到 `/usr/local/tomcat`

设置环境变量

```shell
nano /etc/profile
export TOMCAT\_HOME=/usr/local/tomcat
export CATANILA\_HOME=/usr/local/tomcat
source /etc/profile
```

创建系统服务

`nano /usr/lib/systemd/system/tomcat.service`

添加以下内容并按需修改

```shell
[Unit]

Description=tomcat
After=syslog.target network.target remote-fs.target nss-lookup.target

[Service]

Type=forking
ExecStart=/usr/local/tomcat/bin/startup.sh ExecReload=/usr/local/tomcat/bin/startup.sh
ExecStop=/usr/local/tomcat/bin/shutdown.sh

[Install]

WantedBy=multi-user.target
```

设置启动 tomcat 服务并设置开机自启

`systemctl start tomcat`

`systemctl enable tomcat`

放行 8080 端口

```shell
firewall-cmd --zone=public --add-port=8080/tcp --permanent
firewall-cmd --reload
命令含义：

--zone 作用域
--add-port=6666/tcp 添加端口，格式为：端口/通讯协议
--permanent 永久生效，没有此参数重启后失效

查看端口号
netstat -ntlp 查看当前所有tcp端口
netstat -l 列出所有监听的端口
```

## docker

银河麒麟高级服务器 V10 自带的 podman 和 docker 冲突，卸载冲突软件。

`yum remove podman`

下载并解压 docker 安装包到任意目录,如 `/root/docker`

aarch64 离线包下载链接，选择需要的版本,[下载链接](https://download.docker.com/linux/static/stable/aarch64/)。

复制文件

`cp -p /root/docker/* /usr/bin`

创建系统服务

`nano /etc/systemd/system/docker.service`

添加以下内容按需修改

```shell
[Unit]
Description=Docker Application Container Engine
Documentation=<https://docs.docker.com>
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP \$MAINPID
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process
# restart the docker process if it exits prematurely
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
```

启动并设置开机自启

```shell
systemctl daemon-reload
#重新加载配置文件
systemctl start docker
#启动docker
systemctl enable docker
#设置开机自启
```

## docker安装redis6.2.3

需要哪个版本就拉取哪个版本的镜像，不指定版本默认拉取 latest

拉取镜像

`docker pull redis:6.2.3`

创建容器

`docker run -itd --name redis -p 6379:6379 redis:6.2.3`

持久化参数

`-v /host/path:/container/path`

查看所有容器

`docker ps -a`

## OpenGauss

安装 opengauss 轻量版

**官方文档说目前仅支持在防火墙关闭的状态下进行安装。**

关闭 SELINUX

`nano /etc/selinux/config`

修改修改'SELINUX'的值'disabled'

`SELINUX=disabled`

关闭防火墙

`systemctl disable firewalld.service`

`systemctl stop firewalld.service`

关闭 RemoveIPC

`nano /etc/systemd/logind.conf`

`RemoveIPC=no`

完成后重启系统

创建运行 opengauss 的普通用户

`useradd -d /home/openGauss -m -G wheel openGauss`

-d 指定用户主目录，如果此目录不存在，则用-m 创建

-g 指定用户所属的用户组

-G 指定用户所属的附加用户组，wheel 用户组为 openeuler 的 sudo 组

指定 openGauss 用户的密码

`passwd openGauss`

创建 opengauss 的安装目录，并将所有者改为 opengauss 用户

`mkdir /usr/local/openGauss`

`chown openGauss:openGauss /usr/local/openGauss`

切换普通用户开始安装

`su openGauss`

下载并解压 opengauss 到指定目录

```shell
cd /usr/local/openGauss
wget https://xxxx.opengauss.tar.gz
tar -zxvf opengauss.tar.gz
```

安装

`sh ./install.sh --mode single -D /usr/local/openGauss/data -R /usr/local/openGauss/install --start`

-D 数据库数据路径， 不可和安装目录交叉，必须为空。

-R 数据库安装路径，不可和数据目录交叉。

其他参数可参考[官方文档](https://docs-opengauss.osinfra.cn/zh/docs/5.0.0-lite/docs/InstallationGuide/安装openGauss.html)

输入密码后等待安装完成

安装完成后脚本会自动添加环境变量到 opengauss 用户环境，执行 `source /home/openGauss/.bashrc` 使其生效

验证安装

`ps ux | grep gaussdb`

`gs_ctl query -D /usr/local/openGauss/data`

查看输出结果可看到数据库是否在运行。

数据库安装完成后，默认生成名称为 postgres 的数据库

也可以连接默认的 postgres 数据库来验证。

`gsql -d postgres`

创建系统服务并设置开机自启

`nano /usr/lib/systemd/system/openGauss.service`

添加以下内容

```shell
[Unit]
Description=openGauss
Documentation=openGauss
After=syslog.target
After=network.target

[Service]
Type=forking
#用户名和用户组按实际安装修改
User=openGauss
Group=openGauss
#路径和命令按实际情况修改,不知道的话可以查看安装数据库用户的环境变量。
Environment=PGDATA=/usr/local/openGauss/data
Environment=GAUSSHOME=/usr/local/openGauss/install
Environment=LD_LIBRARY_PATH=/usr/local/openGauss/install/lib
ExecStart=/usr/local/openGauss/install/bin/gs_ctl start
ExecReload=/usr/local/openGauss/install/bin/gs_ctl restart
ExecStop=/usr/local/openGauss/install/bin/gs_ctl stop
KillMode=mixed
KillSignal=SIGINT
TimeoutSec=0

[Install]
WantedBy=multi-user.target
```

设置开机自启

`systemctl enable openGauss`
