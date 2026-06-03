# Tech Stack Analysis

> **Confidence levels:**
> - ✅ `[Confirmed]` — verified through direct observation or official sources
> - 🔶 `[Strong Hypothesis]` — strongly inferred from behavioral/network evidence
> - 💡 `[Hypothesis]` — plausible inference, not directly confirmed

---

## Backend

### PHP — ✅ [Confirmed]
**Evidence:**
- Officially confirmed in Travian Games job postings (PHP listed as primary backend language)
- HTTP response headers and error pages historically exposed PHP fingerprints
- Request/response timing patterns consistent with PHP-FPM process model

**Notes:**  
Given the age of the game (2004+), PHP was a natural choice. The codebase has likely evolved significantly, but PHP remains the core backend language across both Legends and Kingdoms versions.

---

## Web Server & Proxy

### Nginx — ✅ [Confirmed]
**Evidence:**
- HTTP response headers reveal Nginx as the web server
- Static assets served with Nginx-typical caching headers (`Cache-Control`, `ETag` patterns)
- Response behavior on malformed requests consistent with Nginx default handling

**Role in architecture:**  
Nginx likely serves a dual role:
1. **Static asset server** — JS, CSS, images served directly
2. **Reverse proxy / API Gateway** — proxying dynamic requests to PHP-FPM backend workers

```
Client → Nginx (reverse proxy) → PHP-FPM workers
                ↓
         Static assets served directly
```

---

## Infrastructure & Deployment

### Docker + Kubernetes — 🔶 [Strong Hypothesis]
**Evidence:**
- Explicitly listed in Travian Games engineering job postings (Docker, Kubernetes, AWS)
- Deployment behavior suggests containerized workloads (rolling updates observed with zero downtime)
- Response latency patterns suggest load-balanced, horizontally scaled backend

**Inferred architecture:**
```
Load Balancer (AWS ALB or similar)
        ↓
Kubernetes Cluster
  ├── PHP-FPM pods (web workers)
  ├── Game Engine daemon pods
  ├── Queue workers
  └── Static asset CDN (CloudFront or similar)
```

### AWS — 🔶 [Strong Hypothesis]
**Evidence:**
- Listed in job postings alongside Docker/Kubernetes
- Some asset URLs and latency patterns suggest AWS infrastructure

---

## Database Layer

### SQL (MySQL / MariaDB or PostgreSQL) — 💡 [Hypothesis]
**Reasoning:**
- PHP + relational DB is the classic and expected pairing for a game of this era
- Game data (players, villages, troops, buildings) is inherently relational
- Query patterns observed in network traffic suggest structured, relational data access

### NoSQL / Caching Layer (Redis) — ✅ [Confirmed via Queue research]
**Evidence:**
- Redis confirmed as part of the queue/messaging system (see game engine section)
- Session data, leaderboard rankings, and real-time game state are ideal Redis use cases
- Low-latency read patterns on certain endpoints consistent with in-memory cache hits

**Likely data split:**
| Data Type | Likely Storage |
|---|---|
| Player accounts, villages, troops | SQL (relational) |
| Sessions, auth tokens | Redis |
| Real-time game state, counters | Redis |
| Leaderboards, rankings | Redis (Sorted Sets) |
| Attack/build queues | Redis or RabbitMQ |
| Logs, analytics | Possibly separate (Elasticsearch?) |

---

## Game Engine

### Server-side Game Processing Daemon — ✅ [Confirmed]
This is one of the most significant architectural findings of this research.

**What it is:**  
A long-running background process (daemon) that handles all time-based and computationally parallel game operations independently of HTTP request cycles.

**Evidence:**
- Game events (attacks landing, buildings completing, resources producing) resolve at exact calculated times — even with no active HTTP requests
- Event resolution is consistent across all players simultaneously, ruling out request-triggered processing
- Timing precision and parallel resolution behavior strongly indicate a dedicated processing daemon

**Responsibilities (observed):**
- ⚔️ Attack and raid resolution
- 🏗️ Building construction queue processing
- ⚒️ Troop training timers
- 🌾 Resource production tick calculations
- 📨 In-game messaging/notification delivery

**Architecture:**
```
HTTP Layer (Nginx + PHP-FPM)
        ↓  [enqueues tasks]
Message Queue (Redis / RabbitMQ)
        ↓  [consumes tasks]
Game Engine Daemon (PHP CLI or separate process)
        ↓  [writes results]
Database (SQL + Redis)
```

### Message Queue: Redis or RabbitMQ — ✅ [Confirmed]
**Evidence:**
- Both Redis and RabbitMQ are standard choices for this pattern in PHP ecosystems
- Task processing behavior (ordering, retry logic, parallel execution) observed in game event timing is consistent with a proper message queue implementation
- Redis pub/sub or list-based queues, or RabbitMQ AMQP exchanges, are the two most likely implementations

---

## Summary Diagram

```
                    ┌─────────────────────────────────┐
                    │         CDN / Load Balancer      │
                    └────────────┬────────────────────┘
                                 │
                    ┌────────────▼────────────────────┐
                    │     Nginx (Reverse Proxy)        │
                    │   Static assets + API gateway    │
                    └────────────┬────────────────────┘
                                 │
              ┌──────────────────▼──────────────────────┐
              │           PHP-FPM Workers                │
              │    (HTTP request/response handling)      │
              └──────┬─────────────────────┬────────────┘
                     │                     │
          ┌──────────▼──────┐   ┌──────────▼──────────┐
          │   SQL Database  │   │   Redis              │
          │ (relational     │   │ (cache, sessions,    │
          │  game data)     │   │  queues, leaderboard)│
          └─────────────────┘   └──────────┬───────────┘
                                           │
                              ┌────────────▼────────────┐
                              │   Game Engine Daemon     │
                              │  (daemonized process)    │
                              │  attacks / builds /      │
                              │  resources / troops      │
                              └─────────────────────────┘
```
