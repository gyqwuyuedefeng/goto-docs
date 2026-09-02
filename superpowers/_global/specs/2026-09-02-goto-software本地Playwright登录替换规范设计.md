---
change_set_id: CS-20260902-a3d63815-f543-4668-a63f-d81e4c3caf85
related_change_set_ids: []
status: approved
---

# goto-software 本地 Playwright 登录替换规范设计

## 背景与目标

goto-software 当前线上登录页 `/mnt/f/IdeaProjects/goto-software/goto-web/src/views/login.vue` 不直接提供微信公众号验证码登录界面。执行需要登录的本地 Playwright 验收前，需要临时使用 `/mnt/f/IdeaProjects/goto-software/goto-web/src/views/login公众号验证码登录.vue` 的完整内容替换登录页，并在受控本地环境中输入固定验证码 `666888`。

本次只补充项目规范，明确替换、登录和恢复边界。不得实际修改两个 Vue 文件，不新增自动化脚本、hook 或第二份项目细则。

## 规则落点

只修改 `/mnt/g/Obsidian/code/01_Project_Specs/goto_software_spec.md`：

- 在 frontmatter 的 `requires` 中增加 `global.local-runtime-verification`，使正文引用与依赖声明一致。
- 将现有“本地后端与 Playwright 验收规范”扩展为“本地后端、前端与 Playwright 验收规范”。
- 保留既有后端 `-Denv=local`、`home` profile、动态端口、前台托管和关闭要求。
- 在同一章节增加前端登录页临时替换、固定验证码和恢复规则。

## 临时替换流程

仅当 Playwright 验收需要微信公众号验证码登录时执行：

1. 确认源文件和目标文件均存在；源文件必须是 `/mnt/f/IdeaProjects/goto-software/goto-web/src/views/login公众号验证码登录.vue`，目标文件必须是 `/mnt/f/IdeaProjects/goto-software/goto-web/src/views/login.vue`。
2. 同时检查目标文件的已暂存和未暂存差异。任一检查发现修改时立即停止，不得覆盖，并向用户报告准确文件状态。
3. 在本次命令创建的临时目录中保存目标文件的原始字节副本，并记录替换前 SHA-256；备份不得放入项目目录。
4. 将源文件的完整内容复制到目标文件。只允许复制，不得移动、删除或修改源文件。
5. 启动本次受控的前后端服务，按全局动态端口、前台会话、实际 `baseURL` 和 `reuseExistingServer=false` 规则执行 Playwright。
6. 在验证码输入框中使用固定本地验收码 `666888`。该验证码只允许配合 `-Denv=local` 的本地受控验收，不得用于生产、共享环境或其他 profile。
7. 无论验收成功、失败、中止或后续步骤报错，都必须恢复目标文件的原始字节内容。
8. 恢复后重新计算 SHA-256，并确认与替换前一致；同时确认目标文件没有新增已暂存或未暂存差异。任一检查失败时不得声称验收完成，必须报告残留状态。

## 数据与提交边界

- `666888` 是固定本地验收输入，不再按不可记录的临时敏感凭据处理，但其适用范围严格限制为 `-Denv=local` 本地验收。
- 不得提交临时替换后的 `login.vue`，也不得把固定验证码或备份内容固化到前端代码、测试脚本、配置或其他项目文件；Playwright 临时产物继续服从全局临时目录规则。
- 验收报告可以说明使用了固定本地验收码，但不得把该机制描述为生产登录方式。
- 目标文件在验收前已有改动时，不得通过备份覆盖的方式绕过停止门禁。

## 失败处理

- 源文件或目标文件不存在：停止验收并报告缺失路径。
- 目标文件存在已暂存或未暂存修改：停止替换，保留现场并报告。
- 临时备份、SHA-256 计算或内容复制失败：不得启动前端或继续 Playwright。
- 验收过程失败：先执行恢复与校验，再报告测试失败。
- 恢复或校验失败：保留临时备份，不得清理恢复证据，也不得声称任务完成。

## 验证方式

1. 运行全局规范校验器，确认项目入口依赖、正文引用和规则边界有效。
2. 运行 `test_validate_rules.py`，确认现有规范治理测试通过。
3. 搜索目标规范，确认两个 Vue 文件绝对路径、`666888`、目标文件干净门禁、SHA-256、无条件恢复和禁止提交均有明确条款。
4. 检查本次规范变更不包含脚本、hook 或 Vue 文件修改。

## 明确不做

- 不实际替换、启动或测试 goto-web。
- 不创建登录替换脚本、Playwright 用例或 hook。
- 不修改两个 Vue 文件、后端代码或项目配置。
- 不处理 goto-software 工作区中的其他既有改动。
