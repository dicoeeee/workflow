# Allow Workflow Targets To Be Discovered During Runs

Workflow Runs start with Initial Workflow Targets and may gain Discovered Workflow Targets during execution. Enterprise development work often begins from a requirement, change topic, or primary repository before the full multi-repository impact scope is known. The runtime therefore records Workflow Target Provenance for every target and allows later steps or confirmations to extend the run's target set instead of requiring all repositories, services, or components to be listed up front.
