+++

author = "旅店老板"
title = "rabbitmq"
date = "2023-02-15"
description = "rabbitmq"
tags = [
	"tcp",
]
categories = [
    "linux",
]
series = [""]
aliases = ["migrate-from-jekyl"]
image = "rabbitmq.jpg"
mermaid = true

+++
## 交换机
### 默认交换机
* 默认交换机（default exchange）实际上是一个由消息代理预先声明好的没有名字（名字为空字符串）的直连交换机（direct exchange）
* 每个新建队列（queue）都会自动绑定到默认交换机上，绑定的路由键（routing key）名称与队列名称相同
### 直连交换机
* 直连交换机（direct exchange）是根据消息携带的路由键（routing key）将消息投递给对应队列
* 将一个队列绑定到某个直连交换机上，同时赋予该绑定一个路由键（routing key），将一个携带路由键的消息发送给直连交换机时，交换机会把该消息路由给绑定值为该路由键的队列
> **交换机通过路由键不能匹配到对应队列该如何处理？**  
> 1.**使用mandatory参数结合ReturnListener处理**。mandatory参数在生产端配置，mandatory为false时，不能匹配的消息会被RabbitMQ丢弃；
> mandatory为true时，不能匹配的消息会被RabbitMQ返回给生产者，生产者通过ReturnListener来监听此类消息  
> 
> 2.**使用备份交换机**。给交换机绑定一个备份交换机通过**alternate-exchange**参数实现，备份交换机类型为扇形交换机（fanout），未能匹配的消息会发送给备份交换机，路由键不会变化，扇形交换机会忽略该路由键发送给绑定队列
***
### 扇型交换机
* 扇型交换机（funout exchange）将消息路由给绑定到它身上的所有队列，会忽略绑定的路由键
### 主题交换机
* 主题交换机（topic exchanges）通过对消息的路由键和队列到交换机的绑定模式之间的匹配，将消息路由给一个或多个队列
### 头交换机
* 头交换机使用多个消息属性来代替路由键建立路由规则。通过判断消息头的值能否与指定的绑定相匹配来确立路由规则
## 队列
