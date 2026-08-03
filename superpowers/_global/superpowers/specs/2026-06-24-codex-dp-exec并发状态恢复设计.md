# codex-dp-exec 并发状态恢复设计

## 背景

当前混合执行流程采用 GPT-5.5 作为主控、计划者和审核者，使用 `codex-dp-exec` 调用 DeepSeek 执行具体编码任务。现有脚本已经可以同步执行并通过 `--output-last-message` 产出报告，但长期任务、多 GPT 主控并发、电脑宕机恢复和执行状态追踪还缺少稳定约束。

本设计目标是增强 `codex-dp-exec` 的状态管理能力，让多个 GPT 主控可以在不同 `git worktree` 中并行派发 DeepSeek 实现任务，同时保证状态文件不互相覆盖、同一工作目录不会被并发写入、宕机后可以恢复到可审查和可重跑的任务状态。

## 目标

- 支持多个 GPT 主控同时使用 `codex-dp-exec` 执行不同任务。
- 每次执行都有唯一 `run_id` 和独立运行目录，状态文件、日志和报告互不覆盖。
- 同一个 `git worktree` 默认只允许一个 `codex-dp-exec` 写入，避免代码改动混杂。
- 同一仓库下的不同 `git worktree` 可以并行执行，符合全局 worktree 多任务隔离规范。
- 运行状态不放入 `/tmp`，避免系统清理导致长任务记录丢失。
- 宕机后不恢复已消失的模型进程，而是将未完成 run 标记为 `interrupted`，再基于同一任务创建新的 recovery run。

## 非目标

- 不实现一个长期运行的中心调度服务。
- 不让一个旧的 DeepSeek 推理进程从中间继续执行；宕机后只能恢复任务语义，不能恢复模型内部上下文。
- 不支持多个 GPT 主控在同一个 worktree 中同时写代码作为正常路径。
- 不修改现有 `codex-dp` alias/function。
- 不改变 GPT-5.5 主控负责设计、计划、审核和用户决策的职责分工。

## 状态目录

运行状态目录放到项目对应的 `*-superpowers-docs` 根目录下，而不是 `/tmp` 或项目源码目录中的临时目录。

goto-software 项目的默认目录为：

```text
/mnt/f/IdeaProjects/goto-software/goto-docs/superpowers/_global/superpowers/runs/codex-dp-exec/
```

后续如果其他项目有自己的项目规范，应优先使用项目规范中声明的 `*-superpowers-docs` 根目录，再在其下创建 `runs/codex-dp-exec/`。不同项目的规格目录结构可能略有差异，因此实现时不应硬编码单一目录格式，应允许通过参数或项目规则配置 run root。

推荐任务目录结构：

```text
codex-dp-exec/
  task-003/
    task.json
    runs/
      20260624-104512-12345-a8f3c1/
        prompt.md
        status.json
        heartbeat.json
        stdout.log
        stderr.log
        report.md
        exit_code
        command.txt
      20260624-112030-23456-b9e4d2/
        prompt.md
        status.json
        heartbeat.json
        stdout.log
        stderr.log
        report.md
        exit_code
        command.txt
```

`task_id` 表示一个可恢复的业务任务，`run_id` 表示一次实际执行。宕机或失败后，同一个 `task_id` 可以拥有多个 run，旧 run 保留为诊断记录。

## run_id 与并发安全

默认 `run_id` 使用时间、进程号和随机串生成：

```text
YYYYMMDD-HHMMSS-<pid>-<random>
```

创建 run 目录时使用原子 `mkdir`。如果用户显式传入 `--run-id` 且目标目录已经存在，默认直接失败，不覆盖已有状态。只有未来明确设计 `--resume` 或 `--force` 时，才允许复用旧目录。

## 状态文件

`status.json` 记录一次 run 的核心状态：

```json
{
  "task_id": "task-003",
  "run_id": "20260624-104512-12345-a8f3c1",
  "status": "running",
  "controller_id": "codex1",
  "workdir": "/mnt/f/IdeaProjects/goto-software/worktrees/example/feat/001-example",
  "branch": "feat/001-example",
  "base_sha": "abc123",
  "pid": 12345,
  "started_at": "2026-06-24T10:45:12+08:00",
  "ended_at": null,
  "exit_code": null
}
```

状态流转：

```text
created -> running -> succeeded
created -> running -> failed
created -> running -> interrupted
```

写入 `status.json` 和 `heartbeat.json` 时应先写临时文件，再用 `mv` 原子替换，避免 GPT 主控读取到半截 JSON。

## worktree 校验与锁

`codex-dp-exec` 启动时必须记录并校验 `workdir`：

- 使用 `realpath(workdir)` 作为锁粒度。
- 不使用 Git common dir 作为锁粒度，因为同一个仓库的多个 worktree 会共享 common dir，按 common dir 加锁会错误阻塞合法并行任务。
- 同一个 `realpath(workdir)` 同一时间只能存在一个有效写入锁。
- 不同 worktree 可以并行执行，即使它们属于同一个 Git 仓库。

锁文件建议存放在 run root 下的 `locks/` 目录，文件名使用 `realpath(workdir)` 的哈希值：

```text
locks/
  <sha256-realpath-workdir>.lock
```

锁文件内容：

```json
{
  "run_id": "20260624-104512-12345-a8f3c1",
  "task_id": "task-003",
  "workdir": "/mnt/f/IdeaProjects/goto-software/worktrees/example/feat/001-example",
  "pid": 12345,
  "boot_id": "current-linux-boot-id",
  "created_at": "2026-06-24T10:45:12+08:00"
}
```

如果第二个 GPT 主控误用同一个 worktree，脚本应失败并提示切换到独立 worktree。除非未来显式提供高级参数，否则默认不允许跳过该保护。

## 心跳与宕机判断

运行期间定期更新 `heartbeat.json`：

```json
{
  "run_id": "20260624-104512-12345-a8f3c1",
  "pid": 12345,
  "boot_id": "current-linux-boot-id",
  "hostname": "machine-name",
  "workdir": "/mnt/f/IdeaProjects/goto-software/worktrees/example/feat/001-example",
  "updated_at": "2026-06-24T10:46:12+08:00"
}
```

恢复时根据 `status.json`、`heartbeat.json`、`pid`、`boot_id`、`report.md` 和当前 `git diff` 判断旧 run 状态。

判断规则：

- `status = succeeded` 且 `report.md` 存在：进入 GPT 主控审核。
- `status = failed`：读取 stdout、stderr、exit_code 和 report，决定是否重试。
- `status = running` 且 `boot_id` 已变化：标记旧 run 为 `interrupted`。
- `status = running` 且 pid 不存在，并且 heartbeat 超时：标记旧 run 为 `interrupted`。
- 锁文件存在但 pid 或 boot 已失效：视为 stale lock，可以清理。
- 未完成任务需要继续时，创建新的 recovery run，不复用旧 `run_id`。

## 恢复策略

恢复粒度是 task，不是模型进程。

电脑宕机后，旧 DeepSeek 推理进程已经消失，不能从中间继续。GPT 主控恢复时应先审查任务目录和 worktree：

1. 找到目标 `task_id` 的最近 run。
2. 判断最近 run 是 `succeeded`、`failed`、`interrupted` 还是仍在运行。
3. 如果已有完整 `report.md`，先进入 GPT 主控审核。
4. 如果没有完整报告，检查当前 worktree 的 `git diff`。
5. 如果 worktree 有部分改动，GPT 主控决定保留后继续补完、人工整理，或让 DeepSeek 基于当前 diff 修复。
6. 如果需要重新执行，创建新的 recovery run，并在 prompt 中明确说明这是对旧 task 的恢复执行。

默认不自动回滚代码、不自动覆盖旧报告、不自动删除旧 run。所有恢复行为都应保留审计记录。

## GPT 主控查询方式

GPT 主控可以通过以下文件主动查询长任务状态：

- `status.json`：判断 run 当前状态。
- `heartbeat.json`：判断进程是否仍活跃。
- `stdout.log`：查看模型执行输出。
- `stderr.log`：查看 CLI 和运行错误。
- `report.md`：读取 DeepSeek 最终报告。
- `exit_code`：判断脚本退出结果。

同步执行时，GPT 主控可以等待命令结束后读取 `report.md`。未来如果扩展后台执行，可以通过轮询 `status.json` 和 `heartbeat.json` 实现主动查询。

## 参数建议

第一版建议为 `codex-dp-exec` 增加或保留以下参数：

```text
--cd <workdir>
--prompt-file <file>
--output-last-message <report-file>
--task-id <task-id>
--run-root <dir>
--run-id <run-id>
--controller-id <id>
--dry-run
```

默认行为：

- 未提供 `--run-root` 时，根据项目规则推导；无法推导时使用用户级持久目录作为降级方案。
- 未提供 `--task-id` 时自动生成临时 task id。
- 未提供 `--run-id` 时自动生成唯一 run id。
- 默认启用同 worktree 锁。
- 默认将状态、日志和报告写入 run 目录。

## 验收标准

- 同一个 GPT 主控连续执行两次任务时，生成两个不同 run 目录。
- 两个 GPT 主控在不同 worktree 执行时，可以并行运行，状态和日志互不覆盖。
- 两个 GPT 主控误用同一个 worktree 时，第二个执行失败，并提示当前 worktree 已有活动 run。
- `status.json`、`heartbeat.json`、`stdout.log`、`stderr.log`、`report.md`、`exit_code` 和 `command.txt` 能完整记录一次执行。
- 模拟 stale lock 时，恢复逻辑能识别 pid 或 boot 已失效，并允许清理旧锁。
- 模拟 `running` 状态但 heartbeat 超时或 boot 变化时，旧 run 被标记为 `interrupted`。
- 对 `interrupted` task 重新执行时，会创建新的 recovery run，而不是覆盖旧 run。

## 风险与约束

- 如果 DeepSeek 已经写入部分代码但未产出报告，GPT 主控必须审查 `git diff` 后再决定下一步。
- 如果用户绕过 worktree 规则，在同一目录手动启动多个实现器，仍可能造成代码冲突，因此默认锁必须开启。
- 不同项目的 `*-superpowers-docs` 目录结构并不完全一致，实现时需要允许显式配置 run root。
- 状态文件记录的是执行过程，不代表代码质量或需求符合性；最终仍必须由 GPT-5.5 主控审核。
