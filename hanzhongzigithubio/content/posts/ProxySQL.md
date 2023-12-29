---
# 文章标题
title: "ProxySQL 一篇文章搞定"
# 文章内容摘要
description: "ProxySQL 速通"

# 发表日期
date: 2023-12-29T18:48:55+08:00
# 最后修改日期
lastmod: 2023-12-29T18:48:55+08:00
# 分类
categories:
 - '专业技能'
# 标签
tags:
 - 'ProxySQL'

#文章是否开启评论
comment:
  enable: true
# 用于对外访问的地址
url: post/proxysql
# 是否显示目录
toc: true

# 文章封面图片相关属性
cover:
   image: "" #图片路径例如：posts/tech/123/123.png
   zoom: 50% # 图片大小，例如填写 50% 表示原图像的一半大小
   caption: "" #图片底部描述
   alt: ""
   relative: false

#是否为草稿
draft: false
---

ProxySql.md

# ProxySQL

| 编写人 | 编写时间  | 编写说明                                                     |
| ------ | --------- | ------------------------------------------------------------ |
| 章瀚中 | 2020-9-12 | 接上一篇[Mysql5.7主从搭建](http://wiki-private.capitalonline.net:8090/pages/viewpage.action?pageId=59474716)一起观看较佳 |
|        |           |                                                              |

简单介绍: ProxySQL 是基于 MySQL 的一款开源的中间件的产品，是一个灵活的 MySQL 代理层，可以实现读写分离，支持 Query 路由功能，支持动态指定某个 SQL 进行缓存，支持动态加载（无需重启 ProxySQL 服务），故障切换和一些 SQL 的过滤功能。

github地址：https://github.com/sysown/proxysql/releases

官方文档: https://proxysql.com/documentation/

## 下载与安装

### Ubuntu

>#### 下载 
>
> `wget  https://github.com/sysown/proxysql/releases/download/v2.0.14/proxysql_2.0.14-dbg-ubuntu16_amd64.deb `
>
>#### 安装
>
>`dpkg -i proxysql_2.0.14-dbg-ubuntu16_amd64.deb`

### Centos8

>#### 下载
>
>wget https://github.com/sysown/proxysql/releases/download/v2.0.14/proxysql-2.0.14-1-dbg-centos8.x86_64.rpm
>
>#### 安装
>
>` yum -y install proxysql-2.0.14-1-centos8.x86_64.rpm`

centos7 : https://github.com/sysown/proxysql/releases/download/v2.0.14/proxysql-2.0.14-1-centos7.x86_64.rpm

## 启动

启动：`systemctl restart proxysql`

停止:  `systemctl stop proxysql`

默认监听 `6032` 和 `6033` 端口,`6032 `是 ProxySQL 的管理端口号，`6033`是对外服务的端口号 
ProxySQL 的用户名和密码都是默认的 `admin`。

## 登录

需要有mysql的客户端

```
[root@localhost proxysql]# mysql -u admin -padmin -h 127.0.0.1 -P 6032 --prompt='ProxySQL-Admin>  '

ProxySQL-Admin>  show databases;
+-----+---------------+-------------------------------------+
| seq | name          | file                                |
+-----+---------------+-------------------------------------+
| 0   | main          |  内存配置数据库，即 MEMORY，表里存放后端 db 实例、用户验证、路由规则等信息。   |
| 2   | disk          | /var/lib/proxysql/proxysql.db 持久化的磁盘的配置       |
| 3   | stats         |   统计信息的汇总           |
| 4   | monitor       | 一些监控的收集信息，比如数据库的健康状态等  |
| 5   | stats_history | /var/lib/proxysql/proxysql_stats.db这个库是 ProxySQL 收集的有关其内部功能的历史指标 |
+-----+---------------+-------------------------------------+
5 rows in set (0.00 sec)

```

note: If your MySQL client version is version 8.04 or higher add `--default-auth=mysql_native_password` to the above command to connect to the admin interface.

这里面有5个数据库

main: 内存配置数据库，即 MEMORY，表里存放后端 db 实例、用户验证、路由规则等信息。

```
ProxySQL-Admin>  use main
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
ProxySQL-Admin>  show tables;
+----------------------------------------------------+
| tables                                             |
+----------------------------------------------------+
| global_variables                                   |
| mysql_aws_aurora_hostgroups                        |
| mysql_collations                                   |
| mysql_firewall_whitelist_rules                     |
| mysql_firewall_whitelist_sqli_fingerprints         |
| mysql_firewall_whitelist_users                     |
| mysql_galera_hostgroups                            |
| mysql_group_replication_hostgroups                 |
| mysql_query_rules指定 Query 路由到后端不同服务器的规则列表|
| mysql_query_rules_fast_routing                     |
| mysql_replication_hostgroups 配置主从分组信息         |
| mysql_servers后端可以连接 MySQL 服务器的列表            |
| mysql_users配置后端数据库的账号和监控的账号。             |
| proxysql_servers                                   |
| restapi_routes                                     |
| runtime_checksums_values                           |
| runtime_global_variables                           |
| runtime_mysql_aws_aurora_hostgroups                |
| runtime_mysql_firewall_whitelist_rules             |
| runtime_mysql_firewall_whitelist_sqli_fingerprints |
| runtime_mysql_firewall_whitelist_users             |
| runtime_mysql_galera_hostgroups                    |
| runtime_mysql_group_replication_hostgroups         |
| runtime_mysql_query_rules                          |
| runtime_mysql_query_rules_fast_routing             |
| runtime_mysql_replication_hostgroups               |
| runtime_mysql_servers                              |
| runtime_mysql_users                                |
| runtime_proxysql_servers                           |
| runtime_restapi_routes                             |
| runtime_scheduler                                  |
| scheduler                                          |
+----------------------------------------------------+
32 rows in set (0.00 sec)

```

**注意**： 表名以 runtime_开头的表示 ProxySQL 当前运行的配置内容，不能通过 DML 语句修改。

只能修改对应的不以 runtime 开头的表，然后 “LOAD” 使其生效，“SAVE” 使其存到硬盘以供下次重启加载。 

## 多层的配置系统

![](C:\Users\Administrator\Desktop\1033265-20200210150442128-1932198210.png)

整套配置系统分为三层：顶层为 RUNTIME ,中间层为 MEMORY , 底层也就是持久层 DISK 和 CONFIG FILE 。

RUNTIME ： 代表 ProxySQL 当前生效的正在使用的配置，无法直接修改这里的配置，必须要从下一层 “load” 进来。 
MEMORY： MEMORY 层上面连接 RUNTIME 层，下面连接持久层。这层可以正常操作 ProxySQL  配置，随便修改，不会影响生产环境。修改一个配置一般都是现在 MEMORY 层完成的，确认正常之后在加载达到 RUNTIME 和 持久化的磁盘上。

DISK 和 CONFIG FILE：持久化配置信息，重启后内存中的配置信息会丢失，所需要将配置信息保留在磁盘中。重启时，可以从磁盘快速加载回来。



 一般在内存那层修改 ,然后保存到运行系统，保存到磁盘数据库系统

```
load xxx to runtime;
save xxx to disk;
```

- proxysql 启动时，首先去找/etc/proxysql.cnf 找到它的datadir,如果datadir下有proxysql.db 就加载proxysql.db的配置
- 如果启动proxysql时带有`--init`标志，会用/etc/proxsql.cnf的配置，把Runtime，disk全部初始化一下
- 在调用是调用`--reload` 会把/etc/proxysql.cnf 和disk 中配置进行合并。如果冲突需要用户干预。disk会覆盖config file。

## 配置主从分组信息

配置主从分组信息

```

ProxySQL-Admin>  show create table mysql_replication_hostgroups\G;
*************************** 1. row ***************************
       table: mysql_replication_hostgroups
Create Table: CREATE TABLE mysql_replication_hostgroups (
    writer_hostgroup INT CHECK (writer_hostgroup>=0) NOT NULL PRIMARY KEY,
    reader_hostgroup INT NOT NULL CHECK (reader_hostgroup<>writer_hostgroup AND reader_hostgroup>=0),
    check_type VARCHAR CHECK (LOWER(check_type) IN ('read_only','innodb_read_only','super_read_only','read_only|innodb_read_only','read_only&innodb_read_only')) NOT NULL DEFAULT 'read_only',
    comment VARCHAR NOT NULL DEFAULT '', UNIQUE (reader_hostgroup))
1 row in set (0.00 sec)

```

首先创建写组和读组

`insert ``into` `mysql_replication_hostgroups ( writer_hostgroup, reader_hostgroup, comment) values (10,20,``'proxy'``);`

确保此配置均已写入三种配置

```
ProxySQL-Admin>  select * from main.mysql_replication_hostgroups;
+------------------+------------------+------------+---------+
| writer_hostgroup | reader_hostgroup | check_type | comment |
+------------------+------------------+------------+---------+
| 10               | 20               | read_only  | proxy   |
+------------------+------------------+------------+---------+
1 row in set (0.00 sec)

ProxySQL-Admin>  select * from main.runtime_mysql_replication_hostgroups;
Empty set (0.00 sec)

ProxySQL-Admin>  load mysql servers to runtime;
Query OK, 0 rows affected (0.00 sec)

ProxySQL-Admin>  select * from main.runtime_mysql_replication_hostgroups;
+------------------+------------------+------------+---------+
| writer_hostgroup | reader_hostgroup | check_type | comment |
+------------------+------------------+------------+---------+
| 10               | 20               | read_only  | proxy   |
+------------------+------------------+------------+---------+
1 row in set (0.00 sec)

ProxySQL-Admin>  use disk
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
ProxySQL-Admin>  select * from disk.mysql_replication_hostgroups;
Empty set (0.00 sec)

ProxySQL-Admin>  save mysql servers to disk
    -> ;
Query OK, 0 rows affected (0.02 sec)

ProxySQL-Admin>  select * from disk.mysql_replication_hostgroups;
+------------------+------------------+------------+---------+
| writer_hostgroup | reader_hostgroup | check_type | comment |
+------------------+------------------+------------+---------+
| 10               | 20               | read_only  | proxy   |
+------------------+------------------+------------+---------+
1 row in set (0.00 sec)

ProxySQL-Admin> 
```

**ProxySQL 会根据server 的read _only 的取值将服务器进行分组。 read_only=0 的server，master被分到编号为10的写组，read_only=1 的server，slave则被分到编号20的读组**

## 将个服务器添加到mysql_server表

```
ProxySQL-Admin>  show create table mysql_servers\G;
*************************** 1. row ***************************
       table: mysql_servers
Create Table: CREATE TABLE mysql_servers (
    hostgroup_id INT CHECK (hostgroup_id>=0) NOT NULL DEFAULT 0,
    hostname VARCHAR NOT NULL,
    port INT CHECK (port >= 0 AND port <= 65535) NOT NULL DEFAULT 3306,
    gtid_port INT CHECK ((gtid_port <> port OR gtid_port=0) AND gtid_port >= 0 AND gtid_port <= 65535) NOT NULL DEFAULT 0,
    status VARCHAR CHECK (UPPER(status) IN ('ONLINE','SHUNNED','OFFLINE_SOFT', 'OFFLINE_HARD')) NOT NULL DEFAULT 'ONLINE',
    weight INT CHECK (weight >= 0 AND weight <=10000000) NOT NULL DEFAULT 1,
    compression INT CHECK (compression IN(0,1)) NOT NULL DEFAULT 0,
    max_connections INT CHECK (max_connections >=0) NOT NULL DEFAULT 1000,
    max_replication_lag INT CHECK (max_replication_lag >= 0 AND max_replication_lag <= 126144000) NOT NULL DEFAULT 0,
    use_ssl INT CHECK (use_ssl IN(0,1)) NOT NULL DEFAULT 0,
    max_latency_ms INT UNSIGNED CHECK (max_latency_ms>=0) NOT NULL DEFAULT 0,
    comment VARCHAR NOT NULL DEFAULT '',
    PRIMARY KEY (hostgroup_id, hostname, port) )
1 row in set (0.00 sec)

ERROR: 
No query specified

```

使用下面的语句依次插入我们刚刚搭建好的的机器（注意写入组还是读取组的区分）

```
insert into mysql_servers(hostgroup_id,hostname,port,comment) values (10,'xxx.xxx.xxx.xxx',3306,"说明");
```

**将信息同步到runtime和disk**

```
load mysql servers to runtime;
save mysql servers to disk;
```

主要通过下面三句话精选验证

```
 select * from disk.mysql_servers;
 select * from runtime_mysql_servers;
 select * from main.mysql_servers;
```

注意观察` select * from mysql_servers;`时`status`列全为`ONLINE`即可。

```
 +--------------+-----------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------------------------+
| hostgroup_id | hostname        | port | gtid_port | status | weight | compression | max_connections | max_replication_lag | use_ssl | max_latency_ms | comment                   |
+--------------+-----------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------------------------+
| 10           | xxx.xxx.xxx.114 | 3306 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              | 第一次测试的主机          |
| 20           | xxx.xxx.xxx.118 | 3306 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              | 第一次测试的从机1         |
| 20           | xxx.xxx.xxx.115 | 3306 | 0         | ONLINE | 1      | 0           | 1000            | 0                   | 0       | 0              | 第一次测试的从机2         |
+--------------+-----------------+------+-----------+--------+--------+-------------+-----------------+---------------------+---------+----------------+---------------------------+
3 rows in set (0.00 sec)

```



## ProxySQL 监控后端节点

为 ProxySQL 配置监控账号

```

ProxySQL-Admin>  set mysql-monitor_username='monitor';
Query OK, 1 row affected (0.00 sec)

ProxySQL-Admin>  set mysql-monitor_password='Mm-123456';
Query OK, 1 row affected (0.00 sec)

ProxySQL-Admin>  load mysql variables to runtime;
Query OK, 0 rows affected (0.00 sec)

ProxySQL-Admin>  save mysql variables to disk;
Query OK, 136 rows affected (0.00 sec)

```

在 mysql主从 的master节点进行设置账号密码

```
mysql> create user 'monitor'@'%' identified by 'Mm-123456';
Query OK, 0 rows affected (0.00 sec)

mysql> grant all privileges on *.* to 'monitor'@'%';
Query OK, 0 rows affected (0.00 sec)

```

验证监控信息，ProxySQL 监控模块的指标都保存在`monitor库的log表中` 

```
连接：select * from monitor.mysql_server_connect_log;
稍等一会 结果出现 connect_error 列出现 NULL 即可
心跳检测ping：select * from mysql_server_ping_log limit 10;
readonly:select * from mysql_server_read_only_log limit 10;
```

## 配置proxy

SQL 请求所使用的用户配置，都需要再Mysql节点创建上。

​	创建proxysql 对外访问的账户(master)

```
mysql> create user 'proxysql'@'这里是proxy的ip' identified by 'Proxy-123456';
Query OK, 0 rows affected (0.01 sec)

mysql> grant all privileges on *.* to 'proxysql'@'这里是proxy的ip' with grant option;
Query OK, 0 rows affected (0.00 sec)

mysql> flush privileges;
Query OK, 0 rows affected (0.01 sec)

```

在ProxySQL中需要设置的表是

```
ProxySQL-Admin>  show create table mysql_users\G;
*************************** 1. row ***************************
       table: mysql_users
Create Table: CREATE TABLE mysql_users (
    username VARCHAR NOT NULL,
    password VARCHAR,
    active INT CHECK (active IN (0,1)) NOT NULL DEFAULT 1,
    use_ssl INT CHECK (use_ssl IN (0,1)) NOT NULL DEFAULT 0,
    default_hostgroup INT NOT NULL DEFAULT 0,  #默认路由目标
    default_schema VARCHAR,
    schema_locked INT CHECK (schema_locked IN (0,1)) NOT NULL DEFAULT 0,
    transaction_persistent INT CHECK (transaction_persistent IN (0,1)) NOT NULL DEFAULT 1,
    fast_forward INT CHECK (fast_forward IN (0,1)) NOT NULL DEFAULT 0,
    backend INT CHECK (backend IN (0,1)) NOT NULL DEFAULT 1,
    frontend INT CHECK (frontend IN (0,1)) NOT NULL DEFAULT 1,
    max_connections INT CHECK (max_connections >=0) NOT NULL DEFAULT 10000,
    comment VARCHAR NOT NULL DEFAULT '',
    PRIMARY KEY (username, backend),
    UNIQUE (username, frontend))
1 row in set (0.00 sec)

ERROR: 
No query specified

```

我们将在`master`上设置用于`proxy`访问的账户插入到`ProxySql`的`mysql_users`表中

```
ProxySQL-Admin>  insert into mysql_users(username,password,default_hostgroup) values('proxysql','Proxy-123456',10);
Query OK, 1 row affected (0.00 sec)
ProxySQL-Admin>  load mysql users to runtime;
Query OK, 0 rows affected (0.00 sec)

ProxySQL-Admin>  save mysql users to disk;
Query OK, 0 rows affected (0.01 sec)

```

```
ProxySQL-Admin>  select * from mysql_users\G;
*************************** 1. row ***************************
              username: proxysql
              password: Proxy-123456
                active: 1
               use_ssl: 0
     default_hostgroup: 10
        default_schema: NULL
         schema_locked: 0
transaction_persistent: 1
          fast_forward: 0
               backend: 1
              frontend: 1
       max_connections: 10000
               comment: 
1 row in set (0.00 sec)

ERROR: 
No query specified

```

此时可以在有权限登录的机器用proxysql账户登录创建库调试看看。

忘记自己登录到那个服务可以用` select @@server_id;`查看所在机器。

### 配置读写分离

#### 路由规则

路由规则配置非常灵活，可以基于schema，也可以基于用户，或单个sql语句实现路由定制。

【**实际过程中应该从手机的慢日志各项指标中找出压力大，执行频繁的语句单独写路由规则和缓存，建议使用hash code 值做读写纹理，不要建太多规则**】

mysql_query_rules_fast_routing是mysql_query_rules

介绍一下改表mysql_query_rules的几个字段： 
`active`：是否启用这个规则，1表示启用，0表示禁用 
`match_pattern` 字段就是代表设置规则 
`destination_hostgroup` 字段代表默认指定的分组， 
`apply` 代表真正执行应用规则。

#### 规则的写入

ProxySQL 是根据`rule_id`的顺序来进行规则匹配的注意规则的编写顺序

所有以select 开头的语句全部分配到读组中，上面建立读组编号是20 

`insert into mysql_query_rules(rule_id,active,match_pattern,destination_hostgroup,apply) values (2,1,'^select',20,1);`

说有以insert开头的语句全部分配到写组中

`insert into mysql_query_rules(rule_id,active,match_pattern,destination_hostgroup,apply) values (1,1,'^SELECT.*FOR UPDATE$',10,1);`

`insert into mysql_query_rules(rule_id,active,match_pattern,destination_hostgroup,apply) values (3,1,'^insert',10,1);`

注意要

```
load mysql query rules to runtime;
save mysql query rules to disk;
```

#### 读写分离测试

在ProxySQL的主机上运行

##### 测试读

```
mysql -uproxysql -pProxy-123456 -h 127.0.0.1 -P 6033 -e "select @@server_id";
可以返回正确的server_id即可，因为我的后台是俩台只读机器，所以会交替出现访问俩台机器的server_id;


mysql> select @@server_id for update;  此语句SELECT...FOR UPDATE 会申请写锁，所以用在这里比较合适。
+-------------+
| @@server_id |
+-------------+
|         114 |
+-------------+
1 row in set (0.00 sec)

```

##### 测试写

```
mysql -uproxysql -pProxy-123456 -h 127.0.0.1 -P 6033 登录后
mysql> begin;
Query OK, 0 rows affected (0.00 sec)

mysql> insert into hanzhongtest.edu_user values(9,"测试写3","wahah");
Query OK, 1 row affected (0.00 sec)

mysql> select @@server_id;
+-------------+
| @@server_id |
+-------------+
|         114 |
+-------------+
1 row in set (0.00 sec)

mysql> commit;
Query OK, 0 rows affected (0.00 sec)


```

#### 查看路由信息

使用admin账号登录ProxySQL

```
[root@localhost proxysql]# mysql -u admin -padmin -h 127.0.0.1 -P 6032 --prompt='ProxySQL-Admin>  '


ProxySQL-Admin>  SELECT hostgroup hg,
    ->               sum_time,
    ->               count_star,
    ->               digest_text 
    ->        FROM stats_mysql_query_digest
    ->        ORDER BY sum_time DESC;
+----+----------+------------+-------------------------------------------------------+
| hg | sum_time | count_star | digest_text                                           |
+----+----------+------------+-------------------------------------------------------+
| 20 | 83455    | 70         | select @@server_id                                    |
| 10 | 4383     | 2          | commit                                                |
| 10 | 1275     | 2          | insert into hanzhongtest.edu_user values(?,?,?)       |
| 10 | 1205     | 1          | start transantion                                     |
| 20 | 1015     | 1          | select * from stats_mysql_query_digest                |
| 10 | 798      | 2          | begin insert into hanzhongtest.edu_user values(?,?,?) |
| 10 | 792      | 2          | begin                                                 |
| 10 | 454      | 1          | select @@server_id                                    |
| 10 | 0        | 71         | select @@version_comment limit ?                      |
+----+----------+------------+-------------------------------------------------------+
9 rows in set (0.00 sec)


```

### 总结：

路由规则比较灵活需要多看[官方文档](https://github.com/sysown/proxysql/wiki)

利用ProxySQL读写分离可以降低主库的压力，但是当主库宕机后集群将不能再写。所以需要replication manager实现高可用的协助。

其次需要实验对于`mysql galera cluster`的读写分离和负载。

#### 遇到的问题

1.建立账户是访问IP应明确

2.firewalld 实验的时候要不关掉，要不写规则开放端口。

## ProxySql 的高可用

| 编写人 | 编写时间  | 编写说明                                                     |
| ------ | --------- | ------------------------------------------------------------ |
| 章瀚中 | 2020-9-12 | 接上一篇[proxysql 读写分离]([ProxySQL 读写分离](http://wiki-private.capitalonline.net:8090/pages/viewpage.action?pageId=59474862))一起观看较佳 |

有两个组件实现了ProxySQL集群

- **monitoring(集群监控组件)**

- **re-configuration(远程配置)**

  俩组件都有4张表

  **-** mysql_query_rules
  **-** mysql_servers
  **-** mysql_users
  **-** proxysql_servers

### 相关变量

Admin variables ：使用`load admin variables to runtime`使其生效

#### 1.用于同步的变量

	- `admin-checksum_mysql_query_rules` boolean 变量，默认ture，当使用`load mysql query rules to runtime`时，会生成新的配置校验码。如果是false 新的配置不会自动广播和同步远端数据。
	- `admin-checksum_mysql_servers` 布尔类型，默认ture `load mysql servers to runtime` 同上
	- `admin-chechsum_mysql_users`布尔类型，默认ture `load mysql users to runtime`,同上，但是用户有数百万，为了不影响性能就要设置为false。

#### 2.集群凭证变量

- `admin-cluster_username`
- `admin-cluser_password`

这是用于监控其他ProxySQL实例 ，这俩个变量需要再`admin-admin_credentials`中存在。

#### 3.检查频率和间隔时间

- `admin-cluster_check_interval_ms`这个变量定义了检查chechsums的时间，默认1000,最小10，最大 300000.
- `admin-cluster_check_status_frequency`定义了多少次checksums检查就验证一次连接性。

#### 4.将配置同步到磁盘

在远程同步配置之后，通常最好的做法是立即将新的更改保存到磁盘。这样重启时，更改的配置不会丢失。
**-** `admin-cluster_mysql_query_rules_save_to_disk`
**-** `admin-cluster_mysql_servers_save_to_disk`
**-** `admin-cluster_mysql_users_save_to_disk`
**-** `admin-cluster_proxysql_servers_save_to_disk`

这几个变量都是布尔值。当设置为true(默认值)时，在远程同步并load到runtime后，新的mysql_query_rules、mysql_servers、mysql_users、proxysql_servers配置会持久化到磁盘中。

#### 5.是否需要远程同步变量

由于某些原因，可能多个ProxySQL实例会在同一时间进行重新配置。
例如，每个ProxySQL实例都在监控MySQL的replication，且自动探测到MySQL的故障转移，在一个极短的时间内(可能小于1秒)，这些ProxySQL实例可能会自动调整新的配置，而无需通过其它ProxySQL实例来同步新配置。

类似的还有，当所有ProxySQL实例都探测到了和某实例的临时的网络问题，或者某个MySQL节点比较慢(replication lag,  拖后腿)，这些ProxySQL实例都会自动地避开这些节点。这时各ProxySQL实例也无需从其它节点处同步配置，而是同时自动完成新的配置。

配置ProxySQL集群，让各ProxySQL实例暂时无需从其它实例处同步某些配置，而是等待一定次数的检查之后，再触发远程同步。但是，如果本地和远程节点的这些变量阈值不同，则还是会触发远程同步。

**-** `admin-cluster_mysql_query_rules_diffs_before_sync`
**-** `admin-cluster_mysql_servers_diffs_before_sync`
**-** `admin-cluster_mysql_users_diffs_before_sync`
**-** `admin-cluster_proxysql_servers_diffs_before_sync`

分别定义经过多少次的"无法匹配"检查之后，触发mysql_query_rules、mysql_servers、mysql_users、proxysql_servers配置的远程同步。默认值3次，最小值0，表示永不远程同步，最大值1000。

比如各实例监控mysql_servers配置并做校验码检查，如果某实例和本地配置不同，当多次检测到都不同时，将根据load to runtime的时间戳决定是否要从远端将mysql_servers同步到本地。

#### 6.延迟同步
ProxySQL Cluster 可以定义达到多少个checksum 不同之后，才在集群内部进行配置同步。
query rules, servers, users 和proxysql servers 分别有admin-cluster_XXX_diffs_before_sync 相关的参数，取值范围0 ～ 1000，0 代表从不同步。默认3。

### 需要配置的表

#### 1.proxysql_servers

定义proxyssql集群中有哪些实例，可以在线insert，也可以修改配置文件。

```
ProxySQL-Admin>  show create table proxysql_servers;
+------------------+--------------------------
| table            | Create Table            |
+------------------+--------------------------
| proxysql_servers | CREATE TABLE proxysql_servers (
    hostname VARCHAR NOT NULL,
    port INT NOT NULL DEFAULT 6032,
    weight INT CHECK (weight >= 0) NOT NULL DEFAULT 0,
    comment VARCHAR NOT NULL DEFAULT '',
    PRIMARY KEY (hostname, port) ) |
+------------------+--------------------------
1 row in set (0.00 sec)
```

ProxySQL只有在磁盘数据库文件不存在，或者使用了--initial选项时才会读取传统配置文件。

#### 2.runtime_proxysql_servers

正如其它runtime_表一样，runtime_proxysql_servers表和proxysql_servers的结构完全一致，只不过它是runtime数据结构中的配置，也就是当前正在生效的配置。该表的定义语句如下：

```
| runtime_proxysql_servers | CREATE TABLE runtime_proxysql_servers (
    hostname VARCHAR NOT NULL,
    port INT NOT NULL DEFAULT 6032,
    weight INT CHECK (weight >= 0) NOT NULL DEFAULT 0,
    comment VARCHAR NOT NULL DEFAULT '',
    PRIMARY KEY (hostname, port) ) |

```

#### 3.runtime_checksums_values

这个不是基于内存的。用于 load to runtime时的信息。

```
| runtime_checksums_values | CREATE TABLE runtime_checksums_values (
    name VARCHAR NOT NULL,
    version INT NOT NULL,
    epoch INT NOT NULL,
    checksum VARCHAR NOT NULL,
    PRIMARY KEY (name)) |
```

\- name：模块的名称
\- version：执行了多少次load to runtime操作，包括所有隐式和显式执行的(某些事件会导致ProxySQL内部自动执行load to runtime命令)
\- epoch：最近一次执行load to runtime的时间戳
\- checksum：执行load to runtime时生成的配置校验码(checksum)

**特别注意：**  目前6个组件中只有4种模块的配置会生成对应的校验码(checksum)，不能生成的组件是：admin_variables，mysql_variables。 checnsum 只有在执行了load to run ，并且admin-checksum_XXX = true 时，才可以正常生成。即:
\- LOAD MYSQL QUERY RULES TO RUNTIME：当admin-checksum_mysql_query_rules=true时生成一个新的mysql_query_rules配置校验码
\- LOAD MYSQL SERVERS TO RUNTIME：当admin-checksum_mysql_servers=true时生成一个新的mysql_servers配置校验码
\- LOAD MYSQL USERS TO RUNTIME：当admin-checksum_mysql_users=true时生成一个新的mysql_users配置校验码
\- LOAD PROXYSQL SERVERS TO RUNTIME：总是会生成一个新的proxysql_servers配置校验码
\- LOAD ADMIN VARIABLES TO RUNTIME：不生成校验码
\- LOAD MYSQL VARIABLES TO RUNTIME：不生产校验码

**New commands （新命令）:**
\- LOAD PROXYSQL SERVERS FROM MEMORY / LOAD PROXYSQL SERVERS TO RUNTIME
从内存数据库中加载proxysql servers配置到runtime数据结构
\- SAVE PROXYSQL SERVERS TO MEMORY / SAVE PROXYSQL SERVERS FROM RUNTIME
将proxysql servers配置从runtime数据结构持久化保存到内存数据库中
\- LOAD PROXYSQL SERVERS TO MEMORY / LOAD PROXYSQL SERVERS FROM DISK
从磁盘数据库中加载proxysql servers配置到内存数据库中
\- LOAD PROXYSQL SERVERS FROM CONFIG
从传统配置文件中加载proxysql servers配置到内存数据库中
\- SAVE PROXYSQL SERVERS FROM MEMORY / SAVE PROXYSQL SERVERS TO DISK
将proxysql servers配置从内存数据库中持久化保存到磁盘数据库中

### stats tables

#### 1.stats_proxysql_servers_checksums

记录集群中各个实例的组件checksum 信息。

```
 show create table stats.stats_proxysql_servers_checksums;
| stats_proxysql_servers_checksums | CREATE TABLE stats_proxysql_servers_checksums (
    hostname VARCHAR NOT NULL,
    port INT NOT NULL DEFAULT 6032,
    name VARCHAR NOT NULL,
    version INT NOT NULL,
    epoch INT NOT NULL,
    checksum VARCHAR NOT NULL,
    changed_at INT NOT NULL,
    updated_at INT NOT NULL,
    diff_check INT NOT NULL,
    PRIMARY KEY (hostname, port, name) ) |

```

各字段意义如下：
\- hostname：ProxySQL实例的主机名
\- port：ProxySQL实例的端口
\- name：对端runtime_checksums_values中报告的模块名称
\- version：对端runtime_checksum_values中报告的checksum的版本
**注意**，ProxySQL实例刚启动时version=1：ProxySQL实例将永远不会从version=1的实例处同步配置数据，因为一个刚刚启动的ProxyQL实例不太可能是真相的来源，这可以防止新的连接节点破坏当前集群配置
\- epoch：对端runtime_checksums_values中报告的checksum的时间戳epoch值
\- checksum：对端runtime_checksums_values中报告的checksum值
\- changed_at：探测到checksum发生变化的时间戳
\- updated_at：最近一次更新该类配置的时间戳
\- diff_check：一个计数器，用于记录探测到的对端和本地checksum值已有多少次不同
需要等待达到阈值后，才会触发重新配置。前面已经说明，在多个ProxySQL实例同时或极短时间内同时更改配置时，可以让ProxySQL等待多次探测之后再决定是否从远端同步配置。这个字段正是用于记录探测到的配置不同次数。如果diff_checks不断增加却仍未触发同步操作，这意味着对端不是可信任的同步源，例如对端的version=1。另一方面，如果某对端节点不和ProxySQL集群中的其它实例进行配置同步，这意味着集群没有可信任的同步源。这种情况可能是因为集群中所有实例启动时的配置都不一样，它们无法自动判断哪个配置才是正确的。可以在某个节点上执行load to runtime，使该节点被选举为该类配置的可信任同步源。

#### 2.**stats_proxysql_servers_metrics 表**

```
show create table stats.stats_proxysql_servers_metrics;
| stats_proxysql_servers_metrics | CREATE TABLE stats_proxysql_servers_metrics (
    hostname VARCHAR NOT NULL,
    port INT NOT NULL DEFAULT 6032,
    weight INT CHECK (weight >= 0) NOT NULL DEFAULT 0,
    comment VARCHAR NOT NULL DEFAULT '',
    response_time_ms INT NOT NULL,
    Uptime_s INT NOT NULL,
    last_check_ms INT NOT NULL,
    Queries INT NOT NULL,
    Client_Connections_connected INT NOT NULL,
    Client_Connections_created INT NOT NULL,
    PRIMARY KEY (hostname, port) ) |

```

当执行show mysql status语句时，显示一些已检索到的指标。字段意义如下：
\- hostname：ProxySQL实例主机名
\- port：ProxySQL实例端口
\- weight：报告结果同proxysql_servers.weight字段
\- comment：报告结果同proxysql_servers.comment字段
\- response_time_ms：执行show mysql status的响应时长，单位毫秒
\- Uptime_s：ProxySQL实例的uptime，单位秒
\- last_check_ms：最近一次执行check距现在已多久，单位毫秒
\- Queries：该实例已执行query的数量
\- Client_Connections_connected：number of client's connections connected
\- Client_Connections_created：number of client's connections created
**注意：**当前这些状态只为debug目的，但未来可能会作为远程实例的健康指标。

#### 3.stats_proxysql_servers_status

 Currently unused – this table was created to show general statistics related to all the services configured in the `proxysql_servers` table.

```
Admin> show create table stats.stats_proxysql_servers_status\G
*************************** 1. row ***************************
       table: stats_proxysql_servers_status
Create Table: CREATE TABLE stats_proxysql_servers_status (
    hostname VARCHAR NOT NULL,
    port INT NOT NULL DEFAULT 6032,
    weight INT CHECK (weight >= 0) NOT NULL DEFAULT 0,
    master VARCHAR NOT NULL,
    global_version INT NOT NULL,
    check_age_us INT NOT NULL,
    ping_time_us INT NOT NULL, checks_OK INT NOT NULL,
    checks_ERR INT NOT NULL,
    PRIMARY KEY (hostname, port) )
1 row in set (0.00 sec)
```

#### Re-configuration

因为集群间，所有节点都是相互监控的。每个ProxySQL节点都监控集群中的其它实例，它们可以快速探测到某个实例的配置是否发生改变。如果某实例的配置发生改变，其它实例会检查这个配置和自身的配置是否相同，因为其它节点的配置可能和本节点的配置同时(或在极短时间差范围)发生了改变。

由于相互监控，所以当配置发生变动时，它们可以立即发现。当其他节点的配置发生变动时，本节点会先去检查一次它自身的配置，因为有可能remote instance 和local instance 同时发生配置变动。
如果比较结果不同：
**-** 如果它们自身的 version = 1，就去找集群内从version > 1的节点处找出epoch最大值的节点，并从该节点拉取配置应用到本地，并立即同步。
**-** 如果version >1， 该节点开始统计和其他节点间的differ 数。即开始对探测到的不同配置进行计数。
  **-**  当 differ 大于 cluster__name___diffs_before_sync ，  并且cluster__name__diffs_before_sync > 0， 就去找集群内 version >1， 并且epoch  最高的节点，并立即同步。也就是说当探测到不同配置的次数超过cluster_name_diffs_before_sync，且cluster_name_diffs_before_sync大于0时，找出version > 1且epoch值最大的节点，并从该节点拉取配置禁用应用。

同步配置的过程如下：
**-** 用于健康检查的连接，也用来执行一系列类似于select _list_of_columns from runtime_module的select语句。例如：

```
SELECT hostgroup_id, ``hostname``, port, status, weight, compression, max_connections,  max_replication_lag, use_ssl, max_latency_ms, comment FROM  runtime_mysql_servers;``SELECT writer_hostgroup, reader_hostgroup, comment FROM runtime_mysql_replication_hostgroups;
```

删除本地配置。例如：

```
DELETE FROM mysql_servers;DELETE FROM mysql_replication_hostgroups;
```

**-** 向本地配置表中插入已从远端节点检索到的新配置。
**-** 在内部执行LOAD module_name TO RUNTIME：这会递增版本号，并创建一个新的checksum。
**-** 如果cluster_name_save_to_disk=true，再在内部执行SAVE module_name TO DISK。

参考文档：https://www.cnblogs.com/kevingrace/p/10411457.html

#### 进行配置

要使用的VIP： 10.241.10.3

| 机器            | 角色                       | 内网IP      |
| --------------- | -------------------------- | ----------- |
| xxx.xxx.xxx.117 | 原来搭建好的proxysql       | 10.241.10.1 |
| xxx.xxx.xxx.114 | 要新添加组建集群的proxysql | 10.241.10.2 |

写配置文件和实时修改都可以，记得

`load *** to runtime`、`save *** to disk`

**1.规定集群信息（在117上）**

```
ProxySQL-Admin>  set  admin-admin_credentials="admin:admin;cluster_name:123456";
Query OK, 1 row affected (0.00 sec)
ProxySQL-Admin>  set admin-cluster_username="cluster_name";
Query OK, 1 row affected (0.00 sec)
ProxySQL-Admin>  set admin-cluster_password="123456";
Query OK, 1 row affected (0.00 sec)
ProxySQL-Admin>  set admin-cluster_check_interval_ms=200;
Query OK, 1 row affected (0.01 sec)
ProxySQL-Admin>  set admin-cluster_check_status_frequency=100;
Query OK, 1 row affected (0.00 sec)


ProxySQL-Admin>  load admin variables to runtime;
Query OK, 0 rows affected (0.00 sec)

ProxySQL-Admin>  save admin variables  to disk;
Query OK, 35 rows affected (0.01 sec)

```

`select * from global_variables;`可以查看修改结果

**2.写入节点信息**

```
ProxySQL-Admin>  show create table proxysql_servers;
| table            | Create Table                                                         
| proxysql_servers | CREATE TABLE proxysql_servers (
    hostname VARCHAR NOT NULL,
    port INT NOT NULL DEFAULT 6032,
    weight INT CHECK (weight >= 0) NOT NULL DEFAULT 0,
    comment VARCHAR NOT NULL DEFAULT '',
    PRIMARY KEY (hostname, port) ) |
1 row in set (0.00 sec)
```

写入节点

```

ProxySQL-Admin>  insert into proxysql_servers(hostname,port)values('xxx.xxx.xxx.114',6032);
Query OK, 1 row affected (0.00 sec)

ProxySQL-Admin>  insert into proxysql_servers(hostname,port)values('xxx.xxx.xxx.117',6032);
Query OK, 1 row affected (0.00 sec)

ProxySQL-Admin>  load proxysql servers to runtime;
Query OK, 0 rows affected (0.00 sec)

ProxySQL-Admin>  save proxysql servers to disk;
Query OK, 0 rows affected (0.01 sec)


```

为了练习 在另一个节点我们使用配置文件进行修改，值的注意的是

```
这里要特别注意：
如果存在如果存在``"proxysql.db"``文件(在``/var/lib/proxysql``目录下)，则ProxySQL服务只有在第一次启动时才会去读取proxysql.cnf文件并解析；后面启动会就不会读取proxysql.cnf文件了！如果想要让proxysql.cnf文件里的配置在重启proxysql服务后生效(即想要让proxysql重启时读取并解析proxysql.cnf配置文件)，则需要先删除``/var/lib/proxysql/proxysql``.db数据库文件，然后再重启proxysql服务。这样就相当于初始化启动proxysql服务了，会再次生产一个纯净的proxysql.db数据库文件(如果之前配置了proxysql相关路由规则等，则就会被抹掉)。
```

```
[root@localhost ~]# cp /etc/proxysql.cnf /etc/proxysql.cnf.bak
[root@localhost ~]# vim /etc/proxysql.cnf

admin_variables=
{
        admin_credentials="admin:admin;cluster_name:123456"
#       mysql_ifaces="127.0.0.1:6032;/tmp/proxysql_admin.sock"
        mysql_ifaces="0.0.0.0:6032"
#       refresh_interval=2000
#       debug=true
        cluster_username="cluster_name"                                  
        cluster_password="123456"
        cluster_check_interval_ms=2000
        cluster_check_status_frequency=1000
}
proxysql_servers =
(
        {
        hostname="xxx.xxx.xxx.114"
        port=6032
        },
        {
        hostname="xxx.xxx.xxx.117"
        port=6032

        }
)


```

注意里面的圆括号

```
[root@localhost ~]# systemctl restart proxysql
[root@localhost ~]# ll  /var/lib/proxysql/proxysql.db 
-rw------- 1 proxysql proxysql 196608 Oct 20 11:11 /var/lib/proxysql/proxysql.db
```

#### 观察集群状况

```
[root@localhost ~]# mysql -uadmin -padmin -h127.0.0.1 -P6032
MySQL [(none)]> select * from stats_proxysql_servers_metrics; 
+-----------------+------+--------+---------+------------------+----------+---------------+---------+-------------------
| hostname        | port | weight | comment | response_time_ms | Uptime_s | last_check_ms | Queries | Client_Connections
+-----------------+------+--------+---------+------------------+----------+---------------+---------+-------------------
| xxx.xxx.xxx.114  | 6032 | 0      |         | 0                | 0        | 676276571     | 0       | 0                 
| xxx.xxx.xxx.117 | 6032 | 0      |         | 0                | 0        | 676276571     | 0       | 0                 
+-----------------+------+--------+---------+------------------+----------+---------------+---------+-------------------
2 rows in set (0.00 sec)

MySQL [(none)]> select * from proxysql_servers;
+-----------------+------+--------+---------+
| hostname        | port | weight | comment |
+-----------------+------+--------+---------+
| xxx.xxx.xxx.114  | 6032 | 0      |         |
| xxx.xxx.xxx.117 | 6032 | 0      |         |
+-----------------+------+--------+---------+
2 rows in set (0.00 sec)

mysql> select hostname,name,checksum,updated_at from stats_proxysql_servers_checksums;
+-----------------+-------------------+--------------------+------------+
| hostname        | name              | checksum           | updated_at |
+-----------------+-------------------+--------------------+------------+
| xxx.xxx.xxx.117 | admin_variables   |                    | 1603245278 |
| xxx.xxx.xxx.117 | mysql_query_rules | 0xD331FBA2DFADB529 | 1603245278 |
| xxx.xxx.xxx.117 | mysql_servers     | 0xF38804B1395AE050 | 1603245278 |
| xxx.xxx.xxx.117 | mysql_users       | 0xA9B3797834E823FC | 1603245278 |
| xxx.xxx.xxx.117 | mysql_variables   |                    | 1603245278 |
| xxx.xxx.xxx.117 | proxysql_servers  | 0x66124755F5959C8D | 1603245278 |
| xxx.xxx.xxx.114 | admin_variables   |                    | 1603245278 |
| xxx.xxx.xxx.114 | mysql_query_rules | 0xD331FBA2DFADB529 | 1603245278 |
| xxx.xxx.xxx.114 | mysql_servers     | 0xF38804B1395AE050 | 1603245278 |
| xxx.xxx.xxx.114 | mysql_users       | 0xA9B3797834E823FC | 1603245278 |
| xxx.xxx.xxx.114 | mysql_variables   |                    | 1603245278 |
| xxx.xxx.xxx.114 | proxysql_servers  | 0x66124755F5959C8D | 1603245278 |

12 rows in set (0.00 sec)

```

此后多看日志 `tail -f /var/lib/proxysql/proxysql.log`, 这里会提示你需要做哪些操作或是修改变量来进行同步。注意proxysql 账号和monitor账号在mysql机器上是否有限制。

通过调试使得proxysql.log无报错后。

通过`select * from mysql_servers;`观察数据时候同步

下面进行高可用配置。

#### Proxysql的高可用

使用keepalived,

### Keepalived 官方网站 https://www.keepalived.org/

基于（Vitual Router Redundancy Protocol，虚拟路由冗余协议）,VRRP是为了解决静态路由的高可用。VRRP的基本架构。利用VIP资源漂移来实现ProxySQL双节点的无感知故障切换，即对外提供一个统一的vip地址，并且在keepalived.conf文件中配置proxysql服务的监控脚本，当宕机或proxysql服务挂掉时就将vip资源漂移到另一个正常的节点上，从而使proxysql的代理层持续无感应地提供服务。

#### VRRP的基本架构
 虚拟路由器由多个路由器组成，每个路由器都有各自的IP和共同的VRID(0-255)，其中一个VRRP路由器通过竞选成为MASTER，占有VIP，对外提供路由服务，其他成为BACKUP，MASTER以IP组播（组播地址：224.0.0.18）形式发送VRRP协议包，与BACKUP保持心跳连接，若MASTER不可用（或BACKUP接收不到VRRP协议包），则BACKUP通过竞选产生新的MASTER并继续对外提供路由服务，从而实现高可用。
链接：https://www.jianshu.com/p/b050d8861fc1

#### 工作类型

```undefined
抢占式：当出现比现有主服务器优先级高的服务器时，会发送通告抢占角色成为主服务器
非抢占式：
```

核心组件

```
        vrrp stack：vrrp协议的实现
        ipvs wrapper：为集群内的所有节点生成IPVS规则
        checkers：对IPVS集群的各RS做健康状态检测
        控制组件：配置文件分析器，用来实现配置文件的分析和加载
        IO复用器
        内存管理组件，用来管理keepalived高可用是的内存管理
```

**注意：各节点时间需要同步**

##### 下载和安装

```
 wget https://www.keepalived.org/software/keepalived-2.1.5.tar.gz
 tar -xvf keepalived-2.1.5.tar.gz 
 cd keepalived-2.1.5
 ./configure --prefix=/usr/local/keepalived
 yum -y install openssl
 make && make install
```

##### 基本配置

```
# tree -l /usr/local/keepalived/etc
-- keepalived
|   |-- keepalived.conf
|   `-- samples #这个里面是例子，先不管他
`-- sysconfig
    `-- keepalived
# mkdir /etc/keepalived
#  cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf
# cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/keepalived
# chkconfig keepalived on
# service keepalived start   #启动服务
# service keepalived stop    #停止服务
# service keepalived restart #重启服务
```

使用`service keepalived start`命令启动服务时，默认会将`/etc/sysconfig/keepalived`文件中`KEEPALIVED_OPTIONS`参数作为`keepalived`服务启动时的参数，并从`/etc/keepalived/`目录下加载keepalived.conf配置文件，或用-f参数指定配置文件的位置。

##### 进行配置文件的书写

117 作为msater

```
! Configuration File for keepalived

global_defs {
   script_user root
   enable_script_security 
   router_id Master-proxysql #标识本节点的名称，通常为hostname
}
## keepalived会定时执行脚本并对脚本执行的结果进行分析,动态调整vrrp_instance的优先级。
##如果脚本执行结果为0,并且weight配置的值大于0,则优先级相应的增加。如果脚本执行结果非0,
##并且weight配置的值小于 0,则优先级相应的减少。其他情况,维持原本配置的优先级,即配置文件中priority对应的值。
vrrp_script chk_proxysql {
	script "/etc/keepalived/sql_check.sh"
	interval 2 #每2秒检测一次nginx的运行状态
	weight -20 #失败一次，将自己的优先级-20
}
vrrp_instance VI_1 {
    state MASTER  # 状态，主节点为MASTER，备份节点为BACKUP
    interface eth1  # 绑定VIP的网络接口，通过ip a查看自己的网络接口
    virtual_router_id 123 # 虚拟路由的ID号,两个节点设置必须一样,可选IP最后一段使用,相同的VRID为一个组,他将决定多播的MAC地址
    mcast_src_ip 10.241.10.1  # 本机IP地址
    priority 110 # 节点优先级，值范围0～254，MASTER要比BACKUP高
    advert_int 1  # 组播信息发送时间间隔，两个节点必须设置一样，默认为1秒
    # 设置验证信息，两个节点必须一致
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    # 虚拟IP，两个节点设置必须一样。可以设置多个，一行写一个
    virtual_ipaddress {
    	10.241.10.3
	}
    track_script {
		chk_proxysql	# nginx存活状态检测脚本		
	}
}
```



114为BACKUP

```
! Configuration File for keepalived

global_defs {
   script_user root
   enable_script_security 
   router_id Master-proxysql
}

vrrp_script chk_proxysql {
	script "/etc/keepalived/sql_check.sh"
	interval 2
	weight -20
}
vrrp_instance VI_1 {
    state BACKUP
    interface eth1
    virtual_router_id 123
    mcast_src_ip 10.241.10.2
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
    	10.241.10.3
	}
    track_script {
	chk_proxysql			
	}
}

```

检查proxysql的脚本 `cat /etc/keepalived/sql_check.sh`

```
#!/bin/bash
echo '我开始运行啦'
num=`ps aux | grep   proxysql | grep -cv grep`
echo ‘记录好了’ $num
if [ $num -eq 0 ];then
        #这里忽略重启
	echo '进入判断'
        sleep 1
        if [ `ps aux | grep   proxysql | grep -cv grep` -eq 0 ];then
                echo '进入小判断'
		echo 'stop keepalived' > /etc/keepalived/chk_log
		kill -9 $(ps aux | grep keepalived | grep -v grep | cut  -d' ' -f 7)
		echo 'stop !' > /etc/keepalived/chk_log
        fi
fi


```

**记得给脚本执行权限**`chmod a+x /etc/keepalived/proxysql_check.sh `

启动keepalived后`ip a`查看

keepalived-master端

```
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:ab:d7:be brd ff:ff:ff:ff:ff:ff
    inet 10.241.10.1/16 brd 10.241.255.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet 10.241.10.3/32 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:feab:d7be/64 scope link 
       valid_lft forever preferred_lft forever

```

keepalived-slave端

```
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:50:56:ab:4a:16 brd ff:ff:ff:ff:ff:ff
    inet 10.241.10.2/16 brd 10.241.255.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:feab:4a16/64 scope link 
       valid_lft forever preferred_lft forever
```

### 检查

【将主机proxysql关闭，会发现主机上的keepalived关闭，`vip:10.241.10.3` 飘移到了从机器】

使用飘移VIP连接MYSQL 

在读写期间进行从机切换新建`test`数据库 `test`数据表

```
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| dbtest             |
| hanzhongtest       |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
6 rows in set (0.00 sec)
mysql> create database proxysqlHA;
Query OK, 1 row affected (0.01 sec)
mysql> use proxysqlHA;
Database changed
mysql> create table test_id(id int,name varchar(10));
Query OK, 0 rows affected (0.01 sec)
```

在python写入时，进行切换

```
import pymysql
config = {
'host' : '10.241.10.3',
'db' : 'proxysqlHA',
'user' : 'proxysql',
'password' : 'Proxy-123456',
'port' : 6033,
}
for i in range(1,1000000):
insertstr = "insert into test_id values(%s,'zhangsan%s')" % (i,str(i),i)
print(insertstr)
sqlstr = insertstr
db = pymysql.connect(**config)
cursor = db.cursor()
cursor.execute(sqlstr)
db.commit()
print("count: {}".format(i) )
db.close()
```

在开始写入后关闭117主机的proxysql，观察发现确实在切换过程的一瞬间会出现写入失败。切换完成后访问正常。

### 遇到的问题

1.忘记将检查脚本弄成执行权限

2.脚本的名字完全包含了proxysql 导致检查出错。

3.切换时间包括 keepalived运行脚本的间隔时间、检查脚本的sleep时间、组播信息间隔的时间。想要最小化时间需要将所有间隔和sleep减去。

但是组播信息间隔时间`vrrp version 2` 中最小只能到1s 默认启动`vrrp version 2`



----





