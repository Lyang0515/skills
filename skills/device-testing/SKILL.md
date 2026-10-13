---
name: "device-testing"
description: "编译、部署、测试谛声科技 RK3588 嵌入式设备的 appQTAcoustic 应用。用于远程 SSH 部署、模拟 AI 按键/触摸屏、日志分析、语音流程验证。"
---

# 设备远程测试与调试 Skill

> 适用范围: 谛声科技 RK3588 嵌入式设备 (192.168.3.115)
> 更新日期: 2026-09-11

---

## 1. 设备信息

| 项目 | 值 |
|------|-----|
| IP | 192.168.3.115 |
| SSH 用户 | root |
| SSH 密码 | root |
| 平台 | RK3588 ARM64 (aarch64) |
| OS | Buildroot Linux |
| 显示 | Weston (Wayland) + DRM, DSI-1 rotate-270 |
| 设计分辨率 | 1280×720 |

### 关键路径

| 用途 | 路径 |
|------|------|
| App 二进制 | `/usr/local/ensonic/bin/appQTAcoustic` |
| App 日志 | `/userdata/log/frontend-app.log` |
| AI 配置 | `/userdata/config/ai_config.json` |
| NLU 数据 | `/usr/local/ensonic/AIChain/nlu/data/` |
| NLU 缓存 | `/usr/local/ensonic/AIChain/nlu/nlu_cache/` |
| 模拟触摸脚本 | `/userdata/app/auto_test/sim_touch.py` |
| 开机音频 | `/usr/local/ensonic/audio/startup.mp3` |

---

## 2. 编译与部署

### 2.1 编译

```bash
cd /home/luoshigang/work/code/qiandaohu_ui_dev/qiandaohu_ui/build
make -j$(nproc) 2>&1 | tail -20
```

产物: `/home/luoshigang/work/code/qiandaohu_ui_dev/qiandaohu_ui/bin/bin/appQTAcoustic`

交叉编译链: `crosstool/rk3588/atk-dlrk3588-toolchain/bin/aarch64-linux-g++`

### 2.2 部署

```bash
# 1. 停止旧进程
sshpass -p 'root' ssh -o StrictHostKeyChecking=no root@192.168.3.115 \
    'killall -9 appQTAcoustic 2>/dev/null; sleep 1; echo done'

# 2. 上传新版本
sshpass -p 'root' scp -o StrictHostKeyChecking=no \
    /home/luoshigang/work/code/qiandaohu_ui_dev/qiandaohu_ui/bin/bin/appQTAcoustic \
    root@192.168.3.115:/usr/local/ensonic/bin/appQTAcoustic

# 3. 备份旧版本（可选）
sshpass -p 'root' ssh root@192.168.3.115 \
    "cp /usr/local/ensonic/bin/appQTAcoustic /usr/local/ensonic/bin/appQTAcoustic_back_\$(date +%Y%m%d_%H%M%S)"
```

### 2.3 启动 App（SSH 远程启动）

由于 SSH 会话不继承 Weston 的环境变量，需要手动设置：

```bash
sshpass -p 'root' ssh -o StrictHostKeyChecking=no root@192.168.3.115 '
export QT_DIR=/usr/lib/qt6
export QT_PLUGIN_PATH=/usr/lib/qt6/plugins
export QML2_IMPORT_PATH=/usr/lib/qt6/qml
export QT_QPA_FONTDIR=/usr/share/fonts
export QT_FONT_DPI_OVERRIDE=96
export QT_ENABLE_HIGHDPI_SCALING=0
export XDG_RUNTIME_DIR=/run
export WAYLAND_DISPLAY=wayland-0
export LD_LIBRARY_PATH=/usr/local/qiandaohu/bin/:$LD_LIBRARY_PATH
nohup /usr/local/ensonic/bin/appQTAcoustic --platform wayland > /dev/null 2>&1 &
echo "PID=$!"
'
```

**注意**: 不要使用 `killall -9 appQTAcoustic` 后立即 SSH 执行其他命令，进程被杀可能导致设备网络短暂中断。

---

## 3. 模拟 AI 物理按键

### 3.1 原理

AI 按键是 GPIO 按键，通过 Linux input 子系统上报。按键设备为 `/dev/input/event10` (`adc-keys`)，KeyMonitor 通过 `EVIOCGRAB` 独占该设备后监听按键事件。

向同一设备文件写入 `input_event` 结构体注入按键事件，内核会将这些事件也分发给 KeyMonitor，最终到达 `AiVoiceManager::onAiKeyPressed/Released()`。

**Key code**: AI 按键 = `710` (定义在 [KeyMonitor.h:29](../../External/QTAcoustic/KeyMonitor.h#L29))

### 3.2 脚本模板

```python
#!/usr/bin/env python3
"""模拟 AI 按键事件 — 写入 /dev/input/event10 (adc-keys)"""
import struct, time, os

EV_KEY = 0x01
EV_SYN = 0x00
SYN_REPORT = 0x00
AI_KEY_CODE = 710

# struct input_event: timeval(8+8) + type(2) + code(2) + value(4) = 24 bytes
EVENT_FORMAT = 'llHHi'

def send_event(fd, ev_type, code, value):
    t = time.time()
    sec, usec = int(t), int((t - int(t)) * 1_000_000)
    data = struct.pack(EVENT_FORMAT, sec, usec, ev_type, code, value)
    os.write(fd, data)

def main():
    fd = os.open("/dev/input/event10", os.O_WRONLY)

    # ===== 按下 =====
    send_event(fd, EV_KEY, AI_KEY_CODE, 1)   # value=1 表示按下
    send_event(fd, EV_SYN, SYN_REPORT, 0)

    # ===== 保持（决定长按/短按）=====
    time.sleep(0.6)   # ≥500ms → 长按 → PTT 语音录入
    # time.sleep(0.3) # <500ms → 短按 → 打开/关闭面板

    # ===== 释放 =====
    send_event(fd, EV_KEY, AI_KEY_CODE, 0)   # value=0 表示释放
    send_event(fd, EV_SYN, SYN_REPORT, 0)

    os.close(fd)

if __name__ == "__main__":
    main()
```

### 3.3 长按/短按阈值

| 时长 | 识别为 | 行为 |
|------|--------|------|
| < 500ms | 短按 | Idle → 打开面板 / 其他 → togglePanel / Collapsed → expandFromCapsule |
| ≥ 500ms | 长按 | Idle → 打开面板 + PTT 录音 / Awake → PTT 录音 / Collapsed → 展开 + PTT |

---

## 4. 模拟触摸屏点击

### 4.1 原理

触摸设备为 `/dev/input/event6` (`chsc_cap_touch`)，支持多点触控协议 (ABS_MT)。使用设备上的 `sim_touch.py` 脚本注入触摸事件。

### 4.2 坐标映射（关键！）

设备 Weston 配置了 `transform=rotate-270`，物理触控坐标与逻辑渲染坐标不一致：

```
物理坐标 (x_phys, y_phys) ──rotate-270──→ 逻辑坐标 (x_logical, y_logical)

  x_logical = y_phys           （物理 Y → 逻辑 X，范围 0-1280）
  y_logical = 720 - x_phys     （物理 X 翻转 → 逻辑 Y，范围 0-720）

逆转换（逻辑 → 物理，即实际传给 sim_touch.py 的坐标）:

  y_phys = x_logical           （逻辑 X → 物理 Y）
  x_phys = 720 - y_logical     （逻辑 Y 翻转 → 物理 X）
```

**示例**: 点击逻辑坐标 `(1200, 24)` 处的按钮：
```bash
# x_phys = 720 - 24 = 696
# y_phys = 1200
python3 /userdata/app/auto_test/sim_touch.py click 696 1200 -d /dev/input/event6 -t 200
```

### 4.3 定位 UI 元素

**方法**: 批量探测 → 二分逼近

```bash
# 遍历坐标范围，观察日志变化
for x in 680 690 696 700 710; do
    python3 /userdata/app/auto_test/sim_touch.py click $x 1200 -d /dev/input/event6 -t 150
    sleep 1
done
# 然后检查日志中是否出现预期状态变化（如 收缩为胶囊条）
grep "会话状态.*→ 5\|收缩为胶囊条" /userdata/log/frontend-app.log | tail -1
```

**已知 UI 位置参考**（逻辑坐标）:

| 元素 | 逻辑坐标 (x, y) | 物理坐标 (x_phys, y_phys) |
|------|-----------------|--------------------------|
| 收回胶囊按钮 | ~(1200, 24) | (696, 1200) |
| 关闭面板按钮 | ~(1250, 24) | (696, 1250) |

---

## 5. 日志分析

### 5.1 关键日志模式

```bash
# 查看最新 N 行
tail -30 /userdata/log/frontend-app.log

# 过滤 AiVoice 相关
grep "AiVoice" /userdata/log/frontend-app.log | tail -20

# 过滤特定时间段
grep "06:4[3-9]" /userdata/log/frontend-app.log | grep "AiVoice"

# 检查错误
grep -iE "error|ERROR|assert|abort|crash|failed" /userdata/log/frontend-app.log | tail -20
```

### 5.2 状态转换验证

| 期望行为 | 日志关键字 |
|----------|-----------|
| AI 按键按下 | `AI 按键按下` |
| 长按触发 | `AI 按键长按 500ms` |
| 短按触发 | `AI 按键短按` |
| 进入录音 | `会话状态 → 2` + `对话模式: 开始录音` |
| 停止录音 | `会话状态 → 3` + `停止录音 → 进入识别` |
| 后端确认录音 | `voiceState → listening` |
| 唤醒 | `唤醒触发 source=long_press` |
| 面板打开 | `按键唤醒：KWS 已中断，面板已打开` |
| 胶囊条 | `收缩为胶囊条` + `会话状态 → 5` |
| 打开面板 | `会话状态 → 1` |
| 语音连接成功 | `VoiceInteractionAPI 连接成功` |
| Preload 完成 | `预加载完成（模型已就绪` |
| 开机音频播放 | `播放器 aplay 已启动` |
| 开机音频完成 | `播放完成` |
| 后端未就绪 | `后端尚未就绪` / `取消延迟 PTT` |

### 5.3 时序验证

检查开机音频是否在语音连接前完成：

```bash
echo "音频完成: $(grep '播放完成' /userdata/log/frontend-app.log | tail -1 | awk '{print $2,$3}')"
echo "语音连接: $(grep '连接成功' /userdata/log/frontend-app.log | tail -1 | awk '{print $2,$3}')"
echo "startVoice: $(grep '语音服务已启动' /userdata/log/frontend-app.log | tail -1 | awk '{print $2,$3}')"
```

**正确时序**: 音频完成 → `startVoiceService()` → `VoiceInteractionAPI 连接成功`

---

## 6. 常用测试场景

### 6.1 场景 A: 正常启动验证

```bash
# 1. 启动 app
# 2. 等待 15s
# 3. 验证
CHECK_AUDIO=$(grep '播放完成' /userdata/log/frontend-app.log | tail -1)
CHECK_VOICE=$(grep '连接成功.*ai_chain' /userdata/log/frontend-app.log | tail -1)
CHECK_SVC=$(grep '语音服务已启动' /userdata/log/frontend-app.log | tail -1)
echo "音频: $CHECK_AUDIO"
echo "startVoice: $CHECK_SVC"
echo "语音: $CHECK_VOICE"

# 4. 测试语音功能
python3 /tmp/test_longpress.py
sleep 3
grep "会话状态 → 2" /userdata/log/frontend-app.log | tail -1  # 应看到 Listening
```

**预期**: 音频完整播放 → startVoiceService → 语音连接 → 长按录音正常

### 6.2 场景 B: NLU 缓存重建

```bash
# 1. 清除缓存
ssh root@192.168.3.115 'rm -f /usr/local/ensonic/AIChain/nlu/nlu_cache/*'

# 2. 重启 app
# 3. 等待 130s（重建 ~1000 条 embedding 耗时约 120s）
# 4. 验证
ls -la /usr/local/ensonic/AIChain/nlu/nlu_cache/  # 应有 3 个文件
grep '连接成功.*ai_chain' /userdata/log/frontend-app.log | tail -1  # 应在 Preload 完成之后
```

**预期**: 音频完整播放 → startVoiceService 被 Preload 阻塞 → Preload 完成后自动补连

### 6.3 场景 C: 语音录入完整流程

```bash
# 1. 短按打开面板
python3 /tmp/test_shortpress.py
sleep 3

# 2. 长按录音
python3 /tmp/test_longpress.py
sleep 3

# 3. 检查状态链
grep "AI 按键\|会话状态" /userdata/log/frontend-app.log | tail -10
```

**预期状态链**: `Idle(0) → Awake(1) → Listening(2) → Processing(3) → Awake(1)`

### 6.4 场景 D: 胶囊条状态长按

```bash
# 1. 先进入胶囊条状态：短按打开面板 → 触摸点击"收回胶囊"按钮
python3 /tmp/test_shortpress.py && sleep 3
python3 /userdata/app/auto_test/sim_touch.py click 696 1200 -d /dev/input/event6 -t 200
sleep 2

# 2. 验证已进入胶囊条
grep "收缩为胶囊条\|会话状态 → 5" /userdata/log/frontend-app.log | tail -1

# 3. 长按 AI 键
python3 /tmp/test_longpress.py && sleep 3

# 4. 验证：胶囊条收回 → 面板打开 → 开始录音
grep "会话状态\|展开面板" /userdata/log/frontend-app.log | tail -5
```

**预期**: `Collapsed(5) → expandFromCapsule → Awake(1) → Listening(2) → Processing(3)`

---

## 7. 故障排查清单

| 症状 | 检查项 | 命令 |
|------|--------|------|
| SSH 连不上 | 网络 / 设备是否重启 | `ping 192.168.3.115` |
| App 未运行 | 进程是否存在 | `ps aux \| grep appQTAcoustic` |
| 语音不工作 | 后端是否连接 | `grep "连接成功.*ai_chain" /userdata/log/...` |
| 语音不工作 | isConnected() 是否为 false | `grep "后端未连接" /userdata/log/...` |
| 开机音频中断 | 语音连接是否在音频完成前 | 对比 `播放完成` 和 `连接成功` 的时间戳 |
| NLU 缓存异常 | 缓存文件是否存在 | `ls -la /usr/local/ensonic/AIChain/nlu/nlu_cache/` |
| Supervisor 挂掉 | qiandaohu 进程 | `ps aux \| grep qiandaohu \| grep -v island` |
| 触摸不响应 | 触摸设备是否正确 | `ls /dev/input/event6` |

---

## 8. Python 脚本部署

测试脚本存放于 `/tmp/` 目录，设备重启后自动清除。

```bash
# 本地编写脚本后上传
sshpass -p 'root' scp -o StrictHostKeyChecking=no /tmp/test_xxx.py root@192.168.3.115:/tmp/

# 远程执行
sshpass -p 'root' ssh root@192.168.3.115 'python3 /tmp/test_xxx.py'
```
