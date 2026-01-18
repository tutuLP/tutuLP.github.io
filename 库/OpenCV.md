---
title: "OpenCV"
date: 2025-03-02
categories:
  - 库
---

# 下载

* msvc编译的版本

官网：https://opencv.org/releases/ 下载windows版本  

环境变量Path  C:\APP\opencv\build\x64\vc16\bin 

* MinGW编译版

https://github.com/huihut/OpenCV-MinGW-Build 下载

环境变量 C:\APP\OpenCV-MinGW-Build\x64\mingw\bin

# Qt中使用OpenCV

教程：https://blog.csdn.net/kdnnnd/article/details/132840038

* 查看qt中使用的编译器是什么，我用的是MinGW

<img src="./images/OpenCV.assets/image-20250125165808294.png" alt="image-20250125165808294" style="zoom:33%;" />

* pro文件写入内容

```
INCLUDEPATH+= C:\APP\OpenCV-MinGW-Build\include\
              C:\APP\OpenCV-MinGW-Build\include\opencv2

LIBS+=C:\APP\OpenCV-MinGW-Build\x64\mingw\bin\libopencv_*.dll
```

* 测试

```
#include <QCoreApplication>
#include "opencv2/opencv.hpp"
int main(int argc, char *argv[])
{
    QCoreApplication a(argc, argv);
    using namespace cv;

    Mat image=imread("C:/MY/Qt-project/yun1/Default.jpg");
    imshow("Output",image);
    return a.exec();
}
```

* MSVC编译器则为

```
INCLUDEPATH +=C:\OpenCV_s\opencv_vc\opencv\build\include\
              C:\OpenCV_s\opencv_vc\opencv\build\include\opencv\
              C:\OpenCV_s\opencv_vc\opencv\build\include\opencv2

LIBS +=C:\OpenCV_s\opencv_vc\opencv\build\x64\vc15\lib\opencv_world3414.lib 
或 C:\OpenCV_s\opencv_vc\opencv\build\x64\vc15\lib\opencv_world3414d.lib
注意: opencv_world3414d.lib 为debug版,opencv_world3414.lib为release版
```









