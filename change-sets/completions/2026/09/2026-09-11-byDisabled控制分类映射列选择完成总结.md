---
document_type: change_set_completion
change_set_id: CS-20260910-0a209861-7c98-4119-8227-40790dabbef6
related_change_set_ids:
  - CS-20260909-8db76639-0c29-4ef0-a223-5850749acbf0
outcome: completed
closed_at: 2026-09-11T10:40:00+08:00
---

# byDisabled 控制分类映射列选择完成总结

## 结果摘要

变更集已完成并本地提交到 `goto-web/master`。按用户最终确认，热图动态行、列注释颜色映射设置 `byDisabled=true`：允许打开弹窗，不允许改变 by，仍可在 `limitType=false` 时切换离散/连续映射，类型不适配仅警告。新生成的数值列（包括数字字符串）默认连续映射，文本列默认离散映射。真实上传、标记、保存、固定 by、切换映射和清理链路已通过 Playwright 验收。

## 原因与目标

原先弹窗入口与 by 禁用态耦合，导致固定来源列时无法切换映射。同会话曾按中间反馈将动态配置改为 `false`；用户最终明确要求来源列固定、弹窗与模式切换可用。另外，动态生成逻辑硬编码离散类型，并非本次数值数据必然被后端误判为文本。

验收标准包括：新建、重建及加载已有行列注释映射均固定 by；弹窗可打开，列卡和清空不可更改来源；模式可切换且保留警告；Clin_1 默认连续渐变，Time/Gender 默认离散；保存模式后再次打开不丢失来源或用户设置。

## 设计决策

- 保留统一的弹窗入口，不再按 `byDisabled` 分裂为不可点击分支。
- 在 BY 展示组件、工具栏按钮、列卡可访问性状态和全部列状态变更方法上共同落实 `byDisabled`，形成界面与方法双层保护。
- 映射按钮继续只绑定 `limitType`，不改变既有连续能力判断和不适配警告规则。
- 弹窗打开时从状态位初始化当前列；取消选择后同步清除缓存，确认时再次校验状态位，避免提交失效的旧列。
- 热图动态行、列注释条创建和重建统一写入 `byDisabled=true`；加载已有配置时仅归一化锁定标志，保留用户选择的模式和颜色。重建离散映射保留来源文件索引。
- 数值列优先识别 `numberColumn`，否则检查非空值是否全部能转为有限数值，兼容数字字符串和科学计数法；连续映射复用现有双端渐变和范围更新方法，不新增公共协议。
- 仅新生成映射使用数值默认模式，不强制把历史保存的离散模式改成连续，以免覆盖用户选择。

## 实际变更

`goto-web` 修改了分类映射入口、BY 展示组件、弹窗参数同步和列选择弹窗：

- `classifyItem.vue`：所有 by 配置统一允许打开弹窗，并传递禁用状态。
- `by.vue`：接收禁用属性，禁用时隐藏重置入口。
- `classifyItemMixins.js`：向弹窗参数传递并同步 `byDisabled`。
- `classifyChooseHeaderInfo.vue`：禁用列卡和六个工具栏按钮，保护七个列操作入口，初始化与校验当前选择，并修复取消后的旧缓存提交问题。
- 新增、扩展两个单元测试文件，覆盖入口、参数、禁用行为、映射规则和状态回归。
- `heatmapModify.js`：动态生成、重建及加载行/列注释映射时固定 by；按数值能力选择初始映射模式；重建保留来源文件索引。
- `heatmapModify.spec.js`：覆盖真实数值、数字字符串、科学计数法、文本、混合值、全空值、双侧注释重建，以及已有配置锁定但保留模式和颜色。

未修改后端、数据库模板、R 包、R 服务、接口协议或持久化结构。

## 验证结果

- 定向单元测试：本次返修先验证失败用例，再修复通过；最终在 `goto-web/master` 执行热图单元/集成、分类映射入口及弹窗四个套件，4 个套件、83 项全部通过。
- ESLint：`heatmapModify.js` 与 `heatmapModify.spec.js` 通过；`git diff HEAD^ --check` 通过。
- Git 检查：`git diff --check` 通过。此前通用组件阶段独立审查通过；本轮数值默认和固定来源返修未另行委派审查。
- 本地启动：`goto/system` 使用 Java 17、`home` profile、`env=local` 在动态端口成功启动，验收后通过 `Ctrl-C` 正常关闭。
- Playwright：使用动态 system/web 端口 `42194/44094`、独立受控服务且未复用既有服务，上传 `/mnt/c/Users/Administrator/Desktop/metadata.xlsx`，标记 ID 为 X，Time/Gender/Clin_1 为属性并保存。验证 Time 的 `byDisabled=true`，弹窗可打开，列卡和清空禁用，强制点击其他列也不能改变 by；确认切换连续时出现警告，再确认切回离散。验证 Clin_1 默认连续且只有两个渐变端点，可确认切换离散再切回连续，来源列和文件索引不变。页面异常为空；最后删除本次上传并确认状态清空。完整用例 1/1 通过，总耗时 13.2 秒。
- 环境恢复：临时登录页恢复后的 SHA-256 为 `b221075bbcd54d4a993f0e372c356a45636fa9af7a823fdcef0d2a006dfaa8f7`，与替换前一致；动态端口均已释放，主工作区干净。

## 偏差、风险与遗留

- 本次为同会话需求细化，最终规则取代中间版本的 `byDisabled=false`。通用组件的弹窗与列选择权限分离保持不变；仅修改前端热图逻辑和测试。已有保存的离散模式不会自动改写，重新保存标记生成配置或手动切换后可使用连续模式。
- 早期验收的 WSL 存储路径问题已通过进程级路径覆盖解决。本轮两次临时测试分别遇到宽泛定位器匹配弹窗隐藏选项、关闭动画与清理按钮的时序冲突；已收紧定位器并等待弹窗关闭，第二轮清理由独立临时用例完成，随后完整重跑通过。没有因测试脚本问题追加生产代码改动。
- 代码审查指出一个非阻断测试细节：参数化禁用测试的共用样本不是数值列，单独证明数值批量入口早退的力度有限；生产实现的七个方法保护与六个按钮绑定均已独立核验，未作为当前交付阻断项。
- 早期 Windows Git 拉取远端时 SSH 连接被中止；随后完成本地快进合并。本轮修复已本地提交，尚未推送远端。
- 前端启动仍输出仓库既有 PostCSS、Markdown loader 和 Browserslist 警告，不属于本变更新增问题。

## 关联资料

- 设计：`goto-docs/superpowers/goto-web/specs/2026-09-11-分类映射弹窗按byDisabled控制列选择设计.md`
- 计划：`goto-docs/superpowers/goto-web/plans/2026-09-11-byDisabled控制分类映射列选择实施计划.md`
- 实施与测试报告：`/tmp/CS-20260910-task-1-report.md`、`/tmp/CS-20260910-task-2-report.md`、`/tmp/CS-20260910-task-3-branch-verification-report.md`、`/tmp/CS-20260910-lint-fix-report.md`、`/tmp/CS-20260910-state-fix-report.md`
- 最新 Playwright 验收附件：`/tmp/goto-heatmap-fixed-by.dAS6VX`（临时测试、trace、截图及清理用例）。此前阶段附件：`/tmp/by-disabled-e2e-rerun.0pP2H7`、`/tmp/goto-heatmap-by-enabled-node20.DpHp3z`、`/tmp/goto-heatmap-by-enabled-wslpaths.H40B06`。
- `goto-web` 提交：`55bee7f`、`d8a8628`、`1ce58d5`、`e9a8398`、`0170a79`、`de61a596f5e23015122b4ae085e557e34bf17c4c`（最终固定来源与数值默认规则）。
