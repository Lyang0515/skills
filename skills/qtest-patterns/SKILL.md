---
name: qtest-patterns
description: >-
  Qt 测试的 API 速查与 CMake 接线参考，覆盖 C++ QTest（QTEST_MAIN）与 QML TestCase
  两大框架：断言宏全表、QSignalSpy、数据驱动测试（_data/QFETCH）、GUI 输入模拟、
  benchmark 宏、输出格式（JUnit XML/TAP）、CMake 测试目标模板、CTest 配置与常见故障排查。
  用于 "QTest 宏怎么用"、"QCOMPARE 没输出差异"、"qt_add_test 怎么写"、
  "staticMetaObject 未定义"、"测试输出 JUnit"、"Qt 测试 CMake 目标" 等问题。
  QML 用例的生成规范优先用官方 qt-qml-test skill，本 skill 只作速查。
license: MIT（改编自 L3DigitalNet/Claude-Code-Plugins 的 qtest-patterns，已移除 pytest-qt 部分）
---

# Qt 测试模式速查（C++ QTest + QML TestCase）

这是一本**工具书**，不是流程规范。要"从零给一个类写测试"，用项目自己的
`qiandaohu-unit-test`（C++）或官方 `qt-qml-test`（QML）；写的时候来这里查宏和
CMake 写法。

## 分工

| 需求 | 用哪个 |
|---|---|
| 给 C++ 类写白盒测试（流程、规范、项目约定） | `qiandaohu-unit-test` |
| 生成 `tst_*.qml` 用例（规则、模板、控件模式） | `qt-qml-test` |
| 跑 QML 测试并出报告 | `qt-qml-test-run` |
| **查 QTest 宏 / CMake 目标 / 输出格式 / 排错** | **本 skill** |

## 框架选型

| 场景 | 框架 |
|---|---|
| C++ 类 / 业务逻辑 | C++ QTest（`QObject` 子类 + `QTEST_MAIN`） |
| QML 组件行为 | Qt Quick Test（QML `TestCase` 类型） |

> 本项目不涉及 PySide6，故原 skill 的 pytest-qt 章节已移除。

## C++ QTest 要点

**完整宏表、QSignalSpy、数据驱动、输入模拟、benchmark、CMake 模板与排错**，见
[cpp-qtest.md](references/cpp-qtest.md)。

最小结构——测试类是 `QObject` 子类，测试函数是 `private slots`：

```cpp
class TestFoo : public QObject {
    Q_OBJECT
private slots:
    void test_something();       // 测试函数
    void test_something_data();  // 同名 + _data 做数据驱动
};

QTEST_MAIN(TestFoo)
#include "test_foo.moc"   // 文件名必须与 .cpp 一致，否则链接报 staticMetaObject 未定义
```

## QML TestCase 要点

**完整断言 API、组件创建、SignalSpy、异步等待与常见问题**，见
[qml-testcase.md](references/qml-testcase.md)。

最小结构——`TestCase` 项，测试函数必须以 `test_` 开头：

```qml
import QtQuick
import QtTest

TestCase {
    name: "MyComponentTests"
    function test_initialState() { compare(...) }
}
```

## 参考文件

- [references/cpp-qtest.md](references/cpp-qtest.md) — C++ QTest 宏全表、QSignalSpy、数据驱动、输入模拟、benchmark、输出格式、CMake 模式、排错
- [references/qml-testcase.md](references/qml-testcase.md) — QML TestCase 结构、断言、组件创建、信号、异步、导入路径、常见问题

## 三条最容易踩的坑

1. **`#include "<file>.moc"` 必须加**在文件末尾，且名字与 `.cpp` 同名。漏了就是
   `undefined reference to 'TestFoo::staticMetaObject'`。
2. **AUTOMOC 必须开**。作为子工程时父工程通常已开；独立 CMake 工程要自己
   `set(CMAKE_AUTOMOC ON)`，否则 `.moc` 根本不生成。
3. **独立工程要 `enable_testing()`**，否则 `add_test()` 静默失效，`ctest` 显示 0 tests。
