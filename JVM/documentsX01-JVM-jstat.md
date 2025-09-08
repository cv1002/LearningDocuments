# JSTAT
- [JSTAT](#jstat)
  - [介绍](#介绍)
  - [使用](#使用)
    - [输出选项](#输出选项)
      - [-class](#-class)
      - [-compiler](#-compiler)
      - [-gc](#-gc)
      - [-gccapacity](#-gccapacity)
      - [-gcutil](#-gcutil)
      - [-gccause选项](#-gccause选项)
      - [-gcnew](#-gcnew)
      - [-gcnewcapacity](#-gcnewcapacity)
      - [-gcold](#-gcold)
      - [-gcoldcapacity](#-gcoldcapacity)
      - [-gcmetacapacity](#-gcmetacapacity)
      - [-printcompilation](#-printcompilation)
  - [常用](#常用)

## 介绍
jstat（Java Virtual Machine Statistics Monitoring Tool）是JDK提供的一个可以监控Java虚拟机各种运行状态信息的命令行工具。它可以显示Java虚拟机中的类加载、内存、垃圾收集、即时编译等运行状态的信息。

## 使用

命令语法: 
```shell
jstat generalOptions
jstat outputOptions [-t] [-h<lines>] <vmid> [<interval> [<count>]]
```
命令参数说明: 

- `generalOptions`: 通用选项，如果指定一个通用选项，就不能指定任何其他选项或参数。它包括如下两个选项: 
  - `-help`: 显示帮助信息。
  - `-options`: 显示`outputOptions`参数的列表。
- `outputOptions`: 输出选项，指定显示某一种Java虚拟机信息。
- `-t`: 把时间戳列显示为输出的第一列。这个时间戳是从Java虚拟机的开始运行到现在的秒数。
- `-h n`: 每显示n行显示一次表头，其中n为正整数。默认值为 0，即仅在第一行数据显示一次表头。
- `vmid`: 虚拟机唯一ID（LVMID，Local Virtual Machine Identifier），如果查看本机就是Java进程的进程ID。
- `interval`: 显示信息的时间间隔，单位默认毫秒。也可以指定秒为单位，比如: 1s。如果指定了该参数，jstat命令将每个这段时间显示一次统计信息。
- `count`: 显示数据的次数，默认值是无穷大，这将导致jstat命令一直显示统计信息，直到目标JVM终止或jstat命令终止。

### 输出选项
如果不指定通用选项（generalOptions），则可以指定输出选项（outputOptions）。输出选项决定jstat命令显示的内容和格式，具体如下: 

- `-class`: 显示类加载、卸载数量、总空间和装载耗时的统计信息。
- `-compiler`: 显示即时编译的方法、耗时等信息。
- `-gc`: 显示堆各个区域内存使用和垃圾回收的统计信息。
- `-gccapacity`: 显示堆各个区域的容量及其对应的空间的统计信息。
- `-gcutil`: 显示有关垃圾收集统计信息的摘要。
- `-gccause`: 显示关于垃圾收集统计信息的摘要(与-gcutil相同)，以及最近和当前垃圾回收的原因。
- `-gcnew`: 显示新生代的垃圾回收统计信息。
- `-gcnewcapacity`: 显示新生代的大小及其对应的空间的统计信息。
- `-gcold`: 显示老年代和元空间的垃圾回收统计信息。
- `-gcoldcapacity`: 显示老年代的大小统计信息。
- `-gcmetacapacity`: 显示元空间的大小的统计信息。
- `-printcompilation`: 显示即时编译方法的统计信息。

#### -class
- Loaded: 加载的类的数量。
- Bytes: 加载的类所占用的字节数。
- Unloaded: 卸载的类的数量。
- Bytes: 卸载的类所占用的字节数。
- Time: 执行类加载和卸载操作所花费的时间。

#### -compiler
- Compiled: 执行的编译任务的数量。
- Failed: 执行编译任务失败的数量。
- Invalid: 执行编译任务失效的数量。
- Time: 执行编译任务所花费的时间。
- FailedType: 上次编译失败的编译类型。
- FailedMethod: 上次编译失败的类名和方法。

#### -gc
- S0C: 年轻代中第一个Survivor区的容量，单位为KB。
- S1C: 年轻代中第二个Survivor区的容量，单位为KB。
- S0U: 年轻代中第一个Survivor区已使用大小，单位为KB。
- S1U: 年轻代中第二个Survivor区已使用大小，单位为KB。
- EC: 年轻代中Eden区的容量，单位为KB。
- EU: 年轻代中Eden区已使用大小，单位为KB。
- OC: 老年代的容量，单位为KB。
- OU: 老年代已使用大小，单位为KB。
- MC: 元空间的容量，单位为KB。
- MU: 元空间已使用大小，单位为KB。
- CCSC: 压缩类的容量，单位为KB。
- CCSU: 压缩类已使用大小，单位为KB。
- YGC: Young GC的次数。
- YGCT: Young GC所用的时间。
- FGC: Full GC的次数。
- FGCT: Full GC的所用的时间。
- GCT: GC的所用的总时间。

#### -gccapacity
- NGCMN: 年轻代最小的容量，单位为KB。
- NGCMX: 年轻代最大的容量，单位为KB。
- NGC: 当前年轻代的容量，单位为KB。
- S0C: 年轻代中第一个Survivor区的容量，单位为KB。
- S1C: 年轻代中第二个Survivor区的容量，单位为KB。
- EC: 年轻代中Eden区的容量，单位为KB。
- OGCMN: 老年代最小的容量，单位为KB。
- OGCMX: 老年代最大的容量，单位为KB。
- OGC: 当前老年代的容量，单位为KB。
- OC: 当前老年代的容量，单位为KB。
- MCMN: 元空间最小的容量，单位为KB。
- MCMX: 元空间最大的容量，单位为KB。
- MC: 当前元空间的容量，单位为KB。
- CCSMN: 压缩类最小的容量，单位为KB。
- CCSMX: 压缩类最大的容量，单位为KB。
- CCSC: 当前压缩类的容量，单位为KB。
- YGC: Young GC的次数。
- FGC: Full GC的次数。

#### -gcutil
- S0: 年轻代中第一个Survivor区使用大小占当前容量的百分比。
- S1: 年轻代中第二个Survivor区使用大小占当前容量的百分比。
- E: Eden区使用大小占当前容量的百分比。
- O: 老年代使用大小占当前容量的百分比。
- M: 元空间使用大小占当前容量的百分比。
- CCS: 压缩类使用大小占当前容量的百分比。
- YGC: Young GC的次数。
- YGCT: Young GC所用的时间。
- FGC: Full GC的次数。
- FGCT: Full GC的所用的时间。
- GCT: GC的所用的总时间。

#### -gccause选项
- S0: 年轻代中第一个Survivor区使用大小占当前容量的百分比。
- S1: 年轻代中第二个Survivor区使用大小占当前容量的百分比。
- E: Eden区使用大小占当前容量的百分比。
- O: 老年代使用大小占当前容量的百分比。
- M: 元空间使用大小占当前容量的百分比。
- CCS: 压缩类使用大小占当前容量的百分比。
- YGC: Young GC的次数。
- YGCT: Young GC所用的时间。
- FGC: Full GC的次数。
- FGCT: Full GC的所用的时间。
- GCT: GC的所用的总时间。
- LGCC: 上次垃圾回收的原因。
- GCC: 当前垃圾回收的原因。

#### -gcnew
- S0C: 年轻代中第一个Survivor区的容量，单位为KB。
- S1C: 年轻代中第二个Survivor区的容量，单位为KB。
- S0U: 年轻代中第一个Survivor区已使用大小，单位为KB。
- S1U: 年轻代中第二个Survivor区已使用大小，单位为KB。
- TT: 对象在年轻代存活的次数。
- MTT: 对象在年轻代存活的最大次数
- DSS: 期望的Survivor区大小，单位为KB。
- EC: 年轻代中Eden区的容量，单位为KB。
- EU: 年轻代中Eden区已使用大小，单位为KB。
- YGC: Young GC的次数。
- YGCT: Young GC所用的时间。

#### -gcnewcapacity
- NGCMN: 年轻代最小的容量，单位为KB。
- NGCMX: 年轻代最大的容量，单位为KB。
- NGC: 当前年轻代的容量，单位为KB。
- S0CMX: 年轻代中第一个Survivor区最大的容量，单位为KB。
- S0C: 年轻代中第一个Survivor区的容量，单位为KB。
- S1CMX: 年轻代中第二个Survivor区最大的容量，单位为KB。
- S1C: 年轻代中第二个Survivor区的容量，单位为KB。
- ECMX: 年轻代中Eden区最大的容量，单位为KB。
- EC: 年轻代中Eden区的容量，单位为KB。
- YGC: Young GC的次数。
- FGC: Full GC的次数。

#### -gcold
- MC: 元空间的容量，单位为KB。
- MU: 元空间已使用大小，单位为KB。
- CCSC: 压缩类的容量，单位为KB。
- CCSU: 压缩类已使用大小，单位为KB。
- OC: 老年代的容量，单位为KB。
- OU: 老年代已使用大小，单位为KB。
- YGC: Young GC的次数。
- FGC: Full GC的次数。
- FGCT: Full GC的所用的时间。
- GCT: GC的所用的总时间。

#### -gcoldcapacity
- OGCMN: 老年代最小的容量，单位为KB。
- OGCMX: 老年代最大的容量，单位为KB。
- OGC: 当前老年代的容量，单位为KB。
- OC: 当前老年代的容量，单位为KB。
- YGC: Young GC的次数。
- FGC: Full GC的次数。
- FGCT: Full GC的所用的时间。
- GCT: GC的所用的总时间。

#### -gcmetacapacity
- MCMN: 元空间最小的容量，单位为KB。
- MCMX: 元空间最大的容量，单位为KB。
- MC: 当前元空间的容量，单位为KB。
- CCSMN: 压缩类最小的容量，单位为KB。
- CCSMX: 压缩类最大的容量，单位为KB。
- YGC: Young GC的次数。
- FGC: Full GC的次数。
- FGCT: Full GC的所用的时间。
- GCT: GC的所用的总时间。

#### -printcompilation
- Compiled: 最近编译方法执行的编译任务的数量。
- Size: 最近编译方法的字节码的字节数。
- Type: 最近编译方法的编译类型。
- Method: 最近编译方法的类名和方法名。

## 常用

`jstat -gc pid` **最常用**，可以评估程序内存使用及GC压力整体情况。

每一列的意思:
- S0C: 年轻代中第一个Survivor区的容量，单位为KB。
- S1C: 年轻代中第二个Survivor区的容量，单位为KB。
- S0U: 年轻代中第一个Survivor区已使用大小，单位为KB。
- S1U: 年轻代中第二个Survivor区已使用大小，单位为KB。
- EC: 年轻代中Eden区的容量，单位为KB。
- EU: 年轻代中Eden区已使用大小，单位为KB。
- OC: 老年代的容量，单位为KB。
- OU: 老年代已使用大小，单位为KB。
- MC: 元空间的容量，单位为KB。
- MU: 元空间已使用大小，单位为KB。
- CCSC: 压缩类的容量，单位为KB。
- CCSU: 压缩类已使用大小，单位为KB。
- YGC: Young GC的次数。
- YGCT: Young GC所用的时间。
- FGC: Full GC的次数。
- FGCT: Full GC的所用的时间。
- GCT: GC的所用的总时间。

Example:
```shell
jstat -gc 21968
```
![JstatGcInfoExample](assets/jstat-example.png)

