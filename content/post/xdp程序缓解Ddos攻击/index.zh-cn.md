+++

author = "旅店老板"
title = "通过xdp轻松缓解Ddos攻击"
date = "2024-05-06"
description = "缓解ddos攻击通过xdp技术实现"
tags = [
	"ebpf",
    "xdp",
]
categories = [
    "ebpf",
     "xdp",
]
series = [""]
aliases = ["migrate-from-jekyl"]
image = "ebpf-go.png"
mermaid = true
+++
阅读本文之前能对ebpf程序有一定的了解。
## xdp是什么
xdp全称eXpress Data Path，即快速数据路径。是一种ebpf程序的应用，它能够在网络数据包到达网卡驱动层时对其进行处理,在网络协议栈入口处对数据包进行过滤、丢弃、转发。  

xdp程序通常工作在数据链路层(特殊的网卡设备也能运行xdp程序,彻底解放cpu。Netronome智能网卡支持xdp卸载),更早的数据包处理能力能帮助我们实现**高性能的负载均衡器**、**Ddos攻击防护或防火墙**。

![xdp](xdp.png "xdp")   
如上图(图片来源于网络)所示,传统的netfilter过滤数据包在网络协议栈中,对于Ddos攻击抵御效果不佳(数据包分配了skb,也在网络堆栈中运行一定时间占用服务器资源)。  

而xdp程序是在分配skb之前处理数据包。xdp程序处理数据包有特定的返回值,不同的返回值会有不同的逻辑处理。如下图所示:
![xdp](xdp1.png "xdp")  
```c
enum xdp_action {
	XDP_ABORTED = 0,
	XDP_DROP,
	XDP_PASS,
	XDP_TX,
	XDP_REDIRECT,
};
```
* **XDP_DROP**：直接丢弃数据包,缓解Ddos攻击通常用这种  


* **XDP_ABORTED**: 丢弃数据包，但会触发一个eBPF程序静态跟踪点，可以调试监控这种非正常行为  


* **XDP_TX**:将处理后的数据包发回相同网卡


* **XDP_REDIRECT**:将包重定向到其他网络接口（包括虚拟机的虚拟网卡），或者通过AF_XDP socket重定向到用户空间。


* **XDP_PASS**:允许报文上送到内核网络栈，同时处理该报文的CPU会分配并填充一个skb，将其传递到内核协议栈

***
## 以太网帧
本节回顾以太网帧的相关知识,为xdp程序开发做一些准备。
```c
SEC("xdp")
int xdp_hello(struct xdp_md *ctx) {
    return XDP_DROP;
}
char LICENSE[] SEC("license") = "Dual BSD/GPL";
```
这是一个最简单的xdp程序,它的逻辑是丢弃所有数据包。xdp_md是xdp程序的上下文，其中的指针指向一个数据包(以太网帧)。  
```c
struct xdp_md {
	__u32 data;
	__u32 data_end;
	__u32 data_meta;
	/* Below access go through struct xdp_rxq_info */
	__u32 ingress_ifindex; /* rxq->dev->ifindex */
	__u32 rx_queue_index;  /* rxq->queue_index  */
};
```
结构体字段是__u32类型的数字,实际是指向以太网帧的指针。

在开发xdp程序之前,回顾一下以太网帧的结构，xdp的核心就是对以太网帧的解析。以太网帧结构如下:
![以太网帧](以太网帧.png "以太网帧")  
前导码和帧开始符是在物理层由网卡添加上的,严格意义上说不算是以太网帧的一部分。
以太网帧由**头部(header)**、**载荷(data)**、**校验和(FCS/CRC)**。头部包含三个字段：①6字节的目标MAC地址，标记数据由哪台机器接收  ②6字节的源MAC地址，标记数据由哪台机器发送
③帧类型，长度为两个字节。
>从上图可知帧类型有IPv4、ARP、VLAN Tag、IPv6等,帧类型指明了网络层由哪个协议处理数据。  
> 
>当数据帧到达网卡时，在物理层上网卡要先去掉前导同步码和帧开始定界符，然后对帧进行CRC检验，如果帧校验和错，就丢弃此帧。校验和为4字节,值为16进制反码求和，相对于md5是很粗糙的计算，因此在网络协议栈每层都会计算自己的校验和。

当帧类型为Ipv4时,载荷就是一个IP数据报。IP数据报由IP报头和IP载荷构成。
***
## xdp程序实现
①初始化一个名为`xdp_parse.c`的文件,内容为：
```c
//go:build ignore
#include <linux/bpf.h>
#include <linux/if_ether.h>
#include <linux/ip.h>
#include <bpf/bpf_helpers.h>
#include <bpf/bpf_endian.h>

SEC("xdp")
int xdp_parse_func(struct xdp_md *ctx) {
    void *data_end = (void *) (long) ctx->data_end;
    void *data = (void *) (long) ctx->data;
    struct ethhdr *eth;
    struct iphdr *iph;
    __u64 nh_off = sizeof(*eth);
    
    return XDP_PASS;
}

char _license[] SEC("license") = "GPL";
```
代码中我们获取了以太网帧的起始`data`和结束地址`data_end`,申明了变量eth,它是指向以太网头部ethhdr的指针;申明了变量iph,它是指向ip协议头的指针;
申明了变量nh_off,表示以太网头部的内存大小,单位是字节。  

```c
    eth = data;

    if (data + nh_off > data_end) {
        return XDP_PASS;
    }
```
添加如上代码,eth指向数据起始地址,`data + nh_off > data_end`数据起始地址加上偏移量判断eth是否内存访问越界,这在xdp程序中是必不可少的,
当将xdp程序加载到内核中如果没有越界检查,内核将拒绝执行。   

接着:
```c
    __u16 eth_type = eth->h_proto;
    if (eth_type != bpf_ntohs(ETH_P_IP)) {
        return XDP_PASS;
    }
```
获取了以太网帧的类型,上一节中我们知道类型的内存大小和有可能的值,这里我们只处理ip数据报,`bpf_ntohs`的作用是将网络字节序转为主机字节序。

接着获取ip头中的源ip地址:
```c
    if (data + nh_off + sizeof(*iph) > data_end) {
        return XDP_PASS;
    }

    iph = data + nh_off;
    unsigned int sip = iph->saddr;
```
接着我们申明一个ebpf map,如下:
```c
#define MAX_MAP_ENTRIES 16
struct {
    __uint(type, BPF_MAP_TYPE_LRU_HASH);
    __uint(max_entries, MAX_MAP_ENTRIES);
    __type(key, __u32); // source IPv4 address
    __type(value, __u32); // packet count
} xdp_stats_map SEC(".maps");
```
这个map是`BPF_MAP_TYPE_LRU_HASH`类型,key存储源ip, value存储packet计数。继续在程序中添加代码:
```c
    unsigned int sip = iph->saddr;

    __u32 *count = bpf_map_lookup_elem(&xdp_stats_map, &sip);
    if (count) {
        (*count) += 1;
    } else {
        __u32 init_pkt_count = 1;
        bpf_map_update_elem(&xdp_stats_map, &sip, &init_pkt_count, BPF_ANY);
    }


    return XDP_PASS;
```
`bpf_map_lookup_elem`是ebpf中提供的辅助函数，用户获取映射存储map的key的值value,不存在时则返回NULL。上述代码的逻辑就是获取ip的统计次数,有则计数加1,没有则设置为1。  

再创建一个ebpf map,结构与上面的一样，存储ip黑订单
```c
struct {
    __uint(type, BPF_MAP_TYPE_LRU_HASH);
    __uint(max_entries, MAX_MAP_ENTRIES);
    __type(key, __u32);
    __type(value, __u32);
} xdp_black_list_map SEC(".maps");
```
增加ip黑名单检查代码:
```c
    __u32 *val = bpf_map_lookup_elem(&xdp_black_list_map, &sip);
    if (val) {
        return XDP_DROP
    }
    
    return XDP_PASS;
```
以上就是xdp程序内核态的全部,它的作用就是解析以太网帧的ip并统计计数;并检查ip是否在黑名单中，如果在则在网络堆栈的入口处(创建skb之前)丢掉帧。


xdp的用户态程序我们采用golang实现,基本库使用`github.com/cilium/ebpf`,github地址为  [https://github.com/cilium/ebpf](https://github.com/cilium/ebpf)  

golang代码初始结构如下:
```go
package main

import (
	"fmt"
	"github.com/cilium/ebpf"
	"github.com/cilium/ebpf/link"
	"github.com/cilium/ebpf/rlimit"
	"log"
	"net"
	"net/netip"
	"strings"
	"time"
)

//go:generate go run github.com/cilium/ebpf/cmd/bpf2go bpf xdp_parse.c

func main() {
	
}
```
首先执行`go generate`触发`//go:generate`生成`bpf_bpfeb.go`、`bpf_bpfeb.o`、`bpf_bpfel.go`、`bpf_bpfel.o`四个文件,包含ebpf字节码,后面我们通过用户态程序将字节码加载到内核中运行。完成的ebpf用户态程序代码如下:
```go
package main

import (
	"fmt"
	"github.com/cilium/ebpf"
	"github.com/cilium/ebpf/link"
	"github.com/cilium/ebpf/rlimit"
	"log"
	"net"
	"net/netip"
	"strings"
	"time"
)

//go:generate go run github.com/cilium/ebpf/cmd/bpf2go bpf xdp_parse.c

func main() {

	// Allow the current process to lock memory for eBPF resources.
	if err := rlimit.RemoveMemlock(); err != nil {
		log.Fatal(err)
	}

	//The network interface by name.
	ifaceName := "eth0"
	iface, err := net.InterfaceByName(ifaceName)
	if err != nil {
		log.Fatalf("lookup network iface %q: %s", ifaceName, err)
	}

	// Load pre-compiled programs into the kernel.
	objs := bpfObjects{}
	if err := loadBpfObjects(&objs, nil); err != nil {
		log.Fatalf("loading objects: %s", err)
	}
	defer objs.Close()

	// Attach the program.
	l, err := link.AttachXDP(link.XDPOptions{
		Program:   objs.XdpParserFunc,
		Interface: iface.Index,
	})
	if err != nil {
		log.Fatalf("could not attach XDP program: %s", err)
	}
	defer l.Close()

	log.Printf("Attached XDP program to iface %q (index %d)", iface.Name, iface.Index)
	log.Printf("Press Ctrl-C to exit and remove the program")

	ticker := time.NewTicker(2 * time.Second)
	defer ticker.Stop()
	for range ticker.C {
		s, err := formatMapContents(&objs)
		if err != nil {
			log.Printf("Error reading map: %s", err)
			continue
		}
		log.Printf("Map contents:\n%s", s)
	}

}

func formatMapContents(obj *bpfObjects) (string, error) {
	var (
		sb  strings.Builder
		key netip.Addr
		val uint32
	)
	iter := obj.XdpStatsMap.Iterate()
	for iter.Next(&key, &val) {
		sourceIP := key // IPv4 source address in network byte order.
		packetCount := val
		sb.WriteString(fmt.Sprintf("\t%s => %d\n", sourceIP, packetCount))

		if packetCount > 200 {
			err := obj.XdpBlackListMap.Update(&key, &val, ebpf.UpdateAny)
			if err != nil {
				return "", nil
			}
		}
	}
	return sb.String(), iter.Err()
}
```
程序大体逻辑是将ebpf字节码加载到内核,xdp内核程序会处理`eth0`网卡的数据,用户态程序每3秒读取一次`xdp_stats_map`(我们在c中申明的ebpf map),打印ip及对应的计数到控制台。  

当计数超过200时,将ip加入到黑名单的map中,内核态程序读取到ip在黑名单后返回`XDP_DROP`,直接丢失以太网帧,达到缓解ddos的目的。

 执行`go build`生成可执行文件。

## 测试
>注意：启动程序后如果本地访问超过200会导致xshell等远程连接工具断连, 在关闭程序前都无法进行连接。
执行我们生成的可执行文件,控制台打印如下:
```shell
2024/06/07 01:21:06 Map contents:
	169.254.128.13 => 1
	101.86.252.181 => 2
	169.254.0.4 => 8
	183.60.83.19 => 2
2024/06/07 01:21:08 Map contents:
	169.254.128.13 => 1
	101.86.252.181 => 3
	169.254.0.4 => 8
	169.254.128.3 => 1
	183.60.83.19 => 2
	20.194.60.135 => 1
```
服务器中启动了一个nginx进行测试,在网页不断访问,可以观察到对应ip的计数在不断增加。超过200加入黑名单后,nginx访问不通(基本所有对服务器端的连接都会断开,迫不得已只能到云服务器控制台重启服务器)。  

我们实现了一个最简单的缓解ddos攻击的逻辑。还可以通过一些算法检或大数据统计检测IP是否是DDOS攻击。  
实际上可以从端口等进行分析ddos流量;可以处理多个网卡的数据帧;可以设置一个告警通知的速率阈值;可以设置黑名单的速率法制(本文仅简单设置位200个计数)  

还可以将用户态改成一个web管理程序,可以手动增加或删除黑名单中的ip。  

本文是一个简单的、完整的(包含内核态部分和用户态部分)xdp程序,希望对你学习ebpf有帮助。源码地址:[https://github.com/ldlb9527/xdp_ddos_example](https://github.com/ldlb9527/xdp_ddos_example)





