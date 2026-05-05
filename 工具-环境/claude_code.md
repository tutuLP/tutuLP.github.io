# Claude Code

## 下载

```Shell
# mac/linux
curl -fsSL https://claude.ai/install.sh | bash
# powershell
irm https://claude.ai/install.ps1 | iex
```

* 设置环境变量并验证

安装完成之后会提示安装的位置，比如：Location: C:\Users\tutu.local\bin\claude.exe,将bin文件夹的路径添加到环境变量，然后重启终端进行验证，注意如果是vscode中的终端需要重启整个vscode


`claude --version`

## 配置和使用

此处提供的是三方接口，而不是直接使用登录账号的方式\
需要提前准备好API服务令牌 ANTHROPIC\_AUTH\_TOKEN 和API服务地址 ANTHROPIC\_BASE\_URL，设置环境变量之后使用 `claude`启动即可
