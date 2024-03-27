---
title: "oatpp-nginx"
date: 2024-03-27
categories:
  - Web
tags:
  - HTTP 请求处理、路由、JSON 序列化
  - Web开发框架
---

# 安装oatpp

我先在oatpp的github的官方网站上找到windows的下载方式

官方网站：https://github.com/oatpp/oatpp

下载方式：https://oatpp.io/docs/installation/windows/

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240327161303655.png" alt="image-20240327161303655" style="zoom:33%;" />

而后我尝试进行下载，第一次我忽略了下方的cmake参数，下面我整理了完整指令，在想要安装oatpp的地方cmd输入指令

~~~cmd
git clone https://github.com/oatpp/oatpp.git
cd oatpp
MD build
cd build
cmake -DCMAKE_BUILD_TYPE=Debug -DBUILD_SHARED_LIBS=OFF ..#更多参数可选项根据需求，我觉得这两个是必须的
cmake --build . --target INSTALL
~~~

此处建议直接跳转到**链接方法**

既然库安好了，我开始想通过VS新建CMake项目链接库进行，但是我一直尝试都没有成功，文档放在这个最后(直接忽略)

目录结构为外层lib-oatpp test[代码存放] CMakelists.txt lib-oatpp中还有cmakelists test中有src-include-cmakelists



而后我又想直接把oatpp的源码放进项目中但是一直提示无法打开文件/或找不到文件

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240326205418158.png" alt="image-20240326205418158" style="zoom:33%;" />

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240326205448363.png" alt="image-20240326205448363" style="zoom:33%;" />

![image-20240326221209405](http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240326221209405.png)

历尽千辛万苦我终于知道如何链接了

##链接方法

找到之前下载的oatpp/src的位置"D:\oatpp\src"填入 右键项目-属性-c/c++-附加包含目录

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240327163552082.png" alt="image-20240327163552082" style="zoom:33%;" />

找到D:\oatpp\build\src\Debug 填入下图，里面有四个库文件

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240327163852259.png" alt="image-20240327163852259" style="zoom:33%;" />

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240327163753697.png" alt="image-20240327163753697" style="zoom:33%;" />

填入四个库

`oatpp-test.lib
oatpp.lib
wsock32.lib
ws2_32.lib`

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240327163935539.png" alt="image-20240327163935539" style="zoom:33%;" />

然后完事

##CMakelistst(忽略)

以后也许我会再次尝试

顶层cmake文件，执行全局配置

~~~cmake
cmake_minimum_required (VERSION 3.8)
project ("cmake-oatpp-nginx")
set (CMAKE_CXX_STANDARD 17)
set (CMAKE_CXX_STANDARD_REQUIRED ON) #严格遵循c++标准

# 添加宏定义
if (ZO_BT STREQUAL "r")
	add_definitions(-DOATPP_DISABLE_ENV_OBJECT_COUNTERS)
	message (STATUS "Build type release")
endif()
if (UNIX)
	add_definitions(-D_CRT_SECURE_NO_WARNINGS -D_SILENCE_ALL_CXX17_DEPRECATION_WARNINGS)
	add_definitions(-DCPP_JWT_USE_VENDORED_NLOHMANN_JSON)
	add_definitions(-DLINUX)
	add_definitions(-DCHECK_TOKEN)
	add_definitions(-DSTOP_PWD="01star")
	add_definitions(-DOATPP_SWAGGER_SERVICE_NAME="${PROJECT_NAME} for linux")
	add_definitions(-DOATPP_SWAGGER_RES_PATH="res")
	add_definitions(-DBSONCXX_STATIC -DMONGOCXX_STATIC -DENABLE_AUTOMATIC_INIT_AND_CLEANUP=OFF)
else()
	add_definitions(-DOATPP_SWAGGER_SERVICE_NAME="${PROJECT_NAME} for windows")
	add_definitions(-DOATPP_SWAGGER_RES_PATH="res")
endif()

# 在camke .. 的时候会输出提示目录路径
message (STATUS "Prefix dir is ${CMAKE_INSTALL_PREFIX}") #安装
message (STATUS "Binary dir is ${PROJECT_BINARY_DIR}") #二进制
message (STATUS "Source dir is ${PROJECT_SOURCE_DIR}") #源代码
message (STATUS "Build type is ${CMAKE_BUILD_TYPE}") #构建类型
message (STATUS "Platform for x64 = ${CMAKE_CL_64}") #平台

# 定义一个预编译标头宏
MACRO(ADD_MSVC_PRECOMPILED_HEADER PrecompiledHeader PrecompiledSource SourcesVar)
  IF(MSVC)
    GET_FILENAME_COMPONENT(PrecompiledBasename ${PrecompiledHeader} NAME_WE)
    SET(PrecompiledBinary "${CMAKE_CURRENT_BINARY_DIR}/${PrecompiledBasename}.pch")
    SET(Sources ${${SourcesVar}})
    SET_SOURCE_FILES_PROPERTIES(${PrecompiledSource}
                                PROPERTIES COMPILE_FLAGS "/Yc\"${PrecompiledHeader}\" /Fp\"${PrecompiledBinary}\""
                                           OBJECT_OUTPUTS "${PrecompiledBinary}")
    SET_SOURCE_FILES_PROPERTIES(${Sources}
                                PROPERTIES COMPILE_FLAGS "/Yu\"${PrecompiledHeader}\" /FI\"${PrecompiledHeader}\" /Fp\"${PrecompiledBinary}\""
                                           OBJECT_DEPENDS "${PrecompiledBinary}") 
    LIST(APPEND ${SourcesVar} ${PrecompiledSource})
  ENDIF(MSVC)
ENDMACRO(ADD_MSVC_PRECOMPILED_HEADER)

add_subdirectory ("lib-oatpp")

add_subdirectory ("test")
~~~

test目录下cmakelists

~~~cmake
cmake_minimum_required (VERSION 3.8)
set (appName test)
include_directories ("./")
include_directories ("../lib-oatpp/include")
# 链接库路径，程序运行的时候也在这里找
link_directories (${PROJECT_BINARY_DIR}/lib-oatpp)
#链接库的搜索路径
if(UNIX)
	link_directories (${PROJECT_SOURCE_DIR}/lib-oatpp/lib64)
# 如果是Windows环境
elseif(WIN32)
	if (CMAKE_CL_64)
		link_directories (${PROJECT_SOURCE_DIR}/lib-oatpp/lib64/openssl)
		if(ZO_BT STREQUAL "r")
			link_directories (${PROJECT_SOURCE_DIR}/lib-oatpp/lib64)
		else()
			link_directories (${PROJECT_SOURCE_DIR}/lib-oatpp/lib64/debug)
		endif()
	else()
		link_directories (${PROJECT_SOURCE_DIR}/lib-oatpp/lib/openssl)
		if(ZO_BT STREQUAL "r")
			link_directories (${PROJECT_SOURCE_DIR}/lib-oatpp/lib)
		else()
			link_directories (${PROJECT_SOURCE_DIR}/lib-oatpp/lib/debug)
		endif()
	endif()
endif()

# 获取要编译的源代码
file (GLOB_RECURSE SC_FILES ./*.cpp) #通配符-当前目录以及子目录下所有的.cpp文件
list (REMOVE_ITEM SC_FILES "${CMAKE_CURRENT_SOURCE_DIR}/./stdafx.cpp") #从列表SC_FILES中移除指定文件-预编译头文件

# 设置预编译标头
if(WIN32)
	ADD_MSVC_PRECOMPILED_HEADER("stdafx.h" "stdafx.cpp" SC_FILES)
endif()

# 编译可执行文件
add_executable (${appName} ${SC_FILES})

# 链接库
target_link_libraries (${appName} "lib-oatpp")

# Window平台复制dll文件到可执行文件目录
if(WIN32)
	file (GLOB_RECURSE dycopy ${ZO_DY_DIR}/*.dll)
	file (COPY ${dycopy} DESTINATION "${PROJECT_BINARY_DIR}/${appName}")
endif()
# Window平台复制项目配置到可执行文件目录
if(WIN32)
	file (GLOB conf "conf/*")
	file (COPY ${conf} DESTINATION ${PROJECT_BINARY_DIR}/${appName}/conf)
endif()

# 安装文件
# public目录
if(IS_ARCH_DEMO)
	install (DIRECTORY "public" DESTINATION ${appName})
endif()
# 可执行文件
install (TARGETS ${appName} RUNTIME DESTINATION ${appName})
# Window平台复制项目配置到可执行文件目录
if(WIN32)
	install (DIRECTORY "conf" DESTINATION ${appName})
endif()
~~~





















~~~
// main.cpp

#include "oatpp/web/server/HttpConnectionHandler.hpp"
#include "oatpp/web/server/HttpRouter.hpp"
#include "oatpp/web/server/HttpRequestHandler.hpp"
#include "oatpp/web/protocol/http/outgoing/ResponseFactory.hpp"
#include "oatpp/network/tcp/server/ConnectionProvider.hpp"
#include "oatpp/core/macro/component.hpp"

class MyController : public oatpp::web::server::HttpRequestHandler {
public:
  /**
   * Handle incoming HTTP request
   */
  std::shared_ptr<OutgoingResponse> handle(const std::shared_ptr<IncomingRequest>& /*request*/) override {
    // Return "Hello, World!" response
    return oatpp::web::protocol::http::outgoing::ResponseFactory::createResponse(Status::CODE_200, "Hello, World!");
  }
};

/**
 *  Example ConnectionProvider component that listens on port 8080
 */
OATPP_CREATE_COMPONENT(std::shared_ptr<oatpp::network::ServerConnectionProvider>, serverConnectionProvider)([] {
  return oatpp::network::tcp::server::ConnectionProvider::createShared({ "0.0.0.0", 8080, oatpp::network::Address::IP_4 });
}());

/**
 *  Example Router component
 */
OATPP_CREATE_COMPONENT(std::shared_ptr<oatpp::web::server::HttpRouter>, httpRouter)([] {
  auto router = oatpp::web::server::HttpRouter::createShared();
  router->route("GET", "/api/hello", std::make_shared<MyController>());
  return router;
}());

/**
 *  Example ConnectionHandler component that combines ConnectionProvider and Router
 */
OATPP_CREATE_COMPONENT(std::shared_ptr<oatpp::network::ConnectionHandler>, serverConnectionHandler)([] {
  OATPP_COMPONENT(std::shared_ptr<oatpp::network::ServerConnectionProvider>, connectionProvider); // get ConnectionProvider component
  OATPP_COMPONENT(std::shared_ptr<oatpp::web::server::HttpRouter>, router); // get Router component

  auto connectionHandler = oatpp::web::server::HttpConnectionHandler::createShared(router);
  connectionHandler->setConnectionProvider(connectionProvider);
  return connectionHandler;
}());

int main() {
  OATPP_COMPONENT(std::shared_ptr<oatpp::network::ServerConnectionProvider>, connectionProvider);
  OATPP_COMPONENT(std::shared_ptr<oatpp::network::ConnectionHandler>, connectionHandler);

  oatpp::network::Server server(connectionProvider, connectionHandler);

  // Print info about server port
  OATPP_LOGI("MyApp", "Server running on port %s", connectionProvider->getProperty("port").toString()->c_str());

  // Run server
  server.run();

  return 0;
}
~~~

在这个示例中，我们创建了一个名为 `MyController` 的类，用于处理 HTTP 请求，并返回 "Hello, World!" 消息。然后，我们定义了一个 HTTP 路由，将 `/api/hello` 路径映射到 `MyController` 类。最后，我们创建了一个 HTTP 服务器，监听在 8080 端口上，并将路由和连接提供器与之关联。

编译并运行这个程序，它将启动一个 HTTP 服务器，监听在 8080 端口上。接下来，你需要配置 Nginx，将 `/api/hello` 请求代理到这个端口上。在 Nginx 的配置文件中，你可以添加类似以下的配置：

```
nginxCopy codeserver {
    listen 80;
    server_name your_server_domain.com;

    location /api/hello {
        proxy_pass http://127.0.0.1:8080;
    }
}
```

保存并重新加载 Nginx 配置，然后访问 `http://your_server_domain.com/api/hello`，你应该能够看到 "Hello, World!" 的响应。