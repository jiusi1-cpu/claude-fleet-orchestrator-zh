# Claude 舰队调度器 · Claude Fleet Orchestrator

**用 Codex 当调度员，安全地驱动一队 Claude worker 并行干活。默认只跑 DryRun —— 不启动 Claude、不消耗 token、不碰业务代码。**

一套项目级的 agent 编排**规范**。重点不在「让 AI 多干活」，而在 **「让 AI 只干允许它干的那部分活」**。

---

## ⚠️ 先说清楚仓库构成

| 目录 | 是什么 | 来源 |
| --- | --- | --- |
| `Claude舰队调度器/` | 本项目的编排规范文档（原创） | 本仓库作者 |
| `superpowers/` | 第三方开源项目的**完整副本**，未作任何修改 | [obra/superpowers](https://github.com/obra/superpowers) · Jesse Vincent · MIT 许可 · v5.1.0 |

`superpowers` 保留其原始 LICENSE 与版权声明，仅作为通用技能库底座一并收录。

**关于完整度**：`Claude舰队调度器/SKILL.md` 是一份**规范与设计文档**，描述了编排流程、任务包格式与安全门禁。文档中引用的 PowerShell 脚本（`run_claude_worker.ps1`、`watch_claude_fleet.ps1`、`validate_claude_fleet.ps1` 等）**不包含在本仓库内** —— 它们属于使用这套规范落地时的具体项目。你可以直接照着这份文档在自己项目里实现。

---

## 它解决什么问题

让 AI agent 写代码，真正让人睡不着的往往不是「它写错了」，而是 **「它不受控制地写了」**：

- 动了不该动的文件
- 直接在你的 `main` 分支上改
- 静默跑掉一大笔 token
- 声称「完成了」，实际输出残缺

这份规范把这些约束**变成可校验的流程**，而不是靠提醒和自觉。

---

## 核心机制

### 角色分工：Codex 是调度员，Claude 是受管执行者

- **Codex**：读项目上下文 → 生成任务包 → 启动 worker → 跑 watcher → 审查 `output.json` → 决定下一步是否放行
- **Claude worker**：只处理**一个**任务包，只能在 `allowed_files` 里动手；不许 merge / rebase / push，不许用 API key，不许起长驻进程，只返回结构化 JSON

### 默认 DryRun

默认模式下**根本不启动 Claude**，只生成规划产物：
`prompt.txt`、`dryrun_prompt.txt`、`dryrun_plan.json`、`state.json`、`output.json`，跑完状态是 `dryrun_completed`。

先把整套流程空跑验证一遍，确认无误，再考虑真跑。

### 任务包就是一纸合同

每个 worker 拿到的任务包是 JSON（机器校验的唯一来源），字段包括：

`objective` · `read_first` · `allowed_files` · `forbidden_files` · `commands_to_run` · `acceptance_criteria` · `stop_conditions` · `max_minutes`，以及三个**默认关闭**的开关：网络、API key、长驻进程。

Markdown 模板只给人看，JSON 才是被校验的真相。

### 安全门禁（节选）

**任何一次运行前**：校验项目指纹、必需目录、RunId 与 WorkerName；运行路径必须落在 `agents/runs` 之下；拒绝重复 RunId；拒绝开启网络 / API key / 长驻进程。

**进入 Execute 模式前**：确认是 git 仓库；当前分支**不是** `main` / `master`；必须有隔离的 worktree 与分支；记录前后 git 快照；改动文件必须落在允许清单内。

### watcher 只读

监控脚本**从不启动 worker、从不改业务代码、从不 merge**。它支持单次巡检、校验 `output.json`，并会标记两类异常：

- `completed_but_invalid_output` —— 说自己完成了，但输出不合规
- `stale_running` —— 卡住了

---

## 怎么用这份规范

1. 通读 [`Claude舰队调度器/SKILL.md`](Claude舰队调度器/SKILL.md)
2. 在你的项目里落地目录结构，skill 放在 `.superpowers/skills/claude-fleet-orchestrator/`
3. 按文档实现三个脚本：任务包校验、worker 启动、只读 watcher
4. **首次导入只允许 DryRun**，第一个任务用只读审计类任务（如 `demo-readonly-audit.json`）验证全流程
5. DryRun 与 watcher 都通过、并且有人确认过隔离分支方案之后，才谈 Execute 模式

---

## 什么时候不许碰业务代码

只要满足下面任意一条，就不要进入业务代码测试：

- DryRun 校验没通过
- 项目当前在 `main` 或 `master` 上
- 任务包的文件边界过宽或缺失
- 请求了网络或 API key 权限
- 输出校验还没走完

---

## 许可

- 本仓库原创部分（`Claude舰队调度器/`）：MIT
- `superpowers/` 目录：[MIT](https://github.com/obra/superpowers)，版权归 Jesse Vincent
