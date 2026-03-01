# skills

这个仓库用于存放可复用的 Agent Skills。每个 skill 通过 `SKILL.md` 定义触发条件、工作流和安全边界，供 AI 代理在对应任务中调用。

## Skills 列表

| Skill | 路径 | 简介 |
| --- | --- | --- |
| `mac-disk-cleaner` | `mac-disk-cleaner/SKILL.md` | 用于 macOS 磁盘清理，采用“只读扫描 -> 生成清理计划 -> 用户手动执行”流程；默认只输出低风险命令，并在检测到开发者环境时补充开发者候选项。 |

## 快速操作（`npx skills`）

`skills` CLI 可直接通过 `npx` 使用，无需全局安装。

### 1) 搜索技能

```bash
npx skills find disk cleaner
```

### 2) 安装整个 skills 仓库

```bash
npx skills add https://github.com/tianyk/skills.git
```

### 3) 仅安装指定 skill

```bash
npx skills add https://github.com/tianyk/skills.git --skill mac-disk-cleaner
```

### 4) 检查与更新已安装技能

```bash
npx skills check
npx skills update
```

更多 CLI 说明可参考：

- [skills npm 包页面](https://www.npmjs.com/package/skills)
- [Skills CLI 文档](https://skills.sh/docs/cli)
