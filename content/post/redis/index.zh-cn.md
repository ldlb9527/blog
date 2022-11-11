+++

author = "旅店老板"
title = "Redis(1)基础：从入门到入土"
date = "2022-11-10"
description = "基础的、全面的Redis教程"
tags = [
	"redis",
]
categories = [
    "redis",
]
series = [""]
aliases = ["migrate-from-jekyl"]
image = "redis.png"
mermaid = true
+++

# 1.Redis数据类型
* Redis的键可以使用任何二进制序列，例如："first"字符串或mp3文件的内容，空字符串也可作为键
* 不要使用过长的键，除了消耗内存资源，还在查找键时会进行耗时的比较，如果你想一个文件关联另一个文件，可以先对文件进行哈希运算后作为键
* 使你的键更有意义，例如："zhangsan:100:eat"
* 键最大长度为512MB
## 1.1 Strings
* Redis的键是字符串，当我们使用字符串类型时，这是将一个字符串映射到另一个字符串，例如缓存html页面
```shell
> set firstkey firstvalue
OK
> get firstkey
firstvalue
```
使用SET和GET(Redis通过RESP协议完成客户端和服务器端通信，命令不区分大小写)命令设置和获取值，注意：如果SET的键已存在，值将被覆盖  
SET命令有一些附加参数可以进行设置,例如键已经存在，要求SET失败；
```shell
> get firstkey
firstvalue
> set firstkey newvalue nx
null
> get firstkey
firstvalue
```
或者当键存在时，要求SET才能成功
```shell
> get firstkey
firstvalue
> set firstkey newvalue xx  ##如果set一个不存在的键会返回null，设置失败
OK
> get firstkey
newvalue
```
你可以使用原子操作,`incr`将字符串解析为整数递增1,并返回新值，`decr`为递减1 类似的操作还有`incrby`、`decrby`,它们递增或递减给定值
```shell
> set counter 10
OK
> incr counter
11
> incr counter
12
> incrby counter 20
32
```
`getset`命令将键设置为新值返回旧值
```shell
> getset b b
null
> getset b a
b
```
`mset`命令可以设置多个键值对，  
`mget`命令根据多个键返回一个值数组，批量设置和获取可以减少延迟
```shell
> mset a 1 b 2 c 3
OK
> mget a b c
1
2
3
```
`exists`命令可以判断键是否存在，存在时返回1，不存在时返回0，不论值是什么类型，  
`del`命令可以删除键及对应的值，删除成功返回1，删除的键不存在返回0，不论值是什么类型
```shell
> set lisi 26
OK
> exists lisi
1
> del lisi
1
> exists lisi
0
> del lisi
0
```
你可以设置键的过期时间，无论它的值是什么类型，当生存时间过后，键将自动销毁  
* 可以使用秒或毫秒来设置它们，但过期时间分辨率始终为1毫秒
* Redis会保存键过期的日期，当Redis服务器停止运行也不影响持久化的数据正常过期
`expire`命令设置键的过期时间，以秒为单位指定；`pexpire`过期时间以毫秒单位指定
`ttl`命令可以获取键过期的剩余时间，以秒为单位; `pttl`获取剩余过期时间，以毫秒为单位
```shell
#设置lisi的过期时间为10s 第一次检查过期时间还剩6s,
#此时可以获取到lisi对应的值，6s后lisi正常过期
> set lisi 20
OK
> expire lisi 10
1
> ttl lisi
6
> get lisi
20
> get lisi
null
```
你可以在设置值时同时设置过期时间,如下(键lisi在5s后过期)：
```shell
> set lisi 20 ex 5
OK
> get lisi
20
> get lisi
null
```
## 1.2 Lists
List数据类型是通过链表实现的，在列表头部或尾部添加新元素的时间复杂度为O(1),一次添加n个元素的时间复杂度为O(n)  
`lpush`命令将一个新元素从左侧(头部)添加到列表中，时间复杂度O(1)  
`rpush`命令将一个新元素从右侧(尾部)添加到列表中，时间复杂度O(1)  
`lrange`命令从列表中提取元素，时间复杂度O(S+N)，其中 S 是小列表的起始偏移量与 HEAD 的距离，对于大型列表，起始偏移量是与最近端（HEAD 或 TAIL）偏移的距离;N 是指定范围内的元素数。
```shell
> rpush mylist A
1
> rpush mylist B
2
> lpush mylist first
3
> lrange mylist 0 -1
first
A
B
```
`lrange`命令使用两个索引值，两个索引值都可以是负数，-1表示列表的最后一个元素，-2表示列表的倒数第二个元素，以此类推。  
`lpush`、`rpush`命令是变长参数命令，一次可以添加多个值：
```shell
> rpush mylist 1 2 3 4 5 "foo bar"
9
> lrange mylist 0 -1
first
A
B
1
2
3
4
5
foo bar
```
list还具有从左边或右边弹出元素的能力，`lpop` 、`rpop`命令，当列表为空时这两个命令返回null
```shell
> rpush list a b c
3
> lpop list
a
> rpop list
c
> rpop list
b
> rpop list
null
```
通常我们可能会获取用户最近的10个行为，以用户id为列表的键，每用户有行为产生时`lpush userid eat`  `lpush userid see`,获取最近的10个行为：`lrange userid 0 9`  
上面的操作可能会带来严重的后果，随着用户的行为的产生，列表中值会越来越多，占用大量的内存，你可以使用`ltrim`命令  
`ltrim`与`lrange`命令相似，使用两个索引值，将索引范围内设置为新的列表值，索引范围外的值都将被删除,无需记住非常旧的元素。见下面示例：
```shell
> rpush list a b c d e
5
> ltrim list 0 2
OK
> lrange list 0 -1
a
b
c
```
在项目中通常会使用到生产者消费者模式,可以用以下的方式实现：
* 将消息推送到列表，生产者调用`lpush`
* 将消息取出列表消费,消费者调用`rpop`

有时列表会是空的，因此`rpop`只返回null，消费者不断轮训，会有一些缺点：
* 强制Redis和客户端处理无用的命令(列表为空时，没有完成任何实际工作，只会返回null，占用了很多cpu资源)
* 当返回null时可以等待一段时间再获取，这样消息处理的即时性就低，减少等待时间，又会放大上一个缺点

因此Redis实现了`brpop`和`blpop`命令，它们能够在列表为空时阻塞，当新元素添加或超过用户指定的超时时间返回
```shell
> lpush list a b c
3
> brpop list 5
list
a
```
上面的操作代表:等待列表中的元素，5s后如果没有新元素将返回null
注意，你可以设置超时时间为0，表示永久等待元素，还可以指定多个列表，等待到元素则返回，返回值为键名和元素值
