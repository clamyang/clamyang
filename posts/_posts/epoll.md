---
title: epoll 文档翻译
date: 2022-09-07
comments: true
---

epoll -- I/O 事件通知函数

<!--more-->

epoll API 做的工作与 poll 类似：监控多个文件描述符，在他们中是否有 I/O 准备就绪的。epoll API 支持 edge-triggered 和 level-triggered 两种使用方式并且在监控大量文件描述符上表现很好。



epoll API 的核心概念是 epoll instance，从用户空间角度看，它是一个内核的数据结构，可以被认为是一个 container 包含两个 list：

- interest list （有时也被称为 epoll set）：进程已经注册并且有兴趣监听的文件描述符集合。
- ready list：I/O 就绪的文件描述符集合。ready list 是 interest list 的一个子集。根据这些文件描述符的 I/O 活动，内核动态的填充 ready list。



下面的系统调用提供了创建和管理 epoll instance

- epoll_create：创建一个新的 epoll instance 并返回指向这个实例的文件描述符。
- epoll_ctl：通过 epoll_ctl 注册感兴趣的文件描述符，将对象添加到 epoll instance 的 interest list 中。
- epoll_wait：等待 I/O 事件，如果当前获取不到任何事件将会阻塞调用线程。（这个系统调用可以被理解成从 epoll instance 的 ready list 中取出对象）。



LT & ET

epoll 分发事件的接口有两种表现形式，Level-triggered 和 Edge-triggered。下面内容描述了两种机制的不同之处。假设如下这个场景：

1. 通过 pipe 创建两个文件描述符，将读口（rfd）注册到 epoll instance 中。
2. pipe 的写口，向 pipe 中写入 2 KB 的数据。
3. 调用 epoll_wait 将返回 rfd 作为 ready 的文件描述符。
4. 从 rfd 中读出 1 KB 的数据。（此时还余下 1KB 的数据）
5. 完成对 epoll_wait 的调用

如果 rfd 文件描述符使用 EPOLLET 标志被加入 epoll instance 中，在第 5 步对 epoll_wait 的调用会被阻塞，尽管在文件输入缓冲中仍然可以有数据。与此同时，发送者等待一个基于它请求的响应。

这个原因（buffer 中有数据，但是第二次调用 epoll_wait 不返回）在于，edge-triggered 模式下，只有当监听列表中的文件描述符发生变化时才会派发事件。所以，在第 5 步，调用者会等待数据的到来，但是数据其实已经在输入缓冲区中了（存留的 1 KB 数据。）

在上面例子中，由于在第二步完成了写操作，所以会产生一个 rfd 的事件，然后在第 3 步中消费这个事件。因为在第 4 步中的读操作没有读取所有 buffer 中的数据，在第 5 步对 epoll_wait 的调用会无止境的等待下去。

使用 EPOLLET 标志的应用应该使用非阻塞的文件描述符来避免阻塞读或写饿死一个处理多个文件描述符的任务。将 epoll 用作边缘触发的建议如下：

- 使用非阻塞的文件描述符
- 只有当 read 或 write 返回 EAGAIN 后才等待事件，（才调用 epoll_wait 我认为是对 epoll_wait 的调用）

相反，当使用 LT 模式，epoll 是一个更快的 poll，可以像 poll 那样被使用，因为他们语义是相同的。

因为即使使用 ET 模式的 epoll，在收到多个数据块时，会生成多个事件，调用者可以选择指定 EPOLLONESHOT 标志，告诉 epoll 在通过 epoll_wait 收到相关事件后关闭关联的文件描述符（应该是将文件描述符从 epoll instance 中移除）。当 EPOLLONESHOT 指定后，调用者有责任使用带 EPOLL_CTL_MOD 参数的 epoll_ctl 函数，将文件描述符重新加入进来。

如果多个线程被阻塞在 epoll_wait 上， 等待同一个 epoll instance 文件描述符并且一个在 interest list 中的文件描述符就绪了，此刻，只有一个线程会被唤醒。在某些场景下，这样做避免了 ”惊群“。



建议用法示例

当 epoll 以 LT 方式使用时语义与 poll 相同，ET 的使用需要额外的说明以避免在事件循环时暂停。在这个例子中，listener 是一个非阻塞的 socket 已在其上调用了 listen 方法。（这里感觉还是挺奇怪的）

> int listen(int sockfd, int backlog);
>
> 可见第一个参数是 sockfd 就是下面例子中的 listen_sock 



函数 do_use_fd() 使用就绪的文件描述符直到 EAGAIN 被 read 或 write 返回。在接收到 EAGAIN 后，事件驱动的状态机应该记录它当前的状态方便下一次调用 do_use_fd() 它将继续 read 或 write 从上一次结束的位置。

```c
#define MAX_EVENTS 10
struct epoll_event ev, events[MAX_EVENTS];
int listen_sock, conn_sock, nfds, epollfd;

/* Code to set up listening socket, 'listen_sock',
              (socket(), bind(), listen()) omitted. */

epollfd = epoll_create1(0);
if (epollfd == -1) {
    perror("epoll_create1");
    exit(EXIT_FAILURE);
}

ev.events = EPOLLIN;
ev.data.fd = listen_sock;
if (epoll_ctl(epollfd, EPOLL_CTL_ADD, listen_sock, &ev) == -1) {
    perror("epoll_ctl: listen_sock");
    exit(EXIT_FAILURE);
}

for (;;) {
    nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);
    if (nfds == -1) {
        perror("epoll_wait");
        exit(EXIT_FAILURE);
    }

    for (n = 0; n < nfds; ++n) {
        if (events[n].data.fd == listen_sock) {
            conn_sock = accept(listen_sock,
                               (struct sockaddr *) &addr, &addrlen);
            if (conn_sock == -1) {
                perror("accept");
                exit(EXIT_FAILURE);
            }
            setnonblocking(conn_sock);
            ev.events = EPOLLIN | EPOLLET;
            ev.data.fd = conn_sock;
            if (epoll_ctl(epollfd, EPOLL_CTL_ADD, conn_sock,
                          &ev) == -1) {
                perror("epoll_ctl: conn_sock");
                exit(EXIT_FAILURE);
            }
        } else {
            do_use_fd(events[n].data.fd);
        }
    }
}
```



当作为 ET 模式使用时，由于性能原因，在 epoll 接口中一次性指定 EPOLLIN|EPOLLOUT 是可行的。这允许你避免连续地调用 epoll_ctl 在 EPOLLIN 和 EPOLLOUT 之间切换。