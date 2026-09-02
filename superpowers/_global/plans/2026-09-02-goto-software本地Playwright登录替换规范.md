---
change_set_id: CS-20260902-a3d63815-f543-4668-a63f-d81e4c3caf85
related_change_set_ids: []
---

# goto-software 本地 Playwright 登录替换规范实施计划

> **供执行 Agent：** 必须使用 `superpowers:subagent-driven-development`（推荐）或 `superpowers:executing-plans`，按任务逐项实施。所有步骤使用复选框跟踪。

**目标：** 在 goto-software 项目入口中固化本地 Playwright 验收的登录页临时替换、固定验证码和无条件恢复规则。

**架构：** 只增强现有项目入口章节，不新增规则文件、脚本或 hook。入口继续承载后端启动硬约束，并补充前端替换的干净门禁、临时备份、受控登录和恢复校验。

**技术栈：** Markdown、YAML frontmatter、Git 只读状态检查、SHA-256、现有 `validate_rules.py` 与 Python `unittest`。

## 全局约束

- `Change-Set-Id` 固定为 `CS-20260902-a3d63815-f543-4668-a63f-d81e4c3caf85`。
- 只修改 `/mnt/g/Obsidian/code/01_Project_Specs/goto_software_spec.md`。
- 不实际修改 `/mnt/f/IdeaProjects/goto-software/goto-web/src/views/login.vue` 或 `/mnt/f/IdeaProjects/goto-software/goto-web/src/views/login公众号验证码登录.vue`。
- 不创建登录替换脚本、Playwright 用例、hook、第二份项目细则或运行时配置。
- 保留目标规范中现有未提交的后端本地验收段落，并在其基础上追加规则，不覆盖或删除用户内容。
- 目标规范包含用户既有未提交内容，执行阶段不得暂存或提交该文件；完成后保留工作区 diff 供用户审核。
- 固定码 `666888` 只允许配合 `-Denv=local` 的本地受控验收，不得用于生产、共享环境或其他 profile。
- `login.vue` 存在已暂存或未暂存修改时必须停止替换，不得用临时备份绕过门禁。

---

### Task 1：补齐 goto-software 本地 Playwright 登录替换规范

**文件：**
- 修改：`/mnt/g/Obsidian/code/01_Project_Specs/goto_software_spec.md:1`
- 测试：`/mnt/g/Obsidian/code/01_System_Core/tests/test_validate_rules.py`
- 只读确认：`/mnt/f/IdeaProjects/goto-software/goto-web/src/views/login.vue`
- 只读确认：`/mnt/f/IdeaProjects/goto-software/goto-web/src/views/login公众号验证码登录.vue`

**接口：**
- 输入：需要微信公众号验证码登录的 goto-software 本地 Playwright 验收。
- 输出：项目入口中的后端启动、前端替换、固定码输入、失败恢复和完成门禁。
- 不产生可执行脚本、Vue 变更、测试用例或 hook。

- [ ] **步骤 1：运行 RED 语义检查并记录 Vue 文件基线**

在 `/mnt/g/Obsidian` 运行：

```bash
spec='code/01_Project_Specs/goto_software_spec.md'
missing=0
for requirement in \
  'global.local-runtime-verification' \
  '/mnt/f/IdeaProjects/goto-software/goto-web/src/views/login\.vue' \
  '/mnt/f/IdeaProjects/goto-software/goto-web/src/views/login公众号验证码登录\.vue' \
  '666888' \
  'SHA-256' \
  '已暂存.*未暂存' \
  '独立 Git 根' \
  '源/目标文件均已跟踪' \
  '检查命令失败' \
  '成功、失败或中止.*恢复'; do
  if ! rg -q "$requirement" "$spec"; then
    printf '缺少规范语义：%s\n' "$requirement"
    missing=1
  fi
done
test "$missing" -eq 0
```

预期：命令退出码为 `1`，并报告当前缺少的依赖、绝对路径、固定码、干净门禁或恢复语义。

在 `/mnt/f/IdeaProjects/goto-software/goto-web` 运行：

```bash
set -e
test "$(git rev-parse --show-toplevel)" = "/mnt/f/IdeaProjects/goto-software/goto-web"
git ls-files --error-unmatch \
  'src/views/login.vue' \
  'src/views/login公众号验证码登录.vue'
git diff --quiet -- 'src/views/login.vue'
git diff --cached --quiet -- 'src/views/login.vue'
git status --short -- 'src/views/login.vue'
test -f 'src/views/login.vue'
test -f 'src/views/login公众号验证码登录.vue'
```

预期：Git 根精确为独立 `goto-web` 仓库，两份文件均已跟踪且存在，`login.vue` 没有已暂存或未暂存修改，`git status` 没有输出；本任务不得继续修改它们。

- [ ] **步骤 2：声明项目入口对全局本地验收规则的直接依赖**

在 `goto_software_spec.md` frontmatter 中把 `requires` 调整为：

```yaml
requires:
  - global.entry
  - global.local-runtime-verification
  - goto-software.superpowers-docs
```

不得修改其他 frontmatter 字段。

- [ ] **步骤 3：用完整章节替换现有本地验收章节**

保留章节前的分隔线，将现有“本地后端与 Playwright 验收规范”章节替换为：

```markdown
## 本地后端、前端与 Playwright 验收规范

触发条件：为 goto-software 启动本地前端或后端进行验收，或使用 Playwright 验证需要微信公众号验证码登录的页面。

常驻硬约束：
- 后端以 `system/target/goto-manager.jar` 启动用于本地或 Playwright 验收时，必须在 `-jar` 前传入 JVM 参数 `-Denv=local`，避免测试登录成功后删除微信公众号验证码。
- 验收必须使用 `home` Spring profile，即传入 JVM 参数 `-Dspring.profiles.active=home`，以加载 `application-home.yml`；不得因默认 profile 的数据库认证失败而改用共享默认端口或绕过配置。
- 启动命令形态为 `java -Denv=local -Dspring.profiles.active=home -jar system/target/goto-manager.jar --server.port=<本次动态端口>`；前后端端口选择、前台托管、关闭和 Playwright 隔离仍服从全局本地运行验收规范。
- Playwright 验收需要微信公众号验证码登录时，源文件固定为 `/mnt/f/IdeaProjects/goto-software/goto-web/src/views/login公众号验证码登录.vue`，目标文件固定为 `/mnt/f/IdeaProjects/goto-software/goto-web/src/views/login.vue`；替换前必须确认两者均存在。
- `goto-web` 独立 Git 根固定是 `/mnt/f/IdeaProjects/goto-software/goto-web`；替换前必须在该仓库确认 `git rev-parse --show-toplevel` 输出完全匹配该根、源/目标文件均已跟踪，并分别通过 `git diff --quiet -- src/views/login.vue` 和 `git diff --cached --quiet -- src/views/login.vue` 确认目标文件没有未暂存和已暂存修改。
- 仓库归属不符、路径未跟踪、检查命令失败或任一差异存在时一律立即停止，不得覆盖，并向用户报告准确状态。
- 目标文件干净时，先在本次命令创建的项目外临时目录中保存其原始字节副本并记录 SHA-256，再把源文件的完整内容复制到目标文件；只允许复制，不得移动、删除或修改源文件。
- 本地登录时在验证码输入框使用固定验证码 `666888`；该验证码只允许配合 `-Denv=local` 的本地受控验收，不得用于生产、共享环境或其他 profile。
- 无论验收成功、失败、中止或后续步骤报错，都必须恢复目标文件的原始字节内容；验收失败时也必须先完成恢复与校验，再报告失败。
- 恢复后必须确认目标文件 SHA-256 与替换前一致，并在该独立仓库复核目标文件无新增已暂存或未暂存差异；任一检查失败时必须保留临时备份、报告残留状态，且不得声称验收完成。
- 不得提交临时替换后的目标文件，也不得把固定验证码或备份内容固化到前端代码、测试脚本、配置或其他项目文件；Playwright 截图、trace、video 和输出继续使用本次命令创建的临时目录。

执行要求：启动 goto-software 本地前端、后端或进行相关 Playwright 验收前，先读取本节与全局 `rules/local-runtime-verification.md`。
```

- [ ] **步骤 4：运行 GREEN 语义闭环检查**

在 `/mnt/g/Obsidian` 运行：

```bash
spec='code/01_Project_Specs/goto_software_spec.md'
for requirement in \
  'global.local-runtime-verification' \
  '/mnt/f/IdeaProjects/goto-software/goto-web/src/views/login\.vue' \
  '/mnt/f/IdeaProjects/goto-software/goto-web/src/views/login公众号验证码登录\.vue' \
  '666888.*-Denv=local' \
  '独立 Git 根固定是 `/mnt/f/IdeaProjects/goto-software/goto-web`' \
  'git rev-parse --show-toplevel' \
  '源/目标文件均已跟踪' \
  'git diff --quiet -- src/views/login\.vue' \
  'git diff --cached --quiet -- src/views/login\.vue' \
  '检查命令失败.*停止' \
  '原始字节副本.*SHA-256' \
  '成功、失败、中止.*恢复' \
  'SHA-256 与替换前一致' \
  '独立仓库复核.*无新增已暂存或未暂存差异' \
  '不得提交临时替换后的目标文件'; do
  rg -q "$requirement" "$spec" || exit 1
done
if rg -q '敏感临时凭据' "$spec"; then
  exit 1
fi
echo 'Playwright 登录替换规范语义检查通过'
```

预期：输出“Playwright 登录替换规范语义检查通过”。

- [ ] **步骤 5：运行规范校验器和现有测试**

在 `/mnt/g/Obsidian` 运行：

```bash
python3 code/01_System_Core/scripts/validate_rules.py \
  --root code \
  --skills-root skill
python3 code/01_System_Core/tests/test_validate_rules.py
```

预期：校验器输出“规范校验通过”；单元测试运行 `89` 项并以 `OK` 结束。

- [ ] **步骤 6：检查修改范围并保留用户工作区状态**

在 `/mnt/g/Obsidian` 运行：

```bash
git diff --check -- code/01_Project_Specs/goto_software_spec.md
git diff --cached --quiet -- code/01_Project_Specs/goto_software_spec.md
git status --short -- code/01_Project_Specs/goto_software_spec.md
git diff --stat -- code/01_Project_Specs/goto_software_spec.md
```

预期：格式检查通过，目标规范未被暂存，状态仍为工作区修改，diff 只涉及该项目入口。

在 `/mnt/f/IdeaProjects/goto-software/goto-web` 运行：

```bash
set -e
test "$(git rev-parse --show-toplevel)" = "/mnt/f/IdeaProjects/goto-software/goto-web"
git ls-files --error-unmatch \
  'src/views/login.vue' \
  'src/views/login公众号验证码登录.vue'
git diff --quiet -- 'src/views/login.vue'
git diff --cached --quiet -- 'src/views/login.vue'
git status --short -- 'src/views/login.vue'
```

预期：所有检查均通过且 `git status` 没有输出，证明本次只编写规范，未实际替换或修改两个 Vue 文件。

- [ ] **步骤 7：报告结果但不提交目标规范**

报告必须包含：

```text
- 修改文件：/mnt/g/Obsidian/code/01_Project_Specs/goto_software_spec.md
- 规范校验：通过
- test_validate_rules.py：89/89 通过
- 两份 Vue 文件：未修改
- 目标规范：保留为未暂存、未提交工作区修改，供用户审核
- Change-Set-Id：CS-20260902-a3d63815-f543-4668-a63f-d81e4c3caf85
```

不得执行 `git add`、`git commit`、远端操作、Vue 文件复制、服务启动或 Playwright。
