# Prioritize Step-Internal Parallel Agent Work

The Workflow Runtime supports both Step-Internal Parallel Agent Work and Workflow-Level Parallel Agent Work, but the first implementation prioritizes parallel agents inside one Workflow Step. This matches development workflow scenarios where one Entry Workflow Skill asks several agents to analyze, design, code, or review from different perspectives before producing one step outcome. Workflow-level parallel steps remain part of the domain model, but can be implemented after step-internal orchestration is proven.
