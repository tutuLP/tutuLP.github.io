---
title: "go-zero"
date: 2025-08-26
categories:
  - 后端
tags:
  - go-zero
---

# go-zero 

文档：https://go-zero.dev/docs/tutorials

## 环境安装

```sh
# 安装器安装
https://go.dev/dl/
go version
go env -w GOPROXY=https://goproxy.cn,direct
go env GOPROXY
## linux环境变量
~/.bash_profile末尾添加export PATH=$PATH:/usr/local/go/bin   
source ~/.bash_profile

# 安装go-zero 脚手架 goctl
go install github.com/zeromicro/go-zero/tools/goctl@latest
goctl --version
## linux环境变量
echo 'export PATH=$PATH:$(go env GOPATH)/bin' >> ~/.bashrc
source ~/.bashrc
## mac 环境变量
echo 'export PATH=$PATH:$(go env GOPATH)/bin' >> ~/.zshrc
source ~/.zshrc

# 代码生成工具 protoc
goctl env check --install --verbose --force   # 需魔法
goctl env check --verbose

# 安装go-zero
mkdir <project name> && cd <project name>  # project name 为具体值
go mod init <module name> # module name 为具体值
go get -u github.com/zeromicro/go-zero@latest

# 安装etcd
https://github.com/etcd-io/etcd/releases
sudo mv etcd etcdctl /usr/local/bin/

## 临时启动（关闭终端后停止）
etcd
## 编辑配置文件 ~/.zshrc
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=http://localhost:2379

source ~/.zshrc

# vscode 插件：goctl
```

## 新建项目

```sh
go mod init spark 
mkdir api & cd api
goctl api go -api <spark.api> -dir .

# etcd
goctl rpc protoc user.proto --go_out=. --go-grpc_out=. --zrpc_out=.

# run
go mod tidy
go run shorturl.go -f etc/shorturl-api.yaml
```



### api文件demo

```api
syntax = "v1"

type (
	LoginReq {
		phone    string `json:"phone"`
		password string `json:"password"`
	}
	LoginResp {
		access_token  string `json:"access_token"`
		refresh_token string `json:"refresh_token"`
	}
	GetUserInfoResp {
		public_id     string `json:"public_id"`
		username      string `json:"username"`
		avatar_url    string `json:"avatar_url"`
		gender        int    `json:"gender"`
		birthday      string `json:"birthday"`
		country       string `json:"country"`
		last_login_ip string `json:"last_login_ip"`
		status        string `json:"status"`
		ext_json      string `json:"ext_json"`
	}
)

// type (
//  Test {
//   username string `json:"username"`
//   password string `json:"password"`
//  }
// )
@server (
	prefix: /api
	group:  user
)
service main {
	@doc "登录"
	@handler login
	post /user (LoginReq) returns (LoginResp)

	@doc "获取用户信息"
	@handler getUserInfo
	get /user returns (GetUserInfoResp)
}

// @server (
//  prefix: /api
//  group:  test
// )
// service main {
//  @handler test
//  get /test returns (Test)
// }
```

### rpc demo

```proto
syntax = "proto3";

package user;

option go_package = "./user";

message loginReq {
    string phone = 1;
    string password = 2;
}

message loginResp {
    string access_token = 1;
    string refresh_token = 2;
}

message getUserInfoReq {
    string access_token = 1;
}

message getUserInfoResp {
    string public_id = 1;
    string username = 2;
    string avatar_url = 3;
    int32 gender = 4;
    string birthday = 5;
    string country = 6;
    string last_login_ip = 7;
    int32 status = 8;
    string ext_json = 9;
}

message refreshRep {
    string refresh_token = 1;
}

message refreshResp {
    string access_token = 1;
    string refresh_token = 2;
}

service user {
    rpc login(loginReq) returns(loginResp);
    rpc getUserInfo(getUserInfoReq) returns(getUserInfoResp);
    rpc refreshToken(refreshRep) returns(refreshResp);
}
```

## mysql

goctl model mysql ddl --src usersql.sql --dir .

```sql
CREATE TABLE user (
    user_id            BIGINT        NOT NULL COMMENT '用户唯一ID，雪花算法',
    public_id          VARCHAR(64)   UNIQUE COMMENT '外部可见id',
    username           VARCHAR(64)   UNIQUE COMMENT '用户名，唯一',
    email              VARCHAR(128)  UNIQUE COMMENT '邮箱地址',
    phone              VARCHAR(32)   UNIQUE COMMENT '手机号（加密存储）',
    password_hash      VARCHAR(128)  NOT NULL COMMENT '密码哈希',
    avatar_url         VARCHAR(256)  COMMENT '头像URL',
    gender             TINYINT       COMMENT '性别：0-未知 1-男 2-女',
    birthday           DATE          COMMENT '出生日期',
    country            VARCHAR(64)   COMMENT '国家/地区',
    register_ip        VARCHAR(45)   COMMENT '注册IP',
    register_time      DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '注册时间',
    last_login_time    DATETIME      COMMENT '上次登录时间',
    last_login_ip      VARCHAR(45)   COMMENT '上次登录IP',
    status             TINYINT       NOT NULL DEFAULT 1 COMMENT '账号状态：0-禁用 1-启用 2-注销',
    ext_json           JSON          COMMENT '扩展信息（如用户标签、偏好等）',
    PRIMARY KEY (`user_id`),
    INDEX idx_email (email),
    INDEX idx_phone (phone),
    INDEX idx_status (status),
    INDEX idx_last_login_time (last_login_time)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci COMMENT = '用户信息表';
```

