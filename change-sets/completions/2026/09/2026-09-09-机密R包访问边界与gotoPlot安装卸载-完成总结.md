---
document_type: change_set_completion
change_set_id: CS-20260909-49bb4299-7794-4c48-b094-dac7c5f10ffc
related_change_set_ids: []
outcome: completed
closed_at: 2026-09-09T12:04:41+08:00
---

# 机密 R 包、安装卸载与本地测试规范完成总结

## 结果摘要

已在 goto-software 项目规范明确四个机密 R 包的访问边界，补齐 gotoPlot 单包工具、批量重装与容器挂载配置，并完成本地测试服务、编译依赖、Java/R 路径、生产资源使用和并行验收细则。规范和工具均已快进合入本地主仓，未推送远端。生产环境所有测试写入已按用户授权登记为本项目对全局数据库客户端只读规则的显式覆盖。本次未读取四个源码目录内部内容，未安装、卸载真实包、重建容器或执行生产写入。

## 原因与目标

用户明确 gotoTools、gotoBase、gotoPlot、gotoStat 属于公司机密，只允许安装、卸载操作；外部部署工具此前缺少 gotoPlot 的两个单包入口。验收要求是建立不可通过搜索、索引、容器、Git 历史或子 Agent 绕过的规范边界，并使现有工具支持第四个包。

同一会话中用户确认继续完善项目测试规范，并明确“生产允许所有测试写入”。据此补齐运行环境及依赖规则，记录持续授权，沿用同一 Change-Set-Id 和本总结，不新建第二份收尾记录。

## 设计决策

保密约束集中在 goto_software_spec.md，触发索引负责发现。允许安装器在本机内部完成必需读取，但不允许把源码或潜在源码日志送入 Agent；只允许状态、版本与不含源码的错误摘要。

沿用现有 Rscript 单包模式，补两个入口并将 gotoPlot 放在原三包之后安装、之前卸载。同步增加挂载配置以使固定容器路径可用；缺失源码目录时在 Docker 操作之前退出。未新增安装框架或源码扫描器。

本地测试规则集中在 local-testing.md，入口负责摘要和覆盖登记。生产测试授权覆盖应用、脚本和数据库客户端所需写入，在本项目测试范围内持续有效，不重复逐次确认；不推导其他项目或非测试运维权限。共享消费者组和输出路径的真实验收依次执行，不引入新的运行治理服务。

## 实际变更

- 规范仓：项目入口新增常驻机密边界、外部工具清单和容器重建权限说明；全局索引新增 goto-software R 包维护路由。
- 规范仓：新增 goto-software/local-testing.md 和生产测试写入覆盖登记；更新测试路由、限定覆盖作用域的合同测试，并修正 Maven 公共正文中过时的 hsmap-only 项目范围描述，未修改包装脚本或全局数据库只读正文。
- 部署仓：新增 `.bash/install_gotoPlot.sh`、`.bash/remove_gotoPlot.sh`；更新四包批量入口及 `wsl_rbase_start.sh` 的挂载；扩展两个既有脚本测试。
- 文档仓：新增设计、实施计划及本总结，均使用同一 Change-Set-Id。
- 四个机密包目录只检查是否存在，没有读取内部代码、元数据、Git 历史、diff 或索引；没有改动 R 包实现、运行容器或共享生产资源。

## 验证结果

- 基线：两个原有 Bash 测试通过。
- RED：新增合同后，单包测试因缺少 install_gotoPlot.sh 失败；批量测试因缺少第四包源码预检而失败。
- GREEN：`bash tests/bash/test_r_package_scripts.sh` 和 `bash tests/bash/test_reinstall_goto_r_packages.sh` 均通过。模拟 Docker 覆盖四包执行顺序、预检先于卸载、gotoPlot 语法/卸载/安装失败与最终验证失败时立即停止，保持原始退出码。
- 缺失挂载检查使用临时虚构目录，验证失败前不调用 Docker、不创建空源码目录；不使用真实机密目录作为测试夹具。
- `bash -n` 检查两个 Bash 入口和两个测试脚本通过。容器 Rscript 以 `--vanilla` 加载 `invisible(parse(file = stdin()))`，逐个只解析八个外部包装脚本，全部通过，没有执行安装或卸载。
- 规范仓 `python3 code/01_System_Core/scripts/validate_rules.py --root code` 通过；`python3 -m unittest discover -s code/01_System_Core/tests -p test_validate_rules.py` 的 91 项测试通过。
- 两仓均使用 `git merge --ff-only` 集成，无冲突；合入后重新运行两个 Bash 测试、规范校验及 91 项规则测试，全部通过。diff 空白检查通过。
- 本地测试规范补充采用先改合同测试再实现：初始三项失败分别对应缺失细则、覆盖声明和 Maven 旧描述；补齐后规范校验、全部 92 项规则测试和 diff 检查通过。
- 补充规则从 `609389e` 快进合入 `14d86f9` 后，在本地主仓重跑规范校验与全部 92 项规则测试，再次通过。人工核对服务、编译、路径、生产资源和并行验收五项齐备，机密边界未改变，其他项目未取得新授权。

## 偏差、风险与遗留

无未经授权的范围偏差。运行规范及生产测试写入许可由用户后续明确补充，同一执行链已完成对应规范工作。未进行真实包安装或卸载，因此不声称真实包依赖、Rserve 库路径或安装结果已经验证；安装顺序保留原三包顺序并追加 gotoPlot，未读取机密 DESCRIPTION 证明依赖关系。实际安装若因依赖失败，应按不含源码的摘要交由用户处理。

本轮交付是规范，不是应用运行验收；未启动 Java、前端或 Playwright，也未执行生产数据库写入。本地 task-root 挂载和具体测试依赖仍应在后续实际运行时按新规范核验，不以文档中的配置期望代替现场事实。

挂载配置已更新但没有执行容器重建；现有容器不会自动获得新挂载，实际安装前应核验所需挂载。安装日志仍由执行方按新规范受控留存，工具测试不等于技术层面自动阻止泄露。

原有未提交/未跟踪文件及未推送提交均保留。只清理本任务已合入且干净的临时 worktree，保留任务分支。未新增任何业务服务，未执行生产数据操作。

## 关联资料

- [设计](../../../../superpowers/_global/specs/2026-09-09-机密R包访问边界与gotoPlot安装卸载设计.md)
- [实施计划与执行记录](../../../../superpowers/_global/plans/2026-09-09-机密R包访问边界与gotoPlot安装卸载实施计划.md)
- 规范仓 `/mnt/g/Obsidian`：`609389e`，约束四个机密 R 包的访问边界。
- 规范仓 `/mnt/g/Obsidian`：`14d86f9`，补齐本地测试依赖、生产测试写入授权与合同测试。
- 部署仓 `/mnt/f/IdeaProjects/goto-software/goto-deploy-backend`：`a032453`，补齐 gotoPlot 工具和模拟测试。
- 文档仓 `/mnt/f/IdeaProjects/goto-software/goto-docs`：设计、计划和总结沿用原文件更新归档，自身提交通过 Change-Set-Id trailer 查找。
- 测试与验收：无独立附件，命令、结果与验证限制见本总结。
