# code-review

一个面向 AI 编码助手（Trae / Claude Code / Codex 等支持 `SKILL.md` 的工具）的代码审查技能。

参照内置 `TRAE-code-review` 的流程重写，强调**证据优先、误报过滤、分级输出、可选修复闭环**。

## 特点

- **证据驱动**：每个结论都必须先读 diff 及其上下文，禁止基于猜测评论。
- **分级 + 置信度**：Blocker / Major / Minor / Nit，配合 High / Medium 置信度，低于 Medium 直接丢弃。
- **二次校验**：支持派发独立子代理交叉验证，多数确认才进入最终报告，显著降低误报。
- **结构化报告**：汇总结论 + 可点击 `file:///` 行号链接 + 具体修复建议，可直接贴到 PR 评论。
- **修复闭环**：报告后按用户勾选逐项修复，并自审补丁，不引入新问题。
- **明确的反模式清单**：不会输出“建议检查一下”、“可能有风险”这类无价值评论。

## 目录结构

```
code-review-skill/
├── SKILL.md                       # 技能主文件（触发条件 + 完整工作流）
├── references/
│   └── review-checklist.md        # 正确性/安全/性能/并发/测试等逐项检查清单
├── README.md
└── LICENSE
```

## 安装

**全局安装**（对所有项目生效）：

```powershell
# Trae
Copy-Item -Recurse . "$env:USERPROFILE\.trae-cn\skills\code-review"

# Claude Code
Copy-Item -Recurse . "$env:USERPROFILE\.claude\skills\code-review"
```

**项目级安装**（随仓库共享）：

```powershell
Copy-Item -Recurse . "<your-repo>\.trae\skills\code-review"
```

安装后目录中应直接存在 `SKILL.md`，即 `<skills-dir>\code-review\SKILL.md`。

## 使用

在对话中显式要求审查即可触发，例如：

- `帮我 review 一下当前改动`
- `review 这个 PR #128`
- `对比 main 分支审查一下我的分支`
- `审查 src/auth/session.ts 的改动`

未指明范围时，技能会先让你选择：当前工作区改动 / 对比某个分支 / 指定 PR / 指定文件。

输出依次为：变更摘要与结论 → 分级问题表 → 修复选择 → 逐项修复与复审。

## 自定义

- **调整检查项**：直接编辑 `references/review-checklist.md`，按团队技术栈增删条目（如补充特定框架的坑）。
- **收紧或放宽标准**：修改 `SKILL.md` 中 Step 5 的严重级别定义；团队只关心合入风险时，可以删掉 `Nit` 级别。
- **接入自有规范**：在 Step 2 增加团队规范文件路径（如 `docs/review-guidelines.md`），让评审结论对齐内部约定。
- **跳过文件类型**：Step 4 的过滤清单可按仓库情况扩展（如 `dist/`、`*.generated.ts`）。

## License

MIT
