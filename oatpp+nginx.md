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



# Step by Step

~~~c++
#include "oatpp/web/server/HttpConnectionHandler.hpp"

#include "oatpp/network/Server.hpp"
#include "oatpp/network/tcp/server/ConnectionProvider.hpp"

void run() {

	/* Create Router for HTTP requests routing */
	auto router = oatpp::web::server::HttpRouter::createShared();//Http请求的路由，url映射到端点处理程序，现在没有声明端点，返回404

	/* Create HTTP connection handler with router *///http连接处理，每一个连接一个线程
	auto connectionHandler = oatpp::web::server::HttpConnectionHandler::createShared(router);

	/* Create TCP connection provider *///tcp连接，绑定一个端口
	auto connectionProvider = oatpp::network::tcp::server::ConnectionProvider::createShared({ "localhost", 8000, oatpp::network::Address::IP_4 });

	/* Create server which takes provided TCP connections and passes them to HTTP connection handler */
	oatpp::network::Server server(connectionProvider, connectionHandler);//server运行一个循环，获取tcp连接，传到连接处理

	/* Print info about server port */
	OATPP_LOGI("MyApp", "Server running on port %s", connectionProvider->getProperty("port").getData());

	/* Run server */
	server.run();
}
int main() {
	/* Init oatpp Environment */
	oatpp::base::Environment::init();
	/* Run App */
	run();
	/* Destroy oatpp Environment */
	oatpp::base::Environment::destroy();
	return 0;

}
~~~

###添加请求处理程序

当url /hello请求做出处理

访问http:/localhost:8000/hello 返回hello world消息

~~~c++
class Handler : public oatpp::web::server::HttpRequestHandler {
public:
  std::shared_ptr<OutgoingResponse> handle(const std::shared_ptr<IncomingRequest>& request) override {
    return ResponseFactory::createResponse(Status::CODE_200, "Hello World!");
  }
};

void run(){
	router->route("GET", "/hello", std::make_shared<Handler>());
}
~~~

### 使用JSON对象进行响应

为了序列化/反序列化对象，oatpp 使用特殊的[数据传输对象(DTO)和ObjectMappers

~~~c++
#include "oatpp/parser/json/mapping/ObjectMapper.hpp"
#include "oatpp/core/macro/codegen.hpp"

/* Begin DTO code-generation */
#include OATPP_CODEGEN_BEGIN(DTO)

/**
 * Message Data-Transfer-Object
 */
class MessageDto : public oatpp::DTO {

  DTO_INIT(MessageDto, DTO /* Extends */)

  DTO_FIELD(Int32, statusCode);   // Status code field
  DTO_FIELD(String, message);     // Message field

};

/* End DTO code-generation */
#include OATPP_CODEGEN_END(DTO)

class Handler : public oatpp::web::server::HttpRequestHandler {
private:
  std::shared_ptr<oatpp::data::mapping::ObjectMapper> m_objectMapper;
public:

  /**
   * Constructor with object mapper.
   * @param objectMapper - object mapper used to serialize objects.
   */
  Handler(const std::shared_ptr<oatpp::data::mapping::ObjectMapper>& objectMapper)
    : m_objectMapper(objectMapper)
  {}

  /**
   * Handle incoming request and return outgoing response.
   */
  std::shared_ptr<OutgoingResponse> handle(const std::shared_ptr<IncomingRequest>& request) override {
    auto message = MessageDto::createShared();
    message->statusCode = 1024;
    message->message = "Hello DTO!";
    return ResponseFactory::createResponse(Status::CODE_200, message, m_objectMapper);
  }

};

void run() {

  /* Create json object mapper */
  auto objectMapper = oatpp::parser::json::mapping::ObjectMapper::createShared();

  /* Create Router for HTTP requests routing */
  auto router = oatpp::web::server::HttpRouter::createShared();

  /* Route GET - "/hello" requests to Handler */
  router->route("GET", "/hello", std::make_shared<Handler>(objectMapper /* json object mapper */ ));
  }
~~~

## oatpp的结构

DTO 数据传输对象 定义了数据的格式

controller 使用api controller 端点处理程序(处理路由请求) 使用dto中的类封装消息

AppComponent.hpp 定义必须的组件(HTTP连接，请求路由，TCP连接，ObjectMapper)

App.cpp main 路由器(请求) 获取上述组件中的路由-获取controller中的处理程序-建立路由到处理程序的映射

​						  获取HTTP连接-获取TCP连接-建立服务-从TCP连接提供者发送消息到http连接处理

<img src="http://typora-tutu.oss-cn-chengdu.aliyuncs.com/img/image-20240328095617737.png" alt="image-20240328095617737" style="zoom:33%;" />

### 代码

DTOs.hpp

~~~c++
#ifndef DTOs_hpp
#define DTOs_hpp

#include "oatpp/core/data/mapping/type/Object.hpp"
#include "oatpp/core/macro/codegen.hpp"
#include "oatpp/parser/json/mapping/ObjectMapper.hpp"

#include "oatpp/web/server/HttpConnectionHandler.hpp"

#include "oatpp/network/Server.hpp"
#include "oatpp/network/tcp/server/ConnectionProvider.hpp"

/* Begin DTO code-generation */
#include OATPP_CODEGEN_BEGIN(DTO)

/**
 * Message Data-Transfer-Object
 */
class MessageDto : public oatpp::DTO {

	DTO_INIT(MessageDto, DTO /* Extends */)

		DTO_FIELD(Int32, statusCode);   // Status code field
	DTO_FIELD(String, message);     // Message field

};

/* TODO - Add more DTOs here */

/* End DTO code-generation */
#include OATPP_CODEGEN_END(DTO)

#endif /* DTOs_hpp */
~~~

MyController.hpp

~~~c++
#ifndef MyController_hpp
#define MyController_hpp

#include "dto/DTOs.hpp"

#include "oatpp/web/server/api/ApiController.hpp"
#include "oatpp/core/macro/codegen.hpp"
#include "oatpp/core/macro/component.hpp"

#include OATPP_CODEGEN_BEGIN(ApiController) ///< Begin Codegen

/**
 * Sample Api Controller.
 */
class MyController : public oatpp::web::server::api::ApiController {
public:
    /**
     * Constructor with object mapper.
     * @param objectMapper - default object mapper used to serialize/deserialize DTOs.
     */
    MyController(OATPP_COMPONENT(std::shared_ptr<ObjectMapper>, objectMapper))
        : oatpp::web::server::api::ApiController(objectMapper)
    {}
public:

    ENDPOINT("GET", "/hello", root) {
        auto dto = MessageDto::createShared();
        dto->statusCode = 200;
        dto->message = "Hello World!";
        return createDtoResponse(Status::CODE_200, dto);
    }

    // TODO Insert Your endpoints here !!!

};

#include OATPP_CODEGEN_END(ApiController) ///< End Codegen

#endif /* MyController_hpp */
~~~

AppComponent.hpp

~~~cpp
#ifndef AppComponent_hpp
#define AppComponent_hpp

#include "oatpp/parser/json/mapping/ObjectMapper.hpp"
#include "oatpp/web/server/HttpConnectionHandler.hpp"
#include "oatpp/network/tcp/server/ConnectionProvider.hpp"
#include "oatpp/core/macro/component.hpp"

/**
 *  Class which creates and holds Application components and registers components in oatpp::base::Environment
 *  Order of components initialization is from top to bottom
 */
class AppComponent {
public:
    /**
     *  Create ConnectionProvider component which listens on the port
     */
    OATPP_CREATE_COMPONENT(std::shared_ptr<oatpp::network::ServerConnectionProvider>, serverConnectionProvider)([] {
        return oatpp::network::tcp::server::ConnectionProvider::createShared({ "localhost", 8000, oatpp::network::Address::IP_4 });
        }());

    /**
     *  Create Router component
     */
    OATPP_CREATE_COMPONENT(std::shared_ptr<oatpp::web::server::HttpRouter>, httpRouter)([] {
        return oatpp::web::server::HttpRouter::createShared();
        }());

    /**
     *  Create ConnectionHandler component which uses Router component to route requests
     */
    OATPP_CREATE_COMPONENT(std::shared_ptr<oatpp::network::ConnectionHandler>, serverConnectionHandler)([] {
        OATPP_COMPONENT(std::shared_ptr<oatpp::web::server::HttpRouter>, router); // get Router component
        return oatpp::web::server::HttpConnectionHandler::createShared(router);
        }());

    /**
     *  Create ObjectMapper component to serialize/deserialize DTOs in Contoller's API
     */
    OATPP_CREATE_COMPONENT(std::shared_ptr<oatpp::data::mapping::ObjectMapper>, apiObjectMapper)([] {
        return oatpp::parser::json::mapping::ObjectMapper::createShared();
        }());

};

#endif /* AppComponent_hpp */
~~~

App.cpp

~~~cpp
#include "controller/MyController.hpp"
#include "AppComponent.hpp"
#include "oatpp/network/Server.hpp"

void run() {

	/* Register Components in scope of run() method */
	AppComponent components;

	/* Get router component */
	OATPP_COMPONENT(std::shared_ptr<oatpp::web::server::HttpRouter>, router);

	/* Create MyController and add all of its endpoints to router */
	auto myController = std::make_shared<MyController>();
	router->addController(myController);

	/* Get connection handler component */
	OATPP_COMPONENT(std::shared_ptr<oatpp::network::ConnectionHandler>, connectionHandler);

	/* Get connection provider component */
	OATPP_COMPONENT(std::shared_ptr<oatpp::network::ServerConnectionProvider>, connectionProvider);

	/* Create server which takes provided TCP connections and passes them to HTTP connection handler */
	oatpp::network::Server server(connectionProvider, connectionHandler);

	/* Priny info about server port */
	OATPP_LOGI("MyApp", "Server running on port %s", connectionProvider->getProperty("port").getData());

	/* Run server */
	server.run();

}

int main(int argc, const char* argv[]) {

	/* Init oatpp Environment */
	oatpp::base::Environment::init();

	/* Run App */
	run();

	/* Destroy oatpp Environment */
	oatpp::base::Environment::destroy();

	return 0;
}
~~~



~~~cpp
#ifndef MyController_hpp
#define MyController_hpp

#include "dto/DTOs.hpp"
#include "oatpp/web/server/api/ApiController.hpp"
#include "oatpp/core/macro/codegen.hpp"
#include "oatpp/core/macro/component.hpp"

#include OATPP_CODEGEN_BEGIN(ApiController) ///< Begin Codegen

/**
 * Sample Api Controller.
 */
class MyController : public oatpp::web::server::api::ApiController {
public:
    /**
     * Constructor with object mapper.
     * @param objectMapper - default object mapper used to serialize/deserialize DTOs.
     */
    MyController(OATPP_COMPONENT(std::shared_ptr<oatpp::data::mapping::ObjectMapper>, objectMapper))
        : oatpp::web::server::api::ApiController(objectMapper)
    {}

public:

    ENDPOINT("GET", "/hello", roothello) {
        auto dto = MessageDto::createShared();
        dto->statusCode = 200;
        dto->message = "Hello World!";
        return createDtoResponse(Status::CODE_200, dto);
    }

    // TODO Insert Your endpoints here !!!
    ENDPOINT("GET", "/", root) {
        auto response = createResponse(Status::CODE_200, "<html><body><form action=\"/submit\" method=\"post\"><input type=\"text\" name=\"input\" placeholder=\"Enter something\"><button type=\"submit\">Submit</button></form></body></html>");
        response->putHeader(oatpp::web::protocol::http::Header::CONTENT_TYPE, "text/html");
        return response;
    }

    ENDPOINT("POST", "/submit", submit, BODY_STRING(String, body)) {
        OATPP_LOGI("MyController", "Received input: %s", body->c_str());
        auto response = createResponse(Status::CODE_200, "Received input: " + *body);
        response->putHeader(oatpp::web::protocol::http::Header::CONTENT_TYPE, "text/plain");
        return response;
    }

};

#include OATPP_CODEGEN_END(ApiController) ///< End Codegen

#endif /* MyController_hpp */

~~~

