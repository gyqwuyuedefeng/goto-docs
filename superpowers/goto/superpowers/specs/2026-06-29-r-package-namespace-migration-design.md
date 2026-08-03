# R 包命名空间迁移设计

## 背景

`goto/task` 和 `goto/system` 当前有多条 R 调用链路仍依赖 Java 端生成 `source("...")` 或 `source('...')`，然后调用裸函数名。R 代码已逐步包化，后端应改为通过 `包名::函数名` 调用，避免运行时加载散落的 `.R` 文件。

本次迁移范围为所有 Java 生成的 R source 调用。统计和数据处理统一使用 `gotoStat` 包；工具函数使用 `gotoTools` 包；画图和图片保存使用 `gotoPlot` 包。

## 目标

- 后端不再生成任何 `source("...")` 或 `source('...')` R 命令。
- 已明确包化的工具函数改为 `gotoTools::...`。
- 画图函数统一以 `gotoPlot::...` 执行，并兼容数据库中仍保存裸函数名的历史数据。
- 统计和数据处理函数统一以 `gotoStat::...` 执行。
- 更新测试，确保不会回退到 source 加载模式。

## 非目标

- 不迁移数据库字段值。
- 不删除 `PlotType.source`、`CommonTask.source`、`DataProcessTask.source` 等模型字段，除非实现时确认没有编译影响且改动很小。
- 不重构 R 包自身，也不修改 R 函数签名。
- 不处理 `.worktrees/*` 中的历史文档或副本。

## 函数映射

工具任务：

- 拼图：`GT_MergeImages(...)` 改为 `gotoTools::mergeImages(...)`
- 图片格式转换：`fileConvert(...)` 改为 `gotoTools::fileConvert(...)`
- 长宽格式转换：`reshapeData(...)` 改为 `gotoTools::reshapeData(...)`
- 上传文件合并：`mergeTableByColumn(...)` 改为 `gotoTools::mergeTableByColumn(...)`

下载任务：

- 下载生成非 SVG 格式：`GGSAVE(...)` 改为 `gotoPlot::GGSAVE(...)`

画图任务：

- `PlotType.method` 和 `UserDraw.method` 若已包含 `::`，原样使用。
- 若未包含 `::`，运行时包装为 `gotoPlot::<method>`。
- 示例：`GT_Venn(...)` 生成 `gotoPlot::GT_Venn(...)`；`gotoPlot::GT_Pie(...)` 保持不变。

统计任务：

- `GT_BasicStat(...)` 改为 `gotoStat::GT_BasicStat(...)`
- `GT_DifferentialStat(...)` 改为 `gotoStat::GT_DifferentialStat(...)`
- `GT_Corrlation(...)` 改为 `gotoStat::GT_Corrlation(...)`
- `GT_Ropls(...)` 改为 `gotoStat::GT_Ropls(...)`

数据处理任务：

- `remove_duplication(...)` 改为 `gotoStat::remove_duplication(...)`
- `classify_variables(...)` 改为 `gotoStat::classify_variables(...)`
- `quality_assessment(...)` 改为 `gotoStat::quality_assessment(...)`
- `data_cleaning(...)` 改为 `gotoStat::data_cleaning(...)`
- `impute_missing(...)` 改为 `gotoStat::impute_missing(...)`
- `transform_data(...)` 改为 `gotoStat::transform_data(...)`
- `precess_result(...)` 改为 `gotoStat::precess_result(...)`

## 设计

### 命名空间辅助逻辑

在 Java 侧新增一个小型工具方法或复用合适的 util，提供幂等包装：

- 输入为空时返回原值，由调用方按原有流程报错。
- 输入包含 `::` 时返回原值。
- 输入不包含 `::` 时返回 `<packageName>::<functionName>`。

该方法用于画图和数据处理这类函数名来自数据库、前端或任务对象的场景，避免重复生成 `gotoPlot::gotoPlot::GT_Venn`。

### 执行器层

`RFunctionExecutor` 不再遍历 `request.getSources()` 生成 source 命令。执行顺序变为：

1. 创建 Rserve 连接。
2. 创建日志命令。
3. 构建 R 参数。
4. 添加最终函数调用命令。
5. 调用 `ROperationUtils.doCmd(...)`。

`RScriptExecutor` 不再把 `request.getSourceFiles()` 拼成 `source(...)`。Rscript 执行只负责执行 `request.getMethod()(...)` 或已带命名空间的方法。

### CommonStage 链路

`CommonStage` 当前可能在 `invokeR()` 中根据 `obtainSourceNames()` 添加 source。实现时需要定位该逻辑并移除 source 注入。`UserDrawStage`、`GenerateSampleDataStage`、`CommonTaskStage` 的 source 返回值不再作为 R 加载依据。

保守实现可以保留任务对象上的 source 数据字段，但执行时忽略它们。

### 业务生成层

工具 handler 直接生成包函数调用，并删除对应 `setSource(...)`：

- `MergeImageMessageHandler`
- `ImageConvertMessageHandler`
- `FileReshapeMessageHandler`
- `FileMergeMessageHandler`

统计 handler 删除 `Stat/...` source 设置，`StatisticsUtils` 直接生成 `gotoStat::...` 调用。

数据处理 controller 可以保留现有 method 参数传递，但不再传实际 source；`DataProcessBaseHandler` 构建 `RFunctionRequest` 时将 method 包装为 `gotoStat::<method>`，并不设置 sources。

画图调用在 `PlotTypeUtils.autoCreateParam(...)` 里包装 method 为 `gotoPlot::<method>`，保持数据库兼容。

下载调用在 `DrawPlotDownload` 中删除 `addSource(...)`，直接生成 `gotoPlot::GGSAVE(...)`。

## 错误处理

- R 包不存在、函数不存在、函数签名不匹配等错误继续由现有 Rserve/Rscript 执行结果返回。
- Java 侧不吞掉 R 错误；日志继续输出最终 R 命令，方便定位具体缺失的包或函数。
- 幂等包装只负责补包名前缀，不做函数名合法性校验，避免改变现有业务校验语义。

## 测试策略

更新或新增单元测试覆盖：

- `RFunctionExecutor` 不再生成任何 `source('...')` 命令。
- `RScriptExecutor` 不再生成任何 `source('...')` 命令。
- 工具任务生成 `gotoTools::mergeImages`、`gotoTools::fileConvert`、`gotoTools::reshapeData`、`gotoTools::mergeTableByColumn`。
- `DrawPlotDownload` 生成 `gotoPlot::GGSAVE`。
- `PlotTypeUtils.autoCreateParam(...)` 对裸函数名添加 `gotoPlot::`，对已有 `::` 的函数名不重复添加。
- `StatisticsUtils` 生成 `gotoStat::GT_BasicStat`、`gotoStat::GT_DifferentialStat`、`gotoStat::GT_Corrlation`、`gotoStat::GT_Ropls`。
- 数据处理请求最终调用 `gotoStat::<method>`，且不携带 sources。
- 全仓搜索 `source\\s*\\(`，确认 Java 生产代码不再生成 R source 命令。

建议验证命令：

```bash
mvn -pl common,task,system test
rg -n 'source\\s*\\(' goto/common/src/main/java goto/task/src/main/java goto/system/src/main/java
rg -P -n 'GT_MergeImages\\(|(?<!::)fileConvert\\(|(?<!::)reshapeData\\(|(?<!::)mergeTableByColumn\\(|(?<!::)GGSAVE\\(' goto/task/src/main/java goto/system/src/main/java goto/common/src/main/java
```

## 验收标准

- Java 生产代码不再生成 `source("...")` 或 `source('...')`。
- 已知工具、下载、画图、统计、数据处理调用均使用对应 R 包命名空间。
- 历史数据库中的裸画图函数名可继续运行。
- 已带包名前缀的画图函数名不会被重复包装。
- 相关测试通过，或未通过项有明确的外部依赖原因。
