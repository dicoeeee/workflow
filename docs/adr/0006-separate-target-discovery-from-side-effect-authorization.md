# Separate Target Discovery From Side-Effect Authorization

Discovered Workflow Targets may be added to a Workflow Run for reading and analysis, but they must become Authorized Workflow Targets before side-effecting actions such as code modification, externally meaningful test execution, or merge request creation. This separates impact discovery from execution authority in multi-repository workflows. We chose this boundary so agents can freely analyze likely affected repositories while the runtime still enforces explicit approval or policy authorization before changing them.
