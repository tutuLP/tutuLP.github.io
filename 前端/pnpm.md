# pnpm

## 配置全局目录

### windows

```shell
# 设置用户的 PNPM_HOME 变量
pnpm setup
# 保证path变量中存在PNPM_HOME的路径：C:\Users\tutu\AppData\Local\pnpm
pnpm config set global-bin-dir "C:\Users\tutu\AppData\Local\pnpm"
# 如果还是没有成功，使用临时变量
$env:PATH = "C:\Users\tutu\AppData\Local\pnpm;$env:PATH"
```
windows

## pnpm换源

```shell
pnpm config set registry https://registry.npmmirror.com

```



## 清除没有使用的包和lock文件

```shell
pnpm add -g depcheck
# 检查依赖
depcheck

# 移除依赖示例
pnpm remove @element-plus/icons-vue \
  @kangc/v-md-editor \
  @milkdown/core 
  
# 更新lock文件
pnpm install

# 删除未使用的依赖缓存
pnpm store prune
```
