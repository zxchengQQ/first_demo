---
title: "Hermes Agent Kanban Swarm Architecture — Deep Analysis"
date: 2026-07-03
tags: ["Hermes Agent", "Kanban", "Swarm", "Architecture", "Multi-Agent"]
author: "Hermes Agent Publisher"
---

> Research for Swarm: t_540f8eb5 | Worker: researcher (t_80e50e70)
> Date: 2026-07-03 | Version: v0.15/v0.16

---

## 1. Core Finding

`hermes kanban swarm` is a topology helper that creates a complete multi-agent DAG in one command: a completed root/blackboard card, N parallel worker cards, a verifier card gated on all workers, and a synthesizer card gated on the verifier — all wired via durable parent-child dependency links in the SQLite kanban board.

---

## 2. Architecture Layers

### 2.1 Infrastructure Layer

| Component | Role |
|-----------|------|
| **SQLite DB** | `~/.hermes/kanban.db` (or per-board DB) — durable task store |
| **Dispatcher** | Gateway-embedded loop (60s tick default) — reclaims stale claims, promotes tasks `todo→ready` when all parents done, spawns workers |
| **Worker lanes** | OS processes — each worker is a full Hermes profile instance |
| **Kanban tools** | `kanban_show / kanban_complete / kanban_block / kanban_heartbeat / kanban_comment / kanban_create / kanban_link` — the model's interface to the board |

### 2.2 Topology Layer (the Swarm Graph)

```
                     ┌──────────────────────────────┐
                     │  Root / Blackboard (t_root)   │
                     │  (completed immediately)       │
                     │  JSON comments = shared state  │
                     └──────────┬───────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              │                 │                 │
     ┌────────▼──────┐ ┌───────▼───────┐ ┌──────▼────────┐
     │ Worker 1      │ │ Worker 2      │ │ Worker N      │
     │ (researcher)  │ │ (architect)   │ │ (sre)         │
     │ assignee=A    │ │ assignee=B    │ │ assignee=C    │
     └───────┬───────┘ └───────┬───────┘ └───────┬───────┘
             │                 │                 │
             └─────────────────┼─────────────────┘
                               │ (ALL must complete)
                      ┌────────▼────────┐
                      │  Verifier       │
                      │  (reviewer)     │
                      │  gates the work │
                      └────────┬────────┘
                               │ (verifier must pass)
                      ┌────────▼────────┐
                      │  Synthesizer    │
                      │  (writer)       │
                      │  merges output  │
                      └─────────────────┘
```

### 2.3 Communication Layer (Blackboard Pattern)

- **Blackboard = structured JSON comments on the root card**
- Workers write findings via `kanban_comment(task_id=t_root, body="[swarm:blackboard] {...}")`
- Workers read context via `kanban_show()` which returns the full comment thread
- Pattern: `{"key": "findings", "value": {...}}` — keyed so workers can find relevant data
- No direct worker-to-worker communication; all state flows through the root

### 2.4 Orchestrator Layer

When `kanban.auto_decompose: true` (default), tasks landing in `triage` column are automatically decomposed by an auxiliary LLM:

1. Reads your installed profiles with descriptions
2. Produces a JSON task graph: which tasks to spawn, assigned to whom, dependency links
3. Original triage task becomes parent of all leaf tasks
4. Completes when the entire subgraph finishes

Profiles are discovered at runtime — the orchestrator must call `hermes profile list` or equivalent before creating child tasks, otherwise the dispatcher silently drops tasks with unknown assignees.

---

## 3. Command Reference

### `hermes kanban swarm` — One-shot topology builder

```bash
hermes kanban swarm "<goal-string>" \
  --workers <profile1>,<profile2>,...   # N parallel workers (comma-sep)
  --worker <profile>:<label>            # or repeat per worker (with label)
  --verifier <profile>                  # gates after all workers
  --synthesizer <profile>               # gates after verifier
  [--json]                              # machine-readable output
```

**Alias form (repeatable):**
```bash
hermes kanban swarm "Design failover" \
  --worker researcher:Research --worker architect:Architecture \
  --verifier reviewer --synthesizer writer
```

### What it creates atomically

| Card | Status | Dependencies | Role |
|------|--------|-------------|------|
| Root | `done` | none | Blackboard; stores shared context as comments |
| Worker 1..N | `ready` | Root | Parallel work, each assigned to a profile |
| Verifier | `todo` | All workers | Reviews all worker outputs |
| Synthesizer | `todo` | Verifier | Merges into final deliverable |

### Output (with --json)

```json
{
  "root_id": "t_root",
  "worker_ids": ["t_w1", "t_w2", "t_w3"],
  "verifier_id": "t_v1",
  "synthesizer_id": "t_s1"
}
```

---

## 4. Worker Lifecycle (per dispatched task)

1. **Orient** — `kanban_show()` reads title, body, parent handoffs, prior attempts, comment thread
2. **Work** — `cd $HERMES_KANBAN_WORKSPACE` (scratch/dir/worktree), do the work via terminal/file tools
3. **Heartbeat** — `kanban_heartbeat(note="...")` every few minutes (at least once per hour for long tasks)
4. **Handoff** — `kanban_complete(summary="...", metadata={...})` or `kanban_block(reason="...")`

**Protocol enforcement**: If the worker process exits with status 0 while the task is still `running`, the dispatcher emits a `protocol_violation` event and auto-blocks the task — the model must explicitly complete or block itself.

---

## 5. Key Design Decisions

### Why tools instead of CLI shelling

1. **Backend portability** — kanban tools run in the agent's Python process, reaching `~/.hermes/kanban.db` regardless of terminal backend (Docker/Modal/SSH)
2. **No shell-quoting issues** — structured tool args vs. shlex fragility
3. **Better error handling** — structured JSON vs. stderr parsing

### Why kanban vs delegate_task

| Dimension | delegate_task | Kanban Swarm |
|-----------|--------------|--------------|
| Shape | RPC call (fork → join) | Durable message queue + state machine |
| Parent | Blocks until child returns | Fire-and-forget after create |
| Child identity | Anonymous subagent | Named profile with persistent memory |
| Survivability | Lost on parent crash | Survives restarts (SQLite) |
| Human-in-loop | Not supported | Comment / unblock at any point |
| Audit trail | Lost on context compression | Durable rows forever |

### Why the blackboard pattern

- Workers never talk directly to each other
- All state flows through the root card's comment thread
- Any worker can read any other worker's findings via `kanban_show()` on the root
- Human can also read/write the blackboard via `/kanban comment`

---

## 6. Edge Cases & Guarantees

| Scenario | Behavior |
|----------|----------|
| Worker crashes | Dispatcher reclaims after stale TTL (default 4h); task goes back to `ready` |
| Worker exits without completing | `protocol_violation` → auto-block (not re-tried) |
| Verifier fails/rejects | Task blocks; human reviews. Synthesizer never reaches `ready` |
| Human needs to intervene | `/kanban comment t_id "..."` lands on task thread; worker reads it next time |
| Multiple boards | Isolation via `HERMES_KANBAN_BOARD` env var; per-board workspace/log dirs |
| Respawn guard | Dispatcher refuses re-spawn on `blocker_auth`, `recent_success`, `active_pr` |
| Circuit breaker | After `kanban.failure_limit` consecutive failures (default 2), auto-block with last error |

---

## 7. Comparison: Current Swarm vs External Descriptions

| Source | Description |
|--------|-------------|
| Hermes Docs (kanban.md) | "completed root/blackboard card, N parallel worker cards, a verifier card gated on all workers, and a synthesizer card gated on the verifier" |
| FlowStacks | "Fan a goal out to N parallel workers, gate a verifier on all of them, then a synthesizer on the verifier" |
| TECHSY v0.15 Review | "Root node fans work to parallel workers; verifier checks output; synthesizer merges; shared blackboard for state" |

All sources are consistent. The current swarm (`t_80e50e70` + `t_54fa0933` + `t_59fa3c47` + `t_12fa5805`) precisely matches the documented topology.

---

## 8. Recommendations for Writers

1. **Blackboard protocol**: Workers should post findings as structured JSON comments on root/t_540f8eb5 using key-value format for easy parsing
2. **Verifier gates**: The verifier must check that all workers left structured metadata in `kanban_complete()`, not just prose summaries
3. **Synthesizer context**: The synthesizer reads both the root blackboard and each worker's completion metadata to produce the final article
4. **Scratch workspace note**: This worker's workspace (`/kanban/boards/blog/workspaces/t_80e50e70`) is ephemeral — will be deleted on completion
