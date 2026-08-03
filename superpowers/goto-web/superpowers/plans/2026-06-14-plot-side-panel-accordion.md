# PlotSidePanel Accordion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make `PlotSidePanel.vue` collapse groups behave like an accordion so opening one item automatically closes the others.

**Architecture:** Use Element UI's native `el-collapse` `accordion` prop and store the active collapse name as a single value instead of an array. Keep all upload, delete, permission, and parameter rendering behavior unchanged.

**Tech Stack:** Vue 2, Element UI 2.15, Jest via `npm run test:unit`.

---

## File Structure

- Modify: `goto-web/src/views/goto/plot/type/components/PlotSidePanel.vue`
  - Adds `accordion` to `el-collapse`.
  - Changes `activeGroup` initialization and `initActiveGroup()` to use a single value.
- Create: `goto-web/tests/unit/views/goto/plot/type/components/plotSidePanelAccordion.spec.js`
  - Source-level regression test matching existing test style in this repo.
  - Verifies the template has accordion mode and the script no longer initializes or assigns array-based active groups.

---

### Task 1: Add Regression Test

**Files:**
- Create: `goto-web/tests/unit/views/goto/plot/type/components/plotSidePanelAccordion.spec.js`

- [ ] **Step 1: Write the failing test**

Create `goto-web/tests/unit/views/goto/plot/type/components/plotSidePanelAccordion.spec.js` with:

```js
/* eslint-env jest */
const fs = require('fs')
const path = require('path')

function readPlotSidePanelSource() {
  return fs.readFileSync(
    path.resolve(
      __dirname,
      '../../../../../../src/views/goto/plot/type/components/PlotSidePanel.vue'
    ),
    'utf8'
  )
}

describe('PlotSidePanel accordion behavior', () => {
  test('enables native Element UI accordion mode on the collapse container', () => {
    const source = readPlotSidePanelSource()

    expect(source).toMatch(/<el-collapse\s+[^>]*v-model="activeGroup"[^>]*accordion/)
  })

  test('stores only one active collapse item instead of an array', () => {
    const source = readPlotSidePanelSource()

    expect(source).toMatch(/activeGroup:\s*-1/)
    expect(source).not.toMatch(/activeGroup:\s*\[-1\]/)
    expect(source).not.toMatch(/this\.activeGroup\s*=\s*activeNames/)
  })
})
```

- [ ] **Step 2: Run the new test and verify it fails**

Run:

```bash
cd goto-web
npx jest tests/unit/views/goto/plot/type/components/plotSidePanelAccordion.spec.js --runInBand
```

Expected: FAIL. The first test fails because `<el-collapse>` does not include `accordion`, and the second test fails because `activeGroup` is currently initialized as `[-1]`.

- [ ] **Step 3: Commit the failing test**

Run:

```bash
git add goto-web/tests/unit/views/goto/plot/type/components/plotSidePanelAccordion.spec.js
git commit -m "test: cover plot side panel accordion behavior"
```

Expected: commit succeeds with only the new test file staged.

---

### Task 2: Enable Accordion Behavior

**Files:**
- Modify: `goto-web/src/views/goto/plot/type/components/PlotSidePanel.vue`
- Test: `goto-web/tests/unit/views/goto/plot/type/components/plotSidePanelAccordion.spec.js`

- [ ] **Step 1: Add the accordion prop**

Change the collapse opening tag in `PlotSidePanel.vue` from:

```vue
<el-collapse v-model="activeGroup">
```

to:

```vue
<el-collapse v-model="activeGroup" accordion>
```

- [ ] **Step 2: Change activeGroup to a single value**

In `data()`, change:

```js
activeGroup: [-1],
```

to:

```js
activeGroup: -1,
```

- [ ] **Step 3: Simplify initActiveGroup for accordion mode**

Replace the full `initActiveGroup()` method with:

```js
initActiveGroup() {
  // Accordion mode can only keep one item open. Keep upload open by default.
  this.activeGroup = -1
},
```

This intentionally makes accordion behavior take precedence over `defaultExpandAll`, matching the approved spec.

- [ ] **Step 4: Run the focused regression test**

Run:

```bash
cd goto-web
npx jest tests/unit/views/goto/plot/type/components/plotSidePanelAccordion.spec.js --runInBand
```

Expected: PASS.

- [ ] **Step 5: Run lint for the touched Vue file**

Run:

```bash
cd goto-web
npx eslint src/views/goto/plot/type/components/PlotSidePanel.vue
```

Expected: exits with code 0. If unrelated existing lint errors appear outside the changed lines, record them and do not expand the fix scope.

- [ ] **Step 6: Commit the implementation**

Run:

```bash
git add goto-web/src/views/goto/plot/type/components/PlotSidePanel.vue
git commit -m "fix: make plot side panel collapse accordion"
```

Expected: commit succeeds with only the component implementation staged.

---

### Task 3: Final Verification

**Files:**
- Read: `goto-web/src/views/goto/plot/type/components/PlotSidePanel.vue`
- Read: `goto-web/tests/unit/views/goto/plot/type/components/plotSidePanelAccordion.spec.js`

- [ ] **Step 1: Run the focused test once more**

Run:

```bash
cd goto-web
npx jest tests/unit/views/goto/plot/type/components/plotSidePanelAccordion.spec.js --runInBand
```

Expected: PASS.

- [ ] **Step 2: Inspect the final diff**

Run:

```bash
git diff --stat HEAD~2..HEAD -- goto-web/src/views/goto/plot/type/components/PlotSidePanel.vue goto-web/tests/unit/views/goto/plot/type/components/plotSidePanelAccordion.spec.js
```

Expected: output lists only `PlotSidePanel.vue` and `plotSidePanelAccordion.spec.js`.

- [ ] **Step 3: Manual browser verification**

Start the dev server if needed:

```bash
cd goto-web
npm run dev
```

Open the plot draw page and verify:

1. "上传文件" is open by default.
2. Clicking a parameter group closes "上传文件".
3. Clicking another parameter group closes the previous parameter group.
4. Clicking "上传文件" closes the parameter group.
5. Clicking the currently open item again does not open any other item.
