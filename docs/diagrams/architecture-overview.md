# Network & API Analysis

> Research conducted through browser DevTools (Network tab, JS debugger) and analysis of client-side JavaScript bundles.

---

## API Architecture

Travian uses a **hybrid API approach**:

- **REST API** — primary interface for most game actions (older endpoints)
- **GraphQL** — increasingly adopted in newer versions; the codebase shows a clear migration trend toward GraphQL

This suggests an incremental modernization strategy — new features are being built on GraphQL while legacy REST endpoints remain in place.

---

## Response Structure

All API responses follow a consistent envelope pattern. There are three distinct response types:

### Success response — HTTP 200
Returned when an action completes successfully and data is available:
```json
{
  "data": { ... }
}
```

### Error response
Returned when an action fails due to validation or game logic:
```json
{
  "error": "error.badName",
  "errorId": "<random 16-char string>",
  "message": "Human-readable error description"
}
```

- `error` — machine-readable error key (dot-notation namespaced)
- `errorId` — unique identifier for each error occurrence (useful for server-side log correlation)
- `message` — localized, human-readable description via the game's translation system

### Redirect response
Returned when the client should navigate to a different page:
```json
{
  "redirectTo": "dorf1.php"
}
```

The redirect targets are legacy PHP page names — another indicator of the gradual migration from a page-based architecture toward a modern API-driven frontend.

---

## Real-time Layer

### Travian: Legends (v4 and earlier)
Uses **HTTP polling** — the client periodically requests updates from the server at fixed intervals. This is observable through repeated identical requests in DevTools at regular time intervals.

### Travian: Kingdoms / v5+
Uses **WebSocket** — a persistent bidirectional connection is established on page load. Real-time game events (incoming attacks, building completions, market arrivals) are pushed from the server to the client without polling.

This is a significant architectural upgrade — WebSocket eliminates the latency and server load of constant polling, and enables truly real-time notifications.

---

## Client-side Architecture

The frontend JavaScript is delivered as a **compiled and bundled build** (Webpack or similar). Key observations from bundle analysis:

- Code is minified and chunked — feature-based code splitting is used
- Asset URLs follow versioned patterns, indicating a CI/CD pipeline with cache-busting
- The bundle structure reveals separation between game UI logic and API communication layers
- GraphQL queries are embedded in the compiled bundle, which allows inference of the data schema without server access

---

## Authentication & Session

- Session management is handled via standard HTTP cookies
- Requests include session tokens validated server-side on every API call
- No observable use of JWT in legacy endpoints; newer GraphQL endpoints may differ

---

## Summary

| Layer | Technology | Notes |
|---|---|---|
| Primary API | REST | Legacy, still dominant |
| Modern API | GraphQL | Actively expanding |
| Real-time (v4) | HTTP Polling | Periodic client requests |
| Real-time (v5+) | WebSocket | Push-based, persistent connection |
| Frontend | Compiled JS bundle | Minified, chunked, versioned |
| Error format | JSON envelope | Consistent across all endpoints |
| Session | HTTP Cookies | Server-side session validation |
