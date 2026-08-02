# goto-web Baseline Unit Test Repair Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 修复 `goto-web` 当前 4 个基线失败套件，恢复完整前端单元测试全绿，并补齐绘图组可见性过滤运行链路。

**Architecture:** 保留当前 UI 和权限实现，只把源码字符串测试对齐到稳定语义；分类图例恢复正确的局部布局选择器。绘图组过滤实现为独立纯函数，由 `PlotMenu` 在菜单构建边界调用，并分别用纯函数测试和接入测试保护。

**Tech Stack:** Vue 2、JavaScript、Jest、SCSS、ESLint

## Global Constraints

- 不修改后端接口或 `group_name_list` 响应结构。
- 不回退 `PlotSidePanel` 当前 accordion 与自适应网格布局。
- 不改变管理入口权限集合。
- `show === 0` 的绘图组隐藏；缺少 `show` 字段的历史数据继续显示。
- 基线修复必须作为独立提交保留，不改写已有堆叠柱图提交。

---

### Task 1: 稳定三项源码回归断言

**Files:**
- Modify: `tests/unit/styles/drawRegressionFixes.spec.js:28-35,111-117`
- Modify: `tests/unit/views/goto/manage/navigation.spec.js:36-46`

**Interfaces:**
- Consumes: 当前 `PlotSidePanel.vue` 的 `.plot-upload-item` 网格项、`imageConvert.vue` 的铅笔 prepend 图标、`index.vue` 的 `canViewManage` 权限表达式。
- Produces: 不依赖旧布局实现、标签闭合形式或单双引号格式的源码回归断言。

- [ ] **Step 1: 记录现有 RED 基线**

Run:

```bash
node scripts/run-unit-tests.js --runInBand \
  tests/unit/styles/drawRegressionFixes.spec.js \
  tests/unit/views/goto/manage/navigation.spec.js
```

Expected: FAIL；失败点分别是旧 `.file-settings` flex 选择器、自闭合 `<i />` 未匹配，以及权限表达式使用单引号。

- [ ] **Step 2: 将 PlotSidePanel 断言对齐当前完整宽度链**

Replace the four assertions in the upload-card test with:

```js
expect(source).toMatch(/\.plot-upload-item\s*{[\s\S]*display:\s*block !important;[\s\S]*width:\s*100% !important;[\s\S]*min-width:\s*0;/)
expect(source).toMatch(/> \.container\s*{[\s\S]*width:\s*100% !important;[\s\S]*min-width:\s*0;[\s\S]*display:\s*block !important;/)
expect(source).toMatch(/::v-deep \.el-upload-container\s*{[\s\S]*display:\s*block !important;[\s\S]*width:\s*100% !important;[\s\S]*min-width:\s*0;/)
expect(source).toMatch(/::v-deep \.uploader\s*{[\s\S]*display:\s*block !important;[\s\S]*width:\s*100% !important;/)
```

Rename the test to `PlotSidePanel keeps upload cards on the current full-width grid chain`.

- [ ] **Step 3: 让图标断言兼容两种合法闭合形式**

Replace the icon assertion with:

```js
expect(source).toMatch(/<template slot="prepend">\s*<i class="el-icon-edit"(?:\s*\/\>|\s*>\s*<\/i>)\s*<\/template>/)
```

- [ ] **Step 4: 让权限断言忽略引号风格**

Replace the two role assertions with:

```js
expect(source).toMatch(/roles\.includes\(['"]admin['"]\)/)
expect(source).toMatch(
  /roles\.includes\(['"]userPlotTypeStat:popular_plot_types['"]\)/
)
```

- [ ] **Step 5: 验证修正后的测试**

Run the Step 1 command.

Expected: `drawRegressionFixes.spec.js` and `navigation.spec.js` PASS。

- [ ] **Step 6: 提交测试修复**

```bash
git add tests/unit/styles/drawRegressionFixes.spec.js tests/unit/views/goto/manage/navigation.spec.js
git commit -m "test: 稳定前端源码回归断言"
```

---

### Task 2: 恢复分类图例列表布局作用域

**Files:**
- Modify: `src/views/goto/components/form/draw/classifyLegendParam.vue:362-369`
- Test: `tests/unit/styles/classifyLegendParamTheme.spec.js`

**Interfaces:**
- Consumes: 模板中的 `<div class="form-content list-param">`。
- Produces: `.list-param` 纵向 flex 布局，不再覆盖外层 `.form-group`。

- [ ] **Step 1: 验证现有测试按预期失败**

Run:

```bash
node scripts/run-unit-tests.js --runInBand tests/unit/styles/classifyLegendParamTheme.spec.js
```

Expected: FAIL at `allows legend rows to wrap when available width is too small`，提示找不到 `.list-param` 的 flex 规则。

- [ ] **Step 2: 实现最小选择器修复**

Change only the selector:

```scss
.list-param {
  display: flex !important;
  flex-direction: column !important;
  align-items: stretch;
  gap: 10px;
}
```

- [ ] **Step 3: 验证 GREEN**

Run the Step 1 command.

Expected: PASS，4 tests passed。

- [ ] **Step 4: 提交生产修复**

```bash
git add src/views/goto/components/form/draw/classifyLegendParam.vue
git commit -m "fix: 恢复分类图例列表布局作用域"
```

---

### Task 3: 补齐绘图组可见性过滤及菜单接入

**Files:**
- Create: `src/views/goto/components/plotMenu/plotGroupVisibility.js`
- Modify: `src/views/goto/components/PlotMenu.vue:23-57`
- Modify: `tests/unit/views/goto/components/plotGroupVisibility.spec.js`
- Modify: `tests/unit/views/goto/components/plotMenuBuildMenuList.spec.js`

**Interfaces:**
- Consumes: `groupNameList()` 返回的数组。
- Produces: `filterVisiblePlotGroups(groupList: unknown): Array`；返回新数组，仅排除 `show === 0` 条目。

- [ ] **Step 1: 扩展纯函数失败测试**

Keep the existing compatibility test and add:

```js
test("returns an empty list for non-array input", () => {
  expect(filterVisiblePlotGroups(null)).toEqual([])
  expect(filterVisiblePlotGroups({ show: 1 })).toEqual([])
})

test("does not mutate the response list", () => {
  const groupList = [
    { id: 1, show: 1 },
    { id: 2, show: 0 }
  ]

  const result = filterVisiblePlotGroups(groupList)

  expect(result).not.toBe(groupList)
  expect(groupList).toHaveLength(2)
})
```

- [ ] **Step 2: 添加菜单接入失败测试**

At the top of `plotMenuBuildMenuList.spec.js`, import `fs` and `path`, then add:

```js
test("PlotMenu filters hidden groups before building menu items", () => {
  const source = fs.readFileSync(
    path.resolve(
      __dirname,
      "../../../../../src/views/goto/components/PlotMenu.vue"
    ),
    "utf8"
  )

  expect(source).toMatch(
    /import \{ filterVisiblePlotGroups \} from ["']@\/views\/goto\/components\/plotMenu\/plotGroupVisibility["']/
  )
  expect(source).toMatch(
    /this\.groupList\s*=\s*filterVisiblePlotGroups\(res\.data\)/
  )
})
```

- [ ] **Step 3: 验证 RED**

Run:

```bash
node scripts/run-unit-tests.js --runInBand \
  tests/unit/views/goto/components/plotGroupVisibility.spec.js \
  tests/unit/views/goto/components/plotMenuBuildMenuList.spec.js
```

Expected: FAIL because `plotGroupVisibility` module does not exist and `PlotMenu` does not import or call it。

- [ ] **Step 4: 实现纯过滤函数**

Create `plotGroupVisibility.js`:

```js
export function filterVisiblePlotGroups(groupList) {
  if (!Array.isArray(groupList)) {
    return []
  }

  return groupList.filter(group => group.show !== 0)
}

export default filterVisiblePlotGroups
```

- [ ] **Step 5: 在 PlotMenu 数据边界接入过滤**

Add the import:

```js
import { filterVisiblePlotGroups } from "@/views/goto/components/plotMenu/plotGroupVisibility"
```

Replace the response assignment with:

```js
this.groupList = filterVisiblePlotGroups(res.data)
```

Leave `buildPlotMenuList`, icon fallback, active item, and emitted event behavior unchanged.

- [ ] **Step 6: 验证 GREEN**

Run the Step 3 command.

Expected: both suites PASS，pure function compatibility, immutability, and runtime wiring are covered。

- [ ] **Step 7: 提交可见性修复**

```bash
git add \
  src/views/goto/components/plotMenu/plotGroupVisibility.js \
  src/views/goto/components/PlotMenu.vue \
  tests/unit/views/goto/components/plotGroupVisibility.spec.js \
  tests/unit/views/goto/components/plotMenuBuildMenuList.spec.js
git commit -m "fix: 过滤隐藏的绘图分组"
```

---

### Task 4: 完整验证与交付

**Files:**
- Verify only: all files changed since `74d4d85`

**Interfaces:**
- Consumes: Tasks 1-3 的三个独立提交。
- Produces: 完整测试、静态检查和洁净工作树证据。

- [ ] **Step 1: 运行原失败套件和相关菜单测试**

```bash
node scripts/run-unit-tests.js --runInBand \
  tests/unit/styles/drawRegressionFixes.spec.js \
  tests/unit/styles/classifyLegendParamTheme.spec.js \
  tests/unit/views/goto/manage/navigation.spec.js \
  tests/unit/views/goto/components/plotGroupVisibility.spec.js \
  tests/unit/views/goto/components/plotMenuBuildMenuList.spec.js
```

Expected: all listed suites PASS。

- [ ] **Step 2: 对全部新增和改动文件执行 ESLint**

```bash
npx eslint \
  src/views/goto/components/plotMenu/plotGroupVisibility.js \
  src/views/goto/components/PlotMenu.vue \
  src/views/goto/components/form/draw/classifyLegendParam.vue \
  tests/unit/styles/drawRegressionFixes.spec.js \
  tests/unit/styles/classifyLegendParamTheme.spec.js \
  tests/unit/views/goto/manage/navigation.spec.js \
  tests/unit/views/goto/components/plotGroupVisibility.spec.js \
  tests/unit/views/goto/components/plotMenuBuildMenuList.spec.js
```

Expected: exit 0, no lint errors。

- [ ] **Step 3: 运行完整前端单元测试**

```bash
node scripts/run-unit-tests.js --runInBand --silent
```

Expected: 62 suites PASS，431 tests PASS，0 failures；若新增测试增加计数，以 Jest 实际总数为准。

- [ ] **Step 4: 检查差异和工作树**

```bash
git diff --check 74d4d85..HEAD
git status --short --branch
git log --oneline 74d4d85..HEAD
```

Expected: no whitespace errors；工作树无未提交改动；日志包含 Tasks 1-3 的三个独立提交。
