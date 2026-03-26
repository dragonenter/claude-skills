# Claude Code 全局规则

## Plan Guard 自动衔接

当使用 `superpowers:writing-plans` 生成计划后，在让用户选择执行方式之前，必须先调用 `plan-guard` skill 进行计划验证。流程：

1. `writing-plans` 生成计划文档
2. **自动调用 `plan-guard` Phase 1**：验证文件所有权、任务顺序、任务数量、接口定义
3. 如果有阻断性问题（文件冲突、顺序错误），修复后重新验证
4. 验证通过后，再让用户选择执行方式

当使用 `superpowers:subagent-driven-development` 执行计划时，每次派发 implementer subagent 之前，必须执行 `plan-guard` Phase 2（上下文注入）：

1. 编译前序已完成 task 的产出摘要（文件、接口、决策）
2. 生成文件注册表（标注哪些文件不可修改）
3. 读取实际代码中的接口签名（不用 plan 中写的）
4. 将以上内容注入到 implementer prompt 的 `## Context` 部分

每个 subagent 完成后，执行 `plan-guard` Phase 3（执行后验证）：

1. 用 `git diff --name-only` 检查 subagent 是否只修改了自己拥有的文件
2. 越界则拒绝，重新派发

## 工具使用注意事项

- Read 工具有 10000 token 上限，大文件必须用 offset/limit 分段读取
- Bash 命令默认 2 分钟超时，长时间任务（构建、测试）加 timeout 参数
- 不要让 subagent 自己读 plan 文件，把完整文本粘贴到 prompt 中
