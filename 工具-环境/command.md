# command

## Windows

* 记事本打开文本文件 notepad
* 拷贝文件 scp -r tencent01:/远程/文件夹/路径 本地目标路径
* Remove-Item 路径 -Recurse -Force

## Linux

### 压缩

* gzip（最常用，速度/压缩率均衡）

`tar -czf logs.tar.gz logs/`
`tar -xzf data.tar.gz`

`-c`：创建压缩包
`-z`：使用 gzip 压缩
`-f`：指定输出文件名
`-x`：解压缩包

* bzip2 （压缩率比 gzip 高，速度慢）

`tar -cjf data.tar.bz2 data/`
`tar -xjf data.tar.bz2`

* xz（文本压缩率最高，最慢）

`tar -cJf data.tar.xz  data/`
`tar -xJf data.tar.xz`

tar -cJf logs.tar.xz logs/ -I 'xz -9'

* zstd（现代方案，速度最快，压缩率≈gzip）

`tar --zstd -cf data.tar.zst data/`
`tar --zstd -xf data.tar.zst`

> 只压缩，目录内容不带外层目录(带-C)：`tar -czf data.tar.gz -C data .`

***压缩级别***

```shell
gzip     默认：-6   范围：-1 ～ -9
bzip2    默认：-9   范围：-1 ～ -9
xz       默认：-6   范围：-0 ～ -9（部分版本到 -9e）
zstd     默认：-3   范围：-1 ～ -19（可到 -22）

-I 'gzip -9'
-I 'bzip2 -9'
-I 'xz -9'
tar -I 'zstd -19' -cf data.tar.zst data/
```



