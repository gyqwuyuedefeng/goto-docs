---
change_set_id: CS-20260909-49bb4299-7794-4c48-b094-dac7c5f10ffc
related_change_set_ids: []
---

# 机密 R 包访问边界与 gotoPlot 安装卸载实施计划

**目标：** 禁止 Agent 读取四个机密 R 包源码，补齐 gotoPlot 外部安装和卸载工具。

**架构：** 沿用项目入口和现有 Bash/Rscript 包装器；以模拟 Docker 验证调用合同，以 R parse 验证包装脚本语法。

**约束：** 不进入、不搜索、不读取四个机密目录；不执行实际安装、卸载、容器重建或生产操作；只提交本次文件，沿用文首 ID。

## 执行步骤

- [x] 读取外部脚本和项目规则，确认 gotoPlot 工具缺失，现有两个测试通过。
- [x] 使用独立分支和 worktree 隔离部署仓与规范仓修改。
- [x] 更新 `tests/bash/test_r_package_scripts.sh` 与 `test_reinstall_goto_r_packages.sh`，验证缺失脚本和四包合同先失败。
- [x] 新增 `.bash/install_gotoPlot.sh`、`.bash/remove_gotoPlot.sh`；更新 `.bash/reinstall_goto_r_packages.sh`、`.bash/wsl_rbase_start.sh`，保持原三包行为。
- [x] 修改规范仓 `code/01_Project_Specs/goto_software_spec.md` 与 `code/01_System_Core/rules/trigger-index.md`，明确机密源码边界和工具入口。
- [x] 执行两个 Bash 测试、`bash -n`、仅解析外部八个脚本的 R 语法检查、规范校验、91 项规则测试以及 `git diff --check`，均通过。
- [x] 在各主仓 HEAD 未变化且目标文件干净时快进集成，合入后的两个 Bash 测试、规范校验与 91 项规则测试再次通过，保留原有无关修改。
- [x] 在 `goto-docs/change-sets/completions/2026/09/` 归档唯一总结，记录真实验证结果与跨仓提交；文档归档提交按同一 ID 核验。
