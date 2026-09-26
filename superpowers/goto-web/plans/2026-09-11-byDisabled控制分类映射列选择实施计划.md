---
change_set_id: CS-20260910-0a209861-7c98-4119-8227-40790dabbef6
related_change_set_ids:
  - CS-20260909-8db76639-0c29-4ef0-a223-5850749acbf0
---

# byDisabled 控制分类映射列选择实施计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**目标：** 分类映射项始终能打开设置弹窗，并让 by 列的选择、清空和重置严格由 `byDisabled` 控制，同时保持映射模式只受 `limitType` 控制。

**架构：** 绘图分类项负责无条件提供弹窗入口，并把 `byDisabled` 传入弹窗；弹窗在展示层和事件层同时阻止禁用状态下的列修改，但不影响映射模式；BY 展示组件同步隐藏禁用状态下的重置入口。沿用现有 Vue 2 组件、状态位和回调，不引入新服务或协议。

**技术栈：** Vue 2.6、Element UI、Jest 23、Babel Parser、Playwright。

## 全局约束

- 只修改 `goto-web` 前端组件和对应测试，不修改后端、数据库模板或 R 服务。
- 不修改热图注释项既有的 `byDisabled=true`，也不改变 `limitType`、连续/离散识别和警告规则。
- 所有生产代码变更必须先由失败测试证明缺失行为，再写最小实现。
- Playwright 使用本次动态空闲端口、`reuseExistingServer=false`，结束后恢复登录页并关闭服务。
- Git 提交信息使用中文，并携带当前 `Change-Set-Id` 与 `Related-Change-Set-Id` trailer。

---

## 文件结构

- 新建 `tests/unit/views/goto/components/form/draw/classifyItemByDisabled.spec.js`：验证分类项弹窗入口、参数传递和 BY 重置入口。
- 修改 `src/views/goto/components/form/draw/classifyItem.vue`：无论 `byDisabled` 取值都允许打开弹窗，并向 BY 传入禁用状态。
- 修改 `src/views/goto/components/form/draw/by.vue`：禁用时隐藏会清空 by 的“重置”入口。
- 修改 `src/views/goto/components/form/classifyItemMixins.js`：把 `classifyItem.byDisabled` 传给选择弹窗。
- 修改 `tests/unit/views/goto/components/form/draw/classifyChooseHeaderInfo.spec.js`：覆盖禁用状态下的列操作和当前列初始化。
- 修改 `src/views/goto/components/form/draw/classifyChooseHeaderInfo.vue`：展示禁用状态并在事件层阻止 by 修改，映射操作保持独立。

### Task 1：保留弹窗入口并传递 byDisabled

**Files:**
- Create: `tests/unit/views/goto/components/form/draw/classifyItemByDisabled.spec.js`
- Modify: `src/views/goto/components/form/draw/classifyItem.vue:39-46`
- Modify: `src/views/goto/components/form/draw/by.vue:30-65`
- Modify: `src/views/goto/components/form/classifyItemMixins.js:91-100,274-291`

**Interfaces:**
- Consumes: `classifyItem.byDisabled: boolean`。
- Produces: `chooseHeaderInfoParams.byDisabled: boolean`；`BY.disabled: boolean`。

- [ ] **Step 1：编写失败测试**

新建测试并读取三个源文件，明确断言弹窗入口不再按 `byDisabled` 分叉、弹窗参数会接收该值、BY 禁用时不提供重置入口：

```js
/* eslint-env jest */
const fs = require('fs')
const path = require('path')

const drawDir = path.resolve(__dirname, '../../../../../../../src/views/goto/components/form/draw')
const classifyItemSource = fs.readFileSync(path.join(drawDir, 'classifyItem.vue'), 'utf8')
const bySource = fs.readFileSync(path.join(drawDir, 'by.vue'), 'utf8')
const mixinSource = fs.readFileSync(path.join(drawDir, '..', 'classifyItemMixins.js'), 'utf8')

describe('classifyItem byDisabled 交互边界', () => {
  test('byDisabled 只禁用列修改，不移除映射弹窗入口', () => {
    const byVisibleBlock = classifyItemSource.match(
      /<template v-if="classifyItem\.byVisible">([\s\S]*?)<\/template>/
    )[1]

    expect(byVisibleBlock).not.toContain('v-if="classifyItem.byDisabled"')
    expect(byVisibleBlock).toContain('@click="chooseColumn()"')
    expect(byVisibleBlock).toContain(':disabled="classifyItem.byDisabled"')
  })

  test('弹窗参数接收 classifyItem.byDisabled', () => {
    expect(mixinSource).toContain('byDisabled: false')
    expect(mixinSource).toContain(
      'this.chooseHeaderInfoParams.byDisabled = this.classifyItem.byDisabled === true'
    )
  })

  test('BY 禁用时隐藏重置入口', () => {
    expect(bySource).toContain("disabled: { type: Boolean, default: false }")
    expect(bySource).toMatch(/v-if="!disabled"[\s\S]*class="clear-by"/)
  })
})
```

- [ ] **Step 2：运行测试并确认按预期失败**

Run:

```bash
node scripts/run-unit-tests.js tests/unit/views/goto/components/form/draw/classifyItemByDisabled.spec.js --runInBand
```

Expected: FAIL；失败信息指出仍存在 `v-if="classifyItem.byDisabled"`、尚未传递 `byDisabled`，且 BY 没有 `disabled` 属性。

- [ ] **Step 3：写入最小实现**

将 `classifyItem.vue` 的两个 by 分支合并为统一入口：

```vue
<template v-if="classifyItem.byVisible">
  <div class="container" style="width: 100%" @click="chooseColumn()">
    <BY
      :top-param-settings-list="topParamSettingsList"
      :classify-item="classifyItem"
      :disabled="classifyItem.byDisabled"
      @modify="resetUpdate"
    />
  </div>
</template>
```

在 `by.vue` 增加属性，并给 `.clear-by` 增加 `v-if="!disabled"`：

```js
props: {
  disabled: { type: Boolean, default: false },
  classifyItem: { type: Object, default: function() { return null } },
  topParamSettingsList: { type: Array, default: function() { return [] } }
}
```

在 `classifyItemMixins.js` 的 `chooseHeaderInfoParams` 中增加默认值，并在 `chooseColumn()` 打开弹窗前同步：

```js
byDisabled: false,
// ...
this.chooseHeaderInfoParams.byDisabled = this.classifyItem.byDisabled === true
this.chooseHeaderInfoParams.visible = true
```

- [ ] **Step 4：运行测试并确认通过**

Run:

```bash
node scripts/run-unit-tests.js tests/unit/views/goto/components/form/draw/classifyItemByDisabled.spec.js --runInBand
```

Expected: PASS，3 tests passed。

- [ ] **Step 5：提交 Task 1**

```bash
git add src/views/goto/components/form/draw/classifyItem.vue \
  src/views/goto/components/form/draw/by.vue \
  src/views/goto/components/form/classifyItemMixins.js \
  tests/unit/views/goto/components/form/draw/classifyItemByDisabled.spec.js
git commit -m "fix: 恢复分类映射弹窗入口并传递列禁用状态" \
  -m "Change-Set-Id: CS-20260910-0a209861-7c98-4119-8227-40790dabbef6
Related-Change-Set-Id: CS-20260909-8db76639-0c29-4ef0-a223-5850749acbf0"
```

### Task 2：在弹窗内禁用 by 列修改

**Files:**
- Modify: `tests/unit/views/goto/components/form/draw/classifyChooseHeaderInfo.spec.js`
- Modify: `src/views/goto/components/form/draw/classifyChooseHeaderInfo.vue:17-69,174-245,280-291,441-535`

**Interfaces:**
- Consumes: `params.byDisabled: boolean`、`params.limitType: boolean`。
- Produces: 禁用状态下不可变的 `selectedHeaderInfo` 和列状态；映射模式回调签名保持 `callBack(mappingType, selectedHeaderInfo)`。

- [ ] **Step 1：编写失败测试**

在既有测试中补充以下行为测试：

```js
test('byDisabled 时列点击与清空均不改变选择', () => {
  const changeStatus = jest.fn()
  const header = { columnName: 'Time', status: 1 }
  const context = {
    params: { byDisabled: true, fileSetting: { headerInfoList: [header] } },
    changeStatus,
    isChecked: () => true
  }

  component.methods.columnNameClick.call(context, header)
  component.methods.cancelAll.call(context)

  expect(changeStatus).not.toHaveBeenCalled()
})

test('byDisabled=false 时仍可选择其他 by 列', () => {
  const changeStatus = jest.fn()
  const selected = { columnName: 'Time', status: 1 }
  const target = { columnName: 'Gender', status: 0 }
  const context = {
    params: {
      byDisabled: false,
      globalUnique: false,
      multiple: false,
      limitType: true,
      fileSetting: { headerInfoList: [selected, target] }
    },
    changeStatus,
    mappingType: 1,
    selectedHeaderInfo: selected
  }

  component.methods.columnNameClick.call(context, target)

  expect(changeStatus).toHaveBeenCalledTimes(2)
  expect(context.selectedHeaderInfo).toBe(target)
})

test('打开弹窗时用当前已选 by 初始化 selectedHeaderInfo', () => {
  const selected = { columnName: 'Time', status: 1 }
  const context = {
    params: {
      initialMappingType: 1,
      fileSetting: { headerInfoList: [selected] }
    },
    mappingType: null,
    selectedHeaderInfo: null,
    isChecked: header => header === selected,
    $emit: jest.fn()
  }

  component.methods.opened.call(context)

  expect(context.mappingType).toBe(1)
  expect(context.selectedHeaderInfo).toBe(selected)
})

test('列控件受 byDisabled 控制而映射按钮仍只受 limitType 控制', () => {
  expect(source).toContain(':aria-disabled="params.byDisabled ? \'true\' : \'false\'"')
  expect(source).toContain("'disabled-by-config': params.byDisabled")
  expect(source).toContain(':disabled="params.byDisabled"')

  const mappingCheckboxGroup = source.match(
    /<div class="mapping-checkbox-group">([\s\S]*?)<\/div>/
  )[1]
  const disabledBindings = [...mappingCheckboxGroup.matchAll(/:disabled="([^"]+)"/g)]
    .map(match => match[1])
  expect(disabledBindings).toEqual(['params.limitType', 'params.limitType'])
})
```

- [ ] **Step 2：运行测试并确认按预期失败**

Run:

```bash
node scripts/run-unit-tests.js tests/unit/views/goto/components/form/draw/classifyChooseHeaderInfo.spec.js --runInBand
```

Expected: FAIL；`columnNameClick` 或 `cancelAll` 调用了 `changeStatus`，`opened()` 未初始化当前列，模板也没有 by 禁用绑定。

- [ ] **Step 3：写入最小实现**

在列卡片和所有会改变列选择的工具栏按钮上增加禁用状态：

```vue
<el-button v-if="params.multiple" :disabled="params.byDisabled" size="small" @click="checkAll">全选</el-button>
<el-button :disabled="params.byDisabled" size="small" @click="cancelAll">清空</el-button>

<div
  class="variable-card"
  :class="[getColumnNameClass(headerInfo), { 'disabled-by-config': params.byDisabled }]"
  :aria-disabled="params.byDisabled ? 'true' : 'false'"
  @click="columnNameClick(headerInfo)"
>
```

为 `columnNameClick`、`checkAll`、`checkAllNotNumber`、`checkAllNumber`、`cancelAll`、`cancelAllNotNumber`、`cancelAllNumber` 的第一行统一增加：

```js
if (this.params.byDisabled) return
```

在 `opened()` 初始化映射类型后定位当前已选列：

```js
const headerInfoList = this.params.fileSetting && this.params.fileSetting.headerInfoList
this.selectedHeaderInfo = Array.isArray(headerInfoList)
  ? headerInfoList.find(headerInfo => this.isChecked(headerInfo)) || null
  : null
```

复用既有禁用视觉样式：

```scss
.variable-card.disabled-by-other,
.variable-card.disabled-by-config {
  background-color: var(--hover-bg);
  border-color: var(--border-color);
  cursor: not-allowed;
  opacity: 0.6;
}
```

其余 `.disabled-by-other` 子规则同步改成同时覆盖 `.disabled-by-config`，不改变选中态和映射控制台样式。

- [ ] **Step 4：运行两个定向测试并确认通过**

Run:

```bash
node scripts/run-unit-tests.js \
  tests/unit/views/goto/components/form/draw/classifyItemByDisabled.spec.js \
  tests/unit/views/goto/components/form/draw/classifyChooseHeaderInfo.spec.js \
  --runInBand
```

Expected: PASS，两个测试文件全部通过。

- [ ] **Step 5：提交 Task 2**

```bash
git add src/views/goto/components/form/draw/classifyChooseHeaderInfo.vue \
  tests/unit/views/goto/components/form/draw/classifyChooseHeaderInfo.spec.js
git commit -m "fix: 按 byDisabled 禁用分类映射列操作" \
  -m "Change-Set-Id: CS-20260910-0a209861-7c98-4119-8227-40790dabbef6
Related-Change-Set-Id: CS-20260909-8db76639-0c29-4ef0-a223-5850749acbf0"
```

### Task 3：回归验证真实热图流程

**Files:**
- Verify only: `src/views/goto/components/form/draw/classifyItem.vue`
- Verify only: `src/views/goto/components/form/draw/by.vue`
- Verify only: `src/views/goto/components/form/draw/classifyChooseHeaderInfo.vue`
- Verify only: `src/views/goto/components/form/classifyItemMixins.js`

**Interfaces:**
- Consumes: Task 1 和 Task 2 的组件行为。
- Produces: 单元测试、静态检查和真实浏览器链路的验收证据。

- [ ] **Step 1：运行定向单元测试**

```bash
node scripts/run-unit-tests.js \
  tests/unit/views/goto/components/form/draw/classifyItemByDisabled.spec.js \
  tests/unit/views/goto/components/form/draw/classifyChooseHeaderInfo.spec.js \
  --runInBand
```

Expected: PASS，0 failed。

- [ ] **Step 2：运行受影响文件 ESLint**

```bash
./node_modules/.bin/eslint \
  src/views/goto/components/form/draw/classifyItem.vue \
  src/views/goto/components/form/draw/by.vue \
  src/views/goto/components/form/draw/classifyChooseHeaderInfo.vue \
  src/views/goto/components/form/classifyItemMixins.js \
  tests/unit/views/goto/components/form/draw/classifyItemByDisabled.spec.js \
  tests/unit/views/goto/components/form/draw/classifyChooseHeaderInfo.spec.js
```

Expected: exit 0；若存量文件存在与本次无关的历史警告，记录准确行号，不扩大修复范围。

- [ ] **Step 3：按项目规范启动隔离服务**

- 临时复制 `src/views/login公众号验证码登录.vue` 到 `src/views/login.vue`，记录并最终恢复原始 SHA-256。
- 为 system 和 goto-web 即时取得两个空闲端口。
- system 使用：

```bash
java -Denv=local -Dspring.profiles.active=home -jar system/target/goto-manager.jar --server.port=<SYSTEM_PORT>
```

- Playwright 的 `webServer` 使用 `<WEB_PORT>`，`reuseExistingServer=false`，API 指向 `127.0.0.1:<SYSTEM_PORT>`。

- [ ] **Step 4：运行热图 Playwright 验收**

使用 `/mnt/c/Users/Administrator/Desktop/metadata.xlsx` 执行：

1. 登录验证码填写 `666888`。
2. 打开热图测试绘图记录。
3. 在列属性上传文件，将 `ID` 标记为列，将 `Time`、`Gender`、`Clin_1` 标记为属性并保存。
4. 点击列注释条 `Time` 项，断言弹窗可见。
5. 断言当前 `Time` 列卡片为选中且 `aria-disabled=true`，点击其他列和“清空”后 by 仍为 `Time`。
6. 在 `limitType=false` 下切换连续映射，确认映射类型改变且类型不适配时只显示警告。
7. 关闭弹窗，删除测试上传文件。

Expected: 1 passed；弹窗打开，by 不可修改，映射模式可以切换，无失败业务请求和页面异常。

- [ ] **Step 5：恢复环境并复核仓库**

- 从项目外临时备份恢复 `src/views/login.vue`，确认 SHA-256 与替换前一致。
- 通过 Ctrl-C 关闭本次 system 和 Playwright 管理的 goto-web，确认两个端口均释放。
- 确认 `git diff -- src/views/login.vue` 和 `git diff --cached -- src/views/login.vue` 均为空。
- 执行 `git status --short`，只允许存在本计划内的预期状态。

- [ ] **Step 6：记录最终验证提交（仅当验收脚本纳入仓库时）**

若最终新增了可复用的仓库内 Playwright 用例，则提交：

```bash
git add <实际新增的Playwright用例路径>
git commit -m "test: 补充分类映射列禁用端到端回归" \
  -m "Change-Set-Id: CS-20260910-0a209861-7c98-4119-8227-40790dabbef6
Related-Change-Set-Id: CS-20260909-8db76639-0c29-4ef0-a223-5850749acbf0"
```

若 Playwright 用例仅位于项目外临时目录，则不创建空提交，以测试命令、退出码和临时产物路径作为验收证据。
