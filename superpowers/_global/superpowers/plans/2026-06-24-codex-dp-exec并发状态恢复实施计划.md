# codex-dp-exec 并发状态恢复 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `hybrid-subagent-driven-development` to implement this plan task-by-task. If the hybrid skill is unavailable, use `superpowers:subagent-driven-development` or `superpowers:executing-plans`. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为 `/home/gyq/.local/bin/codex-dp-exec` 增加持久化运行状态、worktree 级并发锁、心跳日志和宕机后任务恢复能力。

**Architecture:** 保持单文件 Bash 脚本边界，不引入中心调度服务。脚本在每次执行时创建 `task_id/run_id` 目录，将 prompt、命令、状态、心跳、日志、报告和退出码写入项目 `*-superpowers-docs` 下的持久目录，并用 `realpath(workdir)` 的哈希作为 worktree 写锁。GPT-5.5 主控仍负责计划、派发、审核和恢复决策，DeepSeek 只负责被派发的实现任务。

**Tech Stack:** Bash、Codex CLI、Python 3 标准库 JSON、Linux `/proc/sys/kernel/random/boot_id`、`sha256sum`、`git`、现有 `codex-dp-exec` 脚本。

---

## 文件结构

- Modify: `/home/gyq/.local/bin/codex-dp-exec`
  - 保留当前 CLI 功能。
  - 新增 `--task-id`、`--run-root`、`--run-id`、`--controller-id` 参数。
  - 新增 run root 推导、状态 JSON 写入、worktree 锁、心跳、日志、报告落盘、stale run/lock 恢复。
- Modify: `/home/gyq/.agents/skills/hybrid-subagent-driven-development/SKILL.md`
  - 更新调用示例，要求 GPT 主控传入 `--task-id`、`--controller-id`，并优先读取 run 目录里的报告和状态。
- Test only: 临时目录 `/tmp/codex-dp-exec-test-*`
  - 使用 fake `CODEX_BIN` 离线测试，不提交。
  - 最后一项执行真实 `codex-dp-exec` smoke test。

## Task 1: 参数解析与 run root 推导

**Files:**
- Modify: `/home/gyq/.local/bin/codex-dp-exec`
- Test: shell commands with fake workdir and dry-run

- [ ] **Step 1: 记录当前脚本基线**

Run:

```bash
cp /home/gyq/.local/bin/codex-dp-exec /tmp/codex-dp-exec.before
bash -n /home/gyq/.local/bin/codex-dp-exec
```

Expected: `bash -n` exits 0.

- [ ] **Step 2: 新增参数变量**

在现有变量区追加：

```bash
task_id=""
run_root=""
run_id=""
controller_id="${CODEX_DP_EXEC_CONTROLLER_ID:-}"
```

- [ ] **Step 3: 扩展 usage 文档**

在 `Options:` 中加入：

```text
  --task-id <id>           Stable task identifier. Runs for the same task are grouped together.
  --run-root <dir>         Persistent run state root. Defaults to project *-superpowers-docs when discoverable.
  --run-id <id>            Explicit run id. Fails if the run directory already exists.
  --controller-id <id>     Identifier for the GPT controller session, such as codex1 or codex2.
```

在 `Environment:` 中加入：

```text
  CODEX_DP_EXEC_RUN_ROOT   Default persistent run state root.
  CODEX_DP_EXEC_CONTROLLER_ID
                           Default controller id written to status.json.
```

- [ ] **Step 4: 扩展 case 参数解析**

在 `while [[ $# -gt 0 ]]` 的 `case` 中加入：

```bash
    --task-id)
      [[ $# -ge 2 ]] || { echo "codex-dp-exec: --task-id requires a value" >&2; exit 2; }
      task_id="$2"
      shift 2
      ;;
    --run-root)
      [[ $# -ge 2 ]] || { echo "codex-dp-exec: --run-root requires a value" >&2; exit 2; }
      run_root="$2"
      shift 2
      ;;
    --run-id)
      [[ $# -ge 2 ]] || { echo "codex-dp-exec: --run-id requires a value" >&2; exit 2; }
      run_id="$2"
      shift 2
      ;;
    --controller-id)
      [[ $# -ge 2 ]] || { echo "codex-dp-exec: --controller-id requires a value" >&2; exit 2; }
      controller_id="$2"
      shift 2
      ;;
```

- [ ] **Step 5: 增加 ID 与路径辅助函数**

在参数解析后、校验前加入这些函数：

```bash
now_iso() {
  date '+%Y-%m-%dT%H:%M:%S%:z'
}

random_hex() {
  od -An -N4 -tx1 /dev/urandom | tr -d ' \n'
}

sanitize_id() {
  local value="$1"
  value="${value//[^A-Za-z0-9._-]/-}"
  value="${value##-}"
  value="${value%%-}"
  [[ -n "$value" ]] || value="task"
  printf '%s\n' "$value"
}

boot_id() {
  if [[ -r /proc/sys/kernel/random/boot_id ]]; then
    tr -d '\n' < /proc/sys/kernel/random/boot_id
  else
    hostname
  fi
}

discover_run_root() {
  local start="$1"
  local explicit="${CODEX_DP_EXEC_RUN_ROOT:-}"
  if [[ -n "$run_root" ]]; then
    printf '%s\n' "$run_root"
    return 0
  fi
  if [[ -n "$explicit" ]]; then
    printf '%s\n' "$explicit"
    return 0
  fi

  local current
  current="$(realpath "$start")"
  while [[ "$current" != "/" ]]; do
    local candidate
    candidate="$(find "$current" -maxdepth 1 -type d -name '*-superpowers-docs' 2>/dev/null | sort | head -n 1 || true)"
    if [[ -n "$candidate" ]]; then
      if [[ -d "$candidate/_global/superpowers" ]]; then
        printf '%s\n' "$candidate/_global/superpowers/runs/codex-dp-exec"
      else
        printf '%s\n' "$candidate/_global/runs/codex-dp-exec"
      fi
      return 0
    fi
    current="$(dirname "$current")"
  done

  printf '%s\n' "${HOME}/.local/state/codex-dp-exec/runs"
}
```

- [ ] **Step 6: 初始化 task_id、run_id、run_root**

在参数与文件校验通过后、构造 `cmd` 前加入：

```bash
real_workdir="$(realpath "$workdir")"
task_id="$(sanitize_id "${task_id:-task-$(date '+%Y%m%d-%H%M%S')-$(random_hex)}")"
run_id="$(sanitize_id "${run_id:-$(date '+%Y%m%d-%H%M%S')-$$-$(random_hex)}")"
controller_id="$(sanitize_id "${controller_id:-unknown-controller}")"
run_root="$(discover_run_root "$real_workdir")"
task_dir="${run_root}/${task_id}"
run_dir="${task_dir}/runs/${run_id}"
locks_dir="${run_root}/locks"
```

- [ ] **Step 7: 验证 dry-run 不破坏旧行为**

Run:

```bash
tmp="$(mktemp -d /tmp/codex-dp-exec-test-XXXXXX)"
printf 'hello\n' > "$tmp/prompt.md"
CODEX_DP_EXEC_RUN_ROOT="$tmp/runs" codex-dp-exec \
  --cd /mnt/f/IdeaProjects/goto-software \
  --prompt-file "$tmp/prompt.md" \
  --task-id task-demo \
  --controller-id codex1 \
  --dry-run
```

Expected: 输出仍是 `CODEX_HOME=... codex exec ... < prompt.md` 格式；退出码为 0。

- [ ] **Step 8: 记录变更状态**

```bash
if git -C /mnt/f/IdeaProjects/goto-software ls-files --error-unmatch -- /home/gyq/.local/bin/codex-dp-exec >/dev/null 2>&1; then
  echo "script_tracked=yes"
else
  echo "script_tracked=no"
fi
```

Expected: 通常输出 `script_tracked=no`，表示脚本位于当前仓库外。脚本位于仓库外时，不执行 `git commit`；在任务报告中记录“`/home/gyq/.local/bin/codex-dp-exec` 是用户级脚本，未纳入当前仓库提交”。

## Task 2: 状态目录与 JSON 文件写入

**Files:**
- Modify: `/home/gyq/.local/bin/codex-dp-exec`
- Test: fake `CODEX_BIN` success run

- [ ] **Step 1: 写入 JSON 辅助函数**

在辅助函数区加入：

```bash
json_write() {
  local target="$1"
  shift
  local tmp="${target}.tmp.$$"
  python3 - "$target" "$@" > "$tmp" <<'PY'
import json
import sys

pairs = sys.argv[2:]
data = {}
for pair in pairs:
    key, value = pair.split("=", 1)
    if value == "__NULL__":
        data[key] = None
    elif value.startswith("__INT__:"):
        data[key] = int(value[len("__INT__:"):])
    else:
        data[key] = value
json.dump(data, sys.stdout, ensure_ascii=False, indent=2)
sys.stdout.write("\n")
PY
  mv "$tmp" "$target"
}

write_status() {
  local status="$1"
  local ended_at="${2:-__NULL__}"
  local exit_code="${3:-__NULL__}"
  json_write "$run_dir/status.json" \
    "task_id=$task_id" \
    "run_id=$run_id" \
    "status=$status" \
    "controller_id=$controller_id" \
    "workdir=$real_workdir" \
    "branch=$branch_name" \
    "base_sha=$base_sha" \
    "pid=__INT__:$$" \
    "started_at=$started_at" \
    "ended_at=$ended_at" \
    "exit_code=$exit_code"
}

write_heartbeat() {
  json_write "$run_dir/heartbeat.json" \
    "run_id=$run_id" \
    "pid=__INT__:$$" \
    "boot_id=$current_boot_id" \
    "hostname=$(hostname)" \
    "workdir=$real_workdir" \
    "updated_at=$(now_iso)"
}
```

- [ ] **Step 2: 创建目录并记录 Git 元数据**

在构造 `cmd` 前加入：

```bash
mkdir -p "$task_dir" "$locks_dir"
if ! mkdir "$run_dir" 2>/dev/null; then
  echo "codex-dp-exec: run directory already exists: $run_dir" >&2
  exit 2
fi

started_at="$(now_iso)"
current_boot_id="$(boot_id)"
branch_name="$(git -C "$real_workdir" rev-parse --abbrev-ref HEAD 2>/dev/null || printf 'unknown')"
base_sha="$(git -C "$real_workdir" rev-parse HEAD 2>/dev/null || printf 'unknown')"

cp "$prompt_file" "$run_dir/prompt.md"
json_write "$task_dir/task.json" \
  "task_id=$task_id" \
  "workdir=$real_workdir" \
  "controller_id=$controller_id" \
  "updated_at=$started_at"
write_status "created"
write_heartbeat
```

- [ ] **Step 3: 记录 command.txt**

构造完 `cmd` 后加入：

```bash
{
  printf 'CODEX_HOME=%q' "$CODEX_DP_HOME"
  printf ' %q' "${cmd[@]}"
  printf ' < %q\n' "$run_dir/prompt.md"
} > "$run_dir/command.txt"
```

- [ ] **Step 4: dry-run 输出 run_dir**

把 dry-run 分支改为：

```bash
if [[ "$dry_run" -eq 1 ]]; then
  cat "$run_dir/command.txt"
  printf 'RUN_DIR=%s\n' "$run_dir"
  write_status "succeeded" "$(now_iso)" "__INT__:0"
  exit 0
fi
```

- [ ] **Step 5: 写 fake codex 并验证状态文件**

Run:

```bash
tmp="$(mktemp -d /tmp/codex-dp-exec-test-XXXXXX)"
cat > "$tmp/fake-codex" <<'SH'
#!/usr/bin/env bash
out=""
while [[ $# -gt 0 ]]; do
  case "$1" in
    --output-last-message) out="$2"; shift 2 ;;
    *) shift ;;
  esac
done
cat >/dev/null
echo "fake stdout"
echo "fake stderr" >&2
[[ -n "$out" ]] && printf 'DONE\n' > "$out"
exit 0
SH
chmod +x "$tmp/fake-codex"
printf 'fake prompt\n' > "$tmp/prompt.md"
CODEX_BIN="$tmp/fake-codex" CODEX_DP_EXEC_RUN_ROOT="$tmp/runs" codex-dp-exec \
  --cd /mnt/f/IdeaProjects/goto-software \
  --prompt-file "$tmp/prompt.md" \
  --task-id task-json \
  --controller-id codex1 \
  --dry-run
find "$tmp/runs" -type f | sort
python3 -m json.tool "$tmp"/runs/task-json/runs/*/status.json >/dev/null
python3 -m json.tool "$tmp"/runs/task-json/runs/*/heartbeat.json >/dev/null
```

Expected:
- `status.json` 和 `heartbeat.json` 是合法 JSON。
- `prompt.md`、`command.txt` 已创建。

- [ ] **Step 6: 记录变更状态**

```bash
if git -C /mnt/f/IdeaProjects/goto-software ls-files --error-unmatch -- /home/gyq/.local/bin/codex-dp-exec >/dev/null 2>&1; then
  echo "script_tracked=yes"
else
  echo "script_tracked=no"
fi
```

Expected: 通常输出 `script_tracked=no`。脚本位于仓库外时，不执行提交，只在任务报告中记录。

## Task 3: worktree 锁与 stale lock 清理

**Files:**
- Modify: `/home/gyq/.local/bin/codex-dp-exec`
- Test: fake lock scenarios

- [ ] **Step 1: 增加锁辅助函数**

在辅助函数区加入：

```bash
hash_text() {
  printf '%s' "$1" | sha256sum | awk '{print $1}'
}

pid_alive() {
  local pid="$1"
  [[ "$pid" =~ ^[0-9]+$ ]] && kill -0 "$pid" 2>/dev/null
}

json_get() {
  local file="$1"
  local key="$2"
  python3 - "$file" "$key" <<'PY'
import json
import sys

with open(sys.argv[1], "r", encoding="utf-8") as fh:
    data = json.load(fh)
value = data.get(sys.argv[2], "")
print("" if value is None else value)
PY
}

lock_path_for_workdir() {
  printf '%s/%s.lock' "$locks_dir" "$(hash_text "$real_workdir")"
}
```

- [ ] **Step 2: 增加 acquire_lock 与 release_lock**

加入：

```bash
acquire_lock() {
  lock_dir="$(lock_path_for_workdir)"
  if mkdir "$lock_dir" 2>/dev/null; then
    json_write "$lock_dir/lock.json" \
      "run_id=$run_id" \
      "task_id=$task_id" \
      "workdir=$real_workdir" \
      "pid=__INT__:$$" \
      "boot_id=$current_boot_id" \
      "created_at=$(now_iso)"
    return 0
  fi

  local lock_json="$lock_dir/lock.json"
  if [[ -f "$lock_json" ]]; then
    local lock_pid lock_boot
    lock_pid="$(json_get "$lock_json" pid || true)"
    lock_boot="$(json_get "$lock_json" boot_id || true)"
    if [[ "$lock_boot" != "$current_boot_id" ]] || ! pid_alive "$lock_pid"; then
      rm -rf "$lock_dir"
      if mkdir "$lock_dir" 2>/dev/null; then
        json_write "$lock_dir/lock.json" \
          "run_id=$run_id" \
          "task_id=$task_id" \
          "workdir=$real_workdir" \
          "pid=__INT__:$$" \
          "boot_id=$current_boot_id" \
          "created_at=$(now_iso)"
        return 0
      fi
    fi
  fi

  echo "codex-dp-exec: workdir is already locked: $real_workdir" >&2
  echo "codex-dp-exec: use a separate git worktree for parallel GPT controllers" >&2
  exit 3
}

release_lock() {
  if [[ -n "${lock_dir:-}" && -f "$lock_dir/lock.json" ]]; then
    local owner
    owner="$(json_get "$lock_dir/lock.json" run_id || true)"
    if [[ "$owner" == "$run_id" ]]; then
      rm -rf "$lock_dir"
    fi
  fi
}
```

- [ ] **Step 3: 在执行前后接入锁**

在 `dry-run` 分支之后、真实执行之前调用：

```bash
acquire_lock
trap release_lock EXIT
```

- [ ] **Step 4: 测试同 worktree 活锁拒绝**

Run:

```bash
tmp="$(mktemp -d /tmp/codex-dp-exec-test-XXXXXX)"
printf 'fake prompt\n' > "$tmp/prompt.md"
mkdir -p "$tmp/runs/locks"
real_workdir="$(realpath /mnt/f/IdeaProjects/goto-software)"
hash="$(printf '%s' "$real_workdir" | sha256sum | awk '{print $1}')"
mkdir "$tmp/runs/locks/$hash.lock"
python3 - "$tmp/runs/locks/$hash.lock/lock.json" "$$" "$(cat /proc/sys/kernel/random/boot_id)" "$real_workdir" <<'PY'
import json, sys
with open(sys.argv[1], "w", encoding="utf-8") as fh:
    json.dump({
        "run_id": "existing",
        "task_id": "task-lock",
        "workdir": sys.argv[4],
        "pid": int(sys.argv[2]),
        "boot_id": sys.argv[3],
        "created_at": "2026-06-24T00:00:00+08:00"
    }, fh)
PY
set +e
CODEX_DP_EXEC_RUN_ROOT="$tmp/runs" codex-dp-exec \
  --cd /mnt/f/IdeaProjects/goto-software \
  --prompt-file "$tmp/prompt.md" \
  --task-id task-lock \
  --controller-id codex1 \
  --dry-run
code=$?
set -e
printf 'exit=%s\n' "$code"
```

Expected: 由于 dry-run 不获取锁，此命令仍成功。随后用 fake `CODEX_BIN` 执行真实路径：

```bash
cat > "$tmp/fake-codex" <<'SH'
#!/usr/bin/env bash
cat >/dev/null
exit 0
SH
chmod +x "$tmp/fake-codex"
set +e
CODEX_BIN="$tmp/fake-codex" CODEX_DP_EXEC_RUN_ROOT="$tmp/runs" codex-dp-exec \
  --cd /mnt/f/IdeaProjects/goto-software \
  --prompt-file "$tmp/prompt.md" \
  --task-id task-lock \
  --controller-id codex1
code=$?
set -e
printf 'exit=%s\n' "$code"
```

Expected: exit code `3`，stderr 包含 `workdir is already locked`。

- [ ] **Step 5: 测试 stale lock 清理**

Run:

```bash
tmp="$(mktemp -d /tmp/codex-dp-exec-test-XXXXXX)"
printf 'fake prompt\n' > "$tmp/prompt.md"
mkdir -p "$tmp/runs/locks"
real_workdir="$(realpath /mnt/f/IdeaProjects/goto-software)"
hash="$(printf '%s' "$real_workdir" | sha256sum | awk '{print $1}')"
mkdir "$tmp/runs/locks/$hash.lock"
python3 - "$tmp/runs/locks/$hash.lock/lock.json" "$real_workdir" <<'PY'
import json, sys
with open(sys.argv[1], "w", encoding="utf-8") as fh:
    json.dump({
        "run_id": "stale",
        "task_id": "task-lock",
        "workdir": sys.argv[2],
        "pid": 99999999,
        "boot_id": "old-boot",
        "created_at": "2026-06-24T00:00:00+08:00"
    }, fh)
PY
cat > "$tmp/fake-codex" <<'SH'
#!/usr/bin/env bash
out=""
while [[ $# -gt 0 ]]; do
  case "$1" in
    --output-last-message) out="$2"; shift 2 ;;
    *) shift ;;
  esac
done
cat >/dev/null
[[ -n "$out" ]] && printf 'DONE\n' > "$out"
exit 0
SH
chmod +x "$tmp/fake-codex"
CODEX_BIN="$tmp/fake-codex" CODEX_DP_EXEC_RUN_ROOT="$tmp/runs" codex-dp-exec \
  --cd /mnt/f/IdeaProjects/goto-software \
  --prompt-file "$tmp/prompt.md" \
  --task-id task-lock \
  --controller-id codex1
find "$tmp/runs/locks" -maxdepth 2 -type f -print
```

Expected: 命令成功，旧 stale lock 被替换，执行结束后锁目录被清理。

- [ ] **Step 6: 记录变更状态**

```bash
if git -C /mnt/f/IdeaProjects/goto-software ls-files --error-unmatch -- /home/gyq/.local/bin/codex-dp-exec >/dev/null 2>&1; then
  echo "script_tracked=yes"
else
  echo "script_tracked=no"
fi
```

Expected: 通常输出 `script_tracked=no`。脚本位于仓库外时，不执行提交，只在任务报告中记录。

## Task 4: 受控执行、日志、报告与心跳

**Files:**
- Modify: `/home/gyq/.local/bin/codex-dp-exec`
- Test: fake success and fake failure

- [ ] **Step 1: 统一 report 输出路径**

在构造 `cmd` 前加入：

```bash
requested_output_last_message="$output_last_message"
canonical_report="$run_dir/report.md"
output_last_message="$canonical_report"
stdout_log="$run_dir/stdout.log"
stderr_log="$run_dir/stderr.log"
exit_code_file="$run_dir/exit_code"
```

保持后续 `cmd+=("--output-last-message" "$output_last_message")` 逻辑不变。若用户传入了 `--output-last-message`，执行结束后复制 canonical report 到用户指定位置。

- [ ] **Step 2: 增加受控执行函数**

把最后一行真实执行：

```bash
CODEX_HOME="$CODEX_DP_HOME" "${cmd[@]}" < "$prompt_file"
```

替换为：

```bash
run_codex() {
  write_status "running"
  write_heartbeat

  CODEX_HOME="$CODEX_DP_HOME" "${cmd[@]}" < "$run_dir/prompt.md" \
    > >(tee -a "$stdout_log") \
    2> >(tee -a "$stderr_log" >&2) &
  child_pid=$!

  while kill -0 "$child_pid" 2>/dev/null; do
    write_heartbeat
    sleep 5
  done

  set +e
  wait "$child_pid"
  local code=$?
  set -e

  printf '%s\n' "$code" > "$exit_code_file"
  if [[ -n "$requested_output_last_message" && -f "$canonical_report" ]]; then
    cp "$canonical_report" "$requested_output_last_message"
  fi

  if [[ "$code" -eq 0 ]]; then
    write_status "succeeded" "$(now_iso)" "__INT__:0"
  else
    write_status "failed" "$(now_iso)" "__INT__:$code"
  fi
  return "$code"
}

interrupt_run() {
  local code="$1"
  if [[ -n "${child_pid:-}" ]]; then
    kill "$child_pid" 2>/dev/null || true
  fi
  printf '%s\n' "$code" > "$exit_code_file" 2>/dev/null || true
  write_status "interrupted" "$(now_iso)" "__INT__:$code" 2>/dev/null || true
  exit "$code"
}

trap 'interrupt_run 130' INT
trap 'interrupt_run 143' TERM

run_codex
```

- [ ] **Step 3: 验证成功路径**

Run:

```bash
tmp="$(mktemp -d /tmp/codex-dp-exec-test-XXXXXX)"
cat > "$tmp/fake-codex" <<'SH'
#!/usr/bin/env bash
out=""
while [[ $# -gt 0 ]]; do
  case "$1" in
    --output-last-message) out="$2"; shift 2 ;;
    *) shift ;;
  esac
done
cat >/dev/null
echo "fake stdout"
echo "fake stderr" >&2
[[ -n "$out" ]] && printf 'DONE\n' > "$out"
exit 0
SH
chmod +x "$tmp/fake-codex"
printf 'fake prompt\n' > "$tmp/prompt.md"
CODEX_BIN="$tmp/fake-codex" CODEX_DP_EXEC_RUN_ROOT="$tmp/runs" codex-dp-exec \
  --cd /mnt/f/IdeaProjects/goto-software \
  --prompt-file "$tmp/prompt.md" \
  --task-id task-success \
  --controller-id codex1 \
  --output-last-message "$tmp/final-report.md"
python3 -m json.tool "$tmp"/runs/task-success/runs/*/status.json
cat "$tmp/final-report.md"
cat "$tmp"/runs/task-success/runs/*/exit_code
```

Expected:
- `status` is `succeeded`
- `exit_code` is `0`
- `$tmp/final-report.md` contains `DONE`
- `stdout.log` contains `fake stdout`
- `stderr.log` contains `fake stderr`

- [ ] **Step 4: 验证失败路径**

Run:

```bash
tmp="$(mktemp -d /tmp/codex-dp-exec-test-XXXXXX)"
cat > "$tmp/fake-codex" <<'SH'
#!/usr/bin/env bash
cat >/dev/null
echo "fake failure" >&2
exit 7
SH
chmod +x "$tmp/fake-codex"
printf 'fake prompt\n' > "$tmp/prompt.md"
set +e
CODEX_BIN="$tmp/fake-codex" CODEX_DP_EXEC_RUN_ROOT="$tmp/runs" codex-dp-exec \
  --cd /mnt/f/IdeaProjects/goto-software \
  --prompt-file "$tmp/prompt.md" \
  --task-id task-failure \
  --controller-id codex1
code=$?
set -e
printf 'exit=%s\n' "$code"
python3 -m json.tool "$tmp"/runs/task-failure/runs/*/status.json
cat "$tmp"/runs/task-failure/runs/*/exit_code
```

Expected:
- wrapper exit code is `7`
- `status` is `failed`
- `exit_code` file contains `7`

- [ ] **Step 5: 记录变更状态**

```bash
if git -C /mnt/f/IdeaProjects/goto-software ls-files --error-unmatch -- /home/gyq/.local/bin/codex-dp-exec >/dev/null 2>&1; then
  echo "script_tracked=yes"
else
  echo "script_tracked=no"
fi
```

Expected: 通常输出 `script_tracked=no`。脚本位于仓库外时，不执行提交，只在任务报告中记录。

## Task 5: interrupted 恢复扫描与 skill 文档更新

**Files:**
- Modify: `/home/gyq/.local/bin/codex-dp-exec`
- Modify: `/home/gyq/.agents/skills/hybrid-subagent-driven-development/SKILL.md`
- Test: stale running run, skill validation, real smoke test

- [ ] **Step 1: 增加 stale run 标记函数**

在辅助函数区加入：

```bash
mark_interrupted_status() {
  local status_file="$1"
  local run_path
  run_path="$(dirname "$status_file")"
  local old_run_id
  old_run_id="$(json_get "$status_file" run_id || basename "$run_path")"
  local old_pid
  old_pid="$(json_get "$status_file" pid || true)"
  local old_status
  old_status="$(json_get "$status_file" status || true)"
  [[ "$old_status" == "running" ]] || return 0

  local hb="$run_path/heartbeat.json"
  local old_boot=""
  if [[ -f "$hb" ]]; then
    old_boot="$(json_get "$hb" boot_id || true)"
  fi

  if [[ "$old_boot" != "$current_boot_id" ]] || ! pid_alive "$old_pid"; then
    local tmp="${status_file}.tmp.$$"
    python3 - "$status_file" "$tmp" "$(now_iso)" <<'PY'
import json
import sys

source, target, ended_at = sys.argv[1:4]
with open(source, "r", encoding="utf-8") as fh:
    data = json.load(fh)
data["status"] = "interrupted"
data["ended_at"] = ended_at
data["exit_code"] = 130
with open(target, "w", encoding="utf-8") as fh:
    json.dump(data, fh, ensure_ascii=False, indent=2)
    fh.write("\n")
PY
    mv "$tmp" "$status_file"
  fi
}

mark_stale_runs_for_task() {
  [[ -d "$task_dir/runs" ]] || return 0
  local status_file
  while IFS= read -r status_file; do
    mark_interrupted_status "$status_file"
  done < <(find "$task_dir/runs" -mindepth 2 -maxdepth 2 -name status.json -type f)
}
```

- [ ] **Step 2: 在创建新 run 前扫描同 task 的旧 running run**

把目录创建顺序调整为：

```bash
mkdir -p "$task_dir" "$locks_dir"
current_boot_id="$(boot_id)"
mark_stale_runs_for_task
if ! mkdir "$run_dir" 2>/dev/null; then
  echo "codex-dp-exec: run directory already exists: $run_dir" >&2
  exit 2
fi
```

确保 `current_boot_id` 在扫描前已赋值，且 `task_dir` 在扫描前已定义。

- [ ] **Step 3: 更新 hybrid skill 调用示例**

在 `/home/gyq/.agents/skills/hybrid-subagent-driven-development/SKILL.md` 的命令示例中，把：

```bash
codex-dp-exec --cd <repo> --prompt-file <prompt-file> --output-last-message <report-file>
```

替换为：

```bash
codex-dp-exec \
  --cd <worktree> \
  --prompt-file <prompt-file> \
  --task-id <plan-task-id> \
  --controller-id <codex1-or-codex2> \
  --output-last-message <report-file>
```

在 Process 后补充：

```markdown
When `codex-dp-exec` prints or records a run directory, inspect that run directory first:

- `status.json` for run status.
- `heartbeat.json` for long-running activity.
- `stdout.log` and `stderr.log` for execution details.
- `report.md` for the DeepSeek final report.
- `exit_code` for process result.

If a prior run for the same task is `interrupted`, inspect `git diff` before retrying. Recovery creates a new run for the same task; do not overwrite the interrupted run.
```

- [ ] **Step 4: 测试 stale running run 被标记 interrupted**

Run:

```bash
tmp="$(mktemp -d /tmp/codex-dp-exec-test-XXXXXX)"
old="$tmp/runs/task-recover/runs/old-run"
mkdir -p "$old"
python3 - "$old/status.json" <<'PY'
import json, sys
with open(sys.argv[1], "w", encoding="utf-8") as fh:
    json.dump({
        "task_id": "task-recover",
        "run_id": "old-run",
        "status": "running",
        "controller_id": "codex1",
        "workdir": "/mnt/f/IdeaProjects/goto-software",
        "branch": "main",
        "base_sha": "abc123",
        "pid": 99999999,
        "started_at": "2026-06-24T00:00:00+08:00",
        "ended_at": None,
        "exit_code": None
    }, fh)
PY
python3 - "$old/heartbeat.json" <<'PY'
import json, sys
with open(sys.argv[1], "w", encoding="utf-8") as fh:
    json.dump({
        "run_id": "old-run",
        "pid": 99999999,
        "boot_id": "old-boot",
        "hostname": "old-host",
        "workdir": "/mnt/f/IdeaProjects/goto-software",
        "updated_at": "2026-06-24T00:00:00+08:00"
    }, fh)
PY
cat > "$tmp/fake-codex" <<'SH'
#!/usr/bin/env bash
out=""
while [[ $# -gt 0 ]]; do
  case "$1" in
    --output-last-message) out="$2"; shift 2 ;;
    *) shift ;;
  esac
done
cat >/dev/null
[[ -n "$out" ]] && printf 'DONE\n' > "$out"
exit 0
SH
chmod +x "$tmp/fake-codex"
printf 'fake prompt\n' > "$tmp/prompt.md"
CODEX_BIN="$tmp/fake-codex" CODEX_DP_EXEC_RUN_ROOT="$tmp/runs" codex-dp-exec \
  --cd /mnt/f/IdeaProjects/goto-software \
  --prompt-file "$tmp/prompt.md" \
  --task-id task-recover \
  --controller-id codex1
python3 - "$old/status.json" <<'PY'
import json, sys
with open(sys.argv[1], "r", encoding="utf-8") as fh:
    data = json.load(fh)
print(data["status"])
print(data["exit_code"])
PY
```

Expected:

```text
interrupted
130
```

- [ ] **Step 5: 验证 skill 文件**

Run:

```bash
python3 /home/gyq/.codex_official/skills/.system/skill-creator/scripts/quick_validate.py \
  /home/gyq/.agents/skills/hybrid-subagent-driven-development
```

Expected: `Skill is valid!`

- [ ] **Step 6: 真实 smoke test**

Run:

```bash
tmp="$(mktemp -d /tmp/codex-dp-exec-real-XXXXXX)"
printf '请只回复一行：codex-dp-exec-state-ok。不要读写文件，不要运行命令。\n' > "$tmp/prompt.md"
codex-dp-exec \
  --cd /mnt/f/IdeaProjects/goto-software \
  --prompt-file "$tmp/prompt.md" \
  --task-id smoke-real \
  --controller-id codex1 \
  --run-root "$tmp/runs" \
  --output-last-message "$tmp/report.md"
cat "$tmp/report.md"
find "$tmp/runs" -type f | sort
```

Expected:
- report contains `codex-dp-exec-state-ok`
- run directory contains `status.json`、`heartbeat.json`、`stdout.log`、`stderr.log`、`report.md`、`exit_code`、`command.txt`
- final `status.json` has `status = succeeded`

- [ ] **Step 7: 记录最终变更状态**

```bash
if git -C /mnt/f/IdeaProjects/goto-software ls-files --error-unmatch -- /home/gyq/.local/bin/codex-dp-exec >/dev/null 2>&1; then
  echo "script_tracked=yes"
else
  echo "script_tracked=no"
fi
if git -C /mnt/f/IdeaProjects/goto-software ls-files --error-unmatch -- /home/gyq/.agents/skills/hybrid-subagent-driven-development/SKILL.md >/dev/null 2>&1; then
  echo "skill_tracked=yes"
else
  echo "skill_tracked=no"
fi
```

Expected: 通常两个值都是 `no`。这些文件是用户级脚本和用户级 skill 时，不执行提交，只在最终报告中记录实际修改的绝对路径。

## 最终验收

- [ ] `bash -n /home/gyq/.local/bin/codex-dp-exec` 通过。
- [ ] `codex-dp-exec --help` 展示 `--task-id`、`--run-root`、`--run-id`、`--controller-id`。
- [ ] fake success 测试生成完整 run 目录和 `succeeded` 状态。
- [ ] fake failure 测试生成 `failed` 状态和非零 `exit_code`。
- [ ] 同 worktree 活锁测试返回 exit code `3`。
- [ ] stale lock 测试能清理旧锁并继续执行。
- [ ] stale running run 测试能把旧 run 标记为 `interrupted`。
- [ ] 真实 DeepSeek smoke test 返回 `codex-dp-exec-state-ok`。
- [ ] `/home/gyq/.agents/skills/hybrid-subagent-driven-development` 通过 quick_validate。
- [ ] 未修改现有 `codex-dp` alias/function。
- [ ] 未修改 `/mnt/g/Obsidian/skill/superpowers/skills/subagent-driven-development`。
