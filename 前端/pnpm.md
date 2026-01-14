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

