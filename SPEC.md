# DICE Protocol v0.1.0

**A control protocol for LLM agents managing real-time systems.**

> An LLM thinks in seconds. A simulation ticks in milliseconds.  
> DICE bridges that gap with **D**eclarative policy, **I**mperative commands, **C**ontinuous execution, and **E**vent-driven rules — letting slow minds control fast worlds.

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Design Principles](#2-design-principles)
3. [Architecture](#3-architecture)
4. [Core Primitives](#4-core-primitives)
5. [Data Schemas](#5-data-schemas)
6. [Transport Bindings](#6-transport-bindings)
7. [Execution Model](#7-execution-model)
8. [Authentication](#8-authentication)
9. [Reactive Rules Engine](#9-reactive-rules-engine)
10. [Reference Implementation](#10-reference-implementation)
11. [Implementing a New Server](#11-implementing-a-new-server)
12. [Appendix: Comparison to Existing Protocols](#12-appendix)

---

## 1. Introduction

### The Problem

LLM agents need to control real-time systems — game simulations, robotics, infrastructure, smart environments, trading engines. But LLMs are slow (1–30 seconds per inference) while these systems run at tick rates of 1–100 Hz. A naive request-response protocol forces the agent to poll constantly, wasting tokens and missing events between calls.

### The Solution

DICE introduces a **hybrid control model** with four complementary layers:

| Mechanism | Speed | Persistence | Use Case |
|-----------|-------|-------------|----------|
| **Declarative Policy** | Set once, executes continuously | Persistent between calls | "Keep 55% workers, 25% soldiers" |
| **Reactive Rules** | Fires at tick speed | Persistent, co-evaluated with policy | "If food < 75, retreat" |
| **Imperative Commands** | One-shot, immediate | Ephemeral (until cleared) | "Move unit 42 to (75, 50)" |

The agent sets *intent* through policy and rules, then issues *directives* through commands. The simulation executes autonomously between agent calls, with reactive rules bridging the latency gap.

### Naming

**DICE** = **D**eclarative, **I**mperative, **C**ontinuous, **E**vent-driven.

The four words describe the four layers of the protocol:

| Letter | Layer | Description |
|--------|-------|-------------|
| **D** | Declarative Policy | Set behavior once, sim executes it every tick |
| **I** | Imperative Commands | One-shot overrides for fine-grained control |
| **C** | Continuous Execution | Simulation runs autonomously between agent calls |
| **E** | Event-Driven Rules | Reactive rules fire at simulation speed |

Named after the four architectural pillars, not any single implementation.

---

## 2. Design Principles

1. **Policy over micromanagement.** The agent declares behavior; the simulation executes it. An LLM that sets good policy and lets the sim run will outperform one that tries to command every tick.

2. **Rules bridge the latency gap.** Reactive rules fire at simulation speed, not agent speed. An agent that can't react in 100ms can still write rules that do.

3. **Observation drives decisions.** Rich, structured state snapshots let the agent reason about the world without parsing raw logs.

4. **Commands are overrides, not the default.** Imperative commands override policy temporarily. They're for moments that need precision, not steady-state control.

5. **Transport-agnostic.** The protocol defines operations and schemas. It can ride on MCP, REST, WebSocket, gRPC, or any message bus.

6. **Fail-safe by default.** If the agent disconnects, the simulation continues with the last-set policy. No agent = last known good state, not a crash.

---

## 3. Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    LLM AGENT                            │
│                                                         │
│   ┌──────────┐   ┌──────────┐   ┌──────────────────┐   │
│   │ Observer  │   │ Planner  │   │ Command Issuer   │   │
│   │ (read)    │──▶│ (think)  │──▶│ (write)          │   │
│   └──────────┘   └──────────┘   └──────────────────┘   │
│        ▲                              │                 │
└────────┼──────────────────────────────┼─────────────────┘
         │                              │
    ┌────┴────┐                    ┌────┴────┐
    │ OBSERVE │                    │  ACT    │
    └────┬────┘                    └────┬────┘
         │                              │
         ▼                              ▼
┌─────────────────────────────────────────────────────────┐
│                 SIMULATION SERVER                        │
│                                                         │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │
│   │ World State   │  │ Policy Engine│  │ Command Queue │ │
│   │ (tick-loop)   │  │ (directives) │  │ (one-shot)    │ │
│   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘ │
│          │                  │                  │         │
│          ▼                  ▼                  ▼         │
│   ┌─────────────────────────────────────────────────┐   │
│   │              TICK SIMULATION                     │   │
│   │  1. Process command queue (imperative overrides) │   │
│   │  2. Evaluate reactive rules (triggers)           │   │
│   │  3. Execute policy (directive-driven behavior)   │   │
│   │  4. Step world state (physics, AI, economics)    │   │
│   │  5. Collect notifications (push to agent)        │   │
│   └─────────────────────────────────────────────────┘   │
│                                                         │
│   ┌──────────────┐  ┌──────────────┐                    │
│   │ Notifications │  │ Event Log    │                    │
│   │ (push alerts) │  │ (history)    │                    │
│   └──────────────┘  └──────────────┘                    │
└─────────────────────────────────────────────────────────┘
```

### Key Insight: The Simulation Doesn't Wait

The agent calls OBSERVE → THINK → ACT. Between those calls, the simulation runs autonomously. The agent's policy and reactive rules continue executing at tick speed. The agent is a *supervisor*, not a *driver*.

---

## 4. Core Primitives

These are the abstract operations any DICE-compatible server must support.

### 4.1 OBSERVE — Read World State

```
OBSERVE(entity_id) → WorldState
```

Returns a structured snapshot of the simulation state for the given entity. Includes resource counts, controlled entities, environmental data, combat/economic metrics, and derived intelligence.

**Required fields in response:**
- `entity` — identity and status of the controlled entity
- `resources` — available resource pools with income rates
- `controlled_units` — list of units/agents under this entity's control
- `environment` — world state visible to this entity (fog of war, etc.)
- `derived` — computed metrics (e.g., military summary, economic health)

### 4.2 POLICY_GET — Read Current Policy

```
POLICY_GET(entity_id) → Policy
```

Returns the current policy document (directive) for the entity. The policy is persistent — it survives between agent calls and continues executing when the agent is disconnected.

### 4.3 POLICY_SET — Update Policy

```
POLICY_SET(entity_id, patches) → Policy
```

Applies partial updates to the entity's policy. Uses merge semantics — only specified fields are changed, all others persist. This is the primary way the agent shapes behavior over time.

**Semantics:**
- Nested objects are merged recursively
- Scalar values are replaced
- Empty objects `{}` replace (not merge) the target field
- Arrays are replaced wholesale (not appended)

### 4.4 ACT — Issue Imperative Command

```
ACT(entity_id, command) → CommandResult
```

Issues a one-shot command that overrides policy for a specific unit or action. Commands persist until:
- The unit/agent dies or is destroyed
- The agent issues a CLEAR command
- The game/session resets

**Command types:**
| Command | Scope | Description |
|---------|-------|-------------|
| `move_to` | Any unit | Navigate to coordinates, engage enemies en route |
| `attack` | Combat units | Advance to coordinates with target priority |
| `gather` | Worker units | Harvest a resource node indefinitely |
| `build` | Worker units | Construct a structure at coordinates |
| `hold` | Any unit | Hold position, fight within radius, don't advance |
| `patrol` | Any unit | Loop through waypoints, engage enemies |
| `clear` | Any unit | Remove override, return to policy-driven behavior |

**Batch variants:**
- `ACT_BATCH(entity_id, [command...])` — multiple commands in one call
- `ACT_BY_TYPE(entity_id, unit_type, command, filter_state?)` — all units of a type

### 4.5 SUBSCRIBE — Register for Push Alerts

```
SUBSCRIBE(entity_id) → [Notification]
```

Returns all pending notifications since the last read. Notifications are consumed on read (pull-based pub/sub). If the agent misses notifications, they can be recovered from the event log.

**Notification types:**
| Type | Urgency | Description |
|------|---------|-------------|
| `critical` | Immediate | Entity under direct threat (e.g., headquarters attacked) |
| `contact` | High | Enemy/adversary detected in sensor range |
| `depletion` | Medium | Resource node exhausted |
| `completion` | Low | Construction or upgrade finished |
| `game_over` | Terminal | Simulation ended, outcome determined |

### 4.6 LIFECYCLE — Session Management

```
CREATE_SESSION(config?) → SessionInfo
JOIN_SESSION(entity_id, agent_name) → AuthToken
RELEASE_SESSION(entity_id) → void
CONTROL_SESSION(action) → SessionResult
```

Manages agent identity, session isolation, and simulation lifecycle.

**Actions:**
- `start` — begin simulation from lobby
- `pause` — freeze simulation (policy still accepted)
- `resume` — unfreeze
- `end` — terminate, adjudicate winner
- `reset` — new simulation, clean state

---

## 5. Data Schemas

### 5.1 WorldState

```json
{
  "tick": 450,
  "phase": "running",
  "elapsed_s": 450.0,
  "entity": {
    "id": 0,
    "name": "RED",
    "queen_hp": 900,
    "queen_max_hp": 900
  },
  "resources": {
    "food": 342,
    "food_max": 600,
    "income_per_s": 38.5,
    "dirt": 180,
    "dirt_per_s": 12.0
  },
  "counts": {
    "workers": 30,
    "soldiers": 15,
    "scouts": 4,
    "total": 49
  },
  "tiers": {
    "worker": 1,
    "scout": 0,
    "soldier": 1
  },
  "spawn_queue": {
    "entries": [
      {"type": "soldier", "build_time_remaining": 12},
      {"type": "worker", "build_time_remaining": 28}
    ],
    "reserved_food": 50
  },
  "combat": {
    "soldiers_in_siege": 0,
    "soldiers_adjacent_queen": 0,
    "enemy_queen_hp": null,
    "queen_dps_actual": 0.0,
    "siege_dps_potential": 0.0,
    "siege_hint": null
  },
  "military_summary": {
    "total_soldiers": 15,
    "fighting": 3,
    "patrolling": 10,
    "idle": 2,
    "healthy": 14,
    "wounded": 1,
    "avg_hp_pct": 92.5,
    "building": 0
  },
  "units": [
    {
      "id": 42,
      "type": "soldier",
      "state": "patrolling",
      "hp": 200,
      "max_hp": 200,
      "x": 75,
      "y": 50,
      "carrying": false,
      "override": null
    }
  ],
  "food_intel": {
    "75,50": {"amt": 3800, "max": 4000, "tier": "frontline", "last_seen": 445}
  },
  "enemy_sightings": [
    [80, 50, 3, 5, 440, 1, 1]
  ],
  "own_structures": [
    {"type": "watchtower", "x": 25, "y": 50, "hp": 150, "max_hp": 150}
  ],
  "advisor": [
    "Build larder — 180◆ available, +6♦/tick passive income"
  ]
}
```

### 5.2 Policy (Directive)

```json
{
  "spawn": {
    "worker": {"target_ratio": 0.55, "min": 4, "max": 55, "pause": false},
    "soldier": {"target_ratio": 0.25, "min": 2, "max": 30, "pause": false},
    "scout": {"target_ratio": 0.20, "min": 2, "max": 12, "pause": false},
    "reserve_food": 50,
    "burst_at": 800
  },
  "economy": {
    "upgrade_priority": ["scout", "worker", "soldier"],
    "auto_upgrade": true,
    "priority_food": null,
    "gather_dirt": true,
    "upgrade_reserve": {}
  },
  "military": {
    "stance": "balanced",
    "auto_attack": true,
    "retreat": false,
    "siege_priority": "nearest",
    "rally_point": null,
    "rally_release_at": null
  },
  "unit_types": {
    "worker": {"flee_distance": 4},
    "soldier": {"expansion": [1, 0]},
    "scout": {"expansion": [1, 0], "revisit_pct": 0.12}
  },
  "triggers": [
    {
      "label": "eco_emergency",
      "if": "food < 75 AND elapsed_ticks > 100",
      "then": {"military": {"retreat": true}},
      "priority": 5,
      "cooldown": 50
    }
  ]
}
```

### 5.3 Trigger Rule

```json
{
  "label": "string — unique identifier for this rule",
  "if": "string — boolean expression evaluated against simulation state",
  "then": "object — policy patches to apply when condition is true",
  "priority": "number — higher fires first (default: 1)",
  "cooldown": "number — minimum ticks between firings (default: 0 = every tick)",
  "once": "boolean — if true, rule disables itself after first fire (default: false)"
}
```

**Condition language:** Boolean expressions evaluated against a namespace derived from the current world state. Supports:
- Resource comparisons: `food < 75`, `dirt >= 150`
- Unit counts: `soldier_count > 10`, `worker_count < 5`
- Time gates: `elapsed_ticks > 100`
- Compound logic: `AND`, `OR`, `NOT`
- Derived metrics: `queen_hp_pct < 0.50`, `income_per_s < 10`

**Then-actions:** Policy patches applied using the same merge semantics as POLICY_SET. Special actions:
- `{"buy_upgrade": "scout"}` — purchase the next upgrade for a unit type
- `{"military.retreat": true}` — trigger full retreat
- `{"military.siege_priority": "queen"}` — switch targeting priority

### 5.4 Command Envelope

```json
{
  "type": "unit_command | unit_command_batch | build | convert | buy_upgrade | cancel_spawn",
  "ant_id": "number — target unit ID (for unit_command)",
  "command": "move_to | attack | gather | build | hold | patrol | clear",
  "x": "number — target x coordinate",
  "y": "number — target y coordinate",
  "waypoints": "[[x,y], ...] — for patrol command"
}
```

### 5.5 Notification

```json
{
  "type": "structure_complete | upgrade_complete | queen_under_attack | enemy_contact | food_depleted | game_over | priority_food_cleared",
  "data": {
    "type-specific payload"
  },
  "tick": 450
}
```

**Payloads by type:**
- `structure_complete`: `{"type": "watchtower", "x": 25, "y": 50}`
- `upgrade_complete`: `{"unit": "scout", "tier": 1, "label": "Pathfinding", "effect": "vision 8→12"}`
- `queen_under_attack`: `{"hp": 750, "old_hp": 900, "dmg": 150}`
- `enemy_contact`: `{"cx": 80, "cy": 50, "soldiers": 3, "total": 5}`
- `food_depleted`: `{"x": 30, "y": 80, "tier": "home"}`
- `game_over`: `{"winner": 0, "outcome": "victory", "reason": "enemy queen eliminated"}`

### 5.6 Session Info

```json
{
  "session_id": "uuid",
  "match_id": "string",
  "entity_id": 0,
  "agent_name": "Hermes",
  "token": "bearer-token-for-write-operations",
  "ws_url": "ws://server:8083/ws/match-id",
  "phase": "lobby"
}
```

---

## 6. Transport Bindings

The protocol is transport-agnostic. Here are the standard bindings.

### 6.1 MCP Binding (Model Context Protocol)

Maps each primitive to an MCP tool. This is the primary binding for LLM agents using Claude, GPT, or similar models.

| Primitive | MCP Tool |
|-----------|----------|
| OBSERVE | `get_state(entity_id)`, `get_intel_map(entity_id)` |
| POLICY_GET | `get_directive(entity_id)` |
| POLICY_SET | `patch_directive(entity_id, patches)`, `set_directive(entity_id, directive)` |
| ACT | `command_unit(...)`, `command_units(...)`, `command_type(...)`, `buy_upgrade(...)`, `build_structure(...)`, `convert_unit(...)`, `cancel_spawn(...)` |
| SUBSCRIBE | `get_notifications(entity_id)`, `get_events(entity_id, since_tick)` |
| LIFECYCLE | `create_match(tps)`, `join_seat(entity_id, name)`, `release_seat(entity_id)`, `game_control(action)` |
| META | `get_tick()`, `send_chat(msg)`, `submit_feedback(text, category)` |

**Transport:** stdio (default) or HTTP+SSE via `--port` flag.

### 6.2 REST Binding

Maps primitives to HTTP endpoints. Suitable for web clients, scripts, and non-MCP integrations.

```
GET    /api/tick                         → TickInfo
GET    /api/seats                        → [SeatInfo]
POST   /api/matches                      → MatchInfo
POST   /api/matches/{id}/seat/{eid}      → SessionInfo
DELETE /api/matches/{id}/seat/{eid}      → {ok}
POST   /api/control                      → {action result}

GET    /api/state/{eid}                  → WorldState
GET    /api/notifications/{eid}          → [Notification]
GET    /api/events/{eid}                 → [Event]
GET    /api/intel_map/{eid}              → IntelMap
GET    /api/directive/{eid}              → Policy

POST   /api/directive/{eid}              → {patches: {...}} or {directive: {...}}
POST   /api/command/{eid}                → CommandEnvelope
POST   /api/feedback                     → {ok}
POST   /api/chat                         → {ok}
```

**Authentication:** Bearer token in `Authorization` header for write endpoints. Read endpoints are open.

### 6.3 WebSocket Binding

Real-time state stream for clients that need continuous updates.

```
Connection: ws://server/ws/{match_id}

Server → Client messages:
  {type: "state", data: WorldState}        — periodic state broadcast
  {type: "notification", data: Notification} — push alerts
  {type: "chat", data: ChatMessage}        — game chat

Client → Server messages:
  {type: "command", data: CommandEnvelope} — imperative commands
  {type: "directive", data: PolicyPatch}   — policy updates
  {type: "start_game"}                     — begin simulation
  {type: "reset"}                          — reset to lobby
```

---

## 7. Execution Model

### The Tick Loop

```
Every tick (1/N seconds):
  1. Process command queue — apply imperative overrides
  2. Evaluate reactive rules — fire triggers whose conditions are met
  3. Apply policy — directive-driven behavior (spawn, economy, movement)
  4. Step world — physics, AI, combat, resource extraction
  5. Collect notifications — queue alerts for agent consumption
```

### The Agent Loop

```
While game is running:
  1. OBSERVE    — get_state()                    [~100ms]
  2. THINK      — LLM inference                  [1–30s]
  3. DECIDE     — analyze state, choose actions
  4. ACT        — patch_directive + command_unit  [~100ms]
  5. (sim runs autonomously for N ticks)
  6. SUBSCRIBE  — get_notifications()            [~100ms]
  7. REPEAT
```

### Latency Budget

The agent doesn't need to keep up with the simulation tick rate. The protocol is designed for agents that are **orders of magnitude slower** than the sim:

| Component | Speed | Notes |
|-----------|-------|-------|
| Simulation tick | 10–1000 Hz | Game physics, AI, economics |
| Reactive rules | Tick speed | Fires every tick when condition met |
| Agent inference | 0.1–30 Hz | LLM call latency |
| Agent commands | Batched | Multiple actions per inference cycle |

**The gap is bridged by:**
1. **Declarative policy** — set once, executes every tick
2. **Reactive rules** — write code that runs at tick speed
3. **Batched commands** — issue multiple actions per call
4. **Persistent state** — simulation continues with last-known policy

### Optimal Play Pattern

The most effective agent strategy is:

1. **Set comprehensive policy at game start** (5–10 tool calls)
2. **Configure reactive rules for emergencies** (triggers)
3. **Check state at natural breakpoints** (every 50–200 ticks)
4. **Issue batch commands for major decisions** (attack, retreat, expand)
5. **Let the simulation run** — don't micromanage

Agents that try to play like humans (constant polling, individual commands) waste tokens and miss the strategic advantage of policy-driven control.

---

## 8. Authentication

### Token Model

- **Seat-based bearer tokens** — each entity (colony/player) gets a unique token on join
- **Scope-limited** — token only authorizes writes for the claimed entity
- **Auto-managed** — the MCP bridge stores tokens transparently; agents don't track them
- **Revoked on** — session release, game reset, or rejoin

### Permission Matrix

| Operation | Auth Required | Scope |
|-----------|--------------|-------|
| OBSERVE (all read endpoints) | No | Public |
| POLICY_SET | Yes | Entity-scoped |
| ACT (all commands) | Yes | Entity-scoped |
| LIFECYCLE (join/release) | Yes | Entity-scoped |
| LIFECYCLE (create/list/control) | No | Game-wide |

### Token Flow

```
Agent → JOIN_SESSION(entity_id, name) → {token, session_id}
Agent → POLICY_SET(entity_id, patches, token) → {ok}
Agent → ACT(entity_id, command, token) → {ok}
Agent → RELEASE_SESSION(entity_id, token) → {ok}  // token revoked
```

---

## 9. Reactive Rules Engine

The reactive rules engine is what makes DICE suitable for real-time control. It allows agents to write **event-driven code that executes at simulation speed**, bridging the gap between slow agent inference and fast simulation ticks.

### Rule Evaluation

Every tick, the simulation:
1. Builds a namespace from current world state (resources, unit counts, time, derived metrics)
2. Evaluates each rule's `if` expression against this namespace
3. For rules where the condition is true AND cooldown has elapsed:
   - Applies the `then` patches to the entity's policy
   - Logs the fire event
   - Decrements cooldown

### Rule Priority

Rules are evaluated in priority order (highest first). If multiple rules fire on the same tick, their patches are applied in priority order. Later patches overwrite earlier ones for the same field.

### Cooldown

The `cooldown` field prevents rule spam. After firing, a rule cannot fire again for `cooldown` ticks. This is essential because:
- Conditions like `food < 75` may remain true for many consecutive ticks
- Without cooldown, the trigger fires every tick, flooding the event log
- Cooldown lets the rule fire, apply its effect, and wait to see if the effect worked

### One-Shot Rules

Setting `once: true` makes a rule disable itself after its first fire. Useful for one-time responses:
```json
{
  "label": "first_enemy_sighting",
  "if": "enemy_sighting_count > 0",
  "then": {"military": {"stance": "aggressive"}},
  "once": true
}
```

### Rule vs. Policy: When to Use Which

| Use Policy For | Use Rules For |
|----------------|---------------|
| Steady-state behavior (ratios, stances) | Conditional responses (emergencies) |
| Things the agent sets and forgets | Things that depend on runtime state |
| "Always have 55% workers" | "If queen HP < 50%, switch to defensive" |
| "Auto-upgrade workers" | "If food > 600, buy scout upgrade" |

---

## 10. Reference Implementation

**Agants** (https://github.com/DatTheMaster/agants) is the reference implementation of the DICE protocol.

### Components

| Component | Role | Protocol Layer |
|-----------|------|---------------|
| `server.py` | Simulation engine + REST API + WebSocket | World state, tick loop, policy engine |
| `engine/` | Game constants, colony AI, world simulation | Domain logic |
| `mcp_server.py` | MCP tool bridge (FastMCP) | Transport binding |
| `bot.py` | Bot AI for unclaimed entities | Fallback agent |
| `frontend/` | Web viewer + chat | Client binding |

### Protocol Mapping

| Protocol Primitive | Agants Implementation |
|-------------------|----------------------|
| OBSERVE | `GET /api/state/{id}` → `WorldState` with units, resources, combat, intel |
| POLICY_GET | `GET /api/directive/{id}` → full directive JSON |
| POLICY_SET | `POST /api/directive/{id}` with `patches` or full `directive` |
| ACT | `POST /api/command/{id}` with command envelope |
| SUBSCRIBE | `GET /api/notifications/{id}` (consumes on read) |
| LIFECYCLE | `POST /api/matches`, `POST /api/seat/{id}`, `DELETE /api/seat/{id}` |
| Reactive Rules | `DirectiveEngine.eval_triggers()` — runs every tick |
| Notifications | `Colony.notifications` queue — pushed by `check_alerts()` |

### Stats (as of v2.13)

- **22 MCP tools** covering all protocol primitives
- **6 resource types** (food, dirt, and their derivatives)
- **8 unit states** (idle, foraging, returning, exploring, fighting, patrolling, recruited, building)
- **5 structure types** (watchtower, barracks, guard_post, wall, larder)
- **3 upgrade tiers** per unit type
- **Trigger system** with cooldown, priority, and one-shot support
- **Notification system** with 7 event types
- **Multi-match support** — concurrent independent simulations
- **Token auth** — seat-scoped bearer tokens for write operations

---

## 11. Implementing a New Server

To build a DICE-compatible server for a different domain:

### Step 1: Define Your World State

Map your domain to the WorldState schema:
- `entity` — what the agent controls
- `resources` — what the agent manages (money, energy, materials, etc.)
- `controlled_units` — agents/robots/workers under command
- `environment` — what the agent can observe
- `derived` — computed metrics the agent should see

### Step 2: Define Your Policy Schema

Translate your domain's continuous behavior into a directive:
- Spawn/production ratios → `spawn` section
- Economic targets → `economy` section
- Combat/behavior stances → `military` section
- Per-unit configuration → `unit_types` section

### Step 3: Implement Reactive Rules

Build a trigger evaluator that:
1. Parses condition expressions against current state
2. Applies `then` patches as policy updates
3. Respects cooldown and priority
4. Runs at simulation tick speed

### Step 4: Implement the Command Queue

Process imperative commands each tick:
- Validate commands (unit exists, action possible)
- Apply overrides to specific units
- Handle batch operations
- Auto-clear overrides on unit death/completion

### Step 5: Bind to a Transport

Implement one or more transport bindings:
- **MCP** — wrap your API in MCP tools (use FastMCP for quick setup)
- **REST** — expose HTTP endpoints for the primitives
- **WebSocket** — real-time state streaming

### Step 6: Add Notifications

Queue notifications for events the agent should know about:
- Threats (high urgency)
- Resource changes (medium urgency)
- Completions (low urgency)
- Terminal events (game over)

---

## 12. Appendix: Comparison to Existing Protocols

### vs. MCP (Model Context Protocol)

MCP is a **tool-calling protocol** — it exposes functions an LLM can invoke. DICE uses MCP as a transport binding but adds:
- **Persistent policy** (MCP has no concept of stateful behavior between calls)
- **Reactive rules** (MCP has no event-driven execution model)
- **Real-time execution model** (MCP assumes request-response, not continuous control)

MCP tells the agent "here are tools you can call." DICE tells the agent "here's how to control a system that runs faster than you."

### vs. A2A (Agent-to-Agent, Google)

A2A is for **inter-agent communication** — agents discovering and delegating to other agents. DICE is for **agent-to-system control** — an agent managing a real-time simulation. They're complementary: an A2A agent could delegate a subtask to a DICE-controlled system.

### vs. ACP (Agent Communication Protocol, Anthropic)

ACP focuses on **agent interoperability** — standardizing how agents talk to each other. DICE focuses on **agent-system control** — standardizing how agents manage real-time systems. Different layers of the stack.

### vs. REST APIs

A raw REST API gives you endpoints. DICE gives you a **control model** — the distinction between policy, rules, and commands; the execution model; the notification system. A REST API is a transport binding. DICE is the architecture on top of it.

---

## License

This specification is released under the MIT License.

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 0.1.0 | 2026-06-10 | Initial specification |

---

*Built by agents, for agents.*
