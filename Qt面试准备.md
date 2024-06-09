---
title: "Qt面试"
date: 2024-06-09
categories:
  - 就业
tags:
  - qt
---

## 核心机制

**元对象系统** (信号与槽 运行时类型信息 动态属性)

​	直接或间接继承QObject可以使用

​	声明Q_OBJECT宏

​	元对象编译器moc 将代码转化给c++编译器

**属性系统**

**信号与槽**

## Qt创建多线程的方式

1. QThread 最方便 可开启事件循环 可信号

~~~c++
QThread *t=new QThread;
server * worker=new server;
worker->moveToThread(t);
~~~

新建一个类继承自Qthread 这样可以通过构造函数传递参数

~~~c++
server *serverthread=new server(seed,myserver->nextPendingConnection());
serverthread->start();
~~~

2. QtConcurrent::run() 在单独的线程中异步处理事务 更简洁方便

~~~c++
QT += core gui concurrent
    
QFuture<void> future = QtConcurrent::run([this]() {   //或调用一个函数
            // Simulate a long-running task
            QThread::sleep(5); // Sleep for 5 seconds
            // Update the label text (this part runs in the main thread)
            QMetaObject::invokeMethod(this, "updateLabel", Qt::QueuedConnection);
        });
~~~

3. QThreadPool 和 QRunnable 适合多个对象和复杂情况

~~~c++
MyTask *task = new MyTask(label);
QThreadPool::globalInstance()->start(task);
class MyTask : public QRunnable {
public:
    	MyTask(QLabel *label) : m_label(label) {}
    void run() override {
        // Simulate a long-running task
        QThread::sleep(5); // Sleep for 5 seconds
        QMetaObject::invokeMethod(m_label, "setText", Qt::QueuedConnection,Q_ARG(QString, "Task Completed!"));
    }
private:
    QLabel *m_label;
};
~~~

## 信号与槽

类似观察者模式

对象间传递消息：回调函数 MFC就是使用这个 使用函数指针 不保证类型安全

~~~c++
void printWelcome(int len){printf("欢迎欢迎 -- %d/n", len);}
void printGoodbye(int len){printf("送客送客 -- %d/n", len);}
void callback(int times, void(*print)(int)){
	int i;
	for (i = 0; i < times; ++i) { print(i); }
	printf("/n我不知道你是迎客还是送客!/n/n");
}	
void main(void){
	callback(2, printWelcome); callback(2, printGoodbye);
}
~~~

对象树：自动有效管理继承自QObject的Qt对象 父对象被析构时子对象也析构 避免内存泄漏

~~~c++
signals
emit
slots 可以不用声明这个关键字，但是一般需要
设置connect的第五个参数可以不用阻塞等待槽函数完成执行
//两个原型：
connect(pushButton, SIGNAL(clicked()), dialog, SLOT(close()));
connect(pushButton, &QPushButton::clicked, dialog, &QDialog::close);
//信号发送者指针，信号函数字符串/信号函数地址，槽

//lambda表达式
connect(anysocket, &QTcpSocket::disconnected, this, [=]()
            {
            });
//同一个类中的匿名函数
connect(this, &A::sig_hello, []{
    qDebug() << "hello world!";
});
~~~

可通过ui界面绑定信号与槽

https://blog.csdn.net/ddllrrbb/article/details/88374350

QSignalMapper 绑定对象数据一并传递

第五个参数：

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240609150150846.png" alt="image-20240609150150846" style="zoom:33%;" />

## 事件过滤器

重写eventFilter方法 不会处理这个信号 

给QApplication安装，实现全局

~~~c++
bool eventFilter(QObject *watched, QEvent *event) override {
        if (event->type() == QEvent::MouseButtonPress) {
            qDebug() << "Button clicked";
            // 处理事件，返回 true 表示事件已经被处理，不需要继续传播
            return true;
        }
        // 其他事件，继续传播
        return QObject::eventFilter(watched, event);
    }

MyEventFilter *eventFilter = new MyEventFilter();
// 在按钮上安装事件过滤器
button->installEventFilter(eventFilter);
~~~

## 保证线程安全

1. 互斥量（QMutex）
   QMutex m_Mutex; m_Mutex.lock(); m_Mutex.unlock();

2. 互斥锁（QMutexLocker）
   QMutexLocker mutexLocker(&m_Mutex);
   从声明处开始（在构造函数中加锁），出了作用域自动解锁（在析构函数中解锁）。

3. 等待条件（QWaitCondition）
   QWaitCondtion m_WaitCondition; m_WaitConditon.wait(&m_muxtex, time);
   m_WaitCondition.wakeAll();

4. QReadWriteLock类
   》一个线程试图对一个加了读锁的互斥量进行上读锁，允许；
   》一个线程试图对一个加了读锁的互斥量进行上写锁，阻塞；
   》一个线程试图对一个加了写锁的互斥量进行上读锁，阻塞；、
   》一个线程试图对一个加了写锁的互斥量进行上写锁，阻塞。
   读写锁比较适用的情况是：需要多次对共享的数据进行读操作的阅读线程。
   QReadWriterLock 与QMutex相似，除了它对 “read”,"write"访问进行区别对待。它使得多个读者可以共时访问数据。使用QReadWriteLock而不是QMutex，可以使得多线程程序更具有并发性。

5. 信号量QSemaphore
   但是还有些互斥量（资源）的数量并不止一个，比如一个电脑安装了2个打印机，我已经申请了一个，但是我不能霸占这两个，你来访问的时候如果发现还有空闲的仍然可以申请到的。于是这个互斥量可以分为两部分，已使用和未使用。

6. QReadLocker便利类和QWriteLocker便利类对QReadWriteLock进行加解锁



## 其他

QT中的智能指针封装为QPointer类



**文件流**

文件流(QTextStream):操作轻量级数据（int,double,QString）数据写入文本件中以后以文本的方式呈现。
数据流(QDataStream):通过数据流可以操作各种数据类型，包括对象，存储到文件中数据为二进制。

文件流，数据流都可以操作磁盘文件，也可以操作内存数据。通过流对象可以将对象打包到内存，进行数据的传输。



**post请求**

Qt如何发送一个HTTP post请求并接收消息？
在pro文件添加network模块（使用qmake时）
使用网络连接请求类(QNetworkRequest)，添加URL网址，并设置HTTP的头部行
使用网络访问管理器类(QNetworkAccessManager)发送post请求，同时需要添加HTTP附属体内容，返回得到网络回复类(QNetworkReply)对象
当网络回复对象收到返回的消息时会触发相应的信号，可以通过连接槽函数，在槽函数中获取返回的数据

