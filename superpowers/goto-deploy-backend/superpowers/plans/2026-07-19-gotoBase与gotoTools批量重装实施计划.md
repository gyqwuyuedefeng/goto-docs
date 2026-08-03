# gotoBase 与 gotoTools 批量重装实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**目标：** 从 WSL 宿主机按依赖顺序移除并重新安装 `gotoBase`、`gotoTools`，随后在新 R 进程中验证两个包。

**架构：** 四个 Rscript 文件分别完成单包移除和安装，一个 Bash 入口负责预检、容器选择、顺序编排和最终验证。Bash 入口通过 `docker exec -i` 和标准输入传送 R 代码，不增加挂载、不复制临时文件、不重启容器。

**技术栈：** Bash、Docker CLI、Rscript、R `remotes` 包。

## 全局约束

- 实现仓库：`/mnt/f/IdeaProjects/goto-software/goto-deploy-backend`。
- 默认容器名：`goto_rbase_wsl`；只允许第一个位置参数覆盖。
- 源码路径：`/root/software/gotoBase`、`/root/software/gotoTools`。
- 固定顺序：移除 `gotoTools` → 移除 `gotoBase` → 安装 `gotoBase` → 安装 `gotoTools` → 新 R 进程验证。
- 所有预检发生在首次移除前；包未安装视为成功，其他失败立即停止并返回非零退出码。
- 安装参数固定为 `force = TRUE`、`upgrade = "never"`、`dependencies = FALSE`。
- 不调用 `roxygen2::roxygenize()`，不修改两个 R 包业务代码，不升级依赖，不改挂载，不回滚，不重启容器。
- 说明和输出使用中文；Git 提交信息使用中文；只提交本计划列出的文件。

---

## 文件映射

| 文件 | 操作 | 职责 |
| --- | --- | --- |
| `.bash/install_gotoTools.sh` | 修改 | 安装 `gotoTools` |
| `.bash/remove_gotoTools.sh` | 修改 | 幂等移除 `gotoTools` |
| `.bash/install_gotoBase.sh` | 新增 | 安装 `gotoBase` |
| `.bash/remove_gotoBase.sh` | 新增 | 幂等移除 `gotoBase` |
| `.bash/reinstall_gotoBase_gotoTools.sh` | 新增 | 预检、编排、验证 |
| `tests/bash/test_r_package_scripts.sh` | 新增 | 检查四个 Rscript 的静态约束 |
| `tests/bash/test_reinstall_gotoBase_gotoTools.sh` | 新增 | 用伪 Docker 检查批量入口行为 |

## Task 1：实现四个单包 Rscript

**Files:**

- Create: `.bash/install_gotoBase.sh`
- Create: `.bash/remove_gotoBase.sh`
- Modify: `.bash/install_gotoTools.sh:1-15`
- Modify: `.bash/remove_gotoTools.sh:1-8`
- Create: `tests/bash/test_r_package_scripts.sh`

**Interfaces:**

- Consumes: `.libPaths()`、`remotes::install_local()`、固定源码路径和 `DESCRIPTION`。
- Produces: 四个能由 `Rscript --vanilla /dev/stdin` 执行的文件；成功返回 `0`，R 错误返回非零。

- [ ] **Step 1：写静态约束测试**

创建 `tests/bash/test_r_package_scripts.sh`：

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
readonly TEST_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd -P)"
readonly BASH_DIR="$(cd -- "$TEST_DIR/../../.bash" && pwd -P)"
fail() { printf '测试失败：%s\n' "$1" >&2; exit 1; }
has() { grep -Fq -- "$2" "$1" || fail "$1 缺少：$2"; }
lacks() { ! grep -Fq -- "$2" "$1" || fail "$1 不应包含：$2"; }

check_install() {
    local file="$1" package="$2" path="$3"
    [[ -f "$file" ]] || fail "文件不存在：$file"
    [[ "$(sed -n '1p' "$file")" == '#!/usr/bin/env Rscript' ]] || fail "$file shebang 错误"
    has "$file" "package_name <- \"$package\""
    has "$file" "package_path <- \"$path\""
    has "$file" 'remotes::install_local('
    has "$file" 'force = TRUE'
    has "$file" 'upgrade = "never"'
    has "$file" 'dependencies = FALSE'
    lacks "$file" 'roxygen2::roxygenize'
    lacks "$file" 'remove.packages'
}

check_remove() {
    local file="$1" package="$2"
    [[ -f "$file" ]] || fail "文件不存在：$file"
    [[ "$(sed -n '1p' "$file")" == '#!/usr/bin/env Rscript' ]] || fail "$file shebang 错误"
    has "$file" "package_name <- \"$package\""
    has "$file" '.libPaths()'
    has "$file" 'remove.packages('
    has "$file" 'remaining_libraries'
}

check_install "$BASH_DIR/install_gotoBase.sh" gotoBase /root/software/gotoBase
check_install "$BASH_DIR/install_gotoTools.sh" gotoTools /root/software/gotoTools
check_remove "$BASH_DIR/remove_gotoBase.sh" gotoBase
check_remove "$BASH_DIR/remove_gotoTools.sh" gotoTools
printf '通过：四个单包 Rscript 满足静态约束。\n'
```

- [ ] **Step 2：运行测试确认失败**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-deploy-backend
chmod 0755 tests/bash/test_r_package_scripts.sh
tests/bash/test_r_package_scripts.sh
```

Expected: FAIL，报告 `.bash/install_gotoBase.sh` 不存在。

- [ ] **Step 3：实现两个安装脚本**

`.bash/install_gotoBase.sh`：

```r
#!/usr/bin/env Rscript

# 只安装固定挂载路径中的 gotoBase，不生成文档、不处理依赖。
package_name <- "gotoBase"
package_path <- "/root/software/gotoBase"
description_path <- file.path(package_path, "DESCRIPTION")
if (!file.exists(description_path)) {
    stop(sprintf("未找到 %s 源码描述文件：%s", package_name, description_path))
}
actual_package_name <- unname(read.dcf(description_path)[1, "Package"])
if (!identical(actual_package_name, package_name)) {
    stop(sprintf("源码包名不匹配：期望 %s，实际 %s", package_name, actual_package_name))
}
if (!requireNamespace("remotes", quietly = TRUE)) {
    stop("容器内未安装 remotes，无法安装本地 R 包")
}
message(sprintf("开始从 %s 安装 %s", package_path, package_name))
remotes::install_local(
    path = package_path,
    force = TRUE,
    upgrade = "never",
    dependencies = FALSE
)
message(sprintf("%s 安装成功，版本：%s", package_name,
                as.character(utils::packageVersion(package_name))))
```

`.bash/install_gotoTools.sh`：

```r
#!/usr/bin/env Rscript

# 只安装固定挂载路径中的 gotoTools；批量入口必须先安装 gotoBase。
package_name <- "gotoTools"
package_path <- "/root/software/gotoTools"
description_path <- file.path(package_path, "DESCRIPTION")
if (!file.exists(description_path)) {
    stop(sprintf("未找到 %s 源码描述文件：%s", package_name, description_path))
}
actual_package_name <- unname(read.dcf(description_path)[1, "Package"])
if (!identical(actual_package_name, package_name)) {
    stop(sprintf("源码包名不匹配：期望 %s，实际 %s", package_name, actual_package_name))
}
if (!requireNamespace("remotes", quietly = TRUE)) {
    stop("容器内未安装 remotes，无法安装本地 R 包")
}
message(sprintf("开始从 %s 安装 %s", package_path, package_name))
remotes::install_local(
    path = package_path,
    force = TRUE,
    upgrade = "never",
    dependencies = FALSE
)
message(sprintf("%s 安装成功，版本：%s", package_name,
                as.character(utils::packageVersion(package_name))))
```

- [ ] **Step 4：实现两个幂等移除脚本**

`.bash/remove_gotoBase.sh`：

```r
#!/usr/bin/env Rscript

# 移除所有 R 库目录中的 gotoBase；原本不存在时正常返回。
package_name <- "gotoBase"
find_package_libraries <- function() {
    .libPaths()[vapply(.libPaths(), function(library_path) {
        dir.exists(file.path(library_path, package_name))
    }, logical(1))]
}
installed_libraries <- find_package_libraries()
if (length(installed_libraries) == 0) {
    message(sprintf("%s 未安装，跳过移除", package_name))
    quit(save = "no", status = 0)
}
for (library_path in installed_libraries) {
    message(sprintf("正在从 %s 移除 %s", library_path, package_name))
    utils::remove.packages(package_name, lib = library_path)
}
remaining_libraries <- find_package_libraries()
if (length(remaining_libraries) > 0) {
    stop(sprintf("%s 移除不完整，仍存在于：%s", package_name,
                 paste(remaining_libraries, collapse = ", ")))
}
message(sprintf("%s 已完全移除", package_name))
```

`.bash/remove_gotoTools.sh`：

```r
#!/usr/bin/env Rscript

# 移除所有 R 库目录中的 gotoTools；原本不存在时正常返回。
package_name <- "gotoTools"
find_package_libraries <- function() {
    .libPaths()[vapply(.libPaths(), function(library_path) {
        dir.exists(file.path(library_path, package_name))
    }, logical(1))]
}
installed_libraries <- find_package_libraries()
if (length(installed_libraries) == 0) {
    message(sprintf("%s 未安装，跳过移除", package_name))
    quit(save = "no", status = 0)
}
for (library_path in installed_libraries) {
    message(sprintf("正在从 %s 移除 %s", library_path, package_name))
    utils::remove.packages(package_name, lib = library_path)
}
remaining_libraries <- find_package_libraries()
if (length(remaining_libraries) > 0) {
    stop(sprintf("%s 移除不完整，仍存在于：%s", package_name,
                 paste(remaining_libraries, collapse = ", ")))
}
message(sprintf("%s 已完全移除", package_name))
```

- [ ] **Step 5：设置权限、运行测试并提交**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-deploy-backend
chmod 0755 .bash/{install,remove}_goto{Base,Tools}.sh tests/bash/test_r_package_scripts.sh
tests/bash/test_r_package_scripts.sh
git diff --check
git add .bash/install_gotoBase.sh .bash/remove_gotoBase.sh \
  .bash/install_gotoTools.sh .bash/remove_gotoTools.sh \
  tests/bash/test_r_package_scripts.sh
git commit -m "feat: 增加R包独立安装与移除脚本"
```

Expected: 测试输出 `通过：四个单包 Rscript 满足静态约束。`；`git diff --check` 无输出；提交仅含上述五个文件。

## Task 2：实现 WSL 批量入口

**Files:**

- Create: `.bash/reinstall_gotoBase_gotoTools.sh`
- Create: `tests/bash/test_reinstall_gotoBase_gotoTools.sh`

**Interfaces:**

- Consumes: Task 1 的四个 Rscript、Docker CLI、零个或一个容器名参数。
- Produces: `reinstall_gotoBase_gotoTools.sh [container_name]`；成功为 `0`，参数、预检、步骤或验证失败为非零。

- [ ] **Step 1：写伪 Docker 行为测试**

创建 `tests/bash/test_reinstall_gotoBase_gotoTools.sh`：

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
readonly TEST_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd -P)"
readonly BATCH="$TEST_DIR/../../.bash/reinstall_gotoBase_gotoTools.sh"
readonly TEMP_DIR="$(mktemp -d)"
readonly FAKE_BIN="$TEMP_DIR/bin"
readonly FAKE_LOG="$TEMP_DIR/docker.log"
trap 'rm -rf -- "$TEMP_DIR"' EXIT
fail() { printf '测试失败：%s\n' "$1" >&2; exit 1; }

mkdir -p "$FAKE_BIN"
cat > "$FAKE_BIN/docker" <<'FAKE'
#!/usr/bin/env bash
set -Eeuo pipefail
case "${1:-}" in
  inspect)
    printf 'INSPECT:%s\n' "${@: -1}" >> "$FAKE_LOG"
    [[ "${FAKE_STATE:-running}" != missing ]] || exit 1
    [[ "${FAKE_STATE:-running}" != stopped ]] && printf 'true\n' || printf 'false\n'
    ;;
  exec)
    shift
    stdin=false
    if [[ "${1:-}" == -i ]]; then stdin=true; shift; fi
    printf 'CONTAINER:%s\n' "${1:-}" >> "$FAKE_LOG"
    shift
    $stdin || exit 0
    payload="$(cat)"
    if [[ "$payload" == *'check_package_source <- function'* ]]; then step=preflight
    elif [[ "$payload" == *'package_names <- c("gotoBase", "gotoTools")'* ]]; then step=verify
    elif [[ "$payload" == *'install_local('* && "$payload" == *'"gotoBase"'* ]]; then step=install_gotoBase
    elif [[ "$payload" == *'install_local('* && "$payload" == *'"gotoTools"'* ]]; then step=install_gotoTools
    elif [[ "$payload" == *'remove.packages('* && "$payload" == *'"gotoBase"'* ]]; then step=remove_gotoBase
    elif [[ "$payload" == *'remove.packages('* && "$payload" == *'"gotoTools"'* ]]; then step=remove_gotoTools
    else exit 9
    fi
    printf 'STEP:%s\n' "$step" >> "$FAKE_LOG"
    [[ "${FAKE_FAIL_STEP:-}" != "$step" ]] || exit 42
    ;;
  *) exit 10 ;;
esac
FAKE
chmod 0755 "$FAKE_BIN/docker"

run_batch() {
  PATH="$FAKE_BIN:$PATH" FAKE_LOG="$FAKE_LOG" \
    FAKE_STATE="${FAKE_STATE:-running}" FAKE_FAIL_STEP="${FAKE_FAIL_STEP:-}" \
    "$BATCH" "$@"
}

[[ -f "$BATCH" ]] || fail "批量入口不存在：$BATCH"
: > "$FAKE_LOG"
run_batch >/dev/null
actual="$(grep '^STEP:' "$FAKE_LOG")"
expected=$'STEP:preflight\nSTEP:remove_gotoTools\nSTEP:remove_gotoBase\nSTEP:install_gotoBase\nSTEP:install_gotoTools\nSTEP:verify'
[[ "$actual" == "$expected" ]] || fail "执行顺序错误：$actual"
grep -Fq 'INSPECT:goto_rbase_wsl' "$FAKE_LOG" || fail '默认容器错误'

: > "$FAKE_LOG"
run_batch custom_rbase >/dev/null
grep -Fq 'INSPECT:custom_rbase' "$FAKE_LOG" || fail '覆盖容器未生效'

: > "$FAKE_LOG"
if FAKE_STATE=missing run_batch missing_rbase >/dev/null 2>&1; then
  fail '不存在容器不应成功'
fi
! grep -Fq 'STEP:' "$FAKE_LOG" || fail '预检失败后执行了 R 步骤'

: > "$FAKE_LOG"
if FAKE_FAIL_STEP=install_gotoBase run_batch >/dev/null 2>&1; then
  fail 'gotoBase 安装失败不应成功'
fi
actual="$(grep '^STEP:' "$FAKE_LOG")"
expected=$'STEP:preflight\nSTEP:remove_gotoTools\nSTEP:remove_gotoBase\nSTEP:install_gotoBase'
[[ "$actual" == "$expected" ]] || fail "失败后仍继续：$actual"

: > "$FAKE_LOG"
if run_batch one two >/dev/null 2>&1; then fail '多余参数不应成功'; fi
[[ ! -s "$FAKE_LOG" ]] || fail '参数错误时不应调用 Docker'
printf '通过：批量入口参数、预检、顺序和快速失败正确。\n'
```

- [ ] **Step 2：运行测试确认失败**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-deploy-backend
chmod 0755 tests/bash/test_reinstall_gotoBase_gotoTools.sh
tests/bash/test_reinstall_gotoBase_gotoTools.sh
```

Expected: FAIL，报告批量入口不存在。

- [ ] **Step 3：实现批量入口**

创建 `.bash/reinstall_gotoBase_gotoTools.sh`：

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
readonly SCRIPT_DIR="$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" && pwd -P)"
readonly DEFAULT_CONTAINER_NAME=goto_rbase_wsl
readonly R_SCRIPTS=(remove_gotoTools.sh remove_gotoBase.sh install_gotoBase.sh install_gotoTools.sh)

if (( $# > 1 )); then
  printf '用法：%s [容器名]\n' "$(basename -- "$0")" >&2
  exit 2
fi
if (( $# == 1 )) && [[ -z "$1" ]]; then
  printf '错误：容器名不能为空。\n' >&2
  exit 2
fi
readonly container_name="${1:-$DEFAULT_CONTAINER_NAME}"
current_step=初始化
report_error() {
  local exit_code=$?
  printf '失败：步骤“%s”返回退出码 %d。\n' "$current_step" "$exit_code" >&2
  exit "$exit_code"
}
trap report_error ERR

printf '目标容器：%s\n' "$container_name"
current_step='检查 Docker 命令'
command -v docker >/dev/null 2>&1 || {
  printf '错误：WSL 宿主机未找到 docker 命令。\n' >&2; exit 1;
}
current_step='检查单包脚本'
for script_name in "${R_SCRIPTS[@]}"; do
  [[ -r "$SCRIPT_DIR/$script_name" ]] || {
    printf '错误：单包脚本不存在或不可读：%s\n' "$SCRIPT_DIR/$script_name" >&2
    exit 1
  }
done
current_step='检查目标容器'
if ! running="$(docker inspect --format '{{.State.Running}}' "$container_name" 2>/dev/null)"; then
  printf '错误：目标容器不存在或 Docker 不可访问：%s\n' "$container_name" >&2
  exit 1
fi
[[ "$running" == true ]] || {
  printf '错误：目标容器未运行：%s\n' "$container_name" >&2; exit 1;
}
current_step='检查容器内 Rscript'
docker exec "$container_name" sh -c 'command -v Rscript >/dev/null 2>&1'

current_step='检查容器内 R 环境和源码'
docker exec -i "$container_name" Rscript --vanilla /dev/stdin <<'RSCRIPT'
check_package_source <- function(package_path, expected_package_name) {
  description_path <- file.path(package_path, "DESCRIPTION")
  if (!file.exists(description_path)) stop(sprintf("未找到源码描述文件：%s", description_path))
  actual_package_name <- unname(read.dcf(description_path)[1, "Package"])
  if (!identical(actual_package_name, expected_package_name)) {
    stop(sprintf("源码包名不匹配：期望 %s，实际 %s",
                 expected_package_name, actual_package_name))
  }
}
if (!requireNamespace("remotes", quietly = TRUE)) stop("容器内未安装 remotes")
check_package_source("/root/software/gotoBase", "gotoBase")
check_package_source("/root/software/gotoTools", "gotoTools")
message("容器内 R 环境和源码预检通过")
RSCRIPT

run_r_script() {
  current_step="$1"
  printf '\n[%s] 开始\n' "$current_step"
  docker exec -i "$container_name" Rscript --vanilla /dev/stdin < "$SCRIPT_DIR/$2"
  printf '[%s] 成功\n' "$current_step"
}
run_r_script '移除 gotoTools' remove_gotoTools.sh
run_r_script '移除 gotoBase' remove_gotoBase.sh
run_r_script '安装 gotoBase' install_gotoBase.sh
run_r_script '安装 gotoTools' install_gotoTools.sh

current_step='验证安装结果'
printf '\n[验证安装结果] 开始\n'
docker exec -i "$container_name" Rscript --vanilla /dev/stdin <<'RSCRIPT'
package_names <- c("gotoBase", "gotoTools")
for (package_name in package_names) {
  if (!requireNamespace(package_name, quietly = TRUE)) stop(sprintf("无法加载 %s", package_name))
  message(sprintf("%s 加载成功，版本：%s", package_name,
                  as.character(utils::packageVersion(package_name))))
}
RSCRIPT
printf '[验证安装结果] 成功\n'
trap - ERR
printf '\n批量重装完成：gotoBase 与 gotoTools 均已验证。\n'
```

- [ ] **Step 4：运行测试并提交**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-deploy-backend
chmod 0755 .bash/reinstall_gotoBase_gotoTools.sh tests/bash/test_reinstall_gotoBase_gotoTools.sh
bash -n .bash/reinstall_gotoBase_gotoTools.sh
tests/bash/test_r_package_scripts.sh
tests/bash/test_reinstall_gotoBase_gotoTools.sh
git diff --check
git add .bash/reinstall_gotoBase_gotoTools.sh tests/bash/test_reinstall_gotoBase_gotoTools.sh
git commit -m "feat: 增加goto包批量重装入口"
```

Expected: `bash -n` 和 `git diff --check` 无输出；两个测试均通过；提交仅含批量入口及其测试。

## Task 3：执行容器集成验证

**Files:**

- Verify: `.bash/install_gotoBase.sh`
- Verify: `.bash/remove_gotoBase.sh`
- Verify: `.bash/install_gotoTools.sh`
- Verify: `.bash/remove_gotoTools.sh`
- Verify: `.bash/reinstall_gotoBase_gotoTools.sh`
- Verify: `tests/bash/test_r_package_scripts.sh`
- Verify: `tests/bash/test_reinstall_gotoBase_gotoTools.sh`

**Interfaces:**

- Consumes: Tasks 1-2 的提交、运行中的 `goto_rbase_wsl` 和两个挂载源码目录。
- Produces: Bash/R 语法、真实预检、完整重装、加载版本和容器未重启的验证证据。

- [ ] **Step 1：确认容器状态并记录启动身份**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-deploy-backend
docker inspect --format '{{.State.Running}}' goto_rbase_wsl
container_before="$(docker inspect --format '{{.Id}} {{.State.StartedAt}}' goto_rbase_wsl)"
printf '%s\n' "$container_before"
```

Expected: 输出 `true`，随后输出容器 ID 和启动时间。若容器不存在或未运行，停止并报告，不得自动创建或启动。

- [ ] **Step 2：检查 Bash 和四个 Rscript 的语法**

Run:

```bash
bash -n .bash/reinstall_gotoBase_gotoTools.sh
for script_path in .bash/remove_gotoTools.sh .bash/remove_gotoBase.sh \
  .bash/install_gotoBase.sh .bash/install_gotoTools.sh; do
  docker exec -i goto_rbase_wsl Rscript --vanilla -e \
    'parse(file = "/dev/stdin"); message("R 语法通过")' < "$script_path"
done
```

Expected: Bash 检查无输出；四次 R 检查各输出 `R 语法通过`，且不执行移除或安装。

- [ ] **Step 3：验证不存在容器的安全失败路径**

Run:

```bash
set +e
missing_output="$(.bash/reinstall_gotoBase_gotoTools.sh __goto_missing_container__ 2>&1)"
missing_status=$?
set -e
printf '%s\n' "$missing_output"
test "$missing_status" -ne 0
```

Expected: 包含 `目标容器不存在或 Docker 不可访问`，状态非 `0`，且没有移除或安装步骤开始信息。

- [ ] **Step 4：重新运行全部宿主机测试**

Run:

```bash
tests/bash/test_r_package_scripts.sh
tests/bash/test_reinstall_gotoBase_gotoTools.sh
```

Expected: 两个测试均输出 `通过` 并返回 `0`。

- [ ] **Step 5：执行一次真实批量重装**

Run:

```bash
.bash/reinstall_gotoBase_gotoTools.sh
```

Expected: 严格依次显示 `移除 gotoTools`、`移除 gotoBase`、`安装 gotoBase`、`安装 gotoTools`、`验证安装结果`；最后输出 `批量重装完成：gotoBase 与 gotoTools 均已验证。` 并返回 `0`。

- [ ] **Step 6：独立验证版本和未重启状态**

Run:

```bash
docker exec goto_rbase_wsl Rscript --vanilla -e '
for (package_name in c("gotoBase", "gotoTools")) {
  stopifnot(requireNamespace(package_name, quietly = TRUE))
  message(sprintf("%s=%s", package_name,
                  as.character(utils::packageVersion(package_name))))
}'
container_after="$(docker inspect --format '{{.Id}} {{.State.StartedAt}}' goto_rbase_wsl)"
printf '%s\n' "$container_after"
test "$container_after" = "$container_before"
```

Expected: 输出 `gotoBase=0.1.0`、`gotoTools=0.1.0`；`container_after` 与 `container_before` 完全一致。

- [ ] **Step 7：最终仓库检查**

Run:

```bash
git diff --check
git status --short
git log -2 --oneline
```

Expected: `git diff --check` 无输出；本任务七个文件已提交；原有 `.bash/command.sh`、`.bash/wsl_rbase_start.sh` 等未提交改动保持原状；最近两条提交对应 Tasks 1-2。

## 完成定义

- 四个 Rscript 和一个 Bash 批量入口按设计落地，权限为 `0755`。
- 持久化测试覆盖解释器、安装参数、默认/覆盖容器、执行顺序、预检失败和快速失败。
- 不存在容器的真实测试在首次移除前失败。
- 默认容器完整重装成功，新 R 进程能加载两个包并输出版本。
- 容器 ID 与启动时间前后一致，证明没有重启容器。
- 只提交本计划列出的七个实现与测试文件，既有未提交改动保持原状。
