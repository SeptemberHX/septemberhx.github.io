---
Toc: true
comments: true
title: Joplin插件开发教程
Date: 2022-10-15 16:18:19 +0800
categories: [手册, 开发]
---

[Joplin](https://joplinapp.org/) 是一个 Markdown 笔记工具，相比于 [Obsidian](https://obsidian.md/) 而言，Joplin 是通过关系型数据库存储所有的笔记、信息等，而 Obsidian 是基于文件的。

## 技术要求

1. 会写代码，会开发
2. 能够读懂英语，需要阅读官方 API 文档
3. 熟悉 JavaScript 的使用：**但不是必须得**，使用过其他相似的语言也可以。在开发 [Enhancement 插件](https://github.com/SeptemberHX/joplin-plugin-enhancement) 之前，我也不太熟悉 JS
4. 熟悉 Typescript 的使用：同上，不是必须的
5. 略懂 React、css：同上，不是必须的，仅在开发有 UI 类的插件需要。甚至可以不使用 React 开发，只是能让你的页面代码整洁不少

## 插件功能分类

在插件的最终作用对象上，可以大致分为以下四类。**一般情况下，需要多个类型的功能一起实现才能够提供良好的用户使用体验。**

1. Markdown 编辑器插件：比如编辑器中的自动补全相关的插件。任何和编辑器相关的功能，都通过该类实现。包括但不限于：
   1. **自动补全**：比如 NoteLinkSystem 插件通过 `@@` 即可触发自动补全，实现对其他笔记链接的快速插入。
   2. **编辑器的实时渲染**：比如 [Enhancement 插件](https://github.com/SeptemberHX/joplin-plugin-enhancement)。可以将 Mermaid, Math 等 Markdown 格式的代码直接显示出渲染后的结果。当实时渲染足够强大时，甚至可以不再需要额外的 PDF 预览窗口。
   3. **自定义样式**：和 Joplin 提供的自定义样式文件 `userchrome.css` 类似，但是通过代码级插件可以实现自定义元素的 class 类型，再配合 css 样式，即可实现基本对任意元素自定义样式。比如代码高亮、不同的 Markdown 主题、搜索关键词高亮等等。
   4. **对内容的修改**：一般与快捷键、工具栏等配合实现对内容的自动修改。比如高亮、加背景色等等。
   5. ...
2. Markdown 渲染插件：比如 [Admonition 插件](https://github.com/maxnegro/joplin-plugin-admonition)。该部分负责修改 Markdown 文本转为 HTML 的过程。包括但不限于：
   1. **自定义 Markdown 语法渲染**：比如 Admonition 插件能够将 `!!!note` 渲染成更醒目的格式。
   2. **修改现有 Markdown 渲染结果**：比如 Enhancement 插件能够将 `![title]()` 渲染为带有图标题的结果。
3. 带有独立 Panel 的信息收集、展示、交互等：比如 [NoteLink 插件](https://github.com/ylc395/joplin-plugin-note-link-system)。
4. Joplin 数据库相关插件：比如 BackUp 插件。修改笔记的文件夹路径、标题，新建/删除笔记等等。

![不同插件功能的作用区域](/assets/img/JoplinPluginDevelopment/pluginFunctionCategory.png)

## 插件开发初始化

### 初始化工具

Joplin 提供了插件开发开始时的初始化工具 `generaor-joplin`。具体细节请参考官方网站：[Getting started with plugin development](https://joplinapp.org/api/get_started/plugins/)。

1. 安装 `Node.js`
2. `npm install -g yo generator-joplin`
3. 在任意文件夹内，执行 `yo joplin` 完成初始化操作。

### 文件结构

里面有几个比较关键的文件：

1. `plugin.config.json`：里面的 `extraScripts` 用于存放需要转换成 `*.js` 的 `*.ts` 以及 `*.tsx` 文件路径。**所有的路径都是相对于 `./src` 目录而言！
   * 插件开发时，Joplin 的插件接口有时候需要直接提供 `*.js` 文件路径，但是一般情况下都是用 `*.ts` 或者 `*.tsx` 实现的，这种情况下就需要在这个文件中配置一下，不然编译时无法找到对应的 `*.js` 文件。其它 `*.ts` 文件之间相互引用的情况则无需在这个文件中配置。
2. `./src/index.ts`：插件的入口，初始化后一般包含以下格式，只替换中间的部分即可。
3. `./api`：这个是包含在初始文件中的、配合 Joplin 插件接口、函数、类等，最好**不要修改**。

```typescript
joplin.plugins.register({
	onStart: async function() {
      // 从这里开始写你的插件逻辑
   }
})
```
{: file='./src/index.ts'}

### 注册插件

```typescript
await joplin.contentScripts.register(
   ContentScriptType.MarkdownItPlugin,  // 或者是 ContentScriptType.CodeMirrorPlugin, 分别对应 markdown 渲染插件以及 markdown 编辑器插件
   '唯一的插件ID',
   './插件的路径/index.js'  // 这里的文件如果是用 ts 写的，那么需要再 plugin.config.json 文件中配置
);
```
{: file='./src/index.ts'}

更细致的内容请参考：[joplin开发文档](https://joplinapp.org/api/references/plugin_api/enums/contentscripttype.html)


## Markdown 编辑器插件开发

截止到目前，Joplin 桌面端 的 Markdown 编辑器是基于 [Codemirror 5](https://codemirror.net/5/doc/manual.html) 开发的，目前还没有听到要切换到 [Codemirror 6](https://codemirror.net/) 的消息（Obsidian 等用的就是 v6，v6 相较于 v5 而言新增了一些功能，能够更方便的创建 decoration 等，实现 live preview 功能）。安卓端的 Joplin 使用的则是 v6 版本，但是目前安卓版不支持插件，因此这里仅讨论 v5 版本下的插件开发。

`index.ts` 文件的主体内容：

```typescript
module.exports = {
   default: function(context) {
      return {
         plugin: function(CodeMirror) {
            // 插件逻辑部分

            // 这里是通过 defineOption 方式实现 cm 插件
            CodeMirror.defineOption('option的唯一ID', false, async function(cm, val, old) {
               // xxx
            };

            // 与 defineOption 不同，这里更像是给 codemirror 对象新增了一个函数，defineExtension 之后，可以用 cm.extensionID(paras) 的方式调用了
            // 示例见：https://github.com/CalebJohn/joplin-rich-markdown/blob/434520a5ba3ae4b345fbc88d2679969fcf910adf/src/richMarkdown.ts#L37
            // 与此同时，也可以通过 Joplin 的 command 进行调用，见 https://github.com/SeptemberHX/joplin-plugin-enhancement/blob/master/src/driver/codemirror/commands/index.ts 及 https://github.com/SeptemberHX/joplin-plugin-enhancement/blob/dec1ba2055270c791bacc6df70011f75afa55879/src/index.ts#L86
            CodeMirror.defineExtension('extension的唯一ID', function(paras) {
               // xxx
            });
         },
         codeMirrorResources: [
            // cm5 自带的 addons，cm5 本身就自带了不少的插件，比如搜索高亮等
            // 具体参考：https://codemirror.net/5/doc/manual.html#addons
            // 下面的例子对应 mode/overlay.js 以及 hint/show-hint.js
            'addon/mode/overlay',
            'addon/hint/show-hint'
         ],
         codeMirrorOptions: {
            // 必须在这里进行配置开启，否则无法生效
            '对应上面 option的唯一ID': true,
         },
         assets: {
            // 配置 css 样式文件。Joplin 中 cm5 不允许携带 *.js 脚本
         },
      }
   }
}
```
{: file='./src/具体插件功能/index.ts'}

由于 Joplin 设计上不允许插件进程直接与界面交互，需要通过消息通信实现，codemirror 也是如此。见：[示例](https://github.com/SeptemberHX/joplin-plugin-enhancement/blob/dec1ba2055270c791bacc6df70011f75afa55879/src/index.ts#L86)

剩下的 codemirror 开发相关内容参见[官方文档](https://codemirror.net/5/doc/manual.html)，这里给出几个例子：
1. [通过 command 从插件进程获取 codemirror 数据](https://github.com/SeptemberHX/joplin-plugin-enhancement/blob/dec1ba2055270c791bacc6df70011f75afa55879/src/index.ts#L86)
2. [为多行的特定格式内容分配特定 class，配合 css 实现丰富的渲染效果（如 Admonition）](https://github.com/SeptemberHX/joplin-plugin-enhancement/blob/dec1ba2055270c791bacc6df70011f75afa55879/src/driver/codemirror/admonition/index.ts)
3. [单行的特定格式分配特定 class，配合 css 实现丰富的渲染效果（如 `==高亮==`）](https://github.com/SeptemberHX/joplin-plugin-enhancement/blob/dec1ba2055270c791bacc6df70011f75afa55879/src/driver/codemirror/admonition/index.ts)
4. [将特定格式内容渲染成其他内容，如数学公式 `$latex$`](https://github.com/SeptemberHX/joplin-plugin-enhancement/blob/dec1ba2055270c791bacc6df70011f75afa55879/src/driver/codemirror/linkFolder/mathMaker.ts)
5. [选中文字后悬浮的工具栏](https://github.com/SeptemberHX/joplin-plugin-enhancement/blob/dec1ba2055270c791bacc6df70011f75afa55879/src/driver/codemirror/formattingBar/formattingBart.ts)
6. [编辑过程中动态校验列表编号并自动修正](https://github.com/SeptemberHX/joplin-plugin-enhancement/blob/dec1ba2055270c791bacc6df70011f75afa55879/src/driver/codemirror/listNumber/index.ts)

## Markdown 渲染插件开发

该类插件加载时，插件类型应设定为 `ContentScriptType.MarkdownItPlugin`，对应的插件 Index.ts 为：

```typescript
export default function (context) {
    return {
        plugin: function (markdownIt, _options) {
            const pluginId = context.pluginId;

            xxx(markdownIt, _options);
        },
        assets: function() {
            return [
                { name: 'xxx.css' }
            ];
        },
    }
}
```
{: file='./src/具体插件功能/index.ts'}

上一小节是编辑器开发，是 Joplin 左侧的编辑部分，而渲染插件则是针对 markdown 渲染成 PDF 过程的，通过对 markdown 渲染规则/方法的修改、覆写、新增等操作，可以自由实现自己想要达到的效果，例如将 `- [ ] xxxx +tag1` 中的 `+tag1` 渲染上背景色等。

Joplin 用的是 markdown-it 进行 markdown 渲染，所以其实就是 markdown-it 相关插件的开发。我并没有详细了解过 markdown-it 的工作原理，猜测应该是先由 `md.block.ruler` 对文本进行语法解析，将原始文本按照 markdown 的规则划分为多类 token，然后再由 `markdownIt.renderer.rules` 对标记后的 token 进行渲染，完成从文本到 HTML 的转换。该过程仅仅是猜测，欢迎指正。

目前我用到的就两种：
1. 针对 `md.block.ruler`，实现自定义的 token 识别，如 [Front Matter 的 token 识别](https://github.com/SeptemberHX/joplin-plugin-enhancement/blob/master/src/driver/markdownItRuler/frontMatter/index.ts)
2. 针对 `markdownIt.renderer.rules`，实现对现有 markdown 元素渲染过程的修改，例如：[伪代码块渲染](https://github.com/SeptemberHX/joplin-plugin-enhancement/blob/master/src/driver/markdownItRenderer/pseudocode/pseudocode.ts)

## 独立 Panel 插件开发

## 数据库相关开发

## 注意事项

### 第三方库的使用

由于 Joplin 需要跨平台，它自带了一套自己的 `fs` 以及 `sqlite3` 库，需要通过以下方式引入包，不能直接 `require`：

```typescript
const fs = joplin.require('fs-extra')
const sqlite3 = joplin.require('sqlite3')
```

这进一步导致所有依赖了 `fs` 以及 `sqlite3` 的库都无法正常使用！
