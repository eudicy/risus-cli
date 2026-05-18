# Key Interaction Flows

Four sequence diagrams covering the most important runtime scenarios. The first shows the happy path: a client acquires a lock, edits a player, then releases the lock, with broadcasts propagating to other clients at each step. The second shows concurrent conflict: a second client attempts an edit on an already-locked player and receives a private error. The third shows automatic cleanup: when a client disconnects the server releases all its held locks and broadcasts presence/lock updates. The fourth shows persistence: a client saves the current battle state to a named snapshot or restores one, replacing all live player data.

---

## Lock → Edit → Unlock

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

## Concurrent Edit — Lock Denied

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

## Client Disconnect — Auto Release

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

## Save / Load

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
