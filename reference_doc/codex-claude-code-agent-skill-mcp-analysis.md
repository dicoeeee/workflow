# Codex 与 Claude Code 的 Agent、Skill、MCP 可见性分析

## 结论摘要

Codex 和 Claude Code 都把 `agent` 用作“可以独立承接任务的执行上下文”，但两者的配置模型不同：

- Codex 的自定义 agent 本质上是“给 spawned subagent 叠加一层 Codex 配置”。因此，不同 agent 能看到哪些 MCP 和 skill，主要由该 agent 的配置层中的 `mcp_servers`、`enabled_tools`、`disabled_tools`、`skills.config` 决定。
- Claude Code 的 subagent 是 Markdown 文件中的显式 agent manifest。它直接提供 `tools`、`disallowedTools`、`mcpServers`、`skills` 等 frontmatter 字段，因此对“这个 subagent 能用哪些工具/MCP/预加载哪些 skill”的表达更直接。
- 两者都需要区分三件事：上下文里是否出现、模型是否会自动选择、运行时是否真的允许调用。真正的安全边界通常在 tool/MCP allowlist、denylist、permission policy，而不是“描述是否被展示给模型”。

## 源码与发布包检查

本次已下载并检查：

- Codex 官方源码：`reference_project_when_insight/openai-codex/`
  - npm 包 `@openai/codex` 指向 `git+https://github.com/openai/codex.git`，许可证为 Apache-2.0。
  - 重点检查了 `codex-rs/core/src/agent/`、`codex-rs/config/src/`、`codex-rs/core-skills/src/`、`codex-rs/codex-mcp/src/`。
- Claude Code 公开 npm 包：`reference_project_when_insight/claude-code-npm-package/`
  - 包版本为 `@anthropic-ai/claude-code@2.1.185`。
  - 解包后只有 wrapper、install 脚本、类型声明、README、LICENSE。`package.json` 指向平台 native binary 的 optional dependencies，未提供完整 CLI 源码。
  - `LICENSE.md` 标明 Anthropic 保留权利，因此 Claude Code 侧以官方文档为主，npm 包只作为“公开分发物形态”的边界佐证。

主要官方资料：

- Codex Skills: https://developers.openai.com/codex/skills.md
- Codex MCP: https://developers.openai.com/codex/mcp.md
- Codex Subagents: https://developers.openai.com/codex/subagents.md
- Codex AGENTS.md: https://developers.openai.com/codex/guides/agents-md
- Claude Code Subagents: https://code.claude.com/docs/en/sub-agents
- Claude Code Skills: https://code.claude.com/docs/en/skills
- Claude Code MCP: https://code.claude.com/docs/en/mcp

## 1. Agent 的含义

### Codex

Codex 中的 agent 可以分三层理解：

| 概念 | 含义 |
| --- | --- |
| 主 Codex 会话 | 当前与用户交互的根执行上下文。 |
| subagent | Codex 为并行探索、实现、审查等任务 spawn 出来的子会话，有自己的 thread、上下文和工具调用过程。 |
| custom agent / agent role | 用 TOML 定义的一类可被 spawn 的 agent 配置，用来改变子会话的模型、指令、沙箱、MCP、skills 等。 |

容易混淆的是 `AGENTS.md`。它不是 agent 定义，而是 repo 级持久指令文件。Codex 会把适用范围内的 `AGENTS.md` 内容作为上下文/开发者指令注入，让当前 agent 遵守项目规则。

源码佐证：

- `codex-rs/core/src/agent/registry.rs` 用 `AgentRegistry` 管理 root thread 与 spawned thread，并限制 subagent 数量和嵌套深度。
- `codex-rs/core/src/tools/handlers/multi_agents/spawn.rs` 的 `spawn_agent` 工具接收 `agent_type`，按 role 构建子 agent 配置。
- `codex-rs/core/src/agent/role.rs` 明确说明 role 是在 spawn 时叠加到 session config 的配置层。

### Claude Code

Claude Code 中的 agent 主要指 subagent：

| 概念 | 含义 |
| --- | --- |
| 主 Claude Code 会话 | 用户直接交互的根上下文。 |
| built-in subagent | Claude Code 内置的 Explore、Plan、general-purpose 等 agent。 |
| custom subagent | 用户、项目、插件、CLI 或托管配置定义的 specialized assistant。 |

`CLAUDE.md` 同样不是 agent 定义。它是项目/用户记忆和指令内容。Claude Code 的 subagent 可以加载 `CLAUDE.md`，但 Explore/Plan 等内置 agent 可能会跳过它以保持上下文轻量。

官方文档明确说 subagent 有自己的上下文窗口、system prompt、工具访问和权限。

## 2. Agent 的定义方式

### Codex 定义方式

Codex 官方文档中的 custom agent 使用独立 TOML 文件：

- 个人级：`~/.codex/agents/*.toml`
- 项目级：`.codex/agents/*.toml`

独立 custom agent 文件必须包含：

```toml
name = "reviewer"
description = "PR reviewer focused on correctness, security, and missing tests."
developer_instructions = """
Review code like an owner.
Prioritize correctness, security, behavior regressions, and missing tests.
"""
```

可选字段可以使用普通 Codex `config.toml` 支持的配置键，例如：

```toml
model = "gpt-5.2-codex"
model_reasoning_effort = "high"
sandbox_mode = "read-only"
nickname_candidates = ["Atlas", "Delta", "Echo"]
```

源码里还能看到另一种等价/底层形态：在 `[agents.<name>]` 中声明 role，并用 `config_file` 指向角色配置文件：

```toml
[agents.reviewer]
description = "Review-focused role."
config_file = "./agents/reviewer.toml"
nickname_candidates = ["Atlas", "Delta"]
```

源码路径 `codex-rs/core/src/config/agent_roles.rs` 说明：

- Codex 会从配置层读取 `[agents]`。
- 也会发现配置目录下的 `agents/` 目录。
- agent role 文件可以包含 `name`、`description`、`nickname_candidates`，其余字段会被当成普通 `ConfigToml`。
- 独立发现的 role 文件必须有非空 `developer_instructions`。

更重要的是，`codex-rs/core/src/agent/role.rs` 会在 spawn 时把 role 文件作为高优先级 config layer 叠加到子会话配置上。因此 Codex 的 custom agent 不是单独一套 agent manifest DSL，而是“Codex config layer + agent metadata”。

### Claude Code 定义方式

Claude Code 的 subagent 是 Markdown + YAML frontmatter：

- 项目级：`.claude/agents/*.md`
- 用户级：`~/.claude/agents/*.md`
- 插件级：`<plugin>/agents/*.md`
- 当前会话：`claude --agents '{...}'`
- 托管配置：组织管理员下发
- 交互式创建：`/agents`

最小示例：

```md
---
name: code-reviewer
description: Reviews code for quality and best practices
tools: Read, Glob, Grep
model: sonnet
---

You are a code reviewer. When invoked, analyze the code and provide
specific, actionable feedback on quality, security, and best practices.
```

核心字段：

| 字段 | 作用 |
| --- | --- |
| `name` | subagent 唯一标识。 |
| `description` | Claude 判断何时委托给该 subagent。 |
| `tools` | 该 subagent 可使用的工具 allowlist；省略则继承主会话工具。 |
| `disallowedTools` | 从继承或指定工具集中移除的 denylist。 |
| `model` | subagent 使用的模型或 `inherit`。 |
| `permissionMode` | 该 subagent 的权限模式。插件 subagent 不支持。 |
| `mcpServers` | 该 subagent 额外可用的 MCP server。 |
| `skills` | 启动时预加载的 skills。 |
| `hooks` | subagent 生命周期或工具调用 hook。 |
| `memory` | subagent 级持久记忆。 |
| `isolation` | 例如 `worktree`，为 subagent 创建隔离工作树。 |

## 3. 如何控制不同 Agent 可见不同 MCP 和 Skill

### Codex：控制 MCP 可见性

Codex 的 MCP 配置在 `config.toml` 或 custom agent TOML 中使用 `[mcp_servers.<name>]`。

普通 MCP 示例：

```toml
[mcp_servers.github]
url = "https://example.com/mcp"
enabled = true
enabled_tools = ["search_issues", "get_issue"]
disabled_tools = ["create_issue"]
default_tools_approval_mode = "prompt"

[mcp_servers.github.tools.get_issue]
approval_mode = "approve"
```

源码 `codex-rs/config/src/mcp_types.rs` 中 `McpServerConfig` 明确包含：

- `enabled`
- `required`
- `enabled_tools`
- `disabled_tools`
- `default_tools_approval_mode`
- `tools.<tool>.approval_mode`

源码 `codex-rs/codex-mcp/src/tools.rs` 中 `ToolFilter` 明确执行：

1. 如果设置了 `enabled_tools`，只保留 allowlist 中的工具。
2. 再应用 `disabled_tools`，把 denylist 中的工具移除。

因为 custom agent 文件会作为子会话配置层，所以可以给不同 agent 放不同 MCP 配置。例如：

```toml
# .codex/agents/researcher.toml
name = "researcher"
description = "Read-only code and issue researcher."
developer_instructions = "Research broadly and report concise findings."
sandbox_mode = "read-only"

[mcp_servers.github]
url = "https://example.com/github-mcp"
enabled_tools = ["search_issues", "get_issue", "search_code"]
disabled_tools = ["create_issue", "update_issue"]
```

这个配置的语义是：当 Codex spawn `researcher` agent 时，这个子会话的有效配置中会包含该 MCP server 和工具过滤规则。

### Codex：控制 Skill 可见性

Codex skills 的发现来源包括：

| Scope | 路径 |
| --- | --- |
| Repo | 从 CWD 到 repo root 的 `.agents/skills` |
| User | `$HOME/.agents/skills` |
| Admin | `/etc/codex/skills` |
| System | Codex 内置 bundled skills |
| Plugin | 插件自带 skills |

官方文档说 Codex 一开始只把 skill 的 name、description、path 放进上下文，完整 `SKILL.md` 只有在使用时才加载。源码 `codex-rs/core/src/context/available_skills_instructions.rs` 和 `codex-rs/core/src/session/mod.rs` 也能看到这个“可用 skills 列表”作为 developer context fragment 注入。

禁用或启用 skill 使用 `[[skills.config]]`：

```toml
[[skills.config]]
name = "deploy"
enabled = false

[[skills.config]]
path = "/absolute/path/to/some-skill/SKILL.md"
enabled = false
```

源码 `codex-rs/config/src/skills_config.rs` 表明 `SkillConfig` 支持 `name` 或 `path` selector。源码 `codex-rs/core-skills/src/config_rules.rs` 表明：

- 规则从 user 或 session config layer 读取。
- 同名/同路径规则后者覆盖前者。
- `enabled = false` 会进入 disabled path 集合。
- 显式 `$skill` 选择时也会跳过 disabled path。

因此，Codex 中“不同 agent 看不同 skill”的推荐方式是在 custom agent TOML 中放 `skills.config`：

```toml
# .codex/agents/reviewer.toml
name = "reviewer"
description = "Review-only agent."
developer_instructions = "Review code; do not deploy or mutate production systems."

[[skills.config]]
name = "deploy"
enabled = false

[[skills.config]]
name = "release"
enabled = false
```

还可以控制初始 skills 列表是否进入上下文：

```toml
[skills]
include_instructions = false
```

但这只是不把“可用 skills 列表”注入给模型，不等于彻底禁用 skill。真正禁用某个 skill 仍要用 `[[skills.config]] enabled = false`。

Codex skill 的 `agents/openai.yaml` 还有：

```yaml
policy:
  allow_implicit_invocation: false
```

它的含义是禁止 Codex 根据描述自动隐式调用该 skill；用户仍可显式 `$skill` 调用。它不是 per-agent 可见性控制。

### Claude Code：控制 MCP 可见性

Claude Code 的 subagent 对 MCP 有两层控制。

第一层是 `tools` / `disallowedTools`，用于控制工具级可见性。默认情况下，subagent 继承主会话的内置工具和 MCP tools。要收窄能力，使用 `tools` allowlist：

```md
---
name: safe-researcher
description: Research agent with no write or MCP access
tools: Read, Grep, Glob, Bash
---
```

这个 agent 不能编辑文件，也不能用 MCP 工具，因为 MCP 工具没有进入 allowlist。

MCP 工具可用 `mcp__<server>`、`mcp__<server>__*` 等 pattern 控制：

```md
---
name: local-only
description: Inherits most tools except GitHub MCP
disallowedTools: mcp__github
---
```

或者只允许某个 MCP server：

```md
---
name: issue-reader
description: Reads GitHub issues only
tools: Read, Grep, mcp__github__*
---
```

第二层是 `mcpServers`，用于把 MCP server 作用域放到某个 subagent：

```md
---
name: browser-tester
description: Tests features in a real browser using Playwright
tools: Read, Bash, mcp__playwright__*
mcpServers:
  - playwright:
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
---

Use Playwright tools to navigate, screenshot, and interact with pages.
```

这和把 server 放到全局 `.mcp.json` 不同：inline `mcpServers` 可以只在该 subagent 启动时连接，结束后断开；父会话不必看到该 server 的工具描述。

注意：插件 subagent 不支持 `mcpServers`、`hooks`、`permissionMode` frontmatter。需要这些字段时，应把 agent 文件复制到 `.claude/agents/` 或 `~/.claude/agents/`。

### Claude Code：控制 Skill 可见性

Claude Code skill 位于：

| Scope | 路径 |
| --- | --- |
| Enterprise | 托管配置 |
| Personal | `~/.claude/skills/<skill>/SKILL.md` |
| Project | `.claude/skills/<skill>/SKILL.md` |
| Plugin | `<plugin>/skills/<skill>/SKILL.md` |

Claude Code 的 subagent frontmatter 有 `skills` 字段，但它控制的是“启动时预加载哪些 skill 内容”，不是“该 subagent 只能访问哪些 skill”：

```md
---
name: api-developer
description: Implement API endpoints following team conventions
skills:
  - api-conventions
  - error-handling-patterns
---

Implement API endpoints. Follow the preloaded conventions.
```

官方文档明确说明：`skills` 会把完整 skill 内容注入到 subagent 启动上下文；即使没有列在 `skills` 中，subagent 仍可通过 Skill tool 动态发现和调用项目、用户、插件 skills。

要真正限制 skill 调用，有几种方式：

1. 在 subagent 的 `tools` allowlist 中不包含 `Skill`。

```md
---
name: no-skill-reviewer
description: Reviews code without invoking skills
tools: Read, Grep, Glob
---
```

2. 在 subagent 中显式 deny `Skill`：

```md
---
name: no-skill-agent
description: Cannot invoke skills
disallowedTools: Skill
---
```

3. 在权限规则中控制 Skill tool：

```json
{
  "permissions": {
    "deny": ["Skill(deploy *)"]
  }
}
```

4. 在 skill 自身 frontmatter 中设置：

```yaml
disable-model-invocation: true
```

这会阻止 Claude 自动加载该 skill，也会阻止它被预加载进 subagent。

5. 使用 `skillOverrides` 控制 skill 在 Claude 上下文和 `/` 菜单中的可见性：

```json
{
  "skillOverrides": {
    "legacy-context": "name-only",
    "deploy": "off"
  }
}
```

`skillOverrides` 的状态包括：

| 值 | 给 Claude 看 | `/` 菜单 |
| --- | --- | --- |
| `on` | name + description | 可见 |
| `name-only` | 仅 name | 可见 |
| `user-invocable-only` | 隐藏 | 可见 |
| `off` | 隐藏 | 隐藏 |

重要区别：skill frontmatter 的 `allowed-tools` 不是工具 allowlist。它表示 skill 激活时这些工具可免批准使用，但没有列出的工具仍然可以被调用，只是继续受权限规则约束。若要移除工具，用 `disallowed-tools` 或更高层权限 deny。

## 对照表

| 问题 | Codex | Claude Code |
| --- | --- | --- |
| agent 是什么 | 主会话或 spawned subagent；custom agent 是子会话配置层。 | 主会话或 subagent；custom subagent 是 Markdown manifest。 |
| repo 指令文件是不是 agent | `AGENTS.md` 不是 agent，是项目指令。 | `CLAUDE.md` 不是 agent，是记忆/指令。 |
| agent 文件格式 | TOML。 | Markdown + YAML frontmatter。 |
| agent 文件位置 | `~/.codex/agents/`、`.codex/agents/`，源码也支持 `[agents.<role>] config_file`。 | `.claude/agents/`、`~/.claude/agents/`、plugin `agents/`、`--agents` JSON、托管配置。 |
| 必填字段 | 独立 custom agent：`name`、`description`、`developer_instructions`。 | 文件 subagent：`name`、`description`；正文是 system prompt。 |
| 控制 MCP server | custom agent TOML 中写 `[mcp_servers.<name>]`。 | subagent frontmatter 中写 `mcpServers`。 |
| 控制 MCP tool | `enabled_tools` / `disabled_tools`，deny 在 allow 后应用。 | `tools` / `disallowedTools`，支持 `mcp__server` / `mcp__server__*` pattern。 |
| 控制 skill 预加载 | 没有 per-agent `skills` 预加载字段；初始列表由 `skills.include_instructions` 控制。 | subagent `skills` 字段预加载完整 skill 内容。 |
| 控制 skill 禁用 | `[[skills.config]] name/path enabled=false`。custom agent 可通过配置层应用。 | deny `Skill` tool、`Skill(name)` 权限规则、skill `disable-model-invocation`、`skillOverrides`。 |
| skill 自动触发控制 | `agents/openai.yaml` 的 `allow_implicit_invocation: false`。 | `disable-model-invocation: true`，或 `skillOverrides` / permission deny。 |

## 推荐实践

### Codex

如果目标是“不同 agent 可见不同 MCP 和 skill”，推荐把每个 agent 当作一层专用配置：

```toml
# .codex/agents/reviewer.toml
name = "reviewer"
description = "Review code for correctness and security."
developer_instructions = """
You are a review-only agent.
Do not deploy, release, or mutate production systems.
"""
sandbox_mode = "read-only"

[mcp_servers.github]
url = "https://example.com/github-mcp"
enabled_tools = ["search_issues", "get_issue", "search_code"]
disabled_tools = ["create_issue", "update_issue", "merge_pull_request"]

[[skills.config]]
name = "deploy"
enabled = false

[[skills.config]]
name = "release"
enabled = false
```

要点：

- 用 `mcp_servers.<server>.enabled_tools` 做 MCP 最小权限。
- 用 `skills.config` 禁用不适合该 agent 的 skill。
- 用 `skills.include_instructions = false` 只能减少模型初始上下文中的 skill 列表，不能当成禁用机制。
- 用 `allow_implicit_invocation: false` 控制某个 skill 是否可被自动触发，不能当成 per-agent 隔离。

### Claude Code

如果目标是“不同 subagent 可见不同 MCP 和 skill”，推荐同时设置 `tools`、`mcpServers`、`skills`：

```md
---
name: browser-tester
description: Tests UI flows in a browser and reports screenshots
tools: Read, Bash, mcp__playwright__*
mcpServers:
  - playwright:
      type: stdio
      command: npx
      args: ["-y", "@playwright/mcp@latest"]
skills:
  - ui-testing-conventions
disallowedTools: Write, Edit
---

Use the browser tools to test UI behavior. Do not edit files.
```

要点：

- 需要严格最小权限时，不要省略 `tools`，否则 subagent 默认继承主会话工具。
- 需要只给某个 subagent 使用 MCP server 时，用 inline `mcpServers`。
- `skills` 只是预加载，不是访问控制。要禁止 skill 调用，应移除或 deny `Skill` tool。
- skill 自身的 `allowed-tools` 是免批准列表，不是可见性限制。

## 最终判断

Codex 的优势是 custom agent 能复用完整 Codex config 体系：同一套配置层可以同时控制模型、沙箱、MCP、skill、权限等。但这也意味着它的 agent 定义较“重”，skill 可见性是通过配置层和禁用规则实现，而不是单独的 agent-skill manifest。

Claude Code 的优势是 subagent manifest 更直接：`tools`、`disallowedTools`、`mcpServers`、`skills` 都在一个 Markdown frontmatter 里表达。它对 MCP server 的 subagent 级作用域尤其清晰；但 `skills` 字段只表示预加载，真正禁止 skill 仍要通过 `Skill` tool 权限或 `skillOverrides`。

如果要设计 workflow 系统里的 agent 能力边界，可以抽象成三层：

1. Agent 定义层：名称、描述、指令、模型、运行环境。
2. Tool/MCP 权限层：server allowlist、tool allowlist、denylist、approval policy。
3. Skill 上下文层：是否展示、是否自动触发、是否预加载、是否允许显式调用。

Codex 更像“agent = config layer”；Claude Code 更像“agent = manifest + scoped tools”。设计时不要只照搬字段名，应先明确这三层各自的边界。
