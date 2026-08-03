# R Package Namespace Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `hybrid-subagent-driven-development` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate Java-generated R calls from runtime `source(...)` loading to explicit `gotoTools::`, `gotoPlot::`, and `gotoStat::` package namespace calls.

**Architecture:** The current Codex session remains controller, reviewer, planner, and user-facing decision maker. DeepSeek implementer turns run through `codex-dp-exec` in an isolated `goto` worktree; the controller reviews spec compliance first, then code quality, after each task.

**Tech Stack:** Java, Spring Boot, Maven multi-module repo (`common`, `task`, `system`), Rserve/Rscript command generation, Mockito/JUnit tests, ripgrep, `codex-dp-exec`.

---

## 执行前上下文

Approved spec:

```text
/mnt/f/IdeaProjects/goto-software/goto-docs/superpowers/goto/superpowers/specs/2026-06-29-r-package-namespace-migration-design.md
```

Primary repo:

```text
/mnt/f/IdeaProjects/goto-software/goto
```

Planned implementation worktree:

```text
/mnt/f/IdeaProjects/goto-software/.worktrees/goto-r-package-namespace-migration
```

Base branch observed during planning:

```text
master
```

Known dirty files in `/mnt/f/IdeaProjects/goto-software/goto` before planning:

```text
M common/src/main/resources/application-common-home.yml
M system/src/main/java/com/freedom/service/impl/PlotTypeServiceImpl.java
M task/src/main/java/com/freedom/bean/mq/redisstream/CommonTaskCommandStreamConsumer.java
M task/src/main/java/com/freedom/bean/mq/redisstream/GenerateSampleTaskStreamConsumer.java
```

These files are not part of this migration unless a later task explicitly names them. Do not revert or overwrite them.

Project documentation rule from `AGENTS.md`: superpowers plans for this project must live under:

```text
/mnt/f/IdeaProjects/goto-software/goto-docs/superpowers/goto/superpowers/plans
```

## Hybrid Subagent-Driven 执行约定

Controller command template:

```bash
/home/gyq/.local/bin/codex-dp-exec doctor --cd /mnt/f/IdeaProjects/goto-software/.worktrees/goto-r-package-namespace-migration

/home/gyq/.local/bin/codex-dp-exec \
  --cd /mnt/f/IdeaProjects/goto-software/.worktrees/goto-r-package-namespace-migration \
  --prompt-file <prompt-file> \
  --task-id <task-id> \
  --controller-id <codex1-or-codex2-or-codex-dp-or-deepseek-main> \
  --output-last-message <report-file>

/home/gyq/.local/bin/codex-dp-exec status \
  --cd /mnt/f/IdeaProjects/goto-software/.worktrees/goto-r-package-namespace-migration \
  --task-id <task-id>
```

Controller must check:

- [ ] `codex-dp-exec doctor --cd /mnt/f/IdeaProjects/goto-software/.worktrees/goto-r-package-namespace-migration` before first implementation run.
- [ ] `codex-dp-exec list --cd /mnt/f/IdeaProjects/goto-software/.worktrees/goto-r-package-namespace-migration` when resuming the plan or auditing multiple tasks.
- [ ] `codex-dp-exec status --cd /mnt/f/IdeaProjects/goto-software/.worktrees/goto-r-package-namespace-migration --task-id <task-id>` after each implementer run.
- [ ] Implementer report after each task.
- [ ] `git diff` and changed files after each task.
- [ ] Test output after each task.
- [ ] Spec compliance review before code quality review. A task is complete only after both pass.

Recovery rules:

- If a run is `interrupted`, inspect current `git diff` before retrying.
- On resume, run `codex-dp-exec list --cd /mnt/f/IdeaProjects/goto-software/.worktrees/goto-r-package-namespace-migration` and `codex-dp-exec status --cd /mnt/f/IdeaProjects/goto-software/.worktrees/goto-r-package-namespace-migration --task-id <task-id>`.
- Recovery creates a new run for the same `task_id`; do not overwrite the interrupted run.
- If DeepSeek returns `BLOCKED` or `NEEDS_CONTEXT`, the controller decides whether to provide more context, split the task, retry, or escalate to the user.
- Temporary prompt, report, and run directories must not be committed unless the user explicitly asks.

DeepSeek final report requirements for every implementation task:

```text
最终报告必须包含：
- status: DONE | DONE_WITH_CONCERNS | BLOCKED | NEEDS_CONTEXT
- changed_files: 本任务改动的文件列表
- commits: 本任务产生的 commit hash；没有提交时说明原因
- tests: 实际运行的测试命令和结果
- concerns: 风险、未验证项或需要 controller 判断的问题

执行要求：
- 只能修改本 Task 声明的仓库和文件范围。
- 不得回退用户或其他 agent 的无关改动。
- 不得执行 mvn deploy。
- 不得通过数据库客户端执行写操作。
- 如果计划与代码现状冲突，停止并返回 NEEDS_CONTEXT，不要猜测。
```

Focused fix prompts after review findings must include:

- `Original Task`: task_id, title, goal, worktree, editable files, forbidden changes, and tests.
- `Current State`: git diff summary, changed files, latest commit hash if any, previous report status, and tests already run.
- `Review Findings To Fix`: separate spec compliance findings from code quality findings.
- `Allowed Fix Scope`: files and behavior the fix may touch; explicit out-of-scope items.
- `Required Tests`: tests to rerun after the fix.
- `Self Review Before Final Report`: verify listed findings are fixed, no unrelated changes were introduced, and tests were rerun.

## 最终审核策略与交接包

```text
final_review_policy: current-controller
final_review_status_on_completion: final-review-completed
reviewer_required: none
handoff_package: final-review-handoff.md
```

The final verification task must write or update:

```text
/mnt/f/IdeaProjects/goto-software/.worktrees/goto-r-package-namespace-migration/final-review-handoff.md
```

Recommended handoff command:

```bash
/home/gyq/.local/bin/codex-dp-exec handoff \
  --cd /mnt/f/IdeaProjects/goto-software/.worktrees/goto-r-package-namespace-migration \
  --controller-id <controller-id> \
  --controller-model <controller-model-or-unknown> \
  --final-review-policy current-controller \
  --spec /mnt/f/IdeaProjects/goto-software/goto-docs/superpowers/goto/superpowers/specs/2026-06-29-r-package-namespace-migration-design.md \
  --plan /mnt/f/IdeaProjects/goto-software/goto-docs/superpowers/goto/superpowers/plans/2026-06-29-r-package-namespace-migration-hybrid-plan.md \
  --output /mnt/f/IdeaProjects/goto-software/.worktrees/goto-r-package-namespace-migration/final-review-handoff.md
```

The handoff must include controller id and model/provider, spec path, plan path, worktree path, branch, relevant commits, latest run directory/report/status per task, final verification commands/results, open concerns, and next action.

## 字段/接口/约束映射

R package mapping:

- `gotoTools`: tool/upload file operations.
- `gotoPlot`: plot drawing methods and download image generation.
- `gotoStat`: statistics and data process methods.

Known function mapping:

- `GT_MergeImages(...)` -> `gotoTools::mergeImages(...)`
- `fileConvert(...)` -> `gotoTools::fileConvert(...)`
- `reshapeData(...)` -> `gotoTools::reshapeData(...)`
- `mergeTableByColumn(...)` -> `gotoTools::mergeTableByColumn(...)`
- `GGSAVE(...)` -> `gotoPlot::GGSAVE(...)`
- `GT_BasicStat(...)` -> `gotoStat::GT_BasicStat(...)`
- `GT_DifferentialStat(...)` -> `gotoStat::GT_DifferentialStat(...)`
- `GT_Corrlation(...)` -> `gotoStat::GT_Corrlation(...)`
- `GT_Ropls(...)` -> `gotoStat::GT_Ropls(...)`
- Data process methods `remove_duplication`, `classify_variables`, `quality_assessment`, `data_cleaning`, `impute_missing`, `transform_data`, `precess_result` -> `gotoStat::<method>`.

Compatibility rule:

- If an R function name already contains `::`, return it unchanged.
- If an R function name does not contain `::`, wrap it with the package namespace required by the caller.
- Do not migrate database values for `PlotType.method`, `UserDraw.method`, or source fields.
- Do not remove model fields such as `PlotType.source`, `CommonTask.source`, or `DataProcessTask.source` unless implementation proves this is compile-safe and stays inside the task scope. Conservative default is to leave fields in place and stop using them for source loading.

## 文件结构

Expected touched production files:

```text
common/src/main/java/com/freedom/util/RFunctionNamespaceUtils.java
common/src/main/java/com/freedom/util/PlotTypeUtils.java
common/src/main/java/com/freedom/util/StatisticsUtils.java
task/src/main/java/com/freedom/executor/RFunctionExecutor.java
task/src/main/java/com/freedom/executor/RScriptExecutor.java
task/src/main/java/com/freedom/handler/DataProcessBaseHandler.java
task/src/main/java/com/freedom/handler/StatisticsStartAnalyzeMessageHandler.java
task/src/main/java/com/freedom/handler/MergeImageMessageHandler.java
task/src/main/java/com/freedom/handler/ImageConvertMessageHandler.java
task/src/main/java/com/freedom/handler/FileReshapeMessageHandler.java
task/src/main/java/com/freedom/handler/FileMergeMessageHandler.java
task/src/main/java/com/freedom/model/stage/UserDrawStage.java
task/src/main/java/com/freedom/model/stage/GenerateSampleDataStage.java
task/src/main/java/com/freedom/model/stage/CommonTaskStage.java
system/src/main/java/com/freedom/rest/DataProcessController.java
system/src/main/java/com/freedom/model/file/download/impl/DrawPlotDownload.java
```

Expected touched test files:

```text
common/src/test/java/com/freedom/util/RFunctionNamespaceUtilsTest.java
common/src/test/java/com/freedom/util/PlotTypeUtilsNamespaceTest.java
common/src/test/java/com/freedom/util/StatisticsUtilsNamespaceTest.java
task/src/test/java/com/freedom/invokeR/RTest.java
task/src/test/java/com/freedom/handler/MergeImageMessageHandlerTest.java
task/src/test/java/com/freedom/handler/FileReshapeMessageHandlerTest.java
task/src/test/java/com/freedom/handler/ImageConvertMessageHandlerTest.java
task/src/test/java/com/freedom/handler/FileMergeMessageHandlerTest.java
system/src/test/java/com/freedom/model/file/download/impl/DrawPlotDownloadTest.java
```

Implementation may create a different focused test filename if it is in the same module and clearly tied to this migration.

## Task 0: 工作区核查

**task_id:** `task-000-worktree-check`

**Execution mode:** `controller inline`

**Worktree:** `/mnt/f/IdeaProjects/goto-software/.worktrees/goto-r-package-namespace-migration`

**Goal:** Create or verify the isolated implementation worktree and ensure Hybrid execution can run before DeepSeek changes code.

**Editable Files:** none, except native git worktree metadata created by `git worktree add` if the worktree does not exist.

**Forbidden Changes:**

- Do not modify application source.
- Do not revert dirty files in `/mnt/f/IdeaProjects/goto-software/goto`.
- Do not commit from Task 0.

**Required Steps:**

- [ ] Run `git -C /mnt/f/IdeaProjects/goto-software/goto status --short` and record dirty files.
- [ ] If `/mnt/f/IdeaProjects/goto-software/.worktrees/goto-r-package-namespace-migration` does not exist, create it from `/mnt/f/IdeaProjects/goto-software/goto` on a branch such as `r-package-namespace-migration`.
- [ ] Run `git -C /mnt/f/IdeaProjects/goto-software/.worktrees/goto-r-package-namespace-migration branch --show-current`.
- [ ] Run `git -C /mnt/f/IdeaProjects/goto-software/.worktrees/goto-r-package-namespace-migration status --short`.
- [ ] Run `/home/gyq/.local/bin/codex-dp-exec doctor --cd /mnt/f/IdeaProjects/goto-software/.worktrees/goto-r-package-namespace-migration`.
- [ ] Stop before Task 1 if doctor fails or if the worktree has unexpected dirty files.

**Expected Result:** Worktree exists, branch is known, status is understood, and `codex-dp-exec doctor` succeeds.

## Task 1: Common Namespace Utilities And Generated R Strings

**task_id:** `task-001-common-namespace-generation`

**Execution mode:** `DeepSeek implementer + controller重点复核`

**Exact `--cd`:** `/mnt/f/IdeaProjects/goto-software/.worktrees/goto-r-package-namespace-migration`

**Goal:** Add namespace wrapping utility and migrate common-module generated plot/statistics R strings to package-qualified calls.

**Why This Task Exists:** This establishes the reusable compatibility rule used later by task and system modules. It covers the approved spec requirements for `gotoPlot` method compatibility and `gotoStat` statistics command generation.

**Editable Files:**

```text
common/src/main/java/com/freedom/util/RFunctionNamespaceUtils.java
common/src/main/java/com/freedom/util/PlotTypeUtils.java
common/src/main/java/com/freedom/util/StatisticsUtils.java
common/src/test/java/com/freedom/util/RFunctionNamespaceUtilsTest.java
common/src/test/java/com/freedom/util/PlotTypeUtilsNamespaceTest.java
common/src/test/java/com/freedom/util/StatisticsUtilsNamespaceTest.java
```

**Forbidden Changes:**

- Do not modify database schemas, repositories, controllers, or R code.
- Do not change `AutoParamCreateUtils` output semantics.
- Do not migrate persisted method/source values.
- Do not touch dirty files listed in the execution context unless they are in the editable files list; none are.

**Existing Context:**

- `PlotTypeUtils.autoCreateParam(...)` currently appends `data.getMethod()` directly.
- `StatisticsUtils` currently builds strings beginning with `GT_BasicStat(`, `GT_DifferentialStat(`, `GT_Corrlation(`, and `GT_Ropls(`.
- The utility must return unchanged names when they already contain `::`.

**Required Implementation Steps:**

- [ ] Create `RFunctionNamespaceUtils` with a static method such as `withNamespace(String packageName, String functionName)`.
- [ ] Method behavior: return input unchanged if function name is blank or already contains `::`; otherwise return `packageName + "::" + functionName`.
- [ ] In `PlotTypeUtils.autoCreateParam(...)`, wrap `data.getMethod()` with `gotoPlot` before building the function call.
- [ ] In `StatisticsUtils`, update generated statistics function names to `gotoStat::GT_BasicStat`, `gotoStat::GT_DifferentialStat`, `gotoStat::GT_Corrlation`, and `gotoStat::GT_Ropls`.
- [ ] Add focused tests for utility behavior, plot method wrapping, and statistics generated command prefixes.

**Required Tests:**

```bash
mvn -pl common -Dtest=RFunctionNamespaceUtilsTest,PlotTypeUtilsNamespaceTest,StatisticsUtilsNamespaceTest test
```

Expected result: Maven exits 0. Tests prove namespace wrapping is idempotent and generated strings use `gotoPlot::` / `gotoStat::`.

**Commit Command:**

```bash
git add common/src/main/java/com/freedom/util/RFunctionNamespaceUtils.java \
  common/src/main/java/com/freedom/util/PlotTypeUtils.java \
  common/src/main/java/com/freedom/util/StatisticsUtils.java \
  common/src/test/java/com/freedom/util/RFunctionNamespaceUtilsTest.java \
  common/src/test/java/com/freedom/util/PlotTypeUtilsNamespaceTest.java \
  common/src/test/java/com/freedom/util/StatisticsUtilsNamespaceTest.java
git commit -m "refactor: add R package namespace generation"
```

**Self Review Before Final Report:**

- [ ] Confirm no source-loading behavior was added.
- [ ] Confirm already qualified function names are not double-prefixed.
- [ ] Confirm tests are not overly coupled to unrelated parameter formatting.

## Task 2: Task Module R Executors And Tool/Statistics Handlers

**task_id:** `task-002-task-module-r-source-removal`

**Execution mode:** `DeepSeek implementer + controller重点复核`

**Exact `--cd`:** `/mnt/f/IdeaProjects/goto-software/.worktrees/goto-r-package-namespace-migration`

**Goal:** Remove source generation from task-module R execution paths and migrate task handlers to package-qualified R calls.

**Why This Task Exists:** The task module is the largest producer of Java-generated R commands. It must stop generating `source(...)`, and tool/statistics/data-process tasks must call `gotoTools::` or `gotoStat::` directly.

**Editable Files:**

```text
task/src/main/java/com/freedom/executor/RFunctionExecutor.java
task/src/main/java/com/freedom/executor/RScriptExecutor.java
task/src/main/java/com/freedom/handler/DataProcessBaseHandler.java
task/src/main/java/com/freedom/handler/StatisticsStartAnalyzeMessageHandler.java
task/src/main/java/com/freedom/handler/MergeImageMessageHandler.java
task/src/main/java/com/freedom/handler/ImageConvertMessageHandler.java
task/src/main/java/com/freedom/handler/FileReshapeMessageHandler.java
task/src/main/java/com/freedom/handler/FileMergeMessageHandler.java
task/src/main/java/com/freedom/model/stage/UserDrawStage.java
task/src/main/java/com/freedom/model/stage/GenerateSampleDataStage.java
task/src/main/java/com/freedom/model/stage/CommonTaskStage.java
task/src/test/java/com/freedom/invokeR/RTest.java
task/src/test/java/com/freedom/handler/MergeImageMessageHandlerTest.java
task/src/test/java/com/freedom/handler/FileReshapeMessageHandlerTest.java
task/src/test/java/com/freedom/handler/ImageConvertMessageHandlerTest.java
task/src/test/java/com/freedom/handler/FileMergeMessageHandlerTest.java
```

**Forbidden Changes:**

- Do not modify stream consumer dirty files: `task/src/main/java/com/freedom/bean/mq/redisstream/CommonTaskCommandStreamConsumer.java` or `GenerateSampleTaskStreamConsumer.java`.
- Do not remove model fields from `CommonTask` or `DataProcessTask`.
- Do not alter task queue semantics, ACK behavior, or thread pool key behavior.
- Do not execute R or require R packages during tests.

**Existing Context:**

- `RFunctionExecutor` currently loops `request.getSources()` and adds `source('<codeRoot>/<source>')`.
- `RScriptExecutor` currently adds `source(...)` before calling `request.getMethod()`.
- `DataProcessBaseHandler` builds `RFunctionRequest` from `task.getMethod()` and `Arrays.asList(task.getSource())`.
- `MergeImageMessageHandler` currently builds source list and calls `GT_MergeImages(`.
- `ImageConvertMessageHandler` currently sets source and calls `fileConvert(`.
- `FileReshapeMessageHandler` currently sets source and calls `reshapeData(`.
- `FileMergeMessageHandler` currently sets source and calls `mergeTableByColumn(`.
- `StatisticsStartAnalyzeMessageHandler` currently sets `Stat/...` source entries; statistics param strings are generated by common Task 1.
- `UserDrawStage`, `GenerateSampleDataStage`, and `CommonTaskStage` override `obtainSourceNames()` to return task/data source lists. Since local `CommonStage` source was not present in the repo during planning, return empty lists or otherwise stop these local stages from providing source names without changing their execute flow.

**Required Implementation Steps:**

- [ ] Update `RFunctionExecutor` to ignore `request.getSources()` and never add source commands.
- [ ] Update `RScriptExecutor` to ignore `request.getSourceFiles()` and never append source commands.
- [ ] In `DataProcessBaseHandler`, wrap `task.getMethod()` with `RFunctionNamespaceUtils.withNamespace("gotoStat", task.getMethod())` and do not set sources.
- [ ] In `StatisticsStartAnalyzeMessageHandler`, remove source list construction and `commonTask.setSource(...)`.
- [ ] In `MergeImageMessageHandler`, remove source-list construction/use and generate `gotoTools::mergeImages(`.
- [ ] In `ImageConvertMessageHandler`, remove source setup and generate `gotoTools::fileConvert(`.
- [ ] In `FileReshapeMessageHandler`, remove source setup and generate `gotoTools::reshapeData(`.
- [ ] In `FileMergeMessageHandler`, remove source setup and generate `gotoTools::mergeTableByColumn(`.
- [ ] In local stage classes, ensure source lists are not used to load R source files; prefer returning empty lists if signatures require overriding.
- [ ] Update tests to assert no `source(...)` commands are generated and handler params contain the new package-qualified calls.

**Required Tests:**

```bash
mvn -pl task -Dtest=RTest,MergeImageMessageHandlerTest,FileReshapeMessageHandlerTest,ImageConvertMessageHandlerTest,FileMergeMessageHandlerTest test
```

Expected result: Maven exits 0. If a listed test class does not exist before implementation, create it or adjust the exact test list to the created focused tests and report the change.

**Commit Command:**

```bash
git add task/src/main/java/com/freedom/executor/RFunctionExecutor.java \
  task/src/main/java/com/freedom/executor/RScriptExecutor.java \
  task/src/main/java/com/freedom/handler/DataProcessBaseHandler.java \
  task/src/main/java/com/freedom/handler/StatisticsStartAnalyzeMessageHandler.java \
  task/src/main/java/com/freedom/handler/MergeImageMessageHandler.java \
  task/src/main/java/com/freedom/handler/ImageConvertMessageHandler.java \
  task/src/main/java/com/freedom/handler/FileReshapeMessageHandler.java \
  task/src/main/java/com/freedom/handler/FileMergeMessageHandler.java \
  task/src/main/java/com/freedom/model/stage/UserDrawStage.java \
  task/src/main/java/com/freedom/model/stage/GenerateSampleDataStage.java \
  task/src/main/java/com/freedom/model/stage/CommonTaskStage.java \
  task/src/test/java/com/freedom/invokeR/RTest.java \
  task/src/test/java/com/freedom/handler/MergeImageMessageHandlerTest.java \
  task/src/test/java/com/freedom/handler/FileReshapeMessageHandlerTest.java \
  task/src/test/java/com/freedom/handler/ImageConvertMessageHandlerTest.java \
  task/src/test/java/com/freedom/handler/FileMergeMessageHandlerTest.java
git commit -m "refactor: remove task R source loading"
```

**Self Review Before Final Report:**

- [ ] Run `rg -n 'source\\s*\\(' task/src/main/java` and confirm no Java production source generation remains in task.
- [ ] Confirm generated tool functions use `gotoTools::`.
- [ ] Confirm generated data-process functions use `gotoStat::`.
- [ ] Confirm no queue, stream, or ACK behavior changed.

## Task 3: System Module Download And Data Process Source Cleanup

**task_id:** `task-003-system-module-r-source-removal`

**Execution mode:** `DeepSeek implementer + controller重点复核`

**Exact `--cd`:** `/mnt/f/IdeaProjects/goto-software/.worktrees/goto-r-package-namespace-migration`

**Goal:** Remove system-module source references from download and data-process task publication paths and migrate download image generation to `gotoPlot::GGSAVE`.

**Why This Task Exists:** The system module publishes data-process tasks and has a direct Rserve command for download image conversion. Both must stop depending on source files.

**Editable Files:**

```text
system/src/main/java/com/freedom/rest/DataProcessController.java
system/src/main/java/com/freedom/model/file/download/impl/DrawPlotDownload.java
system/src/test/java/com/freedom/model/file/download/impl/DrawPlotDownloadTest.java
system/src/test/java/com/freedom/rest/DataProcessControllerNamespaceTest.java
```

**Forbidden Changes:**

- Do not modify dirty file `system/src/main/java/com/freedom/service/impl/PlotTypeServiceImpl.java`.
- Do not change REST endpoint paths, request/response structures, Redis Stream publishing semantics, or file path resolution behavior.
- Do not execute database writes through a database client.
- Do not require R packages to be installed for tests.

**Existing Context:**

- `DataProcessController` calls `executeRCode(..., "Stat/data_process.R", ...)` for steps 1-6 and stores source on `DataProcessTask`.
- `DrawPlotDownload` currently calls `rCmdParamCreator.addSource(..., fileProcess)` and then adds `GGSAVE(...)`.
- Existing `DrawPlotDownloadTest` focuses on temp path behavior and only covers SVG download; a new or extended test should cover non-SVG command generation without invoking real Rserve.

**Required Implementation Steps:**

- [ ] In `DataProcessController`, stop passing a real source path for data-process calls. Prefer changing the private `executeRCode` signature to remove `source`; otherwise pass `null` and do not set `task.setSource(...)`.
- [ ] Ensure data-process method names remain unchanged when published; `DataProcessBaseHandler` from Task 2 performs `gotoStat` wrapping.
- [ ] In `DrawPlotDownload`, remove `addSource(...)` and generate `gotoPlot::GGSAVE(...)`.
- [ ] Add or update tests to prove non-SVG download generation adds `gotoPlot::GGSAVE` and no source command.
- [ ] Add or update data-process controller test only if feasible with existing test patterns; otherwise rely on compile plus code search and report why a focused controller unit was not practical.

**Required Tests:**

```bash
mvn -pl system -Dtest=DrawPlotDownloadTest,DataProcessControllerNamespaceTest test
```

If `DataProcessControllerNamespaceTest` is not created because existing controller dependencies make it impractical, run:

```bash
mvn -pl system -Dtest=DrawPlotDownloadTest test
```

and report the controller coverage gap for controller review.

**Commit Command:**

```bash
git add system/src/main/java/com/freedom/rest/DataProcessController.java \
  system/src/main/java/com/freedom/model/file/download/impl/DrawPlotDownload.java \
  system/src/test/java/com/freedom/model/file/download/impl/DrawPlotDownloadTest.java \
  system/src/test/java/com/freedom/rest/DataProcessControllerNamespaceTest.java
git commit -m "refactor: remove system R source loading"
```

If `DataProcessControllerNamespaceTest.java` is not created, omit it from `git add` and state why in the final report.

**Self Review Before Final Report:**

- [ ] Run `rg -n 'source\\s*\\(' system/src/main/java` and confirm no Java production source generation remains in system.
- [ ] Confirm `gotoPlot::GGSAVE` is used exactly where non-SVG download conversion happens.
- [ ] Confirm data-process tasks still publish the same method values and params.
- [ ] Confirm no REST contracts changed.

## Task 4: Full Verification And Repository-Wide Source Audit

**task_id:** `task-004-final-verification`

**Execution mode:** `controller inline`

**Exact `--cd`:** `/mnt/f/IdeaProjects/goto-software/.worktrees/goto-r-package-namespace-migration`

**Goal:** Verify the complete migration, review all commits and diffs, and write the final handoff package.

**Editable Files:**

```text
final-review-handoff.md
```

**Forbidden Changes:**

- Do not change production or test source in this task unless controller review explicitly creates a fix prompt.
- Do not run `mvn deploy`.
- Do not run database-client write operations.

**Required Steps:**

- [ ] Run `git status --short`.
- [ ] Run `git log --oneline -5`.
- [ ] Review each task commit and `git diff master...HEAD` or the appropriate base branch diff.
- [ ] Run module tests:

```bash
mvn -pl common,task,system test
```

- [ ] Run source audit:

```bash
rg -n 'source\\s*\\(' common/src/main/java task/src/main/java system/src/main/java
```

Expected result: no Java production code that generates R `source(...)`. Mentions in comments or unrelated Java annotations must be reviewed and classified.

- [ ] Run naked function audit:

```bash
rg -P -n 'GT_MergeImages\\(|(?<!::)fileConvert\\(|(?<!::)reshapeData\\(|(?<!::)mergeTableByColumn\\(|(?<!::)GGSAVE\\(' common/src/main/java task/src/main/java system/src/main/java
```

Expected result: no legacy naked calls for migrated tool/download functions.

- [ ] Run statistics audit:

```bash
rg -P -n '(?<!::)GT_BasicStat\\(|(?<!::)GT_DifferentialStat\\(|(?<!::)GT_Corrlation\\(|(?<!::)GT_Ropls\\(' common/src/main/java task/src/main/java system/src/main/java
```

Expected result: no naked statistics calls.

- [ ] Run package namespace positive audit:

```bash
rg -n 'gotoTools::|gotoPlot::|gotoStat::' common/src/main/java task/src/main/java system/src/main/java common/src/test/java task/src/test/java system/src/test/java
```

Expected result: migrated code and tests reference the intended namespaces.

- [ ] Use the handoff command from the final review policy section or manually write `final-review-handoff.md` with equivalent data.
- [ ] Perform controller final review: spec compliance first, code quality second.

**Commit Command:**

```bash
git add final-review-handoff.md
git commit -m "docs: add R namespace migration handoff"
```

**Completion Criteria:**

- All spec requirements are mapped to implemented code or explicitly documented concerns.
- Tests pass or failures are documented with external dependency cause.
- Source and naked function audits pass.
- `final-review-handoff.md` exists and records final review status `final-review-completed`.

## Hybrid readiness checklist

- [ ] Every approved spec requirement maps to a task.
- [ ] Every DeepSeek task has stable ASCII `task_id`.
- [ ] Every DeepSeek task has exact `--cd`, editable files, forbidden changes, context, implementation steps, tests, commit command, self-review, and report requirements.
- [ ] Task 0 verifies worktree, branch, status, and `codex-dp-exec doctor`.
- [ ] Controller review gates require spec compliance before code quality review.
- [ ] Recovery rules cover interrupted, blocked, and resumed runs.
- [ ] Final verification writes `final-review-handoff.md`.
- [ ] No task requires database writes, `mvn deploy`, or R package execution.
- [ ] Existing dirty files are named and protected from unrelated edits.
