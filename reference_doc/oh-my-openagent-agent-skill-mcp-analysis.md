# oh-my-openagent agent / skill / MCP 可见性分析

## 范围

参考项目：`reference_project_when_insight/oh-my-openagent`。

本文聚焦 `oh-my-openagent` 如何在 OpenCode 插件层定义 agent，并控制 skill 与 MCP 的可见性。这个项目并不依赖单一的 OpenCode 原生机制，而是在插件加载时构建一条配置流水线：

1. 从多个来源发现 agents、skills、MCPs 和插件组件。
2. 在插件 `config` hook 中修改 OpenCode 的 `config.agent`、`config.tools`、`config.permission`、`config.mcp` 和 command 配置。
3. 添加运行时工具，例如 `skill` 和 `skill_mcp`，用于暴露 skill 内容以及 skill 内嵌 MCP 能力。

## 插件入口流程

OpenCode 插件工厂会先创建 managers，再创建 tools，然后创建 hooks，最后返回 plugin interface。关键点是：agent、skill、tool 和 MCP 的可见性，都会在插件交给 OpenCode 之前计算完成。

- `packages/omo-opencode/src/testing/create-plugin-module.ts` 在 194-201 行创建 managers，在 203-207 行创建 tools，在 209-220 行把合并后的 skill 数据传给 hooks，并在 222-229 行暴露过滤后的 tools。
- `packages/omo-opencode/src/plugin-handlers/config-handler.ts` 的配置应用顺序是：provider、插件组件、hooks、agents、tools、MCPs、commands、runtime skill source。这意味着 agent 会先完成组装，然后才应用 tool 与 MCP 配置。

## Agent 定义模型

### 来源

Agent 来源由 `loadAgentSources` 统一收集：

- Claude Code 风格的用户级 agents：`~/.claude/agents/*.md`。
- Claude Code 风格的项目级 agents：`<project>/.claude/agents/*.md`。
- OpenCode 全局 agents：OpenCode config dirs 下的 `agents/`。
- OpenCode 项目级 agents：`<project>/.opencode/agents/*.md`。
- Claude 兼容插件贡献的 agents。
- 插件配置 `agent_definitions` 中列出的路径。
- `opencode.json` / `opencode.jsonc` 中内联或引用的 agents。
- 已存在的 OpenCode `config.agent`。

加载器会把这些来源保存在不同的 map 中，方便后续组装时应用优先级和内置 agent 名称保护规则。

### Markdown / JSON 格式

Markdown agent 文件由 YAML frontmatter 加正文组成。支持的 frontmatter 字段是 `name`、`description`、`model`、`tools` 和 `mode`。正文会成为 agent prompt。如果没有写 `mode`，默认是 `subagent`。

`tools` 可以是逗号分隔字符串或字符串数组，最终会转成小写的布尔表。例如 `Bash,Read` 会变成 `{ bash: true, read: true }`。

`agent_definitions` 也可以指向 `.json` / `.jsonc` 文件。JSON 定义要求有 `name` 和 `prompt`，并支持 `description`、`model`、`tools` 和 `mode`。

### 组装与保护

`applyAgentConfig` 会先发现 skills，再加载 agent 来源，然后构建内置 agents、组装最终 `config.agent`，最后注册 agent 名称。

当 Sisyphus 启用时，核心 agents 会先被组装出来：`sisyphus`、`hephaestus`、可选的 `prometheus`、`atlas` 和 `sisyphus-junior`。之后再合并自定义来源。普通自定义来源不能覆盖受保护的内置 agent 名称。

自定义来源的优先级通过对象展开顺序实现：

1. plugin agents
2. user agents
3. OpenCode global agents
4. project Claude agents
5. project OpenCode agents
6. 显式 `agent_definitions`
7. OpenCode config agents
8. 已存在的 `config.agent`

对于同一个自定义 agent key，越靠后的来源优先级越高，但受保护的内置名称除外。

## Skill 定义与可见性

### Skill 来源

Skills 会从以下来源发现：

- 插件配置 `skills.sources`
- host / OpenCode config skills
- Claude 用户级 / 项目级 skill dirs
- `.agents/skills`
- `.opencode/skills`
- bundled shared skills
- 插件内置 skills

`createSkillContext` 会合并这些 skills，按浏览器 provider 过滤浏览器相关 skill，应用 disabled-skill 过滤，保护 shared aliases，并返回 `mergedSkills` 和 `availableSkills`。

### Skill Schema

可接受的 skill 字段包括：

- `description`
- `template`
- `from`
- `model`
- `agent`
- `subtask`
- `argument-hint`
- `license`
- `compatibility`
- `metadata`
- `allowed-tools`
- `disable`

重要发现：这个代码仓并没有实现 `restrictedAgents` 数组。真正用于 per-agent 可见性的字段是单数 `agent`。例如带有 `agent: oracle` 的 skill 只属于 `oracle`。

### 全局禁用 Skill

项目支持几种全局禁用方式：

- 顶层 `disabled_skills`
- `skills.disable`
- `skills.<name>: false`
- `skills.<name>.disable: true`

这些配置会被归一化为小写 alias，并用于 skill 合并和 prompt 组装。

### Per-Agent Skill 可见性

这里有三处执行点：

1. 内置 agent prompt 构建会调用 `buildAvailableSkills(..., agentName)`。如果发现到的 skill 有 `definition.agent`，并且不等于当前 agent name，就会从该 agent 的 available skill list 中排除。
2. `skill` 工具会从描述中隐藏 agent-restricted skills，并在执行时检查调用方 `ctx.agent` 是否匹配 skill 的 `definition.agent`。
3. Delegate-task 的 skill 注入会按被委派目标 agent 过滤 agent-restricted skills，然后才注入内容。Slash-command 执行同样会在当前 agent 不匹配时拒绝 restricted skill 调用。

因此，在 `skill` 和 delegate 路径里，可见性不只是描述层面的隐藏，也包含运行时校验。

### Agent Override `skills`

插件配置里的 `agents.<name>.skills` 是另一个独立机制。它会把指定 builtin / configured skill 内容直接注入 agent prompt。它不同于通过 skill 的 `agent` 字段声明 skill 归属。

## MCP 定义与可见性

### 内置与外部 MCP

全局 MCP 配置由 `applyMcpConfig` 组装：

1. 来自 `createBuiltinMcps` 的内置 MCPs
2. Claude Code `.mcp.json` 风格配置，除非 `claude_code.mcp === false`
3. 用户 `config.mcp`
4. 插件贡献的 MCP servers

`disabled_mcps` 会传入 Claude MCP 加载、内置 MCP 创建流程，最后还会从合并结果中删除匹配名称。

内置 MCP 名称包括 `websearch`、`context7`、`grep_app`、`lsp` 和 `codegraph`。schema 也接受任意非空 MCP 名称，所以它也可以禁用插件或用户 MCP。

### Skill 内嵌 MCP

Skills 可以通过两种方式内嵌 MCP 配置：

- YAML frontmatter 字段 `mcp`
- skill 目录下的 `mcp.json`，可以是 `mcpServers` 结构，也可以直接是 server map

这些 skill MCP 不会被注册成全局 OpenCode MCP。实际流程是：

1. 通过 `skill` 工具加载某个 skill 时，如果该 skill 有 `mcpConfig`，输出里会追加 “Available MCP Servers” 小节。
2. 该小节列出从 MCP server 发现到的 tools / resources / prompts。
3. 它会提示模型调用 `skill_mcp(mcp_name="...")`。
4. `skill_mcp` 工具会在 `skillContext.mergedSkills` 里按名称查找 MCP server，并通过 `SkillMcpManager` 调用。

### Skill MCP 安全边界

`skill_mcp` 会在完整合并后的 skill registry 中查找 MCP server。它本身不会再次检查 owning skill 的 `definition.agent`。该项目的预期边界是：agent 只有在成功加载 restricted skill 后，才会知道对应的 MCP 名称；而 `skill` 工具会校验 agent 限制。

如果要做更严格的 workflow runtime，应该在 MCP bridge 内部也加入同样的 owner-agent 校验，而不只是在 skill loading 阶段校验。

### MCP 环境变量处理

MCP 环境变量展开使用内置 allowlist 加 `mcp_env_allowlist`。project / local scope 的 skill MCP 配置会被当作不可信配置处理环境变量展开。Stdio skill MCP 还会创建清理后的 ambient environment，过滤常见 package-manager 环境变量和疑似 secret 的继承环境变量，同时保留 skill 配置中显式声明的 env entries。

## Tool 可见性与权限

插件会注册自己的 tools，并用 `disabled_tools` 过滤最终 tool registry。它还会按 agent 修改 OpenCode permission 配置：

- 全局隐藏一些内部工具，例如 `grep_app_*`、旧 LSP 名称、`task_*` 和 `teammate`
- 如果 host permission 里有 `skill: "deny"`，就禁用 `skill` 和 `skill_mcp`
- 对部分 research / viewer agents 禁用 `task`
- 对 Sisyphus、Hephaestus、Prometheus、Atlas 等 orchestrator / build agents 开启 task / team tools

这和 skill 可见性是两层机制。Agent 的 `tools` / `permission` 控制该 agent 可用哪些 OpenCode tools；skill 的 `agent` 字段控制哪个 agent 可以看到或加载该 skill。

## 对 Workflow Runtime 设计的启发

1. 把 agent definition 视为一个带优先级的合并 registry，而不是单纯读取一个目录。
2. 区分 “是否出现在 prompt 中”、“是否出现在工具描述中” 和 “运行时是否允许执行”。`oh-my-openagent` 在不同位置分别实现了这三层。
3. 如果采用 skill-owned MCP 模型，不要把 skill MCP 注册成全局 MCP。应该通过 scoped bridge 暴露，让 skill loading 成为 discoverability 步骤。
4. 不要只依赖 prompt 隐藏来保证安全。每条执行路径都应重复校验 skill ownership，包括 MCP bridge 调用。
5. 只有在一个 skill 明确只属于一个 agent 时，才适合使用单数 ownership 字段 `agent`。如果 Workflow Skills 需要允许多个 agent，应显式设计多 agent allowlist，而不是从这个项目中隐式推断。
6. 保持全局禁用项与 per-agent affordances 的边界清晰：`disabled_skills`、`disabled_mcps`、`disabled_tools` 是全局控制；`agents.<name>.tools`、`agents.<name>.permission`、skill `agent` 是按 agent 或 skill 的局部控制。
