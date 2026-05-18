# System Overview

Shows all major components of the Risus CLI system and how they connect. The CLI client subgraph contains the main loop, WebSocket client, state mirror, and config reader. The FastAPI server subgraph groups connection management, command dispatch, lock management, REST endpoints, DB helpers, and Pydantic models. PostgreSQL sits at the bottom holding three tables. Arrows show data flow: WebSocket frames between client and server, REST calls for initial state and saves listing, and SQL queries to the DB.

---

```mermaid
graph TB
    subgraph CLIENT["CLI Client (risus.py + client/)"]
        direction TB
        UI["risus.py<br/>Main Loop / UI Renderer<br/>(select.select polling)"]
        WS_C["ws_client.py<br/>Async WebSocket Client<br/>(background thread)"]
        STATE["state.py<br/>ClientState<br/>(thread-safe mirror)"]
        CFG["config.py<br/>Config Persistence<br/>(risus.cfg)"]

        UI -->|"enqueue outbox"| WS_C
        WS_C -->|"enqueue inbox"| UI
        WS_C -->|"apply frame / set update_event"| STATE
        STATE -->|"snapshot_*()"| UI
        CFG -->|"server / name / token"| WS_C
    end

    subgraph SERVER["FastAPI Server (server/)"]
        direction TB
        APP["app.py<br/>Lifespan / DI<br/>(DB pool, managers)"]
        WS_S["ws.py<br/>ConnectionManager<br/>(presence, broadcast)"]
        CMD["commands.py<br/>Message Handlers<br/>(8 command types)"]
        LOCKS["locks.py<br/>LockManager<br/>(in-memory, session)"]
        REST["rest.py<br/>REST Endpoints"]
        DB["db.py<br/>DB Helpers<br/>(asyncpg)"]
        MODELS["models.py<br/>Pydantic Models"]

        APP -->|"injects"| WS_S
        APP -->|"injects"| LOCKS
        APP -->|"injects"| DB
        WS_S -->|"dispatch"| CMD
        CMD -->|"acquire/release"| LOCKS
        CMD -->|"CRUD / snapshots"| DB
        CMD -->|"broadcast"| WS_S
        REST -->|"read"| DB
        MODELS -.->|"schema"| CMD
        MODELS -.->|"schema"| WS_S
    end

    subgraph STORAGE["PostgreSQL"]
        TBL_P["players<br/>(name PK, cliche, dice, lost_dice)"]
        TBL_S["saves<br/>(save_name PK, saved_at, data JSONB)"]
        TBL_L["locks<br/>(player_name PK, locked_by, acquired_at)<br/>audit log — truncated on startup"]
    end

    WS_C -->|"ws[s]://host/ws/{name}?token=…"| WS_S
    UI -->|"GET /state · GET /saves · GET /healthz"| REST
    DB -->|"queries"| STORAGE
```
