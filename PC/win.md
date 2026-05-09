# 安装系统

1. 下载windows系统镜像
从官网下载 https://www.microsoft.com/en-us/software-download/windows11
或从镜像网站下载 https://next.itellyou.cn/

2. 将镜像放入 Ventory 制作的U盘中

3. 重启电脑启动菜单中选择U盘启动
> 如果安装系统过程中无法识别到系统硬盘，需要在bios中将 VMD Controller 修改为 Disabled ，如果菜单中没有VMD可能需要使用ctrl+s打开隐藏选项

4. 安装驱动
> 如果没有wifi驱动需要在另一台电脑上下一个

5. 安装必备软件

Office-Tool-Plus https://www.officetool.plus/zh-cn/ 激活使用：irm https://get.activated.win | iex
7-zip
Potplayer

# 用机

## 全能本开启性能模式

1. 开启电脑自带性能模式
redmi：fn+k

2. 更改电源计划
设置 → 系统 → 电源和电池
电源模式：最佳性能

3. 开启卓越性能模式

控制面板 → 电源选项
→ 高性能 / 卓越性能（如果有）
或
管理员运行powershell：

```powershell
# 查看电源计划
powercfg /list
# 设置为卓越
powercfg -duplicatescheme e9a42b02-d5df-448d-aa00-03f14749eb61
```

4. NAVIDIA控制面板
电源管理模式 → Prefer maximum performance

* 如果无法修改可能需要更新驱动：`https://blog.csdn.net/qq_50981222/article/details/127788191`

5. 其他
更新 BIOS + EC/固件
装 HWInfo64 / MSI Afterburner 看cpu和gpu功率严重是否跑满