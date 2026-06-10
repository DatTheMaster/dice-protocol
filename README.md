# Agants Protocol

**A control protocol for LLM agents managing real-time systems.**

[Read the Spec →](SPEC.md)

## What Is This?

LLMs are slow (seconds per inference). Real-time systems are fast (milliseconds per tick). The Agants Protocol bridges that gap with three complementary control mechanisms:

- **Declarative Policy** — set behavior once, sim executes it every tick
- **Reactive Rules** — write condition-action rules that fire at sim speed
- **Imperative Commands** — one-shot overrides for fine-grained control

The agent is a *supervisor*, not a *driver*. It sets intent and lets the simulation run.

## Why?

Every game, simulation, or real-time system controlled by an LLM faces the same problem: the agent can't keep up. Existing protocols (MCP, REST) assume request-response. Agants assumes the system runs autonomously and the agent shapes its behavior through policy.

## Quick Start

```python
# 1. Agent observes the world
state = get_state(entity_id=0)
# → {food: 342, soldiers: 15, enemy_sightings: [...]}

# 2. Agent sets policy (persists between calls)
patch_directive(entity_id=0, patches={
    "spawn": {"soldier": {"target_ratio": 0.4}},
    "military": {"stance": "aggressive"}
})

# 3. Agent issues commands (one-shot overrides)
command_unit(entity_id=0, ant_id=42, command="attack", x=75, y=50)

# 4. Simulation runs autonomously — policy executes every tick
# 5. Reactive rules fire at tick speed:
#    "if food < 75 then retreat" — no agent call needed
# 6. Agent checks notifications periodically
notifications = get_notifications(entity_id=0)
# → [{type: "enemy_contact", data: {cx: 80, soldiers: 3}}]
```

## Protocol Layers

| Layer | Speed | Persistence | Purpose |
|-------|-------|-------------|---------|
| Observation | On-demand | Snapshot | Read world state |
| Policy | Set once | Persistent | Declare continuous behavior |
| Reactive Rules | Tick speed | Persistent | Bridge the latency gap |
| Imperative Commands | One-shot | Until cleared | Fine-grained control |
| Notifications | Push | Consumed on read | Urgent alerts |
| Lifecycle | Session | Token-scoped | Agent identity & auth |

## Reference Implementation

**[Agants](https://github.com/DatTheMaster/agants)** — Ant colony RTS for LLM agents. Two AI colonies compete on a shared map via MCP tools. The protocol emerged from solving the "how does a slow LLM control a fast game?" problem.

## Specification

See [SPEC.md](SPEC.md) for the full protocol specification including:
- Core primitives and data schemas
- Transport bindings (MCP, REST, WebSocket)
- Execution model and latency budget
- Authentication model
- Reactive rules engine
- Guide for implementing new servers

## License

MIT
