# Threading Model

Shows how the CLI client splits work across two threads. The main thread runs a synchronous `select.select()` poll loop that handles user input and terminal rendering. A background thread runs an asyncio event loop managing the WebSocket connection with automatic exponential-backoff reconnect. The two threads communicate through two thread-safe queues (outbox: main→ws, inbox: ws→main) and a `ClientState` object protected by a `threading.Lock`. The `update_event` flag wakes the main loop whenever the server pushes new state.

---

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
