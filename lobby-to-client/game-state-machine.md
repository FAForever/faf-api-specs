# FA Game Instance State Machine

This diagram documents the **game instance lifecycle** from the **server's perspective**, showing how GPGNet commands drive state transitions inside a `Game` object. States correspond to the `GameState` and `GameConnectionState` enums tracked server-side.

For the player/client lifecycle (PlayerState transitions driven by lobby messages), see [state-machine.md](state-machine.md).
For the full message schema, see [v1/asyncapi.yml](v1/asyncapi.yml).

## Game State Machine

```mermaid
stateDiagram-v2
    [*] --> INITIALIZING : GameService.create_game()

    state "INITIALIZING (GameState.INITIALIZING)" as INITIALIZING
    INITIALIZING : Game created, host assigned
    INITIALIZING : setup_timeout ticking (30s custom / 60s ladder)

    INITIALIZING --> LOBBY : Host → GameState("Idle")
    INITIALIZING --> ENDED : setup_timeout expires

    state "LOBBY (GameState.LOBBY)" as LOBBY {
        state "Host Listening" as HostListening
        HostListening : ← HostGame(map_folder_name) sent to host
        HostListening : game.set_hosted() called

        state "Configuring" as Configuring
        Configuring : → GameOption(key, value)
        Configuring : → PlayerOption(id, key, value)
        Configuring : → AIOption(name, key, value)
        Configuring : → ClearSlot(slot)
        Configuring : → GameMods(mode, uids)

        state "Joiners Connecting" as JoinersConnecting
        JoinersConnecting : ← JoinGame(host_name, host_id) to joiner
        JoinersConnecting : ← ConnectToPeer(name, id, offer) to peers
        JoinersConnecting : → IceMsg relayed between peers

        HostListening --> Configuring
        Configuring --> Configuring : host changes options
        Configuring --> JoinersConnecting : joiner → GameState("Lobby")
        JoinersConnecting --> Configuring : joiner connected
    }

    LOBBY --> LIVE : Host → GameState("Launching")\ngame.launch()
    LOBBY --> ENDED : Host disconnects /\nall connections lost

    state "LIVE (GameState.LIVE)" as LIVE {
        state "Simulation Running" as SimRunning
        SimRunning : _players_at_launch frozen
        SimRunning : game_stats + game_player_stats written to DB
        SimRunning : validity checks applied

        SimRunning --> SimRunning : → GameResult(army, result)\n→ JsonStats(stats)\n→ Desync\n→ TeamkillHappened(...)\n→ OperationComplete(...)

        state "Simulation Finished" as SimFinished
        SimFinished : All connections finished_sim = True
        SimFinished : check_sim_end() then endTime written to DB

        SimRunning --> SimFinished : All players → GameEnded /\nall connections closed
    }

    LIVE --> ENDED : on_game_finish()\nprocess results → rating update

    state "ENDED (GameState.ENDED)" as ENDED
    ENDED : Results persisted, game marked dirty
    ENDED --> [*] : GameService.remove_game()
```

## GameConnectionState Machine

Each player's connection to a game is tracked independently. The host and joiners follow different paths through these states.

```mermaid
stateDiagram-v2
    [*] --> GC_INITIALIZING : GameConnection created\n(in _prepare_launch_game)

    state "INITIALIZING\n(GameConnectionState.INITIALIZING)" as GC_INITIALIZING
    GC_INITIALIZING : setup_timeout ticking (default 60s)

    state "INITIALIZED\n(GameConnectionState.INITIALIZED)" as GC_INITIALIZED
    GC_INITIALIZED : Joiner sent GameState("Idle")
    GC_INITIALIZED : player.state = JOINING

    state "CONNECTED_TO_HOST\n(GameConnectionState.CONNECTED_TO_HOST)" as GC_CONNECTED

    state "ENDED\n(GameConnectionState.ENDED)" as GC_ENDED

    GC_INITIALIZING --> GC_CONNECTED : Host → GameState("Idle")\n[game.state = LOBBY, player.state = HOSTING]
    GC_INITIALIZING --> GC_INITIALIZED : Joiner → GameState("Idle")\n[player.state = JOINING]
    GC_INITIALIZING --> GC_ENDED : setup_timeout expires, abort()

    GC_INITIALIZED --> GC_CONNECTED : Joiner → GameState("Lobby")\n[connect_to_host, peer mesh setup]
    GC_INITIALIZED --> GC_ENDED : abort() (host left, error)

    GC_CONNECTED --> GC_ENDED : → GameState("Ended") /\nconnection closed / abort()

    GC_ENDED --> [*] : player.state = IDLE
```

## Lobby Phase — GPGNet Handshake

This sequence shows the full GPGNet command flow during lobby setup for a host with two joiners. The `offer` parameter on `ConnectToPeer` determines ICE initiator (`true`) vs responder (`false`).

```mermaid
sequenceDiagram
    participant Host as Host (FA)
    participant Server as Server
    participant J1 as Joiner 1 (FA)
    participant J2 as Joiner 2 (FA)

    Note over Host,Server: Host Setup
    Host->>Server: GameState("Idle")
    Note over Server: game.state = LOBBY<br/>conn.state = CONNECTED_TO_HOST<br/>player.state = HOSTING
    Host->>Server: GameState("Lobby")
    Server->>Host: HostGame(map_folder_name)
    Note over Server: game.set_hosted()

    Note over J1,Server: Joiner 1 Connects
    J1->>Server: GameState("Idle")
    Note over Server: conn.state = INITIALIZED<br/>player.state = JOINING
    J1->>Server: GameState("Lobby")
    Server->>J1: JoinGame(host_name, host_id)
    Server->>Host: ConnectToPeer(j1_name, j1_id, offer=true)
    Note over Server: j1 conn.state = CONNECTED_TO_HOST

    Note over J2,Server: Joiner 2 Connects
    J2->>Server: GameState("Idle")
    Note over Server: conn.state = INITIALIZED<br/>player.state = JOINING
    J2->>Server: GameState("Lobby")
    Server->>J2: JoinGame(host_name, host_id)
    Server->>Host: ConnectToPeer(j2_name, j2_id, offer=true)
    Note over Server: Peer mesh: J1 ↔ J2
    Server->>J2: ConnectToPeer(j1_name, j1_id, offer=true)
    Server->>J1: ConnectToPeer(j2_name, j2_id, offer=false)
    Note over Server: j2 conn.state = CONNECTED_TO_HOST

    Note over Host,J2: Game Launch
    Host->>Server: GameState("Launching")
    Note over Server: game.launch()<br/>state = LIVE<br/>freeze _players_at_launch<br/>all players → PLAYING<br/>write game_stats to DB
```

## ICE / Peer-to-Peer Connection Flow

All game traffic flows peer-to-peer over UDP. The server only relays ICE signaling messages to establish the direct connection.

```mermaid
sequenceDiagram
    participant A as Player A (FA)
    participant S as Server
    participant B as Player B (FA)

    S->>A: ConnectToPeer(B_name, B_id, offer=true)
    S->>B: ConnectToPeer(A_name, A_id, offer=false)

    Note over A,B: ICE candidate exchange via server relay
    A->>S: IceMsg(B_id, ice_candidate)
    S->>B: IceMsg(A_id, ice_candidate)
    B->>S: IceMsg(A_id, ice_candidate)
    S->>A: IceMsg(B_id, ice_candidate)

    Note over A,B: Direct UDP connection established
    A-->>B: Game simulation data (peer-to-peer)
```

## Game End Flow

```mermaid
sequenceDiagram
    participant FA as FA Client(s)
    participant GC as GameConnection
    participant Game as Game
    participant DB as Database
    participant MQ as Message Queue

    FA->>GC: GameResult(army, "victory 1")
    GC->>Game: add_result(reporter, army, outcome, score)

    FA->>GC: JsonStats(stats_json)
    GC->>Game: report_army_stats(stats)

    FA->>GC: GameEnded
    GC->>GC: finished_sim = True
    GC->>Game: check_game_finish(player)

    Game->>Game: check_sim_end()
    Note over Game: All finished_sim?<br/>→ game.finished = True
    Game->>DB: UPDATE game_stats SET endTime

    Game->>Game: on_game_finish()
    alt desyncs > 20
        Game->>Game: mark_invalid(TOO_MANY_DESYNCS)
    else normal finish
        Game->>Game: process_game_results()
        Game->>DB: persist_results() — update scores
        Game->>Game: resolve_game_results()
        Game->>MQ: publish_game_results → rating service
    end
    Note over Game: state = ENDED
```

## GPGNet Command Reference

### FA → Server (game client sends)

| Command | Parameters | Phase | Description |
|---|---|---|---|
| `GameState` | `state` ("Idle", "Lobby", "Launching", "Ended") | All | Drives game and connection state transitions |
| `GameOption` | `key, value` | LOBBY | Host sets game option (Victory, Cheats, Speed, etc.) |
| `PlayerOption` | `player_id, key, value` | LOBBY | Host sets player option (Faction, Team, Color, StartSpot, Army) |
| `AIOption` | `ai_name, key, value` | LOBBY | Host configures AI player |
| `ClearSlot` | `slot` | LOBBY | Host removes player/AI from slot |
| `GameMods` | `mode, args` | LOBBY | Report activated mods ("activated" count or "uids" list) |
| `EnforceRating` | _(none)_ | LOBBY | Host requests rating enforcement |
| `GameFull` | _(none)_ | LOBBY | All slots filled (informational) |
| `Chat` | `message` | LOBBY | In-lobby chat message |
| `LaunchStatus` | `status` | LOBBY | "Rejected" if matchmaker launch failed |
| `GameResult` | `army, result_string` | LIVE | Army outcome (e.g. "victory 1", "defeat 0") |
| `GameEnded` | _(none)_ | LIVE | Simulation has ended for this player |
| `JsonStats` | `stats_json` | LIVE | Army statistics JSON blob |
| `OperationComplete` | `primary, secondary, time_delta` | LIVE | Coop mission completion |
| `Desync` | _(none)_ | LIVE | Desync event (>20 → game invalid) |
| `TeamkillHappened` | `gametime, victim_id, victim_name, tk_id, tk_name` | LIVE | Automatic teamkill report |
| `TeamkillReport` | `gametime, reporter_id, reporter_name, tk_id, tk_name` | LIVE | Player-initiated teamkill report |
| `IceMsg` | `receiver_id, ice_msg_json` | LOBBY/LIVE | ICE candidate relay for P2P setup |
| `Bottleneck` | `code, ...args` | LIVE | Player data processing bottleneck |
| `BottleneckCleared` | _(none)_ | LIVE | Bottleneck resolved |
| `Disconnected` | `...args` | LIVE | Peer disconnected (logged only) |
| `Rehost` | `...args` | LOBBY | Game rehosted (unused) |

### Server → FA (server sends to game client)

| Command | Parameters | When Sent | Description |
|---|---|---|---|
| `HostGame` | `map_path` | Host sends GameState("Lobby") | Tell host FA to start listening for peer connections |
| `JoinGame` | `player_name, player_uid` | Joiner sends GameState("Lobby") | Tell joiner FA to connect to host |
| `ConnectToPeer` | `player_name, player_uid, offer` | Joiner enters lobby | Establish peer mesh; `offer=true` = ICE initiator |
| `DisconnectFromPeer` | `player_id` | Peer leaves during LOBBY | Tell FA to disconnect from a specific peer |
| `IceMsg` | `sender_id, ice_msg_json` | During ICE negotiation | Relayed ICE candidate from another peer |

## Game Type Variants

| Property | Custom | Coop | Matchmaker |
|---|---|---|---|
| **Class** | `CustomGame` | `CoopGame` | `LadderGame` |
| **InitMode** | `NORMAL_LOBBY` | `NORMAL_LOBBY` | `AUTO_LOBBY` |
| **GameType** | `custom` | `coop` | `matchmaker` |
| **Rating** | `global` | not ranked | from queue config |
| **Setup timeout** | 30s | 30s | 60s |
| **Lobby config** | Host configures | Host configures | Server pre-sets (faction, team, slot) |
| **Launch** | Host clicks launch | Host clicks launch | Server waits for `wait_hosted` + `wait_launched` |
| **Special** | — | `OperationComplete` for coop leaderboard | `LaunchStatus("Rejected")` on settings mismatch |

## State Reference

| GameState | Value | Description | Entry Trigger |
|---|---|---|---|
| `INITIALIZING` | 0 | Game created, awaiting host's first GPGNet message | `GameService.create_game()` |
| `LOBBY` | 1 | Host listening, players joining and configuring | Host sends `GameState("Idle")` |
| `LIVE` | 2 | Simulation running, results being collected | Host sends `GameState("Launching")` → `game.launch()` |
| `ENDED` | 3 | Game finished, results processed | `on_game_finish()` |

| GameConnectionState | Value | Description | Entry Trigger |
|---|---|---|---|
| `INITIALIZING` | 0 | GameConnection created, FA process starting | `_prepare_launch_game()` |
| `INITIALIZED` | 1 | Joiner sent `GameState("Idle")`, not yet connected to host | Joiner: `_handle_idle_state()` |
| `CONNECTED_TO_HOST` | 2 | Peer setup complete, in the game | Host: `_handle_idle_state()` / Joiner: `_handle_lobby_state()` |
| `ENDED` | 3 | Connection terminated, player returned to IDLE | `abort()` / `on_connection_closed()` |

## Message Legend

| Arrow | Meaning |
|---|---|
| `→ message` | Game client (FA) sends GPGNet command to server |
| `← message` | Server sends GPGNet command to game client (FA) |
| `→ lobby_command` | Client sends lobby protocol message to server |
| `← lobby_message` | Server sends lobby protocol message to client |
