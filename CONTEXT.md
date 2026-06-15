# Development Workflow Runtime

This context defines the language for automating development work inside opencode, from design through implementation and verification.

## Language

**Workflow Runtime**:
The opencode-embedded system that executes configured development workflows.
_Avoid_: standalone workflow platform, BPM system

**Workflow Run**:
One concrete execution of a development workflow inside the **Workflow Runtime**.
_Avoid_: Job, Task, Session

**Workflow Run Record**:
The durable record of a **Workflow Run**, including its targets, stage state, step state, audit trail, and results.
_Avoid_: workflow definition

**Workflow Run State**:
The current execution state recorded for a **Workflow Run**, **Workflow Stage**, or **Workflow Step**.
_Avoid_: definition metadata

**Workflow Definition**:
A configurable workflow description that defines the flow structure and runtime policies for **Workflow Runs**.
_Avoid_: business script, skill implementation

**Workflow Definition Scope**:
The organizational or repository boundary where a **Workflow Definition** is maintained and applies.
_Avoid_: execution target

**Workflow Catalog**:
A discoverable collection of **Workflow Definitions** maintained outside any single business repository.
_Avoid_: code repository, workflow run

**Workflow Catalog Repository**:
A Git-backed repository that stores and reviews **Workflow Definitions** for a **Workflow Catalog**.
_Avoid_: business code repository, database-only workflow store

**Repository Workflow Definition**:
A **Workflow Definition** maintained inside one code repository for repository-specific workflow behavior.
_Avoid_: enterprise workflow

**Workflow Selection Dimension**:
Applicability metadata on a **Workflow Definition** used to recommend whether it fits a run context.
_Avoid_: execution policy

**Workflow Execution Dimension**:
Flow structure or runtime policy on a **Workflow Definition** used to execute a **Workflow Run**.
_Avoid_: selector-only metadata

**Workflow Selector**:
A capability that recommends candidate **Workflow Definitions** for a **Workflow Run** from the current context.
_Avoid_: run creator, target authorizer, step selector

**Workflow Target**:
The business or code object that a **Workflow Run** acts on.
_Avoid_: definition scope, single repository only

**Requirement Target**:
A requirement, change topic, defect, or work item that a **Workflow Run** acts on.
_Avoid_: repository

**Repository Target**:
A code repository that a **Workflow Run** acts on.
_Avoid_: service, component

**Service/Component Target**:
A service, microservice, or shared component that a **Workflow Run** acts on.
_Avoid_: repository

**Initial Workflow Target**:
A **Workflow Target** known when a **Workflow Run** starts.
_Avoid_: complete target set

**Discovered Workflow Target**:
A **Workflow Target** added during a **Workflow Run** after analysis or confirmation.
_Avoid_: implicit repository access

**Candidate Workflow Target**:
A target suggested by a **Target Resolver** but not yet added to a **Workflow Run**.
_Avoid_: final impact scope, authorized target

**Workflow Target Provenance**:
The recorded origin of a **Workflow Target** within a **Workflow Run**.
_Avoid_: untracked target

**Target Resolver**:
An enterprise-provided capability that resolves **Workflow Targets** into related or candidate targets.
_Avoid_: runtime-owned asset catalog

**Workflow Target Effect**:
The level of effect a **Workflow Step** has on a **Workflow Target**.
_Avoid_: target access, implicit permission

**Authorized Workflow Target**:
A **Workflow Target** approved or policy-authorized for side-effecting workflow actions.
_Avoid_: merely discovered target

**Workflow Stage**:
A major phase of a **Workflow Run** used for planning, reporting, and metrics grouping.
_Avoid_: arbitrary nesting level

**Workflow Step**:
An executable unit within a **Workflow Stage** whose execution is carried by one or more **Workflow Skills**.
_Avoid_: sub-stage, task, fixed step type

**opencode Session**:
An opencode-managed conversation and execution context used by a **Workflow Run**.
_Avoid_: Workflow Run

**Workflow Skill**:
A business-provided opencode Skill that carries all or part of a **Workflow Step**.
_Avoid_: step implementation, hard-coded step type

**Entry Workflow Skill**:
The **Workflow Skill** that the **Workflow Runtime** starts for a **Workflow Step** and treats as responsible for that step.
_Avoid_: primary task, step handler

**Entry Skill Result Contract**:
The minimal result shape an **Entry Workflow Skill** reports back to the **Workflow Runtime** after carrying a **Workflow Step**.
_Avoid_: business artifact schema

**Supporting Workflow Skill**:
A **Workflow Skill** invoked by an **Entry Workflow Skill** or another supporting skill during a **Workflow Step**.
_Avoid_: sub-step, hidden step

**Parallel Agent Work**:
Concurrent agent execution coordinated by the **Workflow Runtime** for work that can proceed in parallel.
_Avoid_: hidden sub-workflow

**Step-Internal Parallel Agent Work**:
**Parallel Agent Work** performed within one **Workflow Step**.
_Avoid_: separate workflow branch

**Workflow-Level Parallel Agent Work**:
**Parallel Agent Work** performed across multiple **Workflow Steps**.
_Avoid_: step-internal review

## Relationships

- A **Workflow Runtime** belongs inside opencode rather than outside it as a separate orchestration platform.
- A **Workflow Runtime** starts **Workflow Runs** from **Workflow Definitions**.
- A **Workflow Runtime** executes zero or more **Workflow Runs**.
- A **Workflow Runtime** writes **Workflow Run Records** and **Workflow Run State**.
- A **Workflow Runtime** orchestrates **Workflow Stages**, **Workflow Steps**, and **Parallel Agent Work**.
- A **Workflow Definition** describes **Workflow Stages**, **Workflow Steps**, entry skills, ordering, parallelism, and runtime policies.
- A **Workflow Definition** does not own business execution details inside a **Workflow Step**.
- A **Workflow Definition** has a **Workflow Definition Scope**.
- A **Workflow Definition** has **Workflow Selection Dimensions** and **Workflow Execution Dimensions**.
- A **Workflow Catalog** contains zero or more **Workflow Definitions**.
- A **Workflow Catalog Repository** is the first-version storage and maintenance mechanism for a **Workflow Catalog**.
- A **Workflow Catalog Repository** is separate from business code repositories.
- A **Workflow Selector** recommends one or more **Workflow Definitions** from a **Workflow Catalog**.
- A **Workflow Selector** uses **Workflow Selection Dimensions** to recommend candidate definitions.
- A **Workflow Selector** does not create **Workflow Runs**, select **Workflow Steps** or **Workflow Skills**, authorize targets, or decide final impact scope.
- A **Repository Workflow Definition** belongs to one code repository.
- A **Workflow Run** has one or more **Workflow Targets**.
- A **Workflow Run** has one **Workflow Run Record**.
- A **Workflow Run Record** records state at the run, stage, and step levels.
- A **Workflow Target** may be a **Requirement Target**, **Repository Target**, or **Service/Component Target** in the first version.
- A **Service/Component Target** may resolve to one or more **Repository Targets**.
- A **Workflow Run** starts with one or more **Initial Workflow Targets**.
- A **Workflow Run** may gain **Discovered Workflow Targets** during execution.
- Every **Workflow Target** in a **Workflow Run** has **Workflow Target Provenance**.
- A **Target Resolver** may resolve **Requirement Targets** or **Service/Component Targets** into **Candidate Workflow Targets**.
- A **Candidate Workflow Target** may become a **Discovered Workflow Target** after analysis, policy, or confirmation.
- A **Target Resolver** does not decide final impact scope for a **Workflow Run**.
- A **Target Resolver** is provided by enterprise context rather than owned as an internal asset catalog by the **Workflow Runtime**.
- A **Workflow Step** declares a **Workflow Target Effect** for each relevant target set.
- A **Discovered Workflow Target** may be read or analyzed before it becomes an **Authorized Workflow Target**.
- A **Workflow Target** must be an **Authorized Workflow Target** before side-effecting actions such as code modification, test execution with external effects, or merge request creation.
- A **Workflow Run** contains one or more **Workflow Stages**.
- A **Workflow Stage** contains one or more **Workflow Steps**.
- A **Workflow Step** is carried by one or more **Workflow Skills**.
- A **Workflow Step** has exactly one **Entry Workflow Skill**.
- An **Entry Workflow Skill** reports its step outcome through an **Entry Skill Result Contract**.
- A **Workflow Step** may involve zero or more **Supporting Workflow Skills**.
- A **Workflow Skill** may invoke other **Workflow Skills** while carrying a **Workflow Step**.
- A **Workflow Run** may use one or more **opencode Sessions**.
- **Parallel Agent Work** may create multiple **opencode Sessions** concurrently.
- **Parallel Agent Work** may be **Step-Internal Parallel Agent Work** or **Workflow-Level Parallel Agent Work**.
- An **opencode Session** belongs to at most one **Workflow Run** in this context.

## Example dialogue

> **Dev:** "Should the workflow call opencode from an external service?"
> **Domain expert:** "No — the **Workflow Runtime** lives inside opencode and coordinates opencode-native agents, tools, and sessions."

> **Dev:** "Can we use the opencode Session ID as the workflow execution ID?"
> **Domain expert:** "No — a **Workflow Run** is the workflow execution; it may use one or more opencode Sessions."

> **Dev:** "Is run state part of the workflow definition?"
> **Domain expert:** "No — **Workflow Run State** belongs to the **Workflow Run Record**, not the template definition."

> **Dev:** "Should the workflow definition say how requirement analysis asks follow-up questions?"
> **Domain expert:** "No — the **Workflow Definition** selects the entry skill and runtime policies; the skill owns that business interaction."

> **Dev:** "If the workflow is maintained by a platform team, does the run have to target that team's repository?"
> **Domain expert:** "No — **Workflow Definition Scope** says where the definition is maintained; **Workflow Target** says what a particular run acts on."

> **Dev:** "Should the first catalog live in a database with a management UI?"
> **Domain expert:** "No — start with a Git-backed **Workflow Catalog Repository** so definitions can be reviewed, versioned, and rolled back."

> **Dev:** "Does the selector choose a stage, step, or skill?"
> **Domain expert:** "No — the **Workflow Selector** recommends **Workflow Definitions** only."

> **Dev:** "Do users have to list every repository before starting?"
> **Domain expert:** "No — start with **Initial Workflow Targets** and add **Discovered Workflow Targets** as the run learns the impact scope."

> **Dev:** "If the target is payment-service, is that always the Git repository name?"
> **Domain expert:** "No — it may be a **Service/Component Target** that resolves to one or more **Repository Targets**."

> **Dev:** "Can an agent modify a repository as soon as it discovers it?"
> **Domain expert:** "No — discovery can add a target for analysis, but side-effecting work requires an **Authorized Workflow Target**."

> **Dev:** "Should the runtime maintain the service-to-repository ownership model?"
> **Domain expert:** "No — use a **Target Resolver** supplied by enterprise context, even if the first version is provisional."

> **Dev:** "If the resolver returns three repositories, are they all part of the run?"
> **Domain expert:** "No — they are **Candidate Workflow Targets** until analysis, policy, or confirmation adds them."

> **Dev:** "Can the coding and review parts run in different sessions?"
> **Domain expert:** "Yes — one **Workflow Run** may coordinate multiple **opencode Sessions**."

> **Dev:** "Can a stage contain another stage?"
> **Domain expert:** "No — a **Workflow Run** is modeled with **Workflow Stages** and **Workflow Steps** only."

> **Dev:** "Is review a manual step or an agent step?"
> **Domain expert:** "Neither as a fixed category — the **Workflow Skill** carrying that **Workflow Step** defines whether interaction is needed."

> **Dev:** "Does one step always map to exactly one skill?"
> **Domain expert:** "No — a **Workflow Step** may involve multiple **Workflow Skills**, including one skill invoking others."

> **Dev:** "Who owns the success or failure of a step with multiple skills?"
> **Domain expert:** "The **Entry Workflow Skill** owns the **Workflow Step** outcome; supporting skills are internal to that execution."

> **Dev:** "Does using entry skills mean the runtime cannot run several agents in parallel?"
> **Domain expert:** "No — **Parallel Agent Work** remains a **Workflow Runtime** responsibility."

> **Dev:** "Should parallel review inside one design step be modeled as several workflow branches?"
> **Domain expert:** "No — that is **Step-Internal Parallel Agent Work** unless the steps are independently reportable workflow units."

## Flagged ambiguities

- "workflow 项目" is resolved as the **Workflow Runtime** embedded in opencode, not a standalone workflow platform.
- "job", "task", and "session" are not the canonical names for a workflow execution; use **Workflow Run**.
- "run state" is not definition metadata; record it as **Workflow Run State** in the **Workflow Run Record**.
- "workflow definition" is not a business script; use **Workflow Definition** for flow structure and runtime policies only.
- "workflow catalog" is a logical collection; the first-version storage mechanism is a **Workflow Catalog Repository**.
- "definition scope" and "execution target" are distinct; use **Workflow Definition Scope** and **Workflow Target**.
- "workflow selector" selects **Workflow Definitions**, not runs, stages, steps, skills, targets, or authorization decisions.
- "definition dimensions" are split into **Workflow Selection Dimensions** for recommendation and **Workflow Execution Dimensions** for runtime execution.
- "target" should be typed as **Requirement Target**, **Repository Target**, or **Service/Component Target** in the first version.
- "target" does not mean only startup input; distinguish **Initial Workflow Target** from **Discovered Workflow Target**.
- "candidate target" does not mean final impact scope; use **Candidate Workflow Target** before adding it to the run.
- "discovered" does not mean authorized for modification; use **Authorized Workflow Target** for side-effecting work.
- "target resolution" is not owned by the runtime as an asset catalog; use an enterprise-provided **Target Resolver**.
- "stage" and "step" are resolved as a fixed two-level workflow structure, not arbitrary nesting.
- "agent step", "manual step", and "tool step" are not canonical step categories; a **Workflow Step** is carried by a **Workflow Skill**.
- "one step, one skill" is rejected; a **Workflow Step** may be carried by multiple **Workflow Skills**.
- "main skill" is resolved as **Entry Workflow Skill**; other skills used during the step are **Supporting Workflow Skills**.
- "entry skill owns the step" does not mean the **Workflow Runtime** lacks orchestration; **Parallel Agent Work** is a runtime responsibility.
- "parallel work" may mean **Step-Internal Parallel Agent Work** or **Workflow-Level Parallel Agent Work**; distinguish them explicitly.
