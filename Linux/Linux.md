# 命令

* mkdir 创建目录
* ls -a 查看包括隐藏文件
* touch
* rm -rf *删除

* 防火墙

查看防火墙状态 firewall-cmd --state

关闭防火墙 systemctl stop firewalld

* 查看端口信息

ss -tuln 查看所有端口

- `-t` 表示TCP端口
- `-u` 表示UDP端口
- `-l` 表示仅显示监听状态的端口
- `-n` 表示以数字形式显示地址和端口号，不尝试解析域名、服务名等

* 查看具体某个端口状态

ss -tuln | grep :80

# 系统编程

## shell编程

使用#include <cstdlib>  的system()函数 调用成功返回0

~~~c++
#include <cstdlib>  
#include <iostream>  
using namespace std;
int main() {  
    int result = system("ls -l");  
    if (result == -1) {  
        cerr << "Failed to execute command\n";  
    }else{
        cout<<result<<endl;
    }
    return 0;  
}
~~~

\#include <cstdio>  c的用法FILE结构体类型

~~~c++
#include <cstdio>  
#include <iostream>  
int main() {  
    FILE *fp = popen("ls -l", "r");  
    if (fp == NULL) {  
        std::cerr << "Failed to run command\n";  
        return 1;  
    }  
    char buffer[128];  
    while (fgets(buffer, 128, fp) != NULL) {  
        std::cout << buffer;  
    }  
    pclose(fp);  
    return 0;  
}
~~~

## 文件编程

文件模式

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240828202342232.png" alt="image-20240828202342232" style="zoom:25%;" />

同时读写模式时  先读后写，写会失败 不知道原因？？？

~~~c++
#include <iostream>  
#include <fstream>  
#include <string> 
//写文件
int main() {  
    std::ofstream outFile("example.txt"); // 创建并打开文件用于写入  
    if (!outFile) {  
        std::cerr << "Unable to open file for writing\n";  
        return 1;  
    }  
    outFile << "Hello, fstream!\n"; // 写入字符串到文件  
    outFile.close(); // 关闭文件（虽然当ofstream对象被销毁时会自动关闭，但显式关闭是一个好习惯）  
    return 0;  
}
//读文件
int main() {  
    std::ifstream inFile("example.txt"); // 打开文件用于读取  
    if (!inFile) {  
        std::cerr << "Unable to open file for reading\n";  
        return 1;  
    }  
    std::string line;  
    while (getline(inFile, line)) { // 逐行读取文件  
        std::cout << line << '\n';  
    }  
    inFile.close(); // 关闭文件  
    return 0;  
}
//同时读写
int main() {  
    std::fstream file("example.txt", std::ios::in | std::ios::out | std::ios::app); // 打开文件，用于读写，并追加到文件末尾  
    if (!file) {  
        std::cerr << "Unable to open file\n";  
        return 1;  
    }  
    // 读取文件内容（注意：如果文件很大，这可能不是最佳做法）  
    std::string line;  
    while (getline(file, line)) {  
        std::cout << line << '\n';  
    }  
    // 回到文件末尾以追加内容  
    file.seekp(0, std::ios::end);  
    file << "This is appended text.\n";  
    file.close(); // 关闭文件  
    return 0;  
}
~~~

## 进程编程

* fork()函数来创建新进程 Unix/Linux系统中用于创建进程的唯一方法 POSIX标准的一部分

~~~c++
#include <unistd.h>  
#include <sys/types.h>  
#include <iostream>  
int main() {  
    pid_t pid = fork();  
    if (pid == -1) {  
        // 错误处理  
        std::cerr << "Fork failed" << std::endl;  
        return 1;  
    } else if (pid == 0) {  
        // 子进程代码  
        std::cout << "I am the child process, PID: " << getpid() << std::endl;  
    } else {  
        // 父进程代码  
        std::cout << "I am the parent process, PID: " << getpid() << " Child PID: " << pid << std::endl;  
    }  
    return 0;  
}
~~~



### 进程间通信IPC

进程间通信（IPC）可以通过多种方式实现，包括管道（pipe）、命名管道（FIFO）、消息队列、信号量、共享内存以及套接字（sockets）等。

####管道

父进程首先创建了一个管道，并通过fork()创建了一个子进程。然后，父进程关闭了管道的读端（fd[0]），而子进程关闭了管道的写端（fd[1]）。父进程通过管道的写端向子进程发送了一条消息，而子进程通过管道的读端接收这条消息。

~~~c++
#include <unistd.h>  
#include <sys/wait.h>  
#include <iostream>  
#include <cstring> // 用于memset  
  
int main() {  
    int fd[2];  
    if (pipe(fd) == -1) {  
        std::cerr << "Pipe failed" << std::endl;  
        return 1;  
    }  
    pid_t pid = fork();  
    if (pid == -1) {  
        // fork失败  
        std::cerr << "Fork failed" << std::endl;  
        close(fd[0]);  
        close(fd[1]);  
        return 1;  
    } else if (pid == 0) {  
        // 子进程  
        close(fd[1]); // 关闭写端  
        char buf[100];  
        ssize_t num_read = read(fd[0], buf, sizeof(buf) - 1);  
        if (num_read > 0) {  
            buf[num_read] = '\0'; // 确保字符串以null终止  
            std::cout << "Child received: " << buf << std::endl;  
        }  
        close(fd[0]);  
    } else {  
        // 父进程  
        close(fd[0]); // 关闭读端  
        const char *message = "Hello from parent!";  
        write(fd[1], message, strlen(message) + 1); // 发送消息，包括结尾的null字符  
        close(fd[1]);  
        wait(NULL); // 等待子进程结束  
    }  
    return 0;  
}
~~~

### 进程同步

进程同步通常用于控制多个进程之间的执行顺序，以避免竞争条件和数据不一致。可以使用信号量（semaphores）、互斥锁（mutexes）、条件变量（condition variables）等机制来实现同步。

### 进程的终止

进程的终止可以通过多种方式实现，包括正常退出（通过return或exit()）、异常退出（如接收到信号）、父进程请求终止（如kill()函数）等。

kill(pid, SIGKILL); 放在上述父进程中可以杀死子进程，子进程就收不到消息了

### 进程等待

父进程可以通过wait()或waitpid()函数等待子进程结束。这有助于父进程回收子进程的资源和状态信息。

## 多线程编程

一个进程有多个线程

线程适合高并发和共享资源如 数据库操作，web服务

进程 独立 健壮

