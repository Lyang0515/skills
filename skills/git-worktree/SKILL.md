---
name: "git-worktree"
description: "管理 git worktree 实现多分支并行开发。用于创建、查看、删除 worktree，基于 develop/main 创建 feature/bugfix 分支独立开发，互不干扰。适用场景：需要同时开发多个功能或修复 bug，不想频繁切换分支。"
---

# Git Worktree 并行开发 Skill

> 适用场景: 需要同时在多个分支上开发不同功能/bugfix，互不干扰
> 适用范围: `/home/luoshigang/work/code/qiandaohu_ui_dev/qiandaohu_ui`
> 更新日期: 2026-07-17

---

## 1. 什么是 Git Worktree

`git worktree` 允许从**同一个 Git 仓库**创建多个独立的工作目录，每个目录可以检出不同的分支，共享同一个 `.git` 数据仓库。

```
示意图：
  qiandaohu_ui/              ← 主 worktree (feature/bluetooth-file-tranfer)
  qiandaohu_ui_bugfix/       ← 新 worktree (bugfix/xxx)
  qiandaohu_ui_feature2/     ← 新 worktree (feature/yyy)
          ↓ 都指向同一个 ↓
  qiandaohu_ui/.git/         ← 共享的 Git 数据仓库
```

**优点**：
- 不需要反复 stash / commit / checkout 切换分支
- 多个功能并行开发，互不干扰
- 共享 `.git` 数据，不占用额外磁盘空间（除了工作目录的文件）
- 编译产物各自独立，不会互相覆盖

**缺点**：
- 同一个分支不能被两个 worktree 同时检出
- 每个 worktree 需要单独编译（磁盘占用 ~编译产物大小）

---

## 2. 项目 Git 仓库信息

| 项目 | 值 |
|------|-----|
| Git 仓库路径 | `/home/luoshigang/work/code/qiandaohu_ui_dev/qiandaohu_ui` |
| 默认主分支 | `develop` |
| 基础分支 | `main` |
| Worktree 存放目录 | `/home/luoshigang/work/code/qiandaohu_ui_dev/` |
| 当前主 worktree | `qiandaohu_ui` (feature/bluetooth-file-tranfer) |

---

## 3. 常用操作

### 3.1 前置步骤：处理当前未提交的修改

在创建新 worktree 之前，确保当前工作目录干净（否则可能遇到冲突）：

```bash
cd /home/luoshigang/work/code/qiandaohu_ui_dev/qiandaohu_ui

# 方案 A: 提交（推荐——WIP 提交后续可以通过 git rebase -i 整理）
git add -A
git commit -m "WIP: 中间状态 - $(date +%Y%m%d_%H%M%S)"

# 方案 B: 暂存（适合完全不想提交的情况）
git stash push -m "WIP: 中间状态"
```

### 3.2 查看所有 worktree

```bash
cd /home/luoshigang/work/code/qiandaohu_ui_dev/qiandaohu_ui
git worktree list
```

输出示例：
```
/home/luoshigang/work/code/qiandaohu_ui_dev/qiandaohu_ui          bba8b6e [feature/bluetooth-file-tranfer]
/home/luoshigang/work/code/qiandaohu_ui_dev/qiandaohu_ui_bugfix   a1b2c3d [bugfix/xxx]
```

### 3.3 创建新 worktree（最常用）

#### 场景 A：基于已有分支创建新 worktree（修复 develop 上的 bug）

```bash
cd /home/luoshigang/work/code/qiandaohu_ui_dev/qiandaohu_ui

# 基于 develop 创建新 bugfix 分支的 worktree
git worktree add ../qiandaohu_ui_bugfix -b bugfix/你的bug名称 develop
```

**参数说明**：
- `../qiandaohu_ui_bugfix` — 新 worktree 的目录路径（放在 qiandaohu_ui_dev 下）
- `-b bugfix/xxx` — 创建新分支（如果分支名已存在会报错，去掉 `-b` 可检出已有分支）
- `develop` — 基于哪个分支创建

#### 场景 B：基于已有功能分支创建 worktree（继续开发已有功能）

```bash
# 检出已有的功能分支到新 worktree（不创建新分支）
git worktree add ../qiandaohu_ui_feat feature/AI-chain-test
```

#### 场景 C：基于 main 创建 release 分支

```bash
git worktree add ../qiandaohu_ui_release -b release/v1.2.3 main
```

### 3.4 移除 worktree

```bash
cd /home/luoshigang/work/code/qiandaohu_ui_dev/qiandaohu_ui

# 1. 先清理 worktree 引用
git worktree remove ../qiandaohu_ui_bugfix

# 2. 如果上面命令报错（比如目录已手动删除），强制清理引用
git worktree prune

# 3. 删除分支（如果不再需要）
git branch -d bugfix/xxx
```

### 3.5 在新 worktree 中工作

```bash
# 切换到新 worktree 目录
cd /home/luoshigang/work/code/qiandaohu_ui_dev/qiandaohu_ui_bugfix

# 日常 git 操作跟正常一样
git status
git add -A
git commit -m "fix: 修复 xxx 问题"

# 拉取/推送
git pull origin develop
git push origin bugfix/xxx
```

---

## 4. 完整工作流示例

假设场景：当前在 `feature/bluetooth-file-tranfer` 开发蓝牙功能，同时需要修复 `develop` 上发现的一个 bug。

### Step 1: 保存当前工作

```bash
cd /home/luoshigang/work/code/qiandaohu_ui_dev/qiandaohu_ui
git add -A
git commit -m "WIP: 蓝牙文件传输 - 继续开发中"
```

### Step 2: 创建 bugfix worktree

```bash
git worktree add ../qiandaohu_ui_bugfix -b bugfix/fix-some-issue develop
```

### Step 3: 两个目录并行开发

```
目录 1: /home/luoshigang/work/code/qiandaohu_ui_dev/qiandaohu_ui
分支:   feature/bluetooth-file-tranfer
任务:   继续开发蓝牙功能

目录 2: /home/luoshigang/work/code/qiandaohu_ui_dev/qiandaohu_ui_bugfix
分支:   bugfix/fix-some-issue
任务:   修复 bug
```

在 VSCode 中可以通过 `File → Open Folder` 分别打开两个目录，用两个窗口并行工作。

### Step 4: 完成后清理

```bash
# bugfix 合并后，清理 worktree
cd /home/luoshigang/work/code/qiandaohu_ui_dev/qiandaohu_ui
git worktree remove ../qiandaohu_ui_bugfix
git branch -d bugfix/fix-some-issue
```

---

## 5. 注意事项

### 5.1 同一个分支不能同时在两个 worktree 中检出

```bash
# ❌ 错误：develop 已在主 worktree 检出，不能再在另一个中检出
git worktree add ../qiandaohu_ui_dev develop

# ✅ 正确：基于 develop 创建新分支
git worktree add ../qiandaohu_ui_dev -b bugfix/xxx develop
```

### 5.2 编译产物各自独立

每个 worktree 的编译产物互不影响，这是优点也是代价：
- 优点：不会因为切换分支导致编译缓存失效
- 代价：每个 worktree 的 `build/` 目录需要单独编译，占用更多磁盘

```bash
# 在新 worktree 中需要重新编译
cd /home/luoshigang/work/code/qiandaohu_ui_dev/qiandaohu_ui_bugfix
mkdir -p build && cd build
cmake .. && make -j$(nproc)
```

### 5.3 子模块同步

如果项目使用了 git submodules，每个 worktree 的子模块是独立的，需要在各自目录中更新：

```bash
cd /home/luoshigang/work/code/qiandaohu_ui_dev/qiandaohu_ui_bugfix
git submodule update --init --recursive
```

---

## 6. 故障排查

| 症状 | 原因 | 解决 |
|------|------|------|
| `fatal: 'xxx' already exists` | 分支名已存在 | 改分支名，或不加 `-b` 直接检出已有分支 |
| `fatal: 'xxx' is already checked out` | 该分支已在另一个 worktree 中 | 在另一个 worktree 中切换到其他分支后再试 |
| `fatal: working tree already exists` | 目录已存在 | 换个目录名，或删除已有目录 |
| 创建 worktree 后 pull 失败 | 分支追踪未设置 | `git branch -u origin/分支名` |
| `git worktree list` 显示已删除的目录 | 引用残留 | `git worktree prune` |

---

## 7. 一键命令速查

```bash
# ==== 项目根目录（先 cd 到这里）====
cd /home/luoshigang/work/code/qiandaohu_ui_dev/qiandaohu_ui

# ==== 查看 ====
git worktree list                              # 列出所有 worktree
git branch -a                                  # 列出所有分支

# ==== 创建 ====
# 场景: 从 develop 创建 bugfix worktree
git worktree add ../qiandaohu_ui_bugfix -b bugfix/名称 develop

# 场景: 检出已有分支到 worktree
git worktree add ../qiandaohu_ui_新目录 已有分支名

# ==== 清理 ====
git worktree remove ../qiandaohu_ui_bugfix     # 移除 worktree
git worktree prune                             # 清理残留引用
git branch -d 分支名                            # 删除分支

# ==== 提交 ====
git add -A
git commit -m "WIP: 描述"
```

---

## 8. 与 Claude Code 配合使用

当你在 Claude Code 中使用 worktree 时：

1. **用 `/git-worktree` 触发此 skill**
2. 告诉 Claude 你要做什么：
   - "帮我创建一个基于 develop 的 bugfix worktree"
   - "帮我查看当前有哪些 worktree"
   - "帮我清理 xxx worktree"
3. Claude 会根据此 skill 中的命令自动执行操作

**注意**：Claude Code 的 `EnterWorktree` 工具是内部机制，与本 skill 的 `git worktree` 命令是独立的。本 skill 教的是用原生 `git worktree` 命令管理多分支并行开发。