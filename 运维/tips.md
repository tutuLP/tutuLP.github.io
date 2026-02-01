## 修改文件句柄数

* 查看当前句柄限制

`ulimit -n`

### 永久提升

编辑文件 `/etc/security/limits.conf`,在文件最后写入下述两行

> 需要重新连接服务器生效

```
* soft nofile 65533 
* hard nofile 65533
```

### 当前会话修改

`ulimit -n 65535`