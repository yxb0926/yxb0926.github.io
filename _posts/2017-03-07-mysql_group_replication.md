---
layout: post
title: MySQL Group Replication 初体验
date: 2017-03-07
---
  mysql 5.7.17版本的发布，标志着Group Replication功能的正式发布。Group Replication的引入是为了提高可用性，保证数据0丢失，多主复制，多点写入。让我们来体验下这个新特性。

## 环境搭建
以下将会分别介绍2种情况下的配置方法，即：“全新环境”和“现有传统复制集群”配置的方法。

### 全新环境配置
先安装官方5.7.17版本的mysql，安装方法略过。
为了后面压测性能，我选择了使用3台物理服务器来配置，每台服务器安装一个mysqld实例。  
- 实例列表
    + bbackdb01:3308
    + bbackdb05:3307
    + bbackdb11:3306

my.cnf的配置，除了常规的一些配置外，replication相关的部分需要注意的一些事情。
具体参考:http://mysqlhighavailability.com/mysqlha/gr/doc/limitations.html

以下是开启Group Replication的必要条件。
<pre>
<code>
  gtid_mode=ON  
  enforce_gtid_consistency=ON
  master_info_repository=TABLE
  relay_log_info_repository=TABLE
  binlog_checksum=NONE
  log-slave-updates=ON
  log-bin=/home/mysql/mysql_[port]/logs/mysql-bin
  binlog-format=ROW
</code>
</pre>

以下是group replication本身的配置：
<pre>
<code>
  transaction_write_set_extraction=XXHASH64
  loose-group_replication_group_name="aaaaaaaa-aaaa-aaaa-aaaa-aaaaaaaaaaaa"
  loose-group_replication_start_on_boot=off
  loose-group_replication_local_address= "10.10.10.51:33080"
  loose-group_replication_group_seeds= "10.10.10.51:33080,10.10.30.201:33070,10.10.30.207:33060"
  loose-group_replication_bootstrap_group= off
  loose-group_replication_single_primary_mode=FALSE
  loose-group_replication_enforce_update_everywhere_checks= TRUE
</code>
</pre>

启动实例：
<pre>
<code>
  systemctl start mysqld_3308
</code>
</pre>

登录mysql后创建group replication所需用户：
<pre>
<code>

SET SQL_LOG_BIN=0;
CREATE USER slave@'10.%';
GRANT REPLICATION SLAVE ON *.* TO slave@'10.%' IDENTIFIED BY 'slave123';
FLUSH PRIVILEGES;
SET SQL_LOG_BIN=1;
CHANGE MASTER TO MASTER_USER='slave', MASTER_PASSWORD='slave123' FOR CHANNEL 'group_replication_recovery';

</code>
</pre>

开启group replication
<pre>
<code>

# 安装group replication插件
INSTALL PLUGIN group_replication SONAME 'group_replication.so';

# 启动group replication
SET GLOBAL group_replication_bootstrap_group=ON;
START GROUP_REPLICATION;
SET GLOBAL group_replication_bootstrap_group=OFF;

#验证group replication是否启动成功，如果看到以下内容，则表示启动成功

mysql> SELECT * FROM performance_schema.replication_group_members;
+---------------------------+--------------------------------------+---------------------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST               | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+---------------------------+-------------+--------------+
| group_replication_applier | b89bd4b4-f1ca-11e6-bb62-b82a72ce743a | bbackdb01.niceprivate.com |        3308 | ONLINE       |
+---------------------------+--------------------------------------+---------------------------+-------------+--------------+
1 row in set (0.00 sec)

</code>
</pre>

增加新成员</br>
增加实例:bbackdb05:3307,配置和刚才bbackdb01:3308 配置基本一致。配置文件部分这里就省略了，需要注意的是loose-group_replication_local_address参数要配置为bbackdb05:33070.直到安装完group replication插件。然后直接执行以下命令：
<pre>
<code>

set global group_replication_allow_local_disjoint_gtids_join=ON;

# 启动group replication
mysql> START GROUP_REPLICATION;

# 查看组成员信息：
mysql> SELECT * FROM performance_schema.replication_group_members;
+---------------------------+--------------------------------------+---------------------------+-------------+--------------+
| CHANNEL_NAME              | MEMBER_ID                            | MEMBER_HOST               | MEMBER_PORT | MEMBER_STATE |
+---------------------------+--------------------------------------+---------------------------+-------------+--------------+
| group_replication_applier | 4d469948-f1c6-11e6-b9fc-b82a72ce743a | bbackdb05.niceprivate.com |        3307 | ONLINE       |
| group_replication_applier | b89bd4b4-f1ca-11e6-bb62-b82a72ce743a | bbackdb01.niceprivate.com |        3308 | ONLINE       |
+---------------------------+--------------------------------------+---------------------------+-------------+--------------+
2 rows in set (0.00 sec)

</code>
</pre>


特别注意白名单问题:
在这个问题上卡了我很久，否则增加新成员一致被拒绝，添加失败。

错误日志：
2017-03-07T17:48:55.665669+08:00 0 [Note] Plugin group_replication reported: 'client connected to 10.10.10.51 33080 fd 75'
2017-03-07T17:49:25.666199+08:00 0 [ERROR] Plugin group_replication reported: '[GCS] Timeout while waiting for the group communication engine to be ready
!'


从主节点的错误日志中才看到这个信息。
[Warning] Plugin group_replication reported: '[GCS] Connection attempt from IP address 10.10.30.201 refused. Address is not in the IP whitelist.'

group_replication_ip_whitelist 这个参数如果不指定则值为AUTOMATIC，且不是动态参数，所以需要提前考虑。



### 现有传统复制集群配置Group Replication




参考资料：
https://dev.mysql.com/doc/refman/5.7/en/group-replication-adding-instances.html
http://mysqlhighavailability.com/mysqlha/gr/doc/limitations.html
http://mysqlhighavailability.com/mysqlha/gr/doc/getting_started.html#group-replication
