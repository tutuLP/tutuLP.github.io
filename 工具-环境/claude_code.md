# Claude Code

> 中文文档: <https://code.claude.com/docs/zh-CN/overview>
> 菜鸟教程：<https://www.runoob.com/claude-code/claude-code-tutorial.html>

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

### 使用CC Switch管理账号

<https://github.com/farion1231/cc-switch>

```shell
# 进入目标项目文件夹
cd my-prohect

# 启动
claude
# 使用指定模型启动
claude --model deepseek-v4-pro

# 查看能使用的模型
/model

# 退出
exit
```

### 在vscode中使用

> 同样受cc switch管理

* 关闭登录组件

vscode设置-扩展- claudecode，勾选Disable Login Prompt

参考：<https://code.claude.com/docs/zh-CN/memory>

<https://www.runoob.com/claude-code/vscode-install-claude-code.html>

## 状态栏配置

> 项目地址：<https://github.com/huangguang1999/ccstatusline-zh>

* 安装

  `npm install -g ccstatusline-zh`

* 设置Claude Code状态栏配置

  需要在`~/.claude/settings.json`添加下述内容

```json
{
  "statusLine": {
    "type": "command",
    "command": "ccstatusline-zh",
    "padding": 0,
    "refreshInterval": 10
  }
}
```

* 使用

  `ccstatusline-zh setup`

* 配置参考

  <img src="C:\MY\md\tutuLP.github.io\工具-环境\images\claude_code.assets\image-20260521154231710.png" alt="image-20260521154231710" style="zoom:33%;" />

## skills

多skills的集合，按需安装

<https://github.com/ComposioHQ/awesome-claude-skills>

行动准则，减少无用代码的书写

<https://github.com/multica-ai/andrej-karpathy-skills>

先问再做（长任务，需求模糊）

Using-Superpowers

头脑风暴，模糊的想法转化为可执行方案

Brainstorming

把多种文件格式转化为md

markitdown

论文及科研制图

<https://github.com/Yuan1z0825/nature-skills>

完整的软件开发流程集合

<https://github.com/obra/superpowers>

pdf 读取，合并/拆分

去AI味

humanizer-zh

代码审查

code-review



综合参考

<https://github.com/affaan-m/ECC>



### skill-creator

目前有两个skill-creator，一个是官方的<https://github.com/anthropics/skills/tree/main/skills/skill-creator>,一个是其他作者比较热门的<https://github.com/ComposioHQ/awesome-claude-skills/tree/master/skill-creator>

但是一般写好的skill还是需要自己审查一遍并且完善
