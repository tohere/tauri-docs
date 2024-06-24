---
title: 隔离模式
sidebar:
  badge:
    text: WIP
    variant: caution
---

隔离模式一种在前端发送的 Tauri API 消息到达 Tauri 核心之前拦截并修改这些信息的方法, 所有操作都通过 JavaScript实现, 通过隔离模式注入的安全 JavaScript 代码被称为隔离应用程序.

## 为什么

隔离模式的目的是为开发者提供一种机制, 帮助保护他们的应用程序免受对 Tauri 核心的不必要或恶意的前端调用. 隔离模式的需求来自于前端运行的不可信内容的威胁, 这在依赖较多的应用程序中是常见的情况. 请参阅 [安全: 威胁模型], 以获取应用程序可能遇到的威胁来源的列表.

隔离模式设计时考虑的最大威胁模型是开发过程中的威胁. 许多前端构建工具在构建时通常包含十几个甚至数百个深度嵌套的依赖项, 而复杂的应用程序可能还会有大量(同样经常是深度嵌套的)依赖项, 这些依赖项会被捆绑到最终的输出中.

## 什么时候

Tauri 强烈建议尽可能使用隔离模式, 因为隔离模式可以拦截所有来自前端的消息, 所以它始终可以使用

Tauri 也强烈建议在使用外部 Tauri API 锁定你的应用, 作为开发者, 你可以使用安全隔离应用来尝试并验证 IPC 输入, 以确保它们是期望的参数. 例如, 你可能想要检查读取或写入文件的调用是否试图访问应用程序预期范围之外的路径. 另一个例子是, 确保 Tauri API HTTP fetch 调用仅仅只是设置了你的应用期望的header头

也就是说, 它拦截了所有来自前端的消息, 因此它甚至能和 [Events] 等 在线 API 一起工作.由于某些事件可能会触发你自己的 Rust 代码执行操作, 因此可以使用相同的验证技术来处理它们.

## 怎么做

隔离模式的核心是在你的前端和 Tauri 核心之间注入一个安全的应用程序, 以拦截和修改进来的 IPC 消息. 它通过使用 `<iframe>` 的沙箱功能, 与主前端应用程序一起安全地运行JavaScript来实现这一点. Tauri 在加载页面时强制使用隔离模式, 强制对所有 Tauri 核心的 IPC 调用首先通过沙盒隔离应用程序进行路由. 一旦消息准备好传递给 Tauri 核心, 他就会使用浏览器的 [SubtleCrypto] 实现进行加密, 然后传递回主前端应用程序. 一旦到达住前端应用程序, 消息会直接传递给 Tauri 核心, 在那里像正常情况一样解密和读取.

为了确保别人不能手动读取应用程序特定版本的密钥, 并用其修改加密后的消息, 每次运行应用程序时都会生成新的密钥.

### IPC 消息的大致步骤

为了便于理解, 这里列出了使用隔离模式发送到 Tauri Core 的 IPC 消息的大致步骤:

1. Tauri 的 IPC 处理程序收到一条消息
2. IPC 处理程序 -> 隔离应用程序
3. `[沙箱]` 隔离应用程序钩子运行, 并可能修改消息
4. `[沙箱]` 使用运行时生成的密钥, 用 AES-GCM 对消息进行加密
5. `[加密后]` 隔离应用程序 -> IPC 处理程序
6. `[加密后]` IPC 处理程序 -> Tauri Core

_注意: 箭头 (->) 表示消息传递._

### 性能影响

由于消息被加密, 相对于 [棕地模式], 即使安全的隔离模式什么都不做, 这里也产生了额外的开销. 除了对性能敏感的应用程序(这些应用程序很可能有一套精心维护的小型依赖关系, 以保持足够的性能), 大多数应用应该不会注意到 IPC 消息加密/解密的运行时成本, 因为这些成本相对较小, 并且 AES-GCM 相对较快. 如果你不熟悉 AES-GCM, 在这个上下文中唯一相关的事, 它是 [SubtleCrypto] 中包含的唯一认证模式算法, 而且你可能每天都在 [TLS] 下使用它

每次启动Tauri应用程序时, 都会生成一个密码学安全的密钥, 如果系统已经有足够的熵来, 可以立即返回足够的随机数, 那么一般不会引起注意, 这在桌面环境中极为常见. 如果在无头环境中运行, 以进行一些 [WebDriver 集成测试], 那么如果你的操作系统中没有此类服务, 你可能需要安装某种生成熵的服务, 例如 `haveged` <sup> Linux 5.6（2020 年 3 月）现在包括使用推测执行的熵生成功能 </sup>

### 局限性

因为平台的不一致性, 隔离模式存在一些限制. 最主要的限制是外部文件无法再 Windows 沙箱化 `<iframes>` 中正确加载. 因此, 我们在构建时实现了一个简单的脚本内联步骤, 它获取与隔离应用程序相关的脚本内容, 并将其内联注入. 这意味着像 `<script src="index.js"></script>` 这样的典型打包或简单包含文件仍然可以正常工作, 但像 ES Modules 这样的新机制将无法成功加载.

## 建议

由于隔离应用程序的目的是防止开发威胁, 我们强烈建议尽可能简化你的隔离应用程序. 不仅应该尽量保持依赖项最少, 还应该考虑保持所需的构建步骤最少. 这样, 你就不需要担心除了前端主应用程序之外, 对隔离应用程序的供应链攻击

## 创建隔离应用程序

在这个例子中, 我们将创建一个小型的 hello-world 风格的隔离应用程序, 并将其连接到一个假想的现有 Tauri 应用程序. 它不会对传递的消息进行验证, 仅仅将内容打印到 webview 控制台

在本例中, 假设我们位于 `tauri.conf.json` 文件所在的目录中, 现在Tauri的应用程序将其 `distDir` 设置为 `../dist`

`../dist-isolation/index.html` :

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <title>Isolation Secure Script</title>
  </head>

  <body>
    <script src="index.js"></script>
  </body>
</html>
```

`../dist-isolation/index.js` :

```js
window.__TAURI_ISOLATION_HOOK__ = (payload) => {
  // let's not verify or modify anything, just print the content from the hook
  console.log('hook', payload);
  return payload;
};
```

现在, 我们需要做的是设置我们的 `tauri.conf.json` [配置](#configuration) 使用隔离模式, 并从[棕地模式]切换到隔离模式

## 配置

我们假设我们的主前端 `distDir` 设置的事 `../dist`. 我们也将我们的隔离应用程序设置为 `../dist-isolation`

```json
{
  "build": {
    "distDir": "../dist"
  },
  "tauri": {
    "pattern": {
      "use": "isolation",
      "options": {
        "dir": "../dist-isolation"
      }
    }
  }
}
```

[transport_layer_security]: https://en.wikipedia.org/wiki/Transport_Layer_Security
[安全: 威胁模型]: /security/lifecycle/
[events]: /reference/javascript/api/namespaceevent/
[subtlecrypto]: https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto
[棕地模式]: /concept/inter-process-communication/brownfield/
[WebDriver 集成测试]: /develop/tests/webdriver/
