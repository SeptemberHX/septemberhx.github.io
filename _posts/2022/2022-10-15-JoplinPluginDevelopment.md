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

## Markdown 编辑器插件开发

TODO

## Markdown 渲染插件开发

## 独立 Panel 插件开发

## 数据库相关开发
