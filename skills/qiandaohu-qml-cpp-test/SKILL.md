---
name: qiandaohu-qml-cpp-test
description: >-
  为手持声像仪（qiandaohu_ui / QTAcoustic）的 QML 组件生成**加载真实源码文件**的
  单元测试，覆盖 QML 依赖 C++ 上下文（settingsManager / globalFileScanner 等）与
  pragma Singleton 组件的场景。用于 "给这个 QML 页面写测试"、"QML 依赖 settingsManager
  怎么测"、"tst_*.qml 加载真实组件"、"c++ 上下文 mock"、"单测加载不了 QML 报
  settingsManager is not defined" 等。不改动任何生产代码。
---

# 手持声像仪 QML 组件单元测试（带 C++ 上下文）

> 适用范围: `/home/luoshigang/work/code/qiandaohu_ui_dev/qiandaohu_ui_bugfix/External/QTAcoustic/`
> 技术栈: Qt 6.8.3 / Qt Quick Test + CMake；宿主 x86_64（qt6.8.3-host）

## 铁律：测真实文件，不抄副本

**唯一目标**：让 `tst_*.qml` 加载 `External/QTAcoustic/` 下的**真实** `.qml`，断言其真实行为。

**绝对禁止**把生产逻辑复制进测试再断言副本。副本不会随生产代码更新，
一旦分叉测试就是恒真的假绿。项目里已有两个反面教材（见「反模式」）。

判断方法：**测试文件里出现 `function _standardTemplates()`、`readonly property int stIdle: 0`
这类"再实现一遍"的代码，就是抄副本。**

## 前置：测试工程在哪

| 路径 | 角色 | 能不能改 |
|---|---|---|
| `External/QTAcoustic/main.cpp` | **生产**启动：`setContextProperty` 注入全部 C++ 上下文 | ❌ 绝对不动 |
| `External/QTAcoustic/unit_tests/qml/main.cpp` | **测试** harness：`QUICK_TEST_MAIN_WITH_SETUP` | ✅ 按需扩展 |
| `External/QTAcoustic/unit_tests/qml/support/` | 测试专用 C++ 替身 | ✅ 新增 |
| `External/QTAcoustic/unit_tests/qml/tst_*.qml` | QML 用例 | ✅ 新增 |

**"不影响主项目代码" = 只动 `unit_tests/` 下的文件。** 生产 `main.cpp` 一行不改，
生产数据（`/userdata/config/`）一个字节不碰。

## 找控件用 `testId`，不是 `objectName`

**这是本项目最容易踩的坑。** 项目用自定义属性 `testId` 作为测试钩子
（65 个 qml / 329 处声明），并被 `ElementCoordinateProvider.cpp:173` 消费。
而 QtTest 的 `findChild()` **只匹配 `objectName`**，拿它找控件永远返回 null
（`tst_FactoryExitConfirmPopup.qml` 的 4 个交互用例曾因此长期失败）。

自己写递归查找：

```qml
function findByTestId(node, testId) {
    if (!node) return null
    let kids = node.children ? Array.prototype.slice.call(node.children) : []
    for (let i = 0; i < kids.length; i++) {
        let c = kids[i]
        if (c && c.testId === testId) return c
        let r = findByTestId(c, testId)
        if (r) return r
    }
    return null
}
```

## 决策：三种加载方式

先看被测 `.qml` 属于哪一类，按表选：

| 组件类型 | 加载方式 | 例子 |
|---|---|---|
| **纯 QML，无 C++ 依赖** | ① 相对目录导入 | `tst_ParamCard.qml` |
| **依赖 C++ 上下文属性** | ② 相对目录导入 + harness 注入替身 | `tst_FirmwareUpgradePage.qml` |
| **`pragma Singleton`** | ③ harness 注册单例类型 | `ReportTemplateStore.qml`、`Translations.qml` |

**先判断依赖，再决定要不要写 C++ mock**：很多页面的依赖是 `property var` 注入的
（如 `FirmwareUpgradePage.fwManager`），**用 JS 对象字面量或 QML `QtObject` 就够**，
不必写 C++ 替身。只有当属性/信号需要真正响应式时才需要 `QtObject`（纯 JS 对象没有信号）。

### ① 相对目录导入（无 C++ 依赖）

```qml
import "../.." as App          // 相对 tst_*.qml 定位到 QTAcoustic 源码根

Component {
    id: cardComponent
    App.ParamCard { }          // 直接实例化真实组件
}
```

### ② 注入 C++ 上下文替身（依赖 settingsManager 等）

生产 `main.cpp` 用 `engine.rootContext()->setContextProperty("settingsManager", ...)`
等注入。测试 harness 用**同构**方式注入替身：

```cpp
// unit_tests/qml/main.cpp 的 QmlTestSetup::qmlEngineAvailable()
void qmlEngineAvailable(QQmlEngine *engine) {
    engine->addImportPath(QStringLiteral(QTACOUSTIC_SRC_DIR));
    engine->rootContext()->setContextProperty(
        QStringLiteral("settingsManager"), &m_mockSettings);   // 成员，保活
}
```

替身放 `unit_tests/qml/support/MockSettingsManager.{h,cpp}`，要求：

- **纯内存实现，绝不触碰 `/userdata/config/`**。生产 `SettingsManager.cpp:15-16`
  把路径硬编码成 `/userdata/config/*.json`，注入真实对象会污染设备配置。
- 接口按 **QML 实际调用统计**裁剪，不必实现全部。全量统计方法：
  ```bash
  grep -rhoE "settingsManager\.[a-zA-Z_]+" --include="*.qml" External/QTAcoustic/ \
    | sort | uniq -c | sort -rn
  ```
  当前结果：`getSetting` 67 / `saveSetting` 53 / `getCurrentDateTime` 4 /
  `loadDeviceInfo` 3 / `deviceSerialNumber` 3 / `setTimezone` 2 /
  `resetToFactory` 1 / `factoryDate` 1。
- **测试辅助方法必须标 `Q_INVOKABLE`**，否则 QML 调不到（报
  `Property 'xxx' of object ... is not a function`）。生产对齐的接口用
  `Q_INVOKABLE` / `public slots`，测试辅助（`preset`/`stored`/`clear`/`saveCount`）也要。
- `Q_OBJECT` 宏 + 类的 `.h` 和 `.cpp` 都加进 `qt_add_executable` 的源列表。

### ③ 注册 pragma Singleton

**单例必须显式注册**，不能靠目录导入的隐式发现。`Qt.createComponent()` 也不行。
本工程 QML 模块挂在**可执行文件**上（`qt_add_qml_module(appQTAcoustic ...)`），
没有可链接的 plugin，`import QTAcoustic` 同样不可用。

解法：在 harness 里把真实源码文件注册成单例类型——**零拷贝，测的还是生产文件**：

```cpp
qmlRegisterSingletonType(
    QUrl::fromLocalFile(QStringLiteral(QTACOUSTIC_SRC_DIR "/ReportTemplateStore.qml")),
    "QTAcousticTest", 1, 0, "ReportTemplateStore");
```

**关键**：即使页面源码里写的是 `App.Translations`（目录导入命名空间，不是上面的
URI），显式注册该 QUrl 后**目录导入也能解析到完整单例**。不注册时拿到的是**残缺对象**：
`getPageTitle` 等函数为 `undefined`，且这个状态**与访问顺序有关、时好时坏**——
曾出现同一份代码两次运行结果不同，极难排查。

已知需要注册的单例（看 `External/QTAcoustic/CMakeLists.txt` 里
`QT_QML_SINGLETON_TYPE TRUE` 之前的文件名）：`Translations`、`Logger`、
`ReportConclusion`、`ReportTemplateStore`。

注意：

- `qmlEngineAvailable()` 对**每个测试文件**调用一次，而单例注册是**全局**的，
  用静态标记避免重复注册告警：
  ```cpp
  if (!g_singletonsRegistered) { qmlRegisterSingletonType(...); g_singletonsRegistered = true; }
  ```
- **单例的依赖也必须补齐**：`Translations.qml:23` 写的是
  `languageManager ? languageManager.translationVersion : 0`，未注册时抛
  ReferenceError（不是优雅降级），会导致单例初始化失败。harness 已注入
  `MockLanguageManager`。真实的 `LanguageManager` 构造需要
  `QQmlApplicationEngine*`，无法独立构造。

## 用例写法

沿用仓库既有约定（`qt-qml-test` skill 的完整规则 + 本项目风格）：

- `TestCase { name: "...Tests"; when: windowShown }`（需要交互时）
- 分级注释 `// ====== P0/P1/P2 ======`：
  - **P0** 核心业务逻辑、主流程
  - **P1** 常规边界、持久化往返、信号发射
  - **P2** 边界条件、异常输入、安全性
- `function init()` 里重置替身状态（`settingsManager.clear()`），保证用例互不干扰
- 用 `tryCompare` / `SignalSpy.wait()`，不要裸 `wait()` 大超时
- 断言消息写中文，与仓库一致

**典型结构**（以真实单例为例，见 `tst_ReportTemplateStore.qml`）：

```qml
TestCase {
    name: "ReportTemplateStoreTests"
    function init() { settingsManager.clear() }        // 回到"首次开机"

    function test_P0_seedOnFirstAccess() {
        ReportTemplateStore.loadTemplates()
        compare(ReportTemplateStore.getAll().length, 6, "空配置下应播种 6 个标准模板")
    }
}
```

## 运行（关键：必须独立构建）

⚠️ **主工程顶层 `CMakeLists.txt` 硬编码了 rk3588 交叉工具链**（`CMakeLists.txt:80-95`），
而 `unit_tests/CMakeLists.txt` 在 `CMAKE_CROSSCOMPILING` 时直接 `return()` 跳过测试。
所以 `cmake -B build -DBUILD_TESTS=ON` **跑不了测试**，必须把 `unit_tests/` 当独立工程用
host Qt 构建：

```bash
cd <repo>                                   # qiandaohu_ui_bugfix/
cmake -B build-ut -S External/QTAcoustic/unit_tests \
      -DCMAKE_PREFIX_PATH=$PWD/qt6.8.3-host \
      -DCMAKE_BUILD_TYPE=Debug
cmake --build build-ut --target qml_tests -j$(nproc)
ctest --test-dir build-ut -R qml_tests --output-on-failure
```

单跑可执行（需自己带 offscreen，ctest 会通过 `ENVIRONMENT` 注入）：

```bash
QT_QPA_PLATFORM=offscreen ./build-ut/qml/qml_tests
QT_QPA_PLATFORM=offscreen ./build-ut/qml/qml_tests -o report.xml,junitxml
```

## 覆盖率门槛（全局强制）

覆盖率验收分为两层，不能用单个页面的用例通过替代全局覆盖率，也不能只看被某个测试目标编译到的代码：

1. **全局 C++ 行覆盖率至少 90%**。必须使用完整测试集运行 `bash scripts/run-coverage.sh`，并检查 `coverage-report/summary.md`；16 个核心类的未纳入测试目标代码按 0% 计入分母。全量测试失败或覆盖率低于 90% 都视为未通过。
2. **目标功能行为覆盖率至少 90%**。对用户指定的功能建立覆盖矩阵，至少覆盖主流程、状态组合、边界值、空/非法输入、反向操作、重复操作和真实用户交互路径；QML 行覆盖率不由 lcov 统计时，必须用真实源码行为断言和矩阵清单替代，不能把 C++ 全局数字冒充 QML 功能覆盖率。

标准全局门禁命令：

```bash
MIN_GLOBAL_LINE_COVERAGE=90 ENFORCE_COVERAGE=1 bash scripts/run-coverage.sh
```

只跑单个目标进行快速迭代时可以关闭门禁，但不能作为最终验收：

```bash
ENFORCE_COVERAGE=0 bash scripts/run-coverage.sh -t qml_tests
```

筛选类功能的最小行为矩阵应包含：全部、今日、本周、本月、单边起止、闭区间边界、反向起止、无日期数据、类型筛选、时间与类型组合、快速重复切换、异步数据刷新后的模型与首项显示一致性，以及物理按键/触屏路径。新增或修改筛选逻辑后必须重新运行完整 QML 测试和全局覆盖率门禁。

## 反模式

| 写法 | 为什么是错的 |
|---|---|
| 把生产逻辑抄成 JS 副本再断言副本 | 副本不随生产更新，分叉即恒真假绿。`tst_logic.qml` 曾因此失败（2≠3），已删除 |
| 自己声明一份常量再断言自己 | 断言的是自己刚写的字面量，生产怎么改都过 |
| 用 `findChild` 找控件 | 只匹配 `objectName`，本项目控件标的是 `testId` |

**断言前先问：改坏生产的哪一行会让它变红？** 答不上来就是恒真。

有意义的写法示例：断言**页面上的**常量等于 C++ 枚举值
（`tst_FirmwareUpgradePage.qml` 的 `test_P0_stateConstants_matchCppEnum`）——
C++ 重新编号而 QML 没跟上时它会红。而"测试里声明常量再断言常量"永远不会红。

**动画属性用 `tryCompare`**：`border.width` 这类带
`Behavior + NumberAnimation` 的属性，改值后立即 `compare` 会读到中间值。

## 已修复的历史问题（别退回）

- `tst_FactoryExitConfirmPopup.qml` 4 个交互用例——`findChild` 改用 `findByTestId`
- `tst_logic.qml`（7 个镜像用例）——已删除，模板 CRUD 迁到 `tst_ReportTemplateStore.qml`
- `tst_FirmwareUpgradePage.qml`（28 个恒真用例）——已重写为加载**真实页面**

## 收尾检查

1. **新增用例全绿**，且断言能在生产代码改错时失败（想一下：改坏哪里会让它红？恒真就是写错了）。
2. **失败集合与基线一致**——先跑一次基线存档，改完 `diff` 失败清单，确认零回归。
3. **全局覆盖率门禁通过**：完整测试集无失败，`coverage-report/summary.md` 的 16 类真实基线达到 90%；单目标覆盖率不得替代全局结果。
4. **目标功能行为覆盖率达到 90% 以上**：覆盖矩阵中的每个高风险分支都有真实源码断言。
5. **生产代码零改动**：`git status` 里不该出现 `External/QTAcoustic/*.qml`、`External/QTAcoustic/main.cpp`。
6. **没碰真实路径**：替身无磁盘 IO；`/userdata/` 下不应有任何测试产生的文件。

## 参考

- 完整 QML TestCase API / 规则 / 控件模式 → 官方 `qt-qml-test` skill
- QTest 宏速查 / CMake 测试目标 / 排错 → `qtest-patterns` skill
- C++ 侧白盒测试规范 → `qiandaohu-unit-test` skill
- 覆盖率测量 → `scripts/run-coverage.sh`（host 构建产出 .gcda，无需上板；默认全局 90% 门禁）
- 现成范例（照抄结构即可）:
  - **单例 + C++ 替身** → `unit_tests/qml/tst_ReportTemplateStore.qml`
    + `support/MockSettingsManager.h`、`support/MockFileScanner.h`
  - **页面级真实加载 + testId 查找 + 跨语言契约** → `unit_tests/qml/tst_FirmwareUpgradePage.qml`
  - 纯组件相对导入 → `unit_tests/qml/tst_ParamCard.qml`
  - harness 全貌 → `unit_tests/qml/main.cpp`
