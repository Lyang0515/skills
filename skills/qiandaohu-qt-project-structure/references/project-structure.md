# 千岛湖 Qt 工程结构参考

本参考描述当前仓库中影响结构重构决策的事实。使用时仍应以目标 worktree 的实际文件为最终依据，因为 `qiandaohu_ui` 和 `qiandaohu_ui_bugfix` 可能存在分支差异。

## 主要边界

- 仓库工程：`qiandaohu_ui/` 或 `qiandaohu_ui_bugfix/`
- 应用源码：`External/QTAcoustic/`
- 顶层 AI 运行库：`External/AIChain/`
- AI 链路源码：`External/QTAcoustic/VIChain/`
- 预编译接口和系统集成库：`External/QTAcoustic/EnsonicAPI/`
- 应用单测：`External/QTAcoustic/unit_tests/`
- 主构建目录：通常为 `build/`
- 单测、覆盖率和 release 构建目录：以目标 worktree 的 `AGENTS.md` 和实际目录为准

`QTAcoustic` 同时包含 C++、QML、QML singleton、测试、翻译、报告模板、音频、证书/公钥、预编译库和 AI 模型相关数据。目录下的 `build/`、动态库和运行时缓存是生成物或第三方交付物，不应被误认为可自由重构的源码模块。

## 当前 CMake 形态与重构关注点

典型入口链路为：

```text
顶层 CMakeLists.txt
  -> add_subdirectory(External)
  -> External/QTAcoustic/CMakeLists.txt
       -> qt_add_executable(appQTAcoustic ...)
       -> add_subdirectory(VIChain)       # AI 开启时
       -> qt_add_qml_module(appQTAcoustic ...)
       -> qt_add_resources(...)           # 报告、翻译、静态资源
       -> add_subdirectory(unit_tests)    # BUILD_TESTS 开启时
```

重构前重点检查：

1. `QTAcoustic/CMakeLists.txt` 是否同时包含应用配置、源码清单、链接库、资源复制、AI 库同步和测试入口。
2. 同一生产源码是否既出现在 `qt_add_executable()` 又出现在后续 `target_sources()`。
3. QML 文件是否属于同一个 URI、是否通过 `qmldir` 声明 singleton 或子目录 module，是否被重复打包。
4. 资源同步是否依赖 `CMAKE_SOURCE_DIR`、顶层 `bin` 或外部 `AIChain`；移动子目录时这些路径不能凭相对层级猜测。
5. `VIChain`、`EnsonicAPI` 和 AI 共享库的真实依赖是编译时依赖、链接时依赖还是运行时部署依赖。
6. 单测是否只在桌面构建启用，是否通过 `qtacoustic_add_cpp_test()` 复用真实源码和 stub，而不是复制生产实现。

## 保持不变的接口

除非用户明确要求迁移，否则优先保持：

- 应用 target：`appQTAcoustic`
- C++ 标准：C++17
- Qt 版本：Qt 6.8.3
- 核心选项：`ENABLE_AI_VOICE`、`UIAUTO_TEST_ENABLED`、`ENABLE_GUIDE`、`ENABLE_MOCK_MESSAGES`、`BUILD_TESTS`，以及当前分支实际存在的其他选项
- RK3588/aarch64 交叉编译和桌面测试的分离
- 板端运行目录对应的 `bin/bin`、`bin/lib`、资源目录布局

## 官方 CMake/QML 依据

- [Qt CMake Getting Started](https://doc.qt.io/qt-6/cmake-get-started.html)：顶层工程配置与子目录 target 组织。
- [qt_standard_project_setup](https://doc.qt.io/qt-6/qt-standard-project-setup.html)：应在定义 target 前完成标准 Qt CMake 设置。
- [Qt QML CMake Integration](https://doc.qt.io/qt-6/qtqml-cmake-integration.html)：使用 `qt_add_qml_module()` 管理 QML module、资源和工具链集成。
- [qt_add_qml_module](https://doc.qt.io/qt-6/qt-add-qml-module.html)：URI、QML 文件、资源、类型信息和 module 生成规则。

这些文档用于解释 CMake/QML 的通用行为；本项目的交叉编译、部署和测试约束仍以仓库规则为准。
