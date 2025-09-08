# SystemTap
- [SystemTap](#systemtap)
  - [Reference](#reference)
  - [介绍](#介绍)
  - [Hello World](#hello-world)
  - [基本语法](#基本语法)
  - [Event](#event)
  - [tapset](#tapset)
  - [修饰符](#修饰符)
  - [stap常用命令行参数](#stap常用命令行参数)
  - [systemtap内置函数](#systemtap内置函数)

## Reference
[SystemTap使用方法进阶](https://juejin.cn/post/7177356747730845754)

## 介绍
SystemTap是一款Linux内核调试和应用跟踪调试的工具，可以读取和更改应用程序运行过程中的状态，具有低时延、动态调试的特点。使用SystemTap你需要编写systemtap语法的脚本，SystemTap会自动把脚本翻译成C语言的代码，再编译成一个系统的内核。在脚本运行期间，内核会加载到系统中去，运行结束后，内核模块会被卸载。这个跟目前业界很火的EBPF很像，但是SystemTap没有EBPF的安全检查机制，比如循环检测、禁止不可达指令等机制。但是SystemTap发展了这么多年，有很多配套的工具和成熟的解决方案，还是很值得我们学习的。

从脚本到C语言再到内核模块，再到probe信息输出的流程图如下: 
![SystemTap](systemtap.png)

## Hello World
hello-world.stp的内容如下: 
```perl
#!/usr/bin/env stap
probe oneshot {
  # 嵌入C代码
  %{ printk(KERN_ALERT "Hello Wrold from systemtap c\n") %};
  # 标准stap的脚本语法
  printf("Hello World\n");
  exit();
}
```

运行后，输出如下: 
```
[root@localhost systemtap]# stap -g hello-world.stp Hello World
```

参数解析: 

-g参数，代表guru模式，运行嵌入C代码的时候，需要开启guru模式

代码详解: 

- probe oneshot代表此block的脚本之运行一次，这个语法与AWK这个工具很相似。
- %{%} 内部可以嵌入C的代码，我们就可以实现调用内核开发中使用的printk函数，打印输出（注意这里的输出不会到标准输出中，需要用dmesg去查看）
- printf是stap脚本的打印，常用于打印统计的数据

嵌入C的代码输入如下: 
```
[root@localhost systemtap]# dmesg|tail -n 1 [51913.062057] Hello Wrold from systemtap c
```

## 基本语法

术语: 
- event:  事件，有定时器事件，有函数事件（enter/exit)
- handler: 某个event发生的时候，执行的代码
- probe = event + handler, probe定义的语法是
  - probe event {statements}
- scripte: systemtap的脚本，一个脚本里面可以有多个probe
- function: 函数，probe的statements中可以使用函数
  - function function_name(arguements) {statements}
  - probe evetn {function_name(arguments)}

## Event

systemtap中有两大类事件，分别是同步事件和异步事件。

- 同步事件: 有代码运行到了指定的位置，而触发的事件
- 异步事件: 与特定的代码运行无关。

可以使用man stapprobes 查看详细的Events有哪些。

同步事件例子: 
- syscall.system_call: 系统调用
- vfs.file_operation
- module("module").funciton("function")
- kernel.function("function")
- kernel.trace("tracepoint")

异步事件例子: 
- begin
- end
- oneshot
- timer事件

```
> stap -e 'probe timer.s(1) {print("1 second elapsed...\n")}'
1 second elapsed...
1 second elapsed...
1 second elapsed...
```

可以使用stap -l查看系统预设的Event。

查看内核函数: 
```
> sudo stap -l 'kernel.function("vfs_read")'
kernel.function("vfs_read@fs/read_write.c:436")
kernel.function("vfs_readlink@fs/namei.c:4598")
kernel.function("vfs_readv@fs/read_write.c:833")
```

查看syscall: 
```
> sudo stap -l 'syscall.*read'
syscall.pread
syscall.read
```

查看trace point: 
```
> sudo stap -l 'kernel.trace("*readpage")'
kernel.trace("ext3:ext3_readpage")
kernel.trace("ext4:ext4_readpage")
kernel.trace("f2fs:f2fs_readpage")
```

查看某个用户态程序的函数: 
```
> sudo stap -l 'process("/usr/local/openresty/nginx/sbin/nginx").function("ngx_http_write*")'
process("/usr/local/openresty/nginx/sbin/nginx").function("ngx_http_write_filter@src/http/ngx_http_write_filter_module.c:48")
process("/usr/local/openresty/nginx/sbin/nginx").function("ngx_http_write_filter_init@src/http/ngx_http_write_filter_module.c:357")
process("/usr/local/openresty/nginx/sbin/nginx").function("ngx_http_write_request_body@src/http/ngx_http_request_body.c:529")
process("/usr/local/openresty/nginx/sbin/nginx").function("ngx_http_writer@src/http/ngx_http_request.c:2826")
```

查看tapset中预设的probe:
```
> sudo stap -l 'netdev.change'
netdev.change_mac
netdev.change_mtu
netdev.change_rx_flag
```

## tapset
系统自带的tapset的stp文件在/usr/share/systemtap/tapset中，里面有自定义的probes（event）和函数。

预设的probe例子：
```perl
probe netdev.receive
    = kernel.function("netif_receive_skb_internal") !,
      kernel.function("netif_receive_skb")
{
    try { dev_name = get_netdev_name($skb->dev) } catch { }
    try { length = $skb->len } catch { }
    try { protocol = $skb->protocol } catch { }
    try { truesize = $skb->truesize } catch { }
}
```

定义了netdev.receive这个probe，它是内核函数netif_receive_skb_internal(如果存在)，或者netif_receive_skb函数，并且里面还提前准备了输入参数：

- dev_name
- length
- protocol
- truesize

netdev.receive 定义中，!符号代表的是：netif_receive_skb_internal存在的时候，就使用这个函数。当netif_receive_skb_internal不存在的时候，就使用!后面的函数netif_receive_skb。

- 还有另外一个与!相似的marker: ?。 ?表示该探测点不在的时候，脚本运行也不要报错。
- if {expr} 的maker。signal.*? if (switch) 表示，当switch为true的时候，该探测点才生效。

function的例子：
```perl
function strlen:long(s:string)
%{ /* pure */ /* unprivileged */ /* unmodified-fnargs */
    STAP_RETURN(strlen(STAP_ARG_s));
%}
```
函数strlen返回字符串的长度。

使用tapset的例子nettop.stp: 
```
#! /usr/bin/env stap

global ifxmit, ifrecv
global ifmerged

probe netdev.transmit
{
  ifxmit[pid(), dev_name, execname(), uid()] <<< length
  ifmerged[pid(), dev_name, execname(), uid()] <<< 1
}

probe netdev.receive
{
  ifrecv[pid(), dev_name, execname(), uid()] <<< length
  ifmerged[pid(), dev_name, execname(), uid()] <<< 1
}

function print_activity()
{
  printf("%5s %5s %-12s %7s %7s %7s %7s %-15s\n",
         "PID", "UID", "DEV", "XMIT_PK", "RECV_PK",
         "XMIT_KB", "RECV_KB", "COMMAND")

  foreach ([pid, dev, exec, uid] in ifmerged-) {
    n_xmit = @count(ifxmit[pid, dev, exec, uid])
    n_recv = @count(ifrecv[pid, dev, exec, uid])
    printf("%5d %5d %-12s %7d %7d %7d %7d %-15s\n",
           pid, uid, dev, n_xmit, n_recv,
           @sum(ifxmit[pid, dev, exec, uid])/1024,
           @sum(ifrecv[pid, dev, exec, uid])/1024,
           exec)
  }

  print("\n")

  delete ifxmit
  delete ifrecv
  delete ifmerged
}

probe timer.ms(5000), end, error
{
  print_activity()
}
```
此脚本使用了预设tapset的probe点：netdev.receive，统计使用网络的进程。

输出如下: 
```
> stap nettop.stp
   PID   UID DEV          XMIT_PK RECV_PK XMIT_KB RECV_KB COMMAND
    0     0 eth0               0      28       0       1 swapper/3
 3201  1000 eth0              14       0       1       0 sshd
14913  1000 eth0               5       0       0       0 ping
 1983  1000 eth0               1       0       0       0 sshd
```

## 修饰符
probe函数的时候，我们常看到一些修饰符，比如.inline / .call / .return等。

- .return是probe函数运行返回的时候，在此probe的handler中，我们可以通过$return变量获取到函数的返回值
- .inline代表的，probe的是内联函数。
- .call是与.inline相反的
- .maxactive 修饰该探测点可以同时有多少个实例在运行，如果一个函数由于系统默认的maxactive过低导致没有被探测到，可以适当使用.maxactive调整
- ?表示该探测点不在的时候，脚本运行也不要报错
- ! 表示探测点满足了一个，就不再继续解析下去（resolve）

## stap常用命令行参数

- -x 指定进程的PID进行追踪,target()函数返回的就是-x指定的
- -c 运行一个命令，并且追踪这个命令
- -e 从命令行中输入脚本

```
stap -v -e 'probe vfs.read{ printf("read performed"); exit()}'
Pass 1: parsed user script and 473 library scripts using 271956virt/69188res/3504shr/65752data kb, in 420usr/20sys/442real ms.
Pass 2: analyzed script: 1 probe, 1 function, 7 embeds, 0 globals using 439512virt/233636res/4820shr/233308data kb, in 1360usr/460sys/1813real ms.
Pass 3: using cached /root/.systemtap/cache/89/stap_89794dac39b59bfbd6e29e0bad1d4100_2803.c
Pass 4: using cached /root/.systemtap/cache/89/stap_89794dac39b59bfbd6e29e0bad1d4100_2803.ko
Pass 5: starting run. read performed
Pass 5: run completed in 0usr/40sys/595real ms.
```

- -l ：list probe points
```
> stap -l 'syscall.write*'
syscall.write
syscall.writev
```

- -L： 输出probe points的详细信息，包括支持的变量
```
> sudo stap −L 'netdev.receive'
netdev.receivedev 
    name:string
    length:long
    protocol:long
    truesize:long
    skb:struct sk_buff*
```
netdev.receive可以使用的变量有：dev_name / length / protocol/ truesize / $skb

- --dump-probe-type 获取stap支持的probe语法
- --dump-probe-aliases 获取tapset中预设的probe aliases
- --dump-functions 获取tapset中预设的函数

## systemtap内置函数

所谓的内置函数，也是在tapset中预设的。大部分的函数都是使用内嵌C代码的方式，完成其逻辑。

- pp()
  - 当前probe point的名字
- target()
  - -x PID 或 -c CMD 指定的进程PID
- ctime()
- cpu()
  - 当前CPU的编号
- probefunc()
  - 当前probe point所在的函数

以probefunc为例子，其实现在linux/context-symbols.stp文件中。



