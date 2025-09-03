# Epoll
- [Epoll](#epoll)
  - [Epoll 介绍](#epoll-介绍)
  - [Epoll系统调用函数](#epoll系统调用函数)
    - [epoll\_create](#epoll_create)
    - [epoll\_ctl](#epoll_ctl)
    - [epoll\_wait](#epoll_wait)
  - [LT模式和ET模式](#lt模式和et模式)
    - [LT模式](#lt模式)
    - [ET模式](#et模式)
    - [总结](#总结)

## Epoll 介绍

## Epoll系统调用函数
```c
// 数据结构
// 每一个epoll对象都有一个独立的eventpoll结构体
// 红黑树用于存放通过epoll_ctl方法向epoll对象中添加进来的事件
// epoll_wait检查是否有事件发生时，只需要检查eventpoll对象中的rdlist双链表中是否有epitem元素即可
struct eventpoll {
    ...
    // 红黑树的根节点，这颗树存储着所有添加到epoll中的需要监控的事件
    struct rb_root  rbr;
    // 双链表存储所有就绪的文件描述符
    struct list_head rdlist;
    ...
};

// 内核中间加一个 eventpoll 对象，把所有需要监听的 socket 都放到 eventpoll 对象中
int epoll_create(int size);
// epoll_ctl 负责把 socket 增加、删除到内核红黑树
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
// epoll_wait 检测双链表中是否有就绪的文件描述符，如果有，则返回
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```
简而言之，epoll 有以下几个特点: 
- 使用红黑树存储文件描述符集合
- 使用双链表存储就绪的文件描述符
- 每个文件描述符只需在添加时传入一次，通过事件 callback 更改文件描述符状态。

select、poll 模型都只使用一个函数，而 epoll 模型使用三个函数: 
- epoll_create
- epoll_ctl
- epoll_wait。

epoll_create 创建 eventpoll对象（红黑树，双链表）
- 一棵红黑树，存储监听的所有文件描述符，并且通过 epoll_ctl 将文件描述符添加、删除到红黑树
- 一个双链表，存储就绪的文件描述符列表，epoll_wait调用时，检测此链表中是否有数据，有的话直接返回
- 所有添加到 eventpoll 中的事件都与设备驱动程序建立回调关系

### epoll_create

```c
// 返回一个 int 表示 epoll fd
// 参数 size 表示这个 epoll fd 最多能监听多少 fd
int epoll_create(int size);
```

内核在 epoll 文件系统中建了个 file 结点
- epoll_create 会创建一个 epoll 实例，同时返回一个引用该实例的文件描述符
- 返回的文件描述符仅仅指向对应的 epoll 实例，并不表示真实的磁盘文件节点。其他 API 如 epoll_ctl、epoll_wait 会使用这个文件描述符来操作相应的 epoll 实例
- 使用完，必须调用close()关闭，否则导致fd被耗尽

epoll 实例内部存储: 
- 监听列表：所有要监听的文件描述符，使用红黑树，由 epoll_ctl 传来
- 就绪列表：所有就绪的文件描述符，使用双向链表

### epoll_ctl
epoll_ctl 会监听文件描述符 fd 上发生的 event 事件

```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```
- epfd 即 epoll_create 返回的文件描述符，指向一个 epoll 实例
- fd 表示要监听的目标文件描述符
- event 表示要监听的事件（可读、可写、发送错误…）
- op 表示要对 fd 执行的操作，有以下几种：
  - EPOLL_CTL_ADD：为 fd 添加一个监听事件 event
  - EPOLL_CTL_MOD：Change the event event associated with the target file descriptor fd（event 是一个结构体变量，这相当于变量 event 本身没变，但是更改了其内部字段的值）
  - EPOLL_CTL_DEL：删除 fd 的所有监听事件，这种情况下 event 参数没用
- 返回值 0 或 -1，表示上述操作成功与否。
- Errno
  - EBADF  epfd 或 fd 不是有效的文件描述符
  - EINVAL 各种参数无效情况。如 epfd 不是 epoll 文件描述符，或者 fd 与 epfd 相同，或者此接口不支持请求的操作 fd
  - EMFILE 超过了`/proc/sys/fs/epoll/max_user_instances`限制
  - ENFILE 已达到 FD 总数限制
  - ENOMEM 没有足够内存
  - EEXIST op 为 EPOLL_CTL_ADD，并且 fd 已在该 epoll 实例中注册
  - ENOMEM 没有足够的内存来处理请求的操作控制操作
  - ENOSPC 尝试在主机上注册（EPOLL_CTL_ADD）新文件符时遇到了 `/proc/sys/epoll/max_user_watches` 定义的限制
  - EPERM  目标 fd 不支持 epoll
  - ENOENT op 为 EPOLL_CTL_MOD 或 EPOLL_CTL_DEL，并且 fd 未在该 epoll 实例中注册
  - **注意: 在最开始的 epoll_create 实现中，size 参数将调用者希望添加到的文件描述符的数量告知内核。内核使用该信息作为初始分配空间的提示。如今此提示已不再必须，但是大小必须大于 0，以便新的 epoll 应用程序在较旧内核上运行时，保证兼容。**

epoll_ctl 会将文件描述符 fd 添加到 epoll 实例的监听列表里，同时为 fd 设置一个回调函数，并监听事件 event，如果红黑树中已经存在立刻返回。当 fd 上发生相应事件时，会调用回调函数，将 fd 添加到 epoll 实例的就绪队列上。

### epoll_wait
```c
// epoll_wait 检测双链表中是否有就绪的文件描述符，如果有，则返回
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

epoll_wait 系统调用等待 epfd 引用的 epoll 实例上的事件。事件所指向的存储区域将包含可供调用者使用的事件。

- epfd 即 epoll_create 返回的文件描述符，指向一个 epoll 实例
- events 接口的返回参数，一般都是一个数组，数组长度大于等于maxevents。
- maxevents：期望捕获的事件的个数 参数必须大于零。
- timeout 超时时间，单位 ms。指定为 -1 则无限期阻塞，指定为 0 则不阻塞。
- 返回值 正常捕获事件后返回事件的个数，超时返回 0
  - 只有在下面的情况下才会返回：
    - 1、有至少一个事件发生
    - 2、调用过程中被信号中断
    - 3、超时
- Errno
  - EBADF epfd 不是有效的文件描述符
  - EFAULT 事件的内存区域无效， events 参数指向位置不可访问或不可写入
  - EINTR 任何请求的事件发生或到期之前，信号处理程序中断了该调用
  - EINVAL epfd 不是 epoll fd，或者 maxevents 小于等于零

## LT模式和ET模式
与 poll 的事件宏相比，epoll 新增了一个事件宏 EPOLLET，这就是所谓的边缘触发模式（Edge Trigger，ET），而默认的模式我们称为 水平触发模式（Level Trigger，LT）。这两种模式的区别在于：

- 对于水平触发模式，一个事件只要有数据没处理完，就会一直触发；
- 对于边缘触发模式，只有一个事件从无到有才会触发。

一般都用 ET 的，如果用 LT，来一个大的请求，处理不完就处理不了其他的事件，导致其他事件饿死。


### LT模式
对于读事件 EPOLLIN，只要socket上有未读完的数据，EPOLLIN 就会一直触发；对于写事件 EPOLLOUT，只要socket可写（一说指的是 TCP 窗口一直不饱和，我觉得是TCP缓冲区未满时，这一点还需验证），EPOLLOUT 就会一直触发。

在这种模式下，大家会认为读数据会简单一些，因为即使数据没有读完，那么下次调用epoll_wait()时，它还会通知你在上没读完的文件描述符上继续读，也就是人们常说的这种模式不用担心会丢失数据。

而写数据时，因为使用 LT 模式会一直触发 EPOLLOUT 事件，那么如果代码实现依赖于可写事件触发去发送数据，一定要在数据发送完之后移除检测可写事件，避免没有数据发送时无意义的触发。

### ET模式
对于读事件 EPOLLIN，只有socket上的数据从无到有，EPOLLIN 才会触发；对于写事件 EPOLLOUT，只有在socket写缓冲区从不可写变为可写，EPOLLOUT 才会触发（刚刚添加事件完成调用epoll_wait时或者缓冲区从满到不满）

这种模式听起来清爽了很多，只有状态变化时才会通知，通知的次数少了自然也会引发一些问题，比如触发读事件后必须把数据收取干净，因为你不一定有下一次机会再收取数据了，即使不采用一次读取干净的方式，也要把这个激活状态记下来，后续接着处理，否则如果数据残留到下一次消息来到时就会造成延迟现象。

这种模式下写事件触发后，后续就不会再触发了，如果还需要下一次的写事件触发来驱动发送数据，就需要再次注册一次检测可写事件。

### 总结
- LT模式会一直触发EPOLLOUT，当缓冲区有数据时会一直触发EPOLLIN
- ET模式会在连接建立后触发一次EPOLLOUT，当收到数据时会触发一次EPOLLIN
- LT模式触发EPOLLIN时可以按需读取数据，残留了数据还会再次通知读取
- ET模式触发EPOLLIN时必须把数据读取完，否则即使来了新的数据也不会再次通知了
- LT模式的EPOLLOUT会一直触发，所以发送完数据记得删除，否则会产生大量不必要的通知
- ET模式的EPOLLOUT事件若数据未发送完需再次注册，否则不会再有发送的机会
- 通常发送网络数据时不会依赖EPOLLOUT事件，只有在缓冲区满发送失败时会注册这个事件，期待被通知后再次发送

