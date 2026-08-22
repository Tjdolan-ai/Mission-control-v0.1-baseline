# Mission Control MCP App — Architecture v0.1

**Derived from:** `SPEC.md`  
**Framework target:** Skybridge / MCP Apps-compatible runtime  
**Architecture posture:** server-authoritative, tool-driven, provider-agnostic

---

## 1. Architecture decision summary

Mission Control is implemented as one MCP server with a shared interactive app shell and a server-side domain/persistence layer.

The browser/widget is a **projection and control surface**, not the source of truth.

```text
Human + Assistant
       │
       ▼
MCP Host / Conversation
       │
       ├───────────── model tool calls ─────────────┐
       │                                             │
       ▼                                             ▼
Mission Control View                         Mission Control MCP Server
(React / Skybridge)                          (Skybridge McpServer)
       │                                             │
       ├── useCallTool() ───────────────────────────►│
       │                                             ├── Domain services
       │◄── structuredContent ──────────────────────┤
       │                                             ├── Policy/state machine
       │                                             ├── SQLite repository
       │                                             ├── Audit log
       │                                             ├── Context retrieval
       │                                             └── Adapters
       │                                                   │
       ▼                                                   ▼
Shared visible state                              Executors / Reviewers /
+ user controls                                   Research / Browser / etc.
```

---

## 2. Why server-authoritative state is mandatory

Skybridge can expose host/view context and view-state synchronization, but MCP Apps compatibility does not guarantee that view state persists across view renders. Therefore:

- canonical mission/task state MUST be persisted in the server repository;
- view state may hold temporary UI concerns such as selected task, expanded panel, filters, or local draft text;
- the assistant reads canonical state through tools;
- the view mutates canonical state through tools;
- after every mutation, the view reconciles against the returned canonical entity/version.

This prevents the human UI and assistant from drifting into separate realities.

---

## 3. Skybridge mapping

### Server
Use `McpServer` from `skybridge/server`.

### Interactive app shell
Register a widget such as:

```text
show_mission_control
```

The widget returns enough `structuredContent` to render the current shell quickly, then calls additional tools for mutations/detail refreshes.

### UI-to-server actions
Use component-initiated MCP tool calls for:

- opening mission/task detail;
- changing priority/status where allowed;
- starting execution;
- approving/rejecting gated actions;
- requesting verification;
- marking human-required work complete;
- refreshing canonical state.

### Host context
Use host context for presentation concerns only:

- theme;
- locale;
- display mode;
- viewport/max height;
- tool input/result lifecycle.

### Model context feedback
Expose only concise, non-sensitive UI awareness to the model, for example:

```text
User is viewing Mission M-17, Task 3: Verify mobile hero behavior.
Task status: awaiting_approval.
```

Do not dump full private context or canonical entities into UI model-context attributes.

---

## 4. Tool topology

Separate **query tools**, **command tools**, and **orchestration tools**.

### Query tools
Read-only and idempotent:

```text
show_mission_control
get_mission
list_missions
get_task
get_context
get_audit_events
```

### Command tools
Mutate canonical state:

```text
create_mission
update_mission
create_task
update_task
submit_evidence
request_approval
resolve_approval
complete_task
```

### Orchestration tools
Potentially invoke workers or multiple internal operations:

```text
delegate_task
verify_task
```

Orchestration tools must emit auditable run IDs.

---

## 5. Proposed tool contracts

These are product-level contracts; implementation should use strict Zod schemas.

### `show_mission_control`

Input:

```ts
{
  missionId?: string;
  view?: 'missions' | 'mission' | 'now' | 'staff';
}
```

Output:

```ts
{
  activeView: string;
  missions: MissionSummary[];
  activeMission?: MissionDetail;
  now?: ActionableTaskSummary;
}
```

Purpose: primary widget entry point.

### `create_mission`

Input:

```ts
{
  title: string;
  objective: string;
  binaryOutcome: string;
  targetDate?: string;
  tasks: InitialTaskInput[];
  contextRefs?: string[];
}
```

Server validation:
- 4–10 initial tasks;
- title/objective/binary outcome non-empty;
- task ordinals unique and sequential;
- each task has actor/mode/doneWhen;
- dependencies reference tasks in same proposed mission or are empty;
- no invalid initial terminal state.

Returns canonical persisted mission + task summaries.

### `update_task`

Input:

```ts
{
  taskId: string;
  expectedVersion: number;
  patch: AllowedTaskPatch;
}
```

Server checks:
- optimistic version;
- field-level policy;
- state transition validity;
- actor-specific rules;
- approval requirements.

### `delegate_task`

Input:

```ts
{
  taskId: string;
  expectedVersion: number;
  requestedMode?: 'standard' | 'enhanced' | 'flagship';
  adapterHint?: string;
}
```

Behavior:
1. load canonical task;
2. validate `ready` state;
3. resolve dependencies;
4. retrieve minimum necessary context;
5. select capabilities/adapter;
6. create Run record;
7. transition task to `running`;
8. invoke worker;
9. normalize outputs to Evidence items;
10. transition to `verifying` or appropriate blocked/failure state;
11. return current canonical task/run.

`adapterHint` is advisory and operator-configurable; task semantics remain capability-based.

### `verify_task`

Input:

```ts
{
  taskId: string;
  expectedVersion: number;
  forceFreshVerifier?: boolean;
}
```

Behavior:
1. load task and evidence;
2. derive explicit verification criteria from task definition;
3. run deterministic checks first;
4. invoke independent judgment only for unresolved criteria;
5. store VerificationResult;
6. on pass -> `verified`;
7. on fail -> record largest actionable gap and move to repairable state;
8. never mark `complete` automatically unless policy explicitly permits a verified, non-gated task to auto-complete.

Recommended v0.1: separate `verified` from `complete` for clear observability.

### `request_approval`

Input:

```ts
{
  taskId: string;
  actionType: string;
  summary: string;
  target?: string;
  expectedEffect: string;
  evidenceIds?: string[];
}
```

Returns an Approval entity and moves task to `awaiting_approval` when required.

### `resolve_approval`

Input:

```ts
{
  approvalId: string;
  decision: 'approved' | 'rejected';
  expectedVersion: number;
}
```

Approval must be attributable to an authorized human identity/session.

---

## 6. Internal services

### MissionService
Responsibilities:
- mission invariants;
- progress calculation;
- completion eligibility;
- task ordering/dependency validation.

### TaskService
Responsibilities:
- state transitions;
- actor rules;
- mode changes;
- completion eligibility.

### PlanningService
Responsibilities:
- vet objective;
- discovery question generation;
- decomposition;
- execution-brief generation;
- mode/capability assignment.

Planner does not persist until a mission proposal passes validation.

### RoutingService
Responsibilities:
- capability-to-adapter resolution;
- policy constraints;
- availability;
- sensitivity restrictions;
- cost/run limits where measurable.

### ContextService
Responsibilities:
- relevant-slice retrieval;
- sensitivity filtering;
- scope resolution;
- freshness metadata.

### ExecutionService
Responsibilities:
- create Run;
- call adapter;
- normalize evidence;
- handle cancellation/timeouts/errors.

### VerificationService
Responsibilities:
- criterion construction;
- deterministic checks;
- independent reviewer dispatch;
- verdict persistence;
- gap extraction.

### ApprovalService
Responsibilities:
- identify gated actions;
- create approval records;
- resolve decisions;
- prevent unauthorized continuation.

### AuditService
Responsibilities:
- append-only event records;
- correlation IDs;
- operator/worker attribution.

---

## 7. Persistence model

Recommended v0.1: SQLite behind repository interfaces.

Core tables:

```text
missions
mission_tasks
task_dependencies
context_records
mission_context_refs
task_context_refs
runs
evidence
verification_results
verification_criteria
approvals
audit_events
adapter_registry
```

### Concurrency
Every mutable aggregate has an integer `version`.

Writes use:

```text
WHERE id = ? AND version = expectedVersion
```

Zero affected rows = conflict. Return a typed conflict response with current version rather than silently overwriting.

### Transaction boundaries
Use DB transactions for:
- state transition + audit event;
- approval decision + state transition;
- verification result + task verdict transition;
- mission creation + all initial tasks/dependencies.

---

## 8. Run model

A `Run` represents one attempt by a worker/orchestrator.

```ts
interface Run {
  id: string;
  taskId: string;
  mode: 'standard' | 'enhanced' | 'flagship';
  adapterId: string;
  capabilities: Capability[];
  status: 'created' | 'running' | 'blocked' | 'succeeded' | 'failed' | 'cancelled';
  startedAt?: string;
  finishedAt?: string;
  parentRunId?: string;
  round?: number;
  pieceId?: string;
  usage?: {
    inputTokens?: number;
    outputTokens?: number;
    monetaryCost?: number;
    currency?: string;
    source: 'provider_metering';
  };
}
```

Usage fields are stored only when supplied by a trustworthy adapter/provider meter.

---

## 9. Mode orchestrators

### StandardOrchestrator

```text
resolve adapter
-> execute
-> collect evidence
-> verify
-> return task state
```

### EnhancedOrchestrator

```text
execute
-> independent reviewer
-> if fail: one repair pass
-> verify
-> checkpoint to operator if unresolved
```

Reviewer context excludes executor hidden reasoning.

### FlagshipOrchestrator

Preconditions:
- render path confirmed;
- comparison/reference available or explicit operator-approved substitute;
- required generation/render tools available;
- piece decomposition <= 4 by default.

For each piece:

```text
builder run
-> render evidence
-> BriefCritic
-> SystemCritic
-> CraftCritic
-> aggregate binary verdicts
-> repair if any FAIL
-> next round
```

Stop conditions:
- all critics PASS;
- operator stops;
- configured checkpoint ceiling reached;
- required critic becomes blind because evidence/rendering unavailable;
- external blocker.

Every critic verdict persists as structured evidence/verification detail.

---

## 10. UI architecture

Use one app shell with route-like internal state rather than many disconnected widgets unless later host/runtime behavior proves otherwise.

Primary screens:

```text
Missions
Mission Detail
Now
Staff
```

### App shell local state
Safe ephemeral state:
- selected screen;
- selected task ID;
- expanded panel;
- local filters;
- unsaved form text.

### Canonical state
Never local-only:
- mission status;
- task status;
- approvals;
- verification verdicts;
- dependencies;
- evidence references;
- progress;
- run history.

### Mutation pattern

```text
user click
-> useCallTool(command)
-> server validates/mutates
-> server returns canonical updated entity/version
-> UI replaces local projection with returned data
-> optional refresh query if aggregate changed
```

### Assistant awareness
Expose concise model-context data such as selected mission/task and visible blocker. Do not rely on model context as persistence.

---

## 11. Error contract

Every tool should return machine-readable typed errors.

```ts
type MissionControlError = {
  code:
    | 'NOT_FOUND'
    | 'VALIDATION_ERROR'
    | 'VERSION_CONFLICT'
    | 'INVALID_TRANSITION'
    | 'DEPENDENCY_BLOCKED'
    | 'APPROVAL_REQUIRED'
    | 'ADAPTER_UNAVAILABLE'
    | 'EXECUTION_FAILED'
    | 'VERIFICATION_FAILED'
    | 'EXTERNAL_BLOCKER'
    | 'UNAUTHORIZED'
    | 'PERSISTENCE_ERROR';
  message: string;
  retryable: boolean;
  currentVersion?: number;
  details?: Record<string, unknown>;
};
```

No generic success response after a failed persistence write.

---

## 12. Security architecture

### Boundaries

```text
MCP Host
  -> authenticated MCP server
      -> policy middleware
          -> domain service
              -> adapter with least privilege
```

### Requirements
- authorization enforced server-side;
- secrets never enter mission/context prose;
- adapters receive scoped secret references, not raw global credential dumps;
- read-only vs side-effecting capabilities declared in adapter metadata;
- consequential actions pass ApprovalService;
- context sensitivity checked before dispatch;
- audit records written around external side effects;
- CSP explicitly declares only required network/resource domains for the widget.

---

## 13. Adapter interface

```ts
interface ExecutionAdapter {
  id: string;
  capabilities: Capability[];
  supports(input: RouteRequest): Promise<boolean>;
  execute(input: ExecutionRequest): Promise<ExecutionResult>;
  cancel?(runId: string): Promise<void>;
}
```

`ExecutionResult` returns normalized evidence plus optional provider-metered usage.

No adapter may write Mission Control task state directly. It returns results to ExecutionService, which performs validated transitions.

---

## 14. v0.1 testing strategy

### Unit tests
- state transition matrix;
- mission completeness;
- dependency validation;
- actor classification helper rules where deterministic;
- mode routing rules;
- approval gating;
- version conflicts;
- context-sensitivity filtering.

### Integration tests
- tool -> service -> SQLite -> response;
- command mutation + audit event atomicity;
- delegation adapter normalization;
- verification result persistence;
- approval resolution.

### UI tests
- canonical refresh after mutation;
- conflict display/retry;
- approval controls;
- Flagship round/gap display;
- theme/layout behavior.

### End-to-end release test
Use one real mission. Prove:

```text
create
-> display
-> delegate
-> evidence
-> verify
-> complete
-> restart server
-> display same completed canonical state
```

That sequence is the v0.1 proof.

---

## 15. Skybridge-specific verification requirements

During local development:
- use DevTools for rapid tool/widget iteration;
- treat DevTools as an Apps SDK-style emulator, not final MCP Apps compatibility proof;
- validate the completed flow in an actual target MCP Apps/ChatGPT host before release;
- explicitly test widget-initiated tool calls;
- test theme/display constraints;
- test CSP in the deployed environment.

Because cross-host features differ, the app should degrade gracefully rather than make critical state depend on a host-exclusive API.

---

## 16. First implementation order

1. Domain types + state machine.
2. SQLite schema/repositories + migrations.
3. Audit service.
4. Query/command MCP tools.
5. Mission Control app shell widget.
6. UI-to-tool canonical state synchronization.
7. Manual/fake execution adapter.
8. Verification service with deterministic fake verifier.
9. Approval flow.
10. End-to-end tests.
11. One real executor adapter.
12. Enhanced mode.
13. Flagship skeleton.
14. Real project acceptance run.

This order deliberately proves governance and truth before autonomy.

---

## 17. Deferred integrations

After v0.1 acceptance:

- GitHub/repository workers;
- NotebookLM research adapter;
- Google Drive project/canon adapter;
- Hermes persistent executor;
- Telegram remote surface;
- scheduling/cron;
- browser/visual worker pool;
- multiple provider routing;
- local model execution;
- Gmail/Calendar/Slack executive operations;
- spend/cost policy dashboard.

Each integration must plug into the existing adapter/policy model rather than bypassing it.

---

## 18. Architecture invariants

These must remain true as the system grows:

1. The server repository owns canonical mission state.
2. UI state cannot bypass server policy.
3. Workers cannot self-authorize consequential actions.
4. Workers cannot directly mutate canonical task state.
5. Execution evidence and verification are distinct.
6. Judgment verification is independent from executor reasoning.
7. Model/provider identity is runtime telemetry, not business semantics.
8. Sensitive context is need-to-know.
9. Mission completion is derived from task truth, not narration.
10. Autonomy expands only behind existing state, approval, and audit controls.
