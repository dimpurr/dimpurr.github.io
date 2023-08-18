---
title: 使用 lerna 配合 npm workspace 进行大型全栈项目的 monorepo 包管理
date: 2023-08-18 23:10:13
tags:
---


npm workspaces 是 npm v7 中引入的一个新功能，允许开发者在一个单一的顶层项目中管理多个子项目或包。其主要特点包括：

- 使得多个子项目能够共享一个 node_modules，从而提高安装效率。
- 提供了本地依赖链接，使得在子项目之间引用变得容易。

尽管 npm workspaces 提供了一些相同的功能，Lerna 仍然有其独特的价值和使用场景。下面是一些使用 Lerna 的原因：

- 更复杂的发布流程: Lerna 是为管理 monorepo 并自动处理包版本和发布流程而构建的。通过 lerna publish，它可以智能地确定哪些包发生了变化，为它们版本化，并逐一发布。它还可以处理跨包的版本依赖。
- 版本管理策略: Lerna 允许您选择固定或独立的版本管理策略。在固定模式下，所有包共享一个版本号。而在独立模式下，每个包可以有自己的版本号。
- 跨包脚本执行: lerna run 和 lerna exec 允许您在所有子包或特定的子包上执行脚本和命令。这使得执行跨包任务变得容易。

### 新建项目

1. 初始化项目和配置 Lerna

首先，使用 lerna init 命令初始化 Lerna。此命令将创建一个 lerna.json 文件，其中包含 Lerna 的配置。

2. 启用 npm workspaces

在项目的顶层 package.json 文件中，添加 workspaces 字段并配置子项目的路径，例如：

```json
{
  "name": "my-project",
  "workspaces": ["packages/*"]
}
```

### 创建和管理包

创建新的 Lerna 包的流程是直接的。下面是如何使用 Lerna 创建新包的步骤：

进入你的 packages 目录 (或者你在 lerna.json 中定义的其他目录)，然后手动创建一个新的目录为你的包：

```bash
cd packages
mkdir my-new-package
cd my-new-package
npm init
yarn add some-dependency
lerna add some-dependency --scope=my-new-package
```

上述命令将 some-dependency 添加到 my-new-package。使用 --scope 参数确保只有特定的包接收该依赖。

如果您的新包需要使用存储库中的其他 Lerna 包，您可以使用以下命令确保它们被正确链接：

```bash
lerna bootstrap
```

这会确保所有的内部依赖都被正确地链接。

### lerna add 和 yarn add 的区别

lerna add 和 yarn add 都是命令行工具中的命令，用于添加依赖到项目中，但它们的用途和上下文存在一些不同：

作用范围：

- lerna add: 可以更灵活地为 monorepo 中的特定包或所有包添加依赖。例如，使用 --scope 选项，你可以将一个依赖添加到特定的 Lerna 管理的包中。
- yarn add: 只会将依赖添加到当前工作目录下的项目或包中。

依赖链接：

- lerna add: 当你使用 lerna add 将一个存储库中的 Lerna 包添加为另一个 Lerna 包的依赖时，Lerna 会确保它们在本地正确地链接在一起，而不是从 npm registry 下载。
- yarn add: 默认情况下，它会从 npm registry 或设定的 registry 下载依赖。

### 其他常见工作流和命令

1. 使用 lerna add 和 npm install 为特定的包或所有包添加依赖。
2. 使用 lerna run 执行跨包脚本。
3. 使用 lerna publish 和 npm workspaces 的结合来发布包。