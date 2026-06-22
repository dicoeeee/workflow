# opencode 原生 Agent / Skill / MCP 可见性洞察

## 范围与结论

本文分析的是 `anomalyco/opencode` 官方项目的原生能力，而不是 `oh-my-openagent` 插件扩展。参考源码位于 `reference_project_when_insight/opencode`，本次检出的提交是 `cd292a4ecbaeedd19239edddca77f86d9727c9ae`。对照的官方文档是：

- https://opencode.ai/docs/agents/
- https://opencode.ai/docs/skills/
- https://opencode.ai/docs/mcp-servers/
- https://opencode.ai/docs/config/

核心结论：

1. opencode 中的 agent 是一组可配置的 AI 助手身份，主要由 system prompt、模型、模式和权限组成。
2. 原生 agent 分为 primary agent 与 subagent。primary agent 是用户直接对话的主助手；subagent 是被 primary agent 通过 `task` 工具或用户 `@agent` 调用的专用助手。
3. 用户侧主流定义方式是 `opencode.json` / `opencode.jsonc` 中的 `agent` 字段，以及 `~/.config/opencode/agents/` 或项目 `.opencode/agents/` 下的 Markdown 文件。
4. MCP 在 opencode 原生里会被注册成普通工具。控制某个 agent 是否能看到或调用某个 MCP，本质是控制对应 MCP tool ID 的 `permission` 或旧式 `tools` 配置。
5. Skill 通过原生 `skill` tool 按需加载。不同 agent 的 skill 可见性由 `permission.skill` pattern 控制；若彻底禁用 `skill` tool，则该 agent 不会收到 `<available_skills>` 注入。
6. opencode 原生 skill 没有 `skill.agent` 或 `restrictedAgents` 这种 ownership 字段。`oh-my-openagent` 的 per-agent skill ownership、skill 内嵌 MCP、`skill_mcp` bridge 都是插件扩展能力，不能当作 opencode 原生行为。

## agent 的含义

opencode agent 是“带身份边界的模型运行配置”。它不是单纯的 prompt，也不是独立进程。源码中的运行时 agent 信息包含：

- `name`
- `description`
- `mode`
- `permission`
- `model`
- `prompt`
- `temperature`
- `topP`
- `steps`
- `hidden`
- `options`

源码证据：

- `packages/opencode/src/agent/agent.ts` 定义运行时 `Agent.Info`，字段包括 `name`、`description`、`mode`、`permission`、`model`、`prompt` 等。
- `packages/opencode/src/agent/agent.ts` 内置了 `build`、`plan`、`general`、`explore`、`compaction`、`title`、`summary` 等 agent。
- `packages/opencode/src/tool/registry.ts` 中 `describeTask()` 只把 `mode !== "primary"` 的 agent 作为 task 可调用类型暴露给模型。

### Primary Agent

Primary agent 是用户直接交互的主助手。官方内置 primary agent 包括：

- `build`：默认开发 agent，按配置权限执行工具。
- `plan`：规划与分析 agent，默认限制编辑类能力。
- `compaction`、`title`、`summary`：隐藏系统 agent，用于上下文压缩、标题、摘要等内部任务。

源码中 `build` 与 `plan` 的 `mode` 都是 `primary`。`default_agent` 必须指向非 subagent 且非 hidden 的 agent，否则会被拒绝。

### Subagent

Subagent 是专用助手，被 `task` 工具或用户 `@agent` 显式调用。官方内置 subagent 包括：

- `general`：通用多步任务与并行研究。
- `explore`：只读代码探索。

官方文档还描述了 `scout` 作为外部文档和依赖研究用 subagent。当前检出的源码中内置运行时对象主要能看到 `general` 与 `explore`；这类文档与源码差异应按“官方文档描述用户语义，源码代表当前实现快照”处理。

## agent 的定义方式

### 方式一：在 opencode.json / opencode.jsonc 中定义

最小示例：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "agent": {
    "reviewer": {
      "description": "审查代码质量、风险和测试缺口",
      "mode": "subagent",
      "model": "anthropic/claude-sonnet-4-20250514",
      "permission": {
        "edit": "deny",
        "bash": {
          "*": "deny",
          "git diff*": "allow",
          "git log*": "allow"
        }
      }
    }
  }
}
```

源码路径：

- `packages/core/src/v1/config/config.ts` 的配置 schema 定义了顶层 `agent`。
- `packages/core/src/v1/config/agent.ts` 的 agent schema 支持 `model`、`temperature`、`top_p`、`prompt`、`tools`、`disable`、`description`、`mode`、`hidden`、`options`、`color`、`steps`、`maxSteps`、`permission` 等字段。
- `packages/opencode/src/agent/agent.ts` 会把 `cfg.agent` 合并进内置 agent registry。自定义 agent 默认 `mode: "all"`，权限默认继承全局默认与顶层 `permission`，再追加 agent 自己的 `permission`。

注意：`tools` 仍可用，但已废弃。源码会把 `tools` 归一化为 `permission`：

- `tools.write`、`tools.edit`、`tools.patch` 映射到 `permission.edit`。
- 其他 tool key 原样映射为对应 permission key。

### 方式二：使用 agents 目录下的 Markdown 文件

推荐位置：

- 全局：`~/.config/opencode/agents/<name>.md`
- 项目：`.opencode/agents/<name>.md`

源码还兼容单数目录：

- `agent/`
- `agents/`

Markdown 示例：

```markdown
---
description: 审查代码质量、风险和测试缺口
mode: subagent
model: anthropic/claude-sonnet-4-20250514
permission:
  edit: deny
  bash:
    "*": deny
    "git diff*": allow
    "git log*": allow
---

你是代码审查 agent。优先指出 bug、回归风险和缺失测试，不直接修改文件。
```

Markdown 规则：

- 文件名就是 agent 名。
- 嵌套路径会进入 agent 名，例如 `.opencode/agents/nested/child.md` 会得到 `nested/child`。
- YAML frontmatter 进入 agent 配置。
- 正文会成为 prompt。

源码路径：

- `packages/opencode/src/config/agent.ts` 使用 `{agent,agents}/**/*.md` 扫描 agent Markdown。
- `packages/opencode/src/config/entry-name.ts` 从相对路径去掉 `agent/` 或 `agents/` 前缀，并去掉 `.md` 后缀得到名称。
- `packages/opencode/test/config/config.test.ts` 覆盖了 `.opencode/agents` 复数目录与嵌套 agent 名。

### 方式三：opencode agent create

`opencode agent create` 是官方 CLI 创建方式。源码中它会：

1. 让用户选择保存到当前项目或全局配置目录。
2. 生成 agent 标识、描述和 prompt。
3. 让用户选择允许的 permission。
4. 写出 Markdown agent 文件到 `agents/` 目录。

源码路径：`packages/opencode/src/cli/cmd/agent.ts`。

### 源码补充：其他配置入口

除了用户最常用的 JSON 与 Markdown，源码还有这些会影响最终 agent registry 的入口：

- 全局 config：`config.json`、`opencode.json`、`opencode.jsonc`。
- 项目 config：从当前目录向上查找 `opencode.json` / `opencode.jsonc`。
- `.opencode/opencode.json` / `.opencode/opencode.jsonc`。
- `OPENCODE_CONFIG` 指向的自定义 config 文件。
- `OPENCODE_CONFIG_CONTENT` 注入的 config 内容。
- `OPENCODE_CONFIG_DIR` 指向的额外 config 目录。
- managed config 目录。
- well-known / console 远程配置。
- 插件配置转换层。

这些是配置来源层面的扩展，不改变原生 agent 的主要用户语义：最终仍归并成 agent 配置和 agent Markdown 结果。

源码还兼容 `{mode,modes}/*.md`。这类文件会被强制作为 `primary` agent 载入，是旧式 mode 配置兼容路径。新配置应使用 `agent` / `agents`。

## MCP 可见性

### MCP 原生注册方式

MCP server 在顶层 `mcp` 中定义，支持本地和远程两类。

本地 MCP 示例：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "github": {
      "type": "local",
      "command": ["bun", "x", "github-mcp"],
      "enabled": true,
      "timeout": 30000
    }
  }
}
```

远程 MCP 示例：

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "sentry": {
      "type": "remote",
      "url": "https://mcp.sentry.dev/mcp",
      "oauth": {},
      "enabled": true
    }
  }
}
```

源码路径：

- `packages/core/src/v1/config/mcp.ts` 定义 `local` 与 `remote` MCP schema。
- `packages/opencode/src/mcp/index.ts` 读取 config，连接 MCP server，缓存 tools / prompts / resources。

### MCP tool 命名规则

MCP tools 会被转换成普通模型工具。工具 ID 由 server 名和 MCP tool 名拼接：

```text
sanitize(serverName) + "_" + sanitize(toolName)
```

其中 `sanitize` 会把非 `[a-zA-Z0-9_-]` 字符替换为 `_`。例如：

- server：`my.special-server`
- MCP tool：`tool.b`
- opencode tool ID：`my_special-server_tool_b`

源码路径：

- `packages/opencode/src/mcp/index.ts` 的 `tools()` 构造 MCP tool ID。
- `packages/opencode/src/mcp/catalog.ts` 的 `sanitize()` 定义替换规则。
- `packages/opencode/test/mcp/lifecycle.test.ts` 覆盖了 server 名和 tool 名 sanitization。

### 通过 permission 控制不同 agent 可见不同 MCP

推荐写法是使用 `permission`，因为源码最终就是用 permission 判定工具是否被隐藏或执行前是否需要确认。

示例：默认隐藏 `github` MCP 的所有工具，只允许 `release-manager` agent 使用。

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "github": {
      "type": "local",
      "command": ["bun", "x", "github-mcp"],
      "enabled": true
    }
  },
  "permission": {
    "github_*": "deny"
  },
  "agent": {
    "release-manager": {
      "description": "处理 GitHub release、issue 和 PR 自动化",
      "mode": "subagent",
      "permission": {
        "github_*": "allow",
        "edit": "deny"
      }
    }
  }
}
```

这里的关键是顺序：opencode 会先合并默认权限和全局 `permission`，再追加 agent 自己的 `permission`。`Permission.evaluate()` 使用最后匹配规则，所以 agent 层的 `github_*: allow` 会覆盖全局 `github_*: deny`。

源码路径：

- `packages/opencode/src/agent/agent.ts` 先合并全局 `permission`，再合并 agent 自己的 `permission`。
- `packages/opencode/src/permission/index.ts` 的 `evaluate()` 使用 `findLast()`，即最后匹配规则胜出。
- `packages/opencode/src/session/llm/request.ts` 会在发给模型前用 `Permission.disabled()` 从 tool 列表中过滤全量 deny 的工具。
- `packages/opencode/src/session/tools.ts` 执行 MCP tool 前会 `ctx.ask({ permission: key, patterns: ["*"] })`，其中 `key` 就是 MCP tool ID。

### 旧式 tools 写法

官方 MCP 文档仍展示了 `tools` 写法。源码支持把 `tools` 映射为 `permission`，但官方 agents 文档已经标注 `tools` deprecated。

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "github": {
      "type": "local",
      "command": ["bun", "x", "github-mcp"],
      "enabled": true
    }
  },
  "tools": {
    "github_*": false
  },
  "agent": {
    "release-manager": {
      "tools": {
        "github_*": true
      }
    }
  }
}
```

建议新配置优先使用 `permission`，仅在需要兼容旧示例或旧配置时使用 `tools`。

## Skill 可见性

### Skill 定义与发现

官方文档主推位置：

- 项目：`.opencode/skills/<name>/SKILL.md`
- 全局：`~/.config/opencode/skills/<name>/SKILL.md`
- Claude 兼容：`.claude/skills/<name>/SKILL.md`、`~/.claude/skills/<name>/SKILL.md`
- Agents 兼容：`.agents/skills/<name>/SKILL.md`、`~/.agents/skills/<name>/SKILL.md`

源码还兼容 `.opencode/skill/<name>/SKILL.md` 单数目录。

最小 skill：

```markdown
---
name: pr-review
description: 审查 Pull Request 的风险、测试缺口和可维护性问题
---

## 使用方式

当用户要求审查 PR 或 diff 时，先查看变更范围，再按严重程度输出问题。
```

源码路径：

- `packages/opencode/src/skill/index.ts` 定义了 `.claude`、`.agents`、`.opencode/{skill,skills}` 的扫描模式。
- `packages/opencode/test/skill/skill.test.ts` 覆盖了 `.opencode/skill`、`.opencode/skills`、`.claude/skills`、`.agents/skills` 的发现逻辑。

### Skill frontmatter 字段

官方文档列出的字段包括：

- `name`
- `description`
- `license`
- `compatibility`
- `metadata`

源码当前运行时加载时只使用：

- `name`
- `description`
- `location`
- `content`

`license`、`compatibility`、`metadata` 不参与原生可见性判断。未知字段不会变成 per-agent ownership。

源码路径：

- `packages/opencode/src/skill/index.ts` 的 `isSkillFrontmatter()` 只要求 `name` 是字符串，`description` 可选且为字符串。
- `add()` 只把 `name`、`description`、`location`、`content` 存入 `Skill.Info`。

### Skill 是按需加载

opencode 不会把所有 skill 内容直接塞进 prompt。它会：

1. 在 system prompt 中注入 `<available_skills>`，只包含 name、description、location。
2. 让模型在需要时调用 `skill({ name })`。
3. `skill` tool 再返回完整 skill 内容和 skill 目录下的部分文件列表。

源码路径：

- `packages/opencode/src/session/system.ts` 的 `skills()` 负责注入 skill 说明。
- `packages/opencode/src/skill/index.ts` 的 `fmt(..., { verbose: true })` 生成 `<available_skills>`。
- `packages/opencode/src/tool/skill.ts` 定义原生 `skill` tool。

### 通过 permission.skill 控制不同 agent 可见不同 skill

示例：全局允许普通 skill，但隐藏内部 skill；只有 `internal-docs` agent 能看到并加载 `internal-*`。

```json
{
  "$schema": "https://opencode.ai/config.json",
  "permission": {
    "skill": {
      "*": "allow",
      "internal-*": "deny"
    }
  },
  "agent": {
    "internal-docs": {
      "description": "维护内部文档和团队知识库",
      "mode": "subagent",
      "permission": {
        "skill": {
          "internal-*": "allow"
        }
      }
    }
  }
}
```

Markdown agent 也可以写：

```markdown
---
description: 维护内部文档和团队知识库
mode: subagent
permission:
  skill:
    "*": deny
    "internal-*": allow
---

你只处理内部文档相关问题。
```

生效方式：

- `deny`：该 skill 不出现在 `<available_skills>` 中，调用时也会被权限拒绝。
- `ask`：该 skill 可见，但调用 `skill` tool 时请求用户确认。
- `allow`：直接加载。

源码路径：

- `packages/opencode/src/skill/index.ts` 的 `available(agent)` 会过滤掉 `Permission.evaluate("skill", skill.name, agent.permission).action === "deny"` 的 skill。
- `packages/opencode/src/tool/skill.ts` 执行时会发起 `permission: "skill"`，`patterns: [params.name]` 的权限请求。
- `packages/opencode/src/session/system.ts` 如果 `skill` tool 被完全禁用，就不注入 skills prompt。

### 完全禁用某个 agent 的 skill tool

如果某个 agent 不应该使用任何 skill，可以在该 agent 中禁用 `skill`。

```json
{
  "$schema": "https://opencode.ai/config.json",
  "agent": {
    "readonly-auditor": {
      "description": "只读审计，不加载额外 skill",
      "mode": "subagent",
      "permission": {
        "skill": "deny",
        "edit": "deny",
        "bash": "deny"
      }
    }
  }
}
```

或使用旧式 `tools`：

```yaml
---
description: 只读审计，不加载额外 skill
mode: subagent
tools:
  skill: false
---

只做静态分析，不加载任何 skill。
```

源码中 `Permission.disabled(["skill"], agent.permission).has("skill")` 为 true 时，`<available_skills>` 不会注入。模型请求构建阶段也会把全量 deny 的 `skill` tool 从工具列表中剔除。

### 重要边界：permission.skill 不是 skill ownership

opencode 原生没有在 skill frontmatter 中提供“这个 skill 属于哪个 agent”的字段。也就是说，下面这种写法不是原生语义：

```yaml
---
name: oracle-only
description: 只给 oracle agent 使用
agent: oracle
---
```

在 opencode 原生中，`agent: oracle` 会被当作未知字段，不参与可见性控制。正确做法是在 agent 或全局 `permission.skill` 中写 pattern。

## 同时控制 MCP 与 Skill 的完整示例

目标：

- 默认所有 agent 看不到 `github` MCP tools。
- 默认所有 agent 看不到 `release-*` skills。
- 只有 `release-manager` agent 可以使用 GitHub MCP 和 release skills。

```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "github": {
      "type": "local",
      "command": ["bun", "x", "github-mcp"],
      "enabled": true
    }
  },
  "permission": {
    "github_*": "deny",
    "skill": {
      "*": "allow",
      "release-*": "deny"
    }
  },
  "agent": {
    "release-manager": {
      "description": "创建 release notes、处理 tags、读取 GitHub issue 和 PR",
      "mode": "subagent",
      "permission": {
        "github_*": "allow",
        "skill": {
          "release-*": "allow"
        },
        "edit": "deny"
      }
    },
    "reviewer": {
      "description": "代码审查，不访问 release MCP 和 release skills",
      "mode": "subagent",
      "permission": {
        "edit": "deny",
        "bash": "deny"
      }
    }
  }
}
```

注意：`release-manager` 的 `skill` 配置只写了 `"release-*": "allow"`，并不等于其它 skill 被 deny。因为全局还有 `"*": "allow"`，且 permission 规则是合并后最后匹配胜出。若想让该 agent 只看到 release skills，应写：

```json
{
  "permission": {
    "skill": {
      "*": "deny",
      "release-*": "allow"
    }
  }
}
```

并确保 `"*": "deny"` 在 `"release-*": "allow"` 前面。

## 与 oh-my-openagent 差异

| 维度 | opencode 原生 | oh-my-openagent 插件扩展 |
| --- | --- | --- |
| Agent 定义 | `opencode.json` / `opencode.jsonc` 的 `agent`，以及 `agent(s)/*.md` | 聚合 Claude agents、OpenCode agents、plugin agents、`agent_definitions`、config agents 等 |
| Skill ownership | 无 `skill.agent` / `restrictedAgents` 原生字段 | 支持 skill 的单数 `agent` 字段做 per-agent ownership |
| Skill 可见性 | `permission.skill` pattern 控制隐藏、ask、allow | prompt 构建、`skill` tool、delegate/slash 路径都会按 skill owning agent 过滤 |
| Skill 内嵌 MCP | 无原生 skill-scoped MCP | 支持 skill frontmatter `mcp` 或 skill 目录 `mcp.json` |
| Skill MCP 调用 | MCP 只能通过全局 `mcp` 注册成普通工具 | 通过 `skill_mcp` bridge 暴露 skill 内嵌 MCP |
| MCP 可见性 | MCP tool 作为普通工具，用 `permission` / `tools` glob 控制 | 还有 `disabled_mcps`、内置 MCP、Claude `.mcp.json` 聚合和 skill MCP |

因此，设计 workflow runtime 时不能把 `oh-my-openagent` 的 skill ownership 或 skill-scoped MCP 当成 opencode 原生能力。更稳妥的边界是：

- 原生 opencode：agent permission 控制工具和 skill tool。
- 插件扩展：可以引入 skill ownership、skill 内嵌 MCP、scoped bridge，但需要自己在 prompt、tool description 和执行路径重复校验。

## 对 Workflow Runtime 的启发

1. agent 应被视为“能力视图”，不是单纯角色名。它至少包含 prompt、mode、model 和 permission。
2. MCP 可见性应落在工具层。由于 opencode 把 MCP tool 注册成普通 tool，workflow runtime 也可以把 MCP server 展开为 namespaced tools，再按 agent permission 过滤。
3. Skill 可见性应落在 skill loading 层。原生 `permission.skill` 已经区分了 available list 和执行确认，适合作为最小机制。
4. 如果需要“某 skill 只属于某 agent”，这是 opencode 原生之外的领域规则。应显式设计 allowlist，例如 `agents: ["oracle", "reviewer"]`，并在提示注入、skill tool、delegate、skill MCP bridge 等执行路径重复校验。
5. 不要只依赖 prompt 隐藏。真正安全的路径要同时做到：模型看不到、工具列表不可用、运行时拒绝。

