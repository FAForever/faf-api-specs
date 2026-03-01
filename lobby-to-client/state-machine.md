# Client Lifecycle State Machine

This diagram documents the player/client lifecycle from the **server's perspective**, showing how lobby messages drive state transitions. States correspond to the `PlayerState` enum tracked server-side, extended with connection phases.

For the full message schema, see [v1/asyncapi.yml](v1/asyncapi.yml).

## State Machine

```mermaid
stateDiagram-v2
    [*] --> Connecting : WebSocket opened

    state "Connection & Auth" as auth {
        Connecting --> Authenticating : ← session (server)
        Authenticating --> Idle : ← welcome (server)
        Authenticating --> [*] : ← authentication_failed
    }

    state "Idle (PlayerState.IDLE)" as Idle

    Idle --> Idle : ← player_info, game_info,\nsocial, matchmaker_info

    %% ==================
    %% Custom Game - Host
    %% ==================
    Idle --> StartingGame : → game_host\n← game_launch

    %% ==================
    %% Custom Game - Join
    %% ==================
    Idle --> StartingGame : → game_join\n← game_launch
    Idle --> Idle : ← game_join_failed

    %% ==================
    %% Matchmaking
    %% ==================
    state "Matchmaking" as mm {
        SearchingLadder --> StartingAutomatch : ← match_found
        SearchingLadder --> Idle2 : → game_matchmaking(stop)\n← search_info(stop)
        SearchingLadder --> Idle2 : ← match_cancelled
    }
    state "SearchingLadder\n(PlayerState.SEARCHING_LADDER)" as SearchingLadder
    state "StartingAutomatch\n(PlayerState.STARTING_AUTOMATCH)" as StartingAutomatch
    state "Return to Idle" as Idle2

    Idle --> SearchingLadder : → game_matchmaking(start)\n← search_info(start)
    Idle --> Idle : ← search_timeout (banned)
    Idle2 --> Idle

    StartingAutomatch --> StartingGame2 : ← game_launch
    StartingAutomatch --> Idle : ← match_cancelled

    %% ==================
    %% Game Session
    %% ==================
    state "In Game" as ingame {
        state "StartingGame\n(PlayerState.STARTING_GAME)" as StartingGame
        state "StartingGame\n(from matchmaker)" as StartingGame2

        StartingGame --> Hosting : → GameState(Idle) [host]
        StartingGame --> Joining : → GameState(Idle) [guest]
        StartingGame2 --> Hosting : → GameState(Idle) [host]
        StartingGame2 --> Joining : → GameState(Idle) [guest]

        state "Hosting\n(PlayerState.HOSTING)" as Hosting
        state "Joining\n(PlayerState.JOINING)" as Joining

        Hosting --> Hosting : → GameState(Lobby)\n← HostGame
        Joining --> Joining : → GameState(Lobby)\n← JoinGame, ConnectToPeer

        Hosting --> Playing : → GameState(Launching)\n(game.launch → all PLAYING)
        Joining --> Playing : game.launch\n(triggered by host)

        state "Playing\n(PlayerState.PLAYING)" as Playing
    }

    Playing --> Idle : → GameState(Ended)\n/ connection closed

    %% ==================
    %% Restore Session
    %% ==================
    Idle --> Playing : → restore_game_session\n(rejoin live game)

    %% ==================
    %% Disconnect
    %% ==================
    Idle --> [*] : connection lost
    Playing --> [*] : connection lost
```

## Party Flow (overlay on Idle state)

Party interactions happen while the player is in the **Idle** state. The party is a grouping mechanism — it does not change `PlayerState` directly, but the party owner queues all members together when starting matchmaking.

```mermaid
stateDiagram-v2
    state "Idle (solo party)" as Solo

    [*] --> Solo

    Solo --> Invited : ← party_invite(sender)
    Invited --> InParty : → accept_party_invite\n← update_party
    Invited --> Solo : (ignore invite)

    state "In Party" as InParty
    InParty --> InParty : → set_party_factions\n← update_party
    InParty --> Solo : → leave_party\n← update_party(empty)
    InParty --> Solo : ← kicked_from_party

    Solo --> Solo : → invite_to_party\n← update_party (owner invites)
    InParty --> Searching : owner → game_matchmaking(start)\n← search_info(start) to all members

    state "All Members Searching" as Searching
    Searching --> InParty : ← match_cancelled /\n→ game_matchmaking(stop)
    Searching --> GameLaunch : ← match_found → game_launch
```

## State Reference

| PlayerState | Description | Entry Trigger |
|---|---|---|
| `IDLE` | In lobby, no active game/search | Login, game end, search cancel |
| `SEARCHING_LADDER` | Queued for matchmaking | `game_matchmaking(start)` |
| `STARTING_AUTOMATCH` | Match found, awaiting game setup | `match_found` from server |
| `STARTING_GAME` | Game launch sent, awaiting FA process | `game_launch` from server |
| `HOSTING` | Host in game lobby (FA running) | `GameState("Idle")` as host |
| `JOINING` | Guest connecting to host | `GameState("Idle")` as guest |
| `PLAYING` | Game simulation running | `GameState("Launching")` / `game.launch()` |

## Message Legend

| Arrow | Meaning |
|---|---|
| `→ message` | Client sends to server |
| `← message` | Server sends to client |
| `→ GameState(X)` | Game client sends GPGNet command (target: game) |
| `← HostGame / JoinGame / ConnectToPeer` | Server sends GPGNet command to game client |
