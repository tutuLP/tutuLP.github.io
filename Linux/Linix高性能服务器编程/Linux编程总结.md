

# 服务器框架

## 服务器模型

C/S模型 

> B/S模型 浏览器服务器模型

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20241201133052276.png" alt="image-20241201133052276" style="zoom:33%;" />

P2P模型

每个节点都既是服务器又是客户端，可以直接与其他节点通信。

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20241201131859404.png" alt="image-20241201131859404" style="zoom:33%;" />

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20241201132901998.png" alt="image-20241201132901998" style="zoom:33%;" />

### 服务器编程框架

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240903103947062.png" alt="image-20240903103947062" style="zoom: 33%;" />

| 模块         | 单个服务器                   | 服务器集群                   |
| ------------ | ---------------------------- | ---------------------------- |
| I/O处理单元  | 处理客户端连接，读写网络数据 | 作为接入服务器，实现负载均衡 |
| 逻辑单元     | 业务进程，线程               | 逻辑服务器                   |
| 网络存储单元 | 本地数据库，文件，缓存       | 数据库服务器                 |
| 请求队列     | 各单元之间通信方式           | 各服务器之间永久tcp连接      |

####I/O处理单元

#####I/O模型

在I/O模型中，“同步”和“异步”区分的是内核向应用程序通知的是何种I/O事件（是就绪事件还是完成事件），以及该由谁来完成I/O读写（是应用程序还是内核）。

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240903105336713.png" alt="image-20240903105336713" style="zoom:50%;" />

##### 事件处理模式

###### Reactor模式

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20241201134224699.png" alt="image-20241201134224699" style="zoom: 50%;" />

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20241201134305203.png" alt="image-20241201134305203" style="zoom:33%;" />

Reactor（事件分发器）：1. 管理EventHandler的注册与销毁 2. 分发事件 (根据Demultiplexer返回的注册的事件分发) 

Demultiplexer（多路复用器 ： 1. 底层监听机制（如select、poll、epoll）

EventHandler（事件处理器）：1. 实际处理各类事件的组件

事件源：1. 可以是Socket、Timer、Pipe等   产生各类I/O事件

###### Proactor模式

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20241201135651226.png" alt="image-20241201135651226" style="zoom:33%;" />

Proactor Initiator: 发起异步操作的初始化器

异步操作处理器: 负责提交异步操作到**操作系统**

操作系统执行实际的IO操作

IO操作完成后通知完成端口

Completion Dispatcher接收完成通知

分发给对应的Completion Handler

Handler处理完成事件并返回结果给应用程序

###### 模拟Proactor模式

为了可以在不支持异步I/O的系统上模拟Proactor模式，这样就可以实现异步的接口了

使用同步I/O方式模拟出Proactor模式的一种方法：主线程执行数据读写操作，读写完成之后，主线程向工作线程通知这一“完成事件”。那么从工作线程的角度来看，它们就直接获得了数据读写的结果，接下来要做的只是对读写的结果进行逻辑处理。

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20241201181852869.png" alt="image-20241201181852869" style="zoom:33%;" />

CompletionHandler: 完成事件处理接口

ThreadPool: 执行同步I/O的线程池

ProactorInitiator: 发起异步操作的接口

###### Reactor和Proactor的区别

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20241201140053653.png" alt="image-20241201140053653" style="zoom: 50%;" />

![image-20241201140104227](http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20241201140104227.png)

I/O操作处理方式

Reactor: 应用程序负责进行实际的I/O操作

Proactor: 操作系统负责进行实际的I/O操作

事件通知内容

Reactor: 通知的是I/O就绪事件（可读/可写）

Proactor: 通知的是I/O完成事件

编程复杂度

Reactor: 相对简单，符合同步编程思维

Proactor: 较复杂，需要操作系统支持异步I/O

适用平台

Reactor: 在Linux下应用广泛（如epoll）

Proactor: 在Windows下应用广泛（如IOCP）

性能特点

Reactor: 需要应用程序自己执行I/O，会占用应用程序线程

Proactor: I/O由操作系统执行，应用程序线程可以处理其他任务

实现示例

Reactor: Linux的epoll、select、poll

Proactor: Windows的IOCP、Linux的AIO

资源消耗

Reactor: 较少的系统资源

Proactor: 需要更多的系统资源来支持异步操作

总的来说，Reactor是同步非阻塞I/O模型，而Proactor是异步I/O模型。

##### 并发模式

在并发模式中，“同步”指的是程序完全按照代码序列的顺序执行；“异步”指的是程序的执行需要由系统事件来驱动。常见的系统事件包括中断、信号等。

如果程序是IO密集，而不是计算密集则需要并发  多进程和多线程两种方式

######半同步/半异步模式

同步线程用于处理客户逻辑；异步线程用于处理I/O事件

异步线程监听到客户请求后，就将其封装成请求对象并插入请求队列中。请求队列将通知某个工作在同步模式的工作线程来读取并处理该请求对象



<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20241201184705755.png" alt="image-20241201184705755" style="zoom:33%;" />

* 同步服务层

负责接收客户端请求  以同步的方式工作  通常是多线程实现

* 排队层

作为同步层和异步层之间的缓冲  通常使用队列实现  解耦同步和异步操作

* 异步服务层

从队列中获取任务  管理异步IO操作  协调异步处理流程

* 异步IO线程池

执行实际的异步IO操作  提高系统并发处理能力  避免阻塞主线程

###### 半同步/半反应堆

在服务器程序中，如果结合考虑两种事件处理模式和几种I/O模型，则半同步/半异步模式就存在多种变体。其中有一种变体称为半同步/半反应堆

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240903111921950.png" alt="image-20240903111921950" style="zoom:50%;" />

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20241201194214080.png" alt="image-20241201194214080" style="zoom: 50%;" />

异步线程只有一个，由主线程来充当。它负责监听所有socket上的事件。如果监听socket上有可读事件发生，即有新的连接请求到来，主线程就接受之以得到新的连接socket，然后往epoll内核事件表中注册该socket上的读写事件。如果连接socket上有读写事件发生，即有新的客户请求到来或有数据要发送至客户端，主线程就将该连接socket插入请求队列中。所有工作线程都睡眠在请求队列上，当有任务到来时，它们将通过竞争（比如申请互斥锁）获得任务的接管权

主线程插入请求队列中的任务是就绪的连接socket。这说明该图所示的半同步/半反应堆模式采用的事件处理模式是Reactor模式：它要求工作线程自己从socket上读取客户请求和往socket写入服务器应答。这就是该模式的名称中“half-reactive”的含义。实际上，半同步/半反应堆模式也可以使用模拟的Proactor事件处理模式，即由主线程来完成数据的读写。在这种情况下，主线程一般会将应用程序数据、任务类型等信息封装为一个任务对象，然后将其（或者指向该任务对象的一个指针）插入请求队列。工作线程从请求队列中取得任务对象之后，即可直接处理之，而无须执行读写操作了。

缺点：

* 主线程和工作线程共享请求队列。主线程往请求队列中添加任务，或者工作线程从请求队列中取出任务，都需要对请求队列加锁保护，从而白白耗费CPU时间。

* 每个工作线程在同一时间只能处理一个客户请求。如果客户数量较多，而工作线程较少，则请求队列中将堆积很多任务对象，客户端的响应速度将越来越慢。如果通过增加工作线程来解决这一问题，则工作线程的切换也将耗费大量CPU时间。

一种高效的半同步/半异步模式 每个工作线程都能同时处理多个客户连接

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240903112357921.png" alt="image-20240903112357921" style="zoom:50%;" />

主线程只管理监听socket，连接socket由工作线程来管理。当有新的连接到来时，主线程就接受之并将新返回的连接socket派发给某个工作线程，此后该新socket上的任何I/O操作都由被选中的工作线程来处理，直到客户关闭连接。主线程向工作线程派发socket的最简单的方式，是**往它和工作线程之间的管道里写数据**。工作线程检测到管道上有数据可读时，就分析是否是一个新的客户连接请求到来。如果是，则把该新socket上的读写事件注册到自己的epoll内核事件表中。

###### 领导者/追随者模式

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20241201215046863.png" alt="image-20241201215046863" style="zoom:50%;" />

Leader线程 负责等待和接收新的事件，当事件到达时开始处理，在处理之前提升一个新的Leader，处理完成后重新加入线程池

# socket

~~~c++
#include<netinet/in.h>//转换大端小端
#include <stdio.h> 
#include <iostream>
#include <bits/socket.h>
#include <arpa/inet.h>//ip地址转换
#include <sys/types.h>
#include <sys/socket.h>
#include <assert.h>
using namespace std;
int main() {
    //源ip port
    const char* ip="192.168.6.208";
    unsigned short int port=5005;
    //专用socket结构体
    sockaddr_in addr;
    addr.sin_family=AF_INET;//绑定地址族
    addr.sin_port = htons(port);//绑定端口 网络字节序
    if(inet_aton(ip,&addr.sin_addr)==1){//绑定ip 网络字节序   inet_aton？
        cout<<addr.sin_addr.s_addr<<endl;
    }
    //专用结构体转化为通用结构体
    //const struct sockaddr*new_addr=(sockaddr*)&addr;
    //创建socket
    int pre_name_socket;
    pre_name_socket=socket(PF_INET,SOCK_STREAM,0);//ipv4 tcp SOCK_NONBLOCK  输出3 
    //命名socket
    int ret=bind(pre_name_socket,(sockaddr*)&addr,sizeof(addr));
    assert(ret != -1);
    cout<<"bind success"<<endl;
    //监听
    ret = listen(pre_name_socket, 5);//监听
    return 0;
}
~~~





ET 边缘触发 状态改变才会 epoll会通知一次

LT 水平触发 符合状态就会(持续)通知
