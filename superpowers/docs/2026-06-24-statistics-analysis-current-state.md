# GOTO 统计分析模块现状说明

> 文档日期：2026-06-24
>
> 文档性质：现状基线，不是优化设计或实施计划
>
> 分析范围：
>
> - 前端：`goto-web/src/views/goto/statistics`
> - 前端容器：`goto-web/src/views/goto/index.vue`
> - 后端：`goto`

## 1. 结论摘要

当前统计分析模块已经形成一条可运行的完整链路：

1. 用户从 GOTO 顶部“统计”入口进入方法列表。
2. 前端按 `type/subType` 路由到 11 个统计方法页面之一。
3. 页面首次加载时，后端按“当前用户 + 方法类型”查询配置；没有记录时自动创建默认配置。
4. 用户上传固定命名的 Excel 文件，前端同步更新文件状态和参数配置。
5. 用户发起分析后，后端将领域对象包装为 `CommonTask` 并投递 RabbitMQ。
6. task 模块消费消息，生成 R 函数调用参数，交给通用线程池执行。
7. 任务状态写回对应业务表，并尝试通过公共 WebSocket 通知当前页面。
8. 结果文件保存到用户私有的统计目录。

系统当前支持 4 个一级分类、11 个分析方法：

| 一级分类 | type | 子方法 |
|---|---:|---|
| Basic Statistics | 1 | Basic Statistics |
| Differential Testing | 2 | One-Sample Test、Two-Sample Test、Paired-Sample Test、One-Way Anova |
| Correlation Analysis | 3 | Correlation Analysis |
| Multivarite Analysis | 4 | pca、opls、pls、oplsda、plsda |

当前实现的主要特征是“每个用户、每个统计方法保存一份长期配置，文件也固定覆盖到该方法的私有目录”。它更接近“方法工作台”，不是“每次分析产生一条独立历史任务记录”的分析平台。

## 2. 总体架构

```text
goto-web
  /goto/statistics
        |
        v
statistics/index.vue（方法入口）
        |
        v
/goto/statistics/operate/:type/:subType
        |
        v
11 个方法组件 + statisticMixins.js
        |
        +-- 查询/保存配置 --> StatisticsController --> StatisticsServiceImpl
        |
        +-- 上传/删除文件 --> 通用文件接口 --> StatisticsUpload/StatisticsDelete
        |
        +-- 发起/取消任务 --> RabbitMQ --> CommonTaskConsumer
                                      |
                                      v
                     StatisticsStart/CancelAnalyzeMessageHandler
                                      |
                                      v
                         CommonTaskThreadPool + R 执行
                                      |
                         状态表更新 + WebSocket 通知
```

主要模块职责：

| 模块 | 职责 |
|---|---|
| `goto-web` | 方法选择、文件上传、参数编辑、客户端校验、任务状态展示 |
| `goto/system` | HTTP 接口、鉴权、配置持久化、任务发布 |
| `goto/common` | 领域对象、Repository、默认值初始化、路径与 R 参数生成 |
| `goto/task` | RabbitMQ 消费、任务去重键、线程池执行、取消、状态通知 |
| R 脚本资源 | 实际统计计算，入口函数由 Java 生成 |

## 3. 前端入口与路由

### 3.1 顶层容器

`goto-web/src/views/goto/index.vue` 是 GOTO 主容器。

- 顶部导航中 `activeIndex === "2"` 表示统计模块。
- 点击“统计”执行 `menuClick("2")`，跳转 `/goto/statistics`。
- 只有 `/goto/plot` 和 `/goto/process` 显示左侧动态菜单；统计模块不显示左侧菜单。
- 容器创建公共 WebSocket，并维护 `commonWebSocketMessageNoticeMap`，按 `commonNoticeUUID` 把消息路由到具体页面组件。
- 统计列表页主动执行：
  - `setLeftMenuNotShow`
  - `setMainTopBarShow`

### 3.2 路由

| 路由 | 页面 | 用途 |
|---|---|---|
| `/goto/statistics` | `statistics/index.vue` | 展示 4 个分组、11 个方法入口 |
| `/goto/statistics/operate/:type/:subType` | `statistics/type/operate.vue` | 根据类型选择具体方法组件 |

`operate.vue` 静态导入全部 11 个方法组件，再通过多个 `v-if` 进行匹配。新增方法目前需要同时修改常量、入口页、路由承载页和后端枚举/初始化分派。

### 3.3 页面布局

- 统计首页使用固定的 250×250 方法卡片。
- 操作页主体最小宽度为 1024px，宽度 80%，最大 1500px。
- 操作页统一展示：
  - 文件上传
  - 参数设置
  - 对比组设置（部分方法）
  - 开始分析/取消分析按钮
- 当 `target.status === 3` 时显示 loading 并展示“取消分析”；其他状态显示“开始分析”。

## 4. 前端公共逻辑

公共逻辑集中在：

`goto-web/src/views/goto/statistics/components/statisticMixins.js`

### 4.1 页面初始化

每个方法组件 mounted 后调用：

```text
obtainStatisticsByTypeAndSubType({ type, subType })
  -> statistics = res.data
  -> 子组件 setTarget()
```

后端返回统一 `Statistics` 包装对象，但实际只填充当前一级分类的一个 map：

- `mapBasicStatistics`
- `mapDifferentialTesting`
- `mapCorrelationAnalysis`
- `mapMultivariteAnalysis`

map key 是子类型/module 的字符串形式，例如 `"1"`。

### 4.2 文件上传与删除

上传使用通用文件接口，业务参数放在 JSON 字符串中：

```json
{
  "type": 2,
  "fileType": 2,
  "subType": 3,
  "phenoCol": "group"
}
```

文件类型：

| fileType | 含义 |
|---:|---|
| 1 | INPUT |
| 2 | PHENO |
| 3 | INPUT2 |

上传成功后，前端直接修改 `target.statFileStatus`。上传 PHENO 文件时，还会把响应中的 `phenoFileHeader` 写入当前对象。

删除文件后，前端清除对应文件存在状态，并持久化当前统计对象。

对于 Differential Testing 和 Multivarite Analysis，重新上传或删除 PHENO 文件时会清空：

- `idCol`
- `phenoCol`
- `compares`

### 4.3 参数持久化

绝大多数表单控件在 `@modify` 时立即调用：

```text
modifyStatisticsObject(target)
  -> POST /api/statistics/update_statistics_object
```

因此当前模式不是“点击保存”，而是字段变更后即时保存整个领域对象。

后端会覆盖前端传入的 `userId`，并用 `id + 当前用户`（部分表额外加 module/method）验证数据归属。

### 4.4 分组列表

选择 `phenoCol` 后，以下方法会调用 `obtain_group_list`：

- Two-Sample Test
- Paired-Sample Test
- One-Way Anova
- pca
- oplsda
- plsda

后端读取 PHENO Excel，通过表头找到目标列，按原顺序去重并返回 `LinkedHashSet<String>`。

`opls` 只选择一个 `phenoCol`，不展示 compares；`pls` 允许选择多个 `phenoCol`，也不展示 compares。

### 4.5 发起和取消任务

开始分析：

1. 子组件执行自己的 `parameterValidation`。
2. mixin 按一级类型组装 `StatisticsParams`。
3. 调用 `/api/statistics/start_analyze`。
4. HTTP 成功后，前端立即把 `target.status` 设置为 3。

取消分析：

1. 按一级类型组装同样的领域对象。
2. 调用 `/api/statistics/cancel_analyze`。
3. 前端不立即改变状态，等待异步通知。

### 4.6 状态约定

前端代码按以下状态解释：

| status | 含义 |
|---:|---|
| 1 | 正常/可提交 |
| 2 | 排队 |
| 3 | 分析中 |
| 4 | 成功 |
| 5 | 失败 |
| 6 | 拒绝 |
| 7 | 取消 |

页面只对状态 3 做显著 UI 区分，其他终态没有结果区、错误详情区或历史记录区。

## 5. 统计方法与参数

### 5.1 Basic Statistics

文件：

- `sacurine_dataset.xlsx`
- `sacurine_metadata.xlsx`

参数：

- `idCol`
- `phenoCol`：单列
- `rmNA`
- `method`：统计量复选列表

内置统计量：

- Min
- Max
- Median
- Mean
- SD
- RSD
- SEM
- Var
- Range
- Midrange
- Skewness
- kurtosis

默认选中 Mean、Median、SD。已保存的默认项选中状态会与新版默认列表合并，自定义项会被保留。

前端校验：

- INPUT、PHENO 文件必须存在
- `rmNA` 非空
- 至少选择一个统计方法

默认结果文件名：`basic_stats.xlsx`

R 入口：`GT_BasicStat(...)`

### 5.2 Differential Testing

公共文件：

- `sacurine_dataset.xlsx`
- `sacurine_metadata.xlsx`

公共领域字段：

- `idCol`
- `phenoCol`
- `u0Col`
- `u0`
- `compares`
- `method`
- `fdr`
- `normalityCheck`
- `varianceCheck`
- `paired`

方法矩阵：

| 方法 | module | 主要参数 | 对比组 |
|---|---:|---|---|
| One-Sample Test | 1 | idCol、u0Col、u0、method、fdr、normalityCheck | 无 |
| Two-Sample Test | 2 | idCol、phenoCol、method、fdr、normalityCheck、varianceCheck | 每组恰好 2 个类别 |
| Paired-Sample Test | 3 | idCol、phenoCol、method、fdr、normalityCheck，`paired=T` | 每组恰好 2 个类别 |
| One-Way Anova | 4 | idCol、phenoCol、method、fdr、normalityCheck、varianceCheck | 每组至少 3 个类别 |

method 候选：

- One/Two/Paired：`auto`、`t-test`、`u-test`
- One-Way Anova：`auto`、`anova`、`kw`

FDR 候选：

- none
- BH
- BY
- holm
- hochberg
- hommel
- bonferroni

正态性检验候选：

- none（仅部分非参数条件下提供）
- L-K-S
- S-W
- K-S

方差检验候选：

- none（部分条件下提供）
- Levene
- Bartlett

这 4 个组件没有覆盖 mixin 的 `parameterValidation`，当前前端提交前实际不会执行方法级必填校验。

默认结果文件名：

- `one_sample_test.xlsx`
- `two_sample_test.xlsx`
- `paired_sample_test.xlsx`
- `one_way_anova.xlsx`

R 入口：`GT_DifferentialStat(...)`

### 5.3 Correlation Analysis

文件：

- INPUT：`sacurine_dataset.xlsx`
- INPUT2：`sacurine_pheno.xlsx`

参数：

- `method`

method 候选由页面提供，默认值为 `spearman`。

前端校验：

- 两个输入文件必须存在
- method 非空

默认结果文件名：`sacurine_meta_vs_pheno_spearman.xlsx`

R 入口：`GT_Corrlation(...)`

注意：Java/R 调用名当前拼写为 `Corrlation`，属于既有协议，修改时必须同步 R 端。

### 5.4 Multivarite Analysis

代码和数据库均使用拼写 `Multivarite`，不是标准英文 `Multivariate`。这是现有跨层命名协议。

公共文件：

- `sacurine_dataset.xlsx`
- `sacurine_metadata.xlsx`

公共参数：

- `idCol`
- `phenoCol`
- `compares`
- `method`
- `scaling`
- `cv`
- `permI`

默认值：

- `scaling = standard`
- `cv = 7`
- `permI = 200`

scaling 候选：

- standard
- center
- pareto
- none

方法矩阵：

| 方法 | module | PHENO 要求 | phenoCol | compares |
|---|---:|---|---|---|
| pca | 1 | 可选 | 单列 | 每组至少 2 类，可多类 |
| opls | 2 | 必须 | 单列 | 不展示 |
| pls | 3 | 必须 | 多列 | 不展示 |
| oplsda | 4 | 必须 | 单列 | 每组恰好 2 类 |
| plsda | 5 | 必须 | 单列 | 每组至少 2 类，可多类 |

默认结果文件名：

- `pca_result.xlsx`
- `opls_result.xlsx`
- `pls_result.xlsx`
- `oplsda_result.xlsx`
- `plsda_result.xlsx`

R 入口：`GT_Ropls(...)`

## 6. 后端 HTTP 接口

Controller：

`goto/system/src/main/java/com/freedom/rest/StatisticsController.java`

统一前缀：`/api/statistics`

| 接口 | 权限 | 作用 |
|---|---|---|
| POST `/obtain_statistics_by_type_and_sub_type` | `statistics:obtain_statistics_by_type_and_sub_type` | 查询或初始化当前用户的方法配置，并补充文件状态 |
| POST `/update_statistics_object` | `statistics:update_statistics_object` | 保存整个统计领域对象 |
| POST `/start_analyze` | `statistics:start_analyze` | 发布开始分析任务 |
| POST `/cancel_analyze` | `statistics:cancel_analyze` | 发布取消任务 |
| POST `/obtain_group_list` | `statistics:obtain_group_list` | 从 PHENO 文件读取指定列的去重值 |

`start_analyze` 额外配置：

- 1 秒最多 30 次的接口限流
- 同一防抖键 3 秒最多 1 次

前端 API 文件还保留 `obtainAllStatistics()`，请求 `/obtain_all_statistics`，但当前 Controller 没有该接口，业务页面也未使用它。

## 7. 配置初始化与持久化

### 7.1 初始化机制

应用启动时 `CommonInitializer` 建立：

```text
StatisticsUtils.STATISTICS_INIT_CALLABLE
  type
    -> subType
      -> 初始化回调
```

首次进入某方法时：

1. Repository 按当前用户和 module 查询。
2. 不存在则调用注册的初始化回调。
3. 立即插入一条默认配置记录。
4. 返回给前端。

这意味着“浏览一个方法页面”就会产生数据库记录。

### 7.2 数据表

| 表 | 对应对象 | 区分方法的字段 |
|---|---|---|
| `app_basic_statistics` | `BasicStatistics` | `user_id + module` |
| `app_differential_testing` | `DifferentialTesting` | `user_id + module` |
| `app_correlation_analysis` | `CorrelationAnalysis` | `user_id + module` |
| `app_multivarite_analysis` | `MultivariteAnalysis` | 查询用 `user_id + module`，更新校验用 `id + user_id + method` |

复杂数组通过 JPA AttributeConverter 存入单字段：

- `List<String>`
- `List<List<String>>`
- `List<BasicStatisticsMethod>`

临时响应字段不入库：

- `statFileStatus`
- `phenoFileHeader`
- `groupList`
- `fileSettings`
- `phenoFileSettings`

## 8. 用户文件目录

基础路径来自：

```text
用户命名空间目录 / statistics
```

标准结构：

```text
statistics/
  <一级类型名>/
    <子类型名>/
      upload/
        固定输入文件名.xlsx
      result/
        固定结果文件名.xlsx
```

示例：

```text
statistics/
  DifferentialTesting/
    Two-Sample Test/
      upload/
        sacurine_dataset.xlsx
        sacurine_metadata.xlsx
      result/
        two_sample_test.xlsx
```

当前模型的影响：

- 同一用户、同一方法只有一套当前输入文件。
- 再次上传会替换固定路径文件。
- 同一方法的结果文件也使用固定文件名。
- 没有天然的运行批次、版本、历史参数快照和历史结果目录。

## 9. 异步任务执行

### 9.1 发布

`StatisticsServiceImpl.startAnalyze`：

1. 根据 `type` 选择目标领域对象。
2. 包装为 `CommonTask`。
3. 设置：
   - `taskType = STATISTICS_START_ANALYZE(2)`
   - 当前用户
   - 领域对象 data
   - `statusChange = true`
   - `commonNoticeUUID`
4. 发布到 RabbitMQ。

取消任务使用 `STATISTICS_CANCEL_ANALYZE(3)`。

### 9.2 消费和分派

`CommonTaskConsumer` 从公共任务队列消费，根据 `taskType` 查找 `MessageHandler`：

- `StatisticsStartAnalyzeMessageHandler`
- `StatisticsCancelAnalyzeMessageHandler`

开始任务的线程池 key：

```text
userId_taskType_statisticsType_subType_databaseId
```

示例：

```text
100_2_4_1_25
```

取消任务构造相同 key，但强制使用开始任务类型 2，以定位正在运行的任务。

### 9.3 R 调用

handler 根据领域对象类型选择 R 资源：

| 对象 | R source |
|---|---|
| BasicStatistics | `Stat/basic_statistics` |
| DifferentialTesting | `Stat/differential_analysis` |
| CorrelationAnalysis | `Stat/correlation_functions` |
| MultivariteAnalysis | `Stat/ropls_functions` |

Java 通过 `@AutoParamField` 反射生成 R 函数参数文本，文件路径在执行前按当前用户目录重新计算。

### 9.4 状态持久化

`CommonTaskThreadPool` 根据领域对象的 JPA 表名执行通用 SQL：

```sql
update <table> set status = :status where id = :id
```

因此统计任务没有独立任务表；运行状态直接写在“方法配置记录”的 `status` 字段上。

直接后果：

- 同一配置记录不能自然表达多次运行历史。
- 当前状态只代表最近一次或当前一次任务。
- 排队、执行、失败原因和运行耗时没有统计专属持久化模型。

## 10. 当前结果能力

后端会计算并返回 `resultFileExists`，结果文件按方法保存为 `.xlsx`。

但当前统计方法页面：

- 没有展示 `resultFileExists`
- 没有结果下载按钮
- 没有结果预览
- 没有结果摘要
- 没有任务历史
- 没有失败详情

因此当前页面的可见闭环主要是“配置 + 提交 + loading/通知”，结果消费能力尚未在本模块内完成。

## 11. 已确认的问题与风险

### 11.1 高优先级功能问题

#### 关联分析参数保存字段大小写错误

`statisticMixins.js` 保存 Correlation Analysis 时组装：

```js
data.CorrelationAnalysis = target
```

后端字段是：

```java
private CorrelationAnalysis correlationAnalysis;
```

其他请求均使用小写开头的 `correlationAnalysis`。在默认 Jackson 大小写敏感配置下，此请求会导致后端读取为空并返回“参数异常”，因此关联分析 method 的即时保存可能失效。

#### 统计任务没有把 `commonNoticeUUID` 放入开始/取消请求

公共 WebSocket mixin 会生成并注册 `commonNoticeUUID`，主容器也严格按该 UUID 路由消息；但 `statisticMixins.js` 组装 `startAnalyze` 和 `cancelAnalyze` 请求时没有写入该字段。

后端虽然会把 `StatisticsParams.commonNoticeUUID` 复制到 `CommonTask`，实际收到的值可能为 null，导致统计任务通知无法命中当前页面注册的上下文。

#### Differential Testing 缠绕在默认空校验

4 个 Differential Testing 组件都没有实现自己的 `parameterValidation`，因此使用 mixin 的默认 `return true`。

后端对 Basic、Differential、Correlation 的参数校验代码也被注释，仅 Multivarite Analysis 执行服务端校验。缺文件、缺必填参数或非法 compares 可能进入异步任务后才失败。

### 11.2 数据一致性与安全边界

- `updateStatisticsObject` 对未知 type 或归属校验失败时仍可能返回 success，调用方无法区分“已保存”和“未保存”。
- 更新接口直接保存前端传回的完整实体，允许修改超出当前表单范围的持久化字段。
- `startAnalyze` 直接使用前端提交的整个实体，没有按 `id + 当前用户` 从数据库重新加载可信配置。
- Basic、Differential、Correlation 的服务端参数校验已注释，主要依赖前端。
- `obtainStatisticsByTypeAndSubType` 对非法 type/subType 可能返回空包装对象，而不是明确错误。
- 结果与上传文件采用固定路径，覆盖行为缺少显式版本控制。

### 11.3 前端可维护性

- 11 个页面存在大量重复模板、上传区、参数控件和校验代码。
- `operate.vue` 通过 11 个静态组件和 `v-if` 分发，不是配置驱动。
- 前后端分别维护 type/subType 常量，存在漂移风险。
- 多处使用 magic number 和字符串：
  - type 1~4
  - subType 1~5
  - status 1~7
  - 固定文件名
  - R source 路径
- 页面中保留大量 `console.log`。
- Excel accept 字符串含拼写错误：`pplication/vnd.ms-excel`。
- `TaskType` 已导入 mixin，但实际注册逻辑被注释。
- `obtainAllStatistics` 是失效/遗留 API。
- `Multivarite`、`Corrlation`、`phenoFileNamee` 等拼写已经跨数据库、Java、前端或 R 形成协议，不能简单局部重命名。

### 11.4 用户体验

- 首页方法卡片是固定尺寸，响应式能力有限。
- 操作页最小宽度 1024px，小屏适配弱。
- 自动保存没有保存中、成功、失败重试状态。
- 快速修改多个字段会产生多个全实体更新请求，没有防抖或顺序控制。
- HTTP 提交成功后直接把状态设置为分析中，没有区分“已接收”和“真正开始执行”。
- 除 loading 和通知外，没有清晰的排队、进度、成功结果和错误详情展示。
- 取消按钮只在状态 3 显示，状态 2 排队阶段不能从页面取消。

### 11.5 扩展成本

新增一个一级分类通常需要修改：

- 前端常量
- 统计首页
- `operate.vue`
- `statisticMixins.js` 的多处分支
- API 请求对象映射
- 后端 `StatisticsType`
- `StatisticsParams`
- `Statistics` 返回包装
- Controller/Service 分支
- Domain/Repository
- 初始化 Utils
- 文件上传/删除分派
- R 参数生成
- task handler 类型分派
- WebSocket `simpleData`

新增同一分类下的子方法也需要同步前端常量、入口、组件、后端枚举、初始化回调、路径和默认结果文件名。

## 12. 测试现状

在前端和后端测试目录中未发现统计分析模块的专属自动化测试。

当前缺少至少以下覆盖：

- type/subType 映射一致性
- 首次访问自动初始化
- 参数归属与越权更新
- 文件上传/删除后的状态刷新
- PHENO 表头与 groupList 提取
- 各方法参数校验
- RabbitMQ 提交和取消
- 线程池 key 唯一性
- 状态写回
- WebSocket UUID 路由
- R 参数字符串快照
- 结果文件生成与下载

## 13. 后续优化时的建议边界

后续设计前应先确定产品模型：

### 方案 A：继续维持“方法工作台”

适合只保留每个方法的当前配置和最新结果。

优先事项：

- 修复请求字段和 WebSocket UUID
- 补齐服务端校验
- 配置驱动渲染方法页面
- 提取通用统计表单框架
- 增加结果下载和错误展示
- 明确自动保存状态和请求防抖

### 方案 B：升级为“分析任务平台”

适合需要历史、复现、审计、并行任务和结果管理。

需要引入：

- 独立 analysis/job 记录
- 参数快照
- 输入文件版本或对象存储引用
- 每次运行独立结果目录
- 任务状态时间线
- 日志和失败原因
- 结果清单和下载 API
- 重跑、复制配置、归档和清理策略

这两个方向的数据模型差异很大。开始优化前应先决定是否需要“历史运行记录”，否则仅重构页面可能很快再次遇到模型瓶颈。

## 14. 关键源码索引

前端：

- `goto-web/src/views/goto/index.vue`
- `goto-web/src/router/routers.js`
- `goto-web/src/views/goto/statistics/index.vue`
- `goto-web/src/views/goto/statistics/type/operate.vue`
- `goto-web/src/views/goto/statistics/components/statisticMixins.js`
- `goto-web/src/constant/statisticsConstant.js`
- `goto-web/src/api/statistics.js`
- `goto-web/src/mixins/commonWebSocketOnMessageNotice.js`

后端 HTTP 与服务：

- `goto/system/src/main/java/com/freedom/rest/StatisticsController.java`
- `goto/system/src/main/java/com/freedom/service/StatisticsService.java`
- `goto/system/src/main/java/com/freedom/service/impl/StatisticsServiceImpl.java`

后端领域与工具：

- `goto/common/src/main/java/com/freedom/model/Statistics.java`
- `goto/common/src/main/java/com/freedom/model/StatisticsParams.java`
- `goto/common/src/main/java/com/freedom/model/StatisticsFileParams.java`
- `goto/common/src/main/java/com/freedom/domain/*Statistics.java`
- `goto/common/src/main/java/com/freedom/domain/DifferentialTesting.java`
- `goto/common/src/main/java/com/freedom/domain/MultivariteAnalysis.java`
- `goto/common/src/main/java/com/freedom/util/StatisticsUtils.java`
- `goto/common/src/main/java/com/freedom/util/*StatisticsUtils.java`
- `goto/common/src/main/java/com/freedom/util/DifferentialTestingUtils.java`
- `goto/common/src/main/java/com/freedom/util/MultivariteAnalysisUtils.java`
- `goto/common/src/main/java/com/freedom/runner/CommonInitializer.java`

异步任务：

- `goto/task/src/main/java/com/freedom/bean/mq/rabbitmq/CommonTaskConsumer.java`
- `goto/task/src/main/java/com/freedom/handler/StatisticsStartAnalyzeMessageHandler.java`
- `goto/task/src/main/java/com/freedom/handler/StatisticsCancelAnalyzeMessageHandler.java`
- `goto/task/src/main/java/com/freedom/thread/CommonTaskThreadPool.java`
- `goto/task/src/main/java/com/freedom/thread/CommonTaskRunnable.java`

## 15. 文档使用约定

本文件是 2026-06-24 的代码现状基线。

后续新增优化设计文档时，建议继续放在：

```text
/mnt/f/IdeaProjects/goto-software/goto-docs/superpowers/docs
```

并使用日期前缀：

```text
YYYY-MM-DD-statistics-analysis-<topic>.md
```

后续要求“读取最新统计分析设计文档”时，应优先按日期选择该目录下最新的 `statistics-analysis-*` 文档，同时以本现状基线核对设计是否仍适配当前代码。
