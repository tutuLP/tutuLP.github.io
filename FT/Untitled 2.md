* 后台执行任务

nohup nice -n 0 python news_crawler_v2.py 2021 >/dev/null 2>&1 &

* 根据进程号查询任务状态

ps -p 2302861

* 查询后台进程

ps aux | grep -E "python|news_crawler.py" | grep -v grep

* 查看当前终端所有后台任务

jobs -l

* 统计文件数目

find . -type f | wc -l

* 当前子文件夹大小

du -h --max-depth=1

* grep

grep -n -C 10 "panic: failed to extract any content from NBD article" \
/data2/share/logalert/spider/FT_BASEDATA_SPIDER#ft-news-spider/app-20260108.log

* sha256sum



无法完成此操作，因为必须跳过某些项目。在每个项目下，选取“文件”>“显示简介”，确保取消选择“锁定”，然后检查“共享与权限”部分。在确定项目已解锁且未指定为“只读”或“无法访问”后，请重试。

完全删除后重新安装，使用appcleaner删除



touchbar触控条不显示：

sudo pkill TouchBarServer
