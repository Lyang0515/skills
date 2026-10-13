---
name: qiandaohu-unit-test
description: >-
  为手持声像仪（qiandaohu_ui）项目生成符合既有规范的白盒单元测试（QtTest, QTEST_MAIN）。
  用于 "写这个类的单测"、"给 WifiManager 加测试"、"生成 tst_*.cpp" 等。复用项目中
  unit_tests/ 已有的 Testable* 子类、friend 声明、QSignalSpy、QTemporaryDir 文件隔离、
  SimpleLogger 初始化、P0/P1/P2 分级注释等成熟模式。也用于运行桌面/arm64 交叉编译测试。
---

# qiandaohu（手持声像仪）白盒单元测试 Skill

> 适用范围: `/home/luoshigang/work/code/qiandaohu_ui_dev/qiandaohu_ui_bugfix` 及其下 `External/QTAcoustic`
> 技术栈: Qt 6.8 / Qt Test（QTEST_MAIN）+ CMake + Ninja; 桌面 x86_64 与 RK3588 arm64 交叉编译

## 使命

项目已有**成熟、多样**的白盒测试模式（见 `External/QTAcoustic/unit_tests/*/`）。本 skill 的目标是：
1. **复用既有模式**而非自创风格，生成与仓库一致的 `tst_*.cpp` + `CMakeLists.txt` + stub。
2. 让测试目标能在**桌面环境**干净地编译、链接、运行（用 stub/spy 屏蔽硬件与 QML_ELEMENT 依赖）。
3. 支持 `arm64` 交叉编译的运行/打包校验。

## 参考范例（务必先读再写）

生成任何测试前，先通读一到两个既有范例，严格对齐其结构与命名：

| 目标模块 | 范例测试 | 特点 |
|---------|---------|------|
| WiFi Manager | `unit_tests/wifi/tst_WifiManager.cpp`（1415 行，最全面） | `TestableWifiManager` 子类 + friend、`setupTempConfigDir()`/`flushSave()` 文件隔离、P0/P1/P2 分区注释、`injectMockWifiData()`、`setWifiTestMode()` |
| 消息中心 | `unit_tests/message_manager/tst_MessageManager.cpp` | 信号 spy、状态机 |
| AI 语音 | `unit_tests/ai_voice/` | `stub_AiVoiceManager.cpp` + `spy_*.h` 依赖注入、`SimpleLogger` 初始化 |
| 帮助参数 | `unit_tests/help_param/tst_HelpParam.cpp` | 三语种 .ts 翻译完整性静态校验 |
| OTA | `unit_tests/ota/` | 升级流程相关 |
| 二维码 | `unit_tests/qrcode/` | |

## 标准生成模式

### 1. Testable 子类 + friend
被测类若含私有成员/Qt moc，定义 `Testable<ClassName>` 子类暴露私有成员，并在被测类头文件加 `friend class Testable<ClassName>;`。参考 `tst_WifiManager.cpp` 的 `TestableWifiManager`。

### 2. 文件系统隔离
所有写盘操作用 `QTemporaryDir`，通过显式 `setConfigPath()` / `setConfigDir()` 注入，禁止触碰到真实路径（如 `/userdata/config/`）。参考 `setupTempConfigDir()` + `flushSave()` 模式。

### 3. 分级注释
按重要性分级并在区块顶部用 `// ============` 分隔注释，沿用 P0/P1/P2 约定：
- **P0**: 核心业务逻辑、主流程
- **P1**: 常规边界、持久化往返、信号发射
- **P2**: 边界条件、安全性、异常输入、压力测试

### 4. Mock / 信号
- 状态变化用 `QSignalSpy` 断言（`spy.count()`）。
- 多信号场景：每个目标一个 spy、用描述性 id。
- 无法在桌面复现真实硬件行为时，用 `stub_*.cpp` 提供链接满足 + `spy_*.h` 记录调用。

### 5. 初始化
若被测类依赖 logger/环境，先初始化 `SimpleLogger`（参考 ai_voice 范例）。

## 测试目标在桌面环境运行

每个 `unit_tests/<模块>/` 是独立 CMake 工程。构建并运行：

```bash
cd External/QTAcoustic/unit_tests/<模块>
mkdir -p build && cd build
cmake .. -DCMAKE_PREFIX_PATH=$HOME/qt6.8.3-host
cmake --build . -j$(nproc)
./<target>_tests        # 直接运行，或
ctest                   # 通过 ctest
```

> 可执行目标名通常为 `<模块>_tests`（如 `wifi_tests`）；具体以各模块 `CMakeLists.txt` 的 `qt_add_executable` / `add_test` 命名为准。若模块自带 `CMakeLists_arm64.txt` 交叉配置，arm64 目标按需使用；桌面优先。

## QML 测试

若目标是 QML 组件（页面/弹窗），参考项目内 QML `TestCase` 约定（`createTemporaryObject`、`SignalSpy`、`windowShown`），与通用 QML 测试技能一致，但必须贴合本项目设计系统（见 `DESIGN.md`）。

## 运行与验证

生成后：
1. 桌面构建编译零警告可链接过。
2. 运行所有用例通过（`QVERIFY`/`QCOMPARE` 全绿）。
3. 若改动涉及被测类头文件（加 friend / 加方法），确认不影响生产代码行为。

## Guardrails

- **禁止自创命名**：Testable 子类名、`setupTempConfigDir` 这类辅助函数名、P0/P1/P2 注释风格，全部与既有范例一致。
- **禁止触碰真实路径**：测试内任何文件操作必须落在 `QTemporaryDir`。
- **不打断生产编译**：加 `friend`/暴露成员时，只做最小改动并保持原语义。
- 生成 `tst_*.cpp` 后**写到盘上**（用写文件工具），不要在聊天里贴整段代码。
- 测试类本身注释用中文（对齐仓库风格）。
