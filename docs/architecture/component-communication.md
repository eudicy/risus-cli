# Component Communication Architecture

Risus CLI — multiplayer battle tracker. Pure CLI client + FastAPI server over WebSocket and REST.

This document is an index. Each diagram lives in its own file.

---

## Diagrams

| Diagram | Description |
|---------|-------------|
| [System Overview](system-overview.md) | All client and server components with their internal wiring, cross-boundary WebSocket and REST connections, and the three PostgreSQL tables. |
| [WebSocket Message Protocol](websocket-protocol.md) | All 7 client→server command types and 6 server→client frame types, annotated with lock requirements, broadcast scope, and which commands produce which responses. |
| [Key Interaction Flows](interaction-flows.md) | Four sequence diagrams: the lock/edit/unlock happy path, concurrent edit rejection, automatic lock release on client disconnect, and battle save/load. |
| [REST Endpoints](rest-endpoints.md) | The three HTTP endpoints (`/healthz`, `/state`, `/saves`), the CLI code paths that call them, and their PostgreSQL backing tables. |
| [Threading Model](threading-model.md) | How the CLI splits work across a synchronous main thread and a background asyncio thread, connected via two queues and a thread-safe state mirror with an update event. |
