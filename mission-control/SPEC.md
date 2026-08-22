# Mission Control MCP App — SPEC.md

**Status:** Draft v0.1 — implementation-ready baseline  
**Product type:** MCP App / shared human–assistant control plane  
**Working name:** Mission Control  
**Primary user:** Operator  
**Secondary user:** AI assistant operating against the same mission state  
**Source corpus:** Long-Term Goal, Short-Term Goal, soul.md Hermes template, Three Model Brain, Hermes NotebookLM Skill, Hermes Skills Mega Resource, Hermes VPS/Telegram guide, Claude Fable design prompts, The Design Loop

---

## 1. Product thesis

Mission Control is the control plane above individual agents, models, skills, repositories, and research systems.

It turns a meaningful objective into a persisted mission, decomposes that mission into bounded executable tasks, routes each task by **capability** rather than provider name, preserves shared state between the human and the assistant, requires explicit approval for consequential actions, and separates **execution** from **verification**.

Mission Control is not another chatbot persona. It is an operational system in which the human and the assistant see and manipulate the same mission state.

### Core proposition

> Tell Mission Control what outcome must ship. It turns that outcome into bounded work, chooses the right execution mode and capability, tracks evidence, asks for human input only when genuinely necessary, verifies results independently, and never marks consequential work complete merely because an executor says it succeeded.

---

## 2. v0.1 objective

A user can provide one real project objective and Mission Control can:

1. convert it into a **binary mission outcome**;
2. decompose it into **4–10 bounded tasks**;
3. classify each task as agent-runnable or human-required;
4. assign a work mode: **Standard, Enhanced, or Flagship**;
5. display the mission and tasks in a shared interactive MCP view;
6. generate a self-contained execution brief for each task;
7. delegate at least one agent-runnable task through an execution adapter;
8. collect execution evidence;
9. run an independent verification pass;
10. require the correct human approval where applicable; and
11. persist the resulting mission/task state so the human and assistant can resume from the same truth.

### v0.1 binary release criterion

**YES** only if one real mission can be created, one task can be executed, independently verified, and reflected correctly in the shared Mission Control UI without manual state repair.

---

## 3. Non-goals for v0.1

The following are deliberately excluded from the first release:

- Fully autonomous 24/7 operation.
- Telegram as the primary interface.
- VPS deployment of Hermes as a mandatory dependency.
- Universal computer control.
- Hard-coding Claude, Codex, Gemini, Hermes, or any model as the architecture.
- Automatic sending of money, signing as the user, publishing irreversible content, or other high-consequence actions without approval.
- Large-scale cron/scheduled-job orchestration.
- Automatically installing third-party skills.
- Treating `soul.md` as a monolithic prompt injected into every task.
- Running the full Design Loop on every design task.
- Trusting executor self-report as proof of completion.

These are later integrations, not prerequisites for proving the control plane.

---

## 4. Product principles

### P1 — Outcome before activity
Every mission has a concrete, binary outcome. “Improve,” “polish,” and “optimize” are invalid without a measurable artifact or condition.

### P2 — Bounded execution
Agent tasks should fit inside one coherent working session. If not, split them.

### P3 — Capability routing, not vendor routing
Mission Control requests capabilities such as `BUILD`, `REVIEW`, `VISION`, or `RESEARCH`. A provider/model adapter resolves the current implementation.

### P4 — Execution and verification are separate concerns
A worker may produce evidence, but a verifier decides whether evidence satisfies the task’s definition of done.

### P5 — Human-required is narrowly defined
A task is human-required when it needs the user’s physical presence, biometric/identity act, legally personal act, or waiting on an external party/time boundary. Ordinary taste decisions and approvals remain inside the shared chat/work surface where possible.

### P6 — Approval is a gate, not a fake task
“Approve this” is represented as an approval state on the relevant task/action, not as a separate dashboard card unless there is genuinely independent human work to perform.

### P7 — Evidence over narration
Completion is backed by artifacts, tool results, tests, screenshots, diffs, URLs, or other inspectable evidence.

### P8 — Shared truth
The UI state and the assistant’s operational context must derive from the same persisted mission/task records.

### P9 — Expensive judgment is intentional
Iterative multi-agent critique is reserved for work where it materially improves the result.

### P10 — Context is retrieved, not dumped
Personal and project context is stored in structured domains and only relevant slices are supplied to each task.

---

## 5. Users and roles

### Operator
The human owner of the mission. Can create missions, make taste/priority decisions, approve gated actions, stop runs, override routing, and mark genuinely human physical/external tasks complete.

### Mission Planner
Assistant capability that vets objectives, asks discovery questions, decomposes missions, and writes task briefs. It does not execute tasks while planning.

### Executor
A worker that performs a bounded task using one or more tools/models.

### Verifier
An independent capability/context that checks the executor’s artifacts/evidence against the definition of done.

### Specialist Critic
Optional independent evaluator used by Enhanced or Flagship workflows. A critic judges rendered/output evidence, not the executor’s intent.

---

## 6. Core domain model

### Mission

Required fields:

```ts
interface Mission {
  id: string;
  title: string;
  objective: string;
  binaryOutcome: string;
  status: 'draft' | 'ready' | 'active' | 'blocked' | 'complete' | 'cancelled';
  targetDate?: string;
  createdAt: string;
  updatedAt: string;
  taskIds: string[];
  contextRefs: ContextRef[];
  version: number;
}
```

Rules:
- 4–10 tasks in the initial decomposition.
- Sweet spot: 5–7.
- Mission cannot become `complete` until all required tasks are verified/complete and all blocking approvals are resolved.

### Task

```ts
interface Task {
  id: string;
  missionId: string;
  ordinal: number;
  title: string;
  objective: string;
  doneWhen: string;
  actor: 'agent' | 'human';
  mode: 'standard' | 'enhanced' | 'flagship';
  status:
    | 'queued'
    | 'ready'
    | 'running'
    | 'awaiting_input'
    | 'awaiting_approval'
    | 'blocked_external'
    | 'verifying'
    | 'verified'
    | 'failed'
    | 'complete'
    | 'cancelled';
  capabilities: Capability[];
  executionBrief: string;
  dependencies: string[];
  approvalPolicy: ApprovalPolicy;
  evidence: Evidence[];
  verification?: VerificationResult;
  contextRefs: ContextRef[];
  createdAt: string;
  updatedAt: string;
  version: number;
}
```

### Capability

Initial capability vocabulary:

```ts
type Capability =
  | 'PLAN'
  | 'BUILD'
  | 'CODE'
  | 'REFACTOR'
  | 'TEST'
  | 'REVIEW'
  | 'ADVERSARIAL_REVIEW'
  | 'SECURITY_REVIEW'
  | 'RESEARCH'
  | 'LONG_CONTEXT'
  | 'VISION'
  | 'AUDIO'
  | 'DESIGN'
  | 'DESIGN_CRITIC'
  | 'BROWSER_VERIFY'
  | 'WRITE'
  | 'DEPLOY';
```

Provider/model names MUST NOT appear in persisted task semantics except in execution telemetry.

### Evidence

```ts
interface Evidence {
  id: string;
  type: 'file' | 'diff' | 'test' | 'screenshot' | 'url' | 'tool_result' | 'note';
  locator?: string;
  summary: string;
  producedBy: string;
  createdAt: string;
}
```

### VerificationResult

```ts
interface VerificationResult {
  verdict: 'pass' | 'fail' | 'blocked';
  checkedAt: string;
  verifierId: string;
  criteria: Array<{
    criterion: string;
    verdict: 'pass' | 'fail' | 'blocked';
    evidenceIds: string[];
    note?: string;
  }>;
  largestGap?: string;
}
```

### ApprovalPolicy

```ts
interface ApprovalPolicy {
  required: boolean;
  reason?: string;
  gate?: 'before_execute' | 'before_external_side_effect' | 'before_complete';
}
```

---

## 7. Mission creation workflow

### 7.1 Intake
User provides a sentence or paragraph describing the desired outcome.

### 7.2 Silent vet
Mission Planner checks:
- binary deliverable;
- practical execution horizon;
- decomposable into <=10 tasks;
- no unbounded vague verbs;
- mission belongs to the operator;
- enough information exists to define “shipped.”

If one critical item is missing, ask one sharp question.

### 7.3 Discovery
Ask 4–8 goal-specific questions covering only relevant dimensions:
- current state;
- actual scope/deliverable;
- available assets/access;
- constraints/dependencies;
- definition of ready;
- past attempts/reusable assets;
- audience/release reality.

Maximum two discovery rounds by default.

### 7.4 Decomposition
Generate 4–10 tasks. Each task must be:
- measurable;
- independently understandable;
- right-sized;
- assigned exactly one actor;
- supplied with a self-contained execution brief;
- accompanied by an explicit `doneWhen` condition;
- assigned dependencies;
- assigned work mode and required capabilities.

### 7.5 Persist before claiming success
The mission is “set” only after storage returns a successful persisted record. UI confirmation must derive from returned state, not generated prose.

---

## 8. Actor classification

For each candidate task ask:

> Can an agent finish this in one coherent working session with the operator available in the shared chat for decisions, without requiring the operator to leave the session to perform an act the agent cannot perform, and without waiting on an external person or real time boundary?

- **YES → `agent`**
- **NO → `human`**

Examples of `human`:
- appear physically on camera;
- attend an in-person event;
- complete biometric verification;
- legally sign as oneself where delegation is inappropriate;
- hold a live human-to-human call that the user must personally conduct;
- wait for a third party response before progress can continue.

Examples that remain `agent` with human input/approval gates:
- draft and revise outreach;
- present design options and ask the user to pick;
- prepare a release and ask for deployment approval;
- review a long edit with the user and apply selections;
- send an already-approved communication using an authorized connector.

Mixed work MUST be split into sequential agent-prep and human-execution tasks.

---

## 9. Work modes

### 9.1 Standard
Use for deterministic or routine work.

Pipeline:

```text
plan -> execute -> verify
```

Typical uses:
- straightforward code changes;
- content edits;
- routine data work;
- simple configuration;
- bounded research;
- ordinary UI maintenance.

Default verification: one independent verifier or deterministic tests where sufficient.

### 9.2 Enhanced
Use when independent judgment materially reduces risk or rework.

Pipeline:

```text
plan -> execute -> independent review -> repair if required -> verify
```

Typical uses:
- important feature implementation;
- significant refactors;
- release-sensitive content;
- security-sensitive but reversible changes;
- important UI sections;
- architecture changes.

Default limits:
- one execution pass;
- one independent review;
- one repair pass before returning to operator unless deterministic validation remains unresolved.

### 9.3 Flagship
Use for high-value creative/experiential work where multiple fresh judgments are worth the cost.

Pipeline:

```text
interview
-> preflight
-> reference teardown
-> divide into 3–4 judgeable pieces
-> for each piece:
     builder
     -> brief critic
     -> system critic
     -> craft critic
     -> repair on any FAIL
-> final integration verification
```

Rules:
- Critics operate in fresh contexts and do not inherit builder reasoning.
- Critics judge rendered/output evidence, not implementation intent.
- Verdicts are binary: `PASS` / `FAIL` / `BLOCKED`, not scores.
- All required critics must pass.
- No fixed round count is used as the success criterion.
- Operator may define a hard run/round ceiling; crossing it requires approval.
- UI reports round count and current gaps. Do not invent token/cost figures where metering is unavailable.
- Default piece count: 3–4 maximum.

Typical uses:
- homepage hero;
- signature brand reveal;
- flagship product storytelling;
- launch experience;
- high-visibility campaign surface.

### 9.4 Routing defaults

```text
routine + reversible + deterministic        -> STANDARD
important + judgment/risk + still bounded   -> ENHANCED
signature + experiential + visual bar high  -> FLAGSHIP
```

The operator can override the mode at any time.

---

## 10. Capability router

The router receives:

```ts
interface RouteRequest {
  taskId: string;
  requiredCapabilities: Capability[];
  mode: 'standard' | 'enhanced' | 'flagship';
  contextWindowHint?: 'small' | 'medium' | 'large';
  multimodal?: boolean;
  sensitivity?: 'normal' | 'restricted';
}
```

The router resolves a compatible worker based on configured adapters.

### Architectural rule

Persist:
- requested capability;
- execution policy;
- evidence;
- result.

Do not persist architectural assumptions such as “Codex always reviews” or “Gemini always sees.” Those are adapter configuration decisions and may change without changing mission semantics.

### Initial adapters

v0.1 requires only:
- `local/manual` adapter for development/testing;
- one real delegated execution adapter.

Later adapters may include:
- coding agents/CLIs;
- Hermes;
- research services;
- multimodal workers;
- local models;
- cloud model providers.

---

## 11. Context architecture (`soul.md` redesign)

Do not inject one giant personal profile into every task.

Store structured context domains:

```text
identity/
communication/
objectives/
projects/
canon/
constraints/
tools/
people/
lessons/
operations/
```

Each context record contains:
- `id`;
- domain;
- title;
- content;
- scope (`global`, `project`, `mission`, `task`);
- sensitivity label;
- source/provenance;
- updated timestamp;
- freshness/expiry metadata when relevant.

### Retrieval rule
Only retrieve context that is relevant to the current task and authorized for the selected worker.

Examples:
- CSS change → project design canon + technical constraints.
- Launch copy → brand voice + product canon + audience context.
- Financial dashboard → relevant business metrics only.

Sensitive context must not be supplied to workers that do not need it.

---

## 12. Research integration

NotebookLM-style source-grounded research is a specialist capability, not the master memory store.

Research adapter responsibilities:
- select/set the source collection/notebook;
- query user-curated sources;
- return source-grounded answers with provenance;
- add sources only under explicit configured policy;
- treat artifact generation/download/deletion as governed actions.

Mission Control stores the research result/evidence reference, not a duplicated uncontrolled copy of every source.

---

## 13. Execution brief specification

Every agent task receives a cold-start-safe brief containing:

```text
Mission
Task
Objective
Binary definition of done
Known inputs/assets
Relevant context
Constraints
Allowed actions
Prohibited actions
Approval gates
Expected evidence
Verification procedure
Workspace/output destination
Stop conditions
```

The brief describes **what must be accomplished** and the constraints. It should not prescribe a provider/model unless the task itself genuinely depends on one.

---

## 14. Verification policy

### 14.1 Verification is mandatory
No agent task moves directly from `running` to `complete`.

Normal transition:

```text
running -> verifying -> verified -> complete
```

### 14.2 Verification hierarchy
Prefer the strongest available evidence in this order where applicable:

1. deterministic tests/checks;
2. external system state/API result;
3. rendered/browser evidence;
4. structured artifact inspection;
5. independent reviewer judgment;
6. executor narrative only as supplemental evidence.

### 14.3 Independence
For judgment-based verification, the verifier should not receive the executor’s hidden reasoning. It receives:
- task objective;
- definition of done;
- relevant constraints;
- produced artifacts/evidence.

### 14.4 Failure handling
A failed verification must name the **largest actionable gap** and route back for repair. The task does not silently self-pass.

---

## 15. Approval and side-effect policy

Approval gates are required before actions such as:
- deploying to production when configured as gated;
- publishing externally;
- sending communications where user approval is required;
- destructive deletion;
- creating irreversible external state;
- changing access/permissions;
- spending funds;
- signing or representing the user in a legally consequential way.

The UI must clearly show:
- proposed action;
- expected effect;
- relevant target/environment;
- evidence/preflight state;
- approval button;
- cancel/reject path.

No approval may be inferred from silence.

---

## 16. MCP tool surface — v0.1

Initial public tools:

### `create_mission`
Creates a persisted mission and initial task set after planning/discovery.

### `get_mission`
Returns canonical mission state plus task summaries.

### `update_mission`
Changes mission metadata/status using optimistic version checks.

### `create_task`
Adds a task to an existing mission while preserving mission invariants.

### `update_task`
Updates task fields/status with transition validation and version checks.

### `get_context`
Retrieves authorized context slices for mission/task execution.

### `delegate_task`
Starts execution through a selected capability adapter.

### `submit_evidence`
Attaches structured evidence to a task.

### `verify_task`
Runs/records independent verification against `doneWhen` and verification criteria.

### `request_approval`
Moves a task/action into an explicit approval gate and returns the UI payload.

### `complete_task`
Completes only a verified task, or a human task explicitly marked done by the operator.

### Tool invariant
No tool named `complete_task` may bypass required verification or approval state.

---

## 17. MCP shared UI — v0.1

### 17.1 Mission list
Shows:
- title;
- binary outcome;
- status;
- target date;
- progress by verified tasks;
- blockers/approval count.

### 17.2 Mission detail
Top section remains stable while task details change.

Includes:
- mission title + outcome;
- progress;
- active blocker;
- work-mode distribution;
- ordered task rail.

### 17.3 Task card
Shows:
- ordinal;
- title;
- `doneWhen`;
- actor;
- mode;
- status;
- capability badges;
- dependency state;
- approval indicator;
- evidence/verification state.

### 17.4 Task detail panel
Shows:
- execution brief;
- relevant context refs;
- evidence;
- verification criteria/verdict;
- gap history;
- current run/round count for Flagship;
- available actions based on current state.

### 17.5 Now view
A focused view showing only:
- highest-priority actionable task;
- why it is actionable now;
- next required action;
- blocking decision if any.

### 17.6 Staff view
Displays capabilities/adapters, not a permanent hierarchy of model brands.

Examples:
- Builder
- Reviewer
- Researcher
- Vision analyst
- Design critic
- Browser verifier

Underlying provider/model can be shown as runtime telemetry, not identity.

### 17.7 Runtime state rule
Skybridge view state/model-context features are useful for ephemeral UI awareness, but Mission Control MUST NOT rely on client/view state for durable mission truth. Canonical mission, task, approval, and verification state lives server-side and is re-read through MCP tools. UI interactions mutate canonical state through tool calls, then refresh from the server response.

---

## 18. State machine constraints

### Agent task

```text
queued
  -> ready
  -> running
  -> awaiting_input -> running
  -> awaiting_approval -> running
  -> verifying
  -> verified
  -> complete
```

Possible exceptional transitions:

```text
running/verifying -> failed
running/verifying -> blocked_external
any nonterminal -> cancelled
```

### Human task

```text
queued -> ready -> blocked_external? -> complete
```

Human tasks do not require agent verification unless the mission explicitly defines an artifact that can and should be verified.

### Invalid transitions
Examples:
- `running -> complete` for agent task;
- `failed -> complete` without repair/re-verification;
- `awaiting_approval -> running` without recorded approval;
- mission `active -> complete` while required tasks remain incomplete.

---

## 19. Persistence and concurrency

v0.1 persistence may use SQLite or equivalent local durable storage behind a repository interface.

Requirements:
- stable IDs;
- timestamps;
- explicit schema migrations;
- optimistic concurrency using `version`;
- append-only audit events for consequential state transitions;
- task evidence references remain immutable once used in a verification record (new evidence creates a new item).

Do not make the UI itself the source of truth.

---

## 20. Audit log

Record at minimum:
- mission/task creation;
- state changes;
- delegation start/finish;
- adapter/model actually used;
- evidence submission;
- verification verdict;
- approval request/decision;
- external side-effect execution;
- completion/cancellation.

Each event includes actor, timestamp, entity ID, prior/new state where relevant, and correlation/run ID.

---

## 21. Security baseline

- Principle of least privilege per connector/adapter.
- Secret material never stored in mission/task prose.
- Context records carry sensitivity labels.
- Worker context is capability-scoped and need-to-know.
- External side effects are explicitly distinguishable from read-only operations.
- Approval state is persisted server-side.
- Every write validates entity version/state transition.
- Tool inputs are schema-validated.
- UI never decides authorization on its own.
- Third-party skill installation is outside v0.1 and must be reviewed before later enablement.
- Production deployment and destructive operations require policy gates.

---

## 22. Failure semantics

Mission Control must distinguish:

- **execution failure** — worker could not perform task;
- **verification failure** — artifact exists but does not satisfy criteria;
- **blocked external** — waiting on person/time/system outside the session;
- **missing input** — operator decision/context required;
- **approval blocked** — action ready but explicit permission absent;
- **adapter unavailable** — requested capability has no usable runtime;
- **persistence failure** — state could not be durably recorded.

Never collapse these into a generic “failed.”

---

## 23. Cost and run controls

Mission Control must not fabricate cost estimates from model self-report.

Where provider metering is available, record actual usage telemetry.

Always support deterministic operational limits:
- maximum Flagship pieces;
- maximum repair cycles before operator checkpoint;
- maximum concurrent workers;
- per-task timeout;
- optional user-defined spend ceiling when the adapter exposes reliable billing data.

Default v0.1:
- Standard: 1 execution + verification.
- Enhanced: 1 execution + 1 independent review + at most 1 automatic repair before checkpoint.
- Flagship: 3–4 pieces, binary critics, operator-visible rounds, configurable checkpoint ceiling.

---

## 24. Source-derived concepts retained

The following concepts are intentionally preserved from the supplied guides:

- long-term binary missions;
- bounded mini-goal decomposition;
- single-session short-term execution;
- explicit human-vs-agent classification;
- cold-start-safe execution briefs;
- verification before claiming persistence/success;
- three-role model separation as a capability idea rather than vendor lock-in;
- source-grounded NotebookLM research as a specialist capability;
- persistent agent/VPS operation as a future executor layer;
- fresh-context design critics;
- binary critique verdicts;
- rendering/output-based design judgment;
- restraint around token-hungry flagship design loops;
- personal context as useful operating memory.

---

## 25. Source-derived concepts deliberately changed

### Change A — model brands are not roles
Original multi-model patterns are represented as capabilities and adapters.

### Change B — `soul.md` becomes structured retrieval
Personal context is partitioned and selectively retrieved rather than injected wholesale.

### Change C — Hermes is an executor adapter
A persistent Hermes deployment may execute delegated work later; it does not own mission truth.

### Change D — approvals are explicit state gates
Approval/review is not represented as arbitrary “human work” when it can occur inside the shared surface.

### Change E — completion requires verification
The executor cannot mark its own agent task complete.

### Change F — Design Loop becomes Flagship mode
The expensive critique loop is available by policy/override, not automatically applied site-wide.

---

## 26. v0.1 acceptance tests

### Mission planning
- [ ] Fuzzy objective triggers a sharp missing-information question.
- [ ] Valid objective produces 4–10 tasks.
- [ ] Every task has actor, mode, capabilities, `doneWhen`, and execution brief.
- [ ] Mixed agent/human work can be split correctly.

### Persistence
- [ ] Created mission survives server restart.
- [ ] UI reload shows the same canonical state.
- [ ] Stale version write is rejected.

### Execution
- [ ] An agent task can be delegated through an adapter.
- [ ] Execution produces structured evidence.
- [ ] Executor cannot directly mark task complete.

### Verification
- [ ] `verify_task` evaluates explicit criteria.
- [ ] Failed verification records the largest actionable gap.
- [ ] Successful verification enables completion.

### Approval
- [ ] Gated external side effect cannot execute without recorded approval.
- [ ] Rejection/cancellation leaves an auditable record.

### UI/shared state
- [ ] Human action in the view is reflected in subsequent assistant tool reads.
- [ ] Assistant tool mutations update the same view state.
- [ ] Mission progress counts verified/completed tasks correctly.

### Work modes
- [ ] Standard runs basic execute/verify.
- [ ] Enhanced invokes an independent review path.
- [ ] Flagship exposes pieces, critics, binary verdicts, gaps, and rounds.
- [ ] User override can promote/demote work mode before execution.

---

## 27. Suggested implementation slices

### Slice 1 — Canonical state
- Mission/Task schemas
- SQLite repository
- state-transition validator
- audit log
- unit tests

### Slice 2 — MCP tools
- create/get/update mission
- create/get/update task
- context retrieval
- evidence/verification tools
- approval gate

### Slice 3 — Shared UI
- mission list
- mission detail
- task rail/detail
- Now view
- optimistic refresh/state sync

### Slice 4 — Delegation
- adapter interface
- local/manual adapter
- first real worker adapter
- run/evidence telemetry

### Slice 5 — Modes
- Standard pipeline
- Enhanced review pipeline
- Flagship orchestration skeleton

### Slice 6 — End-to-end verification
- run one real project mission
- execute one real task
- verify independently
- test restart/resume
- record acceptance evidence

---

## 28. Recommended repository structure

```text
mission-control/
  SPEC.md
  src/
    server/
      tools/
      resources/
      policies/
    domain/
      mission.ts
      task.ts
      evidence.ts
      verification.ts
      approvals.ts
      state-machine.ts
    persistence/
      repository.ts
      sqlite/
      migrations/
    orchestration/
      planner/
      router/
      modes/
        standard.ts
        enhanced.ts
        flagship.ts
      verification/
    adapters/
      execution/
      research/
      context/
    ui/
      views/
      components/
      state/
    audit/
  tests/
    unit/
    integration/
    e2e/
```

Exact framework-specific placement may be adjusted during Skybridge scaffolding, but domain boundaries should remain.

---

## 29. Decisions locked for v0.1

1. Mission Control is the control plane; workers are adapters.
2. Mission/task state is persisted server-side.
3. Human and assistant operate on the same state.
4. Missions require binary outcomes.
5. Agent tasks require verification before completion.
6. Approval is explicit state, never inferred.
7. Capability routing is provider-agnostic.
8. Structured context retrieval replaces full `soul.md` injection.
9. Design work supports Standard / Enhanced / Flagship modes.
10. Flagship critics use fresh contexts and binary verdicts.
11. Hermes VPS/Telegram is deferred to a later execution layer.
12. The first release must prove one complete real mission path before broad integrations are added.

---

## 30. Open implementation decisions

These may be selected during architecture/scaffolding without changing the product thesis:

- exact Skybridge server/runtime package versions;
- SQLite library/ORM choice;
- first real execution adapter;
- authentication strategy for initial private deployment;
- hosting target;
- exact UI component library;
- whether planner decomposition is implemented as one tool call or a controlled multi-step server workflow;
- provider-specific usage telemetry integrations.

No open item above blocks creation of the first architecture and scaffold.

---

## 31. Next build gate

Before implementation begins:

1. map this SPEC to Skybridge architecture primitives;
2. define exact MCP tool JSON schemas;
3. define view-to-tool state synchronization contract;
4. choose persistence implementation;
5. create the initial state-transition tests;
6. scaffold only after those contracts are written.

**Do not add Telegram, background autonomy, or additional model providers before the v0.1 end-to-end acceptance path works.**
