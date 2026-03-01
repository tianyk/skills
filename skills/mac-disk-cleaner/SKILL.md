---
name: mac-disk-cleaner
description: 通过“Agent 自动扫描 -> Agent 自动分析并生成报告 -> 用户手动执行最终清理”安全清理 macOS 磁盘空间。默认输出普通用户低风险清理项；在检测到开发者环境或用户要求时，追加开发者清理候选项。仅允许白名单命令，禁止高风险路径删除。
---

# Mac Disk Cleaner

用于 macOS 磁盘清理场景，目标是先定位空间占用，再给出可执行且可解释的清理方案。

## 适用场景

- 磁盘空间不足，需要安全释放空间
- 希望先看占用分布，再决定执行哪些清理动作
- 需要同时覆盖普通用户与开发者常见占用

## 运行模式

- `Standard`（默认）：普通用户常见占用，输出低风险可执行命令
- `Developer add-on`（可选）：检测到 `docker`/`brew`/`npm`/`pnpm`/`yarn`/`flutter`/`xcode` 或用户明确要求时启用

生成计划时必须先声明：

- `Standard: enabled`
- `Developer add-on: enabled|disabled`（说明原因）

## 强制安全规则

### 禁止输出删除命令的路径

- `~/Pictures/Photos Library.photoslibrary`
- `~/Library/Mail`
- `~/Library/Keychains`
- `~/Library/Messages`
- `/System`
- `/Library`
- 任何用途不明确的 `~/Library/Application Support/*`

### 命令输出约束

- 只能使用“允许命令白名单”中的命令
- 每条可执行命令必须附带：
- 预计释放空间（基于扫描结果估算）
- 副作用说明
- 执行后验证命令：`df -h /`

### 风险处理

- `low`：可输出命令
- `medium`/`high`：默认不给命令，只给建议和确认点
- 用户明确要求 `medium`/`high` 命令时：
- 先写风险和确认条件（如已备份、确认不再需要、理解后果）
- 再给命令，优先官方工具命令

## 工作流

### Step 1: Auto Scan（Agent 自动执行）

Agent 应自动运行“只读扫描命令”并收集结果，不要求用户手动执行扫描。

执行要求：

- 仅执行下文列出的只读扫描命令
- 某个命令不存在或失败时继续执行其余命令
- 在报告中标记缺失工具或扫描失败项（例如 `docker`/`brew` 不存在）

### Step 2: Auto Analysis & Report（Agent 自动分析并生成报告）

- 自动识别 `Standard` 和 `Developer add-on` 是否启用
- 先输出 `Standard` 候选项，再按条件追加 `Developer add-on`
- 按“预计可释放空间”降序排序
- 标注风险级别：`low`/`medium`/`high`
- 仅 `low` 给命令，`medium/high` 仅建议+确认点

### Step 3: Execute（用户手动执行）

用户手动复制执行最终清理命令，并用 `df -h /` 校验结果。

## 只读扫描命令

### 磁盘与目录概览

```bash
df -h /
du -h -d 2 ~ 2>/dev/null | sort -hr | head -n 30
```

### Standard 扫描

```bash
du -sh ~/Library/Caches 2>/dev/null
du -sh ~/Library/Logs 2>/dev/null
du -sh ~/Library/Developer 2>/dev/null
du -sh ~/Downloads 2>/dev/null
find ~/Downloads -type f -size +1024M -maxdepth 2 2>/dev/null | head -n 30
du -sh ~/Library/Application\ Support/MobileSync/Backup 2>/dev/null
du -sh ~/.Trash 2>/dev/null
```

### 开发者环境检测与扫描（均只读）

```bash
# Xcode / iOS
du -sh ~/Library/Developer/Xcode/DerivedData 2>/dev/null
du -sh ~/Library/Developer/Xcode/Archives 2>/dev/null
du -sh ~/Library/Developer/CoreSimulator 2>/dev/null
du -sh ~/Library/Developer/Xcode/iOS\ DeviceSupport 2>/dev/null

# Docker
docker system df 2>/dev/null

# Homebrew
brew --cache 2>/dev/null && du -sh "$(brew --cache)" 2>/dev/null
brew cleanup -n 2>/dev/null | head -n 80

# Node caches
npm config get cache 2>/dev/null && du -sh "$(npm config get cache)" 2>/dev/null
pnpm store path 2>/dev/null && du -sh "$(pnpm store path)" 2>/dev/null
yarn cache dir 2>/dev/null && du -sh "$(yarn cache dir)" 2>/dev/null

# Flutter / Dart
du -sh ~/.pub-cache 2>/dev/null
```

## 允许命令白名单

只允许输出以下命令；不得扩展到白名单外命令。

### Standard（默认）

```bash
# Empty Trash (low)
rm -rf ~/.Trash/*

# Browser caches only (low)
rm -rf ~/Library/Caches/com.apple.Safari/*
rm -rf ~/Library/Caches/Google/Chrome/*
rm -rf ~/Library/Caches/com.microsoft.Edge/*
```

约束：

- 不允许“一刀切”删除整个 `~/Library/Caches`
- `Downloads` 默认不输出批量删除命令，只建议用户先按大小筛选后手动删除
- `MobileSync/Backup` 默认为 `medium`，仅建议不直接给命令

### Developer add-on（可选）

```bash
# Xcode
rm -rf ~/Library/Developer/Xcode/DerivedData/*

# Node package managers
npm cache clean --force
pnpm store prune
yarn cache clean

# Homebrew
brew cleanup -n
brew cleanup -s
brew autoremove

# Docker
docker builder prune

# Flutter (inside project dir)
flutter clean
```

## 默认仅建议（medium/high）

### Standard

- `~/Library/Application Support/MobileSync/Backup`（删除后可能影响恢复/迁移）
- `Downloads` 批量清理（误删风险高）

### Developer

- `~/Library/Developer/CoreSimulator`
- `~/Library/Developer/Xcode/iOS DeviceSupport`
- `~/Library/Developer/Xcode/Archives`
- `docker system prune` / `docker system prune --volumes`

## 报告输出格式（必须遵循）

### Enabled modes

- `Standard: enabled`
- `Developer add-on: enabled|disabled`（给出原因）

### Summary

- 当前磁盘：总量/已用/可用（来自 `df -h /`）
- 预计可释放：`X GB`（仅统计 `low` 可执行项）

### Scan status

- 已成功执行的扫描项
- 执行失败或缺失命令的扫描项（含原因）

### Actions (low risk, executable)

按预计释放空间降序。每项必须包含：

- 名称 + 预计释放空间
- 副作用
- 命令（白名单内）

### Candidates (medium/high, advise only)

- 体积
- 风险说明
- 执行前确认点

### Verification

```bash
df -h /
```

## 触发示例

- 按这个 skill 自动扫描并生成清理报告：先给普通用户低风险命令；如果检测到开发者环境，再追加开发者候选项；中高风险只给建议不直接给命令；最终清理由我手动执行。
