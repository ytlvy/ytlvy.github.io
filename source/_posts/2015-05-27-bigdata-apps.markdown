---
layout: post
title: "大数据应用"
date: 2015-05-27 12:53:43 +0800
comments: true
categories: Bigdata
---
storm  最火的流式处理框架 比较轻量级
Spark Streaming  重量级
kryo / FST  JAVA高效序列化与反序列化
rabbitMQ/ Kafka/MetaQ /fqueue RocketMQ
    MetaQ 面向消息有序的场景，能够提供更大的消息堆积能力
    消息系统的作用，就在于能够将“弱”一致变成“最终”一致，保证多方数据的状态的最终正确性
Spanner Google 这套系统是一个跨数据中心的强一致数据库系统
glassfish
osgi
jpa
hive  Hadoop的数据仓库工具
Spark 高效的分布式计算系统 使用内存 支持迭代计算
Nagios zabbix    Zenoss “Hyperic HQ” 运维工具  Ganglia
Nagios
akka
erlang
Tair 缓存
  数据的分布 --- 对照表  --- 客户端同步的
  多备份的支持
  多机架和多数据中心的支持 --- IP掩码来判断机器所属的机架和数据中心
  轻量级的configserver configserver不和client保持心跳，一旦配置更新通过data
Puppet 自动化配置管理工具
DataX 异构的数据库/文件系统之间高速交换数据的工具，实现了在任意的数据处理系统(RDBMS/Hdfs/Local filesystem）之间的数据交换，由淘宝数据平台部门完成
varnish 做网站和小文件的缓存
HAPROXY
raid5
squid
Keepalived