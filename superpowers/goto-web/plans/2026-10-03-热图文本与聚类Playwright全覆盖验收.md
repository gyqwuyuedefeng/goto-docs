---
change_set_id: CS-20260926-7a2fb43b-e186-4328-9cab-9d6fe23081ad
related_change_set_ids:
  - CS-20260909-8db76639-0c29-4ef0-a223-5850749acbf0
document_type: acceptance_report
verified_at: 2026-10-03T14:07:08+08:00
---

# 热图文本与聚类 Playwright 全覆盖验收

## 结论

全部 8 个当前热图模板已执行验收；8/8 个模板完成全部界面检查，共 1344 组界面检查、336/336 个必需覆盖项。整体通过 0/8；真实矩阵检测存在 160 次失败，因此不能宣称全覆盖测试全部通过。

本轮只补充可重复验收脚本、真实 Excel 生成器和记录，没有新增生产参数行为。此前前端联动实现已在基线；本轮没有发送完整绘图请求，也不以页面检查结果替代历史完整 R 绘图失败结论。

## 实际版本和环境

- 实际测试提交：`003b1bcd4d4be09c8f6486a1cdbb00e8ea8903a1`；浏览器使用该基线工作区，只有受控登录页的临时替换，结束后按原字节与 Git 差异检查恢复。
- 脚本 SHA-256：`a6ca7d07fdca4b1f5ceb57f43ab5acd5e8a25fe7c8303399601d76342720d611`；生成器 SHA-256：`e6ae5820052d34cf9171096702ff487ed9274f885772a42f0c5f20c56596b400`。
- 真实前端：`http://127.0.0.1:43512`；home + env=local system：`http://127.0.0.1:44476`；本次隔离 Redis 端口：`43102`，DB1；Rserve 为本机既有依赖。
- `reuseExistingServer=false`；最多三个独立浏览器 context，每个模板自己的新建记录和 UUID 文件目录；不使用 task、Redis Stream 或共享输出文件。Cookie 只在内存，不归档认证状态。
- 修改与删除记录仅接受受控 plot_data 接口的本轮成功新建凭据；快照或历史缓存不能授予写入权限。
- 当前模板来自本次 home 数据库实际查询 app_plot_group、app_plot_type、sys_dict、sys_dict_detail；group13 的 8 模板 ID 与字典 86 的 none/raw/label、字典 28 的 raw/star 已核对。

## 各模板结果

| 模板 ID | 页面检查 | 必需项 | 检测成功 | 检测失败 | 整体 |
| --- | ---: | ---: | ---: | ---: | --- |
| 1 | 168 | 42/42 | 69 | 20 | 未通过 |
| 167 | 168 | 42/42 | 69 | 20 | 未通过 |
| 168 | 168 | 42/42 | 69 | 20 | 未通过 |
| 169 | 168 | 42/42 | 69 | 20 | 未通过 |
| 171 | 168 | 42/42 | 69 | 20 | 未通过 |
| 247 | 168 | 42/42 | 69 | 20 | 未通过 |
| 286 | 168 | 42/42 | 69 | 20 | 未通过 |
| 287 | 168 | 42/42 | 69 | 20 | 未通过 |

## 需求与实测场景

| 要求 | 真实页面验证 |
| --- | --- |
| 取消标签控制历史标记 Tab | 未标记、主次标记、各自取消、换文件及重载核对固定 Tab 状态；当前模板无历史标记 Tab |
| 取消所有单元格旧显示原始值开关 | 主值、次值、对角线组扫描实际控件；当前模板无 showNumber 控件 |
| 主次标签未标禁用 | 下拉 disabled、标记后可选、取消后重新禁用且已选 label 回 none，两侧独立 |
| none 隐藏后续项 | 枚举文本类型之后所有参数逐个检查实际 DOM hidden；前面样式仍可见 |
| 主次 raw/label 判断精确来源 | 主值、次值、主标签、次标签分别覆盖数值/文字，主次来源相反、混合内容、零、数值字符串、空值及全空 |
| 数值格式互斥 | raw 显示小数位隐藏阈值，star 相反；非数值三项隐藏；字体独立 |
| 主次组合 | 每种来源分别执行 none/raw/label 的 3×3 全部九种组合，并检查另一侧配置不串改 |
| 重载及隐藏保留 | 主次 none/raw/label、numeric raw/star 每种状态先持久化自己的记录再重载；实际编辑小数位 3/4，保留当前字体与阈值配置 |
| 轴去重不足三项 | 新建记录真实未标记覆盖0；实际上传1/2/3/4的轴变量文件，重复轴行验证按去重数量；对应FALSE/disabled，普通点击无法开启，恢复仅解禁 |
| 缺必填和恢复 | 清空X/Y分别点击保存，提示请标记且不发save/detect；开始分析提示请先保存标记且不绘图；恢复标记后真实save/detect200 |
| 首载聚类值 | 3/3对称统一聚类TRUE及4/3行列聚类TRUE，持久化重载后保持 |

普通参数编辑按应用现有流程保留在页面内存；“保存后重载”通过应用已有参数保存 API 持久化本次自有记录的10个文本参数和3个聚类开关作为前置，严格验证ID和HTTP/code200，再刷新逐字段比较。没有增加或宣称生产自动保存。

## 真实检测失败

检测成功 552 次，失败 160 次。失败按数据阶段分组：

- 主文本次数值且标签来源相反：88 次。
- 四来源数值与文字混合：72 次。

应用检测接口返回 HTTP 400；直接公开 R 调用的数值主值对照成功，文字和混合文字主值失败。仅证明公开调用失败的关联，具体根因未确认；未知异常没有回显或归档。不读取、解释或修改机密包实现。

gotoBase 源码仓元数据的 Windows Git fetch 成功确认本地与 origin/main 均为 `cd70a32d02da884e8769c57cd63547870776206d`，没有新远端提交；版本元数据为0.1.0。本轮没有重复重装。

## 执行命令与附件

- 生成：`python3 tests/e2e/heatmap-text-cluster.fixtures.py /tmp/heatmap-playwright-full-shafggv6`。
- 全量真实驱动：`node /tmp/heatmap-playwright-full-shafggv6/driver-full.cjs`。
- 仓库脚本：goto-web/tests/e2e/heatmap-text-cluster.playwright.cjs；14文件生成器：同目录 heatmap-text-cluster.fixtures.py。
- 每模板安全JSON与截图：本机临时目录 full-template-ID；总结果 full-results.json、版本 full-runtime-evidence.json、公开诊断 public-probe-result.json。
- 先前两次前置纠正：普通参数没有自动持久化，使用已有API建立已保存状态；缺必填保存根本不发API，改验真实必填拦截。另空工作簿不能加载，已保留 failed-empty-preparation 的失败材料，以新建未标记状态覆盖零项。上述失败尝试未计为全量通过。
- 代码提交70693e8、a677d55、003b1bc携带相同Change-Set-Id；只影响验收脚本和生成器。没有新的Java或生产源码修改，因此本轮不重复编译与构建作为额外验收。

## 清理与遗留

本次20条自有测试记录（含失败前置尝试）均已通过应用删除，MySQL只读复核残留0条；20个UUID目录和本轮公开诊断临时目录已删除。前端、system、隔离Redis均Ctrl-C关闭，43512、43966、44476、43102均释放；登录页SHA-256恢复一致且已暂存/未暂存差异检查通过，临时Redis密钥/配置和浏览器认证状态已清理。关闭system时退出130，最终端口复核已确认释放。保留真实检测失败并将变更集整体维持 blocked；需要公开检测链路修复或确认文字主值的对外契约后重新运行同一脚本。历史完整 R 绘图失败仍需对应出图验收。

代码已合入并推送 goto-web/master=003b1bcd4d4be09c8f6486a1cdbb00e8ea8903a1。文档由独立任务分支归档到远端main；本地文档main有其它正在执行的SVG修改，保留其现场，不合并到该工作区。测试结论仍为未通过；没有部署生产功能。


## 用户要求修复后的最小公开调用排查

本次将公开函数执行与返回解析分开记录。数值对照完成公开调用并解析为合法 JSON；两类失败明确发生在 `public_call`，不属于 Java Map/JSON 返回兼容问题。错误原文未回显，只经本机固定关键词分类为 `numeric_related_error`。此分类只说明异常涉及数值，并不揭示包内部实现或准确错误语句。

进一步用相同文件仅替换 Main 内容作对照；其余行、列、次值和标签全部保留，两个失败文件均恢复成功。另生成只有 X/Y/Main 的 3×3 对称数据，排除附加列、中文内容和不对称数据的干扰：

| 最小文件 | Main 内容 | 公开调用与契约结果 |
| --- | --- | --- |
| numericMinimal.xlsx | `(X序号+1)×(Y序号+1)` 数值 | 成功，true/raw |
| textMinimal.xlsx | 相同数值前加 `category_` | public_call失败，numeric_related_error |
| mixedMinimal.xlsx | 对角线数值，其余 `category_` 文本 | public_call失败，numeric_related_error |
| textNumeric_numericMain.xlsx | 原 textNumeric，只替换 Main 为数值 | 成功，true/raw |
| mixedText_numericMain.xlsx | 原 mixedText，只替换 Main 为数值 | 成功，true/raw |

这已把当前失败定位到公开检测函数处理非数值主值的行为。Java 按用户约定传入 file/col_col/row_col/value_col，不读取或转换单元格；调用端没有已证实可修复的传参错误。2026-10-03 本次 Windows git.exe fetch origin 退出0，gotoBase HEAD 与 origin/main 均为 cd70a32d02da884e8769c57cd63547870776206d，没有新包提交可获取。版本0.1.0不是安装提交的证明；此轮没有重复重装。

### 具体修复路径与验收条件

1. 由 gotoBase 维护者确认并补齐 `check_symmetry` 对纯文本、数字与文本混合主值的公开输入支持。当前前端需求包含主值非数值时隐藏数值格式控制，因此不能通过删掉该覆盖项宣称通过。若包对外约定只支持数值，则需用户确认业务范围和相应预期错误处理，不能自行把它当作成功。
2. 支持这些输入时，对称性须按真实输入判断，返回合法 isSymmetric 布尔值和 dataType；不能在前后端强制false、用0替换文字或跳过检测。支持文字的具体dataType约定仍待维护者明确。数值最小对照的true/raw结果须保持。
3. 维护者提交包更新后，沿用用户已授权的外部工具更新并重装，检查安装状态；重新以公开调用验证上述最小文件。当前项目规范禁止 Agent 读取或修改 gotoBase 内部源码，因此本轮不能代替包维护者实施内部修复。
4. 最小用例通过后，重跑同一受控真实页面脚本，先验证一个模板，再执行8模板全覆盖；保存标记、检测、重载均应按对外约定成功。历史完整R出图仍须单独执行真实出图验证。

公开复现调用（不涉及包内部实现）：

```r
gotoBase::check_symmetry(
  file = "textMinimal.xlsx",
  col_col = "X", row_col = "Y", value_col = "Main"
)
```

本轮没有修改生产前端/Java代码，也没有重跑尚无修复的全覆盖脚本。没有启动新应用服务、替换登录页或写入数据库。公开探针使用本次独立临时业务目录，调用后已删除；现有Rserve容器保留。安全阶段结果保存在 /tmp/heatmap-detection-debug-ve6603a0/phased-public-probe-result.json 和 minimal-public-probe-result.json。整体仍blocked，尚未修复或验收通过。
