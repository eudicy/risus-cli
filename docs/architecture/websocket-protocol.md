# WebSocket Message Protocol

Maps every WebSocket message type in both directions. The left subgraph lists the 7 client-to-server commands (add player, edit cliche, reduce dice, lock, unlock, save, load) with their required fields and lock constraints. The right subgraph lists the 6 server-to-client frames (state, presence, lock_acquired, lock_released, lock_denied, error) annotated with whether they broadcast to all clients or go to the caller only. Edges show which command produces which response frame.

---

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
