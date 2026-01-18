---
title: "IO复用"
date: 2025-03-02
categories:
  - Linux高性能服务器编程
---

# IO复用

为什么要封装？

封装各个接口简化调用，并且方便使用wait然后循环遍历各个事件，根据不同的事件处理不同的逻辑

## epoll

存储形式

int epollFd_;   构造的时候epoll_create(512) 返回fd并指定最大可监听512个，较新的linux内核用epoll_create1

std::vector<struct epoll_event> events_;  wait的时候存储监听到的事件

监听事件：

```
基础事件
EPOLLIN: 可读事件
EPOLLOUT: 可写事件
EPOLLRDHUP: TCP连接对端关闭或者半关闭
EPOLLERR: 错误事件
EPOLLHUP: 挂起事件
EPOLLPRI: 紧急数据到达

事件修饰符
EPOLLET: 边缘触发模式
EPOLLONESHOT: 一次性触发，触发后需要重新添加
EPOLLWAKEUP: 当事件触发时防止系统休眠
EPOLLEXCLUSIVE: 对于同一个文件描述符，在多个 epoll 实例中只有一个会触发（从 Linux 4.5 开始支持）

最常用的组合是：
EPOLLIN | EPOLLET: 边缘触发的读事件
EPOLLIN | EPOLLOUT | EPOLLET: 边缘触发的读写事件
EPOLLIN | EPOLLRDHUP: 用于检测对端关闭
```



```c
struct epoll_event
{
  uint32_t events;
  epoll_data_t data;
} __EPOLL_PACKED;

typedef union epoll_data
{
  void *ptr;
  int fd;
  uint32_t u32;
  uint64_t u64;
} epoll_data_t;
```

### epoller.h

```c++
#ifndef EPOLLER_H
#define EPOLLER_H

#include <sys/epoll.h> //epoll_ctl()
#include <unistd.h> // 对 POSIX 操作系统 API 的访问
#include <assert.h> 
#include <vector>
#include <errno.h>

class Epoller {
public:
    explicit Epoller(int maxEvent = 1024);
    ~Epoller();

    bool AddFd(int fd, uint32_t events);
    bool ModFd(int fd, uint32_t events);//修改指定文件描述符的事件类型
    bool DelFd(int fd);
    int Wait(int timeoutMs = -1);//等待事件发生并返回事件的个数 
    int GetEventFd(size_t i) const;
    uint32_t GetEvents(size_t i) const;
        
private:
    int epollFd_;
    std::vector<struct epoll_event> events_; //或者epoll_event events[MAX_EVENT_NUMBER]; wait的时候就传events   
};

#endif //EPOLLER_H
```

### epoller.cpp

```c++
#include "epoller.h"

Epoller::Epoller(int maxEvent):epollFd_(epoll_create(512)), events_(maxEvent){
    assert(epollFd_ >= 0 && events_.size() > 0); //epollFd_ epoll的实例  vector存放注册的监听事件
}

Epoller::~Epoller() {
    close(epollFd_);
}

bool Epoller::AddFd(int fd, uint32_t events) {
    if(fd < 0) return false;
    epoll_event ev = {0};
    ev.data.fd = fd;
    ev.events = events;
    return 0 == epoll_ctl(epollFd_, EPOLL_CTL_ADD, fd, &ev);
}

bool Epoller::ModFd(int fd, uint32_t events) {
    if(fd < 0) return false;
    epoll_event ev = {0};
    ev.data.fd = fd;
    ev.events = events;
    return 0 == epoll_ctl(epollFd_, EPOLL_CTL_MOD, fd, &ev);
}

bool Epoller::DelFd(int fd) {
    if(fd < 0) return false;
    return 0 == epoll_ctl(epollFd_, EPOLL_CTL_DEL, fd, 0);
}

// 返回事件数量
int Epoller::Wait(int timeoutMs) {
    return epoll_wait(epollFd_, &events_[0], static_cast<int>(events_.size()), timeoutMs);
}//使用events_将发生的epoll_event存入events_中

// 获取事件的fd
int Epoller::GetEventFd(size_t i) const {
    assert(i < events_.size() && i >= 0);
    return events_[i].data.fd;
}

// 获取事件属性
uint32_t Epoller::GetEvents(size_t i) const {
    assert(i < events_.size() && i >= 0);
    return events_[i].events;
}
```

###main.cpp

```cpp
#include "epoller.h"
#include <iostream>
#include <cstring>
#include <thread>
#include <chrono>

void simulate_read_event(int pipe_fd) {
    // 写入 "hello" 到管道
    const char* message = "hello";
    write(pipe_fd, message, strlen(message));
}

int main() {
    int pipe_fds[2];
    if (pipe(pipe_fds) == -1) {
        perror("pipe");
        return 1;
    }

    Epoller epoller(10);

    if (!epoller.AddFd(pipe_fds[0], EPOLLIN)) {
        std::cerr << "Failed to add fd to epoller" << std::endl;
        return 1;
    }

    std::thread waiter([&epoller]() {  // 移除 &pipe_fds，因为不需要
        while (true) {
            int n = epoller.Wait(1000);
            if (n > 0) {
                for (int i = 0; i < n; ++i) {
                    int fd = epoller.GetEventFd(i);
                    uint32_t evt = epoller.GetEvents(i);

                    // 打印触发的事件类型
                    std::cout << "触发的事件类型: ";
                    if (evt & EPOLLIN) std::cout << "EPOLLIN ";
                    if (evt & EPOLLOUT) std::cout << "EPOLLOUT ";
                    if (evt & EPOLLERR) std::cout << "EPOLLERR ";
                    if (evt & EPOLLHUP) std::cout << "EPOLLHUP ";
                    std::cout << std::endl;

                    // 读取并打印数据
                    char buffer[128] = {0};
                    int bytes_read = read(fd, buffer, sizeof(buffer) - 1);
                    if (bytes_read > 0) {
                        std::cout << "收到数据: " << buffer << std::endl;
                    }
                    return;
                }
            }
        }
    });

    std::thread simulator([&pipe_fds]() {
        std::this_thread::sleep_for(std::chrono::milliseconds(500));
        simulate_read_event(pipe_fds[1]);
    });

    simulator.join();
    waiter.join();

    close(pipe_fds[0]);
    close(pipe_fds[1]);

    return 0;
}
```

这里是使用的管道来使用的epoll

当管道的写端往内核缓冲区64KB写入数据时，读端触发EPOLLIN信号，然后打印数据

## 边缘触发和水平触发

水平触发：LT 默认 只要有数据(没有被处理完)就一直触发

边缘触发ET。满足条件的时候触发(只触发一次)   **必须一次性读取完所有数据，需要使用循环读取保证取完，必须设置非阻塞模式，性能高，适合高并发**

> 为什么ET需要设置非阻塞模式？
>
> 如果不设置非阻塞模式，会在循环读取数据的时候不会设置errno == EAGAIN || errno == EWOULDBLOCK，read的时候会一直阻塞等待新数据

如何使用ET

```c++
#include "epoller.h"
#include <iostream>
#include <cstring>
#include <thread>
#include <chrono>
#include <fcntl.h>

void SetNonBlocking(int fd) {
    int flags = fcntl(fd, F_GETFL);
    flags |= O_NONBLOCK;
    fcntl(fd, F_SETFL, flags);
}

void simulate_read_event(int pipe_fd) {
    const char* message = "hello";
    write(pipe_fd, message, strlen(message));
}

int main() {
    int pipe_fds[2];
    if (pipe(pipe_fds) == -1) {
        perror("pipe");
        return 1;
    }

    // 设置非阻塞模式
    SetNonBlocking(pipe_fds[0]);

    Epoller epoller(10);

    // 使用ET模式
    if (!epoller.AddFd(pipe_fds[0], EPOLLIN | EPOLLET)) {
        std::cerr << "Failed to add fd to epoller" << std::endl;
        return 1;
    }

    std::thread waiter([&epoller]() {
        while (true) {
            int n = epoller.Wait(1000);
            if (n > 0) {
                for (int i = 0; i < n; ++i) {
                    int fd = epoller.GetEventFd(i);
                    uint32_t evt = epoller.GetEvents(i);

                    std::cout << "触发的事件类型: ";
                    if (evt & EPOLLIN) std::cout << "EPOLLIN ";
                    if (evt & EPOLLOUT) std::cout << "EPOLLOUT ";
                    if (evt & EPOLLERR) std::cout << "EPOLLERR ";
                    if (evt & EPOLLHUP) std::cout << "EPOLLHUP ";
                    std::cout << std::endl;

                    // ET模式下的读取循环
                    while (true) {
                        char buffer[128] = {0};
                        int bytes_read = read(fd, buffer, sizeof(buffer) - 1);
                        if (bytes_read < 0) {
                            if (errno == EAGAIN || errno == EWOULDBLOCK) {
                                // 数据读取完毕
                                break;
                            }
                            // 处理其他错误
                            perror("read error");
                            break;
                        } else if (bytes_read == 0) {
                            // 连接关闭
                            std::cout << "连接关闭" << std::endl;
                            break;
                        }
                        std::cout << "收到数据: " << buffer << std::endl;
                    }
                    return;  // 完成一次读取后退出
                }
            }
        }
    });

    std::thread simulator([&pipe_fds]() {
        std::this_thread::sleep_for(std::chrono::milliseconds(500));
        simulate_read_event(pipe_fds[1]);
    });

    simulator.join();
    waiter.join();

    close(pipe_fds[0]);
    close(pipe_fds[1]);

    return 0;
}
```

## tcp

监听套接字注册 EPOLLIN | EPOLLET

连接套接字注册 EPOLLIN | EPOLLET *根据需要可以加上EPOLLOUT*

## 关闭和错误

==EPOLLIN==：

对端关闭时会触发

read() 返回 0 表示对端关闭

==EPOLLRDHUP==：

对端关闭连接时触发

需要在注册时指定此事件

==EPOLLHUP==：

表示管道写端被关闭

不需要特别注册，会自动触发

==EPOLLERR==：

发生错误时触发

不需要特别注册，会自动触发

## select、poll 和 epoll 区别

```
                select          poll            epoll
-----------------------------------------------------------------
操作方式        遍历            遍历            回调
底层实现        数组            链表            红黑树
最大连接数      1024           无限制          无限制
fd拷贝          每次都需要      每次都需要      只需一次
返回就绪fd      遍历全部fd      遍历全部fd      只有就绪fd
遍历复杂度      O(n)			  O(n)           O(1)
设置ET		 否			   否			是
平台            多平台          多平台          Linux
```

fd 拷贝指的是文件描述符从用户空间拷贝到内核空间的过程

select需要每次都需要重新注册事件

性能对比：epoll>poll>select

==epoll的优势==：

- **内核态操作，不适合短期活跃连接** 对于select和poll来说，所有文件描述符都是在用户态被加入其文件描述符集合的，每次调用都需要将整个集合拷贝到内核态；epoll则将整个文件描述符集合维护在内核态，每次添加文件描述符的时候都需要执行一个系统调用。**系统调用的开销是很大的**，而且在有很多短期活跃连接的情况下，epoll可能会慢于select和poll由于这些大量的系统调用开销。
- **底层红黑树，并且维护一个ready lisst 读取更快**elect使用线性表描述文件描述符集合，文件描述符有上限；poll使用链表来描述；
- **fd就绪时通过回调通知，更快** select和poll的最大开销来自内核判断是否有文件描述符就绪这一过程：每次执行select或poll调用时，它们会采用遍历的方式，遍历整个文件描述符集合去判断各个文件描述符是否有活动；epoll触发epoll回调函数，放到ready list中等待epoll_wait调用后被处理。
- **epoll同时支持LT和ET模式** select和poll都只能工作在相对低效的LT模式下。

总结：当监测的fd数量较小，且各个fd都很活跃的情况下，建议使用select和poll；当监听的fd数量较多，且单位时间仅部分fd活跃的情况下，使用epoll会明显提升性能。

