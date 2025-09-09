# KDump
- [KDump](#kdump)
  - [介绍](#介绍)
  - [原理](#原理)
  - [配置](#配置)
    - [一、预留内存](#一预留内存)
    - [二、配置文件](#二配置文件)
    - [三、启动服务](#三启动服务)
    - [四、功能验证](#四功能验证)
    - [五、捕获内核](#五捕获内核)

## 介绍
kdump是系统崩溃的时候，用来转储运行内存的一个工具。系统一旦崩溃，内核就没法正常工作了，这个时候将由kdump提供一个用于捕获当前运行信息的内核，该内核会将此时内存中的所有运行状态和数据信息收集到一个dump core文件中以便之后分析崩溃原因。一旦内存信息收集完成，可以让系统将自动重启。

kdump是RHEL5之后才支持的，2006被主线接收为内核的一部分。它的原理简单来说是在内存中保留一块区域，这块区域用来存放capture kernel，当production kernel发生crash的时候，通过kexec把保留区域的capure kernel给运行起来，再由捕获内核负责把产品内核的完整信息 - 包括CPU寄存器、堆栈数据等转储到指定位置的文件中。

## 原理
kexec是kdump机制的关键，包含两部分：

内核空间的系统调用kexec_load。负责在生产内核启动时将捕获内核加载到指定地址。用户空间的工具kexec-tools。将捕获内核的地址传递给生产内核，从而在系统崩溃的时候找到捕获内核的地址并运行。

kdump是一种基于kexec的内核崩溃转储机制。当系统崩溃时，kdump使用kexec启动到第二个内核。第二个内核通常叫做捕获内核，以很小内存启动以捕获转储镜像。第一个内核保留了内存的一部分给第二个内核启动使用。

由于kdump利用kexec启动捕获内核，绕过了BIOS，所以第一个内核的内存得以保留。这是内存崩溃转储的本质。捕获内核启动后，会像一般内核一样，去运行为它创建的ramdisk上的init程序。而各种转储机制都可以事先在init中实现。为了在生产内核崩溃时能顺利启动捕获内核，捕获内核以及它的ramdisk是事先放到生产内核的内存中的。

生产内核的内存是通过/proc/vmcore这个文件交给捕获内核的。为了生成它，用户工具在生产内核中分析出内存的使用和分布等情况，然后把这些信息综合起来生成一个ELF头文件保存起来。捕获内核被引导时会被同时传递这个ELF文件头的地址，通过分析它，捕获内核就可以生成出/proc/vmcore。有了/proc/vmcore这个文件，捕获内核的ramdisk中的脚本就可以通过通常的文件读写和网络来实现各种策略了。

## 配置
RHEL5开始，kexec-tools是默认安装的。

如果需要调试kdump生成的vmcore文件，需要手动安装kernel-debuginfo包。

### 一、预留内存
可以修改内核引导参数，为启动捕获内核预留指定内存。

在/etc/grub.conf (一般为/boot/grub/grub.conf的软链接)中：

crashkernel=Y@X，Y是为kdump捕获内核保留的内存，X是保留部分内存的起始位置。

默认为crashkernel=auto，可自行设定如crashkernel=256M。

### 二、配置文件
配置文件
配置文件为/etc/kdump.conf，以下是几个常用配置：

```shell
path /var/crash
```
默认的vmcore存放目录为/var/crash/%HOST-%DATE/，包括两个文件：vmcore和vmcore-dmesg.txt

### 三、启动服务

```shell
chkconfig kdump on   # 开机启动
service kdump status # start、stop、restart等
```

### 四、功能验证
Magic System request key is a magical key combo you can hit which the kernel will respond to regardless of whatever else it is doing, unless it is completely locked up.

使用sysrq需要编译选项CONFIG_MAGIC_SYSRQ的支持。

故意让系统崩溃，来测试kdump是否正常工作:
```shell
echo c > /proc/sysrq-trigger
```
Will perform a system crash by a NULL pointer dereference. A crash dump will be taken if configured.

Magic SysRq还有一些很有趣的值，有的具有很大的破环性，输出在/var/log/messages：

- f：call oom_kill to kill a memory hog process. 执行oom killer。
- l：shows a stack backtrace for all active CPUs. 打印出所有CPU的stack backtrace。
- m：dump current memory info. 打印出内存使用信息。
- p：dump the current registers and flags. 打印出所在CPU的寄存器信息。

### 五、捕获内核
捕获内核是一个未压缩的ELF映像文件，查看捕获内核是否加载到内存中：
```shell
cat /sys/kernel/kexec_crash_loaded
```

缩小捕获内核占用的内存：
```shell
echo N > /sys/kernel/kexec_crash_size
```
