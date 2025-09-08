# Crash
- [Crash](#crash)
  - [Reference](#reference)
  - [Crash 工具](#crash-工具)

## Reference
[【调试】crash使用方法 ](https://www.cnblogs.com/dongxb/p/17364995.html)
[crash 工具的使用](https://zhuanlan.zhihu.com/p/707500778)

## Crash 工具
crash是redhat的工程师开发的，主要用来离线分析linux内核转存文件，它整合了gdb工具，功能非常强大。可以查看堆栈，dmesg日志，内核数据结构，反汇编等等。

crash支持多种工具生成的转存文件格式，如kdump，LKCD，netdump和diskdump，而且还可以分析虚拟机Xen和Kvm上生成的内核转存文件。同时crash还可以调试运行时系统，直接运行crash即可，ubuntu下内核映象存放在/proc/kcore。





