---
document_type: change_set_completion
change_set_id: CS-20260910-0a209861-7c98-4119-8227-40790dabbef6
related_change_set_ids:
  - CS-20260909-8db76639-0c29-4ef0-a223-5850749acbf0
outcome: completed
closed_at: 2026-09-11T10:03:08+08:00
---

# byDisabled 控制分类映射列选择完成总结

## 结果摘要

变更集已完成并本地合并到 `goto-web/master`。热图根据行、列属性动态生成颜色映射，以及属性类型变化后重建颜色映射时，均显式设置 `byDisabled=false`；用户可以打开映射弹窗、选择 by 列，并在 `limitType=false` 时切换离散/连续映射，类型不适配只显示警告。真实上传、标记、保存、弹窗交互、by 列切换、映射模式切换、恢复和文件清理链路已通过 Playwright 验收。

## 原因与目标

此前热图列属性生成注释条后，动态颜色映射被设置为 `byDisabled=true`。通用组件虽然已把弹窗入口与列选择权限拆开，但热图当前业务要求是这些动态行、列注释条本身允许选择 by，因此目标值应为 `false`。

验收标准包括：动态行、列注释条初次生成和类型切换重建后均保持 `byDisabled=false`；弹窗可打开；Time/Gender 等 by 列可切换；离散/连续映射可切换并保留不适配警告；真实 metadata 上传链路无回归。

## 设计决策

- 保留统一的弹窗入口，不再按 `byDisabled` 分裂为不可点击分支。
- 在 BY 展示组件、工具栏按钮、列卡可访问性状态和全部列状态变更方法上共同落实 `byDisabled`，形成界面与方法双层保护。
- 映射按钮继续只绑定 `limitType`，不改变既有连续能力判断和不适配警告规则。
- 弹窗打开时从状态位初始化当前列；取消选择后同步清除缓存，确认时再次校验状态位，避免提交失效的旧列。
- 热图动态行、列注释条在创建和类型变化重建两个入口统一写入 `byDisabled=false`；没有修改其他图组和显式配置为 `true` 时的通用限制语义。
- 未采用只在界面点击时绕过禁用态的方案，因为动态配置仍会携带错误状态，并在后续重建或其他消费位置再次触发禁用。

## 实际变更

`goto-web` 修改了分类映射入口、BY 展示组件、弹窗参数同步和列选择弹窗：

- `classifyItem.vue`：所有 by 配置统一允许打开弹窗，并传递禁用状态。
- `by.vue`：接收禁用属性，禁用时隐藏重置入口。
- `classifyItemMixins.js`：向弹窗参数传递并同步 `byDisabled`。
- `classifyChooseHeaderInfo.vue`：禁用列卡和六个工具栏按钮，保护七个列操作入口，初始化与校验当前选择，并修复取消后的旧缓存提交问题。
- 新增、扩展两个单元测试文件，覆盖入口、参数、禁用行为、映射规则和状态回归。
- `heatmapModify.js`：动态生成行/列注释颜色映射、属性类型变化后重建映射时，统一设置 `byDisabled=false`。
- `heatmapModify.spec.js`：新增动态生成与行/列重建三类回归断言，防止配置重新退回禁用态。

未修改后端、数据库模板、R 包、R 服务、接口协议或持久化结构。

## 验证结果

- 定向单元测试：本次返修先验证失败用例，再修复通过；最终在 `goto-web/master` 执行热图配置、分类映射弹窗和列信息三个套件，3 个套件、73 项全部通过。
- ESLint：`heatmapModify.js` 与 `heatmapModify.spec.js` 通过；`git diff HEAD^ --check` 通过。
- Git 检查：`git diff --check` 通过；最终独立代码审查结论为 `Ready to merge: Yes`，Critical/Important 均为 0。
- 本地启动：`goto/system` 使用 Java 17、`home` profile、`env=local` 在动态端口成功启动，验收后通过 `Ctrl-C` 正常关闭。
- Playwright：使用动态 system/web 端口 `44432/41870`、独立受控服务且未复用既有服务，上传 `/mnt/c/Users/Administrator/Desktop/metadata.xlsx`，完成 ID、Time、Gender、Clin_1 标记和保存；验证 Time 映射状态为 `byDisabled=false`、弹窗可打开、Time/Gender 列卡可用、by 可在 Gender/Time 间切换、连续映射可切换并显示类型警告、可切回离散映射；随后删除本次上传。最终结果 1/1 通过，耗时约 9 秒。
- 环境恢复：临时登录页恢复后的 SHA-256 为 `b221075bbcd54d4a993f0e372c356a45636fa9af7a823fdcef0d2a006dfaa8f7`，与替换前一致；动态端口均已释放，主工作区干净。

## 偏差、风险与遗留

- 本次返修源于需求语义再次确认：热图动态注释颜色映射应直接设置 `byDisabled=false`。此前已完成的通用 `byDisabled=true` 防护仍保留，当前只调整热图动态配置来源。
- Playwright 首次运行因 WSL Java 加载 `home` 的 Windows 存储路径导致上传/文件读取失败；关闭并恢复首轮环境后，使用新动态端口和显式 WSL Java 路径覆盖重启。随后一次运行完成全部核心交互，但最后文本断言同时命中两个节点；收紧临时测试定位器后完整重跑通过。各轮上传均由 `finally` 清理。
- 代码审查指出一个非阻断测试细节：参数化禁用测试的共用样本不是数值列，单独证明数值批量入口早退的力度有限；生产实现的七个方法保护与六个按钮绑定均已独立核验，未作为当前交付阻断项。
- Windows Git 拉取远端时 SSH 连接被中止；本地主分支与功能分支基线完全一致，因此完成了本地快进合并，但本次变更尚未推送远端。
- 前端启动仍输出仓库既有 PostCSS、Markdown loader 和 Browserslist 警告，不属于本变更新增问题。

## 关联资料

- 设计：`goto-docs/superpowers/goto-web/specs/2026-09-11-分类映射弹窗按byDisabled控制列选择设计.md`
- 计划：`goto-docs/superpowers/goto-web/plans/2026-09-11-byDisabled控制分类映射列选择实施计划.md`
- 实施与测试报告：`/tmp/CS-20260910-task-1-report.md`、`/tmp/CS-20260910-task-2-report.md`、`/tmp/CS-20260910-task-3-branch-verification-report.md`、`/tmp/CS-20260910-lint-fix-report.md`、`/tmp/CS-20260910-state-fix-report.md`
- Playwright 验收附件：`/tmp/by-disabled-e2e-rerun.0pP2H7`、`/tmp/goto-heatmap-by-enabled-node20.DpHp3z`、`/tmp/goto-heatmap-by-enabled-wslpaths.H40B06`
- `goto-web` 提交：`55bee7f`、`d8a8628`、`1ce58d5`、`e9a8398`、`0170a79`
