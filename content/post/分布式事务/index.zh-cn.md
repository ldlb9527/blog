+++
author = "旅店老板"
title = "Golang中map的基本原理"
date = "2024-05-04"
description = "golang中map的基本原理"
tags = [
	"golang","map",
]
categories = [
    "golang","map",
]
series = [""]
aliases = ["golang-map"]
image = "golang.png"
mermaid = true
+++

##导读
保证临界资源的数据一致性，分布式环境下，我们需要跨多个节点进行加锁，需要以来redis etcd这些状态存储组件，
实现分布式锁。

分布式锁是独占的
不能产生死锁 占有锁的使用方因宕机无法执行解锁。锁应该也能被正确释放(超时机制) 这也会导致任务还未处理完
就被动释放了锁  需要一个看门狗

加锁和解锁必须为同一身份

高可用 

## 实现方式
## 主动轮询型
：不断持续对锁尝试获取，直到取锁成功或超时

获取锁：检查锁是否被获取 数据不存在则写入数据获取锁  这两个步骤需要保证原子性 redis的SET NX操作

value存储身份标识  删除时检查是否匹配 匹配再删除   这个两个也需要原子性  redis lua

redis弱一致的问题  master节点加锁成功但未同步到salve就宕机了


回调型：取锁时发现锁被占用，创建watch订阅锁的释放事件。当锁被释放后感知变化，再次尝试获取锁
回调型当多个尝试watch同一把锁，释放事件会引起`惊群效应`
`