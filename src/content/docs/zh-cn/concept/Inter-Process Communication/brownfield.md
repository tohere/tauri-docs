---
title: 棕地模式
sidebar:
  badge:
    text: WIP
    variant: caution
---

_**这是默认的模式**_

> 什么是棕地模式?
>
> 棕地模式指的是在现有软件或系统的基础上进行开发, 而不是从头开始构建全新的系统

这是使用 Tauri 最简单直接的模式, 因为它尽可能的与现有的前端项目箭筒, 简而言之, 它尽量不需要任何额外的东西, 而是使用现有的网页前端项目, 并不是所有在现有浏览器应用中工作的东西都能直接在 Tauri 中使用

如果你对宗地软件开发不熟悉, 可以查阅 [维基百科中关于棕地模式的文章], 这里有一个很好的总结. 对 Tauri 来说,现有软件指的是当前浏览器的支持和行为, 而不是传统系统

## Configuration

因为 棕地模式 是默认的模式, 不需要设置配置选项. 如果要显示设置它, 你可以在 `tauri.conf.json` 配置文件中使用 `tauri > pattern` 对象

```json
{
  "tauri": {
    "pattern": {
      "use": "brownfield"
    }
  }
}
```

_**棕地模式没有额外的配置选项**_

[维基百科中关于棕地模式的文章]: https://en.wikipedia.org/wiki/Brownfield_(software_development)
