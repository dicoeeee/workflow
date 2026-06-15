# Keep Workflow Definitions Structural

Workflow Definitions define flow structure and runtime policies, while Workflow Skills define business execution details inside each Workflow Step. A definition may describe stages, steps, entry skills, ordering, parallelism, retries, timeouts, and metric labels, but it should not script how a skill performs requirement analysis, design, coding, testing, or review. We chose this boundary to keep workflow configuration stable and readable while preserving business-specific execution inside reusable skills.
