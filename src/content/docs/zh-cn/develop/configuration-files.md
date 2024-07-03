---
title: 配置文件
sidebar:
  order: 1
  badge:
    text: WIP (1.x)
    variant: caution
---

因为 Tauri 是一个构建 app 的工具包, 因此可能有许多文件用于配置项目设置.
一些常见的文件包括 `tauri.conf.json`, `package.json` 和 `Cargo.toml`.
我们在本页简要地介绍每个文件, 以帮助你确定应该修改哪个文件.

## Tauri 配置

文件可以是 `tauri.conf.json`, `tauri.conf.json5`, 或者 `Tauri.toml`.
默认是 `tauri.conf.json`. 更多信息, 查看下面的说明

这是 Tauri 进程使用的文件. 你可以定义 build 设置(例如 [在 `tauri build` 运行之前的命令][before-build-command] 或者 [`tauri dev`][before-dev-command]),
设置[app 名称](/reference/config/#productname), [app 版本](/reference/config/#version), [控制Tauri 进程][/reference/config/#configuration-structure], 并且配置插件设置. 你可以在 [`tauri.conf.json` api 参考]中找到所有选项

:::note

Tauri 配置文件默认格式是 `.json`. `.json5` 或者 `.toml` 格式可以通过在 `Cargo.toml` 文件中的 `tauri` 和 `tauri-build` 依赖中分别添加 `config-json5` 或者 `config-toml` features 进行启用.
需要注意的是, `.toml` 格式仅在 Tauri 1.1 版本及以上可用

```toml title=Cargo.toml
[build-dependencies]
// highlight-next-line
tauri-build = { version = "1.0.0", features = [ "config-json5" ] }

[dependencies]
serde_json = "1.0"
serde = { version = "1.0", features = ["derive"] }
// highlight-next-line
tauri = { version = "1.0.0", features = [ "api-all", "config-json5" ] }
```

所有格式的结构和值都是相同的, 但是格式应于各自文件的格式保持一致.

:::

## `Cargo.toml`

Cargo 的 manifest 文件用来声明你的 app 依赖的 Rust crates, 你的 app 的 metadata, 以及其他与Rust相关的 features.
如果你不打算在你的 app 中使用Rust进行后端开发, 那么你可能不会对它做太多的修改, 但是重要的是知道它的存在以及它的作用.

下面是一个针对 Tauri 项目的基本 `Cargo.toml` 文件示例:

```toml title=Cargo.toml
[package]
name = "app"
version = "0.1.0"
description = "A Tauri App"
authors = ["you"]
license = ""
repository = ""
default-run = "app"
edition = "2021"
rust-version = "1.57"

[build-dependencies]
tauri-build = { version = "1.0.0" }

[dependencies]
serde_json = "1.0"
serde = { version = "1.0", features = ["derive"] }
tauri = { version = "1.0.0", features = [ "api-all" ] }

[features]
# by default Tauri runs in production mode
# when `tauri dev` runs it is executed with `cargo run --no-default-features` if `devPath` is an URL
default = [ "custom-protocol" ]
# this feature is used for production builds where `devPath` points to the filesystem
# DO NOT remove this
custom-protocol = [ "tauri/custom-protocol" ]
```

最重要的部分是要注意 `tauri-build` 和 `tauri` 依赖.
通常, 它们都必须是 Tauri CLI 的最新小版本, 但这并不是严格要求的.
如果你在尝试运行 app 时遇到问题, 你应该检查 Tauri 版本(`tauri` 和 `tauri-cli`)是不是它们各自最新的小版本

Cargo 使用[语义化版本(Semantic Versioning)][Semantic Versioning]进行版本号管理.
运行 `cargo update` 将会拉取所有依赖的最新可用 Semver 版本.
例如, 如果你指定 `1.0.0` 作为 `tauri-build` 的版本, Cargo 将会检测并下载 `1.0.4` 版本, 因为它是最新的 Semver 版本.
当 Tauri 引入破坏性变更时, 主版本号会更新, 这意味着你始终能够安全的升级到最新的次版本和补丁版本, 而不用担心代码被破坏.

如果你想要使用一个特殊的 crate 版本, 你可以通过再依赖项的版本号前面加上 `=` 号来指定精确的版本:

```
tauri-build = { version = "=1.0.0" }
```

一个额外需要注意的地方是, tauri 依赖中的 `features=[]` 部分.
运行 `tauri dev` 和 `tauri build` 将根据你在 `tauri.conf.json` 中设置的 `"allowlist"` 属性自动管理需要在你的项目中启用的 features

当你构建你的 app 的时候, 会生成一个 `Cargo.lock` 文件.
这个文件主要用于确保在跨多台机器使用相同的依赖版本(类似于 Node.js 中的 `yarn.loc` 或者 `package-lock.json`).
因为你开发的是一个 Tauri app, 所以应该将这个文件提交到你的源代码仓库中(只有Rust库才应该忽略提交这个文件)

要了解更多关于 `Cargo.toml`, 你可以详细读一下 [官方文档][cargo-manifest]

## `package.json`

这是 Node.js 使用的包文件.
如果这个 Tauri app 的前端使用基于Node.js的技术开发的(例如 `npm`, `yarn` 或者 `pnpm`), 这个文件用于配置前端依赖和脚本(scripts)

一个针对 Tauri 项目的基本 `package.json` 文件示例可能如下所示:

```json title=package.json
{
  "scripts": {
    "dev": "command-for-your-framework",
    "tauri": "tauri"
  },
  "dependencies": {
    "@tauri-apps/api": "^1.0",
    "@tauri-apps/cli": "^1.0"
  }
}
```

通常会使用 `"scripts"` 部分配置启动 Tauri app 前端的命令, 上面的文件指定了 `dev` 命令, 你可以使用 `yarn dev` 或者 `npm run dev` 启动前端框架

The dependencies object specifies which dependencies Node.js should download when you run either `yarn` or `npm install` (in this case the Tauri CLI and API).

dependencies 对象指定依赖项, 当你运行 `yarn` 或者 `npm install` 的时候 Node.js 会下载依赖项(本例中为 Tauri CLI 和 API)

除了 `package.json` 文件, 你可能还会看到 `yarn.lock` 或者 `package-lock.json` 文件.
这些文件确保以后当你下载依赖的时候, 你能获得和开发过程中完全相同的版本(类似于 Rust 中的 `Cargo.lock` 文件)

要了解更多关于 `package.json` 的信息, 你可以阅读 [官方文档][npm-package]

[`tauri.conf.json` api 参考]: /reference/config/
[before-build-command]: /reference/config/#beforebuildcommand
[semantic versioning]: https://semver.org
[cargo-manifest]: https://doc.rust-lang.org/cargo/reference/manifest.html
[npm-package]: https://docs.npmjs.com/cli/v8/configuring-npm/package-json
[before-dev-command]: /reference/config/#beforedevcommand-1
