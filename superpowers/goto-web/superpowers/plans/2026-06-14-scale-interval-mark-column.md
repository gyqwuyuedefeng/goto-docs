# Scale Interval Mark Column Linkage Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add configurable mark-column linkage to scale interval params and keep `start/end/interval` valid against linked mark-column numeric ranges.

**Architecture:** Add a syncable `scaleFileMarkColumnList` field to `ScaleIntervalParam`, parameterize the existing `MarkColumnSelector.vue`, and reuse it in the `moduleType === 24` form. Keep range calculation and correction in `publicFunction.js` so upload, save-mark, and input/selector changes share one implementation.

**Tech Stack:** Java + Lombok + JUnit 5 for backend model tests; Vue 2 + Element UI + Jest for frontend tests.

---

## File Structure

- Modify `goto/common/src/main/java/com/freedom/model/form/ScaleIntervalParam.java`
  - Add `List<FileMarkColumn> scaleFileMarkColumnList` with `@Syncable`.
- Create `goto/common/src/test/java/com/freedom/model/form/ScaleIntervalParamMarkColumnTest.java`
  - Guard field type and annotation.
- Modify `goto-web/src/views/goto/components/form/plotSetting/MarkColumnSelector.vue`
  - Add `fieldName`, `title`, `emptyText`, and `cleanMessage` props.
  - Replace fixed `parent.fileMarkColumnList` reads/writes with `parent[fieldName]`.
- Modify `goto-web/src/constant/modify/publicFunction.js`
  - Add range helpers for `moduleType === 24`.
  - Export `syncScaleIntervalRangeByMarkColumns`.
  - Call it from `uploadFileModify` and `saveFileMarkSuccess`.
- Modify `goto-web/src/views/goto/components/paramSettings.vue`
  - Import/register `MarkColumnSelector`.
  - Render it in the `setting.moduleType === 24` block.
  - Add input and mark-column-change handlers using light/hard validation.
- Create `goto-web/tests/unit/constant/modify/scaleIntervalRange.spec.js`
  - Test range correction and upload/save hard validation.
- Create `goto-web/tests/unit/components/MarkColumnSelector.spec.js`
  - Test custom field-name behavior.
- Create `goto-web/tests/unit/components/paramSettingsScaleInterval.spec.js`
  - Source-level guard that `moduleType === 24` renders `MarkColumnSelector` with `scaleFileMarkColumnList`.

---

### Task 1: Backend Field and Guard Test

**Files:**
- Modify: `goto/common/src/main/java/com/freedom/model/form/ScaleIntervalParam.java`
- Create: `goto/common/src/test/java/com/freedom/model/form/ScaleIntervalParamMarkColumnTest.java`

- [ ] **Step 1: Write the failing backend test**

Create `goto/common/src/test/java/com/freedom/model/form/ScaleIntervalParamMarkColumnTest.java`:

```java
package com.freedom.model.form;

import com.freedom.annotation.Syncable;
import com.freedom.model.FileMarkColumn;
import org.junit.jupiter.api.Test;

import java.lang.reflect.Field;
import java.lang.reflect.ParameterizedType;
import java.lang.reflect.Type;
import java.util.List;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNotNull;
import static org.junit.jupiter.api.Assertions.assertTrue;

class ScaleIntervalParamMarkColumnTest {

    @Test
    void scaleFileMarkColumnListIsSyncableFileMarkColumnList() throws Exception {
        Field field = ScaleIntervalParam.class.getDeclaredField("scaleFileMarkColumnList");

        assertEquals(List.class, field.getType());
        assertNotNull(field.getAnnotation(Syncable.class));

        Type genericType = field.getGenericType();
        assertTrue(genericType instanceof ParameterizedType);
        ParameterizedType parameterizedType = (ParameterizedType) genericType;
        assertEquals(FileMarkColumn.class, parameterizedType.getActualTypeArguments()[0]);
    }
}
```

- [ ] **Step 2: Run the backend test and verify it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto
mvn -pl common -Dtest=ScaleIntervalParamMarkColumnTest test
```

Expected: FAIL with `NoSuchFieldException: scaleFileMarkColumnList`.

- [ ] **Step 3: Add the field**

Modify `goto/common/src/main/java/com/freedom/model/form/ScaleIntervalParam.java`:

```java
import com.freedom.model.FileMarkColumn;

import java.util.List;
```

Add the field after `interval`:

```java
    /**
     * 刻度范围关联标记列
     */
    @Syncable
    private List<FileMarkColumn> scaleFileMarkColumnList;
```

- [ ] **Step 4: Run the backend test and verify it passes**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto
mvn -pl common -Dtest=ScaleIntervalParamMarkColumnTest test
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add goto/common/src/main/java/com/freedom/model/form/ScaleIntervalParam.java \
  goto/common/src/test/java/com/freedom/model/form/ScaleIntervalParamMarkColumnTest.java
git commit -m "feat: add scale interval mark column field"
```

---

### Task 2: Parameterize MarkColumnSelector

**Files:**
- Modify: `goto-web/src/views/goto/components/form/plotSetting/MarkColumnSelector.vue`
- Create: `goto-web/tests/unit/components/MarkColumnSelector.spec.js`

- [ ] **Step 1: Write the failing component test**

Create `goto-web/tests/unit/components/MarkColumnSelector.spec.js`:

```js
/* eslint-env jest */
import { mount, createLocalVue } from '@vue/test-utils'
import MarkColumnSelector from '@/views/goto/components/form/plotSetting/MarkColumnSelector.vue'

const localVue = createLocalVue()

function installGlobals() {
  localVue.prototype.CommonUtils = {
    isNotEmptyArray(value) {
      return Array.isArray(value) && value.length > 0
    }
  }
}

describe('MarkColumnSelector', () => {
  beforeAll(() => {
    installGlobals()
  })

  test('writes selected marks to custom fieldName', async() => {
    const parent = {}
    const wrapper = mount(MarkColumnSelector, {
      localVue,
      propsData: {
        parent,
        fieldName: 'scaleFileMarkColumnList',
        fileSettings: [
          {
            name: 'file-a',
            fileMarkContainerList: [
              { field: 'score', label: 'Score' }
            ]
          }
        ]
      },
      stubs: {
        'el-checkbox': {
          props: ['value', 'indeterminate'],
          template: '<label><input type="checkbox" :checked="value" @change="$emit(`input`, $event.target.checked)"><slot /></label>'
        },
        'el-button': true
      },
      mocks: {
        $message: {
          success: jest.fn(),
          info: jest.fn()
        }
      }
    })

    wrapper.vm.handleMarkSelect(0, 'score', true)
    await wrapper.vm.$nextTick()

    expect(parent.scaleFileMarkColumnList).toEqual([
      {
        filedName: 'score',
        fileSettingIndex: 0
      }
    ])
    expect(parent.fileMarkColumnList).toBeUndefined()
    expect(wrapper.emitted().modify).toHaveLength(1)
  })

  test('keeps fileMarkColumnList as the default fieldName', () => {
    const parent = {}
    const wrapper = mount(MarkColumnSelector, {
      localVue,
      propsData: {
        parent,
        fileSettings: [
          {
            name: 'file-a',
            fileMarkContainerList: [
              { field: 'group', label: 'Group' }
            ]
          }
        ]
      },
      stubs: {
        'el-checkbox': true,
        'el-button': true
      },
      mocks: {
        $message: {
          success: jest.fn(),
          info: jest.fn()
        }
      }
    })

    wrapper.vm.handleMarkSelect(0, 'group', true)

    expect(parent.fileMarkColumnList).toEqual([
      {
        filedName: 'group',
        fileSettingIndex: 0
      }
    ])
  })
})
```

- [ ] **Step 2: Run the component test and verify it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
node scripts/run-unit-tests.js tests/unit/components/MarkColumnSelector.spec.js
```

Expected: FAIL because `fieldName` is not defined and selection still writes `fileMarkColumnList`.

- [ ] **Step 3: Parameterize the component**

Modify `goto-web/src/views/goto/components/form/plotSetting/MarkColumnSelector.vue`.

Replace the fixed label and empty text:

```vue
<div class="selection-label">{{ title }}</div>
```

```vue
<span>{{ emptyText }}</span>
```

Add props:

```js
    fieldName: {
      type: String,
      default: 'fileMarkColumnList'
    },
    title: {
      type: String,
      default: '标记列选择'
    },
    emptyText: {
      type: String,
      default: '暂无可选择的标记列'
    },
    cleanMessage: {
      type: String,
      default: '已清理'
    }
```

Add helper methods at the top of `methods`:

```js
    getMarkColumnList() {
      return this.parent && Array.isArray(this.parent[this.fieldName])
        ? this.parent[this.fieldName]
        : []
    },
    ensureMarkColumnList() {
      if (!this.parent[this.fieldName]) {
        this.$set(this.parent, this.fieldName, [])
      }
      return this.parent[this.fieldName]
    },
```

Change `hasInvalidMarks`:

```js
    hasInvalidMarks() {
      const markColumnList = this.getMarkColumnList()
      if (!this.CommonUtils.isNotEmptyArray(markColumnList)) {
        return false
      }
      if (!this.CommonUtils.isNotEmptyArray(this.fileSettings)) {
        return markColumnList.length > 0
      }
      return markColumnList.some(mark => {
        const fileSetting = this.fileSettings[mark.fileSettingIndex]
        if (!fileSetting || !this.CommonUtils.isNotEmptyArray(fileSetting.fileMarkContainerList)) {
          return true
        }
        return !fileSetting.fileMarkContainerList.some(container => container.field === mark.filedName)
      })
    }
```

Change `isMarkSelected`:

```js
    isMarkSelected(fileIndex, field) {
      return this.getMarkColumnList().some(
        item => item.fileSettingIndex === fileIndex && item.filedName === field
      )
    },
```

Change `handleMarkSelect`:

```js
    handleMarkSelect(fileIndex, field, selected) {
      const markColumnList = this.ensureMarkColumnList()
      const index = markColumnList.findIndex(
        item => item.fileSettingIndex === fileIndex && item.filedName === field
      )
      if (selected && index === -1) {
        markColumnList.push({
          filedName: field,
          fileSettingIndex: fileIndex
        })
      } else if (!selected && index !== -1) {
        markColumnList.splice(index, 1)
      }
      this.$emit('modify')
    },
```

Change `cleanInvalidMarks`:

```js
    cleanInvalidMarks() {
      const markColumnList = this.getMarkColumnList()
      if (!this.CommonUtils.isNotEmptyArray(markColumnList)) {
        return
      }
      const beforeCount = markColumnList.length
      const validMarks = markColumnList.filter(item => {
        const fileSetting = this.fileSettings[item.fileSettingIndex]
        return fileSetting &&
          this.CommonUtils.isNotEmptyArray(fileSetting.fileMarkContainerList) &&
          fileSetting.fileMarkContainerList.some(container => container.field === item.filedName)
      })
      this.$set(this.parent, this.fieldName, validMarks)
      const cleanedCount = beforeCount - validMarks.length
      if (cleanedCount > 0) {
        this.$message.success(`${this.cleanMessage} ${cleanedCount} 个无效标记列`)
        this.$emit('modify')
      } else {
        this.$message.info('没有需要清理的无效标记列')
      }
    }
```

- [ ] **Step 4: Run the component test and existing checkbox style test**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
node scripts/run-unit-tests.js tests/unit/components/MarkColumnSelector.spec.js tests/unit/styles/checkboxTheme.spec.js
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add goto-web/src/views/goto/components/form/plotSetting/MarkColumnSelector.vue \
  goto-web/tests/unit/components/MarkColumnSelector.spec.js
git commit -m "feat: parameterize mark column selector"
```

---

### Task 3: Add Scale Interval Range Helpers

**Files:**
- Modify: `goto-web/src/constant/modify/publicFunction.js`
- Create: `goto-web/tests/unit/constant/modify/scaleIntervalRange.spec.js`

- [ ] **Step 1: Write failing range helper tests**

Create `goto-web/tests/unit/constant/modify/scaleIntervalRange.spec.js`:

```js
/* eslint-env jest */
jest.mock('@/api/plotType', () => ({
  updateSettingsByPlotGroupIdAndPlotTypeId: jest.fn()
}))

jest.mock('@/constant/modify/pieModify', () => ({
  pieDeleteFileModify: jest.fn(),
  pieUploadFileModify: jest.fn(),
  doPieModify: jest.fn(),
  pieSaveFileMarkSuccess: jest.fn(),
  pieSaveFileSuccess: jest.fn()
}))
jest.mock('@/constant/modify/vennModify', () => ({
  vennDeleteFileModify: jest.fn(),
  vennUploadFileModify: jest.fn(),
  doVennModify: jest.fn(),
  vennSaveFileMarkSuccess: jest.fn(),
  vennSaveFileSuccess: jest.fn()
}))
jest.mock('@/constant/modify/boxViolinModify', () => ({
  boxDeleteFileModify: jest.fn(),
  boxUploadFileModify: jest.fn(),
  doBoxViolinModify: jest.fn(),
  boxSaveFileMarkSuccess: jest.fn(),
  boxSaveFileSuccess: jest.fn()
}))
jest.mock('@/constant/modify/barModify', () => ({
  barDeleteFileModify: jest.fn(),
  barUploadFileModify: jest.fn(),
  doBarModify: jest.fn(),
  barSaveFileMarkSuccess: jest.fn(),
  barSaveFileSuccess: jest.fn()
}))
jest.mock('@/constant/modify/curveModify', () => ({
  curveDeleteFileModify: jest.fn(),
  curveUploadFileModify: jest.fn(),
  doCurveModify: jest.fn(),
  curveSaveFileMarkSuccess: jest.fn(),
  curveSaveFileSuccess: jest.fn()
}))
jest.mock('@/constant/modify/heatmapModify', () => ({
  heatmapDeleteFileModify: jest.fn(),
  heatmapUploadFileModify: jest.fn(),
  doHeatmapModify: jest.fn(),
  heatmapSaveFileMarkSuccess: jest.fn(),
  heatmapSaveFileSuccess: jest.fn()
}))
jest.mock('@/constant/modify/scatterModify', () => ({
  scatterDeleteFileModify: jest.fn(),
  scatterUploadFileModify: jest.fn(),
  doScatterModify: jest.fn(),
  scatterSaveFileMarkSuccess: jest.fn(),
  scatterSaveFileSuccess: jest.fn()
}))
jest.mock('@/constant/modify/sourceRingModify', () => ({
  sourceRingDeleteFileModify: jest.fn(),
  sourceRingUploadFileModify: jest.fn(),
  doSourceRingModify: jest.fn(),
  sourceRingSaveFileMarkSuccess: jest.fn(),
  sourceRingSaveFileSuccess: jest.fn()
}))
jest.mock('@/constant/modify/stackBarModify', () => ({
  stackBarDeleteFileModify: jest.fn(),
  stackBarUploadFileModify: jest.fn(),
  doStackBarModify: jest.fn(),
  stackBarSaveFileMarkSuccess: jest.fn(),
  stackBarSaveFileSuccess: jest.fn()
}))
jest.mock('@/constant/modify/sankeyModify', () => ({
  sankeyDeleteFileModify: jest.fn(),
  sankeyUploadFileModify: jest.fn(),
  doSankeyModify: jest.fn(),
  sankeySaveFileMarkSuccess: jest.fn(),
  sankeySaveFileSuccess: jest.fn()
}))
jest.mock('@/constant/modify/networkModify', () => ({
  networkDeleteFileModify: jest.fn(),
  networkUploadFileModify: jest.fn(),
  doNetworkModify: jest.fn(),
  networkSaveFileMarkSuccess: jest.fn(),
  networkSaveFileSuccess: jest.fn()
}))
jest.mock('@/constant/modify/commonParamModify/legendSettingsParamModify', () => ({
  legendSettingsParamModify: jest.fn()
}))

const {
  syncScaleIntervalRangeByMarkColumns
} = require('@/constant/modify/publicFunction')

function createCommonParams(setting, overrides = {}) {
  return {
    fileSettings: [
      {
        headerInfoList: [
          {
            columnName: 'score',
            chooseColumnUniqueList: ['10', '20', '30']
          }
        ],
        fileMarkContainerList: []
      }
    ],
    topParamSettingsList: [
      {
        moduleType: -1,
        paramSettingsList: [setting]
      }
    ],
    ...overrides
  }
}

describe('syncScaleIntervalRangeByMarkColumns', () => {
  test('hard validation clears partial values', () => {
    const setting = {
      moduleType: 24,
      start: 0,
      end: null,
      interval: 5,
      scaleFileMarkColumnList: [
        { filedName: 'score', fileSettingIndex: 0 }
      ]
    }

    syncScaleIntervalRangeByMarkColumns(createCommonParams(setting), { hard: true })

    expect(setting.start).toBeNull()
    expect(setting.end).toBeNull()
    expect(setting.interval).toBeNull()
  })

  test('hard validation fixes only invalid fields', () => {
    const setting = {
      moduleType: 24,
      start: -100,
      end: 40,
      interval: 50,
      scaleFileMarkColumnList: [
        { filedName: 'score', fileSettingIndex: 0 }
      ]
    }

    syncScaleIntervalRangeByMarkColumns(createCommonParams(setting), { hard: true })

    expect(setting.start).toBe(-10)
    expect(setting.end).toBe(40)
    expect(setting.interval).toBe(20)
  })

  test('light validation leaves partial input untouched', () => {
    const setting = {
      moduleType: 24,
      start: -100,
      end: null,
      interval: 50,
      scaleFileMarkColumnList: [
        { filedName: 'score', fileSettingIndex: 0 }
      ]
    }

    syncScaleIntervalRangeByMarkColumns(createCommonParams(setting), { hard: false })

    expect(setting.start).toBe(-100)
    expect(setting.end).toBeNull()
    expect(setting.interval).toBe(50)
  })

  test('hard validation clears values when linked range has no positive span', () => {
    const setting = {
      moduleType: 24,
      start: 1,
      end: 2,
      interval: 1,
      scaleFileMarkColumnList: [
        { filedName: 'score', fileSettingIndex: 0 }
      ]
    }
    const commonParams = createCommonParams(setting, {
      fileSettings: [
        {
          headerInfoList: [
            {
              columnName: 'score',
              chooseColumnUniqueList: ['5', '5']
            }
          ],
          fileMarkContainerList: []
        }
      ]
    })

    syncScaleIntervalRangeByMarkColumns(commonParams, { hard: true })

    expect(setting.start).toBeNull()
    expect(setting.end).toBeNull()
    expect(setting.interval).toBeNull()
  })

  test('skips settings without scaleFileMarkColumnList', () => {
    const setting = {
      moduleType: 24,
      start: -100,
      end: null,
      interval: 50
    }

    syncScaleIntervalRangeByMarkColumns(createCommonParams(setting), { hard: true })

    expect(setting.start).toBe(-100)
    expect(setting.end).toBeNull()
    expect(setting.interval).toBe(50)
  })
})
```

- [ ] **Step 2: Run the helper tests and verify they fail**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
node scripts/run-unit-tests.js tests/unit/constant/modify/scaleIntervalRange.spec.js
```

Expected: FAIL because `syncScaleIntervalRangeByMarkColumns` is not exported.

- [ ] **Step 3: Add helpers to publicFunction.js**

In `goto-web/src/constant/modify/publicFunction.js`, add these functions after `syncMarkColumnContentToGroupParamDirect`:

```js
function forEachScaleIntervalParam(commonParams, handler) {
  if (!commonParams || !CommonUtils.isNotEmptyArray(commonParams.topParamSettingsList)) {
    return
  }
  for (const paramGroup of commonParams.topParamSettingsList) {
    if (paramGroup.moduleType === -1 && CommonUtils.isNotEmptyArray(paramGroup.paramSettingsList)) {
      for (const param of paramGroup.paramSettingsList) {
        if (param && param.moduleType === 24) {
          handler(param)
        }
      }
      continue
    }
    if (paramGroup && paramGroup.moduleType === 24) {
      handler(paramGroup)
    }
  }
}

function hasScaleIntervalValue(setting) {
  return !isBlankScaleIntervalValue(setting.start) ||
    !isBlankScaleIntervalValue(setting.end) ||
    !isBlankScaleIntervalValue(setting.interval)
}

function hasCompleteScaleIntervalValue(setting) {
  return !isBlankScaleIntervalValue(setting.start) &&
    !isBlankScaleIntervalValue(setting.end) &&
    !isBlankScaleIntervalValue(setting.interval)
}

function isBlankScaleIntervalValue(value) {
  return value === null || value === undefined || value === ''
}

function toScaleIntervalNumber(value) {
  if (isBlankScaleIntervalValue(value)) {
    return null
  }
  const numberValue = Number(value)
  return Number.isFinite(numberValue) ? numberValue : null
}

function setScaleIntervalField(commonParams, setting, fieldName, value) {
  if (commonParams && commonParams.vueContext && commonParams.vueContext.$set) {
    commonParams.vueContext.$set(setting, fieldName, value)
  } else {
    setting[fieldName] = value
  }
}

function clearScaleIntervalValues(commonParams, setting) {
  setScaleIntervalField(commonParams, setting, 'start', null)
  setScaleIntervalField(commonParams, setting, 'end', null)
  setScaleIntervalField(commonParams, setting, 'interval', null)
}

function obtainScaleIntervalLinkedRange(commonParams, setting) {
  if (!setting || !CommonUtils.isNotEmptyArray(setting.scaleFileMarkColumnList)) {
    return null
  }

  const markInfoList = findFilteredHeaderInfoListByFileMarkColumns(
    commonParams,
    setting.scaleFileMarkColumnList
  )

  const numbers = []
  for (const markInfo of markInfoList) {
    if (!markInfo || !CommonUtils.isNotEmptyArray(markInfo.chooseColumnUniqueList)) {
      continue
    }
    for (const value of markInfo.chooseColumnUniqueList) {
      const numberValue = Number(value)
      if (Number.isFinite(numberValue)) {
        numbers.push(numberValue)
      }
    }
  }

  if (!CommonUtils.isNotEmptyArray(numbers)) {
    return null
  }

  const min = Math.min(...numbers)
  const max = Math.max(...numbers)
  const span = max - min
  if (!Number.isFinite(span) || span <= 0) {
    return null
  }

  return {
    min,
    max,
    span,
    lowerBound: min - span,
    upperBound: max + span
  }
}

function normalizeScaleIntervalParam(commonParams, setting, options = {}) {
  const hard = options.hard !== false
  if (!setting || !CommonUtils.isNotEmptyArray(setting.scaleFileMarkColumnList)) {
    return
  }

  if (!hasScaleIntervalValue(setting)) {
    return
  }

  if (!hasCompleteScaleIntervalValue(setting)) {
    if (hard) {
      clearScaleIntervalValues(commonParams, setting)
    }
    return
  }

  const range = obtainScaleIntervalLinkedRange(commonParams, setting)
  if (!range) {
    if (hard) {
      clearScaleIntervalValues(commonParams, setting)
    }
    return
  }

  const start = toScaleIntervalNumber(setting.start)
  const end = toScaleIntervalNumber(setting.end)
  const interval = toScaleIntervalNumber(setting.interval)
  if (start === null || end === null || interval === null) {
    if (hard) {
      clearScaleIntervalValues(commonParams, setting)
    }
    return
  }

  if (start < range.lowerBound) {
    setScaleIntervalField(commonParams, setting, 'start', range.lowerBound)
  }
  if (end > range.upperBound) {
    setScaleIntervalField(commonParams, setting, 'end', range.upperBound)
  }
  if (interval <= 0 || interval > range.span) {
    setScaleIntervalField(commonParams, setting, 'interval', range.span)
  }
}

function syncScaleIntervalRangeByMarkColumns(commonParams, options = {}) {
  forEachScaleIntervalParam(commonParams, (setting) => {
    normalizeScaleIntervalParam(commonParams, setting, options)
  })
}
```

Add `syncScaleIntervalRangeByMarkColumns` to the export list at the bottom of `publicFunction.js`.

- [ ] **Step 4: Run the helper tests and verify they pass**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
node scripts/run-unit-tests.js tests/unit/constant/modify/scaleIntervalRange.spec.js
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add goto-web/src/constant/modify/publicFunction.js \
  goto-web/tests/unit/constant/modify/scaleIntervalRange.spec.js
git commit -m "feat: validate scale interval mark column range"
```

---

### Task 4: Hook Upload and Save-Mark Hard Validation

**Files:**
- Modify: `goto-web/src/constant/modify/publicFunction.js`
- Modify: `goto-web/tests/unit/constant/modify/scaleIntervalRange.spec.js`

- [ ] **Step 1: Add failing integration tests for upload/save hooks**

Append to `goto-web/tests/unit/constant/modify/scaleIntervalRange.spec.js`:

```js
const {
  uploadFileModify,
  saveFileMarkSuccess
} = require('@/constant/modify/publicFunction')

describe('scale interval upload/save hooks', () => {
  test('uploadFileModify hard-validates scale interval settings', () => {
    const setting = {
      moduleType: 24,
      start: -100,
      end: 40,
      interval: 50,
      scaleFileMarkColumnList: [
        { filedName: 'score', fileSettingIndex: 0 }
      ]
    }

    uploadFileModify(createCommonParams(setting))

    expect(setting.start).toBe(-10)
    expect(setting.end).toBe(40)
    expect(setting.interval).toBe(20)
  })

  test('saveFileMarkSuccess hard-validates scale interval settings', () => {
    const setting = {
      moduleType: 24,
      start: 0,
      end: null,
      interval: 1,
      scaleFileMarkColumnList: [
        { filedName: 'score', fileSettingIndex: 0 }
      ]
    }

    saveFileMarkSuccess(createCommonParams(setting))

    expect(setting.start).toBeNull()
    expect(setting.end).toBeNull()
    expect(setting.interval).toBeNull()
  })
})
```

- [ ] **Step 2: Run the hook tests and verify they fail**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
node scripts/run-unit-tests.js tests/unit/constant/modify/scaleIntervalRange.spec.js
```

Expected: FAIL because upload/save hooks do not call scale interval range sync yet.

- [ ] **Step 3: Call the helper from upload and save-mark hooks**

Modify `goto-web/src/constant/modify/publicFunction.js`.

In `uploadFileModify(commonParams)`, after `forEachListMarkColumnContentParam(...)` handling completes, add:

```js
  syncScaleIntervalRangeByMarkColumns(commonParams, { hard: true })
```

In `saveFileMarkSuccess(commonParams)`, after `forEachListMarkColumnContentParam(...)` handling completes, add:

```js
  syncScaleIntervalRangeByMarkColumns(commonParams, { hard: true })
```

- [ ] **Step 4: Run the hook tests and existing public function tests**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
node scripts/run-unit-tests.js tests/unit/constant/modify/scaleIntervalRange.spec.js tests/unit/constant/modify/publicFunction.spec.js
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add goto-web/src/constant/modify/publicFunction.js \
  goto-web/tests/unit/constant/modify/scaleIntervalRange.spec.js
git commit -m "feat: sync scale interval range after file mark changes"
```

---

### Task 5: Add moduleType 24 Selector and Input Linkage

**Files:**
- Modify: `goto-web/src/views/goto/components/paramSettings.vue`
- Create: `goto-web/tests/unit/components/paramSettingsScaleInterval.spec.js`

- [ ] **Step 1: Write failing source guard test**

Create `goto-web/tests/unit/components/paramSettingsScaleInterval.spec.js`:

```js
/* eslint-env jest */
const fs = require('fs')
const path = require('path')

function readSource(relativePath) {
  return fs.readFileSync(path.resolve(__dirname, '../../../', relativePath), 'utf8')
}

describe('paramSettings moduleType 24 scale interval mark columns', () => {
  test('renders MarkColumnSelector with scaleFileMarkColumnList for moduleType 24', () => {
    const source = readSource('src/views/goto/components/paramSettings.vue')

    expect(source).toContain('import MarkColumnSelector from "@/views/goto/components/form/plotSetting/MarkColumnSelector.vue"')
    expect(source).toMatch(/MarkColumnSelector[\s\S]*field-name="scaleFileMarkColumnList"/)
    expect(source).toContain('@modify="handleScaleMarkColumnModify(setting)"')
  })

  test('scale interval numeric inputs use input linkage handler', () => {
    const source = readSource('src/views/goto/components/paramSettings.vue')

    expect(source).toContain('@modify="handleScaleIntervalInputModify(setting)"')
    expect(source).toContain('PublicFunctions.syncScaleIntervalRangeByMarkColumns')
  })
})
```

- [ ] **Step 2: Run the source guard test and verify it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
node scripts/run-unit-tests.js tests/unit/components/paramSettingsScaleInterval.spec.js
```

Expected: FAIL because `MarkColumnSelector` is not imported or rendered in `paramSettings.vue`.

- [ ] **Step 3: Import and register dependencies**

Modify `goto-web/src/views/goto/components/paramSettings.vue`.

Change:

```js
import {
  associationParamModify,
  findFromTopParamSettingsList
} from "@/constant/associationParamModify"
```

to:

```js
import {
  associationParamModify,
  findFromTopParamSettingsList
} from "@/constant/associationParamModify"
import * as PublicFunctions from "@/constant/modify/publicFunction"
```

Add:

```js
import MarkColumnSelector from "@/views/goto/components/form/plotSetting/MarkColumnSelector.vue"
```

Register:

```js
    MarkColumnSelector,
```

inside `components`.

- [ ] **Step 4: Add the selector and input linkage in the moduleType 24 block**

In the `setting.moduleType === 24` template block, replace each `@modify="modify"` on the three number inputs with:

```vue
@modify="handleScaleIntervalInputModify(setting)"
```

After the start/end/interval row, add:

```vue
              <div class="group-attribute-line">
                <MarkColumnSelector
                  :parent="setting"
                  :file-settings="fileSettings"
                  field-name="scaleFileMarkColumnList"
                  title="刻度范围标记列选择"
                  empty-text="暂无可选择的刻度范围标记列"
                  clean-message="已清理"
                  @modify="handleScaleMarkColumnModify(setting)"
                />
              </div>
```

Add methods in the `methods` object:

```js
    buildScaleIntervalCommonParams(setting) {
      return {
        vueContext: this,
        setting,
        fileSettings: this.fileSettings,
        groupId: this.groupId,
        plotTypeId: this.plotTypeId,
        userDrawId: this.userDrawId,
        topParamSettingsList: this.data ? this.data.paramSettingsList : [],
        params: null
      }
    },
    handleScaleIntervalInputModify(setting) {
      PublicFunctions.syncScaleIntervalRangeByMarkColumns(
        this.buildScaleIntervalCommonParams(setting),
        { hard: false }
      )
      this.modify(setting)
    },
    handleScaleMarkColumnModify(setting) {
      PublicFunctions.syncScaleIntervalRangeByMarkColumns(
        this.buildScaleIntervalCommonParams(setting),
        { hard: true }
      )
      this.modify(setting)
    },
```

- [ ] **Step 5: Run the source guard and range tests**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
node scripts/run-unit-tests.js tests/unit/components/paramSettingsScaleInterval.spec.js tests/unit/constant/modify/scaleIntervalRange.spec.js
```

Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add goto-web/src/views/goto/components/paramSettings.vue \
  goto-web/tests/unit/components/paramSettingsScaleInterval.spec.js
git commit -m "feat: add scale interval mark selector"
```

---

### Task 6: Initialize New Field When Adding moduleType 24

**Files:**
- Modify: `goto-web/src/views/goto/components/paramSettings.vue`
- Modify: `goto-web/tests/unit/components/paramSettingsScaleInterval.spec.js`

- [ ] **Step 1: Add a failing source guard for initialization**

Append to `goto-web/tests/unit/components/paramSettingsScaleInterval.spec.js`:

```js
  test('initializes scaleFileMarkColumnList when adding moduleType 24', () => {
    const source = readSource('src/views/goto/components/paramSettings.vue')

    expect(source).toContain('this.$set(o, "scaleFileMarkColumnList", [])')
  })
```

- [ ] **Step 2: Run the guard and verify it fails**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
node scripts/run-unit-tests.js tests/unit/components/paramSettingsScaleInterval.spec.js
```

Expected: FAIL because `scaleFileMarkColumnList` is not initialized.

- [ ] **Step 3: Initialize the field**

In `paramSettings.vue`, inside the existing `if (o.moduleType === 24)` block in `addFormModule`, add:

```js
          if (!o.scaleFileMarkColumnList) {
            this.$set(o, "scaleFileMarkColumnList", [])
          }
```

- [ ] **Step 4: Run the guard**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
node scripts/run-unit-tests.js tests/unit/components/paramSettingsScaleInterval.spec.js
```

Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add goto-web/src/views/goto/components/paramSettings.vue \
  goto-web/tests/unit/components/paramSettingsScaleInterval.spec.js
git commit -m "feat: initialize scale interval mark columns"
```

---

### Task 7: Final Verification

**Files:**
- Verify only; no planned source edits.

- [ ] **Step 1: Run targeted frontend tests**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
node scripts/run-unit-tests.js tests/unit/components/MarkColumnSelector.spec.js \
  tests/unit/components/paramSettingsScaleInterval.spec.js \
  tests/unit/constant/modify/scaleIntervalRange.spec.js \
  tests/unit/constant/modify/publicFunction.spec.js \
  tests/unit/styles/checkboxTheme.spec.js
```

Expected: PASS.

- [ ] **Step 2: Run targeted backend test**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto
mvn -pl common -Dtest=ScaleIntervalParamMarkColumnTest test
```

Expected: PASS.

- [ ] **Step 3: Run frontend lint for touched source files**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software/goto-web
npx eslint src/views/goto/components/form/plotSetting/MarkColumnSelector.vue \
  src/views/goto/components/paramSettings.vue \
  src/constant/modify/publicFunction.js
```

Expected: PASS or only pre-existing warnings unrelated to touched lines. If ESLint reports errors in touched lines, fix them before proceeding.

- [ ] **Step 4: Inspect final diff**

Run:

```bash
cd /mnt/f/IdeaProjects/goto-software
git diff --stat HEAD~6..HEAD
git status --short
```

Expected: commits contain only files listed in this plan. Existing unrelated dirty worktree files may remain, but none should be introduced by this implementation outside the listed paths.

---

## Self-Review

Spec coverage:

1. Backend field is covered by Task 1.
2. Reusable `MarkColumnSelector` is covered by Task 2.
3. `moduleType === 24` UI is covered by Task 5.
4. Upload/save hard validation is covered by Tasks 3 and 4.
5. Input and mark-column selection linkage is covered by Task 5.
6. Initialization for newly added moduleType 24 params is covered by Task 6.
7. Verification is covered by Task 7.

Placeholder scan: this plan contains no placeholder steps.

Type consistency:

1. The backend field is `scaleFileMarkColumnList`.
2. Frontend selector uses `fieldName` prop and Vue template kebab-case `field-name`.
3. The shared helper name is `syncScaleIntervalRangeByMarkColumns(commonParams, options)`.
4. Hard validation uses `{ hard: true }`; light validation uses `{ hard: false }`.
