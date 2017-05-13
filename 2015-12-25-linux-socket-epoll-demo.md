---
layout: post
title: Linux环境TCP Socket与Epoll使用备忘
comments: true
categories: Programming
tags: [Epoll, Linux, Socket, TCP, 教程]
keywords: [TCP通信,epoll]
description: "流水帐式地记录了 Linux 下 TCP Socket 通信的方法和基本的 Epoll 使用方法。"
date: 2015-12-25 19:15:13
---
流水帐式地记录了 Linux 下 TCP Socket 通信的方法和基本的 Epoll 使用方法。
没有错误处理。

## 地址解析
``` c
struct addrinfo *listen_addr; //存放解析结果。参见`man getaddrinfo`
getaddrinfo("0.0.0.0", "55553", NULL, &listen_addr); // getaddrinfo([主机名],[端口],[hint],[结果])。成功返回 `0`
// ...
freeaddrinfo(listen_addr); //释放资源，返回void
```

## 监听
这种方式只能同时处理一个连接
``` c
int fd = socket(AF_INET, SOCK_STREAM, 0); // int socket(int domain, int type, int protocol); 参见`man 3 socket` 创建文件描述符， 出错返回-1
bind(fd, listen_addr->ai_addr, listen_addr->ai_addrlen);
    // int bind(int socket, const struct sockaddr *address,socklen_t address_len);
    // 绑定地址，出错返回-1，参见`man 3 bind`
listen(fd, SOMAXCONN);
    // int listen(int fd, int backlog(最大队列长度))
    // 开始监听，出错返回-1
while (1) {
    int new_fd = accept(fd, NULL, NULL);
        // accept([fd], [监听地址], [监听地址结构体长度]) 第2，3个参数同bind()
        // 接受连接请求，若无请求则阻塞(也有可能是EAGAIN,取决于你需要什么)
        // 返回用于和对端通信的新的文件描述符,出错返回-1
    // ... handle(new_fd);
    close(new_fd); // 关闭文件描述符
}
```
<!--more-->

## 发起连接
``` c
struct addrinfo *server_addr;
getaddrinfo("127.0.0.1", "55553", NULL, &server_addr);
int server_socket = socket(AF_INET, SOCK_STREAM, 0);
connect(server_socket, server_addr->ai_addr, server_addr->ai_addrlen)
    // 基本同bind() 参见 man 3 connect
    // 成功后可用server_socket与服务器通信
// ... send(...)
// ... recv(...)
close(server_socket);
freeaddrinfo(server_addr);
```
## 传送数据
``` c
char *payload = "hello"
send(new_fd, payload, strlen(payload), 0); // send([fd], [buffer], [需发送消息长度], [flag]) 返回实际发送的消息长度
char buf[255];
recv(new_fd, buf, sizeof(buf), 0); // send([fd], [buffer], [最大接收消息长度], [flag]) 返回实际接收的消息长度。阻塞模式下，若无消息则阻塞
```
## 多进程请求处理
对于每一个请求fork()一个新的进程进行处理。
```c
while(1) {
    int new_fd = accept(fd, NULL, NULL);
    if (fork() == 0) { // fork()返回0说明是子进程
        // ... handle(new_fd);
        close(new_fd);
        break;
    }
}
```
## Epoll
``` c
int epollfd = epoll_create(1024);
    // 创建epoll文件描述符，出错返回-1
    // int epoll_create(int size) 从Linux2.6.8开始，size值被忽略，不过为保持兼容需要设定为一个正整数
struct epoll_event ev; // 记录套接字相关信息
ev.events = EPOLLIN; // 监视有数据可读事件
ev.data.fd = fd; // 文件描述符数据，其实这里可以放任何数据。
epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &ev);
    // int epoll_ctl([Epoll FD], [Operation], [fd], [epoll_event]);
    // 加入监听列表，当fd上有对应事件产生时，epoll_wait会将epoll_event填充到events_in数组里
    // 出错返回-1
struct epoll_event events_in[16];
while(1) {
    int event_count = epoll_wait(epollfd, events_in, 16, -1);
        // 等待事件，epoll_wait会将事件填充至events_in内
        // int epoll_wait([epoll fd], struct epoll_event *events, [最大事件数量], int timeout);
        // 返回 获得的事件数量，若超时且没有任何事件返回0，出错返回-1。timeout设置为-1表示无限等待。
    for (int i = 0; i<event_count; i++) { // 遍历所有事件
        if (events_in[i].data.fd == fd) { // 新连接请求
            int new_fd = accept(fd, NULL, NULL);
            ev.events = EPOLLIN; // 参见man 7 epoll 如果要使用Edge Trigger还需将new_fd设为非阻塞
            ev.data.fd = new_fd;
            epoll_ctl(epollfd, EPOLL_CTL_ADD, new_fd, &ev); // 将新连接加入监视列表
        } else {
            int new_fd = events_in[i].data.fd;
            // ... handle(new_fd);
            epoll_ctl(epollfd, EPOLL_CTL_DEL, new_fd, NULL); // 不再监听fd，最后一个参数被忽略
            close(new_fd);
        }
    }
}
```

## 拓展阅读
- [Blocking 与 Non-Blocking I/O](http://www.linux-mag.com/id/308/)
- [Epoll 的 Edge-Trigger 与 Level-Trigger](http://www.ccvita.com/515.html)
