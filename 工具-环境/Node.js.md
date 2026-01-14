# 下载

## Mac/Linux

```shell
#使用镜像源安装 nvm
curl -o- https://gitee.com/mirrors/nvm/raw/v0.39.3/install.sh | bash
source ~/.bashrc
nvm --version
#安装Node
nvm install --lts

# 设置环境变量
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
[ -s "$NVM_DIR/bash_completion" ] && . "$NVM_DIR/bash_completion"

source ~/.zshrc
```

## windows

```shell
https://github.com/coreybutler/nvm-windows/releases
nvm install --lts
nvm use --lts
nvm list
nvm use 22.14.0
# 添加环境变量
C:\Users\tutu\AppData\Roaming\nvm
C:\Users\tutu\AppData\Roaming\nvm\v22.14.0
node -v
npm -v
```

