# Use Entry Skills For Step Ownership

Each Workflow Step has one Entry Workflow Skill that the Workflow Runtime starts and treats as responsible for the step outcome. The runtime still owns workflow-level orchestration, including stage and step progression, retries, and Parallel Agent Work that may create multiple opencode Sessions concurrently. We chose this boundary so business-specific skill composition can live inside skills while the runtime remains responsible for coordinating the development workflow as a whole.
