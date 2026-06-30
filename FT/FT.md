```
mysql --default-character-set=utf8mb4 -u ftcd02_user01   -h xz13  -p -P3358  
jnKD9mer2y47s3  权重数据导入/xz03 看板 ftcd02_dashboard

SET NAMES utf8mb4;

mysql -u ftcd02_user01   -h xz14  -P 3509  -p   
yFdNspoNLYvM4s3RL3Xq  基础数据入库

mysql -u root   -h xz20  -P 3308  -p 
mysql_url = "mysql://root:M42PjGmMsxAa4Z2h24otE6uK@xz20:3308"

mysql -u ftcd02_user01 -h xz02  -p -P3509 
UbLRJNr4eTZJC
ftdb_version


redis-cli -u redis://ft11:6371

mysql -u ftcd02_user01   -h ft11  -p -P3302 
jnKD9mer2y47s3
base_data
mysql://ftcd02_user01:jnKD9mer2y47s3@ft11:3302/base_data

ssh broker-dify0

mysql -u ftcd02_user01 -h xz02  -p -P3509 
UbLRJNr4eTZJC    web_serve


```



# 服务器与远程

堡垒机

~~~
ssh xiongjianzhou@sfijnfnepo-public.bastionhost.aliyuncs.com -p60022
# 个人编译机
FtXiongJianZhou
172.16.19.76/22
~~~

~~~
ssh xz04_ftcdcs02    #172.16.14.4
exit
~~~

SSH密钥

~~~
ssh-keygen
cat ~/.ssh/id_rsa.pub
~~~

~~~
git config --global user.email "@xiongjianzhou"
git config --global user.name "熊健州"
~~~

服务器文件传输到堡垒机

~~~
scp xz04_ftcdcs02:/data2/share/ft_files/2025/20250327/RQ.info.etf.20250327.csv  /home/dev/xiongjianzhou
rcp
~~~

# HDFS

hdfs 查询文件命令

~~~
hadoop fs -ls -R /share/etf_stocks/*/${this_td}
hdfs dfs -ls  /share/etf_stocks/HuaxiaFund/20250722/
hdfs dfs -rm -r -skipTrash /share/etf_stocks/HuaxiaFund/20250722/

hdfs 下载文件夹中所有的文件到本地：
hdfs dfs -get /user/api/EXTERNAL_DATA/HUABAO/y2025/20250408Pre/  /home/z/xjz/csifile

hdfs dfs -cat  ...    | iconv -f GBK -t UTF-8 | head
~~~

依赖

~~~rust
data-protocol = { git = "ssh://git@code.non-convex.com/gateway/ft-base-data-series/data-protocol.git", tag = "v0.0.*" } 
data-protocol = { git = "ssh://git@code.non-convex.com/gateway/ft-base-data-series/data-protocol.git", tag = "v0.0.60", features = ["hdfs"] }
data-protocol = { git = "ssh://git@code.non-convex.com/gateway/ft-base-data-series/data-protocol.git", branch = "add_etf_daily_data", features = ["hdfs"] }

~~~

初始化

~~~rust
use hdrs::Client;
use once_cell::sync::Lazy;

static HDFS: Lazy<Client> = Lazy::new(|| {
    hdrs::ClientBuilder::new("hdfs://ftxz-hadoop").connect().unwrap()
});
或
let hdfs = hdrs::ClientBuilder::new("hdfs://ftxz-hadoop").connect().unwrap();
然后作为参数传入函数Some(&hdfs)


~~~

操作远端的单个文件

~~~
if matches!(hdfs.metadata(&path).map_err(|e: std::io::Error| e.kind() == std::io::ErrorKind::NotFound),Err(true)) 
{
    anyhow::bail!("hdfs找不到文件: {:?}", &path);
}
let file = hdfs.open_file().read(true).open(&path)?;
~~~



hdfs dfs -cat  ...    | iconv -f GBK -t UTF-8 | head

hdfs 下载文件夹中所有的文件到本地：
hdfs dfs -get /user/api/EXTERNAL_DATA/HUABAO/y2025/20250408Pre/  /home/z/xjz/csifile



更多操作见项目:http://code.non-convex.com:18118/gateway/ft-base-data-series/data-protocol/-/tree/master/

我使用的例子:http://code.non-convex.com:18118/ftcd/basedata_check/-/blob/csv_check/src/base_check/csv_active_check.rs

sh脚本上传文件:http://code.non-convex.com:18118/ftcd/basedata_fetcher/-/blob/get_dump/sh/dump_etf_weight.sh

## 快速下载文件

dump.sh

> demo:下载指定日期范围的文件到文件夹local_dest

```sh
#!/bin/bash

# 定义日期范围
start_date="20240909"
end_date="20250925"
hdfs_base_path="/user/api/EXTERNAL_DATA/WIND/CLOSE_WEIGHT_AINDEXCSI1000WEIGHT"
local_dest="/data1/ftcd/weight/temp"

# 创建本地目录（如果不存在）
mkdir -p "$local_dest"

# 生成日期列表并多线程下载
seq $(date -d "$start_date" +%s) 86400 $(date -d "$end_date" +%s) | while read timestamp; do
    date=$(date -d @$timestamp +%Y%m%d)
    echo "${hdfs_base_path}/${date}.csv"
done | xargs -n 1 -P 10 -I {} hdfs dfs -get {} "$local_dest/"

echo "所有文件下载完成"
```



# 告警

message_client = { git = "ssh://git@code.2502.info/base/message_client.git", tag = "v2.0.1"}

```
1. 先构建两个字符串，存储告警需要发送的信息
let mut suc_msg = "".to_string();
let mut err_msg = "".to_string();

2.构建报警信息
map_or_else 处理 Result<>返回
suc_msg = format!(
    "{}\n{}",
    suc_msg,
    check_FullgoaFund_etf(date).map_or_else(
        |e| {
            err_msg = format!("{}\n{}", err_msg, e.to_string());
            "".to_string()
        },
        |s| s
    )
);

3. 发送告警
--错误告警

message_client::alert!(
    format!("<font color=\"#dd0000\">**文件检查失败!**</font>\n>**任务说明:** {}_{}\n>**文件说明:** {}\n>**任务Owner:** <@{}>","ETF碎股文件",date,err_msg, "Zed").as_str(),
    format!("[markdown][{}_ERROR][{}]",BASE_ETF_WEIGHT_TAG,"基础数据" ).as_str(),
    60, // 60s发送一次
    API_ALERT_TAG,
    0,
    "qywechat"
);

--成功告警
message_client::alert!(
    format!("<font color=\"#00dd00\">**文件检查通过!**</font>\n>**任务说明:** {}_{}\n>**文件说明:** {}","ETF碎股文件",date, suc_msg).as_str(),
    format!("[markdown][{}_SUCCEESS][{}]",BASE_ETF_WEIGHT_TAG,"基础数据" ).as_str(),
    600, // 60s发送一次
    API_ALERT_TAG,
    0,
    "qywechat"
);

4.睡眠 10 秒再结束程序
std::thread::sleep(core::time::Duration::from_secs(10));
```







# 定时任务

git@code.non-convex.com:gateway/ft-base-data-series/data-generator.git

分支:feat/deploy-ftcdcs



定时任务部署配置:http://code.non-convex.com:18118/gateway/ft-base-data-series/data-generator/-/blob/feat/deploy-ftcdcs/.gitlab-ci.yml

时间规则:https://docs.rs/cron/0.15.0/cron/

定时任务部署位置:cd /home/z/projects/gateway/ft-base-data-series/data-generator/ftcd-data-generator-bin-xz04_ftcdcs02/running

xz04_ftcdcs02的定时测试:
tail -f logs/test_task.log



在conf.xz04_ftcdcs02_tasks.toml中新增

~~~
[[cmd_task]]
crons = [
    "00 30 17 * * *",
    "00 30 20 * * *",
]
cmd = "cd /data1/ftcd/basedata_fetcher/sh/  && chmod +x dump_etf_weight.sh && ./dump_etf_weight.sh"
msg = "邮件附件下载和推送"
receipt = "FT_BASEDATA_CHECK_TEST"
owner = "XiongJianZhou"
succ_alarm = true
run_on_trading_day = true
~~~

push之后,需要CICD中Compile中点击xz01_ftcdcs02然后Deploy

# 交易日以及时间判断

stock-tool-box = { git = "ssh://git@code.non-convex.com/gateway/stock-tool-box.git", tag = "v0.0.43" }

```
use stock_tool_box::{trade_date_util::{get_this_date, is_trade_date, last_n_trade_date}, trade_time_util::{get_hour, now_nanos}};

//交易日判断
let args: Vec<String> = env::args().collect();
let date = args[1].parse::<i32>().unwrap();
if !is_trade_date(date) {
    info!("is not trade date {}]", date);
    return Ok(());
}

//小时判断
let hour = get_hour(now_nanos());
if hour >= 9 
```

# 日志系统

ftlog = "0.2.12"

日志路径：error 日志会自动检测 panic

`/data2/share/logalert/FT_BASEDATA_CHECK_TEST#xiongjianzhou_index_weight_mysql_xz/app.log`

```
sudo mkdir 
sudo chmod 777 FT_BASEDATA_CHECK_TEST#xiongjianzhou_index_weight_mysql_xz
```



使用说明：

```
use ftlog::{info,error,warn,debug,trace};
info!("这是 info 告警示例");

logger().flush();
```





```
use ftlog::{appender::{Duration, Period}, info, LevelFilter};

pub fn log_init() {
    let log_path = format!(); //构建日志存放位置
    ftlog::builder()
        .max_log_level(LevelFilter::Info)  //设置日志级别
        .unbounded()
        .root(ftlog::appender::FileAppender::rotate_with_expire(
            log_path,
            Period::Day,
            Duration::days(5),   //日志保存时间
        ))
        .build()
        .expect("logger build failed")
        .init()
        .expect("set logger failed ");
    std::thread::sleep(core::time::Duration::from_secs(5));
}

main 中先调用 log_init()初始化日志系统
info!("is not trade date {}]", date);  //使用
```







# 导出数据

1. 导出位置 xz06 ～/xjz
2. 确认查询条件

A股的股票数据

```
REGEXP_LIKE(S_INFO_WINDCODE, '^[0-9]{6}.SZ$') OR REGEXP_LIKE(S_INFO_WINDCODE, '[0-9]{6}.SH$') OR REGEXP_LIKE(S_INFO_WINDCODE, '^[0-9]{6}.BJ$') OR REGEXP_LIKE(S_INFO_WINDCODE, '[0-9]{6}.BJ$')"
```

其中S_INFO_WINDCODE 字段在不同表中名字不同

> 在 万得数据_数据字典 文件夹中搜索表名，打开文件，其中WIND 代码就是

3. 替换表名和日期执行

AShareISQA

```sh
table=AShareISQA && date=20250508 && sqluldr2 user=bingzhao/bingzhao@172.16.14.4:11010/WIND 'field=|' txt=CSV file=${table}_${date}.csv  charset=UTF8 safe=Yes head=Yes query="SELECT * FROM ${table} WHERE
REGEXP_LIKE(S_INFO_WINDCODE, '^[0-9]{6}.SZ$') OR REGEXP_LIKE(S_INFO_WINDCODE, '[0-9]{6}.SH$') OR REGEXP_LIKE(S_INFO_WINDCODE, '^[0-9]{6}.BJ$') OR REGEXP_LIKE(S_INFO_WINDCODE, '[0-9]{6}.BJ$')"
```



## 任务1

每天导出表AShareFinancialIndicator 最近9个季度的数据到/user/t0/jam/fd_base_data/日期/表名.parquet或者csv

本地数据位置：/home/z/xjz/sql/user/t0/jam/fd_base_data



```sh
table=AShareFinancialIndicator && date=20250508 && sqluldr2 user=bingzhao/bingzhao@172.16.14.4:11010/WIND 'field=|' txt=CSV file=${table}.csv  charset=UTF8 safe=Yes head=Yes query=" SELECT *
FROM ${table}
WHERE TO_CHAR(TO_DATE(REPORT_PERIOD, 'YYYYMMDD'), 'YYYYQ') IN (
  SELECT DISTINCT TO_CHAR(TO_DATE(REPORT_PERIOD, 'YYYYMMDD'), 'YYYYQ') AS year_quarter
  FROM ${table}
  ORDER BY year_quarter DESC
  FETCH FIRST 9 ROWS ONLY
);"
```

项目：xz06：xjz/ft_oracle_dump
分支：jingchangjin_dev

```
function dump_t0_jam() {
  local trade_date=$1
  local out_dir="/data1/nick/data/jam/${trade_date}"
  mkdir -p ${out_dir}
  local last_day=$(date +"%Y%m%d" --date "${trade_date} -1 day")

  local tables=(
    AShareBalanceSheet
    AShareCashFlow
    AShareIncome
  )
  for table in ${tables[@]}; do
    local sql_file=./sql/DUMP_JAM_TABLE_BY_TRADE_DT.sql
    local out_csv="${out_dir}/${table}.csv"
    dump_to_csv $sql_file $out_csv '|' ${table} ${last_day} ${trade_date}
  done

  local out_csv="${out_dir}/${table}.csv"
  local table=AShareFinancialIndicator
  local sql_file=./sql/DUMP_ASHAREFINANCIALINDICATOR_TABLE_IN_LAST27MONTH.sql
  local last_date=$(date +"%Y%m01" --date "${trade_date} -27 month")

  dump_to_csv $sql_file $out_csv '|' ${last_date} ${trade_date}

}

SELECT * FROM ASHAREFINANCIALINDICATOR WHERE REPORT_PERIOD >= :param1 AND REPORT_PERIOD <= :param2;
```

## 验证

```
source exp_oracle.sh
dump_t0_jam
ll /data1/nick/data/jam/20250521# 
```

# reader

> data-protocol = {  git = "ssh://git@code.non-convex.com/gateway/ft-base-data-series/data-protocol.git",  branch = "get_wind_data_xjz",  features = ["hdfs"]  }

* daily_wind_reader

`cargo run --bin daily_wind_reader -- -d 20250717 -t Aindexcsi1000weight --from-hdfs`

* daily_ai_agent_reader

`cargo run --bin daily_ai_agent_reader -- -d 20250717 -t Aindexeodprices --from-hdfs`
