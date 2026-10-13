# Qt/C++ 内存检查清单

按风险从高到低逐项排查。每一项都应有「对象数量 × 单对象大小」的估算，先排查数量多者。

## 1. 图像 / 像素数据（最常见大头）

| 检查项 | 做法 |
|--------|------|
| QImage 格式 | 无 Alpha 用 `Format_RGB32`；纯灰度用 `Format_Grayscale8`（比 ARGB32 省 3/4）；不透明度 1 的图避免 ARGB32_Premultiplied |
| 大图常驻 | 仅保存 scaled 后缩略图；原图用后 `flush()` 或释放引用 |
| QPixmapCache | 设置合理 `setCacheLimit`（按设备 RAM 预算，如 4GB 设备给 ~64MB）；权衡 UI 重绘成本 |
| QImage↔QPixmap 互转 | 每次互转都会产生新内存副本，避免在热路径反复互转 |
| GPU 纹理 | QPixmap 在窗口上下文存在时分配；remove 引用后显式让出，避免残留到切屏 |

## 2. 字符串 / 字节流

| 检查项 | 做法 |
|--------|------|
| 隐式共享 detach | 只读参数用 `const &`；`qAsConst()`；避免给共享对象写成员 |
| 反复编码 | 日志/网络热路径避免每帧 `toUtf8()`；复用 QByteArray 缓冲 |
| 逐字符追加 | 大字符串用 `QStringBuilder` 或 `append` 而不是 `"a" + "b" + ...` 多次临时拼接 |
| QStringLiteral / u"" | 常量字符串避免重复构造与堆分配 |
| 日志字符串 | 高频日志用格式化回调而非每行格式化为 QString |

## 3. 容器

| 检查项 | 做法 |
|--------|------|
| 未预留 | 已知规模先 `reserve()`，避免 realloc 翻倍浪费 |
| QList/QVector | Qt6 已统一，无差异；大的自定义结构用 `QVarLengthArray`（栈上） |
| QHash/QMap 误用 | QHash 比 QMap 省（无树节点开销）但注意加载因子；数量稳定用 `QHash::reserve` |
| 大对象按值存放 | `QList<T>` 放大的 T 时改放 `std::shared_ptr` 或指针，避免拷贝 |

## 4. 生命周期 / 缓存

| 检查项 | 做法 |
|--------|------|
| 全局/静态大对象 | 懒加载：`std::once_flag` 或首次访问时构造；确认构造函数不 preload 整库 |
| Preload 残留 | 启动预加载（模型/语音/NLU embedding）完成后，评估能否释放中间缓冲 |
| 定时任务持有 | 低频定时器若持有大缓存，定时 `squeeze`/`clear` |
| deleteLater | 大量 QObject 快速销毁时用 `deleteLater()` 避免栈帧内直接 delete 累积效 |
| 缓存失效策略 | 选择 TTL / 事件触发 / 版本键 三选一，避免多种碰撞 |

## 5. 线程

| 检查项 | 做法 |
|--------|------|
| 线程栈 | `RLIMIT_STACK` 默认 8MB×线程数；循环线程（如音频/采集）用 `pthread_attr_setstacksize(1~2MB)` |
| 线程池上限 | 确认线程数与设备核心匹配，不会随任务无限增长 |
| 栈上大数组 | 检查栈上是否分配大 `QByteArray`/数组，移到堆或成员 |

## 6. QML 侧（若有）

| 检查项 | 做法 |
|--------|------|
| ListView delegate 持有图像 | delegate 中 QImage 直接加载全图 → 改为异步缩放 + 付费 thumbnail |
| Image.source 原图 | 使用 fs 缩放后的资源；`sourceSize` 控制解码尺寸 |
| 重复创建组件 | `Loader`/`Repeater` 数量上限；Component 缓存 |
| 动画/缓存多余纹理 | 检查 Offscreen/材质是否不必要的分配 |

## 7. 进程 / 系统层面

| 检查项 | 做法 |
|--------|------|
| 共享库 | 确认无重复符号加载、无人为 dlopen 泄漏 |
| 文件描述符 + mmap | 大量 QFile/QTemporaryFile 遗留会占用 RSS |
| 设备 ta 上下文 | 触控/视频/音频设备句柄关闭后释放关联缓冲 |