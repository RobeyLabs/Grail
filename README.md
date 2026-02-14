# GRAIL

**Goal-Regressive Adaptive Inference Layer**

> A distributed architecture for autonomous, gossip-coordinated agent workloads.

---

## What Is GRAIL?

GRAIL is an architectural proposition for running autonomous agent systems without a central coordinator. Workers are stateless and interchangeable. Coordination emerges from a gossip-based cluster layer and a CRDT-backed shared block state. Rather than planning forward from an initial state, GRAIL workers begin by attempting the declared final goal and discover prerequisites through structured failure — a process called **goal regression**.

The dependency graph of any given task is not defined upfront. It emerges from the work itself.

---

## The Problem It Solves

Most agent orchestration systems require either:

- A **central manager** that plans and directs work — a single point of failure that requires knowing the full plan before starting, or
- **Peer-to-peer negotiation** — which scales poorly and produces consistency guarantees that are hard to reason about under failure.

GRAIL proposes a third model suited to open-ended operational tasks where the correct sequence of steps cannot be enumerated at invocation time. Examples:

- *"Harden all ingress and egress rules of this network."*
- *"Update software and configurations on every node in this ecosystem."*
- *"Identify and mitigate this zero-day risk across all affected systems."*

---

## Core Principles

### Workers Are Stateless

A worker accepts one task, does work, produces a result, and fully resets. It holds no memory between tasks and maintains no peer connections. The task queue is the only source of truth for pending work.

### Start at the Goal

When a lineage initialises, workers immediately attempt the terminal task — the declared final outcome. That attempt will fail. The structured failure is a **block declaration**: a precise statement of what prerequisite is missing. Those prerequisites become new tasks. Each may fail and produce further declarations. The full dependency tree grows top-down; execution proceeds bottom-up as leaf tasks complete and unblock their parents.

```
Goal (Depth 0):    Harden all ingress/egress rules
                       │
                       ├── BLOCKED BY (Depth 1): Enumerate all ingress rules
                       ├── BLOCKED BY (Depth 1): Enumerate all egress rules
                       └── BLOCKED BY (Depth 1): Identify all network interfaces
                                   │
                                   └── BLOCKED BY (Depth 2): Authenticate to network API

Execution order: Depth 2 → Depth 1 → Depth 0 (terminal — lineage ends)
```

### Blocks Require Quorum

A single worker cannot redirect the cluster. Block declarations require confirmation from at least two peer workers before they are treated as authoritative. A single refutation — one worker that successfully makes progress on the supposedly blocked task — dismisses the claim immediately.

```
Authority              Permitted Action
─────────────────────────────────────────────────────────────────
Single worker          Determine next step after completing work
Quorum (≥ 3 workers)   Declare an actionable block
Scanner worker         Emit block events when quorum cannot converge
```

### Block State Is a 2P-Set CRDT

All block declarations and resolutions are stored in a Two-Phase Set CRDT replicated across every node via gossip. The 2P-Set guarantees convergence under concurrent updates regardless of message order. A resolved block can never be re-activated by a stale node — removal is permanent.

```go
type BlockSet struct {
    Added   map[string]ActionableBlock   // ClaimID → block
    Removed map[string]BlockResolution   // ClaimID → resolution
}

func (bs BlockSet) IsActive(claimID string) bool {
    _, resolved := bs.Removed[claimID]
    return !resolved
}

// Merge is commutative, associative, and idempotent
func (bs BlockSet) Merge(other BlockSet) BlockSet { ... }
```

### Termination Is Mechanical

Done is not a judgment call. When any worker successfully executes the task flagged `IsTerminal = true`, the lineage is complete. No manager vote required.

---

## Architecture

```
                   ┌─────────────────┐
                   │   Task Queue    │  ← durable, at-least-once
                   │ (NATS/SQS/Redis)│
                   └────────┬────────┘
                            │ dequeue
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
        ┌─────────┐   ┌─────────┐   ┌─────────┐
        │Worker 0 │   │Worker 1 │   │Worker N │   stateless, interchangeable
        └────┬────┘   └────┬────┘   └────┬────┘
             │             │             │
             └─────────────┼─────────────┘
                           │ gossip (Serf / SWIM)
                   ┌───────▼────────┐
                   │  2P-Set CRDT   │  ← block state, replicated
                   │  Block State   │
                   └───────┬────────┘
                           │
                   ┌───────▼────────┐
                   │ Scanner Worker │  ← observes, arbitrates, no authority
                   └────────────────┘
```

### Components

| Component | Role |
|---|---|
| **Task Queue** | Durable at-least-once delivery. Source of truth for pending work. |
| **Worker Pool** | N stateless workers. Ingest, process, reset. No shared state. |
| **Gossip Layer** | Serf or equivalent SWIM protocol. Membership, failure detection, event propagation. Not a consensus system. |
| **Block State** | 2P-Set CRDT replicated across all nodes. Canonical record of active and resolved blocks per lineage. |
| **Scanner Worker** | Meta-worker with full lineage read access. Emits block events when quorum stalls. No directive authority. |

---

## Block Claim Lifecycle

```
Worker A cannot complete task
         │
         ▼
Emit UnconfirmedBlockClaim via gossip
Hold task (do not ack)
         │
         ├──────────────────────────────┐
         ▼                              ▼
Workers B & C attempt              Worker D attempts
the blocked task type              the blocked task type
         │                              │
    Also fail                     Succeeds
         │                              │
Emit BlockConfirmation             Emit BlockRefutation
         │                              │
         ▼                              ▼
  Quorum reached                  Claim dismissed
  ActionableBlock                 Worker A resumes
  gossiped to cluster
         │
         ▼
Claimant returns task to pool
Takes up blocking task type itself
Other workers check block list
before selecting next task
```

---

## The Scanner Worker

The Scanner activates under three conditions:

- **Conflict** — two workers declared incompatible next steps for the same task type.
- **Stall** — no tasks have completed within a lineage beyond the stall threshold and the block graph has not changed.
- **Ambiguity** — the terminal task succeeded but intent may not be fully satisfied against the original directive.

The Scanner has no special protocol authority. It participates using the same block declaration and resolution primitives as every other worker — distinguished only by its observability scope.

---

## Coordination Mechanisms

| Need | Mechanism |
|---|---|
| Cluster membership | Gossip — Serf SWIM protocol |
| Block state convergence | Gossip + 2P-Set CRDT |
| Block resolution lock | Lightweight lease (etcd / Redis SETNX) |
| Task delivery | At-least-once queue with idempotency keys |
| Lineage termination | Mechanical — terminal task success |

---

## Implementation-Defined Parameters

The architecture is agnostic to the following — treat them as deployment decisions:

- **Worker capability model** — fully agnostic or capability-tagged via Serf node tags
- **Error classification** — criteria for Blocked / Transient / Fatal are domain-specific
- **State store backend** — S3, Redis, etcd, or equivalent
- **Queue backend** — NATS JetStream, Redis/Asynq, AWS SQS
- **Quorum parameters** — confirmations required, refutations to dismiss, confirmation window
- **MaxDepth** — cycle guard ceiling (recommended default: 16)
- **Stall threshold** — Scanner activation delay (recommended: 3× P95 task duration)

---

## Safety Properties

| Property | Mechanism |
|---|---|
| No infinite loops | Depth counter with hard MaxDepth cap |
| Idempotent execution | Deterministic task IDs (SHA-256 of type + payload ref + origin) |
| No lost lineage branches | Enqueue next tasks before acking current task |
| No phantom blocks | Quorum confirmation required; single refutation dismisses |
| No stale block reactivation | 2P-Set CRDT — removal is permanent |
| Worker failure recovery | Serf SWIM detects departure; peers re-initiate abandoned claims |

---

## Open Questions

This is a first-draft proposition. The following are unresolved and represent areas for further design work:

- **Back-pressure** — fan-out can flood the queue; rate limiting strategy TBD
- **Trust boundaries** — quorum helps but does not protect against coordinated bad actors
- **Observability** — gossip-coordinated systems require an explicit event stream for tracing
- **Fan-in synchronization** — no native barrier primitive; barrier task type + completion counter is the recommended pattern
- **Initial decomposition quality** — the dependency tree quality depends on how precisely first workers articulate block declarations

---

## White Paper

The full architectural proposition is available in [`GRAIL_Whitepaper.docx`](./GRAIL_Whitepaper.docx).

---

## Relationship to Existing Work

| System / Concept | Relationship |
|---|---|
| Temporal.io | Durable workflow execution — GRAIL differs in using gossip coordination and goal regression rather than a central workflow service |
| Celery / Machinery | Task queue worker pools — GRAIL adds gossip-coordinated block state and goal-regression planning |
| LangGraph / CrewAI | LLM-specific orchestration — GRAIL is worker-agnostic with stronger coordination consistency |
| Serf (HashiCorp) | SWIM-protocol gossip — used directly as the coordination transport |
| CRDT literature (Shapiro et al.) | 2P-Set semantics drawn from the formal specification |
| AI Goal Regression | Classical backwards chaining applied to a distributed multi-worker system |

---

## Status

**Version 0.1 — Architecture Proposition**

This repository contains the white paper and this README. A reference implementation is not yet available. Contributions, critique, and questions are welcome via Issues.

---

*GRAIL — Goal-Regressive Adaptive Inference Layer*
