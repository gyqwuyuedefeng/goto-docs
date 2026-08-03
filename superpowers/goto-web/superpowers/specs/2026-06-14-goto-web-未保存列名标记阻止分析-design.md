# goto-web 未保存列名标记阻止分析设计

## 背景

用户反馈：在绘图页列名标记区域，X 轴原本已经保存，用户又标记了 Y 轴但没有保存，此时点击“开始分析”可以提交分析，但后续分析会报错。期望行为是根据后台配置的必选标记规则，在未完成有效标记状态时给出提示。

本次排查的相关入口：

- 前端结果与列名标记容器：[PlotResultCanvas.vue](/mnt/f/IdeaProjects/goto-software/goto-web/src/views/goto/plot/type/components/PlotResultCanvas.vue)
- 前端开始分析入口：[draw.vue](/mnt/f/IdeaProjects/goto-software/goto-web/src/views/goto/plot/type/draw.vue)
- 列名标记保存逻辑：[plot.js](/mnt/f/IdeaProjects/goto-software/goto-web/src/views/goto/plot/type/fileMark/plot.js)
- 列名选择弹窗：[chooseHeaderInfo.vue](/mnt/f/IdeaProjects/goto-software/goto-web/src/views/goto/plot/type/fileMark/chooseHeaderInfo.vue)
- 后端分析入口：[PlotServiceImpl.java](/mnt/f/IdeaProjects/goto-software/goto/system/src/main/java/com/freedom/service/impl/PlotServiceImpl.java)
- 后端标记保存入口：[PlotTypeServiceImpl.java](/mnt/f/IdeaProjects/goto-software/goto/system/src/main/java/com/freedom/service/impl/PlotTypeServiceImpl.java)

## 现状结论

列名选择弹窗会直接修改前端内存中的 `headerInfo.status`。因此，只要用户在弹窗里选择了列，`draw.vue` 当前的“必选列名标记”校验就会把这次未保存的选择当成已完成。

`custom_draw_plot` 请求体也会携带这份前端内存态，所以后端 `customDrawPlot(...)` 当前也可能看到未保存标记并放行。

但真正保存列名标记的是 `save_file_mark_settings`。该接口不仅保存 `headerInfoList`，还会调用 `PlotTypeUtils.fileMarkHandler(...)` 生成分析实际使用的 `file-mark-*.xlsx` 文件。

分析提交后，后端只把 `userDrawId` 放入 Redis Stream。异步任务消费端会重新从数据库查询 `UserDraw`，并基于已保存的任务与文件状态执行分析。未点击“保存标记”的前端内存态不能可靠进入异步分析上下文，也不会生成对应的 `file-mark-*.xlsx`。

因此，本问题的根因不是“必选标记校验完全缺失”，而是“未保存标记被前端校验和提交接口误认为可用于分析”。

## 目标

1. 用户只选择列名但未保存时，点击“开始分析”必须被阻止。
2. 阻止时提示 `请先保存标记`。
3. 已保存但仍缺少后台配置的必选列名标记时，继续使用现有提示 `请先标记列名`。
4. 不自动保存标记，避免用户无感落盘。
5. 不改变现有 `save_file_mark_settings` 的保存语义，不改变 `custom_draw_plot` 请求结构。

## 非目标

1. 不新增“保存并分析”确认框。
2. 不自动切换到缺失标记的文件 tab。
3. 不重构列名标记 UI。
4. 不改变标记文件 `file-mark-*.xlsx` 的生成规则。
5. 不扩大到统计分析模块或其它页面。

## 方案对比

### 方案 A：前端检测未保存标记并阻止分析

在列名标记组件维护 dirty 状态：用户确认选择列名后标记为未保存，保存成功后清除。开始分析前先检查是否存在 dirty 标记，存在则提示 `请先保存标记` 并阻止提交。

优点：

- 符合用户确认的产品口径。
- 改动集中在前端交互状态和提交入口。
- 不引入额外接口或数据库字段。
- 能在请求发出前给出明确反馈。

缺点：

- 只能覆盖当前页面交互产生的未保存状态。
- 浏览器刷新后 dirty 状态会丢失，但刷新后前端内存态也回到已保存状态，不会继续误判这次未保存选择。

### 方案 B：开始分析前重新拉取数据库保存态对比

点击开始分析时额外请求后端保存态，与当前页面 `headerInfoList` 对比，发现差异则阻止。

优点：

- 可以显式验证前后端保存态一致。

缺点：

- 需要额外接口或复用现有查询接口，增加请求链路。
- 需要定义复杂的对比规则。
- 对本问题而言成本偏高。

### 方案 C：后端在 `custom_draw_plot` 中读取数据库保存态重新校验

后端忽略请求体中的未保存 `fileSettings`，改为读取数据库里的 `UserDraw.fileSettings` 判断必选标记。

优点：

- 接口兜底更强。

缺点：

- 仍无法告诉用户“你当前页面有未保存标记”，只能提示缺失或不一致。
- 需要处理新建任务、示例任务、请求体与数据库状态的边界。
- 用户体验不如前端即时阻止。

## 决策

采用方案 A：前端检测未保存列名标记并阻止分析。

后端现有 `customDrawPlot(...)` 中“缺少必选列名标记”的兜底校验保留。本次不把它扩展成“保存态对比”，因为当前用户反馈要解决的是页面中“只标记但未保存”的提交入口误放行。

## 交互设计

用户在列名选择弹窗中确认选择后：

1. 当前文件标记组件进入 dirty 状态。
2. 页面不立即保存。
3. 用户点击“开始分析”时，如果任意文件标记组件 dirty，则提示 `请先保存标记`。
4. 阻止后不调用 `drawReCreateParamSettings()`，不设置 `drawing = true`，不调用 `custom_draw_plot`。
5. 用户点击“保存标记”且接口返回成功后，清除对应文件标记组件的 dirty 状态。
6. 保存失败或保存异常时，dirty 状态保持不变。

提示优先级：

1. 若存在未保存标记，提示 `请先保存标记`。
2. 若不存在未保存标记，但必选列名标记缺失，提示 `请先标记列名`。
3. 两者都通过后，进入现有分析提交流程。

## 前端设计

### FileMark 组件状态

在 `fileMark/plot.js` 或 `fileSetting.vue` 所在组件实例中新增局部状态，例如 `markDirty`。

状态变化：

- 初始值为 `false`。
- 列名选择弹窗确认后设为 `true`。
- `save()` 成功后设为 `false`。
- `save()` 失败或异常后保持 `true`。
- `reload()` 成功重新拉取列名信息后可以重置为 `false`，因为页面状态回到后端初始化结果。

对外暴露一个方法，例如：

- `hasUnsavedFileMark()`

返回当前文件标记组件是否存在未保存标记。

### PlotResultCanvas 聚合状态

`PlotResultCanvas.vue` 当前通过 `v-for` 渲染多个 `FileMark`，并使用 `ref="fileMark"`。Vue 2 在 `v-for` 中的同名 ref 会形成数组。

新增一个聚合方法，例如：

- `hasUnsavedFileMarks()`

该方法遍历 `this.$refs.fileMark`，只要任意子组件 `hasUnsavedFileMark()` 返回 `true`，整体返回 `true`。

### draw.vue 提交入口

`draw.vue` 当前通过 `this.$refs.resultCanvas` 访问结果画布组件。

在 `handleSubmit()` 的最前面增加未保存标记校验：

1. 如果 `this.$refs.resultCanvas.hasUnsavedFileMarks()` 为 `true`，提示 `请先保存标记` 并返回。
2. 再执行现有 `validateRequiredColumnMarks()`。
3. 校验通过后调用 `doSubmit()`。

这样可以保证“未保存”提示优先于“缺少必选标记”提示。

## 后端设计

本次不新增后端接口和字段。

保留 `PlotServiceImpl.customDrawPlot(...)` 现有必选列名标记兜底校验。该校验仍用于拦截请求体中确实缺少必选标记的场景。

不在本次改动中让后端自动保存标记，也不在 `custom_draw_plot` 内调用 `save_file_mark_settings`。原因是保存标记会更新 `headerInfoList` 并生成标记文件，属于明确的用户保存动作。

## 测试设计

### 前端单元测试

新增或扩展 `draw.vue` 提交流程测试，覆盖：

1. `resultCanvas.hasUnsavedFileMarks()` 返回 `true` 时，`handleSubmit()` 提示 `请先保存标记`，不调用 `doSubmit()`。
2. 存在未保存标记时，不继续触发现有 `validateRequiredColumnMarks()` 的错误提示。
3. 不存在未保存标记且必选标记缺失时，仍提示 `请先标记列名`。

新增或扩展列名标记组件测试，覆盖：

1. 选择列名确认后 dirty 变为 `true`。
2. 保存成功后 dirty 变为 `false`。
3. 保存失败后 dirty 保持 `true`。
4. reload 成功后 dirty 重置为 `false`。

### 手工验证

1. 打开需要 X/Y 轴必选列名标记的图表。
2. 使用已有保存过的 X 轴标记。
3. 新选择 Y 轴标记，但不点击“保存标记”。
4. 点击“开始分析”。
5. 预期：提示 `请先保存标记`，不进入分析中状态，不发起分析请求。
6. 点击“保存标记”成功后，再点击“开始分析”。
7. 预期：进入现有分析流程。

## 风险与边界

1. 如果用户选择列名后直接关闭页面，dirty 状态不会保存，这是符合“未保存不生效”的口径。
2. 如果保存接口返回成功但后端生成 `file-mark-*.xlsx` 实际失败，现有接口目前会返回成功；这个属于保存接口内部可靠性问题，不在本次范围内。
3. 如果未来引入自动保存或“保存并分析”，需要重新设计提交链路，避免保存请求和分析请求竞态。
4. 如果 `FileMark` 组件在某些图表中不渲染，聚合方法应返回 `false`，不影响无列名标记图表。

## 验收标准

1. 只标记但不保存时，开始分析被阻止并提示 `请先保存标记`。
2. 保存失败后，开始分析仍被阻止。
3. 保存成功后，开始分析不再因 dirty 状态被阻止。
4. 已保存但缺少后台必选标记时，仍提示 `请先标记列名`。
5. 无列名标记配置的图表不受影响。
