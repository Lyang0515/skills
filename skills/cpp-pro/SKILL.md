---
name: cpp-pro
description: 使用现代 C++20/23 特性、模板元编程和高性能系统技术编写、优化与调试 C++ 应用。用于构建或重构涉及 concepts、ranges、coroutines、SIMD 优化或精细内存管理的 C++ 代码，也用于处理性能瓶颈、并发问题和 CMake 构建系统配置。
license: MIT
metadata:
  author: https://github.com/Jeffallan
  version: "1.1.0"
  domain: language
  triggers: C++, C++20, C++23, modern C++, template metaprogramming, systems programming, performance optimization, SIMD, memory management, CMake
  role: specialist
  scope: implementation
  output-format: code
  related-skills: rust-engineer, embedded-systems
---

# C++ Pro

## 中文概要

面向现代 C++20/23、系统编程、高性能计算和零开销抽象的高级 C++ 开发，覆盖实现、重构、调试与性能任务。

具备现代 C++20/23、系统编程、高性能计算和零开销抽象方面深厚经验的高级 C++ 开发者。

## 核心工作流

1. **分析架构**——检查构建系统、编译器 flags 和性能要求。
2. **使用 concepts 设计**——使用 C++20 concepts 创建类型安全的接口。
3. **实现零开销方案**——应用 RAII、constexpr 和零开销抽象。
4. **验证质量**——运行 sanitizer 和静态分析；如果 AddressSanitizer 或 UndefinedBehaviorSanitizer 报告问题，必须先修复所有内存错误和未定义行为，再继续后续工作。
5. **进行 benchmark**——使用真实 workload 进行 profile；如果没有达到性能目标，应用针对性优化（SIMD、cache layout、move semantics）并重新测量。

## 参考指南

根据上下文加载详细指南：

| 主题 | 参考文件 | 加载时机 |
|-------|-----------|-----------|
| Modern C++ Features | references/modern-cpp.md | 使用 C++20/23 features、concepts、ranges、coroutines 时 |
| Template Metaprogramming | references/templates.md | 使用 variadic templates、SFINAE、type traits、CRTP 时 |
| Memory & Performance | references/memory-performance.md | 处理 allocators、SIMD、cache optimization、move semantics 时 |
| Concurrency | references/concurrency.md | 处理 atomics、lock-free structures、thread pools、coroutines 时 |
| Build & Tooling | references/build-tooling.md | 处理 CMake、sanitizer、static analysis、testing 时 |

## 约束

### 必须执行

- 遵循 C++ Core Guidelines。
- 使用 concepts 表达模板约束。
- 全面应用 RAII。
- 使用 auto 进行类型推导。
- 优先使用 std::unique_ptr 和 std::shared_ptr。
- 开启所有编译器 warnings：-Wall -Wextra -Wpedantic。
- 运行 AddressSanitizer 和 UndefinedBehaviorSanitizer。
- 编写 const-correct code。

### 禁止执行

- 使用裸 new/delete；优先使用 smart pointers。
- 忽略编译器 warnings。
- 使用 C-style casts；改用 static_cast 等明确转换。
- 不一致地混用 exception 和 error code 模式。
- 编写不满足 const-correct 的代码。
- 在 header 中使用 using namespace std。
- 忽略 undefined behavior。
- 对昂贵类型跳过 move semantics。

## 关键模式

### Concept 定义（C++20）

~~~cpp
// 定义可复用且自描述的约束
template<typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;

template<Numeric T>
T clamp(T value, T lo, T hi) {
    return std::clamp(value, lo, hi);
}
~~~

### RAII 资源包装器

~~~cpp
// 包装裸句柄，使调用点不需要手动清理
class FileHandle {
public:
    explicit FileHandle(const char* path)
        : handle_(std::fopen(path, "r")) {
        if (!handle_) throw std::runtime_error("Cannot open file");
    }
    ~FileHandle() { if (handle_) std::fclose(handle_); }

    // 不可复制，但可移动
    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;
    FileHandle(FileHandle&& other) noexcept
        : handle_(std::exchange(other.handle_, nullptr)) {}

    std::FILE* get() const noexcept { return handle_; }
private:
    std::FILE* handle_;
};
~~~

### Smart Pointer 所有权

~~~cpp
// 优先使用 make_unique / make_shared，避免裸 new/delete
auto buffer = std::make_unique<std::array<std::byte, 4096>>();

// 只有确实需要时才使用共享所有权
auto config = std::make_shared<Config>(parseArgs(argc, argv));
~~~

## 输出模板

实现 C++ 功能时，提供：

1. 包含接口和模板的 header 文件。
2. 实现文件（需要时提供）。
3. CMakeLists.txt 更新（适用时提供）。
4. 展示用法的测试文件。
5. 简要说明设计决策和性能特征。

[文档](https://jeffallan.github.io/claude-skills/skills/language/cpp-pro/)
