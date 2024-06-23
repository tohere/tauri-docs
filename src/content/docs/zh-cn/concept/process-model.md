---
title: 进程模型
sidebar:
  order: 0
  badge:
    text: WIP
    variant: caution
---

Tauri 采用类似于 Electron 和 很多现代 web 浏览器的多进程架构, 本指南探讨了这种设计选择背后的原因, 以及为什么为什么它对编写安全应用程序至关重要

## 为什么使用多个进程?

在 GUI 应用的早期阶段, 通常使用单一进行来执行计算, 绘制界面和响应用户的输入. 正如你可能猜想的那样, 这意味着长时间的运行, 高昂的计算成本会导致用户界面无响应, 或者更糟糕的是, 一个应用程序组件的失败会导致整个应用程序崩溃.

很明显, 需要一种更具弹性的架构, 应用程序开始在不同的进程中运行不能的组件. 这样可以更好地利用现代多核 CPU, 并且创建更安全的应用程序, 一个组件的崩溃不会再影响整个系统, 因为各个组件被隔离在不同的进程中. 如果一个进程进入无效状态, 我们可以很轻松地重新启动它.

我们还可以通过仅分配每个进程所需的最小权限来限制潜在攻击的影响范围, 确保它们能够完成工作即可. 这种模式被称为 [最小权限原则], 在现实世界中你也会经常看到. 如果你请园丁修剪你的篱笆, 你会给他们花园的钥匙, 你不会给他们房子的钥匙. 为什么他们需要这个权限? 相同的概念同样适用于计算机程序. 我们给它们更小的权限, 它们能造成的危害也就越小

## 核心进程

每个 Tauri 应用都有一个核心进程, 它作为应用程序的入口点, 并且是唯一具有对操作系统完全访问权限的组件.

这个核心的主要责任是利用这种访问权限来创建和协调应用程序窗口, 系统托盘菜单, 或者通知. Tauri实现了必要的跨平台抽象, 使这一过程变得简单, 它还通过核心进程列出了所有的 [进程间通信], 允许你在一个中心位置拦截, 过滤和操作 IPC 通信

核心进程还应负责管理全局状态, 例如设置或数据库连接. 这允许你在窗口之间很轻松的进行状态同步, 并且保护前端中的业务敏感数据免受窥视.

我们选择 Rust 实现 Tauri 的原因是因为它的 [所有权] 概念在确保内存安全的同时还能保证优秀的性能

<figure>

```d2 sketch pad=50
direction: right

Core: {
  shape: diamond
}

"Events & Commands 1": {
  WebView1: WebView
}

"Events & Commands 2": {
  WebView2: WebView
}

"Events & Commands 3": {
  WebView3: WebView
}

Core -> "Events & Commands 1"{style.animated: true}
Core -> "Events & Commands 2"{style.animated: true}
Core -> "Events & Commands 3"{style.animated: true}

"Events & Commands 1" -> WebView1{style.animated: true}
"Events & Commands 2" -> WebView2{style.animated: true}
"Events & Commands 3" -> WebView3{style.animated: true}
```

<figcaption>Tauri 进程模型的简化表示, 一个核心进程管理一个或者多个 webview 进程</figcaption>
</figure>

## WebView 进程

核心进程本身不会渲染实际的用户界面(UI), 它启动利用操作系统提供的 webview 库的 webview 进程. webview 是一个类似浏览器的环境, 用于执行你的 HTML, CSS, JavaScript

这意味着可以使用传统 web 开发中的大多数技术和工具来创建 Tauri 应用程序. 例如, 很多 Tauri 示例是使用 [Svelte] 前端框架和 [Vite] 打包工具编写

安全最佳实践同样适用, 例如, 你必须始终对用户输入进行处理, 永远不要在前端处理机密信息, 尽可能将业务逻辑推迟到核心进程中, 以减小攻击面.

和其他类似的解决方案不同, webview 库不会包含在最终的可执行文件中, 而是在运行时动态链接[^1]. 这使得你的应用体积显著减小, 但是这也意味着你需要像传统的网页开发平台一样, 考虑到平台差异

[^1]: 目前, Tauri 在 Windows 上使用 [Microsoft Edge WebView2], 在 macOS 上使用 [WKWebView], 在 Linux 上使用 [webkitgtk]

[最小权限原则]: https://en.wikipedia.org/wiki/Principle_of_least_privilege
[进程间通信]: /concept/inter-process-communication/
[所有权]: https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html
[microsoft edge webview2]: https://docs.microsoft.com/en-us/microsoft-edge/webview2/
[wkwebview]: https://developer.apple.com/documentation/webkit/wkwebview
[webkitgtk]: https://webkitgtk.org
[svelte]: https://svelte.dev/
[vite]: https://vitejs.dev/
