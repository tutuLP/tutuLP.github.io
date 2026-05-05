

* SHA-256 哈希值
sha256sum

## 文件传输
scp rsync

传输脚本
```shell
#!/bin/bash
# 远程服务器信息
USER="your_username"
HOST="remote_host"
PASSWORD="your_password"
REMOTE_PATH="/remote/directory/"
LOCAL_FILE="/path/to/local/file"
# 使用 sshpass 和 scp 进行文件传输
sshpass -p "$PASSWORD" scp "$LOCAL_FILE" "$USER@$HOST:$REMOTE_PATH"
# 检查传输是否成功
if [ $? -eq 0 ]; then
    echo "文件传输成功"
else
    echo "文件传输失败"
fi
```

## 任务与进程

```shell
# 根据进程号查询任务状态
ps -p <pid>

# 根据关键字查询后台进程，grep -v grep表示去掉自身
ps aux | grep -E "python|news_crawler.py" | grep -v grep
ps -ef | grep your_command

# 查看当前终端所有后台任务
jobs -l

# 后台执行任务 nice指定优先级(默认为0，从低到高为19到-20，高优先级需要root) 标准重定向输出，如果没有指定则默认输出到nohup.out，>/dev/null表示丢弃输出，2>&1错误输出重定向到标准输出，&表示后台运行
nohup nice -n 0 <...任务运行命令> >/dev/null 2>&1 &
```

## 查找与统计

```shell
# 递归统计当前文件夹中所有文件数目
find . -type f | wc -l

# 当前文件夹所有子文件夹大小
du -h --max-depth=1

# 查找文件中的内容  忽略大小写/显示行号/递归目录查找/-C 3前后三行
grep [-i -n -r -C] "关键词" 文件名 

# 查找文件名 忽略大小写-iname
find 路径 -name "文件名"

```

## 网络

```shell
# 测试网络是否通（路由问题）
ping ip

# 测试网络，服务，防火墙
telnet ip port  
# Connection timed out   防火墙阻隔或网络不通
# Connection refused 机器通，端口没服务

# 查看建立的tcp连接
netstat -atn

# 抓包
tcpdump -i 网口名
```

## c程序优化

### cpu资源占用分析

```shell 
# 按线程维度查看 CPU 使用情况
top -H -p 12345
# 查看 CPU 正在执行哪些函数
perf top -p 进程ID
```

### 绑核

```shell
# 绑核（CPU 密集型任务可以绑，IO密集无需，先分析如果某个线程占满了，可以考虑绑核）
# 启动时绑定，只使用 CPU 0 和 1
taskset -c 0,1 ./app
# 对已有进程
taskset -cp 0 12345
# 绑定线程
# 找线程 ID
top -H -p 12345
# 把线程 12346 绑到 CPU 2
taskset -cp 2 12346
```

### 内存分析

记录所有 malloc/free 行为，最后输出泄漏情况
valgrind --leak-check=full --show-leak-kinds=all ./my_program

## tool

### vim

```shell
#显示行号
:set number
```

### tcpdump

> 工具下载地址：https://archive.fedoraproject.org/pub/archive/epel/7/aarch64/Packages/t/

```shell
# 抓包并保存
sudo tcpdump -i eth0 -w capture.pcap
# 重放
sudo tcpreplay -i eth0 capture.pcap
# 指定速度
sudo tcpreplay -i eth0 -x 2 capture.pcap
# 指定次数
sudo tcpreplay -i eth0 -l 10 capture.pcap
```

