# Editor

## @milkdown/kit 纯净版

没有插件

```vue
<template>
  <div class="add-page">
    <div class="editor-container">
      <div id="milkdown-editor" class="milkdown-editor"></div>
    </div>
  </div>
</template>

<script setup>
import { Editor, rootCtx } from "@milkdown/kit/core";
import { commonmark } from "@milkdown/kit/preset/commonmark";
import { onMounted, onUnmounted } from "vue";
import "@milkdown/kit/prose/view/style/prosemirror.css";

import { history } from "@milkdown/kit/plugin/history";
import { nord } from "@milkdown/theme-nord";
import "@milkdown/theme-nord/style.css";

let editor = null;

onMounted(async () => {
  try {
    // 创建编辑器实例并挂载到指定元素
    editor = await Editor.make()
      .config(nord)
      .config((ctx) => {
        ctx.set(rootCtx, document.getElementById("milkdown-editor"));
      })
      .use(commonmark)
      .use(history)
      .create();

    console.log("Editor created successfully");
  } catch (error) {
    console.error("Failed to create editor:", error);
  }
});

onUnmounted(() => {
  if (editor) {
    editor.destroy();
  }
});
</script>

<style scoped>
.add-page {
  padding: 20px;
  height: calc(100vh - 120px); /* 减去导航栏高度 */
  background: #f5f5f5;
}

.editor-container {
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  height: 100%;
  overflow: hidden;
}

.milkdown-editor {
  height: 100%;
  width: 100%;
  padding: 16px;
  box-sizing: border-box;
}

/* 确保 Milkdown 编辑器内容正确显示 */
:deep(.milkdown) {
  height: 100%;
  outline: none;
}

:deep(.ProseMirror) {
  height: 100%;
  outline: none;
  padding: 0;
  margin: 0;
}
</style>
```

## @milkdown/crepe 内置版

使用了很多插件 https://milkdown.dev/docs/api/crepe

crepe的可用方法console.log("Crepe 可用方法:", Object.getOwnPropertyNames(crepe));

[
    "addFeature",
    "create",
    "destroy",
    "setReadonly",
    "getMarkdown",
    "on"
]

~~~vue
<template>
  <div class="add-page">
    <div class="editor-container">
      <div id="milkdown-editor" class="milkdown-editor"></div>
    </div>
  </div>
</template>

<script setup>
import { onMounted, onUnmounted } from "vue";
import { Crepe } from "@milkdown/crepe";
import "@milkdown/crepe/theme/common/style.css";
/**
 * Available themes:
 * frame, classic, nord
 * frame-dark, classic-dark, nord-dark
 */
import "@milkdown/crepe/theme/frame.css";

let crepe = null;

onMounted(() => {
  // 创建编辑器实例
  crepe = new Crepe({
    root: "#milkdown-editor",
    defaultValue: "# 开始编写您的内容\n\n在这里输入您的文本...",
  });

  crepe
    .create()
    .then(() => {
      console.log("Milkdown编辑器创建成功");
    })
    .catch((error) => {
      console.error("编辑器创建失败:", error);
    });
});

onUnmounted(() => {
  // 组件销毁时清理编辑器
  if (crepe) {
    crepe.destroy();
    crepe = null;
  }
});
</script>

<style scoped>
.add-page {
  padding: 20px;
  height: calc(100vh - 120px); /* 减去导航栏高度 */
  background: #f5f5f5;
}

.editor-container {
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  height: 100%;
  overflow: hidden;
}

.milkdown-editor {
  height: 100%;
  width: 100%;
}
</style>

~~~

# 插件

7.x版本的milkdown现在好像已经不提供工具栏插件了

只有选中文字的工具和/唤起工具

现在只能自定义实现

## Editor版的自定义插件

```js
<template>
  <div class="add-page">
    <div class="editor-container">
      <div class="custom-toolbar">
        <button class="toolbar-btn" @click="toggleBold" title="粗体">
          <strong>B</strong>
        </button>
        <button class="toolbar-btn" @click="toggleItalic" title="斜体">
          <em>I</em>
        </button>
        <button class="toolbar-btn" @click="toggleCode" title="代码">
          <code>&lt;/&gt;</code>
        </button>
        <button class="toolbar-btn" @click="toggleStrikethrough" title="删除线">
          <s>S</s>
        </button>
      </div>
      <div id="milkdown-editor" class="milkdown-editor"></div>
    </div>
  </div>
</template>

<script setup>
import { onMounted, onUnmounted } from "vue";
import { Editor, rootCtx, commandsCtx } from "@milkdown/core";
import { commonmark } from "@milkdown/preset-commonmark";
import { nord } from "@milkdown/theme-nord";
import {
  toggleStrongCommand,
  toggleEmphasisCommand,
  toggleInlineCodeCommand,
} from "@milkdown/preset-commonmark";
import "@milkdown/theme-nord/style.css";

let editor = null;
let commands = null;

// 工具栏按钮处理函数
const toggleBold = () => {
  if (commands) {
    commands.call(toggleStrongCommand.key);
  }
};

const toggleItalic = () => {
  if (commands) {
    commands.call(toggleEmphasisCommand.key);
  }
};

const toggleCode = () => {
  if (commands) {
    commands.call(toggleInlineCodeCommand.key);
  }
};

const toggleStrikethrough = () => {
  // 删除线功能需要额外的插件支持
  console.log("删除线功能需要额外插件");
};

onMounted(() => {
  // 创建编辑器实例
  editor = Editor.make()
    .use(commonmark)
    .use(nord)
    .config((ctx) => {
      ctx.set(rootCtx, document.querySelector("#milkdown-editor"));
      commands = ctx.get(commandsCtx);
    })
    .create()
    .then(() => {
      console.log("Milkdown编辑器创建成功");
    })
    .catch((error) => {
      console.error("编辑器创建失败:", error);
    });
});

onUnmounted(() => {
  // 组件销毁时清理编辑器
  if (editor) {
    editor.destroy();
    editor = null;
    commands = null;
  }
});
</script>

<style scoped>
.add-page {
  padding: 20px;
  height: calc(100vh - 120px); /* 减去导航栏高度 */
  background: #f5f5f5;
}

.editor-container {
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  height: 100%;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.custom-toolbar {
  display: flex;
  align-items: center;
  padding: 8px 12px;
  border-bottom: 1px solid #e0e0e0;
  background: #fafafa;
  gap: 4px;
}

.toolbar-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 32px;
  height: 32px;
  border: 1px solid #d0d0d0;
  background: white;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.2s ease;
}

.toolbar-btn:hover {
  background: #f0f0f0;
  border-color: #b0b0b0;
}

.toolbar-btn:active {
  background: #e0e0e0;
  transform: translateY(1px);
}

.milkdown-editor {
  flex: 1;
  width: 100%;
  overflow: auto;
}
</style>

```

## crepe版的自定义工具栏

```vue
<template>
  <div class="add-page">
    <div class="editor-container">
      <div class="custom-toolbar">
        <button class="toolbar-btn" @click="toggleBold" title="粗体">
          <strong>B</strong>
        </button>
        <button class="toolbar-btn" @click="toggleItalic" title="斜体">
          <em>I</em>
        </button>
        <button class="toolbar-btn" @click="toggleCode" title="代码">
          <code>&lt;/&gt;</code>
        </button>
        <button class="toolbar-btn" @click="toggleStrikethrough" title="删除线">
          <s>S</s>
        </button>
      </div>
      <div id="milkdown-editor" class="milkdown-editor"></div>
    </div>
  </div>
</template>

<script setup>
import { onMounted, onUnmounted } from "vue";
import { Crepe } from "@milkdown/crepe";
import { commandsCtx } from "@milkdown/core";
import {
  toggleStrongCommand,
  toggleEmphasisCommand,
  toggleInlineCodeCommand,
} from "@milkdown/preset-commonmark";
import "@milkdown/crepe/theme/common/style.css";
import "@milkdown/crepe/theme/frame.css";

let crepe = null;
let commands = null;

// 工具栏按钮处理函数
const toggleBold = () => {
  if (commands) {
    commands.call(toggleStrongCommand.key);
  }
};

const toggleItalic = () => {
  if (commands) {
    commands.call(toggleEmphasisCommand.key);
  }
};

const toggleCode = () => {
  if (commands) {
    commands.call(toggleInlineCodeCommand.key);
  }
};

const toggleStrikethrough = () => {
  // 删除线功能需要额外的插件支持
  console.log("删除线功能需要额外插件");
};

onMounted(() => {
  // 创建编辑器实例
  crepe = new Crepe({
    root: "#milkdown-editor",
    defaultValue: "# 开始编写您的内容\n\n在这里输入您的文本...",
  });

  crepe
    .create()
    .then(() => {
      console.log("Milkdown编辑器创建成功");
      // 获取命令管理器
      if (crepe.editor && crepe.editor.action) {
        crepe.editor.action((ctx) => {
          commands = ctx.get(commandsCtx);
        });
      }
    })
    .catch((error) => {
      console.error("编辑器创建失败:", error);
    });
});

onUnmounted(() => {
  // 组件销毁时清理编辑器
  if (crepe) {
    crepe.destroy();
    crepe = null;
    commands = null;
  }
});
</script>

<style scoped>
.add-page {
  padding: 20px;
  height: calc(100vh - 120px); /* 减去导航栏高度 */
  background: #f5f5f5;
}

.editor-container {
  background: white;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  height: 100%;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.custom-toolbar {
  display: flex;
  align-items: center;
  padding: 8px 12px;
  border-bottom: 1px solid #e0e0e0;
  background: #fafafa;
  gap: 4px;
}

.toolbar-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 32px;
  height: 32px;
  border: 1px solid #d0d0d0;
  background: white;
  border-radius: 4px;
  cursor: pointer;
  font-size: 14px;
  transition: all 0.2s ease;
}

.toolbar-btn:hover {
  background: #f0f0f0;
  border-color: #b0b0b0;
}

.toolbar-btn:active {
  background: #e0e0e0;
  transform: translateY(1px);
}

.milkdown-editor {
  flex: 1;
  width: 100%;
  overflow: auto;
}
</style>
```

# 内容

主题+代码块主题，高亮

````
# 开始编写您的内容

在这里输入您的文本...

<br />

```C++
# include<iostream>
int main(){
  return 0;
}
```

> 这也是一个测试

~~代码高亮~~

***

什么\
你说
````



xlsx导入 xlsx + file-saver 表格导出





微信登录 

材料库 成品库 

材料设计：  多少钱一单位 单位为面积或者数量

成平库：普通成品、非标成平(从材料中组)   只有一个价格，按数量算

报价：从成平库中选择/重新组价



材料：

* 标准规格

长 5米    标准价格 50块

宽 1米

* 单价

10元   单位：平方米

公式：标准价格/长*宽    可选择公式   expr-eval



成品：

xxx   规格：x单位 y单位   价格  x单位

公式：

面积=长*宽

选择材料

1. xx  单价：10 元/平方米  用量： x平方米 (扩展按钮)

   ​		       用量： x*2平方米

2. yy  单价：10 元/个  用量： y个





# 其他markdown组件

静态站点生成：https://vuepress.vuejs.org/zh/guide/introduction.html

编辑器：https://tiptap.dev/

编辑器：(左右分屏显示)http://editor.md.ipandao.com/

编辑器：(左右分屏)https://imzbf.github.io/md-editor-v3/zh-CN

富文本编辑器：https://www.wangeditor.com/

