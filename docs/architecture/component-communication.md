# Component Communication Architecture

Risus CLI — multiplayer battle tracker. Pure CLI client + FastAPI server over WebSocket and REST.

---

## System Overview

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

---

## WebSocket Message Protocol

```mermaid
graph LR
    subgraph C2S["Client → Server"]
        direction TB
        c1["add_player<br/>name · cliche · dice"]
        c2["switch_cliche<br/>player_name · cliche · dice<br/>⚠ lock required"]
        c3["reduce_dice<br/>player_name · amount · is_dead<br/>⚠ lock required"]
        c4["lock<br/>player_name"]
        c5["unlock<br/>player_name"]
        c6["save<br/>save_name"]
        c7["load<br/>save_name"]
    end

    subgraph S2C["Server → Client"]
        direction TB
        s1["state<br/>players[]<br/>→ broadcast all<br/>→ triggers UI refresh"]
        s2["presence<br/>clients[]<br/>→ broadcast all<br/>→ triggers UI refresh"]
        s3["lock_acquired<br/>player_name · locked_by<br/>→ broadcast all<br/>→ triggers UI refresh"]
        s4["lock_released<br/>player_name<br/>→ broadcast all<br/>→ triggers UI refresh"]
        s5["lock_denied<br/>player_name · locked_by<br/>→ caller only"]
        s6["error<br/>message<br/>→ caller only"]
    end

    c1 -->|"DB insert →"| s1
    c2 -->|"lock check / DB update →"| s1
    c3 -->|"lock check / DB update/delete →"| s1
    c4 -->|"acquire"| s3
    c4 -->|"already held"| s5
    c5 -->|"release"| s4
    c6 -->|"DB save (no broadcast)"| s6
    c7 -->|"release all locks / DB replace →"| s1
```

---

## Key Interaction Flows

### Lock → Edit → Unlock

```mermaid
sequenceDiagram
    actor A as Client A
    participant S as Server (ws.py + commands.py)
    participant L as LockManager
    participant D as Database
    actor B as Client B

    A->>S: lock {player_name}
    S->>L: acquire(player_name, A)
    L-->>S: granted
    S-->>A: lock_acquired (caller)
    S-->>B: lock_acquired (broadcast)

    A->>S: switch_cliche {player_name, cliche, dice}
    S->>L: check holder == A
    L-->>S: ok
    S->>D: UPDATE players
    S-->>A: state (broadcast)
    S-->>B: state (broadcast)

    A->>S: unlock {player_name}
    S->>L: release(player_name)
    S-->>A: lock_released (broadcast)
    S-->>B: lock_released (broadcast)
```

### Concurrent Edit — Lock Denied

```mermaid
sequenceDiagram
    actor A as Client A (holds lock)
    participant S as Server
    participant L as LockManager
    actor B as Client B

    Note over A,L: A already holds lock on "Aragorn"

    B->>S: switch_cliche {player_name: "Aragorn", ...}
    S->>L: check holder
    L-->>S: locked by A
    S-->>B: error "lock required" (caller only)
```

### Client Disconnect — Auto Release

```mermaid
sequenceDiagram
    participant S as Server (ws.py)
    participant L as LockManager
    participant D as Database
    actor X as Client (disconnects)
    actor O as Other Clients

    X--xS: WebSocket closed
    S->>L: release_all(client_name)
    loop for each released lock
        S-->>O: lock_released (broadcast)
    end
    S->>D: update presence
    S-->>O: presence (broadcast)
```

### Save / Load

```mermaid
sequenceDiagram
    actor C as Client
    participant S as Server
    participant D as Database

    C->>S: save {save_name}
    S->>D: INSERT/UPSERT saves (JSONB snapshot)
    D-->>S: ok
    S-->>C: (no broadcast — silent)

    C->>S: load {save_name}
    S->>D: SELECT saves WHERE save_name
    D-->>S: snapshot data
    S->>D: DELETE players, INSERT from snapshot
    S-->>C: state (broadcast all)
```

---

## REST Endpoints

```mermaid
graph LR
    subgraph CLIENT
        UI2["risus.py<br/>Load menu<br/>Health check"]
    end

    subgraph REST_LAYER["server/rest.py"]
        R1["GET /healthz<br/>→ {ok: true}"]
        R2["GET /state<br/>→ {type: state, players: [...]}"]
        R3["GET /saves<br/>→ [{save_name, saved_at}]"]
    end

    subgraph DB2["PostgreSQL"]
        P["players"]
        SV["saves"]
    end

    UI2 -->|"liveness check"| R1
    UI2 -->|"initial state on connect"| R2
    UI2 -->|"list saves in menu"| R3

    R1 -->|"SELECT 1"| P
    R2 --> P
    R3 --> SV
```

---

## Threading Model

```mermaid
graph TB
    subgraph MAIN["Main Thread (sync)"]
        LOOP["select.select() poll loop"]
        RENDER["Terminal render<br/>print() / clear()"]
        INPUT["input() handlers"]
        SNAP["state.snapshot_*()"]
    end

    subgraph BG["Background Thread (asyncio)"]
        ALOOP["asyncio event loop"]
        WSC["WebSocket connection<br/>auto-reconnect<br/>1→2→4→8s backoff"]
        RECV["recv task"]
        SEND["send task"]
    end

    subgraph SYNC["Shared (thread-safe)"]
        OUTBOX["outbox Queue<br/>(main→ws)"]
        INBOX["inbox Queue<br/>(ws→main)"]
        CSTT["ClientState<br/>(threading.Lock)<br/>update_event"]
    end

    INPUT -->|"put"| OUTBOX
    OUTBOX -->|"get"| SEND
    SEND -->|"ws.send_str"| WSC
    WSC -->|"ws.receive_str"| RECV
    RECV -->|"put"| INBOX
    INBOX -->|"get / apply frame"| CSTT
    CSTT -->|"update_event.set()"| LOOP
    LOOP -->|"update_event triggered"| SNAP
    SNAP --> RENDER
```
