---
document_type: change_set_completion
change_set_id: CS-20260910-0a209861-7c98-4119-8227-40790dabbef6
related_change_set_ids:
  - CS-20260909-8db76639-0c29-4ef0-a223-5850749acbf0
outcome: completed
closed_at: 2026-09-11T09:08:53+08:00
---

# byDisabled 控制分类映射列选择完成总结

## 结果摘要

变更集已完成并本地合并到 `goto-web/master`。热图列注释条的分类映射卡片在 `byDisabled=true` 时仍可打开映射弹窗；当前 by 列和所有列选择操作保持禁用，但 `limitType=false` 时离散/连续映射仍可切换，类型不适配只显示警告。真实上传、标记、保存、弹窗交互、映射切换、恢复和文件清理链路已通过 Playwright 验收。

## 原因与目标

此前热图列属性生成注释条后，`byDisabled=true` 同时阻断了整个分类映射弹窗入口，导致用户无法进入弹窗切换映射模式。目标是把两类权限拆开：`byDisabled` 只决定 by 列能否选择，`limitType` 独立决定映射模式能否切换，并保证禁用列选择时仍能查看和确认当前列。

验收标准包括：弹窗可打开；禁用态不能选择、清空或重置 by；映射模式不受 `byDisabled` 阻断；清空后不得提交旧缓存列；真实 metadata 上传链路无回归。

## 设计决策

- 保留统一的弹窗入口，不再按 `byDisabled` 分裂为不可点击分支。
- 在 BY 展示组件、工具栏按钮、列卡可访问性状态和全部列状态变更方法上共同落实 `byDisabled`，形成界面与方法双层保护。
- 映射按钮继续只绑定 `limitType`，不改变既有连续能力判断和不适配警告规则。
- 弹窗打开时从状态位初始化当前列；取消选择后同步清除缓存，确认时再次校验状态位，避免提交失效的旧列。
- 未采用隐藏弹窗、隐藏列卡或将 `byDisabled` 复用为映射模式限制的方案，因为这些方案会重新耦合两项独立业务规则。

## 实际变更

`goto-web` 修改了分类映射入口、BY 展示组件、弹窗参数同步和列选择弹窗：

- `classifyItem.vue`：所有 by 配置统一允许打开弹窗，并传递禁用状态。
- `by.vue`：接收禁用属性，禁用时隐藏重置入口。
- `classifyItemMixins.js`：向弹窗参数传递并同步 `byDisabled`。
- `classifyChooseHeaderInfo.vue`：禁用列卡和六个工具栏按钮，保护七个列操作入口，初始化与校验当前选择，并修复取消后的旧缓存提交问题。
- 新增、扩展两个单元测试文件，覆盖入口、参数、禁用行为、映射规则和状态回归。

未修改后端、数据库模板、R 包、R 服务、接口协议或持久化结构。

## 验证结果

- 定向单元测试：合并前 2 个套件、27 项全部通过；本地合并后再次执行，主工作区和尚未清理的工作树被 Jest 同时发现，共 4 个套件、54 项全部通过。
- ESLint：返修涉及文件通过；全变更相对基线新增 ESLint 问题为 0，仓库基线已有 170 项历史问题未纳入本次范围。
- Git 检查：`git diff --check` 通过；最终独立代码审查结论为 `Ready to merge: Yes`，Critical/Important 均为 0。
- 本地启动：`goto/system` 使用 Java 17、`home` profile、`env=local` 在动态端口成功启动，验收后通过 `Ctrl-C` 正常关闭。
- Playwright：使用动态 system/web 端口 `41876/42264`、`reuseExistingServer=false`，上传 `/mnt/c/Users/Administrator/Desktop/metadata.xlsx`，完成 ID、Time、Gender、Clin_1 标记和保存；验证弹窗可打开、Time/Gender 列卡禁用态、清空按钮禁用、Gender 点击不改变 by、连续映射可切换并显示 Time 类型警告；随后恢复离散映射并删除本次上传。最终结果 1/1 通过，耗时约 1.5 分钟。
- 环境恢复：临时登录页恢复后的 SHA-256 为 `b221075bbcd54d4a993f0e372c356a45636fa9af7a823fdcef0d2a006dfaa8f7`，与替换前一致；动态端口均已释放，主工作区干净。

## 偏差、风险与遗留

- 首次 Playwright 运行在核心交互完成后因临时测试选择器命中两个同名元素而失败；修正临时断言为读取真实组件状态后，使用全新动态端口完整重跑并通过。首次运行的上传文件已由 `finally` 清理，登录页和服务也先恢复、关闭后才重跑。
- 代码审查指出一个非阻断测试细节：参数化禁用测试的共用样本不是数值列，单独证明数值批量入口早退的力度有限；生产实现的七个方法保护与六个按钮绑定均已独立核验，未作为当前交付阻断项。
- Windows Git 拉取远端时 SSH 连接被中止；本地主分支与功能分支基线完全一致，因此完成了本地快进合并，但本次变更尚未推送远端。
- 前端启动仍输出仓库既有 PostCSS、Markdown loader 和 Browserslist 警告，不属于本变更新增问题。

## 关联资料

- 设计：`goto-docs/superpowers/goto-web/specs/2026-09-11-分类映射弹窗按byDisabled控制列选择设计.md`
- 计划：`goto-docs/superpowers/goto-web/plans/2026-09-11-byDisabled控制分类映射列选择实施计划.md`
- 实施与测试报告：`/tmp/CS-20260910-task-1-report.md`、`/tmp/CS-20260910-task-2-report.md`、`/tmp/CS-20260910-task-3-branch-verification-report.md`、`/tmp/CS-20260910-lint-fix-report.md`、`/tmp/CS-20260910-state-fix-report.md`
- Playwright 验收附件：`/tmp/by-disabled-e2e-rerun.0pP2H7`
- `goto-web` 提交：`55bee7f`、`d8a8628`、`1ce58d5`、`e9a8398`
