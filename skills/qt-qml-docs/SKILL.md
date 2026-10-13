---
name: qt-qml-docs
description: >-
  为 QML 组件和应用生成独立的 Markdown 参考文档。用户需要记录 QML 文件、创建 QML 组件或模块的 API 参考、记录 Qt Quick 应用，或根据 .qml 源码生成面向开发者的文档时使用。触发词包括“document this QML”“write docs for my QML”“create reference docs”“document QML component”“QML API docs”“document my Qt Quick component”“document my Qt app”，以及提供一个或多个 .qml 文件并需要文档的情况。支持单文件、粘贴代码和整个工程目录；若用户要求 QDoc 格式则不要使用。
license: LicenseRef-Qt-Commercial OR BSD-3-Clause
compatibility: >-
  面向 Claude Code、GitHub Copilot 及类似 agent。
disable-model-invocation: false
metadata:
  author: qt-ai-skills
  version: "1.0"
  qt-version: "6.x"
  category: process
---

# QML 文档技能

## 中文概要

把 QML 组件、模块或 Qt Quick 应用整理成独立 Markdown 参考文档；不输出 QDoc 格式。

你是 Qt/QML 专家，负责为 QML 组件编写清晰、准确、便于开发者使用的参考文档。读取 QML 源文件及相关文件（C++ backend、QML module、资源文件、CMakeLists.txt、qmldir 等），生成结构化 Markdown 参考文档，让开发者完整了解组件在项目中的位置和作用。

## 核心要求

- **除 Usage Example 外不写代码片段。** 不要使用 Markdown 代码围栏包裹代码；可复用组件的 Usage Example 部分（第 8 节）除外。
- **结合项目上下文。** 理解应用或模块的用途、组件扮演的角色以及组件依赖的内容。
- **属性使用表格记录。** 始终使用 Markdown 表格记录属性，不要使用项目符号列表。
- **遵循项目约定。** 从项目文档或代码模式中推断并遵守 QML 开发约定。

## 文档结构

为每个 QML 组件生成名为 ComponentName.md 的 Markdown 文件。没有内容的章节可以省略。

### 1. 组件概览

说明应用或模块的用途，以及组件在项目架构中的位置。然后解释组件自身的功能、视觉或逻辑角色、适用场景和解决的问题。保持简洁，让刚接触代码库的开发者一眼理解组件用途。

### 2. 项目结构与依赖

说明组件与项目的关系：

- 哪些文件导入或实例化它？
- 它导入什么内容（Qt Quick modules、项目自定义 QML types、C++ registered types）？
- 对于 custom QML types，说明它们提供什么以及来源在哪里。
- 记录相关构建或模块要求，例如 CMake targets、qmldir、qmltypes。

### 3. 组件层级与角色

如果组件继承或组合了其他元素，说明其层级关系，并解释 base type 提供了什么、当前组件新增或覆盖了什么。

### 4. 属性

使用包含以下列的 Markdown 表格：

| 属性 | 类型 | 默认值 | 必填 | 描述 |
|----------|------|---------|----------|-------------|

- 列出所有声明的属性，包括 property alias。
- 对 required property，将 Required 列标记为 Yes。
- 从属性控制什么或启用什么的角度描述每个属性。
- 对接受固定值集合的属性（枚举、字符串字面量），列出有效值及其含义。

### 5. 信号

对于每个 signal：

- 说明信号名和参数列表，包括每个参数的类型与名称。
- 解释信号由什么条件触发。
- 描述连接的 handler 收到信号后应如何响应。

每个信号使用一个子章节，格式为 signalName(paramType paramName)。

### 6. 方法

对于每个 function：

- 说明函数名、参数名称与类型，以及返回类型（如有）。
- 解释函数功能和调用时机。
- 记录副作用，例如发射 signal、修改状态或重启 timer。

每个方法使用一个子章节，格式为 methodName(paramType paramName) : returnType。

### 7. 组件间交互

描述组件如何与应用其他部分通信：

- 哪些属性由外部 binding 驱动？
- 哪些 signal 被 parent 或 sibling components 消费？
- 哪些 function 从此文件外部调用？
- 它读取或写入哪些共享状态、model 或 singleton？

### 8. 使用示例（仅限可复用组件）

仅当组件是可复用的时才包含本节，也就是设计为由其他 QML 文件实例化，而不是作为独立应用入口。满足以下条件时，可认为组件可复用：

- root type 不是 Window 或 ApplicationWindow，它们是顶层应用窗口，不是可嵌入的组件。
- 声明了一个或多个调用方需要设置的 property，尤其是 required property 或 property alias。
- 组件用于组合更大的 UI，或作为代码库中的构建单元。

编写简短、自包含的片段，展示开发者实例化组件的最小正确方式，设置所有 required property 以及通常需要设置的属性。

---

## 前置检查：检查已有文档

读取任何源文件前，检查即将记录的文件是否已经有文档。这样可以节省时间，并让用户选择重新生成还是只更新。

### 检查方法

1. 确定预期输出位置。文档写入源文件旁边的 doc/ 子目录。例如源文件在 src/ 中，文档写入 src/doc/；单个文件 Foo.h 对应 src/doc/Foo.md，main.cpp 对应 src/doc/main.md。

2. 检查 doc/ 目录和相关 .md 文件是否已存在。使用 Glob 工具或执行 ls shell 命令，此时不要读取源文件。

3. 根据检查结果行动：

   - **未找到已有文档**——正常读取源文件并生成文档。
   - **部分或全部文档已存在**——暂时不要读取源文件，而是使用 AskUserQuestion 询问用户：

     > 我发现以下文件已有文档：[列出已有文档的文件]。你希望我怎么处理？
     >
     > 选项：
     > - **更新已有文档**——重新读取源文件，并原地重写受影响的 .md 文件。
     > - **跳过已有文档的文件**——只为缺少文档的源文件生成文档。
     > - **为全部文件重新生成**——无条件覆盖所有已有文档。
     > - **取消**——停止，不做任何修改。

   在继续其他操作前等待用户选择。

4. 遵守用户选择：

   - **更新**或**重新生成** → 读取所有相关源文件并正常执行，覆盖已有 .md 文件。
   - **跳过** → 只读取缺少对应 .md 的源文件，并仅为这些文件生成文档。
   - **取消** → 停止，并向用户确认没有修改任何内容。

## 输入处理

**单个文件或粘贴的代码：**只记录该组件。根据 imports、属性名和组件结构推断应用上下文。

**文件夹或工程：**遍历目录树，查找所有 .qml 文件。同时读取 CMakeLists.txt、qmldir 或 C++ 头文件，它们提供模块结构和注册类型的上下文。每个组件生成一个 .md。如果记录的文件超过一个，还要创建 doc/index.md，列出每个组件的一行说明和链接。

---

## 准确解析 QML

仔细读取源码：

- root element 是 base type，记录它继承的类型。
- property <type> <name>: <default> 是带可选默认值的 custom property。
- property alias <name>: <target> 是 alias，按 target 对应的类型记录。
- required property 表示组件使用者必须显式设置。
- signal <name>(<params>) 是 custom signal。
- function <name>(<params>) { } 是 JS function。
- readonly property 不能从外部设置，应标记为只读。
- component <Name> : BaseType { } 是 inline component definition，应在同一文件中作为独立组件记录。
- 以 _ 开头的内部 helper 通常是私有的，除非明显作为 public API，否则跳过。
- 如果属性缺少清晰说明，根据名称、类型和使用上下文推断有意义的描述。

## 语气和风格

- 面向了解 QML 但没看过该组件的开发者。
- 准确描述类型：string、int、real、color、bool、var、list<Type> 等。
- 使用现在时：“控制宽度……”而不是“将控制宽度……”。
- 避免填充语句，直接、清晰地描述。
- 描述行为而不是实现，解释发生了什么。
- 属性可接受值是固定集合时，始终在描述中列出所有值。

## 输出位置

- 在源 QML 文件旁边的 doc/ 子目录中生成文档。
- 仅当记录 2 个或更多组件时创建 doc/index.md。单文件文档只创建组件对应的 .md 文件。

## 质量检查

保存前确认：

- 每个 property、signal 和 function 都已记录，没有静默跳过。
- 只要存在可观察的 binding 或外部调用，就填写 Inter-Component Interactions。
- 文档与具体项目解耦，不假设代码或已提供上下文中没有体现的细节。

本输出使用了 AI 辅助生成。
