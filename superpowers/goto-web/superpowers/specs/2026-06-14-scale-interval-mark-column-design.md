# 刻度间隔关联标记列设计

## 背景

`ScaleIntervalParam` 用于配置刻度间隔参数，当前包含 `start`、`end`、`interval` 和 `keyValues`。业务需要让刻度间隔参数关联文件标记列，并在上传文件或保存标记后，根据关联列的数值范围整理旧的刻度配置。

现有 `ClassifyParam` 使用 `byFileMarkColumnList` 记录 BY 相关标记列，`FacetParams` 使用 `facetFileMarkColumnList` 记录分面相关标记列。前端已经存在 `MarkColumnSelector.vue`，但它固定读写 `fileMarkColumnList`，还不能直接复用到分类、分面或刻度间隔自定义字段。

## 目标

1. 为 `ScaleIntervalParam` 增加刻度范围关联标记列字段。
2. 将标记列选择 UI 抽成可配置字段名的通用组件，后续其他 moduleType 可直接引入。
3. 在 `paramSettings.vue` 的 `moduleType === 24` 表单中增加刻度范围标记列选择。
4. 上传文件或保存标记后，按关联列的数值范围校验并修正 `start/end/interval`。
5. 用户输入 `start/end/interval` 或修改关联标记列时，按合适的联动规则修正配置。

## 非目标

1. 不修改刻度间隔的 R 输出结构，新增字段只用于配置、同步和前端联动。
2. 不迁移历史 `.worktrees/*` 文档。
3. 不把历史测试结果迁移到中心文档目录。
4. 不重构无关 moduleType 的配置表单。

## 后端字段

在 `goto/common/src/main/java/com/freedom/model/form/ScaleIntervalParam.java` 中新增字段：

```java
@Syncable
private List<FileMarkColumn> scaleFileMarkColumnList;
```

字段含义：刻度范围关联标记列。

字段命名不用 `by`，避免和分类 BY 语义混淆。`scaleFileMarkColumnList` 表示该字段属于刻度范围校验，数据结构与现有 `FileMarkColumn` 一致：

```java
{
  filedName: string,
  fileSettingIndex: number
}
```

该字段不参与 `AutoParamCreateUtils.obtainScaleIntervalParamValue` 输出。

## 通用标记列选择组件

扩展现有 `goto-web/src/views/goto/components/form/plotSetting/MarkColumnSelector.vue`，让它支持自定义字段名。

新增 props：

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

组件内部所有 `parent.fileMarkColumnList` 改为 `parent[fieldName]`。默认值保持 `fileMarkColumnList`，因此 moduleType 27/30 现有调用不需要改也能保持行为。

分类和分面后续可用同一个组件替换当前重复选择逻辑：

```vue
<MarkColumnSelector
  :parent="setting"
  field-name="byFileMarkColumnList"
  title="BY标记列选择"
  empty-text="暂无可选择的BY标记列"
  @modify="modify"
/>
```

刻度间隔使用：

```vue
<MarkColumnSelector
  :parent="setting"
  :file-settings="fileSettings"
  field-name="scaleFileMarkColumnList"
  title="刻度范围标记列选择"
  empty-text="暂无可选择的刻度范围标记列"
  @modify="handleScaleMarkColumnModify(setting)"
/>
```

## moduleType 24 表单

`goto-web/src/views/goto/components/paramSettings.vue` 中已有 `setting.moduleType === 24` 内联表单，包含 `start`、`end`、`interval` 和 `keyValues`。

在该区域加入 `MarkColumnSelector`，位置放在 `start/end/interval` 和 `keyValues` 附近。选择变化后触发刻度范围硬校验，因为关联列变化会改变范围基础。

新增选择组件依赖 `fileSettings`，沿用当前 `paramSettings.vue` 已有 props。

如果绘图侧 `goto-web/src/views/goto/components/form/draw/scaleIntervalParam.vue` 也实际承担 moduleType 24 的配置编辑，则计划阶段补同等能力：传入 `fileSettings`、渲染 `MarkColumnSelector`、复用同一套校验工具函数。

## 范围计算

范围来源为 `scaleFileMarkColumnList` 对应的标记列内容。

使用已有工具：

```js
findFilteredHeaderInfoListByFileMarkColumns(commonParams, scaleFileMarkColumnList)
```

从返回的 `markInfoList` 汇总所有 `chooseColumnUniqueList`，过滤为有效数字，然后计算：

```js
min = Math.min(...numbers)
max = Math.max(...numbers)
span = max - min
lowerBound = min - span
upperBound = max + span
```

多个关联标记列时，合并所有有效数值，使用整体 `min/max/span`。

未配置 `scaleFileMarkColumnList` 时，表示该刻度间隔参数没有启用关联列范围校验，直接跳过。

已配置关联列但无有效范围的场景：

1. 关联文件或字段不存在。
2. 标记列没有 `chooseColumnUniqueList`。
3. 所有内容都不是有效数字。
4. `span <= 0`，因为 `interval` 要求大于 0 且不能大于 `span`。

## 校验规则

三项字段：

```js
start
end
interval
```

数值约束：

```js
start >= min - span
end <= max + span
interval > 0
interval <= span
```

修正规则：

1. `start` 不合规则修正为 `min - span`。
2. `end` 不合规则修正为 `max + span`。
3. `interval` 不合规则修正为 `span`。
4. 合规字段不动。

## 输入联动

输入联动是轻校验，目标是不打断用户输入。

触发场景：

1. 用户修改 `start`。
2. 用户修改 `end`。
3. 用户修改 `interval`。

规则：

1. 没有关联标记列，或暂时算不出有效 `min/max/span`：不处理。
2. 三项没有全部填完：不清空、不修正，让用户继续输入。
3. 三项都有值时执行校验。
4. 越界字段按边界值修正，合规字段不动。
5. 非数字值按未填完处理，不立即清空。

当前 `Input` 组件基于 `@input` 触发 `modify`，所以输入联动不能执行“三项任一为空就清空”，否则用户无法逐项输入。

## 标记列选择联动

修改 `scaleFileMarkColumnList` 按硬校验处理。

原因：关联列变化代表范围基础发生变化，不是普通输入过程。

规则：

1. 三项全空：跳过。
2. 三项任一为空：清空 `start/end/interval`。
3. 算不出有效范围：清空三项。
4. 三项都有值且范围有效：只修正不合规字段，合规字段不动。

## 上传文件和保存标记后校验

上传文件成功、保存标记成功后按硬校验处理。

接入位置：

1. `PublicFunctions.uploadFileModify(commonParams)`
2. `PublicFunctions.saveFileMarkSuccess(commonParams)`

新增统一工具函数，例如：

```js
syncScaleIntervalRangeByMarkColumns(commonParams)
```

它扫描 `commonParams.topParamSettingsList` 下所有 moduleType 24 参数，处理其 `scaleFileMarkColumnList`。

硬校验规则：

1. 没有关联 `scaleFileMarkColumnList`：跳过。
2. `start/end/interval` 三项全空：跳过。
3. 三项任一为空：清空三项。
4. 关联列算不出有效范围：清空三项。
5. 三项都有值且范围有效：只修正不合规字段，合规字段不动。

清空使用：

```js
start = null
end = null
interval = null
```

如果有 `commonParams.vueContext`，用 `$set` 保持 Vue 响应式。

## 错误处理和用户提示

上传文件或保存标记后不弹错误提示。原因是这些流程可能批量影响多个参数，频繁提示会干扰用户。

输入联动也不弹提示，直接通过字段修正体现结果。

需要保留必要的开发期日志时，应避免新增高频 `console.log`。

## 测试计划

后端：

1. `ScaleIntervalParam` 存在 `scaleFileMarkColumnList` 字段。
2. 字段类型为 `List<FileMarkColumn>`。
3. 字段带 `@Syncable`。

前端组件：

1. `MarkColumnSelector` 默认仍读写 `fileMarkColumnList`。
2. 传入 `fieldName="scaleFileMarkColumnList"` 后读写自定义字段。
3. 清理无效标记时使用自定义字段。

moduleType 24：

1. `paramSettings.vue` 在 `moduleType === 24` 时渲染刻度范围标记列选择。
2. 选择结果写入 `setting.scaleFileMarkColumnList`。
3. 标记列选择变化触发硬校验。

范围工具函数：

1. 三项全空跳过。
2. 硬校验时任一为空清空三项。
3. `start < min - span` 时只修正 `start`。
4. `end > max + span` 时只修正 `end`。
5. `interval <= 0` 或 `interval > span` 时只修正 `interval`。
6. 合规字段不动。
7. 无有效数字或 `span <= 0` 时硬校验清空三项。
8. 输入联动在三项未填完时不清空。

回归：

1. moduleType 27/30 的 `MarkColumnSelector` 默认行为不变。
2. 分类和分面已有标记列选择逻辑不因本次改动失效。
3. 上传文件和保存标记后原有 ListMarkColumnContentParam 同步逻辑仍执行。

## 实施顺序建议

1. 后端新增 `ScaleIntervalParam.scaleFileMarkColumnList`。
2. 参数化 `MarkColumnSelector.vue`。
3. 在 `paramSettings.vue` 注册并使用 `MarkColumnSelector`，为 moduleType 24 增加选择区域。
4. 增加刻度范围计算和校验工具函数。
5. 接入 `uploadFileModify`、`saveFileMarkSuccess` 和 moduleType 24 输入/标记列选择联动。
6. 增加或更新单元测试。
7. 运行前后端相关验证命令。
