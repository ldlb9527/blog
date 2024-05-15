+++

author = "旅店老板"
title = "初识eBPF"
date = "2024-05-03"
description = "介绍eBPF的基础知识,用一个eBPF的HelloWorld程序展开说明"
tags = [
	"ebpf",
]
categories = [
    "ebpf",
]
series = [""]
aliases = ["migrate-from-jekyl"]
image = "ebpf.jpg"
mermaid = true
+++
## eBPF是什么
eBPF全称是extended Berkeley Packet Filter,是Linux内核上的一个强大的网络和性能分析工具，它允许开发者在内核运行时动态加载、更新和运行用户定义的代码。 

eBPF程序主要由两部分构成：内核态部分和用户态部分。内核态部分包含eBPF程序的实际逻辑，用户态部分负责加载、运行和监控内核态程序。通常内核态可以由C或Rust编写，用户态可以用Python、Golang、C等等编写，用户态部分负责加载、运行和监控内核态程序。  



在linux内核中有一套hook函数机制,hook机制的作用是挂载一种未确定的逻辑处理过程以实现对特定位置(hook点)的处理。eBPF程序加载到内核后通过事件驱动，当内核或应用程序通过某个hook点时触发执行。预定义的钩子包括：系统调用、函数进入/退出、内核跟踪点、网络事件等。  
***
## eBPF开发环境准备
```shell
apt install clang llvm libelf1 libelf-dev zlib1g-dev
```
`LLVM`和`Clang`是编译eBPF程序所需要的依赖环境。

```shell
wget https://github.com/eunomia-bpf/eunomia-bpf/releases/latest/download/ecc && chmod +x ./ecc && cp ecc /usr/bin/
```
`ecc`将c语言源代码编译成eBPF字节码

```shell
wget https://aka.pw/bpf-ecli -O ecli && chmod +x ./ecli && cp ecli /usr/bin/
```
`ecli`可以帮助我们将eBPF字节码加载到内核运行
***
## 第一个eBPF的HelloWorld程序
创建一个`helloworld.c`文件,内容如下:
```c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h>

char LICENSE[] SEC("license") = "Dual BSD/GPL";

SEC("tracepoint/sched/sched_process_fork")
int tr_hello(void* ctx) {
    int pid = bpf_get_current_pid_tgid() >> 32;

    bpf_printk("Hello World from PID:%d", pid);
    return 0;
}
```
 * include引入必要的头文件
 * `LICENSE`变量定义了许可证信息,暂且可以当作一个固定写法
 * `SEC("tracepoint/sched/sched_process_fork")` `SEC`是一个宏定义,它里面的值说明这是一个`tracepoint`类型的程序,插入如函数为`sched_process_fork`,这是一个内核中的函数，系统创建进程时
最终会调用该函数
 * 定义了`tr_hello`函数,`void* ctx`ctx本来是具体类型的参数， 但是由于我们这里没有使用这个参数，因此就将其写成void *类型
 * `bpf_get_current_pid_tgid()`是BPF中的一个辅助函数，用户获取当前进程ID
 * `bpf_trace_printk()`是一种将信息输出到trace_pipe(/sys/kernel/debug/tracing/trace_pipe)简单机制  

**这个代码的逻辑就是当系统创建一个进程时就打印`Hello World from PID:**`**

用ecc工具编译为eBPF字节码,如下:
```shell
root@LAPTOP-J85AQ3DD:~/helloworld# ecc helloworld.c
INFO [ecc_rs::bpf_compiler] Compiling bpf object...
INFO [ecc_rs::bpf_compiler] Generating package json..
INFO [ecc_rs::bpf_compiler] Packing ebpf object and config into package.json...
```
`ls`命令查看:
```shell
ls 
helloworld.bpf.o  helloworld.c  helloworld.skel.json  package.json
```

编译成功后用`ecli`工具运行，如下：
```shell
root@LAPTOP-J85AQ3DD:~/helloworld# ecli run package.json
INFO [faerie::elf] strtab: 0x244 symtab 0x280 relocs 0x2c8 sh_offset 0x2c8
INFO [bpf_loader_lib::skeleton::poller] Running ebpf program...
```

打开另一个终端执行`cat /sys/kernel/debug/tracing/trace_pipe`,

再打开一个终端执行`ls`或`ps`等命令,这些命令会创建进程，此时查看`cat /sys/kernel/debug/tracing/trace_pipe`的终端窗口有如下结果
```shell
root@LAPTOP-J85AQ3DD:~# cat /sys/kernel/debug/tracing/trace_pipe
            bash-1410    [014] d...1   641.640351: bpf_trace_printk: Hello World from PID:1410
            bash-1471    [014] d...1   646.854774: bpf_trace_printk: Hello World from PID:1471
            bash-1471    [014] d...1   720.095740: bpf_trace_printk: Hello World from PID:1471
```
每创建一个进程,都会打印一次对应的记录,`ctrl`+`C`结束运行的程序,自此我们运行了我们的第一个eBPF程序
***
## eBPF程序的类型
* `Tracepoint`: Tracepoint程序跟踪内核中的事件和函数调用,用于进行静态插桩。静态插桩是指内核中已经预先定义了许多hook点,eBPF字节码加载进内核后会在对应位置执行。
* `Kprobes`:Kprobes程序与Tracepoint程序相反，用作动态插桩，可以跟踪内核中绝大部分函数(部分禁止访问),甚至在某个指令前后插入我们的代码。
> Kprobes包括三种探测手段分别时kprobe、jprobe和kretprobe。kprobe是最基本的探测方式，是实现后两种的基础，它可以在任意的位置放置探测点（就连函数内部的某条指令处也可以）。jprobe基于kprobe实现，它用于获取被探测函数的入参值
> kretprobe不仅用于获取被探测函数的返回值,还能够修改函数的返回值。  
> 
> Kprobes动态插桩可以将我们的代码插入任意位置运行，缺点是不稳定，不同版本的内核代码指令位置不同，效率也比Tracepoint低，因为Tracepoint的预定义hook点是精心设计的,不会对系统性能产生较大影响
>   
> 如果开发者对内核不熟悉，将Kprobes程序插入到某些执行频率极高的位置，可能会明显影响性能  

* `fentry`和`fexit`：用于在 Linux 内核函数的入口和退出处进行跟踪。与kprobes相比，fentry和fexit程序有更高的性能和可用性，fexit程序可以访问函数的输入参数和返回值，而kretprobe只能访问返回值。从5.5内核开始，fentry和fexit对eBPF程序可用。

* `uprobe`:uprobe是一种用户空间探针，uprobe探针允许在用户空间程序中动态插桩，插桩位置包括：函数入口、特定偏移处，以及函数返回处。
> uprobe基于文件，当一个二进制文件中的一个函数被跟踪时，所有使用到这个文件的进程都会被插桩，包括那些尚未启动的进程，这样就可以在全系统范围内跟踪系统调用。
* `xdp`：全称eXpress Data Path,是基于eBPF实现的高性能、可编程的数据平面技术。
> XDP需要网卡的支持，位于网卡驱动层，当数据包经过DMA存放到ring buffer之后，分配skb之前，即可被XDP处理。可用于高速数据包处理,比如缓解DDOS、负载均衡。

## eBPF MAP
特殊的数据结构,用于用户态程序和eBPF程序通信




