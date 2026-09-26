---
change_set_id: CS-20260910-0a209861-7c98-4119-8227-40790dabbef6
related_change_set_ids:
  - CS-20260909-8db76639-0c29-4ef0-a223-5850749acbf0
---

# 分类映射弹窗按 byDisabled 控制列选择设计

## 背景与目标

热图上传列属性文件并保存列标记后，列注释条会为属性列生成分类映射项。当前生成项带有 `byDisabled=true`，绘图表单因此不绑定弹窗点击事件，导致用户既不能重新选择 by 列，也不能使用同一弹窗内的映射模式切换。

本次修复将“能否打开弹窗”和“能否选择 by 列”解耦：分类映射项只要显示 by 区域就允许打开弹窗；进入弹窗后，by 列是否可编辑仅由 `byDisabled` 决定。映射模式是否可编辑仍仅由既有 `limitType` 决定。

## 已确认行为

| 条件 | 打开弹窗 | 选择或清空 by 列 | 切换映射模式 |
|---|---|---|---|
| `byDisabled=true`、`limitType=false` | 允许 | 禁止 | 允许 |
| `byDisabled=true`、`limitType=true` | 允许 | 禁止 | 禁止 |
| `byDisabled=false`、`limitType=false` | 允许 | 允许 | 允许 |
| `byDisabled=false`、`limitType=true` | 允许 | 允许，但延续既有类型过滤 | 禁止 |

`byDisabled=true` 时，列列表继续显示，用禁用样式标识不可操作；当前 by 列保持选中，用户仍可确认映射模式修改。

## 方案比较

### 方案一：弹窗入口与列选择分别控制（采用）

- 分类项的 by 展示区域始终绑定弹窗入口。
- 打开弹窗时传入 `byDisabled`。
- 弹窗根据 `byDisabled` 禁用列卡片及“清空”等会改变选择的操作，并在事件处理层再次拒绝修改。
- 映射模式按钮继续只读取 `limitType`。

该方案完整保留两个字段各自的业务含义，影响范围最小，也能避免仅靠视觉禁用被事件调用绕过。

### 方案二：`byDisabled=true` 时隐藏列列表

实现更简单，但用户无法在弹窗中确认当前绑定列，信息不完整，不采用。

### 方案三：将热图注释条的 `byDisabled` 改为 `false`

能够重新打开弹窗，但会错误放开 by 列编辑，破坏 `byDisabled` 的既有语义，不采用。

## 组件与数据流

1. `classifyItem.vue` 的 by 区域点击统一调用 `chooseColumn()`，不再因 `byDisabled=true` 丢失入口。
2. `by.vue` 在 `byDisabled=true` 时隐藏会清空当前 by 的“重置”入口。
3. `classifyItemMixins.js` 将当前分类项的 `byDisabled` 写入弹窗参数。
4. `classifyChooseHeaderInfo.vue`：
   - 列卡片展示禁用状态；
   - `byDisabled=true` 时，列卡片点击和清空操作不改变状态；
   - 打开弹窗时沿用当前已选 by 列；
   - 确认时仍把当前列和映射模式交给既有回调。
5. `limitType` 的过滤、映射按钮禁用和类型不适配警告逻辑保持不变。

## 测试与验收

### 单元测试

- `byDisabled=true` 时分类项仍具有打开弹窗的点击入口。
- `byDisabled=true` 时分类项外部的重置入口不可用。
- 弹窗列卡片和清空操作处于禁用状态，直接调用处理方法也不会改变 by 选择。
- `byDisabled=false` 时列选择行为保持原样。
- 映射按钮仍只受 `limitType` 控制，不受 `byDisabled` 影响。

### Playwright 端到端测试

按实际反馈路径执行：

1. 热图列属性上传 `metadata.xlsx`。
2. 将 `ID` 标记为列，将 `Time`、`Gender`、`Clin_1` 标记为属性并保存。
3. 展开列注释条，点击 `Time` 映射项。
4. 断言弹窗打开，当前 by 为 `Time`，列卡片不可编辑，映射模式在 `limitType=false` 时可切换。
5. 删除测试上传文件并恢复临时登录页，确认测试端口释放。

## 范围边界

- 不修改热图生成注释项时的 `byDisabled` 值。
- 不改变 `limitType`、连续/离散识别和不适配警告规则。
- 不修改后端、数据库模板或 R 服务。
- 不借此重构分类映射组件的其他历史逻辑。
