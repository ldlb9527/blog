+++

author = "旅店老板"
title = "时序数据库InfluxDB初体验"
date = "2022-11-18"
description = "InfluxDB"
tags = [
	"influxdb",
]
categories = [
    "influxdb",
]
series = [""]
aliases = ["migrate-from-jekyl"]
image = "influxdb.jpg"
mermaid = true
+++
# 一、介绍
## 1.基本介绍
InfluxDB是使用GO编写的基于时间序列的数据库,用作涉及大量时间戳数据的任何用例的后备存储，包括 DevOps 监控、应用程序指标、IoT 传感器数据和实时分析

## 2.关键名词
* **database** 数据库,用户、保留策略、连续查询和时序数据的逻辑容器。
* **measurement** 度量,用于描述存储在关联字段中的数据。 度量是字符串,类似于传统数据库中的表。
* **point** 点,表示单个数据记录，每一点都具有measurement、tag set、a field key, a field value, and a timestamp，类似于传统数据库表中的一行。
* **tag** 标签，InfluxDB数据结构中记录元数据的键值对,标签是数据结构的可选部分，具有索引，是“key-value”的形式
* **field** 字段， InfluxDB数据结构中的键值对，用于记录元数据和实际数据值，字段是必需的，并且它们没有索引，相对于标签而言，它们的查询性能不高
* **timestamp** 时间戳，与点关联的日期和时间。 InfluxDB中的所有时间都是UTC。在插入数据时可以自己指定也可留空让系统指定。
* **TSM** 时间结构化合并树，由LSM改进
## 3.客户端命令(通过influx进入)
### 数据库操作
* 创建数据库,`CREATE DATABASE {NAME};`
```
create database test;
```
* 使用数据库`USE {NAME};`
```
use test;
```
* 删除数据库,`DROP DATABASE {NAME};`
```
drop database test;
```
* 查看数据库
```
show databases;
```
### 数据表操作
* 查看所有表
```
show measurements;
```
* 新增数据
>`insert <measurement>[,<tag-key>=<tag-value>...] <field-key>=<field-value>[,<field2-key>=<field2-value>...] [unix-nano-timestamp]`  
>InfluxDB 下没有细分的表的概念，InfluxDB 下的表在插入数据库的时候自动会创建。  
>写数据的时候如果不添加时间戳，系统会默认添加一个时间。
```
insert http_requests,method=GET,status=200 A=0.6,B=0.4
insert http_requests,method=POST,status=200 A=1,B=2
insert http_requests,method=POST,status=400 A=1,B=2
```
* 删除表,`DROP MEASUREMENT {NAME}`
```
drop measurement http_requests;
```
* 查询数据
```
select A,B from http_requests
```
* 修改和删除数据
>InfluxDB属于时序数据库，没有提供修改和删除数据的方法。  
>删除可以通过 InfluxDB 的数据保存策略（Retention Policies）来实现
### series操作
>series 表示这个表里面的数据，可以在图表上画成几条线  
```
show series from "http_requests";
key
---
http_requests,method=GET,status=200
http_requests,method=POST,status=200
http_requests,method=POST,status=400
```
### 数据保留策略
>InfluxDB的保留策略除了描述数据的TTL duration之外，还包含数据的副本数（replication factor），以及每个Shard Group的时间区间长度  
>InfluxDB的保留策略是定义在数据库上的，并且以Shard Group为单位做批量过期  
> 创建数据库时，默认会随着创建一个名为autogen的保留策略，shard duration为7天，而duration为0秒（即永远不过期）。
* 查询保留策略
```
> SHOW RETENTION POLICIES ON "mydb"
name    duration shardGroupDuration replicaN default
----    -------- ------------------ -------- -------
autogen 0s       168h0m0s           1        true

```
* 创建保留策略 `CREATE RETENTION POLICY <retention_policy_name> ON <database_name> DURATION <duration> REPLICATION <replication_num> [SHARD DURATION <shard_duration>]  [DEFAULT]`  
<retention_policy_name>是保留策略的名称；  
<database_name>是数据库的名称；  
\<duration>是数据的TTL；  
<replication_num>是数据的副本数；  
<shard_duration>是可选项，表示每个Shard Group的时间区间长度；
DEFAULT也是可选项，如果指定，表示将此策略顺便设为默认。
>如果不指定shard duration的话，当duration小于2天时，shard duration默认为1小时；duration介于2天和6个月之间时，shard duration默认为1天；duration大于6个月时，shard duration默认为7天  

创建一个新的TTL为一周的保留策略，并设为默认
```
CREATE RETENTION POLICY "one_week" ON "mydb" DURATION 7d REPLICATION 1 DEFAULT
```
* 修改保留策略
```
ALTER RETENTION POLICY "one_week" ON "mydb" DURATION 48h DEFAULT
```
* 删除保留策略,`DROP RETENTION POLICY "rp_name" ON "db_name"`
```
drop retention POLICY "one_week" ON "mydb"
```
## 4.常用函数
### COUNT()函数
>返回一个（field）字段中的非空值的数量。  
>SELECT COUNT(<field_key>) FROM <measurement_name> [WHERE \<stuff>] [GROUP BY \<stuff>]
> 注意：count()函数中使用field_key,而不能使用tag_key
```
select count(A) from http_requests where status=200
```
