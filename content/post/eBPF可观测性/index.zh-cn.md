+++

author = "旅店老板"
title = "eBPF可观测性:动态日志"
date = "2024-06-08"
description = "通过eBPF对线上程序动态添加日志记录"
tags = [
	"ebpf",
    "uprobe",
]
categories = [
    "ebpf",
     "uprobe",
]
series = [""]
aliases = ["migrate-from-jekyl"]
image = "cilium.png"
mermaid = true
+++
阅读本文之前能对ebpf程序有一定的了解。

## 背景
为了观测线上程序的运行情况,通常我们会在对应点记录响应的日志。对于日志不完善的情况，可以添加日志代码并重新打包部署上线。有时候这在生产环境成本是巨大的，比如
添加日志代码上线后通过新的日志记录定位问题,处理后又需要重新打包部署上线。  

我们可以利用ebpf解决这个问题。ebpf的uprobe探针允许我们将探针动态插入到任意用户程序位置,且不影响线上程序的正常运行。
***
## eBPF的uprobe和uretprobe

uprobe是用户空间的插桩技术,它允许用户在用户空间中动态地插入和观察程序的探针,以便监视应用程序的行为、性能和调试信息。

uretprobe不只是能够获取函数的返回值。它能够将我们自定义的返回值附加到原来的返回值上。即我们能在uretprobe中修改函数的返回值，也就是能动态修改程序的逻辑。
>uretprobe修改线上运行程序的返回值是个非常危险的行为，尤其是对于有自己运行时的语言(例如Golang)。 
> 
> 这些语言的参数和返回值存在自己的堆栈中,uretprobe修改的是寄存器上的值,具体可了解**ABI规范**问题。


## 模拟一个线上程序
```c
//go:build ignore

#include <stdio.h>

int AddNum(int i, int i1);


int main() {
    printf("%d\n", AddNum(3,2));
    return 0;
}

int AddNum(int n1, int n2) {
    return n1 + n2;
}
```
文件名为`test.c`,这是一个最简单的c语言程序,自定义了一个`AddNum`函数，用于计算两数之和。每执行一次代表我们线上程序被调用一次接口。

## eBPF程序开发
eBPF程序分为内核态程序和用户态程序,内核态我们采用c语言,用户态我们采用golang的cilium/ebpf库

### 内核态程序开发
程序文件命名为`uprobe_kernel.c`,下面进行一些初始化定义,头文件会在最后的源码中给出。
```c
//go:build ignore

#include "common.h"
#include <bpf/bpf_tracing.h>
#include <stdbool.h>


char __license[] SEC("license") = "Dual MIT/GPL";

struct {
    __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
} events SEC(".maps");

// Force emitting struct event into the ELF.
const struct data_t *unused __attribute__((unused));


struct data_t {
    __u32 pid;
    __u8 comm[80];//TASK_COMM_LEN
    __u64 args[5];
    __u64 ret;
    bool entry;
    __u64 ktime_ns;
};
```
定义了一个ebpf Map,类型为`BPF_MAP_TYPE_PERF_EVENT_ARRAY`,它可以帮助我们将自定义的事件数据共享到用户态。`data_t`是我们自定义的结构  
* pid记录进程id
* comm记录程序名
* args记录函数的入参
* entry记录入参时为true, 记录返回值是为false
* ktime_ns时间字段,计算观测函数执行的耗时
* ret记录返回值

增加代码:
```c
SEC("uprobe")
int uprobe_entry(struct pt_regs *ctx) {
    struct data_t data = {};
    __u32 pid = bpf_get_current_pid_tgid() >> 32;

    data.pid = pid;
    bpf_get_current_comm(&data.comm, sizeof(data.comm));
    // Fill in the arguments
    data.args[0] = PT_REGS_PARM1(ctx);
    data.args[1] = PT_REGS_PARM2(ctx);
    data.args[2] = PT_REGS_PARM3(ctx);
    data.args[3] = PT_REGS_PARM4(ctx);
    data.args[4] = PT_REGS_PARM5(ctx);
    data.entry = true;
    data.ktime_ns = bpf_ktime_get_ns();

    bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU, &data, sizeof(data));
    return 0;
}
```
`SEC("uprobe")`eBPF程序的section声明，表明以下的函数用于uprobe,在SEC中并未指定可执行程序位置和函数名,我们在eBPF用户态程序中指定。另外一种写法是`SEC("uprobe//proc/self/exe:TestFunc")`  
`uprobe_entry`这个函数从`ctx`中获取输入参数等,uprobe获取的入参最多支持6个,似乎是从寄存器上取值,构造完成后将`data`存储到ebpf Map中。

继续完善代码:
```c
SEC("uretprobe")
int uprobe_return(struct pt_regs *ctx) {
    struct data_t data = {};
    __u32 pid = bpf_get_current_pid_tgid() >> 32;

    data.pid = pid;
    bpf_get_current_comm(&data.comm, sizeof(data.comm));
    u64 ret = PT_REGS_RC(ctx);
    data.ret = ret;
    data.entry = false;
    data.ktime_ns = bpf_ktime_get_ns();
    bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU, &data, sizeof(data));

    return 0;
}
```
逻辑与上一个函数类似,`SEC("uretprobe")`表示以下函数用于uretprobe,可用于获取或修改返回值。

>uprobe正常工作需要满足两个条件： 
> 
>1.被attach的函数名称
> 
>2.函数所在的可执行文件路径
> 
> 这里的函数名称可以是一个抽象概念。两个条件的目的是为了获取可执行文件符号表中函数符号的地址。这个地址我们可以加上一定的偏移量，因此uprobe探针可以动态插入到用户程序的任意位置
***

### 用户态程序开发态程序开发
eth_type = eth->h_proto;
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
	"bytes"
	"encoding/binary"
	"errors"
	"fmt"
	"github.com/cilium/ebpf/link"
	"github.com/cilium/ebpf/perf"
	"github.com/cilium/ebpf/rlimit"
	"golang.org/x/sys/unix"
	"log"
	"os"
	"os/signal"
	"syscall"
	"time"
)

//go:generate go run github.com/cilium/ebpf/cmd/bpf2go bpf xdp_parse.c

func main() {
	
}
```
首先执行`go generate`触发`//go:generate`生成对应的文件。  

`main`的逻辑如下:
```go
func main() {

	stopper := make(chan os.Signal, 1)
	signal.Notify(stopper, os.Interrupt, syscall.SIGTERM)

	// Allow the current process to lock memory foMakefiler eBPF resources.
	if err := rlimit.RemoveMemlock(); err != nil {
		log.Fatal(err)
	}

	// Load pre-compiled programs and maps into the kernel.
	objs := bpfObjects{}
	if err := loadBpfObjects(&objs, nil); err != nil {
		log.Fatalf("loading objects: %s", err)
	}
	defer objs.Close()

	// Open an ELF binary and read its symbols.
	ex, err := link.OpenExecutable("./test")
	if err != nil {
		log.Fatalf("opening executable: %s", err)
	}
	uprobe, err := ex.Uprobe("AddNum",objs.UprobeEntry, nil)
	if err != nil {
		log.Fatalf("creating uprobe: %s", err)
	}
	defer uprobe.Close()

	// Open a Uretprobe at the exit point of the symbol and attach
	// the pre-compiled eBPF program to it.
	up, err := ex.Uretprobe("AddNum", objs.UprobeReturn, nil)
	if err != nil {
		log.Fatalf("creating uretprobe: %s", err)
	}
	defer up.Close()

	// Open a perf event reader from userspace on the PERF_EVENT_ARRAY map
	// described in the eBPF C program.
	rd, err := perf.NewReader(objs.Events, os.Getpagesize())
	if err != nil {
		log.Fatalf("creating perf event reader: %s", err)
	}
	defer rd.Close()

	go func() {
		// Wait for a signal and close the perf reader,
		// which will interrupt rd.Read() and make the program exit.
		<-stopper
		log.Println("Received signal, exiting program..")

		if err := rd.Close(); err != nil {
			log.Fatalf("closing perf event reader: %s", err)
		}
	}()

	log.Printf("Listening for events..")

	// bpfEvent is generated by bpf2go.
	var event bpfDataT
	for {
		record, err := rd.Read()
		if err != nil {
			if errors.Is(err, perf.ErrClosed) {
				return
			}
			log.Printf("reading from perf event reader: %s", err)
			continue
		}

		if record.LostSamples != 0 {
			log.Printf("perf event ring buffer full, dropped %d samples", record.LostSamples)
			continue
		}

		// Parse the perf event entry into a bpfEvent structure.
		if err := binary.Read(bytes.NewBuffer(record.RawSample), binary.LittleEndian, &event); err != nil {
			log.Printf("parsing perf event: %s", err)
			continue
		}

		comment := fmt.Sprintf(" pid:%v ", event.Pid)
		comment += fmt.Sprintf(" comm:%s ", unix.ByteSliceToString(event.Comm[:]))
		comment += fmt.Sprintf(" current_time:%s ", ktimeToTime(event.KtimeNs).String())

		if event.Entry {
			comment += fmt.Sprintf(" arg1:%v ", event.Args[0])
			comment += fmt.Sprintf(" arg2:%v ", event.Args[1])
		}else {
			comment += fmt.Sprintf(" return:%v ", event.Ret)
		}
		fmt.Println(comment)
	}

}

// ktimeToTime 将内核时间（自开机以来的纳秒数）转换为 time.Time 类型的时间对象
func ktimeToTime(ktime uint64) time.Time {
	// 获取当前时间的纳秒数
	now := time.Now().UnixNano()

	// 计算自开机以来经过的纳秒数
	uptimeNanos := now - int64(ktime)

	// 转换为 time.Duration 对象
	uptimeDuration := time.Duration(uptimeNanos)

	// 创建 time.Time 对象
	bootTime := time.Unix(0, 0).Add(uptimeDuration)

	return bootTime
}
```
上面的逻辑比较简单,将我们的内核态程序加载到内核中，并将uprobe和uretprobe探针附加到指定位置。这里指定位置是指: 可执行文件路径:./test (还记得最开始模拟的线上程序吗,我们会执行编译`gcc -o test test.c`)   函数名: AddNum

## 测试
完整源码地址为: [https://github.com/ldlb9527/ebpf-dynamic-log](https://github.com/ldlb9527/ebpf-dynamic-log)  

附一个Makefile文件的内容:
```makefile
build-test:
	gcc -o test test.c

build:
	go generate && go build -o uprobe

run:
	./uprobe

run-test:
	./test
```
依次执行`make build-test` `make build`  `make run`,此时我们的ebpf程序已经被我们加载到内核并从`PERF_EVENT_ARRAY`获取对应的数据  

打开另一个终端相同目录下执行`make run-test`,每执行一次控制台会打印
```shell
root# make run-test
./test
5
root# make run-test
./test
5
root# make run-test
./test
5
```
打印了`5`,因为我们的test程序就是打印3+2的和。回到另一个终端，控制台打印了
```shell
root# make run
./uprobe
2024/06/10 21:17:07 Listening for events..
 pid:2481911  comm:test  current_time:2024-06-07 01:19:51.254030467 +0800 CST  arg1:3  arg2:2 
 pid:2481911  comm:test  current_time:2024-06-07 01:19:51.254029907 +0800 CST  return:5 
 pid:2483341  comm:test  current_time:2024-06-07 01:19:51.254019975 +0800 CST  arg1:3  arg2:2 
 pid:2483341  comm:test  current_time:2024-06-07 01:19:51.254013336 +0800 CST  return:5 
 pid:2483349  comm:test  current_time:2024-06-07 01:19:51.253990432 +0800 CST  arg1:3  arg2:2 
 pid:2483349  comm:test  current_time:2024-06-07 01:19:51.254012139 +0800 CST  return:5 
 pid:2483415  comm:test  current_time:2024-06-07 01:19:51.25399224 +0800 CST  arg1:3  arg2:2 
 pid:2483415  comm:test  current_time:2024-06-07 01:19:51.254004328 +0800 CST  return:5 
```
第一个参数arg1为3  第二个参数arg2为2  返回值return为5 两个current_time的差就是函数执行耗时

## 分析
上面我们说明uprobe成功的两个条件是:可执行文件和函数名  我们指定的可执行文件是`./test`  函数名是`AddNum`

实际上函数名只是一个符号,程序会去可执行文件中寻找对应地址。将uprobe探针附加到对应位置。执行命令`nm ./test | grep AddNum`可查看到对应函数的地址:
```shell
root@VM-20-15-ubuntu:~/ebpf-dynamic-log# nm ./test | grep AddNum
000000000000117d T AddNum
```
因此我们在程序中并不完全是指定**函数名**,我们可以通过一些开源库分析可执行文件获取地址,设置一定偏移量即可将探针附加到程序任意位置了。  
cilium/ebpf库支持直接设置地址,而不用指定函数名。

## 扩展


