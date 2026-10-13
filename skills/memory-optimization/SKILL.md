---
name: memory-optimization
description: >-
  优化并降低 Linux 上 C++/Qt 应用的内存占用，尤其是嵌入式 ARM（RK3588）设备。使用
  /proc/PID/status、smem、ps、perf mem、valgrind massif（主机 debug build）、
  malloc_stats 进行测量，然后定位泄漏、峰值分配、碎片化以及 Qt implicit-sharing /
  QImage / QPixmapCache 问题；实施针对性修复并重新测量验证。触发词包括：memory、RSS、
  RES、VmRSS、VmHWM、VmPeak、leak、OOM、memory footprint、reduce memory、heap、
  fragmentation、malloc、QImage、QPixmap、QCache、内存、内存占用、内存优化、内存泄漏、
  峰值、RSS、占用；“内存排查”“低内存”“卡顿伴随内存增长”也适用。
---

# Linux/Qt 内存优化 Skill

> 适用范围: Linux x86 (主机调试) 与 ARM RK3588 (谛声科技设备) 上的 C++/Qt 应用
> 关联: 部署/日志细节见 `device-testing` skill

核心原则：**先测量，再优化**。没有测量数据的"优化"是猜测，会过早优化、引入复杂度而不真正改善任何指标。

---

## 1. 工作流程（每次循环执行）

```
MEASURE  → 采集基线数据（RSS/HWM/分配热点），留下可复现的基线
IDENTIFY → 定位真实瓶颈（是泄漏、峰值、还是碎片化？而不是凭感觉）
FIX      → 针对单一瓶颈做最小修改
VERIFY   → 重新测量同一场景，对比基线；无改善则回退
GUARD    → 加监控/单测防止回归（如 RSS 上限断言、重复场景自动化采样）
```

循环直到达到目标（如"常驻 < X MB"）。一次只处理一个瓶颈。

---

## 2. 测量：数据采集命令

### 2.1 进程内存快照（最常用）

```bash
# 单次快照
grep -E "VmRSS|VmHWM|VmPeak|VmSwap|RssAnon|RssFile" /proc/<PID>/status

# 定时采样：观察是否随操作单调增长（泄漏信号）
while true; do
  date +%H:%M:%S; grep -E "^(VmRSS|VmHWM|VmSwap)" /proc/<PID>/status
  sleep 5
done

# vsz/rss 概览，按内存排序
ps -eo pid,rss,vsz,comm --sort=-rss | head -20

# 设备上配合系统状态
free -m
```

- `VmRSS` = 当前常驻内存（真实物理占用）
- `VmHWM` = 历史峰值常驻（程序自启动以来最高 RSS，**最重要的指标**）
- `VmPeak` = 历史峰值虚拟内存
- `VmSwap` = 被换出的部分（换出说明内存紧张）

### 2.2 共享/独占内存分解

```bash
# smem：区分 USS（独占）/ PSS（含按比例共享）/ RSS（含全部共享）
smem -tk -s uss
smem -tk -P appQTAcoustic
```

### 2.3 硬件采样：perf（需要板子/内核支持）

```bash
# 数据采集
perf mem record -p <PID> -- sleep 60
perf mem report

# 分配热点（Linux 5.7+ 支持 perf mem + malloc 追踪）
# 定位：哪条调用链在不停申请内存
```

### 2.4 主机调试构建：valgrind massif（板上通常没有，在 x86 主机跑）

```bash
# 需要 -g 编译；建议用调试构建（Release + debug info 也可）
valgrind --tool=massif --stacks=yes --time-unit=B ./app <args>
ms_print massif.out.<pid>          # 看峰值与分配热点调用链

# 替代：heaptrack（更快，输出更友好）
heaptrack ./app <args>
heaptrack_print heaptrack.app.*.zst
```

### 2.5 程序内统计：malloc 信息（无需外部工具）

```cpp
// glibc：程序任意点打印堆统计到 stderr
#include <malloc.h>
malloc_info(0, stderr);

// (musl 无 malloc_info)嵌入式常见做法：周期性采样 /proc/self/status 写入日志
```

### 2.6 设备远程采样模板（RK3588）

```bash
# 采样 60s，间隔 3s，输出 RSS/HWM 序列
sshpass -p 'root' ssh root@192.168.3.115 '
  PID=$(pgrep -f appQTAcoustic | head -1)
  echo "pid=$PID"
  for i in $(seq 1 20); do
    grep -E "^(VmRSS|VmHWM|VmSwap)" /proc/$PID/status | tr "\n" " "
    date +%H:%M:%S
    sleep 3
  done'
```

---

## 3. IDENTIFY：信号解读

| 现象 | 含义 | 方向 |
|------|------|------|
| VmHWM 随重复操作单调上升，RSS 不回落 | **真实泄漏**（有指针残留） | 逐操作二分定位；valgrind memcheck |
| RSS 在高位平台，VSS 巨大但 RSS 平稳 | 大块映射（纹理/共享库/GPU） | 查目标平台缓冲、QImage/纹理缓存 |
| RSS 先涨后平稳但明显偏大 | 一次性大分配（缓存/模型/图片）被长期持有 | 延迟加载、压缩、按需释放 |
| RSS 上下波动但平均高 | 对象反复创建销毁（碎片化） | 池化/复用、reserve、减少小分配 |
| RSS 上升后下降，但每次峰值递增 | 缓存累积后部分回收（QPixmapCache 等） | 调节 cacheLimit、到期清理 |
| RSS 持续缓慢增长且伴随 Swap 增长 | 接近 OOM | 立即查最大持有者，考虑总内存预算 |

**关键区分**：先确认是「泄漏」还是「持有过多」。泄漏要改生命周期；持有过多要改缓存策略/数据结构。

---

## 4. FIX：常见修复模式

### 4.1 Qt/C++ 高收益项（按常见收益排序）

1. **QImage 格式**（最频繁、收益最大）
   - `Format_ARGB32_Premultiplied` → 若不需要 Alpha 用 `Format_RGB32`；灰度图用 `Format_Grayscale8`（1/4 内存）
   - 大图按需 scaled，不要整张常驻
2. **QPixmap / QPixmapCache**
   - `QPixmapCache::setCacheLimit()` 按设备 RAM 预算设上限
   - 停止使用后主动 `clear()`/释放引用的 QPixmap（底层是 GPU 纹理，须在窗口上下文内释放）
3. **隐式共享与 detach**
   - 传参用 `const QImage&`/`const QString&`，避免触发 COW detach
   - 只读遍历用 `qAsConst(list)`（Qt 5 时代）；Qt6 已无 detach 但注意拷贝仍在
   - 拼接尽量 `QStringBuilder`（`%1 %2` 或用 `+`），避免重复 `toUtf8()/toLocal8Bit()` 临时对象
4. **容器预留**
   - 已知规模的容器先 `reserve()`，避免多次 realloc 翻倍浪费（最高浪费 2x 峰值）
5. **延迟加载**
   - 大资源（NLU 模型、语音库、图片集）改为用时加载；`Preload` 完成后考虑释放不再需要的缓冲
6. **QByteArray/音频/视频缓冲**
   - 检查环形缓冲、播放缓冲的固定容量 vs 实际需求
7. **线程栈与线程数**
   - `pthread_attr_setstacksize` 收紧大栈线程；确认线程池大小与并发上限
8. **单例吞全局**
   - 静态/全局大对象（QImage、模型、缓存）改为懒加载 (`std::once_flag` / 首次访问时构造)

### 4.2 参考 `performance-patterns` skill

代码层面若伴随 CPU 热点，其模式目录（伪共享、缓存行布局、strided 访问）同时影响内存带宽与占用。

### 4.3 修复纪律

- 一次只改一处，重测量对比
- 修改要有可回退性（记录前后 VmRSS/VmHWM 数值到提交信息或文档）
- 若改了 3 处仍无收益，回到测量步骤重新定位，不要盲目继续

---

## 5. VERIFY / GUARD

- 对比同一 UI 操作序列下的 VmRSS、VmHWM、峰值分配（valgrind massif `Peak`）
- 建立可复现场景并自动化：重启 app → 跑 N 轮关键操作 → 采样 RSS 序列 → 断言 `VmHWM` 不超过阈值
- 在 CI/回归中用 massif/heaptrack 对关键路径做峰值断言

---

## 6. 设备部署验证（RK3588）

> 设备 IP 192.168.3.115，root/root，应用路径 `/usr/local/ensonic/bin/`，日志 `/userdata/log/frontend-app.log`
> 完整部署命令见 `device-testing` skill。

1. 交叉构建 Release（`build/make`，工具链 `crosstool/rk3588/.../aarch64-linux-g++`）
2. `sshpass -p root scp ...` 上传到 `/usr/local/ensonic/bin/appQTAcoustic`
3. 远程启动（设置 QT 环境变量 + Weston wayland-0），确认无回归（对比日志中 error/assert/crash）
4. 跑复现场景并采样 RSS（见 §2.6）

---

## 7. References

- `references/qt-memory-checklist.md` — Qt/C++ 逐类内存检查清单（含 QML 侧）
- `references/proc-memory-field-guide.md` — /proc 内存字段与 smem/perf/valgrind 参数速查

**何时不要用本 skill**：没有内存/性能诉求的常规编码；纯粹的 CPU 热点分析（用 `linux-perf`）；无测量条件的凭空"优化"。
