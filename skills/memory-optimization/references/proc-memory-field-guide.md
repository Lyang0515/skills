# /proc 内存字段与采样工具速查

## 1. /proc/<pid>/status 字段

| 字段 | 含义 | 何时关注 |
|------|------|----------|
| VmPeak | 进程历史峰值**虚拟**内存 | 大映射（纹理/库） |
| VmSize | 当前虚拟内存 | 一般参考 |
| VmRSS | 当前常驻物理内存 | **首要指标** |
| RssAnon | RSS 中匿名页（堆/栈/数据） | 堆泄漏主要在此 |
| RssFile | RSS 中文件映射页（可被回收） | 库/二进制 |
| RssShmem | RSS 中共享内存 | 多进程共享缓冲 |
| VmHWM | 历史峰值常驻内存 | **最重要：真实物理峰值** |
| VmSwap | 换出部分 | 内存紧张信号 |
| VmData | 数据段+堆 | malloc 增长 |
| VmStack | 栈 | 栈溢出/大栈线程 |
| Threads | 线程数 | 线程栈总量估算 |

## 2. 采样工具对比

| 工具 | 用途 | 限制 |
|------|------|------|
| `grep VmRSS /proc/pid/status` | 最轻量采样 | 单点，需循环 |
| `ps -o rss` | 快照列表 | 只有 RSS |
| `smem` | USS/PSS/RSS 分解 | 板子需安装 |
| `perf mem` | 硬件内存采样（cache miss/负载延迟、malloc 热点） | 需要 PMU/板卡支持 |
| `valgrind massif` | 堆分配热点+峰值调用链 | 慢（~20x），需 -g，板子一般不装，主机调试用 |
| `heaptrack` | massif 的现代替代，更快 | 同上 |
| `malloc_info` | 运行中 glibc 堆统计 | glibc 专有；musl 无 |
| `/proc/meminfo` + `free` | 系统级内存 | 进程级需配合 |

## 3. 常用操作模板

```bash
# 进程 RSS + HWM 单点
PID=<pid>
grep -E "^(VmRSS|VmHWM|VmSwap)" /proc/$PID/status

# 60s 连续采样（看趋势/峰值）
for i in $(seq 1 20); do
  awk '/^VmRSS|^VmHWM/{printf "%s=%dKB ", $1, $2} END{print ""}' /proc/$PID/status
  sleep 3
done

# 系统内存要点
free -m; echo ---; cat /proc/meminfo | grep -E "MemTotal|MemAvailable|SwapTotal|SwapFree"

# 单个进程排序
ps -eo pid,rss,comm --sort=-rss | head
```

## 4. massif 常用参数

```bash
valgrind --tool=massif \
  --stacks=yes \          # 含栈（默认关，更慢）
  --time-unit=B \         # 按字节时间轴（内存压力型）或 ms（时长型）
  --depth=30 \            # 调用链深度，越大越精确越慢
  --peak-inaccuracy=1.0 \ # 峰值精确度（默认 1%，越小越准）
  ./prog <args>
ms_print massif.out.<pid>
```

heaptrack 替代：

```bash
heaptrack ./prog <args>
heaptrack --pid <pid> -o out  # 附加到运行中进程
heaptrack_print out.zst
```

## 5. perf mem 参数

```bash
# 数据采样（需 root/perf_event_paranoid 放行）
perf mem record -p <PID> -- sleep 60
perf mem report --sort=comm,dso,sym

# 只看 load/store
perf mem record --type=load -p <PID> -- sleep 60
```