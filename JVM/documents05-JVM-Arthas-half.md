# Arthas
- [Arthas](#arthas)
  - [介绍](#介绍)
  - [Reference](#reference)
  - [常用命令操作](#常用命令操作)
    - [CAT 查看文件内容](#cat-查看文件内容)
    - [GREP 匹配查找](#grep-匹配查找)
    - [PWD 查看当前工作目录](#pwd-查看当前工作目录)
    - [CLS 清空屏幕](#cls-清空屏幕)
    - [RESET 重置](#reset-重置)
    - [HISTORY 查看历史指令](#history-查看历史指令)
    - [SESSION 查看当前会话](#session-查看当前会话)
    - [VERSION 查看当前版本](#version-查看当前版本)
    - [KEYMAP 查看所有快捷键](#keymap-查看所有快捷键)
    - [QUIT / EXIT / STOP 退出](#quit--exit--stop-退出)
    - [DASHBOARD 仪表盘](#dashboard-仪表盘)
    - [THREAD 获取线程信息](#thread-获取线程信息)
    - [JVM 获取当前JVM信息](#jvm-获取当前jvm信息)
    - [JAD 反编译 class 获取源码](#jad-反编译-class-获取源码)
    - [MC 将 java 编译成 class](#mc-将-java-编译成-class)
    - [REDEFINE 将外部 class 文件加载到 JVM 中](#redefine-将外部-class-文件加载到-jvm-中)
    - [GETSTATIC 查看类的静态属性](#getstatic-查看类的静态属性)
    - [OGNL 执行 ognl 表达式](#ognl-执行-ognl-表达式)
    - [SC 查看 JVM 已经加载的类的信息](#sc-查看-jvm-已经加载的类的信息)
    - [SM 查看已加载类的信息](#sm-查看已加载类的信息)
    - [DUMP 加载类的字节码文件保存到特定目录](#dump-加载类的字节码文件保存到特定目录)
    - [CLASSLOADER 获取类加载器的信息](#classloader-获取类加载器的信息)
    - [MONITOR 监控指定类中方法的执行情况](#monitor-监控指定类中方法的执行情况)
    - [WATCH 观察到指定方法的调用情况](#watch-观察到指定方法的调用情况)
    - [TRACE 方法调用耗时追踪](#trace-方法调用耗时追踪)
    - [STACK 查看方法调用链](#stack-查看方法调用链)
    - [TT 查看方法的出入参数](#tt-查看方法的出入参数)
    - [SYSPROP 查看和修改 JVM 的系统属性](#sysprop-查看和修改-jvm-的系统属性)
    - [SYSENV 查看当前 JVM 的环境属性](#sysenv-查看当前-jvm-的环境属性)
    - [VMOPTION 查看，更新 VM 诊断相关的参数](#vmoption-查看更新-vm-诊断相关的参数)
    - [PROFILER 火焰图](#profiler-火焰图)

## 介绍
Arthas 是一款线上监控诊断产品，通过全局视角实时查看应用 load、内存、gc、线程的状态信息，并能在不修改应用代码的情况下，对业务问题进行诊断，包括查看方法调用的出入参、异常，监测方法执行耗时，类加载信息等，大大提升线上问题排查效率。

Arthas 是 Alibaba 开源的 Java 诊断工具，深受开发者喜爱。

当你遇到以下类似问题而束手无策时，Arthas可以帮助你解决：

- 这个类从哪个 jar 包加载的？为什么会报各种类相关的 Exception？
- 我改的代码为什么没有执行到？难道是我没 commit？分支搞错了？
- 遇到问题无法在线上 debug，难道只能通过加日志再重新发布吗？
- 线上遇到某个用户的数据处理有问题，但线上同样无法 debug，线下无法重现！
- 是否有一个全局视角来查看系统的运行状况？
- 有什么办法可以监控到 JVM 的实时运行状态？
- 怎么快速定位应用的热点，生成火焰图？
- 怎样直接从 JVM 内查找某个类的实例？
- Arthas 支持 JDK 6+（4.x 版本不再支持 JDK 6 和 JDK 7），支持 Linux/Mac/Windows，采用命令行交互模式，同时提供丰富的 Tab 自动补全功能，进一步方便进行问题的定位和诊断。

## Reference
[Arthas指令大全](https://blog.csdn.net/qq_43692950/article/details/122686329)

## 常用命令操作

### CAT 查看文件内容

打印文件内容，和Linux里的cat命令类似，如果没有写路径，则显示当前目录下的文件
```shell
cat a.txt
```

### GREP 匹配查找
匹配查找，和Linux里的grep命令类似，但它只能用于管道

语法
| 参数            | 作用                                 |
| --------------- | ------------------------------------ |
| -n              | 显示行号                             |
| -i              | 忽略大小写查找                       |
| -m 行数         | 最大显示行数，要与查询字符串一起使用 |
| -e "正则表达式" | 使用正则表达式查找                   |

查看系统属性中包含Java的行: 
```shell
sysprop | grep java
```

### PWD 查看当前工作目录
```shell
pwd
```
### CLS 清空屏幕
```shell
cls
```

### RESET 重置
重置增强类，将被Arthas增强过的类全部还原，Arthas服务端关闭时会重置所有增强过的类

还原指定类
```shell
reset Test
```

还原所有以List结尾的类
```shell
reset *List
```

还原所有的类
```shell
reset
```

### HISTORY 查看历史指令
打印命令历史
```shell
history
```

### SESSION 查看当前会话
查看当前会话的信息
```shell
session
```

### VERSION 查看当前版本
输出当前目标Java进程加载的Arthas版本号。
```shell
version
```

### KEYMAP 查看所有快捷键
Arthas快捷键列表及自定义快捷键
```shell
keymap
```

### QUIT / EXIT / STOP 退出
如果只是退出当前连接，可以使用 quit 或者 exit 命令。Attach到目标进程上的Arthas还会继续，端口会保持开放，下次可以直接连上。

如果想完全退出Arthas，可以执行 stop 命令。

### DASHBOARD 仪表盘




### THREAD 获取线程信息
| 参数          | 说明                                |
| ------------- | ----------------------------------- |
| 数字          | 线程ID                              |
| -n 数字       | 指定最忙的前N个线程并打印堆栈       |
| -b           | 找出当前阻塞其他线程的线程          |
| -i \<value\> | 指定CPU占比统计采样的间隔，单位毫秒 |

当没有参数时，显示所有线程的信息
```shell
thread
```

展示当前最忙的3个线程并打印堆栈
```shell
thread -n 3
```

显示1号线程的运行堆栈
```shell
thread 1
```

找出当前阻塞其他线程的线程
```shell
thread -b
```

指定采样间隔，每隔1000ms采样，显示最忙的3个线程
```shell
thread -i 1000 -n 3
```

查看处于等待状态的线程
```shell
thread --state WAITING
```

### JVM 获取当前JVM信息
Thread相关
- COUNT
  - JVM当前活跃线程数
- DAEMON-COUNT
  - JVM当前活跃的守护线程数
- PEAK-COUNT
  - 从JVM启动开始曾经活着的最大线程数
- STARTED-COUNT
  - 从JVM启动开始总共启动过的线程次数
- DEADLOCK-COUNT
  - JVM当前死锁的线程数
文件描述符相关
- MAX-FILE-DESCRIPTOR-COUNT
  - JVM进程最大可以打开的文件描述符数
- OPEN-FILE-DESCRIPTOR-COUNT
  - JVM当前打开的文件描述符数

### JAD 反编译 class 获取源码
jad 命令将 JVM 中实际运行的 class 的 bytecode 反编译为 java 代码，方便你理解业务逻辑。

在 Arthas Console 中，反编译出来的源码是自带语法高亮的，阅读更方便。

当然，反编译出来的 Java 代码可能会存在语法错误，但不影响你进行阅读理解。

参数说明
| 名称          | 说明                                 |
| ------------- | ------------------------------------ |
| class-pattern | 类名表达式匹配                       |
| [E]           | 开启正则表达式匹配，默认为通配符匹配 |

编译 `java.lang.String`
```shell
jad java.lang.String
```

反编译只显示源代码，默认情况下，反编译结果里会带有 ClassLoader 信息，通过 -source-only 选项，可以只打印源代码。方便和 mc/redefine 命令结合使用。
```shell
jad --source-only demo.MathGame
```

反编译指定的方法
```shell
jad demo.MathGame main
```

### MC 将 java 编译成 class
Memory Compiler / 内存编译器，编译 java 文件 生成 class

内存编译 Hello.java 为 Hello.class
```shell
mc /root/Hello.java
```

可以通过 -d 指定输出目录
```shell
mc -d /root/some/dir /root/Hello.java
```

### REDEFINE 将外部 class 文件加载到 JVM 中
加载外部的 class 文件， redefine 到 JVM 里

> **注意** redefine 之后，原来的类不能恢复， redefine 可能失败。 reset 命令对 redefine 的类无效。如果想重置，需要 redefine 原始的字节码。 redefine 命令和 jad/watch/trace/monitor/tt 等命令会冲突。执行完 redefine 之后，如果再执行上面提到的命令，则会把 redefine 的字节码重置。

redefine 的限制
- 不允许新增 field/method
- 正在跑的函数没有退出不能生效

使用 redefine 命令加载新的字节码
```shell
redefine /root/demo/MathGame.class
```

### GETSTATIC 查看类的静态属性
### OGNL 执行 ognl 表达式
### SC 查看 JVM 已经加载的类的信息
### SM 查看已加载类的信息
### DUMP 加载类的字节码文件保存到特定目录
### CLASSLOADER 获取类加载器的信息
### MONITOR 监控指定类中方法的执行情况
### WATCH 观察到指定方法的调用情况
### TRACE 方法调用耗时追踪
### STACK 查看方法调用链
### TT 查看方法的出入参数
### SYSPROP 查看和修改 JVM 的系统属性
### SYSENV 查看当前 JVM 的环境属性
### VMOPTION 查看，更新 VM 诊断相关的参数
### PROFILER 火焰图
