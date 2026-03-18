# Multi-Agent Orchestration

Build a **reliable multi-agent system** that can plan, delegate, execute tools, and recover from failures with **traceability, budgets, and evaluation**.

This module is written as a **build spec**: use it to research, design, and implement a production-grade orchestration layer (not a demo).

---

## Key concepts (deep dive with examples)

- `KEY_CONCEPTS_WITH_EXAMPLES.md` — core concepts with concrete examples (roles, tools, state, budgets, observability, security, DAG/state machine).
- `concepts.html` — scrollable concept walkthrough (with a 3D mnemonic panel that updates as you scroll).

---

## What you are building (scope)

### Core outcome

A multi-agent orchestrator that supports:
- **Role-based agents** (planner, executor, reviewer, retriever, etc.)
- **Tool use** with validation, timeouts, retries, and sandboxing
- **Stateful execution** with memory and context management
- **Budgeting** (tokens, time, tool calls, cost) and **termination guarantees**
- **Observability** (structured logs + traces + metrics)
- **Evaluation** (task success rate, cost, latency, reliability)

### Non-goals (for v1)

- Training new models
- Fancy UI
- “Infinite autonomy” without constraints

---

## Why multi-agent systems fail in production (design for this)

Multi-agent systems break because of **systems issues**, not model quality:
- **Non-termination**: loops, bouncing between agents, tool retry storms
- **State drift**: stale/contradictory shared memory, missing invariants
- **Tool brittleness**: invalid schemas, flaky APIs, slow tools, partial failures
- **Cost blow-ups**: uncontrolled tool calls + large contexts + expensive models
- **Security**: tool injection, prompt injection, data exfil via tools
- **Poor debugging**: no traces, no step-by-step artifacts, no reproducibility

Your orchestrator must treat these as first-class constraints.

---

## Reference architecture (recommended)

### Execution graph

Use an explicit DAG/state machine rather than ad-hoc recursion:

1. **Intake**: normalize request + context, assign tenant/user, initialize budgets
2. **Plan**: create a plan (steps + responsible agent + required tools)
3. **Dispatch**: route a step to the right agent with scoped context
4. **Tool run**: validate tool call → run with timeout → validate output schema
5. **Review/guard**: detect hallucinated tool outputs, policy violations, unsafe actions
6. **Commit**: update state, write artifacts, checkpoint
7. **Terminate**: stop on success, budget exhaustion, or safety constraint

### Key components

- **Orchestrator**: step scheduler, budgets, termination logic, checkpointing
- **Agent runtime**: agent interface, message formatting, structured output parsing
- **Tool runtime**: tool registry, schema validation, execution, sandbox/timeouts
- **Memory**:
  - *Short-term*: current run state, step results, tool outputs
  - *Long-term*: optional vector store / DB keyed by user/project/task
- **Policy layer**: allow/deny tools, redact sensitive fields, enforce compliance
- **Observability**: trace spans per step + tool + model call; metrics for SLOs
- **Eval harness**: deterministic replay, golden tasks, reliability & cost metrics

---

## Interfaces (define these first)

### Agent contract (minimum)

- **Inputs**: `task`, `context`, `state_snapshot`, `allowed_tools`, `budget`
- **Outputs**:
  - `action`: one of `tool_call | delegate | ask_clarifying | finalize | retry`
  - `tool_call` (if any): name + JSON args
  - `delegation` (if any): target agent + subtask + context slice
  - `final_answer` (if finalize): user-facing result + citations to artifacts
  - `state_updates`: structured patches (what to write to memory/state)

### Tool contract (minimum)

- `name`
- `input_schema` (JSON schema)
- `output_schema`
- `timeout_ms`
- `side_effects`: `none | external_api | filesystem | db`
- `safety_level`: `low | medium | high` (gates what’s allowed in a run)

### Orchestrator invariants

- A run must produce:
  - A **step-by-step trace**
  - A final status: `success | failed | budget_exceeded | unsafe | invalid_output`
  - A bounded number of steps/tool calls (no infinite loops)

---

## Reliability features (must-have)

- **Max steps** and **max retries per tool**
- **Idempotency keys** for external calls (avoid duplicate writes)
- **Timeouts** for every tool and model call
- **Schema validation** for tool inputs and tool outputs
- **Structured outputs** for agent decisions (avoid free-text parsing)
- **Checkpointing**: write state after each step so runs can be resumed/replayed
- **Fallback strategy**:
  - If tool fails: retry with backoff → alternative tool → degrade gracefully
  - If agent output invalid: re-prompt with stricter schema → switch to “reviewer” agent

---

## Security & governance (must-have for enterprise)

Threats to design against:
- **Prompt injection**: user content tries to override tool policies
- **Tool injection**: tool output contains instructions to call other tools
- **Data exfiltration**: agent leaks secrets via web/search or logging
- **Unsafe actions**: destructive tools invoked without approval

Controls:
- **Tool allowlist** per agent + per run
- **Redaction** in logs/traces (PII/keys)
- **Policy checks** before executing tool calls
- **Audit trail**: who/what/when for tool usage

---

## Evaluation (how you prove it works)

### Metrics to track

- **Task success rate** (binary + graded)
- **Steps to completion** (mean, p95)
- **Tool error rate** and **retry rate**
- **Non-termination rate** (should be ~0)
- **Cost per task** (tokens + $) and variance
- **Latency per stage** (plan, dispatch, tool, review)
- **Safety violations caught** (and false positives)

### Test sets (start small, expand)

Create a `tasks/` suite of 20–50 tasks across:
- research + summarize
- data extraction + validation
- code generation + unit tests
- multi-step workflows with one flaky tool
- adversarial tasks (injection attempts)

### Deterministic replay

Record every run as an artifact (inputs, model responses, tool IO) so you can replay and regress-test changes.

---

## Suggested codebase structure (when you start implementing)

Create this skeleton inside this module when you’re ready to add code:

```
multi-agent-orchestration/
├── README.md
├── docs/
│   ├── architecture.mmd
│   ├── threat_model.md
│   └── adr/
├── src/
│   ├── orchestration/
│   │   ├── orchestrator.py
│   │   ├── scheduler.py
│   │   ├── budgets.py
│   │   ├── termination.py
│   │   └── state_store.py
│   ├── agents/
│   │   ├── base.py
│   │   ├── planner.py
│   │   ├── executor.py
│   │   └── reviewer.py
│   ├── tools/
│   │   ├── registry.py
│   │   ├── schemas.py
│   │   ├── runner.py
│   │   └── builtins/
│   ├── policies/
│   │   ├── allowlists.py
│   │   ├── redaction.py
│   │   └── validators.py
│   ├── observability/
│   │   ├── tracing.py
│   │   ├── metrics.py
│   │   └── logging.py
│   └── eval/
│       ├── harness.py
│       ├── tasks/
│       └── scoring.py
├── tests/
│   ├── test_orchestrator.py
│   ├── test_tool_runner.py
│   └── test_termination.py
└── examples/
    ├── run_local_demo.py
    └── sample_tasks.jsonl
```

Keep all **artifacts** (run traces, eval results) out of `src/` and store them in a top-level `artifacts/` directory once you start running experiments.

---

## Research checklist (what to read/compare)

When researching, focus on **implementation mechanics** and failure handling:

- **Orchestration patterns**: Plan→Execute, ReAct, supervisor-worker, debate/reviewer loops
- **State & memory**: how to store state, summarize history, and avoid context bloat
- **Tool reliability**: timeouts, retries, schema validation, sandboxing
- **Evaluation**: task suites + replay + regression tests
- **Observability**: traces per step, tool, and model call; cost attribution

---

## Build phases (practical roadmap)

1. **Phase 1 — Single agent + tool runner**: strict tool schemas, timeouts, artifacts
2. **Phase 2 — Supervisor + worker**: delegation, shared state, termination rules
3. **Phase 3 — Multi-agent**: role separation (planner/executor/reviewer), conflict handling
4. **Phase 4 — Safety + governance**: policy checks, audit trail, redaction
5. **Phase 5 — Eval + reliability**: task suite, deterministic replay, regression pipeline

If you follow these phases, you’ll avoid the most common trap: “multi-agent complexity before basic tool reliability.”

