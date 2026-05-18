# REST Endpoints

Shows the three HTTP endpoints exposed by `server/rest.py` and the CLI code paths that call them. The client calls `/healthz` for liveness checks, `/state` to load the current player list on startup, and `/saves` to populate the load-battle menu. Each endpoint is backed directly by a PostgreSQL table — `players` for healthz and state, `saves` for the saves listing.

---

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
