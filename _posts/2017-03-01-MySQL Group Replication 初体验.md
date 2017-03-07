---
layout: post
title: MySQL Group Replication 初体验
date: 2017-03-01
---

  mysql 5.7.17版本的发布，标志着Group Replication功能的正式发布。Group Replication的引入是为了提高可用性，保证数据0丢失，多主复制，多点写入。让我们来体验下这个新特性。

## 环境搭建
  以下将会分别介绍2种情况下的配置方法，即：“全新环境”和“现有传统复制集群”配置的方法。

### 全新环境配置
  先安装官方5.7.17版本的mysql，安装方法略过。
  为了后面压测性能，我选择了使用3台物理服务器来配置，每台服务器安装一个mysqld实例。  
- 实例列表
  + bbackdb05:3307
  + bbackdb11:3306
  + bbackdb01:3308

  my.cnf的配置，除了正常的一些配置外，replication相关的部分需要注意的一些事情。

参考:
http://mysqlhighavailability.com/mysqlha/gr/doc/limitations.html

### 现有传统复制集群配置Group Replication

