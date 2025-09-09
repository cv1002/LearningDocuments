# Crash
- [Crash](#crash)
  - [Reference](#reference)
  - [Crash 工具](#crash-工具)
  - [常用命令](#常用命令)
    - [bt 命令](#bt-命令)
    - [log 命令](#log-命令)
    - [sym 命令](#sym-命令)
    - [net 命令](#net-命令)
    - [rd 命令](#rd-命令)
    - [set 命令](#set-命令)
    - [struct 命令](#struct-命令)
    - [dis 命令](#dis-命令)
    - [kmem 命令](#kmem-命令)
    - [p 命令](#p-命令)
    - [dev 命令](#dev-命令)
    - [files 命令](#files-命令)
    - [irq 命令](#irq-命令)
    - [mount 命令](#mount-命令)
    - [ps 命令](#ps-命令)
    - [mod 命令](#mod-命令)
    - [search 命令](#search-命令)
    - [list 命令](#list-命令)

## Reference
[【调试】crash使用方法 ](https://www.cnblogs.com/dongxb/p/17364995.html)
[crash 工具的使用](https://zhuanlan.zhihu.com/p/707500778)

## Crash 工具
crash是redhat的工程师开发的，主要用来离线分析linux内核转存文件，它整合了gdb工具，功能非常强大。可以查看堆栈，dmesg日志，内核数据结构，反汇编等等。

crash主要用于分析内核崩溃文件。

crash支持多种工具生成的转存文件格式，如kdump，LKCD，netdump和diskdump，而且还可以分析虚拟机Xen和Kvm上生成的内核转存文件。同时crash还可以调试运行时系统，直接运行crash即可，ubuntu下内核映象存放在/proc/kcore。

## 常用命令

### bt 命令
bt 显示函数调用栈，可以显示所有CPU或指定CPU的栈，或者指定pid

- bt 显示当前CPU栈
- bt -a 显示所有CPU栈
- bt -f 显示所有堆栈
- bt -l 显示堆栈trace的文件与行号

### log 命令
显示系统消息缓存区

### sym 命令
symbol 与虚拟地址转换

### net 命令
- net 列出网络设备
- net -a 显示ARP Cache
- net -s 列出所有sock

### rd 命令
直接读内存

### set 命令
获取crash的线程号

### struct 命令
解析结构体

### dis 命令
反汇编地址

### kmem 命令
- kmem -S 显示 slab 对象信息

### p 命令
查看全局变量

### dev 命令
显示系统中块设备和字符设备的信息

### files 命令
显示发生panic的任务打开的所有文件的信息

### irq 命令
显示中断信息

### mount 命令
显示挂载的文件系统信息

### ps 命令
查看系统中进程的信息

### mod 命令
显示或者加载内核模块

### search 命令
在内存中查找所有存放目标数据的地址

### list 命令
显示链表的内容


