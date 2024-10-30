

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

