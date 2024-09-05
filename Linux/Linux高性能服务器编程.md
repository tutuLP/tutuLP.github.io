# 深入解析高性能服务器编程

## Linux网络编程基础API

* socket地址API。socket最开始的含义是一个IP地址和端口对（ip，port）。它唯一地表示了使用TCP通信的一端。本书称其为socket地址。

* socket基础API。socket的主要API都定义在sys/socket.h头文件中，包括创建socket、命名socket、监听socket、接受连接、发起连接、读写数据、获取地址信息、检测带外标记，以及读取和设置socket选项。

* 网络信息API。Linux提供了一套网络信息API，以实现主机名和IP地址之间的转换，以及服务名称和端口号之间的转换。这些API都定义在netdb.h头文件中，我们将讨论其中几个主要的函数。

###socket地址API

先要理解主机字节序和网络字节序

#### 主机字节序和网络字节序

现代CPU的累加器一次都能装载（至少）4字节（这里考虑32位机），即一个整数。那么这4字节在内存中排列的顺序将影响它被累加器装载成的整数的值。这就是字节序问题。字节序分为大端字节序（big endian）和小端字节序（little endian）。

大端字节序是指一个整数的高位字节（23～31 bit）存储在内存的低地址处，低位字节（0～7 bit）存储在内存的高地址处。小端字节序则是指整数的高位字节存储在内存的高地址处，而低位字节则存储在内存的低地址处。

判断机器字节序

~~~c
#include<stdio.h>
void byteorder(){
    union{
		short value;  //short占两字节  
		char union_bytes[sizeof(short)];
	}test;
	test.value=0x0102;  //01一个字节 02一个字节 
    //union的成员共享一个内存，虽然是给short value赋值的，两个成员的内存都是一样的，只是用short类型解释是十六进制的0102，用char数组解释时，恰好是两个成员，每个1字节，都是十进制的第一个是01第二个是02
	if((test.union_bytes[0]==1)&&(test.union_bytes[1]==2)){
		printf("big endian\n");
	}
	else if((test.union_bytes[0]==2)&&(test.union_bytes[1]==1))
	{
	printf("little endian\n");
	}
	else{
	printf("unknown...\n");
	}
}
int main(){
	byteorder();
	return 0;
}
~~~

我在linux上运行是little endian

我们可以调试查看内存的表示，也是符合逻辑的

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240902200404176.png" alt="image-20240902200404176" style="zoom:50%;" />

* ==小端字节序又被称为主机字节序== 现代PC大多采用小端字节序
* ==大端字节序也称为网络字节序== **发送端总是把要发送的数据转化成大端字节序数据后再发送**，而接收端知道对方传送过来的数据总是采用大端字节序，所以接收端可以根据自身采用的字节序决定是否对接收到的数据进行转换（小端机转换，大端机不转换）

*即使是同一台机器上的两个进程（比如一个由C语言编写，另一个由JAVA编写）通信，也要考虑字节序的问题（JAVA虚拟机采用大端字节序）。*

Linux提供了如下4个函数来完成主机字节序和网络字节序之间的转换

~~~c
#include＜netinet/in.h＞
unsigned long int htonl(unsigned long int hostlong);  //无符号长整型 主机->网络
unsigned short int htons(unsigned short int hostshort);//无符号短整型
unsigned long int ntohl(unsigned long int netlong);  //网络->主机
unsigned short int ntohs(unsigned short int netshort);
~~~

长整型（32 bit）常用来转换IP地址，短整型(16位)用来转换端口号

这里的类型都是可以换的，比如我转换一个short类型为大端序

~~~c++
#include<netinet/in.h>
#include <stdio.h> 
#include <iostream>
using namespace std;
int main() {
    unsigned short int host_short = 0x1234;
    unsigned short int net_short = htons(host_short); 
    printf("Network byte order short: 0x%X\n", net_short);  //3412
    cout << net_short << endl; //13330的十六进制3421
	return 0;
}
~~~

#### 通用socket地址

socket网络编程接口中表示socket地址的是结构体sockaddr，其定义如下

~~~c
#include＜bits/socket.h＞
struct sockaddr{
	sa_family_t sa_family; //地址族类型
	char sa_data[14];
}
~~~

* 地址族类型通常与协议族类型对应  常见的协议族（protocol family，也称domain)

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240831112454497.png" alt="image-20240831112454497" style="zoom:50%;" />

宏PF\_\*和AF\_\*都定义在bits/socket.h头文件中，且后者与前者有完全相同的值，所以二者通常混用。

* sa_data成员用于存放socket地址值。但是，不同的协议族的地址值具有不同的含义和长度

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240831112643843.png" alt="image-20240831112643843" style="zoom: 50%;" />

可见14的大小完全不够，linux定义了新的结构

~~~c
#include＜bits/socket.h＞
struct sockaddr_storage{
	sa_family_t sa_family; 
	unsigned long int__ss_align;
	char__ss_padding[128-sizeof(__ss_align)];
}
~~~

__ss_align成员使内存对齐

#### 专用socket地址

上面这两个通用socket地址结构体显然很不好用，比如设置与获取IP地址和端口号就需要执行烦琐的位操作。所以Linux为各个协议族提供了专门的socket地址结构体

* UNIX本地域协议族

~~~c
#include＜sys/un.h＞
struct sockaddr_un{
	sa_family_t sin_family;/*地址族：AF_UNIX*/
	char sun_path[108];/*文件路径名*/
};
~~~

* TCP/IP协议族有sockaddr_in和sockaddr_in6两个专用socket地址结构体，它们分别用于IPv4和IPv6：

~~~c
struct sockaddr_in{
	sa_family_t sin_family;/*地址族：AF_INET*/
	u_int16_t sin_port;/*端口号，要用网络字节序表示*/
	struct in_addr sin_addr;/*IPv4地址结构体，见下面*/
};
struct in_addr{
	u_int32_t s_addr;/*IPv4地址，要用网络字节序表示*/
};

struct sockaddr_in6{
	sa_family_t sin6_family;/*地址族：AF_INET6*/
	u_int16_t sin6_port;/*端口号，要用网络字节序表示*/
	u_int32_t sin6_flowinfo;/*流信息，应设置为0*/
	struct in6_addr sin6_addr;/*IPv6地址结构体，见下面*/
	u_int32_t sin6_scope_id;/*scope ID，尚处于实验阶段*/
};
struct in6_addr{
	unsigned char sa_addr[16];/*IPv6地址，要用网络字节序表示*/
};
~~~

所有专用socket地址（以及sockaddr_storage）类型的变量在实际使用时都需要转化为通用socket地址类型sockaddr（强制转换即可），因为所有socket编程接口使用的地址参数的类型都是sockaddr。

#### IP地址转换函数

通常，人们习惯用可读性好的字符串来表示IP地址，比如用点分十进制字符串表示IPv4地址，以及用十六进制字符串表示IPv6地址。

* 编程中我们需要先把它们转化为整数（二进制数）方能使用。
* 记录日志时，我们要把整数表示的IP地址转化为可读的字符串。

下面3个函数可用于用*点分十进制字符串表示的IPv4地址*和用*网络字节序整数表示的IPv4地址*之间的转换：

~~~c
#include＜arpa/inet.h＞
in_addr_t inet_addr(const char*strptr);
int inet_aton(const char*cp,struct in_addr*inp);
char*inet_ntoa(struct in_addr in);
~~~

* inet_addr IPv4地址转化为用网络字节序它  失败时返回INADDR_NONE。

* inet_aton  IPv4地址转化为用网络字节序，但是将转化结果存储于参数inp指向的地址结构中。它成功时返回1，失败则返回0。

* inet_ntoa  网络字节序转化为用点分十进制  但需要注意的是，该函数内部用一个静态变量存储转化结果，函数的返回值指向该静态内存，因此inet_ntoa是不可重入的。(意思是连续调用两个这个函数 后一个的结果会把前一个覆盖)

~~~c
#include＜arpa/inet.h＞
int inet_pton(int af,const char*src,void*dst);
const char*inet_ntop(int af,const void*src,char*dst,socklen_tcnt);
~~~

适用于ipv4和ipv6

* inet_pton  IP地址src转换成用网络字节序  并把转换结果存储于dst指向的内存中。其中，af参数指定地址族，可以是AF_INET或者AF_INET6。inet_pton成功时返回1，失败则返回0并设置errno[1]。

* inet_ntop函数进行相反的转换，前三个参数的含义与inet_pton的参数相同，最后一个参数cnt指定目标存储单元的大小。下面的两个宏能帮助我们指定这个大小（分别用于IPv4和IPv6）：    inet_ntop成功时返回目标存储单元的地址，失败则返回NULL并设置errno。

* * \#include＜netinet/in.h＞

    \#define INET_ADDRSTRLEN 16

    \#define INET6_ADDRSTRLEN 46

**使用示例**

~~~c++
#include <stdio.h> 
#include <iostream>
#include <arpa/inet.h>//ip地址转换
using namespace std;
int main() {
    //函数1
    const char* ip="192.168.6.208";
    in_addr_t ipAddr = inet_addr(ip); 
    cout<<ipAddr<<endl;
    cout<<inet_addr("192.168.6.208")<<endl; //3490097344 返回in_addr_t(unsigned long)
	
    //函数2
    //使用最多
    //获取端口号的网络字节
    unsigned short int port=5005;
    unsigned short int new_port=htons(port);
    //建立ipv4专用socket地址
    struct sockaddr_in ip_addr;//专用socket结构体
    ip_addr.sin_family=AF_INET;//指定地址族ipv4
    ip_addr.sin_port=new_port;//指定网络端口号
    if(inet_aton(ip,&ip_addr.sin_addr)==1){ //ip转化为网络地址
        cout<<ip_addr.sin_addr.s_addr<<endl;
    }
    cout<<ip_addr.sin_port<<endl;//网络端口36115
	//专用地址强制转化为通用地址
    const struct sockaddr*addr=(sockaddr*)&ip_addr;
    
    //函数3
    char* ip2=inet_ntoa(ip_addr.sin_addr);
    cout<<ip2<<endl;
}
~~~

### 创建socket

UNIX/Linux的一个哲学是：所有东西都是文件。socket也不例外，它就是可读、可写、可控制、可关闭的文件描述符。下面的socket系统调用可创建一个socket：

~~~c
#include＜sys/types.h＞
#include＜sys/socket.h＞
int socket(int domain,int type,int protocol);
~~~

* domain参数告诉系统使用哪个底层协议族。PF_INET（Protocol Family of Internet，用于IPv4）或PF_INET6（用于IPv6）；对于UNIX本地域协议族而言，该参数应该设置为PF_UNIX。

* type参数指定服务类型。服务类型主要有SOCK_STREAM服务（流服务）和SOCK_UGRAM（数据报）服务。对TCP/IP协议族而言，其值取SOCK_STREAM表示传输层使用TCP协议，取SOCK_DGRAM表示传输层使用UDP协议。

* * 值得指出的是，自Linux内核版本2.6.17起，type参数可以接受上述服务类型与下面两个重要的标志相与的值：SOCK_NONBLOCK和SOCK_CLOEXEC。它们分别表示将新创建的socket设为非阻塞的，以及用fork调用创建子进程时在子进程中关闭该socket。在内核版本2.6.17之前的Linux中，文件描述符的这两个属性都需要使用额外的系统调用（比如fcntl）来设置。

* protocol参数是在前两个参数构成的协议集合下，再选择一个具体的协议。不过这个值通常都是唯一的（前两个参数已经完全决定了它的值）。几乎在所有情况下，我们都应该把它设置为0，表示使用默认协议。

socket系统调用成功时返回一个socket文件描述符，失败则返回-1并设置errno

### 命名socket

将一个socket与socket地址绑定称为给socket命名。在服务器程序中，我们通常要命名socket，因为只有命名后客户端才能知道该如何连接它。客户端则通常不需要命名socket，而是采用匿名方式，即使用操作系统自动分配的socket地址。命名socket的系统调用是bind，其定义如下：

~~~c
#include＜sys/types.h＞
#include＜sys/socket.h＞
int bind(int sockfd,const struct sockaddr*my_addr,socklen_t addrlen);
~~~

* bind将my_addr所指的socket地址分配给未命名的sockfd文件描述符，addrlen参数指出该socket地址的长度。bind成功时返回0，失败则返回-1并设置errno。其中两种常见的errno是EACCES和EADDRINUSE，它们的含义分别是：

* * EACCES，被绑定的地址是受保护的地址，仅超级用户能够访问。比如普通用户将socket绑定到知名服务端口（端口号为0～1023）上时，bind将返回EACCES错误
  * EADDRINUSE，被绑定的地址正在使用中。比如将socket绑定到一个处于TIME_WAIT状态的socket地址。

### 监听socket

socket被命名之后，还不能马上接受客户连接，我们需要使用如下系统调用来创建一个监听队列以存放待处理的客户连接：

~~~c
#include＜sys/socket.h＞
int listen(int sockfd,int backlog);
~~~

* sockfd参数指定被监听的socket。
* backlog参数提示内核监听队列的最大长度。监听队列的长度如果超过backlog，服务器将不受理新的客户连接，客户端也将收到ECONNREFUSED错误信息。在内核版本2.2之前的Linux中，backlog参数是指所有处于半连接状态（SYN_RCVD）和完全连接状态（ESTABLISHED）的socket的上限。但自内核版本2.2之后，它只表示处于完全连接状态的socket的上限，处于半连接状态的socket的上限则由/proc/sys/net/ipv4/tcp_max_syn_backlog内核参数定义。backlog参数的典型值是5。

listen成功时返回0，失败则返回-1并设置errno

**研究backlog参数对listen系统调用的实际影响**

~~~c++
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <signal.h>
#include <unistd.h>
#include <stdlib.h>
#include <assert.h>//assert宏
#include <stdio.h>
#include <string.h>//bzero
//#include <libgen.h>
static bool stop = false;
/*SIGTERM信号的处理函数，触发时结束主程序中的循环*/
static void handle_term(int sig)
{
    stop = true;
}
int main(int argc, char *argv[])
{
    signal(SIGTERM, handle_term);//设置SIGTERM信号的处理函数为handle_term    SIGTERM是一个常用的终止程序运行的信号
    if (argc <= 3)//检查命令行参数是否满足要求
    {
        printf("usage:%s ip_address port_numberbacklog\n", basename(argv[0]));
        return 1;
    }
    const char *ip = argv[1];
    int port = atoi(argv[2]);
    int backlog = atoi(argv[3]);
    int sock = socket(PF_INET, SOCK_STREAM, 0);//创建socket
    assert(sock >= 0);   //使用 assert 宏来验证 sock 的值是否大于或等于0。 不满足则退出程序
    /*创建一个IPv4 socket地址*/
    struct sockaddr_in address;
    bzero(&address, sizeof(address));//bzero 将内存块清空 现在更多使用memset(&address, 0, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &address.sin_addr);//绑定ip 网络字节序
    address.sin_port = htons(port);//绑定ip
    int ret = bind(sock, (struct sockaddr *)&address, sizeof(address));//命名
    assert(ret != -1);
    ret = listen(sock, backlog);//监听
    assert(ret != -1);
    /*循环等待连接，直到有SIGTERM信号将它中断*/
    while (!stop)
    {
        sleep(1);
    }
    /*关闭socket，见后文*/
    close(sock);
    return 0;
}
~~~

我写的：

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
    if(inet_aton(ip,&addr.sin_addr)==1){//绑定ip 网络字节序
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

该服务器程序（名为testlisten）接收3个参数：IP地址、端口号和backlog值。我们在Kongming20上运行该服务器程序，并在ernestlaptop上多次执行telnet命令来连接该服务器程序。同时，每使用telnet命令建立一个连接，就执行一次netstat命令来查看服务器上连接的状态。具体操作过程如下：

~~~
$./testlisten 192.168.1.109 12345 5 #运行代码 监听12345端口，给backlog传递典型值5
$telnet 192.168.1.109 12345#多次执行之
$netstat-nt|grep 12345#多次执行之
~~~

是netstat命令某次输出的内容，它显示了这一时刻listen监听队列的内容。

~~~
Proto Recv-Q Send-Q Local Address Foreign Address Statetcp
tcp 0 0 192.168.1.109:12345 192.168.1.108:2240 SYN_RECV
tcp 0 0 192.168.1.109:12345 192.168.1.108:2228 SYN_RECV[1]
tcp 0 0 192.168.1.109:12345 192.168.1.108:2230 SYN_RECV
tcp 0 0 192.168.1.109:12345 192.168.1.108:2238 SYN_RECV
tcp 0 0 192.168.1.109:12345 192.168.1.108:2236 SYN_RECV
tcp 0 0 192.168.1.109:12345 192.168.1.108:2217 ESTABLISHED
tcp 0 0 192.168.1.109:12345 192.168.1.108:2226 ESTABLISHED
tcp 0 0 192.168.1.109:12345 192.168.1.108:2224 ESTABLISHED
tcp 0 0 192.168.1.109:12345 192.168.1.108:2212 ESTABLISHED
tcp 0 0 192.168.1.109:12345 192.168.1.108:2220 ESTABLISHED
tcp 0 0 192.168.1.109:12345 192.168.1.108:2222 ESTABLISHED
~~~

可见，在监听队列中，处于ESTABLISHED状态的连接只有6个（backlog值加1），其他的连接都处于SYN_RCVD状态(TCP状态转移)。我们改变服务器程序的第3个参数并重新运行之，能发现同样的规律，即完整连接最多有（backlog+1）个。在不同的系统上，运行结果会有些差别，不过监听队列中完整连接的上限通常比backlog值略大。

**所以服务器可以 接受多个连接，能建立连接的个数取决于backlog值，不能连接的就处于监听状态**？？？

### 接受连接

~~~c
#include＜sys/types.h＞
#include＜sys/socket.h＞
int accept(int sockfd,struct sockaddr*addr,socklen_t*addrlen);
~~~

* sockfd参数是执行过listen系统调用的监听socket。
* addr参数用来获取被接受连接的远端socket地址，该socket地址的长度由addrlen参数指出。
* accept成功时返回一个新的连接socket，该socket唯一地标识了被接受的这个连接，服务器可通过读写该socket来与被接受连接对应的客户端通信。
* accept失败时返回-1并设置errno。

现在考虑如下情况：如果监听队列中处于ESTABLISHED状态的连接对应的客户端出现网络异常（比如掉线），或者提前退出，那么服务器对这个连接执行的accept调用是否成功？我们编写一个简单的服务器程序来测试之

~~~c++
#include <sys/socket.h>
#include <netinet/in.h>//用于inet_ntop函数使用宏
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
int main(int argc, char *argv[])
{
    if (argc <= 2)
    {
        printf("usage:%s ip_address port_number\n", basename(argv[0]));
        return 1;
    }
    const char *ip = argv[1];
    int port = atoi(argv[2]);
    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);
    int sock = socket(PF_INET, SOCK_STREAM, 0);
    assert(sock >= 0);
    int ret = bind(sock, (struct sockaddr *)&address, sizeof(address));
    assert(ret != -1);
    ret = listen(sock, 5);
    assert(ret != -1);
    
    
    /*暂停20秒以等待客户端连接和相关操作（掉线或者退出）完成*/
    sleep(20);
    //建立新的地址
    struct sockaddr_in client;
    socklen_t client_addrlength = sizeof(client);
    //接受连接
    int connfd = accept(sock, (struct sockaddr *)&client, &client_addrlength);
    if (connfd < 0)
    {
        printf("errno is:%d\n", errno);
    }
    else
    {
        /*接受连接成功则打印出客户端的IP地址和端口号*/
        char remote[INET_ADDRSTRLEN];//宏 用于inet_ntop接收大小
        printf("connected with ip:%s and port:%d\n", inet_ntop(AF_INET, &client.sin_addr, remote, INET_ADDRSTRLEN), ntohs(client.sin_port));
        close(connfd);//关闭这个连接socket
    }
    close(sock);//关闭服务器监听socket
    return 0;
}
~~~

$./testaccept 192.168.1.109 54321#监听54321端口

$telnet 192.168.1.109 54321 另一个机器上运行   启动telnet客户端程序后，立即断开该客户端的网络连接（建立和断开连接的过程要在服务器启动后20秒内完成）。结果发现accept调用能够正常返回，服务器输出如下：

connected with ip:192.168.1.108 and port:38545

在服务器上运行netstat命令以查看accept返回的连接socket的状态

$netstat-nt|grep 54321

tcp 0 0 192.168.1.109:54321 192.168.1.108:38545 ESTABLISHED

netstat命令的输出说明，accept调用对于客户端网络断开毫不知情。

下面我们重新执行上述过程，不过这次不断开客户端网络连接，而是在建立连接后立即退出客户端程序。这次accept调用同样正常返回，在服务器上运行netstat命令以查看accept返回的连接socket的状态也正常



由此可见，accept只是从监听队列中取出连接，而不论连接处于何种状态（如上面的ESTABLISHED状态和CLOSE_WAIT状态），更不关心任何网络状况的变化

我们把执行过listen调用、处于LISTEN状态的socket称为监听socket，而所有处于ESTABLISHED状态的socket则称为连接socket。

### 发起连接

如果说服务器通过listen调用来被动接受连接，那么客户端需要通过如下系统调用来主动与服务器建立连接

~~~c
#include＜sys/types.h＞
#include＜sys/socket.h＞
int connect(int sockfd,const struct sockaddr*serv_addr,socklen_t addrlen);
~~~

* sockfd参数由socket系统调用返回一个socket。serv_addr参数是服务器监听的socket地址，addrlen参数则指定这个地址的长度

* connect成功时返回0。一旦成功建立连接，sockfd就唯一地标识了这个连接，客户端就可以通过读写sockfd来与服务器通信。connect失败则返回-1并设置errno。其中两种常见的errno是ECONNREFUSED和ETIMEDOUT，它们的含义如下：
* * ECONNREFUSED，目标端口不存在，连接被拒绝
  * ETIMEDOUT，连接超时

### 关闭连接

关闭一个连接实际上就是关闭该连接对应的socket，这可以通过如下关闭普通文件描述符的系统调用来完成

~~~c
#include＜unistd.h＞
int close(int fd);
~~~

fd参数是待关闭的socket。不过，close系统调用并非总是立即关闭一个连接，而是将fd的引用计数减1。只有当fd的引用计数为0时，才真正关闭连接。多进程程序中，一次fork系统调用默认将使父进程中打开的socket的引用计数加1，因此我们必须在父进程和子进程中都对该socket执行close调用才能将连接关闭。

如果无论如何都要立即终止连接（而不是将socket的引用计数减1），可以使用如下的shutdown系统调用（相对于close来说，它是专门为网络编程设计的）

~~~c
#include＜sys/socket.h＞
int shutdown(int sockfd,int howto);
~~~

sockfd参数是待关闭的socket。howto参数决定了shutdown的行为，它可取表的某个值

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240831124846783.png" alt="image-20240831124846783" style="zoom:50%;" />

由此可见，shutdown能够分别关闭socket上的读或写，或者都关闭。而close在关闭连接时只能将socket上的读和写同时关闭。

shutdown成功时返回0，失败则返回-1并设置errno。

### 数据读写

#### TCP数据读写

对文件的读写操作read和write同样适用于socket。但是socket编程接口提供了几个专门用于socket数据读写的系统调用，它们增加了对数据读写的控制。其中用于TCP流数据读写的系统调用是：

~~~c
#include＜sys/types.h＞
#include＜sys/socket.h＞
ssize_t recv(int sockfd,void*buf,size_t len,int flags);
ssize_t send(int sockfd,const void*buf,size_t len,int flags);
~~~

recv读取sockfd上的数据，buf和len参数分别指定读缓冲区的位置和大小，flags参数的含义见后文，通常设置为0即可。recv成功时返回实际读取到的数据的长度，它可能小于我们期望的长度len。因此我们可能要多次调用recv，才能读取到完整的数据。recv可能返回0，这意味着通信对方已经关闭连接了。recv出错时返回-1并设置errno。

send往sockfd上写入数据，buf和len参数分别指定写缓冲区的位置和大小。send成功时返回实际写入的数据的长度，失败则返回-1并设置errno

* flags参数为数据收发提供了额外的控制

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240831142715399.png" alt="image-20240831142715399" style="zoom:50%;" />

我们举例来说明如何使用这些选项。MSG_OOB选项给应用程序提供了发送和接收带外数据的方法

* 客户端

~~~c++
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
int main(int argc, char *argv[])
{
    if (argc <= 2)
    {
        printf("usage:%s ip_address port_number\n", basename(argv[0]));
        return 1;
    }
    const char *ip = argv[1];
    int port = atoi(argv[2]);
    //socket地址
    struct sockaddr_in server_address;
    bzero(&server_address, sizeof(server_address));
    server_address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &server_address.sin_addr);
    server_address.sin_port = htons(port);
    //创建socket
    int sockfd = socket(PF_INET, SOCK_STREAM, 0);
    assert(sockfd >= 0);
    //不用命名和监听直接连接-始终只有一个socket
    if (connect(sockfd, (struct sockaddr *)&server_address, sizeof(server_address)) < 0)
    {
        printf("connection failed\n");
    }
    else
    {
        const char *oob_data = "abc";
        const char *normal_data = "123";
        send(sockfd, normal_data, strlen(normal_data), 0);
        send(sockfd, oob_data, strlen(oob_data), MSG_OOB);
        send(sockfd, normal_data, strlen(normal_data), 0);
    }
    close(sockfd);
    return 0;
}
~~~

* 服务器

~~~c++
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>
#define BUF_SIZE 1024
int main(int argc, char *argv[])
{
    if (argc <= 2)
    {
        printf("usage:%s ip_address port_number\n", basename(argv[0]));
        return 1;
    }
    const char *ip = argv[1];
    int port = atoi(argv[2]);
    //地址
    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);
    //创建
    int sock = socket(PF_INET, SOCK_STREAM, 0);
    assert(sock >= 0);
    //命名
    int ret = bind(sock, (struct sockaddr *)&address, sizeof(address));
    assert(ret != -1);
    //监听
    ret = listen(sock, 5);
    assert(ret != -1);
    //连接地址
    struct sockaddr_in client;
    socklen_t client_addrlength = sizeof(client);
    //接受连接
    int connfd = accept(sock, (struct sockaddr *)&client, &client_addrlength);
    if (connfd < 0)
    {
        printf("errno is:%d\n", errno);
    }
    else
    {
        char buffer[BUF_SIZE];
        memset(buffer, '\0', BUF_SIZE);
        //使用监听socket接受消息
        ret = recv(connfd, buffer, BUF_SIZE - 1, 0);
        printf("got%d bytes of normal data'%s'\n", ret, buffer);
        memset(buffer, '\0', BUF_SIZE);
        ret = recv(connfd, buffer, BUF_SIZE - 1, MSG_OOB);
        printf("got%d bytes of oob data'%s'\n", ret, buffer);
        memset(buffer, '\0', BUF_SIZE);
        ret = recv(connfd, buffer, BUF_SIZE - 1, 0);
        printf("got%d bytes of normal data'%s'\n", ret, buffer);
        close(connfd);
    }
    close(sock);
    return 0;
}
~~~

* 运行并tcpdump抓取这一过程中客户端和服务器交换的TCP报文段

~~~
$./testoobrecv 192.168.1.109 54321  #在Kongming20上执行服务器程序，监听54321端口
$./testoobsend 192.168.1.109 54321  #在ernest-laptop上执行客户端程序
$sudo tcpdump-ntx-i eth0 port 54321
~~~

* 服务器输出

got 5 bytes of normal data'123ab'

got 1 bytes of oob data'c'

got 3 bytes of normal data'123'

由此可见，客户端发送给服务器的3字节的带外数据“abc”中，仅有最后一个字符“c”被服务器当成真正的带外数据接收。并且，服务器对正常数据的接收将被带外数据截断，即前一部分正常数据“123ab”和后续的正常数据“123”是不能被一个recv调用全部读出的。

* 含带外数据的TCP报文段

~~~
IP 192.168.1.108.60460＞192.168.1.109.54321:Flags[P.U],seq 4:7,ack 1,win 92,urg 3,options[nop,nop,TS val 102794322 ecr
154703423],length 3
~~~

这里我们第一次看到tcpdump输出标志U，这表示该TCP报文段的头部被设置了紧急标志。

“urg 3”是紧急偏移值，它指出带外数据在字节流中的位置的下一字节位置是7（3+4，其中4是该TCP报文段的序号值相对初始序号值的偏移）。

因此，带外数据是字节流中的第6字节，即字符“c”

值得一提的是，flags参数只对send和recv的当前调用生效，而后面我们将看到如何通过setsockopt系统调用永久性地修改socket的某些属性。

#### UDP数据读写

~~~c++
#include＜sys/types.h＞
#include＜sys/socket.h＞
ssize_t recvfrom(int sockfd,void*buf,size_t len,int flags,struct sockaddr*src_addr,socklen_t*addrlen);
ssize_t sendto(int sockfd,const void*buf,size_t len,int flags,const struct sockaddr*dest_addr,socklen_t addrlen);
~~~

* recvfrom读取sockfd上的数据，buf和len参数分别指定读缓冲区的位置和大小。因为UDP通信没有连接的概念，所以我们每次读取数据都需要获取发送端的socket地址，即参数src_addr所指的内容，addrlen参数则指定该地址的长度

* sendto往sockfd上写入数据，buf和len参数分别指定写缓冲区的位置和大小。dest_addr参数指定接收端的socket地址，addrlen参数则指定该地址的长度。

值得一提的是，recvfrom/sendto系统调用也可以用于面向连接（STREAM）的socket的数据读写，只需要把最后两个参数都设置为NULL以忽略发送端/接收端的socket地址（因为我们已经和对方建立了连接，所以已经知道其socket地址了）。

#### 通用数据读写函数

适用tcp和udp

~~~c
#include＜sys/socket.h＞
ssize_t recvmsg(int sockfd,struct msghdr*msg,int flags);
ssize_t sendmsg(int sockfd,struct msghdr*msg,int flags);
~~~

* sockfd参数指定被操作的目标socket。msg参数是msghdr结构体类型的指针，msghdr结构体的定义如下：

~~~c
struct msghdr
{
    void*msg_name;/*socket地址*/
    socklen_t msg_namelen;/*socket地址的长度*/
    struct iovec*msg_iov;/*分散的内存块，见后文*/
    int msg_iovlen;/*分散内存块的数量*/
    void*msg_control;/*指向辅助数据的起始位置*/
    socklen_t msg_controllen;/*辅助数据的大小*/
    int msg_flags;/*复制函数中的flags参数，并在调用过程中更新*/
};
~~~

msg_name成员指向一个socket地址结构变量。它指定通信对方的socket地址。对于面向连接的TCP协议，该成员没有意义，必须被设置为NULL。这是因为对数据流socket而言，对方的地址已经知道。msg_namelen成员则指定了msg_name所指socket地址的长度。msg_iov成员是iovec结构体类型的指针，iovec结构体的定义如下

~~~c
struct iovec
{
void*iov_base;/*内存起始地址*/
size_t iov_len;/*这块内存的长度*/
};
~~~

由上可见，iovec结构体封装了一块内存的起始位置和长度。msg_iovlen指定这样的iovec结构对象有多少个。对于recvmsg而言，数据将被读取并存放在msg_iovlen块分散的内存中，这些内存的位置和长度则由msg_iov指向的数组指定，这称为分散读（scatter read）；对于sendmsg而言，msg_iovlen块分散内存中的数据将被一并发送，这称为集中写（gather write）

msg_control和msg_controllen成员用于辅助数据的传送。我们不详细讨论它们，仅在第13章介绍如何使用它们来实现在进程间传递文件描述符。

msg_flags成员无须设定，它会复制recvmsg/sendmsg的flags参数的内容以影响数据读写过程。recvmsg还会在调用结束前，将某些更新后的标志设置到msg_flags中。

recvmsg/sendmsg的flags参数以及返回值的含义均与send/recv的flags参数及返回值相同。

由于socket连接是全双工的，这里的“读端”是针对通信对方而言的

### 实现自己的简单的tcp通信



### 带外标记

紧急标志

###地址信息函数

获取一个连接socket的本端socket地址，以及远端的socket地址

### socket选项

专门用来读取和设置socket文件描述符属性

### 网络信息API

ipv4转移到ipv6不便扩展，使用主机名访问机器，避免直接使用ip

## 高级I/O函数

不常用，但是特定和场景下性能优秀

* 用于创建文件描述符的函数，包括pipe、dup/dup2函数
* 用于读写数据的函数，包括readv/writev、sendfile、mmap/munmap、splice和tee函数
* 用于控制I/O行为和属性的函数，包括fcntl函数

### pipe函数

创建管道 进程间通信

## Linux服务器程序规范

* Linux服务器程序一般以后台进程形式运行。后台进程又称守护进程（daemon）。它没有控制终端，因而也不会意外接收到用户输入。守护进程的父进程通常是init进程（PID为1的进程）。
* Linux服务器程序通常有一套日志系统，它至少能输出日志到文件，有的高级服务器还能输出日志到专门的UDP服务器。大部分后台进程都在/var/log目录下拥有自己的日志目录。
* Linux服务器程序一般以某个专门的非root身份运行。比如mysqld、httpd、syslogd等后台进程，分别拥有自己的运行账户mysql、apache和syslog
* Linux服务器程序通常是可配置的。服务器程序通常能处理很多命令行选项，如果一次运行的选项太多，则可以用配置文件来管理。绝大多数服务器程序都有配置文件，并存放在/etc目录下。比如第4章讨论的squid服务器的配置文件是/etc/squid3/squid.conf。
* Linux服务器进程通常会在启动的时候生成一个PID文件并存入/var/run目录中，以记录该后台进程的PID。比如syslogd的PID文件是/var/run/syslogd.pid
* Linux服务器程序通常需要考虑系统资源和限制，以预测自身能承受多大负荷，比如进程可用文件描述符总数和内存总量等

### 日志

####Linux系统日志

工欲善其事，必先利其器。

服务器的调试和维护都需要一个专业的日志系统。Linux提供一个守护进程来处理系统日志——syslogd，不过现在的Linux系统上使用的都是它的升级版——rsyslogd。rsyslogd守护进程既能接收用户进程输出的日志，又能接收内核日志。用户进程是通过调用syslog函数生成系统日志的。该函数将日志输出到一个UNIX本地域socket类型（AF_UNIX）的文件/dev/log中，rsyslogd则监听该文件以获取用户进程的输出。内核日志在老的系统上是通过另外一个守护进程rklogd来管理的，rsyslogd利用额外的模块实现了相同的功能。内核日志由printk等函数打印至内核的环状缓存（ring buffer）中。环状缓存的内容直接映射到/proc/kmsg文件中。rsyslogd则通过读取该文件获得内核日志

rsyslogd守护进程在接收到用户进程或内核输入的日志后，会把它们输出至某些特定的日志文件。默认情况下，调试信息会保存至/var/log/debug文件，普通信息保存至/var/log/messages文件，内核消息则保存至/var/log/kern.log文件。不过，日志信息具体如何分发，可以在rsyslogd的配置文件中设置。rsyslogd的主配置文件是/etc/rsyslog.conf，其中主要可以设置的项包括：内核日志输入路径，是否接收UDP日志及其监听端口（默认是514，见/etc/services文件），是否接收TCP日志及其监听端口，日志文件的权限，包含哪些子配置文件（比如/etc/rsyslog.d/*.conf）。rsyslogd的子配置文件则指定各类日志的目标存储文件

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240903102130167.png" alt="image-20240903102130167" style="zoom:50%;" />

#### syslog函数

与rsyslogd守护进程通信

\#include＜syslog.h＞   void syslog(int priority,const char*message,...)



## 高性能服务器程序框架

* I/O处理单元。本章将介绍I/O处理单元的四种I/O模型和两种高效事件处理模式

* 逻辑单元。本章将介绍逻辑单元的两种高效并发模式，以及高效的逻辑处理方式——有限状态机
* 存储单元。本书不讨论存储单元，因为它只是服务器程序的可选模块，而且其内容与网络编程本身无关

### 服务器模型

#### C/S模型

TCP/IP协议在设计和实现上并没有客户端和服务器的概念，所有机器都是对等的，但是由于资源的垄断，自然采用了（客户端/服务器）模型

* 服务器启动后，首先创建一个（或多个）监听socket，并调用bind函数将其绑定到服务器感兴趣的端口上，然后调用listen函数等待客户连接。

* 服务器稳定运行之后，客户端就可以调用connect函数向服务器发起连接了。

* * 由于客户连接请求是随机到达的异步事件，服务器需要使用某种I/O模型来监听这一事件。I/O模型有多种，图8-2中，服务器使用的是I/O复用技术之一的**select系统调用**。当监听到连接请求后，服务器就调用accept函数接受它，并分配一个逻辑单元为新的连接服务。逻辑单元可以是新创建的子进程、子线程或者其他。图8-2中，服务器给客户端分配的逻辑单元是由fork系统调用创建的子进程。逻辑单元读取客户请求，处理该请求，然后将处理结果返回给客户端。


* 服务器在处理一个客户请求的同时还会继续监听其他客户请求，否则就变成了效率低下的串行服务器了图8-2中，服务器同时监听多个客户请求是通过select系统调用实现的。

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240903102816273.png" alt="image-20240903102816273" style="zoom:50%;" />

C/S模型非常适合资源相对集中的场合，并且它的实现也很简单，但其缺点也很明显：服务器是通信的中心，当访问量过大时，可能所有客户都将得到很慢的响应。下面讨论的P2P模型解决了这个问题

####P2P模型

P2P（Peer to Peer，点对点）模型比C/S模型更符合网络通信的实际情况。它摒弃了以服务器为中心的格局，让网络上所有主机重新回归对等的地位。

P2P模型如图8-3a所示。P2P模型使得每台机器在消耗服务的同时也给别人提供服务，这样资源能够充分、自由地共享。云计算机群可以看作P2P模型的一个典范。但P2P模型的缺点也很明显：当用户之间传输的请求过多时，网络的负载将加重。

图8-3a所示的P2P模型存在一个显著的问题，即主机之间很难互相发现。所以实际使用的P2P模型通常带有一个专门的发现服务器，如图8-3b所示。这个发现服务器通常还提供查找服务（甚至还可以提供内容服务），使每个客户都能尽快地找到自己需要的资源。

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240903103826023.png" alt="image-20240903103826023" style="zoom:50%;" />

从编程角度来讲，P2P模型可以看作C/S模型的扩展：每台主机既是客户端，又是服务器。因此，我们仍然采用C/S模型来讨论网络编程

### 服务器编程框架

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240903103947062.png" alt="image-20240903103947062" style="zoom:50%;" />

该图既能用来描述一台服务器，也能用来描述一个服务器机群

| 模块         | 单个服务器                   | 服务器集群                   |
| ------------ | ---------------------------- | ---------------------------- |
| I/O处理单元  | 处理客户端连接，读写网络数据 | 作为接入服务器，实现负载均衡 |
| 逻辑单元     | 业务进程，线程               | 逻辑服务器                   |
| 网络存储单元 | 本地数据库，文件，缓存       | 数据库服务器                 |
| 请求队列     | 各单元之间通信方式           | 各服务器之间永久tcp连接      |

* 负载均衡，从所有逻辑服务器中选取负荷最小的一台来为新客户服务。
* 服务器通常拥有多个逻辑单元，以实现对多个客户任务的并行处理。
* 网络存储单元不是必须的，比如ssh、telnet等登录服务就不需要这个单元
* 请求队列是各单元之间的通信方式的抽象。I/O处理单元接收到客户请求时，需要以某种方式通知一个逻辑单元来处理该请求。同样，多个逻辑单元同时访问一个存储单元时，也需要采用某种机制来协调处理竞态条件。请求队列通常被实现为池的一部分。对于服务器机群而言，请求队列是各台服务器之间预先建立的、静态的、永久的TCP连接。这种TCP连接能提高服务器之间交换数据的效率，因为它避免了动态建立TCP连接导致的额外的系统开销。

### I/O模型

socket在创建的时候默认是阻塞的。我们可以给socket系统调用的第2个参数传递SOCK_NONBLOCK标志，或者通过fcntl系统调用的F_SETFL命令，将其设置为非阻塞的。阻塞和非阻塞的概念能应用于所有文件描述符，而不仅仅是socket。我们称阻塞的文件描述符为阻塞I/O，称非阻塞的文件描述符为非阻塞I/O。

* 针对阻塞I/O执行的系统调用可能因为无法立即完成而被操作系统挂起，直到等待的事件发生为止。比如，客户端通过connect向服务器发起连接时，connect将首先发送同步报文段给服务器，然后等待服务器返回确认报文段。如果服务器的确认报文段没有立即到达客户端，则connect调用将被挂起，直到客户端收到确认报文段并唤醒connect调用。socket的基础API中，可能被阻塞的系统调用包括accept、send、recv和connect。

* 针对非阻塞I/O执行的系统调用则总是立即返回，而不管事件是否已经发生。如果事件没有立即发生，这些系统调用就返回-1，和出错的情况一样。此时我们必须**根据errno来区分这两种情况**。对accept、send和recv而言，事件未发生时errno通常被设置成EAGAIN（意为“再来一次”）或者EWOULDBLOCK（意为“期望阻塞”）；对connect而言，errno则被设置成EINPROGRESS（意为“在处理中”）。很显然，我们只有在事件已经发生的情况下操作非阻塞I/O（读、写等），才能提高程序的效率。因此，非阻塞I/O通常要和其他I/O通知机制一起使用，比如I/O复用和SIGIO信号。

* I/O复用是最常使用的I/O通知机制。它指的是，应用程序通过I/O复用函数向内核注册一组事件，内核通过I/O复用函数把其中就绪的事件通知给应用程序。Linux上常用的I/O复用函数是select、poll和epoll_wait。需要指出的是，I/O复用函数本身是阻塞的，它们能提高程序效率的原因在于它们具有同时监听多个I/O事件的能力。

* SIGIO信号也可以用来报告I/O事件。我们可以为一个目标文件描述符指定宿主进程，那么被指定的宿主进程将捕获到SIGIO信号。这样，当目标文件描述符上有事件发生时，SIGIO信号的信号处理函数将被触发，我们也就可以在该信号处理函数中对目标文件描述符执行非阻塞I/O操作了。

从理论上说，阻塞I/O、I/O复用和信号驱动I/O都是同步I/O模型。因为在这三种I/O模型中，I/O的读写操作，都是在I/O事件发生之后，由应用程序来完成的。而POSIX规范所定义的异步I/O模型则不同。对异步I/O而言，用户可以直接对I/O执行读写操作，这些操作告诉内核用户读写缓冲区的位置，以及I/O操作完成之后内核通知应用程序的方式。异步I/O的读写操作总是立即返回，而不论I/O是否是阻塞的，因为真正的读写操作已经由内核接管。也就是说，同步I/O模型要求用户代码自行执行I/O操作（将数据从内核缓冲区读入用户缓冲区，或将数据从用户缓冲区写入内核缓冲区），而异步I/O机制则由内核来执行I/O操作（数据在内核缓冲区和用户缓冲区之间的移动是由内核在“后台”完成的）。你可以这样认为，同步I/O向应用程序通知的是I/O就绪事件，而异步I/O向应用程序通知的是I/O完成事件。Linux环境下，aio.h头文件中定义的函数提供了对异步I/O的支持。不过这部分内容不是本书的重点，所以只做简单的讨论

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240903105336713.png" alt="image-20240903105336713" style="zoom:50%;" />

### 两种高效的事件处理模式

服务器程序通常需要处理三类事件：I/O事件、信号及定时事件。

两种高效的事件处理模式：Reactor和Proactor。

同步I/O模型通常用于实现Reactor模式，异步I/O模型则用于实现Proactor模式。

#### Reactor模式

主线程（I/O处理单元）只负责监听文件描述上是否有事件发生，有的话就立即将该事件通知工作线程（逻辑单元，下同）。除此之外，主线程不做任何其他实质性的工作。读写数据，接受新的连接，以及处理客户请求均在工作线程中完成。

使用同步I/O模型（以epoll_wait为例）实现的Reactor模式的工作流程是：

1）主线程往epoll内核事件表中注册socket上的读就绪事件。

2）主线程调用epoll_wait等待socket上有数据可读。

3）当socket上有数据可读时，epoll_wait通知主线程。主线程则将socket可读事件放入请求队列。

4）睡眠在请求队列上的某个工作线程被唤醒，它从socket读取数据，并处理客户请求，然后往epoll内核事件表中注册该socket上的写就绪事件。

5）主线程调用epoll_wait等待socket可写。

6）当socket可写时，epoll_wait通知主线程。主线程将socket可写事件放入请求队列。

7）睡眠在请求队列上的某个工作线程被唤醒，它往socket上写入服务器处理客户请求的结果

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240903105623824.png" alt="image-20240903105623824" style="zoom:50%;" />

工作线程从请求队列中取出事件后，将根据事件的类型来决定如何处理它：对于可读事件，执行读数据和处理请求的操作；对于可写事件，执行写数据的操作。因此，所示的Reactor模式中，没必要区分所谓的“读工作线程”和“写工作线程”。

#### Proactor模式

与Reactor模式不同，Proactor模式将所有I/O操作都交给主线程和内核来处理，工作线程仅仅负责业务逻辑。Proactor模式更符合服务器编程框架。

使用异步I/O模型（以aio_read和aio_write为例）实现的Proactor模式的工作流程是：

1）主线程调用aio_read函数向内核注册socket上的读完成事件，并告诉内核用户读缓冲区的位置，以及读操作完成时如何通知应用程序（这里以信号为例，详情请参考sigevent的man手册）。

2）主线程继续处理其他逻辑。

3）当socket上的数据被读入用户缓冲区后，内核将向应用程序发送一个信号，以通知应用程序数据已经可用。

4）应用程序预先定义好的信号处理函数选择一个工作线程来处理客户请求。工作线程处理完客户请求之后，调用aio_write函数向内核注册socket上的写完成事件，并告诉内核用户写缓冲区的位置，以及写操作完成时如何通知应用程序（仍然以信号为例）。

5）主线程继续处理其他逻辑。

6）当用户缓冲区的数据被写入socket之后，内核将向应用程序发送一个信号，以通知应用程序数据已经发送完毕。

7）应用程序预先定义好的信号处理函数选择一个工作线程来做善后处理，比如决定是否关闭socket。

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240903110142310.png" alt="image-20240903110142310" style="zoom:50%;" />

连接socket上的读写事件是通过aio_read/aio_write向内核注册的，因此内核将通过信号来向应用程序报告连接socket上的读写事件。所以，主线程中的epoll_wait调用仅能用来检测监听socket上的连接请求事件，而不能用来检测连接socket上的读写事件。

#### 模拟Proactor模式

使用同步I/O方式模拟出Proactor模式的一种方法。其原理是：主线程执行数据读写操作，读写完成之后，主线程向工作线程通知这一“完成事件”。那么从工作线程的角度来看，它们就直接获得了数据读写的结果，接下来要做的只是对读写的结果进行逻辑处理。

使用同步I/O模型（仍然以epoll_wait为例）模拟出的Proactor模式的工作流程如下：

1）主线程往epoll内核事件表中注册socket上的读就绪事件。

2）主线程调用epoll_wait等待socket上有数据可读。

3）当socket上有数据可读时，epoll_wait通知主线程。主线程从socket循环读取数据，直到没有数据可读，然后将读取到的数据封装成一个请求对象并插入请求队列。

4）睡眠在请求队列上的某个工作线程被唤醒，它获得请求对象并处理客户请求，然后往epoll内核事件表中注册socket上的写就绪事件。

5）主线程调用epoll_wait等待socket可写。

6）当socket可写时，epoll_wait通知主线程。主线程往socket上写入服务器处理客户请求的结果。

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240903110744520.png" alt="image-20240903110744520" style="zoom:50%;" />

### 两种高效的并发模式

并发编程的目的是让程序“同时”执行多个任务。如果程序是计算密集型的，并发编程并没有优势，反而由于任务的切换使效率降低。

但如果程序是I/O密集型的，比如经常读写文件，访问数据库等，则情况就不同了。由于I/O操作的速度远没有CPU的计算速度快，所以让程序阻塞于I/O操作将浪费大量的CPU时间。

并发编程主要有多进程和多线程两种方式，这一节先讨论并发模式。

服务器主要有两种并发编程模式：半同步/半异步（half-sync/halfasync）模式和领导者/追随者（Leader/Followers）模式。

#### 半同步/半异步模式

在I/O模型中，“同步”和“异步”区分的是内核向应用程序通知的是何种I/O事件（是就绪事件还是完成事件），以及该由谁来完成I/O读写（是应用程序还是内核）。

在并发模式中，“同步”指的是程序完全按照代码序列的顺序执行；“异步”指的是程序的执行需要由系统事件来驱动。常见的系统事件包括中断、信号等。

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240903111202443.png" alt="image-20240903111202443" style="zoom:50%;" />

按照同步方式运行的线程称为同步线程，按照异步方式运行的线程称为异步线程。显然，异步线程的执行效率高，实时性强，这是很多嵌入式程序采用的模型。但编写以异步方式执行的程序相对复杂，难于调试和扩展，而且不适合于大量的并发。而同步线程则相反，它虽然效率相对较低，实时性较差，但逻辑简单。因此，对于像服务器这种既要求较好的实时性，又要求能同时处理多个客户请求的应用程序，我们就应该同时使用同步线程和异步线程来实现，即采用半同步/半异步模式来实现。

半同步/半异步模式中，同步线程用于处理客户逻辑；异步线程用于处理I/O事件。异步线程监听到客户请求后，就将其封装成请求对象并插入请求队列中。请求队列将通知某个工作在同步模式的工作线程来读取并处理该请求对象。具体选择哪个工作线程来为新的客户请求服务，则取决于请求队列的设计。比如最简单的轮流选取工作线程的Round Robin算法，也可以通过条件变量或信号量来随机地选择一个工作线程。

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240903111908331.png" alt="image-20240903111908331" style="zoom:50%;" />

在服务器程序中，如果结合考虑两种事件处理模式和几种I/O模型，则半同步/半异步模式就存在多种变体。其中有一种变体称为半同步/半反应堆（half-sync/half-reactive）模式

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240903111921950.png" alt="image-20240903111921950" style="zoom:50%;" />

异步线程只有一个，由主线程来充当。它负责监听所有socket上的事件。如果监听socket上有可读事件发生，即有新的连接请求到来，主线程就接受之以得到新的连接socket，然后往epoll内核事件表中注册该socket上的读写事件。如果连接socket上有读写事件发生，即有新的客户请求到来或有数据要发送至客户端，主线程就将该连接socket插入请求队列中。所有工作线程都睡眠在请求队列上，当有任务到来时，它们将通过竞争（比如申请互斥锁）获得任务的接管权。这种竞争机制使得只有空闲的工作线程才有机会来处理新任务，这是很合理的。

主线程插入请求队列中的任务是就绪的连接socket。这说明该图所示的半同步/半反应堆模式采用的事件处理模式是Reactor模式：它要求工作线程自己从socket上读取客户请求和往socket写入服务器应答。这就是该模式的名称中“half-reactive”的含义。实际上，半同步/半反应堆模式也可以使用模拟的Proactor事件处理模式，即由主线程来完成数据的读写。在这种情况下，主线程一般会将应用程序数据、任务类型等信息封装为一个任务对象，然后将其（或者指向该任务对象的一个指针）插入请求队列。工作线程从请求队列中取得任务对象之后，即可直接处理之，而无须执行读写操作了。我们将在第15章给出一个用半同步/半反应堆模式实现的简单Web服务器的代码。

半同步/半反应堆模式存在如下缺点：

* 主线程和工作线程共享请求队列。主线程往请求队列中添加任务，或者工作线程从请求队列中取出任务，都需要对请求队列加锁保护，从而白白耗费CPU时间。

* 每个工作线程在同一时间只能处理一个客户请求。如果客户数量较多，而工作线程较少，则请求队列中将堆积很多任务对象，客户端的响应速度将越来越慢。如果通过增加工作线程来解决这一问题，则工作线程的切换也将耗费大量CPU时间。

一种高效的半同步/半异步模式 每个工作线程都能同时处理多个客户连接。

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240903112357921.png" alt="image-20240903112357921" style="zoom:50%;" />

主线程只管理监听socket，连接socket由工作线程来管理。当有新的连接到来时，主线程就接受之并将新返回的连接socket派发给某个工作线程，此后该新socket上的任何I/O操作都由被选中的工作线程来处理，直到客户关闭连接。主线程向工作线程派发socket的最简单的方式，是往它和工作线程之间的管道里写数据。工作线程检测到管道上有数据可读时，就分析是否是一个新的客户连接请求到来。如果是，则把该新socket上的读写事件注册到自己的epoll内核事件表中。

每个线程（主线程和工作线程）都维持自己的事件循环，它们各自独立地监听不同的事件。因此，在这种高效的半同步/半异步模式中，每个线程都工作在异步模式，所以它并非严格意义上的半同步/半异步模式。我们将在第15章给出一个用这种高效的半同步/半异步模式实现的简单CGI服务器的代码。

#### 领导者/追随者模式

领导者/追随者模式是多个工作线程轮流获得事件源集合，轮流监听、分发并处理事件的一种模式。在任意时间点，程序都仅有一个领导者线程，它负责监听I/O事件。而其他线程则都是追随者，它们休眠在线程池中等待成为新的领导者。当前的领导者如果检测到I/O事件，首先要从线程池中推选出新的领导者线程，然后处理I/O事件。此时，新的领导者等待新的I/O事件，而原来的领导者则处理I/O事件，二者实现了并发。

领导者/追随者模式包含如下几个组件：句柄集（HandleSet）、线程集（ThreadSet）、事件处理器（EventHandler）和具体的事件处理器（ConcreteEventHandler）。

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240903112552656.png" alt="image-20240903112552400" style="zoom:50%;" />



### 有限状态机

有的应用层协议头部包含数据包类型字段，每种类型可以映射为逻辑单元的一种执行状态，服务器可以根据它来编写相应的处理逻辑

~~~c
STATE_MACHINE(Package_pack){
	PackageType_type=_pack.GetType();
	switch(_type){
		case type_A:
			process_package_A(_pack);
			break;
		case type_B:
			process_package_B(_pack);
			break;
	}
}
~~~

该状态机的每个状态都是相互独立的，即状态之间没有相互转移。状态之间的转移是需要状态机内部驱动的

~~~~c
STATE_MACHINE(){
	State cur_State=type_A;
	while(cur_State!=type_C){
		Package_pack=getNewPackage();
		switch(cur_State){
			case type_A:
                  process_package_state_A(_pack);
				cur_State=type_B;
				break;
			case type_B:
				process_package_state_B(_pack);
				cur_State=type_C;
				break;
		}
	}
}
~~~~

该状态机包含三种状态：type_A、type_B和type_C，其中type_A是状态机的开始状态，type_C是状态机的结束状态。状态机的当前状态记录在cur_State变量中。在一趟循环过程中，状态机先通过getNewPackage方法获得一个新的数据包，然后根据cur_State变量的值判断如何处理该数据包。数据包处理完之后，状态机通过给cur_State变量传递目标状态值来实现状态转移。那么当状态机进入下一趟循环时，它将执行新的状态对应的逻辑。

下面我们考虑有限状态机应用的一个实例：HTTP请求的读取和分析。很多网络协议，包括TCP协议和IP协议，都在其头部中提供头部长度字段。程序根据该字段的值就可以知道是否接收到一个完整的协议头部。但HTTP协议并未提供这样的头部长度字段，并且其头部长度变化也很大，可以只有十几字节，也可以有上百字节。根据协议规定，

我们判断HTTP头部结束的依据是遇到一个空行，该空行仅包含一对回车换行符（＜CR＞＜LF＞）。如果一次读操作没有读入HTTP请求的整个头部，即没有遇到空行，那么我们必须等待客户继续写数据并再次读入。因此，我们每完成一次读操作，就要分析新读入的数据中是否有空行。不过在寻找空行的过程中，我们可以同时完成对整个HTTP请求头部的分析（记住，空行前面还有请求行和头部域），以提高解析HTTP请求的效率。代码清单8-3使用主、从两个有限状态机实现了最简单的HTTP请求的读取和分析。为了使表述简洁，我们约定，直接称HTTP请求的一行（包括请求行和头部字段）为行

~~~c++
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#define BUFFER_SIZE 4096 /*读缓冲区大小*/
/*主状态机的两种可能状态，分别表示：当前正在分析请求行，当前正在分析头部字段*/
enum CHECK_STATE
{
    CHECK_STATE_REQUESTLINE = 0,
    CHECK_STATE_HEADER
};
/*从状态机的三种可能状态，即行的读取状态，分别表示：读取到一个完整的行、行出错和行数据尚且不完整*/
enum LINE_STATUS
{
    LINE_OK = 0,
    LINE_BAD,
    LINE_OPEN
};
/*服务器处理HTTP请求的结果：NO_REQUEST表示请求不完整，需要继续读取客户数
据；GET_REQUEST表示获得了一个完整的客户请求；BAD_REQUEST表示客户请求有语法错
误；FORBIDDEN_REQUEST表示客户对资源没有足够的访问权限；INTERNAL_ERROR表示服
务器内部错误；CLOSED_CONNECTION表示客户端已经关闭连接了*/
enum HTTP_CODE
{
    NO_REQUEST,
    GET_REQUEST,
    BAD_REQUEST,
    FORBIDDEN_REQUEST,
    INTERNAL_ERROR,
    CLOSED_CONNECTION
};
/*为了简化问题，我们没有给客户端发送一个完整的HTTP应答报文，而只是根据服务器
的处理结果发送如下成功或失败信息*/
static const char *szret[] = {"I get a correct result\n", "Something wrong\n "};
/*从状态机，用于解析出一行内容*/
LINE_STATUS parse_line(char *buffer, int &checked_index, int &read_index)
{
    char temp;
    /*checked_index指向buffer（应用程序的读缓冲区）中当前正在分析的字节，
    read_index指向buffer中客户数据的尾部的下一字节。buffer中第0～checked_index
    字节都已分析完毕，第checked_index～(read_index-1)字节由下面的循环挨个分析*/
    for (; checked_index < read_index; ++checked_index)
    {
        /*获得当前要分析的字节*/
        temp = buffer[checked_index];
        /*如果当前的字节是“\r”，即回车符，则说明可能读取到一个完整的行*/
        if (temp == '\r')
        {
            /*如果“\r”字符碰巧是目前buffer中的最后一个已经被读入的客户数据，那么这次分
            析没有读取到一个完整的行，返回LINE_OPEN以表示还需要继续读取客户数据才能进一步分
            析*/
            if ((checked_index + 1) == read_index)
            {
                return LINE_OPEN;
            }
            /*如果下一个字符是“\n”，则说明我们成功读取到一个完整的行*/
            else if (buffer[checked_index + 1] == '\n')
            {
                buffer[checked_index++] = '\0';
                buffer[checked_index++] = '\0';
                return LINE_OK;
            }
            /*否则的话，说明客户发送的HTTP请求存在语法问题*/
            return LINE_BAD;
        }
        /*如果当前的字节是“\n”，即换行符，则也说明可能读取到一个完整的行*/
        else if (temp == '\n')
        {
            if ((checked_index > 1) && buffer[checked_index - 1] == '\r')
            {
                buffer[checked_index - 1] = '\0';
                buffer[checked_index++] = '\0';
                return LINE_OK;
            }
            return LINE_BAD;
        }
    }
    /*如果所有内容都分析完毕也没遇到“\r”字符，则返回LINE_OPEN，表示还需要继续读
    取客户数据才能进一步分析*/
    return LINE_OPEN;
}
/*分析请求行*/
HTTP_CODE parse_requestline(char *temp, CHECK_STATE &checkstate)
{
    char *url = strpbrk(temp, "\t");
    /*如果请求行中没有空白字符或“\t”字符，则HTTP请求必有问题*/
    if (!url)
    {
        return BAD_REQUEST;
    }
    *url++ = '\0';
    char *method = temp;
    if (strcasecmp(method, "GET") == 0) /*仅支持GET方法*/
    {
        printf("The request method is GET\n");
    }
    else
    {
        return BAD_REQUEST;
    }
    url += strspn(url, "\t");
    char *version = strpbrk(url, "\t");
    if (!version)
    {
        return BAD_REQUEST;
    }
    *version++ = '\0';
    version += strspn(version, "\t");
    /*仅支持HTTP/1.1*/
    if (strcasecmp(version, "HTTP/1.1") != 0)
    {
        return BAD_REQUEST;
    }
    /*检查URL是否合法*/
    if (strncasecmp(url, "http://", 7) == 0)
    {
        url += 7;
        url = strchr(url, '/');
    }
    if (!url || url[0] != '/')
    {
        return BAD_REQUEST;
    }
    printf("The request URL is:%s\n", url);
    /*HTTP请求行处理完毕，状态转移到头部字段的分析*/
    checkstate = CHECK_STATE_HEADER;
    return NO_REQUEST;
}
/*分析头部字段*/
HTTP_CODE parse_headers(char *temp)
{
    /*遇到一个空行，说明我们得到了一个正确的HTTP请求*/
    if (temp[0] == '\0')
    {
        return GET_REQUEST;
    }
    else if (strncasecmp(temp, "Host:", 5) == 0) /*处理“HOST”头部字段*/
    {
        temp += 5;
        temp += strspn(temp, "\t");
        printf("the request host is:%s\n", temp);
    }
    else /*其他头部字段都不处理*/
    {
        printf("I can not handle this header\n");
    }
    return NO_REQUEST;
}
/*分析HTTP请求的入口函数*/
HTTP_CODE parse_content(char *buffer, int &checked_index, CHECK_STATE &checkstate, int &read_index, int &start_line)
{
    LINE_STATUS linestatus = LINE_OK; /*记录当前行的读取状态*/
    HTTP_CODE retcode = NO_REQUEST;   /*记录HTTP请求的处理结果*/
    /*主状态机，用于从buffer中取出所有完整的行*/
    while ((linestatus = parse_line(buffer, checked_index, read_index)) == LINE_OK)
    {
        char *temp = buffer + start_line; /*start_line是行在buffer中的起始位置*/
        start_line = checked_index;       /*记录下一行的起始位置*/
        /*checkstate记录主状态机当前的状态*/
        switch (checkstate)
        {
        case CHECK_STATE_REQUESTLINE: /*第一个状态，分析请求行*/
        {
            retcode = parse_requestline(temp, checkstate);
            if (retcode == BAD_REQUEST)
            {
                return BAD_REQUEST;
            }
            break;
        }
        case CHECK_STATE_HEADER: /*第二个状态，分析头部字段*/
        {
            retcode = parse_headers(temp);
            if (retcode == BAD_REQUEST)
            {
                return BAD_REQUEST;
            }
            else if (retcode == GET_REQUEST)
            {
                return GET_REQUEST;
            }
            break;
        }
        default:
        {
            return INTERNAL_ERROR;
        }
        }
    }
    /*若没有读取到一个完整的行，则表示还需要继续读取客户数据才能进一步分析*/
    if (linestatus == LINE_OPEN)
    {
        return NO_REQUEST;
    }
    else
    {
        return BAD_REQUEST;
    }
}
int main(int argc, char *argv[])
{
    if (argc <= 2)
    {
        printf("usage:%s ip_address port_number\n", basename(argv[0]));
        return 1;
    }
    const char *ip = argv[1];
    int port = atoi(argv[2]);
    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);
    int listenfd = socket(PF_INET, SOCK_STREAM, 0);
    assert(listenfd >= 0);
    int ret = bind(listenfd, (struct sockaddr *)&address, sizeof(address));
    assert(ret != -1);
    ret = listen(listenfd, 5);
    assert(ret != -1);
    struct sockaddr_in client_address;
    socklen_t client_addrlength = sizeof(client_address);
    int fd = accept(listenfd, (struct sockaddr *)&client_address, &client_addrlength);
    if (fd < 0)
    {
        printf("errno is:%d\n", errno);
    }
    else
    {
        char buffer[BUFFER_SIZE]; /*读缓冲区*/
        memset(buffer, '\0', BUFFER_SIZE);
        int data_read = 0;
        int read_index = 0;    /*当前已经读取了多少字节的客户数据*/
        int checked_index = 0; /*当前已经分析完了多少字节的客户数据*/
        int start_line = 0;    /*行在buffer中的起始位置*/
        /*设置主状态机的初始状态*/
        CHECK_STATE checkstate = CHECK_STATE_REQUESTLINE;
        while (1) /*循环读取客户数据并分析之*/
        {
            data_read = recv(fd, buffer + read_index, BUFFER_SIZE - read_index, 0);
            if (data_read == -1)
            {
                printf("reading failed\n");
                break;
            }
            else if (data_read == 0)
            {
                printf("remote client has closed the connection\n");
                break;
            }
            read_index += data_read;
            /*分析目前已经获得的所有客户数据*/
            HTTP_CODE
            result = parse_content(buffer, checked_index, checkstate, read_index, start_line);
            if (result == NO_REQUEST) /*尚未得到一个完整的HTTP请求*/
            {
                continue;
            }
            else if (result == GET_REQUEST) /*得到一个完整的、正确的HTTP请求*/
            {
                send(fd, szret[0], strlen(szret[0]), 0);
                break;
            }
            else /*其他情况表示发生错误*/
            {
                send(fd, szret[1], strlen(szret[1]), 0);
                break;
            }
        }
        close(fd);
    }
    close(listenfd);
    return 0;
}
~~~

我们将代码清单中的两个有限状态机分别称为主状态机和从状态机，这体现了它们之间的关系：主状态机在内部调用从状态机。下面先分析从状态机，即parse_line函数，它从buffer中解析出一个行。图8-15描述了其可能的状态及状态转移过程

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240903113737658.png" alt="image-20240903113737658" style="zoom:50%;" />



### 提高服务器性能的其他建议

#### 池

既然服务器的硬件资源“充裕”，那么提高服务器性能的一个很直接的方法就是以空间换时间，这就是池（pool）的概念。池是一组资源的集合，这组资源在服务器启动之初就被完全创建好并初始化，这称为静态资源分配。当服务器进入正式运行阶段，即开始处理客户请求的时候，如果它需要相关的资源，就可以直接从池中获取，无须动态分配。很显然，直接从池中取得所需资源比动态分配资源的速度要快得多。当服务器处理完一个客户连接后，可以把相关的资源放回池中，无须执行系统调用来释放资源。从最终的效果来看，池相当于服务器管理系统资源的应用层设施，它避免了服务器对内核的频繁访问。

不过，既然池中的资源是预先静态分配的，我们就无法预期应该分配多少资源。这个问题又该如何解决呢？

最简单的解决方案就是分配“足够多”的资源 这通常会导致资源的浪费

还有一种解决方案是预先分配一定的资源，此后如果发现资源不够用，就再动态分配一些并加入池中。

根据不同的资源类型，池可分为多种，常见的有内存池、进程池、线程池和连接池。

* 内存池通常用于socket的接收缓存和发送缓存。对于某些长度有限的客户请求，比如HTTP请求，预先分配一个大小足够（比如5000字节）的接收缓存区是很合理的。当客户请求的长度超过接收缓冲区的大小时，我们可以选择丢弃请求或者动态扩大接收缓冲区。

* 进程池和线程池都是并发编程常用的“伎俩”。当我们需要一个工作进程或工作线程来处理新到来的客户请求时，我们可以直接从进程池或线程池中取得一个执行实体，而无须动态地调用fork或pthread_create等函数来创建进程和线程。

* 连接池通常用于服务器或服务器机群的内部永久连接。每个逻辑单元可能都需要频繁地访问本地的某个数据库。连接池是服务器预先和数据库程序建立的一组连接的集合。当某个逻辑单元需要访问数据库时，它可以直接从连接池中取得一个连接的实体并使用之。待完成数据库的访问之后，逻辑单元再将该连接返还给连接池。

#### 数据复制

高性能服务器应该避免不必要的数据复制，尤其是当数据复制发生在用户代码和内核之间的时候。如果内核可以直接处理从socket或者文件读入的数据，则应用程序就没必要将这些数据从内核缓冲区复制到应用程序缓冲区中。这里说的“直接处理”指的是应用程序不关心这些数据的内容，不需要对它们做任何分析。比如ftp服务器，当客户请求一个文件时，服务器只需要检测目标文件是否存在，以及客户是否有读取它的权限，而绝对不会关心文件的具体内容。这样的话，ftp服务器就无须把目标文件的内容完整地读入到应用程序缓冲区中并调用send函数来发送，而是可以使用“零拷贝”函数sendfile来直接将其发送给客户端。

此外，用户代码内部（不访问内核）的数据复制也是应该避免的。举例来说，当两个工作进程之间要传递大量的数据时，我们就应该考虑使用共享内存来在它们之间直接共享这些数据，而不是使用管道或者消息队列来传递。又比如代码清单8-3所示的解析HTTP请求的实例中，我们用指针（start_line）来指出每个行在buffer中的起始位置，以便随后对行内容进行访问，而不是把行的内容复制到另外一个缓冲区中来使用，因为这样既浪费空间，又效率低下。

#### 上下文切换和锁

并发程序必须考虑上下文切换（context switch）的问题，即进程切换或线程切换导致的的系统开销。即使是I/O密集型的服务器，也不应该使用过多的工作线程（或工作进程，下同），否则线程间的切换将占用大量的CPU时间，服务器真正用于处理业务逻辑的CPU时间的比重就显得不足了。因此，为每个客户连接都创建一个工作线程的服务器模型是不可取的。图8-11所描述的半同步/半异步模式是一种比较合理的解决方案，它允许一个线程同时处理多个客户连接。此外，多线程服务器的一个优点是不同的线程可以同时运行在不同的CPU上。当线程的数量不大于CPU的数目时，上下文的切换就不是问题了。

并发程序需要考虑的另外一个问题是共享资源的加锁保护。锁通常被认为是导致服务器效率低下的一个因素，因为由它引入的代码不仅不处理任何业务逻辑，而且需要访问内核资源。因此，服务器如果有更好的解决方案，就应该避免使用锁。显然，图8-11所描述的半同步/半异步模式就比图8-10所描述的半同步/半反应堆模式的效率高。如果服务器必须使用“锁”，则可以考虑减小锁的粒度，比如使用读写锁。当所有工作线程都只读取一块共享内存的内容时，读写锁并不会增加系统的额外开销。只有当其中某一个工作线程需要写这块内存时，系统才必须去锁住这块区域

## I/O复用

I/O复用允许单个线程或进程同时处理多个I/O操作,而不需要为每个I/O操作创建一个新的线程或进程

当内核发现进程指定的一个或多个I/O条件就绪时（如输入数据已准备好被读取，或输出缓冲区有空间接收数据），它会给进程一个通知。这种功能主要通过`select`、`poll`和`epoll`等系统调用实现。

I/O复用使得程序能同时监听多个文件描述符，这对提高程序的性能至关重要

* 客户端程序要同时处理多个socket。比如本章将要讨论的非阻塞connect技术
* 客户端程序要同时处理用户输入和网络连接。比如本章将要讨论的聊天室程序
* TCP服务器要同时处理监听socket和连接socket。这是I/O复用使用最多的场合。
* 服务器要同时处理TCP请求和UDP请求。比如本章将要讨论的回射服务器
* 服务器要同时监听多个端口，或者处理多种服务。比如本章将要讨论的xinetd服务器

I/O复用虽然能同时监听多个文件描述符，但它本身是阻塞的。并且当多个文件描述符同时就绪时，如果不采取额外的措施，程序就只能按顺序依次处理其中的每一个文件描述符，这使得服务器程序看起来像是串行工作的。如果要实现并发，只能使用多进程或多线程等编程手段。

###select系统调用

select系统调用的用途是：在一段指定时间内，监听用户感兴趣的文件描述符上的可读、可写和异常等事件。

#### select API

~~~c
#include＜sys/select.h＞
int select(int nfds,fd_set* readfds,fd_set* writefds,fd_set* exceptfds,struct timeval*timeout);
~~~

* nfds参数指定被监听的文件描述符的总数。它通常被设置为select监听的所有文件描述符中的最大值加1，因为文件描述符是从0开始计数的
* readfds、writefds和exceptfds参数分别指向**可读**、**可写**和**异常**等事件对应的文件描述符集合。应用程序调用select函数时，通过这3个参数传入自己感兴趣的文件描述符。select调用返回时，内核将修改它们来通知应用程序哪些文件描述符已经就绪。

fd_set结构体

~~~c
#include＜typesizes.h＞
#define__FD_SETSIZE 1024
#include＜sys/select.h＞
#define FD_SETSIZE__FD_SETSIZE
typedef long int__fd_mask;
#undef__NFDBITS
#define__NFDBITS(8*(int)sizeof(__fd_mask))
typedef struct{
#ifdef__USE_XOPEN
__fd_mask fds_bits[__FD_SETSIZE/__NFDBITS];
#define__FDS_BITS(set)((set)-＞fds_bits)
#else
__fd_mask__fds_bits[__FD_SETSIZE/__NFDBITS];
#define__FDS_BITS(set)((set)-＞__fds_bits)
#endif
}fd_set;
~~~

由以上定义可见，fd_set结构体仅包含一个整型数组，该数组的每个元素的每一位（bit）标记一个文件描述符。fd_set能容纳的文件描述符数量由FD_SETSIZE指定，这就限制了select能同时处理的文件描述符的总量。

由于位操作过于烦琐，我们应该使用下面的一系列宏来访问fd_set结构体中的位：

~~~c
#include＜sys/select.h＞
FD_ZERO(fd_set*fdset);/*清除fdset的所有位*/
FD_SET(int fd,fd_set*fdset);/*设置fdset的位fd*/
FD_CLR(int fd,fd_set*fdset);/*清除fdset的位fd*/
int FD_ISSET(int fd,fd_set*fdset);/*测试fdset的位fd是否被设置*/
~~~

* timeout参数用来设置select函数的超时时间。它是一个timeval结构类型的指针，采用指针参数是因为内核将修改它以告诉应用程序select等待了多久。不过我们不能完全信任select调用返回后的timeout值，比如调用失败时timeout值是不确定的。timeval结构体的定义如下：

~~~c
struct timeval{
long tv_sec;/*秒数*/
long tv_usec;/*微秒数*/
};
~~~

由以上定义可见，select给我们提供了一个微秒级的定时方式。如果给timeout变量的tv_sec成员和tv_usec成员都传递0，则select将立即返回。如果给timeout传递NULL，则select将一直阻塞，直到某个文件描述符就绪。select成功时返回就绪（可读、可写和异常）文件描述符的总数。如果在超时时间内没有任何文件描述符就绪，select将返回0。select失败时返回-1并设置errno。如果在select等待期间，程序接收到信号，则select立即返回-1，并设置errno为EINTR

#### 文件描述符就绪条件

哪些情况下文件描述符可以被认为是可读、可写或者出现异常，对于select的使用非常关键。

* 下列情况下socket可读：
* * socket内核接收缓存区中的字节数大于或等于其低水位标记SO_RCVLOWAT。此时我们可以无阻塞地读该socket，并且读操作返回的字节数大于0。
  * **socket通信的对方关闭连接**。此时对该socket的读操作将返回0。
  * **监听socket上有新的连接请求**。
  * socket上有未处理的错误。此时我们可以使用getsockopt来读取和清除该错误。
* 下列情况下socket可写：
* * socket内核发送缓存区中的可用字节数大于或等于其低水位标记SO_SNDLOWAT。此时我们可以无阻塞地写该socket，并且写操作返回的字节数大于0。
  * socket的写操作被关闭。对写操作被关闭的socket执行写操作将触发一个SIGPIPE信号。
  * **socket使用非阻塞connect连接成功或者失败（超时）之后**。
  * socket上有未处理的错误。此时我们可以使用getsockopt来读取和清除该错误。

网络程序中，select能处理的异常情况只有一种：socket上接收到带外数据

#### 处理带外数据

socket上接收到普通数据和带外数据都将使select返回，但socket处于不同的就绪状态：前者处于可读状态，后者处于异常状态。代码清单9-1描述了select是如何同时处理二者的。

同时接收普通数据和带外数据

~~~c++
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#include <stdlib.h>
int main(int argc, char *argv[])
{
    if (argc <= 2)
    {
        printf("usage:%s ip_address port_number\n", basename(argv[0]));
        return 1;
    }
    const char *ip = argv[1];
    int port = atoi(argv[2]);
    int ret = 0;
    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);
    int listenfd = socket(PF_INET, SOCK_STREAM, 0);
    assert(listenfd >= 0);
    ret = bind(listenfd, (struct sockaddr *)&address, sizeof(address));
    assert(ret != -1);
    ret = listen(listenfd, 5);
    assert(ret != -1);
    struct sockaddr_in client_address;
    socklen_t client_addrlength = sizeof(client_address);
    int connfd = accept(listenfd, (struct sockaddr *)&client_address, &client_addrlength);
    if (connfd < 0)
    {
        printf("errno is:%d\n", errno);
        close(listenfd);
    }
    char buf[1024];
    fd_set read_fds;//声明文件描述符 
    fd_set exception_fds;
    FD_ZERO(&read_fds);
    FD_ZERO(&exception_fds);//清除所有位
    while (1)
    {
        memset(buf, '\0', sizeof(buf));
        /*每次调用select前都要重新在read_fds和exception_fds中设置文件描述符connfd，因为事件发生之后，文件描述符集合将被内核修改*/
        FD_SET(connfd, &read_fds);//设置位
        FD_SET(connfd, &exception_fds);
        ret = select(connfd + 1, &read_fds, NULL, &exception_fds, NULL);
        if (ret < 0)
        {
            printf("selection failure\n");
            break;
        }
        /*对于可读事件，采用普通的recv函数读取数据*/
        if (FD_ISSET(connfd, &read_fds))//如果位被设置
        {
            ret = recv(connfd, buf, sizeof(buf) - 1, 0);
            if (ret <= 0)
            {
                break;
            }
            printf("get%d bytes of normal data:%s\n", ret, buf);
        }
        /*对于异常事件，采用带MSG_OOB标志的recv函数读取带外数据*/
        else if (FD_ISSET(connfd, &exception_fds))
        {
            ret = recv(connfd, buf, sizeof(buf) - 1, MSG_OOB);
            if (ret <= 0)
            {
                break;
            }
            printf("get%d bytes of oob data:%s\n", ret, buf);
        }
    }
    close(connfd);
    close(listenfd);
    return 0;
}
~~~

### poll系统调用

poll系统调用和select类似，也是在指定时间内轮询一定数量的文件描述符，以测试其中是否有就绪者

~~~c
#include＜poll.h＞
int poll(struct pollfd*fds,nfds_t nfds,int timeout);
~~~

* fds参数是一个pollfd结构类型的数组，它指定所有我们感兴趣的文件描述符上发生的可读、可写和异常等事件。pollfd结构体的定义如下：

~~~c
struct pollfd{
	int fd;/*文件描述符*/
	short events;/*注册的事件*/ //告诉poll监听哪些事件
	short revents;/*实际发生的事件，由内核填充*/
};
~~~

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240903181311707.png" alt="image-20240903181311707" style="zoom:50%;" />

使用POLLRDHUP事件时，我们需要在代码最开始处定义_GNU_SOURCE

* nfds参数指定被监听事件集合fds的大小。其类型nfds_t的定义

typedef unsigned long int nfds_t;

* timeout参数指定poll的超时值，单位是毫秒。当timeout为-1时，poll调用将永远阻塞，直到某个事件发生；当timeout为0时，poll调用将立即返回。
* 返回值与select返回值含义相同 如果在超时时间内没有任何文件描述符就绪，返回0。失败时返回-1并设置errno。如果在等待期间，程序接收到信号，则立即返回-1，并设置errno为EINTR

### epoll系列系统调用

#### 内核事件表

epoll是Linux特有的I/O复用函数。它在实现和使用上与select、poll有很大差异。首先，epoll使用一组函数来完成任务，而不是单个函数。其次，epoll把用户关心的文件描述符上的事件放在内核里的一个事件表中，从而无须像select和poll那样每次调用都要重复传入文件描述符集或事件集。但epoll需要使用一个额外的文件描述符，来唯一标识内核中的这个事件表。这个文件描述符使用如下epoll_create函数来创建：

~~~c
#include＜sys/epoll.h＞
int epoll_create(int size)
~~~

size参数现在并不起作用，只是给内核一个提示，告诉它事件表需要多大。该函数返回的文件描述符将用作其他所有epoll系统调用的第一个参数，以指定要访问的内核事件表。

下面的函数用来操作epoll的内核事件表：

int epoll_ctl(int epfd,int op,int fd,struct epoll_event*event)

* fd参数是要操作的文件描述符

* op参数则指定操作类型。操作类型有如下3种：

* * EPOLL_CTL_ADD，往事件表中注册fd上的事件
  * EPOLL_CTL_MOD，修改fd上的注册事件
  * EPOLL_CTL_DEL，删除fd上的注册事件

* event参数指定事件，它是epoll_event结构指针类型。epoll_event的定义如下

* * ~~~c
    struct epoll_event{
    __uint32_t events;/*epoll事件*/
    epoll_data_t data;/*用户数据*/
    };
    ~~~

  * 其中events成员描述事件类型。epoll支持的事件类型和poll基本相同。表示epoll事件类型的宏是在poll对应的宏前加上“E”，比如epoll的数据可读事件是EPOLLIN。但epoll有两个额外的事件类型——EPOLLET和EPOLLONESHOT。它们对于epoll的高效运作非常关键

  * data成员用于存储用户数据，其类型epoll_data_t的定义如下：

  * * ~~~c
      typedef union epoll_data{
      void*ptr;
      int fd;
      uint32_t u32;
      uint64_t u64;
      }epoll_data_t;
      ~~~

    * epoll_data_t是一个联合体，其4个成员中使用最多的是fd，它指定事件所从属的目标文件描述符。ptr成员可用来指定与fd相关的用户数据。但由于epoll_data_t是一个联合体，我们不能同时使用其ptr成员和fd成员，因此，如果要将文件描述符和用户数据关联起来（正如8.5.2小节讨论的将句柄和事件处理器绑定一样），以实现快速的数据访问，只能使用其他手段，比如放弃使用epoll_data_t的fd成员，而在ptr指向的用户数据中包含fd。

    * epoll_ctl成功时返回0，失败则返回-1并设置errno。

####epoll_wait函数

epoll系列系统调用，它在一段超时时间内等待一组文件描述符上的事件

~~~c
#include＜sys/epoll.h＞
int epoll_wait(int epfd,struct epoll_event*events,int maxevents,int timeout);
~~~

* 该函数成功时返回就绪的文件描述符的个数，失败时返回-1并设置errno。

* timeout参数的含义与poll接口的timeout参数相同。
* maxevents参数指定最多监听多少个事件，它必须大于0。

epoll_wait函数如果检测到事件，就将所有就绪的事件从内核事件表（由epfd参数指定）中复制到它的第二个参数events指向的数组中。这个数组只用于输出epoll_wait检测到的就绪事件，而不像select和poll的数组参数那样既用于传入用户注册的事件，又用于输出内核检测到的就绪事件。这就极大地提高了应用程序索引就绪文件描述符的效率。

poll和epoll在使用上的差别

~~~c++
/*如何索引poll返回的就绪文件描述符*/
int ret=poll(fds,MAX_EVENT_NUMBER,-1);
/*必须遍历所有已注册文件描述符并找到其中的就绪者（当然，可以利用ret来稍做优化）*/
for(int i=0;i＜MAX_EVENT_NUMBER;++i){
	if(fds[i].revents＆POLLIN){  /*判断第i个文件描述符是否就绪*/
		int sockfd=fds[i].fd;
	/*处理sockfd*/
	}
}
/*如何索引epoll返回的就绪文件描述符*/
int ret=epoll_wait(epollfd,events,MAX_EVENT_NUMBER,-1);
/*仅遍历就绪的ret个文件描述符*/
for(int i=0;i＜ret;i++){
	int sockfd=events[i].data.fd;
/*sockfd肯定就绪，直接处理*/
}
~~~

#### LT和ET模式

epoll对文件描述符的操作有两种模式：LT（Level Trigger，电平触发）模式和ET（Edge Trigger，边沿触发）模式。LT模式是默认的工作模式，这种模式下epoll相当于一个效率较高的poll。当往epoll内核事件表中注册一个文件描述符上的EPOLLET事件时，epoll将以ET模式来操作该文件描述符。ET模式是epoll的高效工作模式。

* 对于采用LT工作模式的文件描述符，当epoll_wait检测到其上有事件发生并将此事件通知应用程序后，应用程序可以不立即处理该事件。这样，当应用程序下一次调用epoll_wait时，epoll_wait还会再次向应用程序通告此事件，直到该事件被处理。

* 对于采用ET工作模式的文件描述符，当epoll_wait检测到其上有事件发生并将此事件通知应用程序后，应用程序必须立即处理该事件，因为后续的epoll_wait调用将不再向应用程序通知这一事件。可见，ET模式在很大程度上降低了同一个epoll事件被重复触发的次数，因此效率要比LT模式高。

LT和ET模式

~~~c++
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#include <stdlib.h>
#include <sys/epoll.h>
#include <pthread.h>
#define MAX_EVENT_NUMBER 1024
#define BUFFER_SIZE 10
/*将文件描述符设置成非阻塞的*/
int setnonblocking(int fd)
{
    int old_option = fcntl(fd, F_GETFL);
    int new_option = old_option | O_NONBLOCK;
    fcntl(fd, F_SETFL, new_option);
    return old_option;
}
/*将文件描述符fd上的EPOLLIN注册到epollfd指示的epoll内核事件表中，参数
enable_et指定是否对fd启用ET模式*/
void addfd(int epollfd, int fd, bool enable_et)
{
    epoll_event event;
    event.data.fd = fd;
    event.events = EPOLLIN;
    if (enable_et)
    {
        event.events |= EPOLLET;
    }
    epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event);
    setnonblocking(fd);
}
/*LT模式的工作流程*/
void lt(epoll_event *events, int number, int epollfd, int listenfd)
{
    char buf[BUFFER_SIZE];
    for (int i = 0; i < number; i++)
    {
        int sockfd = events[i].data.fd;
        if (sockfd == listenfd)
        {
            struct sockaddr_in client_address;
            socklen_t client_addrlength = sizeof(client_address);
            int connfd = accept(listenfd, (struct sockaddr *)&client_address,
                                &client_addrlength);
            addfd(epollfd, connfd, false); /*对connfd禁用ET模式*/
        }
        else if (events[i].events & EPOLLIN)
        {
            /*只要socket读缓存中还有未读出的数据，这段代码就被触发*/
            printf("event trigger once\n");
            memset(buf, '\0', BUFFER_SIZE);
            int ret = recv(sockfd, buf, BUFFER_SIZE - 1, 0);
            if (ret <= 0)
            {
                close(sockfd);
                continue;
            }
            printf("get%d bytes of content:%s\n", ret, buf);
        }
        else
        {
            printf("something else happened\n");
        }
    }
}
/*ET模式的工作流程*/
void et(epoll_event *events, int number, int epollfd, int listenfd)
{
    char buf[BUFFER_SIZE];
    for (int i = 0; i < number; i++)
    {
        int sockfd = events[i].data.fd;
        if (sockfd == listenfd)
        {
            struct sockaddr_in client_address;
            socklen_t client_addrlength = sizeof(client_address);
            int connfd = accept(listenfd, (struct sockaddr *)&client_address, &client_addrlength);
            addfd(epollfd, connfd, true); /*对connfd开启ET模式*/
        }
        else if (events[i].events & EPOLLIN)
        {
            /*这段代码不会被重复触发，所以我们循环读取数据，以确保把socket读缓存中的所
            有数据读出*/
            printf("event trigger once\n");
            while (1)
            {
                memset(buf, '\0', BUFFER_SIZE);
                int ret = recv(sockfd, buf, BUFFER_SIZE - 1, 0);
                if (ret < 0)
                {
                    /*对于非阻塞IO，下面的条件成立表示数据已经全部读取完毕。此后，epoll就能再次
                    触发sockfd上的EPOLLIN事件，以驱动下一次读操作*/
                    if ((errno == EAGAIN) || (errno == EWOULDBLOCK))
                    {
                        printf("read later\n");
                        break;
                    }
                    close(sockfd);
                    break;
                }
                else if (ret == 0)
                {
                    close(sockfd);
                }
                else
                {
                    printf("get%d bytes of content:%s\n", ret, buf);
                }
            }
        }
        else
        {
            printf("something else happened\n");
        }
    }
}
int main(int argc, char *argv[])
{
    if (argc <= 2)
    {
        printf("usage:%s ip_address port_number\n", basename(argv[0]));
        return 1;
    }
    const char *ip = argv[1];
    int port = atoi(argv[2]);
    int ret = 0;
    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);
    int listenfd = socket(PF_INET, SOCK_STREAM, 0);
    assert(listenfd >= 0);
    ret = bind(listenfd, (struct sockaddr *)&address, sizeof(address));
    assert(ret != -1);
    ret = listen(listenfd, 5);
    assert(ret != -1);
    epoll_event events[MAX_EVENT_NUMBER];
    int epollfd = epoll_create(5);
    assert(epollfd != -1);
    addfd(epollfd, listenfd, true);
    while (1)
    {
        int ret = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
        if (ret < 0)
        {
            printf("epoll failure\n");
            break;
        }
        lt(events, ret, epollfd, listenfd); /*使用LT模式*/
        // et(events,ret,epollfd,listenfd);/*使用ET模式*/
    }
    close(listenfd);
    return 0;
}
~~~

运行一下这段代码，然后telnet到这个服务器程序上并一次传输超过10字节（BUFFER_SIZE的大小）的数据，然后比较LT模式和ET模式的异同。你会发现，正如我们预期的，ET模式下事件被触发的次数要比LT模式下少很多

每个使用ET模式的文件描述符都应该是非阻塞的。如果文件描述符是阻塞的，那么读或写操作将会因为没有后续的事件而一直处于阻塞状态（饥渴状态）。

#### EPOLLONESHOT事件

即使我们使用ET模式，一个socket上的某个事件还是可能被触发多次。这在并发程序中就会引起一个问题。比如一个线程（或进程）在读取完某个socket上的数据后开始处理这些数据，而在数据的处理过程中该socket上又有新数据可读（EPOLLIN再次被触发），此时另外一个线程被唤醒来读取这些新的数据。于是就出现了两个线程同时操作一个socket的局面。这当然不是我们期望的。我们期望的是一个socket连接在任一时刻都只被一个线程处理。这一点可以使用epoll的EPOLLONESHOT事件实现。

对于注册了EPOLLONESHOT事件的文件描述符，操作系统最多触发其上注册的一个可读、可写或者异常事件，且只触发一次，除非我们使用epoll_ctl函数重置该文件描述符上注册的EPOLLONESHOT事件。这样，当一个线程在处理某个socket时，其他线程是不可能有机会操作该socket的。但反过来思考，注册了EPOLLONESHOT事件的socket一旦被某个线程处理完毕，该线程就应该立即重置这个socket上的EPOLLONESHOT事件

~~~c++

#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#include <stdlib.h>
#include <sys/epoll.h>
#include <pthread.h>
#define MAX_EVENT_NUMBER 1024
#define BUFFER_SIZE 1024
struct fds
{
    int epollfd;
    int sockfd;
};
int setnonblocking(int fd)
{
    int old_option = fcntl(fd, F_GETFL);
    int new_option = old_option | O_NONBLOCK;
    fcntl(fd, F_SETFL, new_option);
    return old_option;
}
/*将fd上的EPOLLIN和EPOLLET事件注册到epollfd指示的epoll内核事件表中，参
数oneshot指定是否注册fd上的EPOLLONESHOT事件*/
void addfd(int epollfd, int fd, bool oneshot)
{
    epoll_event event;
    event.data.fd = fd;
    event.events = EPOLLIN | EPOLLET;
    if (oneshot)
    {
        event.events |= EPOLLONESHOT;
    }
    epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event);
    setnonblocking(fd);
}
/*重置fd上的事件。这样操作之后，尽管fd上的EPOLLONESHOT事件被注册，但是操
作系统仍然会触发fd上的EPOLLIN事件，且只触发一次*/
void reset_oneshot(int epollfd, int fd)
{
    epoll_event event;
    event.data.fd = fd;
    event.events = EPOLLIN | EPOLLET | EPOLLONESHOT;
    epoll_ctl(epollfd, EPOLL_CTL_MOD, fd, &event);
}
/*工作线程*/
void *worker(void *arg)
{
    int sockfd = ((fds *)arg)->sockfd;
    int epollfd = ((fds *)arg)->epollfd;
    printf("start new thread to receive data on fd:%d\n", sockfd);
    char buf[BUFFER_SIZE];
    memset(buf, '\0', BUFFER_SIZE);
    /*循环读取sockfd上的数据，直到遇到EAGAIN错误*/
    while (1)
    {
        int ret = recv(sockfd, buf, BUFFER_SIZE - 1, 0);
        if (ret == 0)
        {
            close(sockfd);
            printf("foreiner closed the connection\n");
            break;
        }
        else if (ret < 0)
        {
            if (errno == EAGAIN)
            {
                reset_oneshot(epollfd, sockfd);
                printf("read later\n");
                break;
            }
        }
        else
        {
            printf("get content:%s\n", buf);
            /*休眠5s，模拟数据处理过程*/
            sleep(5);
        }
    }
    printf("end thread receiving data on fd:%d\n", sockfd);
}
int main(int argc, char *argv[])
{
    if (argc <= 2)
    {
        printf("usage:%s ip_address port_number\n", basename(argv[0]));
        return 1;
    }
    const char *ip = argv[1];
    int port = atoi(argv[2]);
    int ret = 0;
    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);
    int listenfd = socket(PF_INET, SOCK_STREAM, 0);
    assert(listenfd >= 0);
    ret = bind(listenfd, (struct sockaddr *)&address, sizeof(address));
    assert(ret != -1);
    ret = listen(listenfd, 5);
    assert(ret != -1);
    epoll_event events[MAX_EVENT_NUMBER];
    int epollfd = epoll_create(5);
    assert(epollfd != -1);
    /*注意，监听socket listenfd上是不能注册EPOLLONESHOT事件的，否则应用程序
    只能处理一个客户连接！因为后续的客户连接请求将不再触发listenfd上的EPOLLIN事件
    */
    addfd(epollfd, listenfd, false);
    while (1)
    {
        int ret = epoll_wait(epollfd, events, MAX_EVENT_NUMBER, -1);
        if (ret < 0)
        {
            printf("epoll failure\n");
            break;
        }
        for (int i = 0; i < ret; i++)
        {
            int sockfd = events[i].data.fd;
            if (sockfd == listenfd)
            {
                struct sockaddr_in client_address;
                socklen_t client_addrlength = sizeof(client_address);
                int connfd = accept(listenfd, (struct sockaddr *)&client_address, &client_addrlength);
                /*对每个非监听文件描述符都注册EPOLLONESHOT事件*/
                addfd(epollfd, connfd, true);
            }
            else if (events[i].events & EPOLLIN)
            {
                pthread_t thread;
                fds fds_for_new_worker;
                fds_for_new_worker.epollfd = epollfd;
                fds_for_new_worker.sockfd = sockfd;
                /*新启动一个工作线程为sockfd服务*/
                pthread_create(&thread, NULL, worker, (void *)&fds_for_new_worker);
            }
            else
            {
                printf("something else happened\n");
            }
        }
    }
    close(listenfd);
    return 0;
}
~~~

从工作线程函数worker来看，如果一个工作线程处理完某个socket上的一次请求（我们用休眠5 s来模拟这个过程）之后，又接收到该socket上新的客户请求，则该线程将继续为这个socket服务。并且因为该socket上注册了EPOLLONESHOT事件，其他线程没有机会接触这个socket，如果工作线程等待5 s后仍然没收到该socket上的下一批客户数据，则它将放弃为该socket服务。同时，它调用reset_oneshot函数来重置该socket上的注册事件，这将使epoll有机会再次检测到该socket上的EPOLLIN事件，进而使得其他线程有机会为该socket服务。

由此看来，尽管一个socket在不同时间可能被不同的线程处理，但同一时刻肯定只有一个线程在为它服务。这就保证了连接的完整性，从而避免了很多可能的竞态条件。

### 三组I/O复用函数的比较

*直接看最后*

这3组系统调用都能同时监听多个文件描述符。它们将等待由timeout参数指定的超时时间，直到一个或者多个文件描述符上有事件发生时返回，返回值是就绪的文件描述符的数量。返回0表示没有事件发生。现在我们从事件集、最大支持文件描述符数、工作模式和具体实现等四个方面

这3组函数都通过某种结构体变量来告诉内核监听哪些文件描述符上的哪些事件，并使用该结构体类型的参数来获取内核处理的结果。

* select的参数类型fd_set没有将文件描述符和事件绑定，它仅仅是一个文件描述符集合，因此select需要提供3个这种类型的参数来分别传入和输出可读、可写及异常等事件。内核对fd_set集合的在线修改，应用程序下次调用select前不得不重置这3个fd_set集合。
* poll的参数类型pollfd则多少“聪明”一些。它把文件描述符和事件都定义其中，任何事件都被统一处理，从而使得编程接口简洁得多。并且内核每次修改的是pollfd结构体的revents成员，而events成员保持不变，因此下次调用poll时应用程序无须重置pollfd类型的事件集参数。

由于每次select和poll调用都返回整个用户注册的事件集合（其中包括就绪的和未就绪的），所以应用程序索引就绪文件描述符的时间复杂度为O（n）。

* epoll则采用与select和poll完全不同的方式来管理用户注册的事件。它在内核中维护一个事件表，并提供了一个独立的系统调用epoll_ctl来控制往其中添加、删除、修改事件。这样，每次epoll_wait调用都直接从该内核事件表中取得用户注册的事件，而无须反复从用户空间读入这些事件。epoll_wait系统调用的events参数仅用来返回就绪的事件，这使得应用程序索引就绪文件描述符的时间复杂度达到O（1）。

poll和epoll_wait分别用nfds和maxevents参数指定最多监听多少个文件描述符和事件。这两个数值都能达到系统允许打开的最大文件描述符数目，即65 535（cat/proc/sys/fs/file-max）。而select允许监听的最大文件描述符数量通常有限制。虽然用户可以修改这个限制，但这可能导致不可预期的后果

select和poll都只能工作在相对低效的LT模式，而epoll则可以工作在ET高效模式。并且epoll还支持EPOLLONESHOT事件。该事件能进一步减少可读、可写和异常等事件被触发的次数。

select和poll采用的都是轮询的方式，即每次调用都要扫描整个注册文件描述符集合，并将其中就绪的文件描述符返回给用户程序，因此它们检测就绪事件的算法的时间复杂度是O（n）。epoll_wait则不同，它采用的是回调的方式。内核检测到就绪的文件描述符时，将触发回调函数，回调函数就将该文件描述符上对应的事件插入内核就绪事件队列。内核最后在适当的时机将该就绪事件队列中的内容拷贝到用户空间。因此epoll_wait无须轮询整个文件描述符集合来检测哪些事件已经就绪，其算法时间复杂度是O（1）。但是，当活动连接比较多的时候，epoll_wait的效率未必比select和poll高，因为此时回调函数被触发得过于频繁。所以**epoll_wait适用于连接数量多，但活动连接较少的情况。** **其他情况用poll**，select没啥好处

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240903184558006.png" alt="image-20240903184558006" style="zoom:50%;" />

### I/O复用的高级应用一：非阻塞connect

这段话描述了connect出错时的一种errno值：EINPROGRESS。这种错误发生在对非阻塞的socket调用connect，而连接又没有立即建立时。根据man文档的解释，在这种情况下，我们可以调用select、poll等函数来监听这个连接失败的socket上的可写事件。当select、poll等函数返回后，再利用getsockopt来读取错误码并清除该socket上的错误。如果错误码是0，表示连接成功建立，否则连接失败。

通过上面描述的非阻塞connect方式，我们就能同时发起多个连接并一起等待

~~~c++
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <assert.h>
#include <stdio.h>
#include <time.h>
#include <errno.h>
#include <fcntl.h>
#include <sys/ioctl.h>
#include <unistd.h>
#include <string.h>
#define BUFFER_SIZE 1023
int setnonblocking(int fd)
{
    int old_option = fcntl(fd, F_GETFL);
    int new_option = old_option | O_NONBLOCK;
    fcntl(fd, F_SETFL, new_option);
    return old_option;
}
/*超时连接函数，参数分别是服务器IP地址、端口号和超时时间（毫秒）。函数成功时
返回已经处于连接状态的socket，失败则返回-1*/
int unblock_connect(const char *ip, int port, int time)
{
    int ret = 0;
    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);
    int sockfd = socket(PF_INET, SOCK_STREAM, 0);
    int fdopt = setnonblocking(sockfd);
    ret = connect(sockfd, (struct sockaddr *)&address, sizeof(address));
    if (ret == 0)
    {
        /*如果连接成功，则恢复sockfd的属性，并立即返回之*/
        printf("connect with server immediately\n");
        fcntl(sockfd, F_SETFL, fdopt);
        return sockfd;
    }
    else if (errno != EINPROGRESS)
    {
        /*如果连接没有立即建立，那么只有当errno是EINPROGRESS时才表示连接还在进
        行，否则出错返回*/
        printf("unblock connect not support\n");
        return -1;
    }
    fd_set readfds;
    fd_set writefds;
    struct timeval timeout;
    FD_ZERO(&readfds);
    FD_SET(sockfd, &writefds);
    timeout.tv_sec = time;
    timeout.tv_usec = 0;
    ret = select(sockfd + 1, NULL, &writefds, NULL, &timeout);
    if (ret <= 0)
    {
        /*select超时或者出错，立即返回*/
        printf("connection time out\n");
        close(sockfd);
        return -1;
    }
    if (!FD_ISSET(sockfd, &writefds))
    {
        printf("no events on sockfd found\n");
        close(sockfd);
        return -1;
    }
    int error = 0;
    socklen_t length = sizeof(error);
    /*调用getsockopt来获取并清除sockfd上的错误*/
    if (getsockopt(sockfd, SOL_SOCKET, SO_ERROR, &error, &length) < 0)
    {
        printf("get socket option failed\n");
        close(sockfd);
        return -1;
    }
    /*错误号不为0表示连接出错*/
    if (error != 0)
    {
        printf("connection failed after select with the error:%d\n", error);
        close(sockfd);
        return -1;
    }
    /*连接成功*/
    printf("connection ready after select with the socket:%d\n", sockfd);
    fcntl(sockfd, F_SETFL, fdopt);
    return sockfd;
}
int main(int argc, char *argv[])
{
    if (argc <= 2)
    {
        printf("usage:%s ip_address port_number\n", basename(argv[0]));
        return 1;
    }
    const char *ip = argv[1];
    int port = atoi(argv[2]);
    int sockfd = unblock_connect(ip, port, 10);
    if (sockfd < 0)
    {
        return 1;
    }
    close(sockfd);
    return 0;
}
~~~

但遗憾的是，这种方法存在几处移植性问题。首先，非阻塞的socket可能导致connect始终失败。其次，select对处于EINPROGRESS状态下的socket可能不起作用。最后，对于出错的socket，getsockopt在有些系统（比如Linux）上返回-1（正如代码清单9-5所期望的），而在有些系统（比如源自伯克利的UNIX）上则返回0。这些问题没有一个统一的解决方法

### I/O复用的高级应用二：聊天室程序

像ssh这样的登录服务通常要同时处理网络连接和用户输入，这也可以使用I/O复用来实现。

我们以poll为例实现一个简单的聊天室程序，以阐述如何使用I/O复用技术来同时处理网络连接和用户输入。该聊天室程序能让所有用户同时在线群聊，它分为客户端和服务器两个部分。

其中客户端程序有两个功能：一是从标准输入终端读入用户数据，并将用户数据发送至服务器；二是往标准输出终端打印服务器发送给它的数据。

服务器的功能是接收客户数据，并把客户数据发送给每一个登录到该服务器上的客户端（数据发送者除外）。

#### 客户端

客户端程序使用poll同时监听用户输入和网络连接，并利用splice函数将用户输入内容直接定向到网络连接上以发送之，从而实现数据零拷贝，提高了程序执行效率。

~~~c++
#define _GNU_SOURCE 1 //为了使用POLLRDHUP事件
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <poll.h>
#include <fcntl.h>
#define BUFFER_SIZE 64
int main(int argc, char *argv[])
{
    //socket地址 创建 连接
    if (argc <= 2){
        printf("usage:%s ip_address port_number\n", basename(argv[0]));
        return 1;
    }
    const char *ip = argv[1];
    int port = atoi(argv[2]);
    struct sockaddr_in server_address;
    bzero(&server_address, sizeof(server_address));
    server_address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &server_address.sin_addr);
    server_address.sin_port = htons(port);
    int sockfd = socket(PF_INET, SOCK_STREAM, 0);
    assert(sockfd >= 0);
    if (connect(sockfd, (struct sockaddr *)&server_address, sizeof(server_address)) < 0)
    {
        printf("connection failed\n");
        close(sockfd);
        return 1;
    }
    
    pollfd fds[2];//poll轮询函数的第一个参数/*注册文件描述符0（标准输入）和文件描述符sockfd上的可读事件*/
    fds[0].fd = 0;//文件描述符
    fds[0].events = POLLIN;//监听事件：数据可读
    fds[0].revents = 0;//实际事件，内核填充
    fds[1].fd = sockfd;//文件描述符为 连接socket
    fds[1].events = POLLIN | POLLRDHUP; //poll监听：数据可读 或 tcp被对方关闭或对方关闭了写操作
    fds[1].revents = 0;
    char read_buf[BUFFER_SIZE];
    int pipefd[2];
    int ret = pipe(pipefd);//pipefd创建文件描述符
    assert(ret != -1);
    while (1)
    {                           //第二个参数是被监听事件集合fds的大小
        ret = poll(fds, 2, -1);//轮询文件描述符，看是否有就绪者  -1：永远阻塞。直到监听的事件发生
        if (ret < 0)//失败返回-1
        {
            printf("poll failure\n");
            break;
        }
        if (fds[1].revents & POLLRDHUP)
        {
            printf("server close the connection\n");
            break;
        }
        else if (fds[1].revents & POLLIN)
        {
            memset(read_buf, '\0', BUFFER_SIZE);
            recv(fds[1].fd, read_buf, BUFFER_SIZE - 1, 0);
            printf("%s\n", read_buf);
        }
        if (fds[0].revents & POLLIN)//？？？为啥不是POLLOUT
        {
            /*使用splice将用户输入的数据直接写到sockfd上（零拷贝）*/
            ret = splice(0, NULL, pipefd[1], NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE);
            ret = splice(pipefd[0], NULL, sockfd, NULL, 32768, SPLICE_F_MORE | SPLICE_F_MOVE);
        }
    }
    close(sockfd);
    return 0;
}
~~~

哪里写数据？？？

#### 服务器

使用poll同时管理监听socket和连接socket，并且使用牺牲空间换取时间的策略来提高服务器性能

~~~c++
#define _GNU_SOURCE 1
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#include <stdlib.h>
#include <poll.h>
#define USER_LIMIT 5   /*最大用户数量*/
#define BUFFER_SIZE 64 /*读缓冲区的大小*/
#define FD_LIMIT 65535 /*文件描述符数量限制*/
/*客户数据：客户端socket地址、待写到客户端的数据的位置、从客户端读入的数据*/
struct client_data
{
    sockaddr_in address;
    char *write_buf;
    char buf[BUFFER_SIZE];
};
int setnonblocking(int fd)
{
    int old_option = fcntl(fd, F_GETFL);
    int new_option = old_option | O_NONBLOCK;
    fcntl(fd, F_SETFL, new_option);
    return old_option;
}
int main(int argc, char *argv[])
{
    if (argc <= 2)
    {
        printf("usage:%s ip_address port_number\n", basename(argv[0]));
        return 1;
    }
    const char *ip = argv[1];
    int port = atoi(argv[2]);
    int ret = 0;
    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);
    int listenfd = socket(PF_INET, SOCK_STREAM, 0);
    assert(listenfd >= 0);
    ret = bind(listenfd, (struct sockaddr *)&address, sizeof(address));
    assert(ret != -1);
    ret = listen(listenfd, 5);
    assert(ret != -1);
    /*创建users数组，分配FD_LIMIT个client_data对象。可以预期：每个可能的
    socket连接都可以获得一个这样的对象，并且socket的值可以直接用来索引（作为数组的
    下标）socket连接对应的client_data对象，这是将socket和客户数据关联的简单而高
    效的方式*/
    client_data *users = new client_data[FD_LIMIT];
    /*尽管我们分配了足够多的client_data对象，但为了提高poll的性能，仍然有必要限制用户的数量*/
    pollfd fds[USER_LIMIT + 1];
    int user_counter = 0;
    for (int i = 1; i <= USER_LIMIT; ++i)
    {
        fds[i].fd = -1;
        fds[i].events = 0;
    }
    fds[0].fd = listenfd;
    fds[0].events = POLLIN | POLLERR;
    fds[0].revents = 0;
    while (1)
    {
        ret = poll(fds, user_counter + 1, -1);
        if (ret < 0)
        {
            printf("poll failure\n");
            break;
        }
        for (int i = 0; i < user_counter + 1; ++i)
        {
            if ((fds[i].fd == listenfd) && (fds[i].revents & POLLIN))
            {
                struct sockaddr_in client_address;
                socklen_t client_addrlength = sizeof(client_address);
                int connfd = accept(listenfd, (struct sockaddr *)&client_address, &client_addrlength);
                if (connfd < 0)
                {
                    printf("errno is:%d\n", errno);
                    continue;
                }
                /*如果请求太多，则关闭新到的连接*/
                if (user_counter >= USER_LIMIT)
                {
                    const char *info = "too many users\n";
                    printf("%s", info);
                    send(connfd, info, strlen(info), 0);
                    close(connfd);
                    continue;
                }
                /*对于新的连接，同时修改fds和users数组。前文已经提到，users[connfd]对应
                于新连接文件描述符connfd的客户数据*/
                user_counter++;
                users[connfd].address = client_address;
                setnonblocking(connfd);
                fds[user_counter].fd = connfd;
                fds[user_counter].events = POLLIN | POLLRDHUP | POLLERR;
                fds[user_counter].revents = 0;
                printf("comes a new user,now have%d users\n", user_counter);
            }
            else if (fds[i].revents & POLLERR)
            {
                printf("get an error from%d\n", fds[i].fd);
                char errors[100];
                memset(errors, '\0', 100);
                socklen_t length = sizeof(errors);
                if (getsockopt(fds[i].fd, SOL_SOCKET, SO_ERROR, &errors,
                               &length) < 0)
                {
                    printf("get socket option failed\n");
                }
                continue;
            }
            else if (fds[i].revents & POLLRDHUP)
            {
                /*如果客户端关闭连接，则服务器也关闭对应的连接，并将用户总数减1*/
                users[fds[i].fd] = users[fds[user_counter].fd];
                close(fds[i].fd);
                fds[i] = fds[user_counter];
                i--;
                user_counter--;
                printf("a client left\n");
            }
            else if (fds[i].revents & POLLIN)
            {
                int connfd = fds[i].fd;
                memset(users[connfd].buf, '\0', BUFFER_SIZE);
                ret = recv(connfd, users[connfd].buf, BUFFER_SIZE - 1, 0);
                printf("get%d bytes of client data%sfrom%d\n", ret, users[connfd].buf, connfd);
                if (ret < 0)
                {
                    /*如果读操作出错，则关闭连接*/
                    if (errno != EAGAIN)
                    {
                        close(connfd);
                        users[fds[i].fd] = users[fds[user_counter].fd];
                        fds[i] = fds[user_counter];
                        i--;
                        user_counter--;
                    }
                }
                else if (ret == 0)
                {
                }
                else
                {
                    /*如果接收到客户数据，则通知其他socket连接准备写数据*/
                    for (int j = 1; j <= user_counter; ++j)
                    {
                        if (fds[j].fd == connfd)
                        {
                            continue;
                        }
                        fds[j].events |= ~POLLIN;
                        fds[j].events |= POLLOUT;
                        users[fds[j].fd].write_buf = users[connfd].buf;
                    }
                }
            }
            else if (fds[i].revents & POLLOUT)
            {
                int connfd = fds[i].fd;
                if (!users[connfd].write_buf)
                {
                    continue;
                }
                ret = send(connfd, users[connfd].write_buf, strlen(users[connfd].write_buf), 0);
                users[connfd].write_buf = NULL;
                /*写完数据后需要重新注册fds[i]上的可读事件*/
                fds[i].events |= ~POLLOUT;
                fds[i].events |= POLLIN;
            }
        }
    }
    delete[] users;
    close(listenfd);
    return 0;
}
~~~

###I/O复用的高级应用三：同时处理TCP和UDP服务

### 超级服务xinetd

## 信号

信号是由用户、系统或者进程发送给目标进程的信息，以通知目标进程某个状态的改变或系统异常

* 对于前台进程，比如输入Ctrl+C通常会给进程发送一个中断信号
* 系统异常。比如浮点异常和非法内存段访问。
* 系统状态变化。比如alarm定时器到期将引起SIGALRM信号。
* 运行kill命令或调用kill函数。

服务器程序必须处理（或至少忽略）一些常见的信号，以免异常终止

### Linux信号概述

#### 发送信号

Linux下，一个进程给其他进程发送信号的API是kill函数。

##定时器

比如定期检测一个客户连接的活动状态。我们要将每个定时事件分别封装成定时器，并使用某种容器类数据结构，比如链表、排序链表和时间轮，将所有定时器串联起来，以实现对定时事件的统一管理。

定时是指在一段时间之后触发某段代码的机制，我们可以在这段代码中依次处理所有到期的定时器

linux三种定时方法

* socket选项SO_RCVTIMEO和SO_SNDTIMEO
* SIGALRM信号
* I/O复用系统调用的超时参数

###socket选项SO_RCVTIMEO和SO_SNDTIMEO

分别设置socket接收数据超时时间和发送数据超时时间

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240903200850739.png" alt="image-20240903200850739" style="zoom:50%;" />

我们可以根据系统调用（send、sendmsg、recv、recvmsg、accept和connect）的返回值以及errno来判断超时时间是否已到，进而决定是否开始处理定时任务。

connect为例，说明程序中如何使用SO_SNDTIMEO选项来定时

~~~c++
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <assert.h>
#include <stdio.h>
#include <errno.h>
#include <fcntl.h>
#include <unistd.h>
#include <string.h>
/*超时连接函数*/
int timeout_connect(const char *ip, int port, int time)
{
    int ret = 0;
    struct sockaddr_in address;
    bzero(&address, sizeof(address));
    address.sin_family = AF_INET;
    inet_pton(AF_INET, ip, &address.sin_addr);
    address.sin_port = htons(port);
    int sockfd = socket(PF_INET, SOCK_STREAM, 0);
    assert(sockfd >= 0);
    /*通过选项SO_RCVTIMEO和SO_SNDTIMEO所设置的超时时间的类型是timeval，这和
    select系统调用的超时参数类型相同*/
    struct timeval timeout;
    timeout.tv_sec = time;
    timeout.tv_usec = 0;
    socklen_t len = sizeof(timeout);
    ret = setsockopt(sockfd, SOL_SOCKET, SO_SNDTIMEO, &timeout, len);
    assert(ret != -1);
    ret = connect(sockfd, (struct sockaddr *)&address, sizeof(address));
    if (ret == -1)
    {
        /*超时对应的错误号是EINPROGRESS。下面这个条件如果成立，我们就可以处理定时任
        务了*/
        if (errno == EINPROGRESS)
        {
            printf("connecting timeout,process timeout logic\n");
            return -1;
        }
        printf("error occur when connecting to server\n");
        return -1;
    }
    return sockfd;
}
int main(int argc, char *argv[])
{
    if (argc <= 2)
    {
        printf("usage:%s ip_address port_number\n", basename(argv[0]));
        return 1;
    }
    const char *ip = argv[1];
    int port = atoi(argv[2]);
    int sockfd = timeout_connect(ip, port, 10);
    if (sockfd < 0)
    {
        return 1;
    }
    return 0;
}
~~~

### SIGALRM信号

由alarm和setitimer函数设置的实时闹钟一旦超时，将触发SIGALRM信号。因此，我们可以利用该信号的信号处理函数来处理定时任务。但是，如果要处理多个定时任务，我们就需要不断地触发SIGALRM信号，并在其信号处理函数中执行到期的任务。一般而言，SIGALRM信号按照固定的频率生成，即由alarm或setitimer函数设置的定时周期T保持不变。如果某个定时任务的超时时间不是T的整数倍，那么它实际被执行的时间和预期的时间将略有偏差。因此定时周期T反映了定时的精度。

本节中我们通过一个实例——处理非活动连接，来介绍如何使用SIGALRM信号定时。不过，我们需要先给出一种简单的定时器实现——基于升序链表的定时器，并把它应用到处理非活动连接这个实例中。这样，我们才能观察到SIGALRM信号处理函数是如何处理定时器并执行定时任务的。此外，我们介绍这种定时器也是为了和后面要讨论的高效定时器——时间轮和时间堆做对比。

#### 基于升序链表的定时器

定时器通常至少要包含两个成员：一个超时时间（相对时间或者绝对时间）和一个任务回调函数。有的时候还可能包含回调函数被执行时需要传入的参数，以及是否重启定时器等信息。

升序定时器链表将其中的定时器按照超时时间做升序排序。

代码。。。

为了便于阅读，我们将实现包含在头文件中。sort_timer_lst是一个升序链表。其核心函数tick相当于一个心搏函数，它每隔一段固定的时间就执行一次，以检测并处理到期的任务。判断定时任务到期的依据是定时器的expire值小于当前的系统时间。从执行效率来看，添加定时器的时间复杂度是O(n)，删除定时器的时间复杂度是O(1)，执行定时任务的时间复杂度是O(1)。

#### 处理非活动连接

升序定时器链表的实际应用-定期处理非活动连接

给客户端发一个重连请求，或者关闭该连接，或者其他。Linux在内核中提供了对连接是否处于活动状态的定期检查机制，我们可以通过socket选项KEEPALIVE来激活它。不过使用这种方式将使得应用程序对连接的管理变得复杂。因此，我们可以考虑在应用层实现类似于KEEPALIVE的机制，以管理所有长时间处于非活动状态的连接。比如，代码清单11-3利用alarm函数周期性地触发SIGALRM信号，该信号的信号处理函数利用管道通知主循环执行定时器链表上的定时任务——关闭非活动的连接。

代码。。。

### I/O复用系统调用的超时参数

Linux下的3组I/O复用系统调用都带有超时参数，因此它们不仅能统一处理信号和I/O事件，也能统一处理定时事件。但是由于I/O复用系统调用可能在超时时间到期之前就返回（有I/O事件发生），所以如果我们要利用它们来定时，就需要不断更新定时参数以反映剩余的时间

代码。。。

### 高性能定时器

#### 时间轮

基于排序链表的定时器存在一个问题：添加定时器的效率偏低

#### 时间堆

前面讨论的定时方案都是以固定的频率调用心搏函数tick，并在其中依次检测到期的定时器，然后执行到期定时器上的回调函数。设计定时器的另外一种思路是：将所有定时器中超时时间最小的一个定时器的超时值作为心搏间隔。这样，一旦心搏函数tick被调用，超时时间最小的定时器必然到期，我们就可以在tick函数中处理该定时器。然后，再次从剩余的定时器中找出超时时间最小的一个，并将这段最小时间设置为下一次心搏间隔。如此反复，就实现了较为精确的定时

最小堆很适合处理这种定时方案。最小堆是指每个节点的值都小于或等于其子节点的值的完全二叉树。图11-2给出了一个具有6个元素的最小堆。

## 高性能I/O框架库Libevent

前面我们利用三章的篇幅较为细致地讨论了Linux服务器程序必须处理的三类事件：I/O事件、信号和定时事件。在处理这三类事件时我们通常需要考虑如下三个问题：

* 统一事件源。很明显，统一处理这三类事件既能使代码简单易懂，又能避免一些潜在的逻辑错误。前面我们已经讨论了实现统一事件源的一般方法——利用I/O复用系统调用来管理所有事件

* 可移植性。不同的操作系统具有不同的I/O复用方式，比如Solaris的dev/poll文件，FreeBSD的kqueue机制，Linux的epoll系列系统调用。
* 对并发编程的支持。在多进程和多线程环境下，我们需要考虑各执行实体如何协同处理客户连接、信号和定时器，以避免竞态条件

所幸的是，开源社区提供了诸多优秀的I/O框架库。它们不仅解决了上述问题，让开发者可以将精力完全放在程序的逻辑上，而且稳定性、性能等各方面都相当出色。比如ACE、ASIO和Libevent。本章将介绍其中相对轻量级的Libevent框架库。

## 多进程编程

* 复制进程映像的fork系统调用和替换进程映像的exec系列系统调用
* 僵尸进程以及如何避免僵尸进程
* 进程间通信（Inter-Process Communication，IPC）最简单的方式：管道。
* 3种System V进程间通信方式：信号量、消息队列和共享内存。它们都是由AT＆T System V2版本的UNIX引入的，所以统称为System V IPC
* 在进程间传递文件描述符的通用方法：通过UNIX本地域socket传递特殊的辅助数据

###fork系统调用

创建新进程

## 多线程编程

（请忽略）早期Linux不支持线程，直到1996年，Xavier Leroy等人才开发出第一个基本符合POSIX标准的线程库LinuxThreads。但LinuxThreads效率低而且问题很多。自内核2.6开始，Linux才真正提供内核级的线程支持，并有两个组织致力于编写新的线程库：NGPT（Next Generation POSIX Threads）和NPTL（Native POSIX Thread Library）。不过前者在2003年就放弃了，因此新的线程库就称为NPTL。NPTL比LinuxThreads效率高，且更符合POSIX规范，所以它已经成为glibc的一部分。本书所有线程相关的例程使用的线程库都是NPTL。

本章要讨论的线程相关的内容都属于POSIX线程（简称pthread）标准，而不局限于NPTL实现，具体包括：

* 创建线程和结束线程
* 读取和设置线程属性
* POSIX线程同步方式：POSIX信号量、互斥锁和条件变量

### Linux线程概述

#### 线程模型

线程是程序中完成一个独立任务的完整执行序列，即一个可调度的实体。根据运行环境和调度者的身份，线程可分为内核线程和用户线程。内核线程，在有的系统上也称为LWP（Light Weight Process，轻量级进程），运行在内核空间，由内核来调度；用户线程运行在用户空间，由线程库来调度。当进程的一个内核线程获得CPU的使用权时，它就加载并运行一个用户线程。可见，内核线程相当于用户线程运行的“容器”。一个进程可以拥有M个内核线程和N个用户线程，其中M≤N。并且在一个系统的所有进程中，M和N的比值都是固定的。按照M:N的取值，线程的实现方式可分为三种模式：完全在用户空间实现、完全由内核调度和双层调度（two level scheduler）。

* 完全在用户空间实现的线程无须内核的支持，内核甚至根本不知道这些线程的存在。线程库负责管理所有执行线程，比如线程的优先级、时间片等。线程库利用longjmp来切换线程的执行，使它们看起来像是“并发”执行的。但实际上内核仍然是把整个进程作为最小单位来调度的。换句话说，一个进程的所有执行线程共享该进程的时间片，它们对外表现出相同的优先级。因此，对这种实现方式而言，N=1，即M个用户空间线程对应1个内核线程，而该内核线程实际上就是进程本身。完全在用户空间实现的线程的优点是：创建和调度线程都无须内核的干预，因此速度相当快。并且由于它不占用额外的内核资源，所以即使一个进程创建了很多线程，也不会对系统性能造成明显的影响。其缺点是：对于多处理器系统，一个进程的多个线程无法运行在不同的CPU上，因为内核是按照其最小调度单位来分配CPU的。此外，线程的优先级只对同一个进程中的线程有效，比较不同进程中的线程的优先级没有意义。早期的伯克利UNIX线程就是采用这种方式实现的
* 完全由内核调度的模式将创建、调度线程的任务都交给了内核，运行在用户空间的线程库无须执行管理任务，这与完全在用户空间实现的线程恰恰相反。二者的优缺点也正好互换。较早的Linux内核对内核线程的控制能力有限，线程库通常还要提供额外的控制能力，尤其是线程同步机制，不过现代Linux内核已经大大增强了对线程的支持。完全由内核调度的这种线程实现方式满足M:N=1:1，即1个用户空间线程被映射为1个内核线程。
* 双层调度模式是前两种实现模式的混合体：内核调度M个内核线程，线程库调度N个用户线程。这种线程实现方式结合了前两种方式的优点：不但不会消耗过多的内核资源，而且线程切换速度也较快，同时它可以充分利用多处理器的优势

#### Linux线程库

Linux上两个最有名的线程库是LinuxThreads和NPTL，它们都是采用1:1的方式实现的。由于LinuxThreads在开发的时候，Linux内核对线程的支持还非常有限，所以其可用性、稳定性以及POSIX兼容性都远远不及NPTL。现代Linux上默认使用的线程库是NPTL。用户可以使用如下命令来查看当前系统上所使用的线程库：

$getconf GNU_LIBPTHREAD_VERSION

NPTL 2.28

### 创建线程和结束线程

**pthread_create**

~~~c++
#include＜pthread.h＞
int pthread_create(pthread_t* thread,const pthread_attr_t* attr,void*(*start_routine)(void*),void*arg);
~~~

* thread参数是新线程的标识符 类型pthread_t的定义如下：

\#include＜bits/pthreadtypes.h＞

typedef unsigned long int pthread_t;

* attr参数用于设置新线程的属性。给它传递NULL表示使用默认线程属性。线程拥有众多属性，我们将在后面详细讨论之。
* start_routine和arg参数分别指定新线程将运行的函数及其参数
* pthread_create成功时返回0，失败时返回错误码。一个用户可以打开的线程数量不能超过RLIMIT_NPROC软资源限制。此外，系统上所有用户能创建的线程总数也不得超过/proc/sys/kernel/threads-max内核参数所定义的值

**pthread_join**

线程一旦被创建好，内核就可以调度内核线程来执行start_routine函数指针所指向的函数了。线程函数在结束时最好调用如下函数，以确保安全、干净地退出：

~~~c
#include＜pthread.h＞
void pthread_exit(void*retval);
~~~

pthread_exit函数通过retval参数向线程的回收者传递其退出信息。它执行完之后不会返回到调用者，而且永远不会失败。

**pthread_join**

一个进程中的所有线程都可以调用pthread_join函数来回收其他线程（前提是目标线程是可回收的），即等待其他线程结束，这类似于回收进程的wait和waitpid系统调用。pthread_join的定义如下：

~~~c
#include＜pthread.h＞
int pthread_join(pthread_t thread,void**retval);
~~~

thread参数是目标线程的标识符，retval参数则是目标线程返回的退出信息。该函数会一直阻塞，直到被回收的线程结束为止。该函数成功时返回0，失败则返回错误码。

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240903205058260.png" alt="image-20240903205058260" style="zoom:50%;" />

**pthread_cancel**

异常终止一个线程，即取消线程

int pthread_cancel(pthread_t thread);

数成功时返回0，失败则返回错误码。

接收到取消请求的目标线程可以决定是否允许被取消以及如何取消，这分别由如下两个函数完成

~~~c
#include＜pthread.h＞
int pthread_setcancelstate(int state,int*oldstate);
int pthread_setcanceltype(int type,int*oldtype);
~~~

这两个函数的第一个参数分别用于设置线程的取消状态（是否允许取消）和取消类型（如何取消），第二个参数则分别记录线程原来的取消状态和取消类型。

state参数有两个可选值：

❑PTHREAD_CANCEL_ENABLE，允许线程被取消。它是线程被创建时的默认取消状态。

❑PTHREAD_CANCEL_DISABLE，禁止线程被取消。这种情况下，如果一个线程收到取消请求，则它会将请求挂起，直到该线程允许被取消。

type参数也有两个可选值

❑PTHREAD_CANCEL_ASYNCHRONOUS，线程随时都可以被取消。它将使得接收到取消请求的目标线程立即采取行动。



### 线程属性

~~~c
#include＜bits/pthreadtypes.h＞
#define__SIZEOF_PTHREAD_ATTR_T 36
typedef union{
	char__size[__SIZEOF_PTHREAD_ATTR_T];
	long int__align;
}pthread_attr_t;
~~~

各种线程属性全部包含在一个字符数组中。

~~~c
#include＜pthread.h＞
/*初始化线程属性对象*/
int pthread_attr_init(pthread_attr_t*attr);
/*销毁线程属性对象。被销毁的线程属性对象只有再次初始化之后才能继续使用*/
int pthread_attr_destroy(pthread_attr_t*attr);
/*下面这些函数用于获取和设置线程属性对象的某个属性*/
int pthread_attr_getdetachstate(const pthread_attr_t*attr,int*detachstate);
int pthread_attr_setdetachstate(pthread_attr_t* attr,int detachstate);
int pthread_attr_getstackaddr(const pthread_attr_t*attr,void**stackaddr);
int pthread_attr_setstackaddr(pthread_attr_t*attr,void*stackaddr);
int pthread_attr_getstacksize(const pthread_attr_t*attr,size_t*stacksize);
int pthread_attr_setstacksize(pthread_attr_t*attr,size_t stacksize);
int pthread_attr_getstack(const pthread_attr_t*attr,void**stackaddr,size_t*stacksize);
int pthread_attr_setstack(pthread_attr_t*attr,void*stackaddr,size_t stacksize);
int pthread_attr_getguardsize(const pthread_attr_t*__attr,size_t*guardsize);
int pthread_attr_setguardsize(pthread_attr_t*attr,size_t guardsize);
int pthread_attr_getschedparam(const pthread_attr_t*attr,struct sched_param*param);
int pthread_attr_setschedparam(pthread_attr_t*attr,const struct sched_param*param);
int pthread_attr_getschedpolicy(const pthread_attr_t*attr,int*policy);
int pthread_attr_setschedpolicy(pthread_attr_t*attr,int policy);
int pthread_attr_getinheritsched(const pthread_attr_t*attr,int*inherit);
int pthread_attr_setinheritsched(pthread_attr_t*attr,int inherit);
int pthread_attr_getscope(const pthread_attr_t*attr,int*scope);
int pthread_attr_setscope(pthread_attr_t*attr,int scope);
~~~

* detachstate，线程的脱离状态。它有PTHREAD_CREATE_JOINABLE和PTHREAD_CREATE_DETACH两个可选值。前者指定线程是可以被回收的，后者使调用线程脱离与进程中其他线程的同步。脱离了与其他线程同步的线程称为“脱离线程”。脱离线程在退出时将自行释放其占用的系统资源。线程创建时该属性的默认值是PTHREAD_CREATE_JOINABLE。此外，我们也可以使用pthread_detach函数直接将线程设置为脱离线程。
* stackaddr和stacksize，线程堆栈的起始地址和大小
* guardsize，保护区域大小
* schedparam，线程调度参数
* schedpolicy，线程调度策略
* inheritsched，是否继承调用线程的调度属性
* scope，线程间竞争CPU的范围，即线程优先级的有效范围

### POSIX信号量

线程同步的机制：POSIX信号量、互斥量和条件变量

信号量API有两组。一组是讨论过的System VIPC信号量，另外一组是POSIX信号量 原理语义相同

~~~c
#include＜semaphore.h＞
int sem_init(sem_t*sem,int pshared,unsigned int value);
int sem_destroy(sem_t*sem);
int sem_wait(sem_t*sem);
int sem_trywait(sem_t*sem);
int sem_post(sem_t*sem);
~~~

* sem指向被操作的信号量
* sem_init函数用于初始化一个未命名的信号量（POSIX信号量API支持命名信号量）。pshared参数指定信号量的类型。如果其值为0，就表示这个信号量是当前进程的局部信号量，否则该信号量就可以在多个进程之间共享。value参数指定信号量的初始值。此外，初始化一个已经被初始化的信号量将导致不可预期的结果。
* sem_destroy函数用于销毁信号量，以释放其占用的内核资源。如果销毁一个正被其他线程等待的信号量，则将导致不可预期的结果
* sem_wait函数以原子操作的方式将信号量的值减1。如果信号量的值为0，则sem_wait将被阻塞，直到这个信号量具有非0值。
* sem_trywait与sem_wait函数相似，不过它始终立即返回，而不论被操作的信号量是否具有非0值，相当于sem_wait的非阻塞版本。当信号量的值非0时，sem_trywait对信号量执行减1操作。当信号量的值为0时，它将返回-1并设置errno为EAGAIN。
* sem_post函数以原子操作的方式将信号量的值加1。当信号量的值大于0时，其他正在调用sem_wait等待信号量的线程将被唤醒。
* 上面这些函数成功时返回0，失败则返回-1并设置errno。

### 互斥锁

互斥锁（也称互斥量）可以用于保护关键代码段，以确保其独占式的访问，这有点像一个二进制信号量。当进入关键代码段时，我们需要获得互斥锁并将其加锁，这等价于二进制信号量的P操作；当离开关键代码段时，我们需要对互斥锁解锁，以唤醒其他等待该互斥锁的线程，这等价于二进制信号量的V操作。

####互斥锁基础API

~~~c
#include＜pthread.h＞
int pthread_mutex_init(pthread_mutex_t*mutex,const pthread_mutexattr_t*mutexattr);
int pthread_mutex_destroy(pthread_mutex_t*mutex);
int pthread_mutex_lock(pthread_mutex_t*mutex);
int pthread_mutex_trylock(pthread_mutex_t*mutex);
int pthread_mutex_unlock(pthread_mutex_t*mutex);
~~~

* mutex指向要操作的目标互斥锁，互斥锁的类型是pthread_mutex_t结构体。
* pthread_mutex_init函数用于初始化互斥锁。mutexattr参数指定互斥锁的属性。如果将它设置为NULL，则表示使用默认属性。

使用另一种方式初始化一个互斥锁

* pthread_mutex_t mutex=PTHREAD_MUTEX_INITIALIZER; 这个宏把互斥锁的各个字段都初始化为0
* pthread_mutex_destroy函数用于销毁互斥锁，以释放其占用的内核资源。销毁一个已经加锁的互斥锁将导致不可预期的后果
* pthread_mutex_lock函数以原子操作的方式给一个互斥锁加锁。如果目标互斥锁已经被锁上，则pthread_mutex_lock调用将阻塞，直到该互斥锁的占有者将其解锁。
* pthread_mutex_trylock，始终立即返回，相当于pthread_mutex_lock的非阻塞版本。当目标互斥锁未被加锁时，pthread_mutex_trylock对互斥锁执行加锁操作。当互斥锁已经被加锁时，pthread_mutex_trylock将返回错误码EBUSY。
* pthread_mutex_unlock函数以原子操作的方式给一个互斥锁解锁。如果此时有其他线程正在等待这个互斥锁，则这些线程中的某一个将获得它。
* 上面这些函数成功时返回0，失败则返回错误码。

#### 互斥锁属性

pthread_mutexattr_t结构体定义了一套完整的互斥锁属性。线程库提供了一系列函数来操作pthread_mutexattr_t类型的变量，以方便我们获取和设置互斥锁属性。这里我们列出其中一些主要的函数：

~~~c
#include＜pthread.h＞
/*初始化互斥锁属性对象*/
int pthread_mutexattr_init(pthread_mutexattr_t*attr);
/*销毁互斥锁属性对象*/
int pthread_mutexattr_destroy(pthread_mutexattr_t*attr);
/*获取和设置互斥锁的pshared属性*/
int pthread_mutexattr_getpshared(const pthread_mutexattr_t*attr,int*pshared);
int pthread_mutexattr_setpshared(pthread_mutexattr_t*attr,int pshared);
/*获取和设置互斥锁的type属性*/
int pthread_mutexattr_gettype(const pthread_mutexattr_t*attr,int*type);
int pthread_mutexattr_settype(pthread_mutexattr_t*attr,int type);
~~~

两种常用属性:pshared和type。

pshared指定是否允许跨进程共享互斥锁，其可选值有两个

* PTHREAD_PROCESS_SHARED。互斥锁可以被跨进程共享
* PTHREAD_PROCESS_PRIVATE。互斥锁只能被和锁的初始化线程隶属于同一个进程的线程共享

type指定互斥锁的类型

* PTHREAD_MUTEX_NORMAL，普通锁。这是互斥锁默认的类型。当一个线程对一个普通锁加锁以后，其余请求该锁的线程将形成一个等待队列，并在该锁解锁后按优先级获得它。这种锁类型保证了资源分配的公平性。但这种锁也很容易引发问题：一个线程如果对一个已经加锁的普通锁再次加锁，将引发死锁；对一个已经被其他线程加锁的普通锁解锁，或者对一个已经解锁的普通锁再次解锁，将导致不可预期的后果
* PTHREAD_MUTEX_ERRORCHECK，检错锁。一个线程如果对一个已经加锁的检错锁再次加锁，则加锁操作返回EDEADLK。对一个已经被其他线程加锁的检错锁解锁，或者对一个已经解锁的检错锁再次解锁，则解锁操作返回EPERM
* PTHREAD_MUTEX_RECURSIVE，嵌套锁。这种锁允许一个线程在释放锁之前多次对它加锁而不发生死锁。不过其他线程如果要获得这个锁，则当前锁的拥有者必须执行相应次数的解锁操作。对一个已经被其他线程加锁的嵌套锁解锁，或者对一个已经解锁的嵌套锁再次解锁，则解锁操作返回EPERM
* PTHREAD_MUTEX_DEFAULT，默认锁。一个线程如果对一个已经加锁的默认锁再次加锁，或者对一个已经被其他线程加锁的默认锁解锁，或者对一个已经解锁的默认锁再次解锁，将导致不可预期的后果。这种锁在实现的时候可能被映射为上面三种锁之一。

#### 死锁举例

死锁使得一个或多个线程被挂起而无法继续执行，而且这种情况还不容易被发现。前文提到，在一个线程中对一个已经加锁的普通锁再次加锁，将导致死锁。这种情况可能出现在设计得不够仔细的递归函数中。另外，如果两个线程按照不同的顺序来申请两个互斥锁，也容易产生死锁

~~~c++
#include <pthread.h>
#include <unistd.h>
#include <stdio.h>
int a = 0;
int b = 0;
pthread_mutex_t mutex_a;
pthread_mutex_t mutex_b;
void *another(void *arg)
{
    pthread_mutex_lock(&mutex_b);
    printf("in child thread,got mutex b,waiting for mutex a\n");
    sleep(5);
    ++b;
    pthread_mutex_lock(&mutex_a);
    b += a++;
    pthread_mutex_unlock(&mutex_a);
    pthread_mutex_unlock(&mutex_b);
    pthread_exit(NULL);
}
int main()
{
    pthread_t id;
    pthread_mutex_init(&mutex_a, NULL);
    pthread_mutex_init(&mutex_b, NULL);
    pthread_create(&id, NULL, another, NULL);
    pthread_mutex_lock(&mutex_a);
    printf("in parent thread,got mutex a,waiting for mutex b\n");
    sleep(5);
    ++a;
    pthread_mutex_lock(&mutex_b);
    a += b++;
    pthread_mutex_unlock(&mutex_b);
    pthread_mutex_unlock(&mutex_a);
    pthread_join(id, NULL);
    pthread_mutex_destroy(&mutex_a);
    pthread_mutex_destroy(&mutex_b);
    return 0;
}
~~~

主线程试图先占有互斥锁mutex_a，然后操作被该锁保护的变量a，但操作完毕之后，主线程并没有立即释放互斥锁mutex_a，而是又申请互斥锁mutex_b，并在两个互斥锁的保护下，操作变量a和b，最后才一起释放这两个互斥锁；

与此同时，子线程则按照相反的顺序来申请互斥锁mutex_a和mutex_b，并在两个锁的保护下操作变量a和b。

我们用sleep函数来模拟连续两次调用pthread_mutex_lock之间的时间差，以确保代码中的两个线程各自先占有一个互斥锁（主线程占有mutex_a，子线程占有mutex_b），然后等待另外一个互斥锁（主线程等待mutex_b，子线程等待mutex_a）。这样，两个线程就僵持住了，谁都不能继续往下执行，从而形成死锁。

如果代码中不加入sleep函数，则这段代码或许总能成功地运行，从而为程序留下了一个潜在的BUG。

### 条件变量

如果说互斥锁是用于同步线程对共享数据的访问的话，那么条件变量则是用于在线程之间同步共享数据的值。条件变量提供了一种线程间的通知机制：当某个共享数据达到某个值的时候，唤醒等待这个共享数据的线程

~~~c
#include＜pthread.h＞
int pthread_cond_init(pthread_cond_t*cond,const pthread_condattr_t* cond_attr);
int pthread_cond_destroy(pthread_cond_t*cond);
int pthread_cond_broadcast(pthread_cond_t*cond);
int pthread_cond_signal(pthread_cond_t*cond);
int pthread_cond_wait(pthread_cond_t*cond,pthread_mutex_t*mutex);
~~~

这些函数的第一个参数cond指向要操作的目标条件变量，条件变量的类型是pthread_cond_t结构体

* pthread_cond_init函数用于初始化条件变量。cond_attr参数指定条件变量的属性。如果将它设置为NULL，则表示使用默认属性。条件变量的属性不多，而且和互斥锁的属性类型相似，所以我们不再赘述。

我们还可以使用如下方式来初始化一个条件变量：

pthread_cond_t cond=PTHREAD_COND_INITIALIZER;  是把条件变量的各个字段都初始化为0

* pthread_cond_destroy函数用于销毁条件变量，以释放其占用的内核资源。销毁一个正在被等待的条件变量将失败并返回EBUSY。
* pthread_cond_broadcast函数以广播的方式唤醒所有等待目标条件变量的线程。
* pthread_cond_signal函数用于唤醒一个等待目标条件变量的线程。至于哪个线程将被唤醒，则取决于线程的优先级和调度策略。有时候我们可能想唤醒一个指定的线程，但pthread没有对该需求提供解决方法。不过我们可以间接地实现该需求：定义一个能够唯一表示目标线程的全局变量，在唤醒等待条件变量的线程前先设置该变量为目标线程，然后采用广播方式唤醒所有等待条件变量的线程，这些线程被唤醒后都检查该变量以判断被唤醒的是否是自己，如果是就开始执行后续代码，如果不是则返回继续等待。
* pthread_cond_wait函数用于等待目标条件变量。mutex参数是用于保护条件变量的互斥锁，以确保pthread_cond_wait操作的原子性。在调用pthread_cond_wait前，必须确保互斥锁mutex已经加锁，否则将导致不可预期的结果。pthread_cond_wait函数执行时，首先把调用线程放入条件变量的等待队列中，然后将互斥锁mutex解锁。可见，从pthread_cond_wait开始执行到其调用线程被放入条件变量的等待队列之间的这段时间内，pthread_cond_signal和pthread_cond_broadcast等函数不会修改条件变量。换言之，pthread_cond_wait函数不会错过目标条件变量的任何变化。当pthread_cond_wait函数成功返回时，互斥锁mutex将再次被锁上。

上面这些函数成功时返回0，失败则返回错误码。

### 线程同步机制包装类

为了充分复用代码，同时由于后文的需要，我们将前面讨论的3种线程同步机制分别封装成3个类，实现在locker.h文件中

~~~cpp
#ifndef LOCKER_H
#define LOCKER_H
#include <exception>
#include <pthread.h>
#include <semaphore.h>
/*封装信号量的类*/
class sem
{
public:
    /*创建并初始化信号量*/
    sem()
    {
        if (sem_init(&m_sem, 0, 0) != 0)
        {
            /*构造函数没有返回值，可以通过抛出异常来报告错误*/
            throw std::exception();
        }
    }
    /*销毁信号量*/
    ~sem()
    {
        sem_destroy(&m_sem);
    }
    /*等待信号量*/
    bool wait()
    {
        return sem_wait(&m_sem) == 0;
    }
    /*增加信号量*/
    bool post()
    {
        return sem_post(&m_sem) == 0;
    }

private:
    sem_t m_sem;
};
/*封装互斥锁的类*/
class locker
{
public:
    /*创建并初始化互斥锁*/
    locker()
    {
        if (pthread_mutex_init(&m_mutex, NULL) != 0)
        {
            throw std::exception();
        }
    }
    /*销毁互斥锁*/
    ~locker()
    {
        pthread_mutex_destroy(&m_mutex);
    }
    /*获取互斥锁*/
    bool lock()
    {
        return pthread_mutex_lock(&m_mutex) == 0;
    }
    /*释放互斥锁*/
    bool unlock()
    {
        return pthread_mutex_unlock(&m_mutex) == 0;
    }

private:
    pthread_mutex_t m_mutex;
};
/*封装条件变量的类*/
class cond
{
public:
    /*创建并初始化条件变量*/
    cond()
    {
        if (pthread_mutex_init(&m_mutex, NULL) != 0)
        {
            throw std::exception();
        }
        if (pthread_cond_init(&m_cond, NULL) != 0)
        {
            /*构造函数中一旦出现问题，就应该立即释放已经成功分配了的资源*/
            pthread_mutex_destroy(&m_mutex);
            throw std::exception();
        }
    }
    /*销毁条件变量*/
    ~cond()
    {
        pthread_mutex_destroy(&m_mutex);
        pthread_cond_destroy(&m_cond);
    }
    /*等待条件变量*/
    bool wait()
    {
        int ret = 0;
        pthread_mutex_lock(&m_mutex);
        ret = pthread_cond_wait(&m_cond, &m_mutex);
        pthread_mutex_unlock(&m_mutex);
        return ret == 0;
    }
    /*唤醒等待条件变量的线程*/
    bool signal()
    {
        return pthread_cond_signal(&m_cond) == 0;
    }

private:
    pthread_mutex_t m_mutex;
    pthread_cond_t m_cond;
};
#endif
~~~

### 多线程环境

#### 可重入函数

如果一个函数能被多个线程同时调用且不发生竞态条件，则我们称它是线程安全的（thread safe），或者说它是可重入函数。Linux库函数只有一小部分是不可重入的，比如inet_ntoa函数，以及getservbyname和getservbyport函数。这些库函数之所以不可重入，主要是因为其内部使用了静态变量。不过Linux对很多不可重入的库函数提供了对应的可重入版本，这些可重入版本的函数名是在原函数名尾部加上_r。比如，函数localtime对应的可重入函数是localtime_r。在多线程程序中调用库函数，一定要使用其可重入版本，否则可能导致预想不到的结果。

#### 线程和进程

思考这样一个问题：如果一个多线程程序的某个线程调用了fork函数，那么新创建的子进程是否将自动创建和父进程相同数量的线程呢？答案是“否”，正如我们期望的那样。子进程只拥有一个执行线程，它是调用fork的那个线程的完整复制。并且子进程将自动继承父进程中互斥锁（条件变量与之类似）的状态。这就引起了一个问题：子进程可能不清楚从父进程继承而来的互斥锁的具体状态（是加锁状态还是解锁状态）。这个互斥锁可能被加锁了，但并不是由调用fork函数的那个线程锁住的，而是由其他线程锁住的。如果是这种情况，则子进程若再次对该互斥锁执行加锁操作就会导致死锁，如代码清单所示。

~~~cpp
#include <pthread.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <wait.h>
pthread_mutex_t mutex;
/*子线程运行的函数。它首先获得互斥锁mutex，然后暂停5 s，再释放该互斥锁*/
void *another(void *arg)
{
    printf("in child thread,lock the mutex\n");
    pthread_mutex_lock(&mutex);
    sleep(5);
    pthread_mutex_unlock(&mutex);
}
int main()
{
    pthread_mutex_init(&mutex, NULL);
    pthread_t id;
    pthread_create(&id, NULL, another, NULL);
    /*父进程中的主线程暂停1 s，以确保在执行fork操作之前，子线程已经开始运行并获
    得了互斥变量mutex*/
    sleep(1);
    int pid = fork();
    if (pid < 0)
    {
        pthread_join(id, NULL);
        pthread_mutex_destroy(&mutex);
        return 1;
    }
    else if (pid == 0)
    {
        printf("I am in the child,want to get the lock\n");
        /*子进程从父进程继承了互斥锁mutex的状态，该互斥锁处于锁住的状态，这是由父进
        程中的子线程执行pthread_mutex_lock引起的，因此，下面这句加锁操作会一直阻塞，
        尽管从逻辑上来说它是不应该阻塞的*/
        pthread_mutex_lock(&mutex);
        printf("I can not run to here,oop...\n");
        pthread_mutex_unlock(&mutex);
        exit(0);
    }
    else
    {
        wait(NULL);
    }
    pthread_join(id, NULL);
    pthread_mutex_destroy(&mutex);
    return 0;
}
~~~

pthread提供了一个专门的函数pthread_atfork，以确保fork调用后父进程和子进程都拥有一个清楚的锁状态。该函数的定义如下：

~~~c
int pthread_atfork(void(*prepare)(void),void(*parent)(void),void(*child)(void));
~~~

该函数将建立3个fork句柄来帮助我们清理互斥锁的状态。prepare句柄将在fork调用创建出子进程之前被执行。它可以用来锁住所有父进程中的互斥锁。parent句柄则是fork调用创建出子进程之后，而fork返回之前，在父进程中被执行。它的作用是释放所有在prepare句柄中被锁住的互斥锁。child句柄是fork返回之前，在子进程中被执行。和parent句柄一样，child句柄也是用于释放所有在prepare句柄中被锁住的互斥锁。该函数成功时返回0，失败则返回错误码

fork调用前加入代码

~~~c
void prepare(){
	pthread_mutex_lock(＆mutex);
}
void infork(){
	pthread_mutex_unlock(＆mutex);
}
pthread_atfork(prepare,infork,infork);
~~~

#### 线程和信号

每个线程都可以独立地设置信号掩码。我们讨论过设置进程信号掩码的函数sigprocmask，但在多线程环境下我们应该使用如下所示的pthread版本的sigprocmask函数来设置线程信号掩码：

~~~c
#include＜pthread.h＞
#include＜signal.h＞
int pthread_sigmask(int how,const sigset_t*newmask,sigset_t*oldmask);
~~~

该函数的参数的含义与sigprocmask的参数完全相同，因此不再赘述。pthread_sigmask成功时返回0，失败则返回错误码。

由于进程中的所有线程共享该进程的信号，所以线程库将根据线程掩码决定把信号发送给哪个具体的线程。因此，如果我们在每个子线程中都单独设置信号掩码，就很容易导致逻辑错误。此外，所有线程共享信号处理函数。也就是说，当我们在一个线程中设置了某个信号的信号处理函数后，它将覆盖其他线程为同一个信号设置的信号处理函数。这两点都说明，我们应该定义一个专门的线程来处理所有的信号。这可以通过如下两个步骤来实现

* 在主线程创建出其他子线程之前就调用pthread_sigmask来设置好信号掩码，所有新创建的子线程都将自动继承这个信号掩码。这样做之后，实际上所有线程都不会响应被屏蔽的信号了
* 在某个线程中调用如下函数来等待信号并处理之

~~~c
#include＜signal.h＞
int sigwait(const sigset_t*set,int*sig);
~~~

set参数指定需要等待的信号的集合。我们可以简单地将其指定为在第1步中创建的信号掩码，表示在该线程中等待所有被屏蔽的信号。参数sig指向的整数用于存储该函数返回的信号值。sigwait成功时返回0，失败则返回错误码。一旦sigwait正确返回，我们就可以对接收到的信号做处理了。很显然，如果我们使用了sigwait，就不应该再为信号设置信号处理函数了。这是因为当程序接收到信号时，二者中只能有一个起作用。

代码取自pthread_sigmask函数的man手册。它展示了如何通过上述两个步骤实现在一个线程中统一处理所有信号。

~~~c++
#include <pthread.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>
#include <errno.h>
#define handle_error_en(en, msg) \
    do                           \
    {                            \
        errno = en;              \
        perror(msg);             \
        exit(EXIT_FAILURE);      \
    } while (0)
static void *sig_thread(void *arg)
{
    sigset_t *set = (sigset_t *)arg;
    int s, sig;
    for (;;)
    {
        /*第二个步骤，调用sigwait等待信号*/
        s = sigwait(set, &sig);
        if (s != 0)
            handle_error_en(s, "sigwait");
        printf("Signal handling thread got signal%d\n", sig);
    }
}
int main(int argc, char *argv[])
{
    pthread_t thread;
    sigset_t set;
    int s;
    /*第一个步骤，在主线程中设置信号掩码*/
    sigemptyset(&set);
    sigaddset(&set, SIGQUIT);
    sigaddset(&set, SIGUSR1);
    s = pthread_sigmask(SIG_BLOCK, &set, NULL);
    if (s != 0)
        handle_error_en(s, "pthread_sigmask");
    s = pthread_create(&thread, NULL, &sig_thread, (void *)&set);
    if (s != 0)
        handle_error_en(s, "pthread_create");
    pause();
}
~~~

最后，pthread还提供了下面的方法，使得我们可以明确地将一个信号发送给指定的线程

\#include＜signal.h＞

int pthread_kill(pthread_t thread,int sig);

其中，thread参数指定目标线程，sig参数指定待发送的信号。如果sig为0，则pthread_kill不发送信号，但它任然会执行错误检查。我们可以利用这种方式来检测目标线程是否存在。pthread_kill成功时返回0，失败则返回错误码

## 进程池和线程池

在前面的章节中，我们是通过动态创建子进程（或子线程）来实现并发服务器的。这样做有如下缺点

* 动态创建进程（或线程）是比较耗费时间的，这将导致较慢的客户响应。
* 动态创建的子进程（或子线程）通常只用来为一个客户服务（除非我们做特殊的处理），这将导致系统上产生大量的细微进程（或线程）。进程（或线程）间的切换将消耗大量CPU时间
* 动态创建的子进程是当前进程的完整映像。当前进程必须谨慎地管理其分配的文件描述符和堆内存等系统资源，否则子进程可能复制这些资源，从而使系统的可用资源急剧下降，进而影响服务器的性能

### 进程池和线程池概述

进程池和线程池相似，以进程池为例进行介绍。

* **数目**
* * 进程池是由服务器预先创建的一组子进程，这些子进程的数目在3～10个之间（典型情况）。
  * 线程池中的线程数量应该和CPU数量差不多。

进程池中的所有子进程都运行着相同的代码，并具有相同的属性，比如优先级、PGID等。因为进程池在服务器启动之初就创建好了，所以每个子进程都相对“干净”，没有打开不必要的文件描述符（从父进程继承而来），也不会错误地使用大块的堆内存（从父进程复制得到）。

* **任务分配**

* * 主进程使用某种算法来主动选择子进程。最简单、最常用的算法是随机算法和Round Robin（轮流选取）算法，但更好的算法均匀分配任务到工作线程
  * 主进程和所有子进程通过一个共享的工作队列来同步，子进程都睡眠在该工作队列上。当有新的任务到来时，主进程将任务添加到工作队列中。这将唤醒正在等待任务的子进程，只有一个子进程将获得新任务的“接管权”，它可以从工作队列中取出任务并执行之
* **任务通知和数据传递**

任务分配之后，主进程还需要告诉目标子进程有新任务需要处理，并传递必要的数据。

* * 最简单的方法是，在父进程和子进程之间预先建立好一条管道，然后通过该管道来实现所有的进程间通信（当然，要预先定义好一套协议来规范管道的使用）。
  * 在父线程和子线程之间传递数据就要简单得多，因为我们可以把这些数据定义为全局的，那么它们本身就是被所有线程共享的。

### 处理多客户

* **监听socket和连接socket是否都由主进程来统一管理**
* * 半同步/半反应堆模式是由主进程统一管理这两种socket的
  * 高效的半同步/半异步模式，以及领导者/追随者模式，则是由主进程管理所有监听socket，而各个子进程分别管理属于自己的连接socket的
* * 对于前一种情况，主进程接受新的连接以得到连接socket，然后它需要将该socket传递给子进程（对于线程池而言，父线程将socket传递给子线程是很简单的，因为它们可以很容易地共享该socket。但对于进程池而言，我们必须传递该socket）。
  * 后一种情况的灵活性更大一些，因为子进程可以自己调用accept来接受新的连接，这样父进程就无须向子进程传递socket，而只需要简单地通知一声：“我检测到新的连接，你来接受它

**一个客户连接上的所有任务是否始终由一个子进程来处理**

* 如果说客户任务是无状态的，那么我们可以考虑使用不同的子进程来为该客户的不同请求服务
* 如果客户任务是存在上下文关系的，则最好一直用同一个子进程来为之服务，因为我们不得不在各子进程之间传递上下文数据。我们讨论了epoll的EPOLLONESHOT事件，这一事件能够确保一个客户连接在整个生命周期中仅被一个线程处理

### 半同步/半异步进程池实现

为了避免在父、子进程之间传递文件描述符，我们将接受新连接的操作放到子进程中。很显然，对于这种模式而言，一个客户连接上的所有任务始终是由一个子进程来处理的。

~~~c++
// filename:processpool.h
#ifndef PROCESSPOOL_H
#define PROCESSPOOL_H
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <assert.h>
#include <stdio.h>
#include <unistd.h>
#include <errno.h>
#include <string.h>
#include <fcntl.h>
#include <stdlib.h>
#include <sys/epoll.h>
#include <signal.h>
#include <sys/wait.h>
#include <sys/stat.h>
/*描述一个子进程的类，m_pid是目标子进程的PID，m_pipefd是父进程和子进程通
信用的管道*/
class process
{
public:
    process() : m_pid(-1) {}

public:
    pid_t m_pid;
    int m_pipefd[2];
};
/*进程池类，将它定义为模板类是为了代码复用。其模板参数是处理逻辑任务的类*/
template <typename T>
class processpool
{
private:
    /*将构造函数定义为私有的，因此我们只能通过后面的create静态函数来创建
    processpool实例*/
    processpool(int listenfd, int process_number = 8);

public:
    /*单体模式，以保证程序最多创建一个processpool实例，这是程序正确处理信号的
    必要条件*/
    static processpool<T> *create(int listenfd, int process_number = 8)
    {
        if (!m_instance)
        {
            m_instance = new processpool<T>(listenfd, process_number);
        }
        return m_instance;
    }
    ～processpool()
    {
        delete[] m_sub_process;
    }
    /*启动进程池*/
    void run();

private:
    void setup_sig_pipe();
    void run_parent();
    void run_child();

private:
    /*进程池允许的最大子进程数量*/
    static const int MAX_PROCESS_NUMBER = 16;
    /*每个子进程最多能处理的客户数量*/
    static const int USER_PER_PROCESS = 65536;
    /*epoll最多能处理的事件数*/
    static const int MAX_EVENT_NUMBER = 10000;
    /*进程池中的进程总数*/
    int m_process_number;
    /*子进程在池中的序号，从0开始*/
    int m_idx;
    /*每个进程都有一个epoll内核事件表，用m_epollfd标识*/
    int m_epollfd;
    /*监听socket*/
    int m_listenfd;
    /*子进程通过m_stop来决定是否停止运行*/
    int m_stop;
    /*保存所有子进程的描述信息*/
    process *m_sub_process;
    /*进程池静态实例*/
    static processpool<T> *m_instance;
};
template <typename T>
processpool<T> *processpool<T>::m_instance = NULL;
/*用于处理信号的管道，以实现统一事件源。后面称之为信号管道*/
static int sig_pipefd[2];
static int setnonblocking(int fd)
{
    int old_option = fcntl(fd, F_GETFL);
    int new_option = old_option | O_NONBLOCK;
    fcntl(fd, F_SETFL, new_option);
    return old_option;
}
static void addfd(int epollfd, int fd)
{
    epoll_event event;
    event.data.fd = fd;
    event.events = EPOLLIN | EPOLLET;
    epoll_ctl(epollfd, EPOLL_CTL_ADD, fd, &event);
    setnonblocking(fd);
}
/*从epollfd标识的epoll内核事件表中删除fd上的所有注册事件*/
static void removefd(int epollfd, int fd)
{
    epoll_ctl(epollfd, EPOLL_CTL_DEL, fd, 0);
    close(fd);
}
static void sig_handler(int sig)
{
    int save_errno = errno;
    int msg = sig;
    send(sig_pipefd[1], (char *)&msg, 1, 0);
    errno = save_errno;
}
static void addsig(int sig, void(handler)(int), bool restart = true)
{
    struct sigaction sa;
    memset(&sa, '\0', sizeof(sa));
    sa.sa_handler = handler;
    if (restart)
    {
        sa.sa_flags |= SA_RESTART;
    }
    sigfillset(&sa.sa_mask);
    assert(sigaction(sig, &sa, NULL) != -1);
}
/*进程池构造函数。参数listenfd是监听socket，它必须在创建进程池之前被创建，
否则子进程无法直接引用它。参数process_number指定进程池中子进程的数量*/
template <typename T>
processpool<T>::processpool(int listenfd, int process_number)
    : m_listenfd(listenfd), m_process_number(process_number), m_idx(-1),
      m_stop(false)
{
    assert((process_number > 0) && (process_number <
                                    = MAX_PROCESS_NUMBER));
    m_sub_process = new process[process_number];
    assert(m_sub_process);
    /*创建process_number个子进程，并建立它们和父进程之间的管道*/
    for (int i = 0; i < process_number; ++i)
    {
        int
            ret = socketpair(PF_UNIX, SOCK_STREAM, 0, m_sub_process[i].m_pipefd);
        assert(ret == 0);
        m_sub_process[i].m_pid = fork();
        assert(m_sub_process[i].m_pid >= 0);
        if (m_sub_process[i].m_pid > 0)
        {
            close(m_sub_process[i].m_pipefd[1]);
            continue;
        }
        else
        {
            close(m_sub_process[i].m_pipefd[0]);
            m_idx = i;
            break;
        }
    }
}
/*统一事件源*/
template <typename T>
void processpool<T>::setup_sig_pipe()
{
    /*创建epoll事件监听表和信号管道*/
    m_epollfd = epoll_create(5);
    assert(m_epollfd != -1);
    int ret = socketpair(PF_UNIX, SOCK_STREAM, 0, sig_pipefd);
    assert(ret != -1);
    setnonblocking(sig_pipefd[1]);
    addfd(m_epollfd, sig_pipefd[0]);
    /*设置信号处理函数*/
    addsig(SIGCHLD, sig_handler);
    addsig(SIGTERM, sig_handler);
    addsig(SIGINT, sig_handler);
    addsig(SIGPIPE, SIG_IGN);
}
/*父进程中m_idx值为-1，子进程中m_idx值大于等于0，我们据此判断接下来要运行
的是父进程代码还是子进程代码*/
template <typename T>
void processpool<T>::run()
{
    if (m_idx != -1)
    {
        run_child();
        return;
    }
    run_parent();
}
template <typename T>
void processpool<T>::run_child()
{
    setup_sig_pipe();
    /*每个子进程都通过其在进程池中的序号值m_idx找到与父进程通信的管道*/
    int pipefd = m_sub_process[m_idx].m_pipefd[1];
    /*子进程需要监听管道文件描述符pipefd，因为父进程将通过它来通知子进程accept
    新连接*/
    addfd(m_epollfd, pipefd);
    epoll_event events[MAX_EVENT_NUMBER];
    T *users = new T[USER_PER_PROCESS];
    assert(users);
    int number = 0;
    int ret = -1;
    while (!m_stop)
    {
        number = epoll_wait(m_epollfd, events, MAX_EVENT_NUMBER, -1);
        if ((number < 0) && (errno != EINTR))
        {
            printf("epoll failure\n");
            break;
        }
        for (int i = 0; i < number; i++)
        {
            int sockfd = events[i].data.fd;
            if ((sockfd == pipefd) && (events[i].events & EPOLLIN))
            {
                int client = 0;
                /*从父、子进程之间的管道读取数据，并将结果保存在变量client中。如果读取成
                功，则表示有新客户连接到来*/
                ret = recv(sockfd, (char *)&client, sizeof(client), 0);
                if (((ret < 0) && (errno != EAGAIN)) || ret == 0)
                {
                    continue;
                }
                else
                {
                    struct sockaddr_in client_address;
                    socklen_t client_addrlength = sizeof(client_address);
                    int connfd = accept(m_listenfd, (struct sockaddr *)&client_address,
                                        &client_addrlength);
                    if (connfd < 0)
                    {
                        printf("errno is:%d\n", errno);
                        continue;
                    }
                    addfd(m_epollfd, connfd);
                    /*模板类T必须实现init方法，以初始化一个客户连接。我们直接使用connfd来索引
                    逻辑处理对象（T类型的对象），以提高程序效率*/
                    users[connfd].init(m_epollfd, connfd, client_address);
                }
            }
            /*下面处理子进程接收到的信号*/
            else if ((sockfd == sig_pipefd[0]) && (events[i].events & EPOLLIN))
            {
                int sig;
                char signals[1024];
                ret = recv(sig_pipefd[0], signals, sizeof(signals), 0);
                if (ret <= 0)
                {
                    continue;
                }
                else
                {
                    for (int i = 0; i < ret; ++i)
                    {
                        switch (signals[i])
                        {
                        case SIGCHLD:
                        {
                            pid_t pid;
                            int stat;
                            while ((pid = waitpid(-1, &stat, WNOHANG)) > 0)
                            {
                                continue;
                            }
                            break;
                        }
                        case SIGTERM:
                        case SIGINT:
                        {
                            m_stop = true;
                            break;
                        }
                        default:
                        {
                            break;
                        }
                        }
                    }
                }
            }
            /*如果是其他可读数据，那么必然是客户请求到来。调用逻辑处理对象的process方法
            处理之*/
            else if (events[i].events & EPOLLIN)
            {
                users[sockfd].process();
            }
            else
            {
                continue;
            }
        }
    }
    delete[] users;
    users = NULL;
    close(pipefd);
    // close(m_listenfd);这句话注释掉，以提醒读者：应该由m_listenfd的创建者来关闭这个文件描述符（见后文）,
    // 即所谓的对象（比如一个文件描述符，又或者一段堆内存）由哪个函数创建，就应该由哪个函数销毁”*/
    close(m_epollfd);
}
template <typename T>
void processpool<T>::run_parent()
{
    setup_sig_pipe();
    /*父进程监听m_listenfd*/
    addfd(m_epollfd, m_listenfd);
    epoll_event events[MAX_EVENT_NUMBER];
    int sub_process_counter = 0;
    int new_conn = 1;
    int number = 0;
    int ret = -1;
    while (!m_stop)
    {
        number = epoll_wait(m_epollfd, events, MAX_EVENT_NUMBER, -1);
        if ((number < 0) && (errno != EINTR))
        {
            printf("epoll failure\n");
            break;
        }
        for (int i = 0; i < number; i++)
        {
            int sockfd = events[i].data.fd;
            if (sockfd == m_listenfd)
            {
                /*如果有新连接到来，就采用Round Robin方式将其分配给一个子进程处理*/
                int i = sub_process_counter;
                do
                {
                    if (m_sub_process[i].m_pid != -1)
                    {
                        break;
                    }
                    i = (i + 1) % m_process_number;
                } while (i != sub_process_counter);
                if (m_sub_process[i].m_pid == -1)
                {
                    m_stop = true;
                    break;
                }
                sub_process_counter = (i + 1) % m_process_number;
                send(m_sub_process[i].m_pipefd[0],
                     (char *)&new_conn, sizeof(new_conn), 0);
                printf("send request to child%d\n", i);
            }
            /*下面处理父进程接收到的信号*/
            else if ((sockfd == sig_pipefd[0]) && (events[i].events & EPOLLIN))
            {
                int sig;
                char signals[1024];
                ret = recv(sig_pipefd[0], signals, sizeof(signals), 0);
                if (ret <= 0)
                {
                    continue;
                }
                else
                {
                    for (int i = 0; i < ret; ++i)
                    {
                        switch (signals[i])
                        {
                        case SIGCHLD:
                        {
                            pid_t pid;
                            int stat;
                            while ((pid = waitpid(-1, &stat, WNOHANG)) > 0)
                            {
                                for (int i = 0; i < m_process_number; ++i)
                                {
                                    /*如果进程池中第i个子进程退出了，则主进程关闭相应的通信管道，并设置相应的
                                    m_pid为-1，以标记该子进程已经退出*/
                                    if (m_sub_process[i].m_pid == pid)
                                    {
                                        printf("child%d join\n", i);
                                        close(m_sub_process[i].m_pipefd[0]);
                                        m_sub_process[i].m_pid = -1;
                                    }
                                }
                            }
                            /*如果所有子进程都已经退出了，则父进程也退出*/
                            m_stop = true;
                            for (int i = 0; i < m_process_number; ++i)
                            {
                                if (m_sub_process[i].m_pid != -1)
                                {
                                    m_stop = false;
                                }
                            }
                            break;
                        }
                        case SIGTERM:
                        case SIGINT:
                        {
                            /*如果父进程接收到终止信号，那么就杀死所有子进程，并等待它们全部结束。当然，
                            通知子进程结束更好的方法是向父、子进程之间的通信管道发送特殊数据，读者不妨自己实
                            现之*/
                            printf("kill all the clild now\n");
                            for (int i = 0; i < m_process_number; ++i)
                            {
                                int pid = m_sub_process[i].m_pid;
                                if (pid != -1)
                                {
                                    kill(pid, SIGTERM);
                                }
                            }
                            break;
                        }
                        default:
                        {
                            break;
                        }
                        }
                    }
                }
            }
            else
            {
                continue;
            }
        }
    }
    // close(m_listenfd);/*由创建者关闭这个文件描述符（见后文）*/
    close(m_epollfd);
}
#endif
~~~

### 用进程池实现的简单CGI服务器

### 半同步/半反应堆线程池实现

该线程池的通用性要高得多，因为它使用一个工作队列完全解除了主线程和工作线程的耦合关系：主线程往工作队列中插入任务，工作线程通过竞争来取得任务并执行它。不过，如果要将该线程池应用到实际服务器程序中，那么我们必须保证所有客户请求都是无状态的，因为同一个连接上的不同请求可能会由不同的线程处理。



### 用线程池实现的简单Web服务器

我们曾使用有限状态机实现过一个非常简单的解析HTTP请求的服务器。下面我们将利用前面介绍的线程池来重新实现一个并发的Web服务器

