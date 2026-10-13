---
name: qiandaohu-qt-project-structure
description: >-
  维护和重构 qiandaohu_ui / QTAcoustic 的 Qt 6.8.3、QML、CMake 和 RK3588
  交叉编译工程结构。用于拆分 CMake target、整理 C++/QML 模块、隔离测试与
  运行时资源、修复构建接线或制定渐进式项目结构迁移方案；不用于普通 Qt
  代码审查、单个 QML 组件开发或设备部署本身。
metadata:
  short-description: 千岛湖 Qt 工程结构维护与重构
---

# 千岛湖 Qt 工程结构

## 目标

在不破坏现有 Qt 6.8.3、RK3588/aarch64 交叉编译、QML 运行时、桌面单测和板端产物布局的前提下，逐步把 `QTAcoustic` 从“大型单 target + 混合资源目录”整理为职责清晰、可独立验证的 CMake 工程。

先阅读 [project-structure.md](references/project-structure.md)；涉及具体目录、构建选项、产物路径或验证命令时，以该文件和仓库实际内容为准，不凭记忆补写路径。

## 使用边界

- 修改 `CMakeLists.txt`、target、Qt module、资源复制、测试接线或工程目录时使用本 skill。
- 写/改 QML 组件时同时使用 `qt-qml`；做 CMake 工程编辑时同时使用 `qt-cmake-project`。
- 写 C++ 白盒测试、QML 测试或执行测试时分别使用项目已有的单测 skill、`qtest-patterns`、`qt-qml-test-run`。
- 需要 SSH、设备部署、模拟触摸/按键或分析板端日志时使用 `device-testing`；本 skill 不自行连接设备。
- 普通 C++/QML 代码审查分别使用 `qt-cpp-review` 或 `qt-qml-review`，不要把结构重构规则当作完整代码审查。

## 工作方式

1. 先确认当前 worktree、分支和项目级 `AGENTS.md`，再检查源码树、构建目录是否为生成物，以及当前 CMake 配置是否已有未提交修改。
2. 画出当前依赖边界：顶层工程、QTAcoustic 应用、`VIChain`、`EnsonicAPI`、AI 运行库、QML 模块、资源和 `unit_tests`。不要因为目录名称相似就假定它们是独立 target。
3. 将重构拆成可回滚的小步：先移动 CMake 职责和 target 边界，再移动源码目录，最后调整 QML module、资源和产物同步。每一步都保持旧 target 名称和构建选项可用，除非用户明确要求改名。
4. 优先使用 target 级属性：`target_sources()`、`target_include_directories()`、`target_compile_definitions()`、`target_link_libraries()`。避免继续扩大全局 `include_directories`、全局编译选项和隐式路径。
5. QML 组织优先采用 `qt_add_qml_module()`；按功能边界拆分 module 时，明确 URI、版本、Singleton、`qmldir`、资源前缀和 C++ 类型注册的归属，避免同一个 QML 文件同时被多个 module 或旧资源机制打包。
6. 预编译库、交叉编译 sysroot、AI 运行库和板端目录属于部署边界。重构时不得把它们当作普通源码移动，也不得把凭据、私钥或构建产物加入版本控制。
7. 每次修改后至少执行配置检查、目标构建和相关测试；涉及 QML 时运行 `qmllint`，涉及资源/翻译时检查生成资源，涉及交叉编译输出时核对 `bin` 目录布局。设备验证必须先询问用户设备 IP。

## 结构决策原则

- 顶层 `CMakeLists.txt` 负责项目级工具链、Qt 根路径、全局标准和子目录入口；应用目录负责应用 target；功能目录负责自己的源码、依赖和选项。
- 一个目录只有在拥有稳定的编译边界、依赖边界或测试边界时才拆成 library/object library；不要为了目录数量而抽象。
- 生产源码清单、测试源码清单、QML module 清单和运行时资源清单分开维护；禁止通过复制同一份列表制造多个来源。
- 资源复制应表达源文件到产物的依赖关系，避免只在链接成功后执行、无法感知资源变化的脆弱 `POST_BUILD` 逻辑。
- 优先保留现有 target、选项和输出路径，采用兼容迁移；只有在确认所有调用点和 CI 都已更新后才删除旧入口。
- 不要把 `build*`、`bin*`、生成的 `.so`、`.qm` 或临时部署目录当成源码结构的一部分。

## 验证完成标准

结构调整只有在以下条件都满足时才算完成：

- CMake 能在主工程和项目要求的独立测试场景中成功配置。
- 目标平台构建能生成原有应用 target，且 AI 开关、测试开关和其他现有选项语义不变。
- 桌面单测和受影响的 QML 测试通过；测试仍加载真实生产源码，不复制生产逻辑作为被测副本。
- QML 静态检查、翻译/资源打包和增量构建没有新增错误。
- 产物目录仍符合板端部署约定；需要真机验证时，先由用户提供设备 IP，再使用 `device-testing`。
