# Agent 任务完成判断机制调研

本文整理业界对 “agent 是否完成任务、是否成功” 的常见判断逻辑。这里先做观察，不直接收敛成本项目方案。

## 调研对象

本次查看了官方文档与浅克隆源码：

- OpenAI Agents SDK：`reference_project_when_insight/openai-agents-python`
- LangGraph：`reference_project_when_insight/langgraph`
- Microsoft AutoGen：`reference_project_when_insight/autogen`
- CrewAI：`reference_project_when_insight/crewai`
- Temporal 文档、SWE-bench 文档与相关研究资料

## 关键区分

业界通常不会把 “agent 停止了” 直接等同于 “任务成功了”。

- **Execution completion**：运行循环结束，例如达到 END 节点、产生 final output、触发 termination condition。
- **Output acceptance**：输出通过格式、内容或质量校验，例如 guardrail 通过。
- **Goal verification**：外部世界状态或工程证据满足目标，例如测试通过、文件存在、补丁可应用。
- **Human acceptance**：用户、reviewer 或审批人确认继续。
- **Operational failure**：运行错误、超时、超预算、工具失败、用户取消。

这几个维度经常叠加使用，而不是只依赖单一信号。

## 模式一：运行时终止条件

AutoGen 将停止逻辑建模为 `TerminationCondition`。条件接收自上次调用以来产生的消息或事件，返回 `StopMessage` 表示应终止。内置条件包括最大消息数、文本命中、超时、handoff、外部停止、StopMessage、函数调用命中等。多个条件可用 AND / OR 组合。

这种机制解决的是 “什么时候停”，但不天然说明 “停下来是否代表成功”。例如 `MaxMessageTermination` 只是达到预算上限，通常更像 `failed` 或 `incomplete`，而 `TextMentionTermination("APPROVE")` 才更接近某种业务认可。

参考：

- https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/tutorial/termination.html
- `reference_project_when_insight/autogen/python/packages/autogen-agentchat/src/autogen_agentchat/base/_termination.py`
- `reference_project_when_insight/autogen/python/packages/autogen-agentchat/src/autogen_agentchat/conditions/_terminations.py`

## 模式二：图状态机结束

LangGraph 将 agent workflow 表达为 State、Node、Edge。Graph 通过状态更新与边推进，通常由条件边进入 `END`。官方文档说明，图执行在所有节点 inactive 且没有传输中的消息时终止。LangGraph 还提供 recursion limit，避免 agent/workflow 无限循环；提供 interrupt/checkpoint/resume，支持等待人工输入。

这种模式的重点是：成功与否可以由图状态和路由函数显式表达，而不是由模型自己说 “done”。不过 END 仍只是流程终点，是否成功取决于进入 END 前状态里记录了什么。

参考：

- https://docs.langchain.com/oss/python/langgraph/graph-api
- https://docs.langchain.com/oss/python/langgraph/interrupts
- `reference_project_when_insight/langgraph/libs/cli/uv-examples/simple/src/agent/graph.py`

## 模式三：最终输出与 Guardrail

OpenAI Agents SDK 的 `RunResult` 暴露多个 result surface：`final_output`、`new_items`、`raw_responses`、guardrail results、interruptions、run state 等。`final_output` 是最后一个 agent 的输出；如果运行因 approval interruption 暂停，可能为 `None`。Guardrails 可检查输入、最终输出或工具调用，tripwire 触发时会中断执行并抛出对应异常。

这里的设计说明一个边界：final output 是一个结果面，但不是唯一判断面。审计和调试通常需要看 `new_items`、工具调用、guardrail 结果和 interruptions。

参考：

- https://openai.github.io/openai-agents-python/results/
- https://openai.github.io/openai-agents-python/guardrails/
- `reference_project_when_insight/openai-agents-python/src/agents/result.py`
- `reference_project_when_insight/openai-agents-python/src/agents/guardrail.py`

## 模式四：任务输出校验与重试

CrewAI 的 Task 有 `expected_output`，并将任务结果封装为 `TaskOutput`，支持 raw、JSON、Pydantic 等形式。Task guardrails 用于在输出传给下一个任务前验证和转换输出。函数型 guardrail 返回 `(bool, result)`；失败时错误会反馈给 agent，并在 `guardrail_max_retries` 范围内重试。CrewAI 也支持 LLM-based guardrail，用自然语言描述主观或复杂标准。

这种模式接近 “输出验收”。它比纯自然语言结束更稳定，但 LLM-based guardrail 仍带有主观判断和不确定性；函数型 guardrail 更适合可程序化规则。

参考：

- https://docs.crewai.com/en/concepts/tasks
- `reference_project_when_insight/crewai/lib/crewai/src/crewai/task.py`
- `reference_project_when_insight/crewai/docs/en/concepts/tasks.mdx`

## 模式五：外部证据验证

软件工程 agent 更常使用外部可观察证据判断成功。SWE-bench 的评估逻辑是将模型生成的 patch 应用到真实仓库，并在 Docker 环境中运行测试，结果包括 completed、resolved、resolution rate 等指标。

这类机制优点是客观、可复现、可审计；缺点是测试集合可能不充分。相关研究指出，测试通过的 patch 仍可能不符合真实开发者预期，因此 “tests pass” 更适合作为强证据之一，而不是所有场景的完整成功定义。

参考：

- https://www.swebench.com/SWE-bench/guides/evaluation/
- https://arxiv.org/abs/2503.15223

## 模式六：人工介入与审批

OpenAI Agents SDK 暴露 interruptions 和 resumable run state；LangGraph 的 interrupt 会保存 checkpoint 并等待 resume；AutoGen 有 handoff termination；Magentic-One 示例也建议对代码执行等高风险行为引入人工审批。

这类机制不直接判断成功，而是把状态转为 “等待输入/授权”。在高风险动作、目标不明确、验证证据不足时，人工确认是常见兜底。

参考：

- https://openai.github.io/openai-agents-python/results/
- https://docs.langchain.com/oss/python/langgraph/interrupts
- https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/magentic-one.html

## 模式七：监督者/评审者判断

AutoGen 的 multi-agent 模式常用 reviewer/critic agent 产生 `APPROVE`，或用 function call 表达 approval。Magentic-One 的 Orchestrator 会维护 Task Ledger 与 Progress Ledger，持续检查任务是否完成、是否需要重新规划。

这类机制适合开放任务和主观质量判断，但风险是 evaluator 本身仍可能误判。因此业界通常将它和预算上限、外部工具结果、人工审批组合使用。

参考：

- https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/tutorial/termination.html
- https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/magentic-one.html

## 近期研究趋势

近期研究强调两个问题：

- **World completion 与 self-termination 不应混为一谈**。VIGIL 将 “世界状态是否完成” 和 “agent 是否正确宣布完成” 分开评分，用于暴露完成但没停、没完成却宣布成功等问题。
- **防止虚假成功需要可验证 gate**。Goal-Autopilot 提出 gated finite-state machine，禁止 agent 在可证伪 gate 未执行并通过前声明 done。

这说明业界趋势不是相信 agent 自报成功，而是将终止声明绑定到可观察证据、状态机 gate、外部验证或人工确认。

参考：

- https://arxiv.org/abs/2605.08747
- https://arxiv.org/abs/2606.11688

## 对本项目的非决策性观察

结合当前 Workflow Runtime 语境，值得后续继续分析的问题是：

- Step Driver 是否应区分 “运行结束原因” 与 “业务成功证据”。
- Step Completion Signal 是否应记录证据来源，而不只记录状态枚举。
- 不同 Step 是否需要不同证据层级：文本输出、产物存在、工具结果、测试结果、人工确认。
- `waiting_for_input`、`waiting_for_authorization` 应视为非终态暂停，而不是失败。
- 对 coding/test/MR 类 Step，外部证据通常比 agent 自评更重要。
- 对 requirement/design/review 类 Step，可能需要 LLM evaluator、人工确认或产物检查组合。

暂不建议从调研直接得出统一规则。更稳妥的下一步，是先把 “完成判断信号” 拆成可观察证据类型，再回到 Workflow Step Driver 讨论它如何组合这些证据。
