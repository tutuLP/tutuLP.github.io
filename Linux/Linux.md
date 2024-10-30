# 命令

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

~~~pwd 显示当前目录
ls -a 查看包括隐藏文件
mkdir 新建文件夹		touch 新建文件
rm 删除		rm -rf 删除文件夹 -r
history 查看历史命令	
mv a b a重命名为b或移动
reboot 重启
~~~

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

* **创建**

* fork()函数来创建新进程 	Unix/Linux系统中用于创建进程的唯一方法 POSIX标准的一部分

* 原型 pid_t fork(void); 

* * 父进程得到子进程的PID 
  * 子进程得到0
  * 失败得到一个负值

* **新进程是原进程的副本**       
* * 子进程将复制父进程的内存空间(包括代码、数据、堆和栈等)、打开的文件描述符、环境变量等资源。
  * 需要小心处理资源释放、文件描述符等问题，以避免资源泄露和竞态条件。

* 子进程与父进程共享代码段，但在进程空间中有自己的数据段和堆栈段。 
* ==pid_t pid = fork();==   父子进程都有pid这个变量，只是他们的值不一样 根据这个值控制他们的操作
* **父子进程关系**：
  - 子进程是父进程的副本，但在很多方面又是独立的。父子进程之间通常是异步执行的，它们的执行顺序不确定。
  - 父子进程会同时执行下去，但操作系统会保证父进程先于子进程结束，以避免僵尸进程的产生。
* **典型应用场景**：
  - 服务器编程：常用于创建子进程处理客户端请求，父进程监听新的连接。
  - 并行编程：通过fork()可以创建多个进程并行执行任务，实现并行计算。

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

###c++ 11 thread

<thread>   pthread听说很难用，我就不管了

g++ test.cpp -lpthread 	在使用std::thread时，编译器需要将pthread库链接到可执行文件中。

####构造函数

* **默认构造函数**

thread() noexcept; 创建一个空的对象，不会抛出异常

* **初始化构造函数**

template <class Fn, class... Args>
explicit thread(Fn&& fn, Args&&... args); 

* * class Fn：这是函数的类型参数，可以是函数指针、成员函数指针、函数对象（如std::function、lambda表达式、或任何重载了operator()的对象）等。
  * class... Args：这是一个可变参数模板，表示Fn函数所需的参数类型列表。...是C++11中引入的参数包扩展语法，允许函数模板接受**任意数量和类型的参数**。

thread线程会调用fn函数并将参数传递给这个函数

* **移动构造函数** 

 thread(thread&& x) noexcept; 移动后x不代表任何执行对象

*std::thread t4(std::move(t3));*

* ~~~c++
  thread t([]() {  
          cout << "Hello from lambda thread!" << endl;  
      });  
      t.join();
  ~~~

*注意：可被joinable的std::thread对象必须在他们销毁之前被主线程join或者将其设置为detached   空对象和被移动的对象不用*

~~~c++
#include <iostream>
#include <utility>
#include <thread>
#include <chrono>
#include <functional>
#include <atomic>
void f1(int n)
{
    for (int i = 0; i < 5; ++i) {
        std::cout << "Thread " << n << " executing\n";
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
}
void f2(int& n)
{
    for (int i = 0; i < 5; ++i) {
        std::cout << "Thread 2 executing\n";
        ++n;
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
}
int main()
{
    int n = 0;
    std::thread t1; // t1 is not a thread
    std::thread t2(f1, n + 1); // pass by value
    std::thread t3(f2, std::ref(n)); // pass by reference
    std::thread t4(std::move(t3)); // t4 is now running f2(). t3 is no longer a thread
    t2.join();
    t4.join();
    std::cout << "Final value of n is " << n << '\n';
}
~~~

**thread执行带有引用参数的函数** 

thread传递参数以右值传递

如何传递左值？

`std::ref` 可以包装按引用传递的值。
`std::cref` 可以包装按const引用传递的值。

#### 赋值

thread& operator=(thread&& rhs) noexcept;

当前对象不可 joinable，需要传递一个右值引用(rhs)给 move 赋值操作；如果当前对象可被 joinable，则会调用 terminate() 报错。

~~~c++
#include <stdio.h>
#include <stdlib.h>
#include <chrono>    // std::chrono::seconds
#include <iostream>  // std::cout
#include <thread>    // std::thread, std::this_thread::sleep_for
void thread_task(int n) {
    std::this_thread::sleep_for(std::chrono::seconds(n));
    std::cout << "hello thread "
        << std::this_thread::get_id()
        << " paused " << n << " seconds" << std::endl;
}
int main(int argc, const char *argv[])
{
    std::thread threads[5];
    std::cout << "Spawning 5 threads...\n";
    for (int i = 0; i < 5; i++) {
        threads[i] = std::thread(thread_task, i + 1);
    }
    std::cout << "Done spawning threads! Now wait for them to join\n";
    for (auto& t: threads) {
        t.join();
    }
    std::cout << "All threads joined.\n";
    return EXIT_SUCCESS;
}
~~~

#### 其他成员函数

* join

阻塞当前线程，直到由 *this 所标示的线程执行完毕 join 才返回

* get_id 线程ID

~~~
  std::thread t1(foo);
  std::thread::id t1_id = t1.get_id();
~~~

* joinable  检查是否能被join 

t.joinable() 能返回1 不能返回0

*默认构造的线程不能被join*

*某个线程 已经执行完任务，但是没有被 join 的话，该线程依然会被认为是一个活动的执行线程，因此也是可以被 join 的*？？？

* detach

将当前线程对象所代表的执行实例与该线程对象分离，使得线程的执行可以单独进行。一旦线程执行完毕，它所分配的资源将会被释放

失去对这个线程的仍和控制权，也无法访问到

~~~c++
#include <iostream>
#include <chrono>
#include <thread>
void independentThread() 
{
    std::cout << "Starting concurrent thread.\n";
    std::this_thread::sleep_for(std::chrono::seconds(2));
    std::cout << "Exiting concurrent thread.\n";
}
void threadCaller() 
{
    std::cout << "Starting thread caller.\n";
    std::thread t(independentThread);
    t.detach();
    std::this_thread::sleep_for(std::chrono::seconds(1));
    std::cout << "Exiting thread caller.\n";
}
int main() 
{
    threadCaller();
    std::this_thread::sleep_for(std::chrono::seconds(5));
}
~~~

上面这段函数如果没有t.detach(); 当threadCaller函数结束之后会销毁t，但是这个线程还在运行，所以要等待运行完成使用t.join()

* swap 交换两个线程的底层句柄

std::swap(t1, t2);
t1.swap(t2);

* native_handle 返回 native handle 线程句柄，这个句柄允许直接访问和操作 操作系统底层的线程对象

thread是包装过的，这里返回后可以使用怕pthread库中的函数，如改变线程优先级，设置线程属性等

~~~c++
#include <thread>
#include <iostream>
#include <chrono>
#include <cstring>
#include <pthread.h>
std::mutex iomutex;
void f(int num){
  std::this_thread::sleep_for(std::chrono::seconds(1));

 sched_param sch;
 int policy; 
 pthread_getschedparam(pthread_self(), &policy, &sch);
 std::lock_guard<std::mutex> lk(iomutex);
 std::cout << "Thread " << num << " is executing at priority "
           << sch.sched_priority << '\n';
}
int main(){
  std::thread t1(f, 1), t2(f, 2);
  sched_param sch;
  int policy; 
  pthread_getschedparam(t1.native_handle(), &policy, &sch);
  sch.sched_priority = 20;
  if(pthread_setschedparam(t1.native_handle(), SCHED_FIFO, &sch)) {
      std::cout << "Failed to setschedparam: " << std::strerror(errno) << '\n';
  }
  t1.join();
  t2.join();
}
~~~

* std::this_thread 命名空间中相关辅助函数

**get_id**: 获取线程 ID。

~~~c++
#include <iostream>
#include <thread>
#include <chrono>
#include <mutex>
std::mutex g_display_mutex;
void foo(){
  std::thread::id this_id = std::this_thread::get_id();
  g_display_mutex.lock();
  std::cout << "thread " << this_id << " sleeping...\n";
  g_display_mutex.unlock();
  std::this_thread::sleep_for(std::chrono::seconds(1));
}
int main(){
  std::thread t1(foo);
  std::thread t2(foo);
  t1.join();
  t2.join();
}
~~~

**yield**: 当前线程放弃执行，操作系统调度另一线程继续执行。

~~~c++
#include <iostream>
#include <chrono>
#include <thread>
// "busy sleep" while suggesting that other threads run 
// for a small amount of time
void little_sleep(std::chrono::microseconds us){
  auto start = std::chrono::high_resolution_clock::now();
  auto end = start + us;
  do {
      std::this_thread::yield();
  } while (std::chrono::high_resolution_clock::now() < end);
}
int main(){
  auto start = std::chrono::high_resolution_clock::now();
  little_sleep(std::chrono::microseconds(100));
  auto elapsed = std::chrono::high_resolution_clock::now() - start;
  std::cout << "waited for "
            << std::chrono::duration_cast<std::chrono::microseconds>(elapsed).count()
            << " microseconds\n";
}
~~~

**sleep_until**: 线程休眠至某个指定的时刻(time point)，该线程才被重新唤醒。

template< class Clock, class Duration >
void sleep_until( const std::chrono::time_point<Clock,Duration>& sleep_time );

**sleep_for**: 线程休眠某个指定的时间片(time span)，该线程才被重新唤醒，不过由于线程调度等原因，实际休眠时间可能比 sleep_duration 所表示的时间片更长。

~~~c++
#include <iostream>
#include <chrono>
#include <thread>
int main(){
  std::cout << "Hello waiter" << std::endl;
  std::chrono::milliseconds dura( 2000 );
  std::this_thread::sleep_for( dura );
  std::cout << "Waited 2000 ms\n";
}
~~~

####多个线程操作同一变量

`std::atomic`和`std::mutex`

~~~c++
#include <iostream>
#include <thread>
using namespace std;
int n = 0;
void count10000() {
	for (int i = 1; i <= 10000; i++)
		n++;
}
int main() {
	thread th[100];
	// 这里偷了一下懒，用了c++11的foreach结构
	for (thread &x : th)
		x = thread(count10000);
	for (thread &x : th)
		x.join();
	cout << n << endl;
	return 0;
}
~~~

这里我输出989816，理论上应该是1000000，因为多个线程同时访问n

一个线程将mutex锁住时，其它的线程就不能操作mutex，直到这个线程将mutex解锁

~~~c++
#include <iostream>
#include <thread>
#include <mutex>
using namespace std;
int n = 0;
mutex mtx;
void count10000() {
	for (int i = 1; i <= 10000; i++) {
		mtx.lock();
		n++;
		mtx.unlock();
	}
}
int main() {
	thread th[100];
	for (thread &x : th)
		x = thread(count10000);
	for (thread &x : th)
		x.join();
	cout << n << endl;
	return 0;
}
~~~

mutex还有一个函数bool try_lock() 尝试上锁，没有被锁则锁返回true，否则false

mutex固然可以解决问题，但是会浪费很多时间

std::atomic

只需要把int n=0改为atomic_int n = 0;或atomic<int> n = 0;

*原子操作是最小的且不可并行化的操作。*

###std::async

thread可以快速、方便地创建线程，但在async面前，就是小巫见大巫了。
async可以根据情况选择同步执行或创建新线程来异步执行，当然也可以手动选择。对于async的返回值操作也比thread更加方便。

async是一个函数

<future>

~~~c++
#include <iostream>
#include <thread>
#include <future>
using namespace std;
int main() {
	async(launch::async, [](const char *message){
		cout << message << flush;
	}, "Hello, ");
	cout << "World!" << endl;
	return 0;
}
~~~

*  template <class Fn, class… Args>
   future<typename result_of<Fn(Args…)>::type>
   async (Fn&& fn, Args&&… args) 或 async (launch policy, Fn&& fn, Args&&… args); 同样地，传递引用参数需要`std::ref`或`std::cref`
* std::launch有2个枚举值和1个特殊值

枚举值：launch::async	                  0x1（1）	异步启动
枚举值：launch::deferred	               0x2（2）	在调用future::get、future::wait时同步启动（std::future见后文）
特殊值：launch::async | launch::defereed	0x3（3）	同步或异步，根据操作系统而定

####使用std::future获取线程的返回值

~~~c++
#include <iostream>
#include <future> // std::async std::future
using namespace std;
template<class ... Args> decltype(auto) sum(Args&&... args) {
	// C++17折叠表达式
	// "0 +"避免空参数包错误
	return (0 + ... + args);
}
int main() {
	// 注：这里不能只写函数名sum，必须带模板参数
	future<int> val = async(launch::async, sum<int, int, int>, 1, 10, 100);
	// future::get() 阻塞等待线程结束并获得返回值
	cout << val.get() << endl;
	return 0;
}
~~~

linux下报错，win下可以运行

我们定义了一个函数sum，它可以计算多个数字的和，之后我们又定义了一个对象val，它的类型是std::future<int>，这里的int代表这个函数的返回值是int类型。在创建线程后，我们使用了future::get()来阻塞等待线程结束并获取其返回值。至于sum函数中的折叠表达式（fold expression），不是我们这篇文章的重点。

* 成员函数

* * 一般：T get()   阻塞等待线程结束并获取返回值
    当类型为引用：R& future<R&>::get()
    当类型为void：void future::get()       若类型为void，则与future::wait()相同。只能调用一次。
* void wait() const 阻塞等待线程结束
* template <class Rep, class Period>	   future_status wait_for(const chrono::duration<Rep,Period>& rel_time) const;  等待rel_time

若在这段时间内线程结束则返回`future_status::ready`若没结束则返回`future_status::timeout`
若async是以`launch::deferred`启动的，则**不会阻塞**并立即返回`future_status::deferred`

~~~~c++
#include <iostream>
#include <future>
using namespace std;
void count_big_number() {
	// C++14标准中，可以在数字中间加上单引号 ' 来分隔数字，使其可读性更强
	for (int i = 0; i <= 10'0000'0000; i++);
}
int main() {
	future<void> fut = async(launch::async, count_big_number);
	cout << "Please wait" << flush;
	// 每次等待1秒
	while (fut.wait_for(chrono::seconds(1)) != future_status::ready)
		cout << '.' << flush;
	cout << endl << "Finished!" << endl;
	return 0;
}
~~~~

搞懂这个就知道软件的加载画面怎么实现的了

### std::promise

c++ 11 <future>

获得thread的返回值  在一个线程中通过promise获取另一个线程的值

* 引用传递返回值

promise实际上是std::future的一个包装，在讲解future时，我们并没有牵扯到改变future值的问题，但是如果使用thread以引用传递返回值的话，就必须要改变future的值，那么该怎么办呢？
实际上，future的值不能被改变，但你可以通过promise来创建一个拥有特定值的future。

*future的值不能改变，promise的值可以改变。*



~~~c++
#include <iostream>
#include <thread>
#include <future> // std::promise std::future
using namespace std;
template<class ... Args> decltype(auto) sum(Args&&... args) {
	return (0 + ... + args);
}
template<class ... Args> void sum_thread(promise<long long> &val, Args&&... args) {
	val.set_value(sum(args...));//通过调用promise对象的set_value()或set_exception()方法来设置值或异常
}
int main() {
	promise<long long> sum_value;
	thread get_sum(sum_thread<int, int, int>, ref(sum_value), 1, 10, 100);
	cout << sum_value.get_future().get() << endl;//通过调用promise对象的get_future()方法来获取一个future 对象
	get_sum.join();    //调用future对象的get()方法来获取设置的值。如果promise设置了异常，get()方法将重新抛出该异常。
	return 0;
}
//111
~~~

- 使用 `std::promise` 和 `std::future` 时，应确保在调用 `get()` 方法之前，`set_value()` 或 `set_exception()` 已经被调用，否则 `get()` 方法将阻塞等待值的到来。

###std::this_thread

c++11 <thread>的一个命名空间 实现线程对自己的控制

* 函数

| std::thread::id get_id() noexcept                            | 获取当前线程id                           |
| ------------------------------------------------------------ | ---------------------------------------- |
| template<class Rep, class Period><br/>void sleep_for( const std::chrono::duration<Rep, Period>& sleep_duration ) | 等待sleep_duration 一段时间              |
| void yield() noexcept                                        | 暂时放弃线程的执行，将主动权交给其他线程 |

~~~c++
#include <iostream>
#include <thread>
#include <atomic>
using namespace std;
atomic_bool ready = 0;
// uintmax_t ==> unsigned long long
void sleep(uintmax_t ms) {
	this_thread::sleep_for(chrono::milliseconds(ms));
}
void count() {
	while (!ready) this_thread::yield();
	for (int i = 0; i <= 20'0000'0000; i++);
	cout << "Thread " << this_thread::get_id() << " finished!" << endl;
	return;
}
int main() {
	thread th[10];
	for (int i = 0; i < 10; i++)
		th[i] = thread(::count);
	sleep(5000);
	ready = true;
	cout << "Start!" << endl;
	for (int i = 0; i < 10; i++)
		th[i].join();
	return 0;
}
~~~

c++20 std::jthread

###线程同步

避免竞态条件（Race Condition）和数据访问冲突等问题。常见的线程同步机制包括互斥锁（Mutex）、条件变量（Condition Variable）、信号量（Semaphore）、读写锁（Read-Write Lock）等

####原子

常用于实现锁、信号量、计数器等线程同步机制

`std::atomic`、`std::atomic_flag`等。

std::atomic<int> atomicCounter(0);

* 无锁编程

需要注意避免一些常见的陷阱和错误，如ABA问题、内存重排等。

#####ABA问题

具体来说，ABA问题的发生可以分为以下步骤：

1. 初始时刻，共享变量的值为A。
2. 线程T1读取共享变量的值A，并进行一些操作。
3. 在此过程中，另一个线程T2将共享变量的值从A修改为B，然后又将其修改回A，此时共享变量的值又变成了A。
4. 线程T1继续执行，根据共享变量值是否为A做出判断，认为共享变量的值没有被修改，继续执行操作。

在这个过程中，虽然共享变量的值在T1执行期间被修改了，但T1并没有察觉到这一变化，因为它只关心共享变量的当前值是否为A。这种情况下，T1可能会基于错误的假设继续执行，导致程序出现逻辑错误或安全问题。

为了解决ABA问题，通常需要使用一些技术手段，如版本号、时间戳等，来辅助判断共享变量的值是否发生了变化。另外，一些数据结构，如无锁队列、无锁栈等，也会使用类似的技术来避免ABA问题的发生。

#####内存重排

内存重排包括两种类型的重排：

1. **编译器重排**：编译器可能会对代码进行优化，改变指令的执行顺序，但不会改变代码的语义。例如，编译器可能会将一些独立的指令重排以提高局部性或减少分支预测失败的可能性。
2. **处理器重排**：处理器在执行指令时，为了提高性能，可能会对指令进行乱序执行或者延迟执行。处理器会根据指令之间的数据依赖关系和相关性进行重排，以尽可能地利用处理器的各种功能单元，提高指令执行效率。

内存重排的存在使得在多线程编程中保证程序的正确性变得更加困难。为了避免内存重排导致的问题，通常需要使用同步原语（如互斥锁、原子操作、内存栅栏等）来保证指令的顺序一致性，以及使用`volatile`关键字来告诉编译器不要对指令进行重排。

####条件变量

条件变量（Condition Variable）是一种线程同步机制，用于在线程间进行通信和同步。它允许一个线程在某个条件不满足时等待（阻塞），而另一个线程在满足条件时通知等待线程继续执行。条件变量通常与互斥锁（Mutex）一起使用，以确保线程之间的安全访问共享资源。

下面是条件变量的一般用法：

1. 创建条件变量和互斥锁：

   ```
   std::condition_variable condVar; // 条件变量
   std::mutex mtx; // 互斥锁
   ```

2. 等待条件：

   - 等待线程在进入临界区之前先锁定互斥锁，然后调用wait方法等待条件变量满足：

     ```
     std::unique_lock<std::mutex> lock(mtx); // 锁定互斥锁
     condVar.wait(lock, [](){ return condition; }); // 等待条件满足
     ```

3. 唤醒等待线程：

   - 唤醒线程在进入临界区之前先锁定互斥锁，然后调用notify_one或notify_all方法通知等待线程：

     ```
     std::unique_lock<std::mutex> lock(mtx); // 锁定互斥锁
     condition = true; // 修改条件
     condVar.notify_one(); // 唤醒等待线程
     ```

条件变量的使用可以帮助线程之间进行高效的同步和通信，避免了轮询等低效的等待方式。它常用于生产者-消费者模型、读者-写者模型等多线程编程场景中。

~~~c++
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>

std::queue<int> dataQueue; // 共享数据队列
std::mutex mtx; // 互斥锁
std::condition_variable condVar; // 条件变量

void producer() {
    for (int i = 0; i < 10; ++i) {
        {
            std::unique_lock<std::mutex> lock(mtx); // 锁定互斥锁

            // 生产数据并放入队列
            dataQueue.push(i);
            std::cout << "Produced data: " << i << std::endl;
        } // 在解锁前释放互斥锁

        // 唤醒等待的消费者线程
        condVar.notify_one();

        // 模拟生产数据的间隔
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }
}

void consumer() {
    for (int i = 0; i < 10; ++i) {
        std::unique_lock<std::mutex> lock(mtx); // 锁定互斥锁

        // 等待条件变量满足
        condVar.wait(lock, [](){ return !dataQueue.empty(); });

        // 从队列中获取数据并消费
        int data = dataQueue.front();
        dataQueue.pop();
        std::cout << "Consumed data: " << data << std::endl;

        // 模拟消费数据的处理时间
        std::this_thread::sleep_for(std::chrono::milliseconds(250));
    }
}

int main() {
    // 创建生产者线程和消费者线程
    std::thread producerThread(producer);
    std::thread consumerThread(consumer);

    // 等待生产者线程和消费者线程结束
    producerThread.join();
    consumerThread.join();

    return 0;
}
~~~

####信号量

在这个示例中，使用了 `Semaphore` 类来实现信号量的功能。`Semaphore` 类包含了 `notify()` 和 `wait()` 方法，用于通知和等待信号量。

生产者线程在每次生产数据后，调用 `semProducer.wait()` 等待生产者信号量，然后将数据放入队列并通知消费者信号量。消费者线程在每次消费数据前，调用 `semConsumer.wait()` 等待消费者信号量，然后从队列中取出数据并通知生产者信号量。

~~~c++
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>
#include <chrono>
class Semaphore {
public:
    Semaphore(int count = 0) : count_(count) {}

    void notify() {
        std::unique_lock<std::mutex> lock(mutex_);
        ++count_;
        condition_.notify_one();
    }
    void wait() {
        std::unique_lock<std::mutex> lock(mutex_);
        while (count_ == 0) {
            condition_.wait(lock);
        }
        --count_;
    }
private:
    std::mutex mutex_;
    std::condition_variable condition_;
    int count_;
};
std::queue<int> dataQueue; // 共享数据队列
Semaphore semProducer(1), semConsumer(0); // 生产者和消费者信号量
void producer() {
    for (int i = 0; i < 10; ++i) {
        semProducer.wait(); // 等待生产者信号量
        dataQueue.push(i); // 生产数据
        std::cout << "Produced data: " << i << std::endl;
        semConsumer.notify(); // 通知消费者信号量
        std::this_thread::sleep_for(std::chrono::milliseconds(100)); // 模拟生产数据的间隔
    }
}
void consumer() {
    for (int i = 0; i < 10; ++i) {
        semConsumer.wait(); // 等待消费者信号量
        int data = dataQueue.front(); // 消费数据
        dataQueue.pop();
        std::cout << "Consumed data: " << data << std::endl;
        semProducer.notify(); // 通知生产者信号量
        std::this_thread::sleep_for(std::chrono::milliseconds(250)); // 模拟消费数据的处理时间
    }
}
int main() {
    // 创建生产者线程和消费者线程
    std::thread producerThread(producer);
    std::thread consumerThread(consumer);
    // 等待生产者线程和消费者线程结束
    producerThread.join();
    consumerThread.join();

    return 0;
}
~~~

####读写锁

~~~c++
#include <iostream>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <chrono>
#include <vector>
class RWLock {
public:
    RWLock() : readerCount(0), writerCount(0) {}
    void lockRead() {
        std::unique_lock<std::mutex> lock(mutex_);
        while (writerCount > 0) {
            condition_.wait(lock);
        }
        ++readerCount;
    }
    void unlockRead() {
        std::unique_lock<std::mutex> lock(mutex_);
        --readerCount;
        if (readerCount == 0) {
            condition_.notify_one();
        }
    }
    void lockWrite() {
        std::unique_lock<std::mutex> lock(mutex_);
        ++writerCount;
        while (readerCount > 0 || writerCount > 1) {
            condition_.wait(lock);
        }
    }
    void unlockWrite() {
        std::unique_lock<std::mutex> lock(mutex_);
        --writerCount;
        condition_.notify_all();
    }
private:
    std::mutex mutex_;
    std::condition_variable condition_;
    int readerCount;
    int writerCount;
};
int data = 0; // 共享数据
RWLock rwLock; // 读写锁
void reader() {
    for (int i = 0; i < 10; ++i) {
        rwLock.lockRead(); // 加读锁
        std::cout << "Reader " << std::this_thread::get_id() << " reads data: " << data << std::endl;
        rwLock.unlockRead(); // 解读锁
        std::this_thread::sleep_for(std::chrono::milliseconds(100)); // 模拟读操作的间隔
    }
}
void writer() {
    for (int i = 0; i < 5; ++i) {
        rwLock.lockWrite(); // 加写锁
        data++; // 修改数据
        std::cout << "Writer " << std::this_thread::get_id() << " writes data: " << data << std::endl;
        rwLock.unlockWrite(); // 解写锁
        std::this_thread::sleep_for(std::chrono::milliseconds(250)); // 模拟写操作的间隔
    }
}
int main() {
    // 创建多个读者线程和写者线程
    std::vector<std::thread> readers, writers;
    for (int i = 0; i < 3; ++i) {
        readers.emplace_back(reader);
    }
    for (int i = 0; i < 2; ++i) {
        writers.emplace_back(writer);
    }
    // 等待所有线程结束
    for (auto& readerThread : readers) {
        readerThread.join();
    }
    for (auto& writerThread : writers) {
        writerThread.join();
    }
    return 0;
}
~~~

在这个示例中，使用了 `RWLock` 类来实现读写锁的功能。`RWLock` 类包含了 `lockRead()`、`unlockRead()`、`lockWrite()` 和 `unlockWrite()` 方法，分别用于读锁的加锁、读锁的解锁、写锁的加锁和写锁的解锁。

读者线程在每次读取数据前，调用 `rwLock.lockRead()` 加读锁，读取数据后调用 `rwLock.unlockRead()` 解读锁。写者线程在每次修改数据前，调用 `rwLock.lockWrite()` 加写锁，修改数据后调用 `rwLock.unlockWrite()` 解写锁。

通过使用读写锁，实现了读者-写者模型中的线程同步和并发控制，提高了读操作的并发性能。

####线程局部存储TLS

线程局部存储（Thread-Local Storage，TLS）是一种线程特定数据（Thread-Specific Data，TSD）的存储机制

允许每个线程都拥有自己独立的数据副本。这意味着每个线程都可以独立地访问和修改自己的数据副本，而不会影响其他线程的数据。

但是自C++11起，标准库提供了一个名为`std::thread_local`的关键字，可以轻松实现线程局部存储。

~~~c++
#include <iostream>
#include <thread>

// 定义线程局部变量
thread_local int tls_variable = 0;

void threadFunction() {
    // 每个线程都有自己的tls_variable副本
    tls_variable++;
    std::cout << "Thread ID: " << std::this_thread::get_id() << ", TLS variable: " << tls_variable << std::endl;
}

int main() {
    std::thread t1(threadFunction);
    std::thread t2(threadFunction);

    t1.join();
    t2.join();

    return 0;
}
~~~

`tls_variable`是一个线程局部变量，每个线程都有自己独立的副本。在`threadFunction()`函数中，每个线程都会递增自己的`tls_variable`副本，并打印出当前线程的ID和TLS变量的值。因为每个线程都有自己的副本，所以输出的结果会显示不同的线程ID和不同的TLS变量值。

## 网络IO

<内存分为内核缓冲区和用户缓冲区>

用户不能直接操作内核缓存区 

而 IO 操作、网络请求加载到内存的数据一开始是放在内核缓冲区的

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240829161908376.png" alt="image-20240829161908376" style="zoom: 80%;" />

五种网络IO模型

**BIO-阻塞模式I/O**

用户发起请求(系统调用)，如果有(内核缓存区),从内核缓存区拷贝到用户缓存区,如果没有用户进行阻塞等待 进行IO操作将数据拿到内核缓存区

**NIO-非阻塞模式**

用户进程发起请求，如果数据没有准备好，直接返回错误；可以选择轮询访问或者做点其他事再来访问,直到被告知数据准备完毕，可以开始接收为止； 数据会由用户进程完成拷贝

整个过程  轮询(轮询消耗资源,影响性能)+等待(内核缓冲区到用户缓存区) 

同一个线程，同一时刻只能监听一个socket，造成浪费，引入io多路复用，同时监听读个socket

**IO多路复用**

使用select、poll或epoll等系统调用监听多个IO事件。这些系统调用可以让应用程序同时监视多个IO操作，并在某个IO操作就绪时通知应用程序

数据会由用户进程完成拷贝

适用同时处理多个连接如网络服务器  单个连接效率低

**信号驱动IO模型**

应用程序通过系统调用请求系统在IO完成时向应用程序发送一个信号。当IO操作完成时，操作系统会向应用程序发送一个信号，应用程序在==信号处理函数==中处理IO完成事件。

适用于多个IO事件 设计复杂,过多连接导致消耗大  

**AIO 异步I/O模型**

应用程序发起IO操作后，可以立即返回并继续执行其他操作。当IO操作完成时，操作系统会通知应用程序，并在==回调函数中==处理IO完成事件。

发起请求立刻得到回复，不用挂起等待； 数据会由内核进程主动完成拷贝

高并发高性能,大规模并行计算和网络服务

**C/C++ Linux 服务器开发高级架构学习视频点击**https://ke.qq.com/course/417774?flowToken=1013189

选择哪个框架取决于你的具体需求和项目背景。如果你需要一个跨平台的解决方案，Boost.Asio可能是最佳选择；如果你需要一个功能齐全且易于使用的库，Poco是一个好选择；而如果你正在构建高性能的Linux服务器应用，MUDuo则非常合适。





- 《unix环境高级编程》
- 《unix网络编程》

如果你要写一个WebServer，入门的书籍当然最好是游双的《Linux高性能服务器编程》(读书笔记:https://github.com/HiganFish/Notes-HighPerformanceLinuxServerProgramming)了，当这本内容基本都掌握后，如果有能力还可以再去看一下陈硕大佬的《Linux多线程服务器编程》使用muduo C++网络库，去阅读源码，自制一个属于自己的网络库，相信对你的coding能力是一个很大的提升。


祖师爷[GitHub - qinguoyi/TinyWebServer: :fire: Linux下C++轻量级WebServer服务器](https://github.com/qinguoyi/TinyWebServer)

师叔[GitHub - markparticle/WebServer: C++ Linux WebServer服务器](https://github.com/markparticle/WebServer)

师哥https://love6.blog.csdn.net/article/details/123754194 源码https://github.com/Cooi-Boi/High-Performance-WebServer

师弟https://blog.csdn.net/weixin_51322383/article/details/130464403 源码https://github.com/JehanRio/TinyWebServer?tab=readme-ov-file

c++网络编程三大框架对比https://zhuanlan.zhihu.com/p/714229209







