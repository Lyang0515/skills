# Skills 仓库

集中管理日常开发中使用的 Agent Skills（技能包），用于版本管理、跨机器同步与复用。

本仓库内容来自开发服务器 `192.168.3.52` 上 `~/.codex/skills/` 中用户自建的技能，
共 **25 个 skill**，以统一单目录方式存放于 [`skills/`](skills/) 下，每个 skill 一个目录。

> **拉取信息**
> - 来源：`luoshigang@192.168.3.52:/home/luoshigang/.codex/skills`
> - 同步时间：2026-10-13
> - 内容：25 个用户技能，208 个文件
> - 说明：服务器上 `~/.claude/skills/`、`~/.config/opencode/skills/` 为同一批技能的**较早副本**（部分 `SKILL.md` 版本落后），故以最新的 `~/.codex/skills` 为准统一收录，不重复保留旧副本；Codex 自带的 `.system/`（imagegen、openai-docs、plugin-creator、review-agent、skill-creator、skill-installer）属于 Agent 内置基础设施，未纳入本仓库。

---

## 目录结构

```
.
├── README.md          # 本文件
├── .gitignore
└── skills/            # 每个 skill 一个目录，内含 SKILL.md 及附属资源
    ├── cpp-pro/
    ├── linux-perf/
    ├── performance-patterns/
    ├── memory-optimization/
    ├── phoronix-test-suite/
    ├── qt-qml/
    ├── qt-cmake-project/
    ├── qt-cpp-docs/
    ├── qt-cpp-review/
    ├── qt-qml-docs/
    ├── qt-qml-review/
    ├── qt-qml-profiler/
    ├── qt-qml-test/
    ├── qt-qml-test-run/
    ├── qt-ui-design/
    ├── qt-figma-token-extraction/
    ├── qt-figma-component-generation/
    ├── qtest-patterns/
    ├── qiandaohu-unit-test/
    ├── qiandaohu-qml-cpp-test/
    ├── qiandaohu-qt-project-structure/
    ├── device-testing/
    ├── git-worktree/
    ├── grill-me/
    └── requirement-analysis/
```

每个 skill 目录遵循 Agent Skills 约定：核心为 `SKILL.md`（YAML frontmatter 定义 `name`、`description` 等），
并按需附带 `references/`、`assets/`、`examples/`、`platforms/` 等子目录存放参考资料与模板。

---

## Skill 一览

### 一、C++ 与性能工程

| Skill | 用途 | 典型触发场景 |
|-------|------|--------------|
| **cpp-pro** | 使用现代 C++20/23 特性、模板元编程和高性能系统技术编写、优化与调试 C++ 应用 | 涉及 concepts、ranges、coroutines、SIMD 优化、精细内存管理；处理性能瓶颈、并发问题、CMake 构建配置 |
| **linux-perf** | 使用 `perf` 分析和修复 Linux 性能问题 | 硬件计数器（IPC/缓存未命中/分支预测）、热点定位、cache line 竞争（false sharing、HITM、`perf c2c`）、核数扩展性分析 |
| **performance-patterns** | 从源码或性能分析输出中识别并修复 x86/C/C++ 性能模式 | serial accumulator、过窄 SIMD、`_mm*` intrinsics、false sharing、thundering herd、缺失 restrict/vzeroupper、快速 CRC32C/Distance 计算、SIMD sort |
| **memory-optimization** | 优化并降低 Linux（尤其是 RK3588 嵌入式 ARM）上 C++/Qt 应用的内存占用 | 用 `/proc/PID/status`、smem、perf mem、valgrind massif 测量，定位泄漏/峰值分配/碎片化、Qt 隐式共享与 QImage/QPixmapCache 问题 |
| **phoronix-test-suite** | 安装、运行、解析和优化 Phoronix Test Suite（PTS）基准测试 | 提到 "phoronix"、"pts/"、或要求运行/优化 PTS 测试，如 `run pts/mt-dgemm`、`pts/compress-zstd` |

### 二、Qt / QML 开发（qt-ai-skills 官方技能集）

| Skill | 用途 | 典型触发场景 |
|-------|------|--------------|
| **qt-qml** | 编写或处理 QML 源码时应用 QML 最佳实践 | 编写、审查、修复、重构、优化或调试 QML 文件、组件、绑定（纯概念问答不触发） |
| **qt-cmake-project** | 生成或更新 Qt 6 CMake 工程 | 编辑 `CMakeLists.txt`、添加源文件/资源、定义 executable / QML module / library target |
| **qt-cpp-docs** | 为 Qt/C++ 源文件生成独立 Markdown 参考文档 | 记录 Qt Widgets 类、Qt Quick 后端、模块、普通 C++ 工具、struct、main.cpp 等 |
| **qt-cpp-review** | Qt6 C++ 代码只读审查（确定性 lint + 六个并行深度分析 agent） | "review / check / audit this Qt C++ code"，覆盖模型契约、所有权、线程、API 正确性、错误处理、性能；只报高置信度问题，不改代码 |
| **qt-qml-docs** | 为 QML 组件和应用生成独立 Markdown 参考文档 | "document this QML"、"QML API docs"，支持单文件 / 粘贴代码 / 整个工程目录 |
| **qt-qml-review** | Qt6 QML 代码只读审查（确定性 lint + 六个并行分析 agent） | "review / check / audit this QML code"，覆盖 bindings、layout、loaders、delegates、states、性能，可选用 `qmllint` |
| **qt-qml-profiler** | 调查 QML / Qt Quick 2D 性能 | "UI 卡顿 / 掉帧 / 抖动"，采集解析 `.qtd` trace，输出 frame-time、memory、pixmap-cache 摘要（不含 Qt Quick 3D） |
| **qt-qml-test** | 为 QML 组件生成 Qt Quick Test 用例 | "write QML tests"、"qml test"，使用 TestCase、SignalSpy、tryCompare |
| **qt-qml-test-run** | 构建并运行 QML 工程测试（qmltestrunner / CTest）并生成 Markdown 报告 | "run qml tests"、"run qmltestrunner" |
| **qt-ui-design** | 为 Qt/QML、Qt 工程、Web 或嵌入式 MPU/MCU 目标设计或审查 UI | 创建页面、布局、导航，审查 UX/可用性/设计系统 |
| **qt-figma-token-extraction** | 从 Figma design system 提取 design tokens，生成 `design-tokens.json` 与 QML singletons | "export tokens from Figma"、"set up my design system"、"pull colors/typography from Figma" |
| **qt-figma-component-generation** | 从 Figma 组件元数据生成生产级 QML controls | "generate components from Figma"、"convert Figma components to QML"（依赖 token extraction 产物） |

### 三、测试

| Skill | 用途 | 典型触发场景 |
|-------|------|--------------|
| **qtest-patterns** | Qt 测试 API 速查与 CMake 接线参考（C++ QTest + QML TestCase） | 断言宏全表、QSignalSpy、数据驱动测试（`_data`/`QFETCH`）、benchmark 宏、JUnit XML/TAP 输出、CMake 测试目标模板、CTest 排障 |
| **qiandaohu-unit-test** | 为手持声像仪（qiandaohu_ui）项目生成符合既有规范的白盒单元测试 | "给 WifiManager 加测试"、"生成 tst_*.cpp"，复用项目 Testable*/friend/QSignalSpy/QTemporaryDir 等成熟模式，支持桌面与 arm64 交叉编译 |
| **qiandaohu-qml-cpp-test** | 为手持声像仪 QML 组件生成**加载真实源码**的单元测试 | 覆盖 QML 依赖 C++ 上下文（settingsManager/globalFileScanner）、`pragma Singleton` 组件场景，不改动生产代码 |

### 四、千岛湖项目工程

| Skill | 用途 | 典型触发场景 |
|-------|------|--------------|
| **qiandaohu-qt-project-structure** | 维护和重构 qiandaohu_ui / QTAcoustic 的 Qt 6.8.3 / QML / CMake / RK3588 交叉编译工程结构 | 拆分 CMake target、整理 C++/QML 模块、隔离测试与运行时资源、修复构建接线、制定渐进式结构迁移方案 |
| **device-testing** | 编译、部署、测试谛声科技 RK3588 嵌入式设备的 appQTAcoustic 应用 | 远程 SSH 部署（设备 192.168.3.115）、模拟 AI 按键/触摸屏、日志分析、语音流程验证 |

### 五、工作流与通用

| Skill | 用途 | 典型触发场景 |
|-------|------|--------------|
| **git-worktree** | 管理 git worktree 实现多分支并行开发 | 创建/查看/删除 worktree，基于 develop/main 创建 feature/bugfix 分支独立开发，避免频繁切换分支 |
| **grill-me** | 围绕计划或设计持续访谈用户，直到形成共识 | 对计划做压力测试、"interview me"；逐个解决设计树上的依赖决策，可先探索代码库再给出推荐答案 |
| **requirement-analysis** | 需求分析与开发文档生成 | "帮我分析需求"、"写开发文档"、"需求规格"、"技术方案"；强制先澄清再输出文档，禁止跳过提问直接给方案 |

---

## 使用方法

### 复用到本机 Agent

把需要的 skill 目录复制（或软链接）到对应 Agent 的用户技能目录即可：

```bash
# Claude Code
cp -r skills/* ~/.claude/skills/

# Codex
cp -r skills/* ~/.codex/skills/

# opencode
cp -r skills/* ~/.config/opencode/skills/
```

> 建议按需挑选：`qt-*` / `cpp-pro` / `linux-perf` / `performance-patterns` /
> `memory-optimization` / `phoronix-test-suite` 为通用技能，可直接复用；
> `qiandaohu-*` / `device-testing` 等含项目专有路径与设备信息，需按实际环境调整。

### 更新本仓库

在服务器上修改 skill 后，重新从服务器同步即可：

```bash
# 服务器打包（排除 .system 等内置技能）
ssh luoshigang@192.168.3.52 \
  "cd ~/.codex/skills && tar czf - <skill-dir> ..." | tar xzf - -C skills/
```

---

## 许可证说明

各 skill 的许可证以自身 `SKILL.md` 中的 `license` 字段为准：

- `cpp-pro`：MIT
- `linux-perf` / `performance-patterns` / `phoronix-test-suite`：MIT（Intel Corporation）
- `qt-*`（qt-ai-skills）：LicenseRef-Qt-Commercial OR BSD-3-Clause
- `qtest-patterns`：MIT（改编自 L3DigitalNet/Claude-Code-Plugins）
- `qiandaohu-*` / `device-testing` / `git-worktree` / `grill-me` / `requirement-analysis`：项目内部技能
