# Game Engine Architecture

> Research based on observable behavior of Travian's server-side processing system.

---

## Overview

The game engine runs as a **background processing service** completely separate from the web layer. This design allows the game to process thousands of simultaneous time-based events — attacks, construction, troop training, resource production — independently of player activity or web requests.

The engine is organized into specialized **processing groups**, each responsible for a distinct domain of game logic, running concurrently as isolated workers.

---

## Processing Groups

### Construction & Market Worker
Handles all building and economy events:

| Worker | Responsibility |
|---|---|
| Trade routes | Process recurring trade route deliveries |
| Research | Complete smithy research upgrades |
| Market | Resolve merchant arrivals and deliveries |
| Auction | Process hero auction bids |
| Auction simulation | Generate NPC auction activity |
| Building | Resolve completed building constructions |
| Auto-raid | Execute scheduled farmlist raids (optional feature) |

---

### Movement Worker
Highest-priority worker — resolves events the moment they arrive:

| Worker | Responsibility |
|---|---|
| Combat resolver | Resolve battles when attacks reach their destination |
| Movement resolver | Handle reinforcements, returns, trade convoys, settlers |

---

### Training Worker
Resolves completed troop training queues across all villages.

---

### Game State Worker
Manages long-running game progression events:

| Worker | Responsibility |
|---|---|
| Win condition checker | Evaluate end-game victory conditions |
| Server initializer | Bootstrap a new game round |
| Gold processor | Confirm and apply gold purchases |
| Ban processor | Apply player bans and restrictions |
| Artifact releaser | Release Natars artifacts at scheduled times |
| Medal manager | Reset and award player medals |
| Daily quest manager | Refresh daily quest pool |
| Server extender | Handle server end-date extensions |
| Artifact effect updater | Apply Fool artifact effects to targets |
| Artifact activation | Activate newly captured artifacts |
| Server cleanup | Purge stale and orphaned data |
| Oasis manager | Finalize oasis ownership changes |
| Alliance bonus manager | Process alliance bonus upgrades |
| Ghost village handler | Handle villages that reach zero population |

---

### Maintenance Worker
Background housekeeping tasks:

| Worker | Responsibility |
|---|---|
| Adventure generator | Spawn new hero adventure events |
| Inactive player cleanup | Process and remove inactive accounts |
| Game end checker | Verify round termination conditions |
| Ranking updater | Rebuild player and alliance ranking tables |
| Index optimizer | Maintain database index health |
| Country flag updater | Refresh player geolocation flags |
| Daily gold resetter | Reset daily gold allowances |
| Backup | Trigger automated database backups |

---

### AI Worker
Manages all NPC and bot-controlled entities:

| Worker | Responsibility |
|---|---|
| NPC player handler | Execute actions for AI-controlled players |
| NPC expansion handler | Manage AI village founding and growth |
| Natar village handler | Control Natar NPC village behavior |
| Natar expansion handler | Execute Natar conquest of unoccupied territories |

---

### End-of-Round Worker
Runs exclusively during the post-game phase after a round concludes — handles final scoring, rewards, and cleanup before the server resets.

---

## Architecture

```
Game Engine (background service)
│
├── Construction & Market Worker
│   ├── Trade routes
│   ├── Research
│   ├── Market
│   ├── Auction
│   └── Building
│
├── Movement Worker
│   ├── Combat resolver       ← highest priority
│   └── Movement resolver
│
├── Training Worker
│
├── Game State Worker
│   └── 14 specialized sub-workers
│
├── Maintenance Worker
│   └── 8 specialized sub-workers
│
├── AI Worker
│   └── NPC players, Natars
│
└── End-of-Round Worker
```

---

## Key Design Principles

**Parallel, isolated workers**
Each processing group runs independently — a failure or slowdown in one domain does not affect others.

**Database-driven control**
The engine's behavior is governed by runtime flags stored in the database, allowing operators to pause, restart, or terminate processing without touching the server process directly.

**Graceful shutdown**
On shutdown signal, the master process notifies all workers to finish their current task and exit cleanly — no events are left mid-resolution.

**Fault tolerance**
Every worker is wrapped in error handling — exceptions are logged and the worker pauses briefly before resuming, preventing failure loops from overloading the database.

**Memory isolation**
Each worker has its own dedicated memory space, preventing one subsystem's memory usage from impacting the stability of others.
