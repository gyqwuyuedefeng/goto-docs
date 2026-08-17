# GotoSettings 管理页面视觉精修 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 在不改变两个管理页面原有功能的前提下，统一并精修其视觉，同时保持与用户端样式物理隔离。

**Architecture:** 新建 `views/gotoSettings/styles/_management-pages.scss` 作为管理端共享视觉层；两个 Vue 页面仅增加展示容器/类名并在各自 scoped style 中导入该文件。业务脚本通过哈希契约测试冻结。

**Tech Stack:** Vue 2、Element UI、SCSS、Jest、Playwright MCP。

## Global Constraints

- 不修改两个目标页面的 `<script>` 内容、接口、事件、权限、字段或数据流。
- 不导入、移动或修改 `views/goto/plot` 的用户端样式。
- 不新增依赖或全局样式覆盖。
- 所有视觉验证由自动化完成，不要求用户中途人工查看。

---

### Task 1: 建立样式隔离和功能冻结契约

**Files:**
- Create: `goto-web/tests/unit/views/gotoSettings/managementPagesStyle.spec.js`

**Interfaces:**
- Consumes: 两个目标 Vue 文件与用户端参考页源码。
- Produces: 管理端根类、共享样式导入、用户端隔离和脚本哈希契约。

- [ ] 编写测试，断言共享 SCSS 文件存在、两个页面导入它、根节点使用 `goto-settings-page`，用户端页面没有导入该文件，并锁定两个 `<script>` 的 SHA-256。
- [ ] 运行 `node scripts/run-unit-tests.js tests/unit/views/gotoSettings/managementPagesStyle.spec.js --runInBand`，确认因样式文件/类名尚不存在而失败。

### Task 2: 实现独立管理端视觉层

**Files:**
- Create: `goto-web/src/views/gotoSettings/styles/_management-pages.scss`
- Modify: `goto-web/src/views/gotoSettings/plotGroup/index.vue`
- Modify: `goto-web/src/views/gotoSettings/plotType/index.vue`

**Interfaces:**
- Consumes: 项目既有 `theme-variables.scss` 语义变量和 Element UI 组件类。
- Produces: `.goto-settings-page`、`.management-page-header`、`.management-toolbar-card`、`.management-table-card` 及响应式规则。

- [ ] 只在模板中增加页面标题、语义容器和样式类，不修改绑定、事件或字段。
- [ ] 在管理端专属 SCSS 中实现画布、标题、面板、工具栏、表格、分页、弹窗和窄屏规则。
- [ ] 保留两个页面各自特有的图片上传、排序卡片等 scoped 样式，并替换不协调的硬编码颜色为语义变量。
- [ ] 重新运行 Task 1 测试，确认通过。

### Task 3: 静态和构建回归

**Files:**
- Verify only.

**Interfaces:**
- Consumes: 完成后的页面、共享样式及项目测试配置。
- Produces: 单元测试、lint 和生产构建证据。

- [ ] 运行目标页相关单元测试。
- [ ] 运行完整 `npm run test:unit`。
- [ ] 运行 `npm run lint`；若存在历史问题，单独运行只覆盖本次文件的 ESLint 并明确区分。
- [ ] 使用 WSL 兼容的 `NODE_OPTIONS=--openssl-legacy-provider ./node_modules/.bin/vue-cli-service build` 完成生产构建。

### Task 4: Playwright MCP 实机回归

**Files:**
- Verify only; screenshots go to temporary test output and are not committed.

**Interfaces:**
- Consumes: 运行中的 goto-web 与可用后端/登录状态。
- Produces: 两页桌面/窄屏截图、布局测量、控制台和交互结果。

- [ ] 启动应用并通过 Playwright MCP 打开两个管理路由。
- [ ] 在桌面视口验证标题、工具栏、表格、按钮、筛选和主要弹窗打开/关闭。
- [ ] 在窄屏验证页面无视口级横向溢出、工具栏换行、弹窗与表格仍可用。
- [ ] 记录控制台异常并修复由本次改动导致的问题；完成后关闭应用和 MCP 浏览器进程。

### Task 5: 完成审计

**Files:**
- Verify only.

**Interfaces:**
- Consumes: 原始目标、Git diff、全部新鲜验证输出。
- Produces: 逐项可追溯的完成证据。

- [ ] 检查 Git diff 仅包含设计/计划、测试、管理端共享样式和两个目标页面模板/样式。
- [ ] 复核脚本哈希、样式隔离、两个页面视觉、响应式和关键交互全部满足验收标准。
- [ ] 确认无测试应用留在后台后再报告完成。
