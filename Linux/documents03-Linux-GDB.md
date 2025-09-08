# GDB
- [GDB](#gdb)
  - [CoreDump 调试](#coredump-调试)
    - [Core文件](#core文件)
    - [调试Core文件](#调试core文件)
    - [打印所有线程的backtrace信息](#打印所有线程的backtrace信息)
  - [调试正在运行的线程](#调试正在运行的线程)
    - [先找到进程的PID](#先找到进程的pid)
    - [attach](#attach)
    - [直接调试相关id进程](#直接调试相关id进程)
    - [已运行程序没有调试信息](#已运行程序没有调试信息)
  - [GDB 各种命令](#gdb-各种命令)
    - [启动退出GDB:](#启动退出gdb)
    - [运行程序](#运行程序)
    - [断点管理](#断点管理)
    - [调用栈分析](#调用栈分析)
    - [多线程调试](#多线程调试)

## CoreDump 调试

### Core文件

当程序core dump时，可能会产生core文件，它能够很大程序帮助我们定位问题。但前提是系统没有限制core文件的产生。可以使用命令limit -c查看：
```
$ ulimit -c
0
```
如果结果是0，即便程序core dump了也不会有core文件留下。
如果需要让core文件能够产生，需要这么做：
```
$ ulimit -c unlimied  #表示不限制core文件大小
$ ulimit -c 10        #设置最大大小，单位为块，一块默认为512字节
```
上面两种方式可选其一。第一种无限制，第二种指定最大产生的大小。
### 调试Core文件
调试core文件也很简单：
```
$ gdb 程序文件名 core文件名
```
具体如何调试可以参考这篇文章：
https://www.yanbinghu.com/2018/09/26/61877.html

### 打印所有线程的backtrace信息

```
thread apply all bt
```

## 调试正在运行的线程

### 先找到进程的PID
首先使用ps命令找到进程id：
```bash
ps -ef | grep 进程名
```
或者：
```bash
pgrep 进程名
```
或者：
```bash
pidof 进程名
```
### attach
假设获取到进程id为20829，则可用下面的方式调试进程：
```
$ gdb
(gdb) attach 20829
```
接下来就可以继续调试。
可能会有下面的错误提示：
```
Could not attach to process.  If your uid matches the uid of the target
process, check the setting of /proc/sys/kernel/yama/ptrace_scope, or try
again as the root user.  For more details, see /etc/sysctl.d/10-ptrace.conf
ptrace: Operation not permitted.
```
解决方法，切换到root用户：
将/etc/sysctl.d/10-ptrace.conf中的
```
kernel.yama.ptrace_scope = 1
```
修改为
```
 kernel.yama.ptrace_scope = 0
```
### 直接调试相关id进程
还可以是用这样的方式:
```
gdb hello 20829  
```
或者：
```
gdb hello --pid 20829
```
### 已运行程序没有调试信息
为了节省磁盘空间，已经运行的程序通常没有调试信息。
但如果又不能停止当前程序重新启动调试，那怎么办呢？
还有办法，那就是同样的代码，再编译出一个带调试信息的版本。
然后使用和前面提到的方式操作。对于attach方式，在attach之前，使用file命令即可：
```
$ gdb
(gdb) file hello
Reading symbols from hello...done.
(gdb)attach 20829
```

## GDB 各种命令

### 启动退出GDB:
```shell
# 启动 gdb 并加载程序
gdb ./program
# 启动 gdb 并附加到正在运行的进程
gdb -p PID

# 退出 gdb
(gdb) quit
# 或简写
(gdb) q
```

### 运行程序
```shell
# 运行程序
(gdb) run
# 或简写
(gdb) r

# 带参数运行
(gdb) run arg1 arg2
```

### 断点管理
```shell
# 在指定行设置断点
(gdb) break 10
# 或简写
(gdb) b 10

# 在函数入口设置断点
(gdb) break main
(gdb) break function_name

# 查看所有断点
(gdb) info breakpoints

# 删除断点
(gdb) delete 1  # 删除编号为1的断点
(gdb) delete    # 删除所有断点
```

### 调用栈分析
```shell
# 查看调用栈
(gdb) backtrace
# 或简写
(gdb) bt

# 切换到指定栈帧
(gdb) frame 2
# 或简写
(gdb) f 2
```

### 多线程调试
```shell
# 查看所有线程
(gdb) info threads

# 切换到指定线程
(gdb) thread 2

# 只允许当前线程执行
(gdb) set scheduler-locking on
```