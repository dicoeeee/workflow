# Use Two-Level Workflow Structure

The Workflow Runtime models a Workflow Run as Workflow Stages containing Workflow Steps, and does not support arbitrary nested stages in the first version. This keeps configuration validation, runtime state, UI reporting, retries, and metrics aggregation predictable while still allowing future expressiveness through step dependencies or conditions. Arbitrary nesting was deferred because it adds workflow-engine complexity before the core development workflow semantics are proven.
