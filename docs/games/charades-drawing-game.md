# Game Plan: Multiplayer Charades / Drawing Game

Design and technical reference for a "draw & guess" party game (charades-style,
skribbl.io-like) intended as one of the titles produced through this
repository's game pipeline (see [`../steam-publishing-plan.md`](../steam-publishing-plan.md)).

This is a **planning document**. No implementation exists yet. It is meant to
let development start quickly once picked up.

Last updated: August 2026.

## 1. Decisions & open items

| Item | Status | Notes |
|---|---|---|
| Networking backend | **Undecided** | See [Section 3](#3-networking-backend-decision) for the trade-off analysis; needs a decision before Stage A implementation begins. |
| Player count per room | **Decided: 4–12 players** | Drives networking backend cost/limits (see Section 3) and UI layout for drawing/guessing/scoreboard screens. |
| Round duration | **Decided: configurable 30–90 seconds**, host-set parameter | Default suggestion: 60s. |
| Number of rounds | **Decided: host-configurable** | Each configured "round count" = each player drawing that many times (i.e. `rounds = 3` means every player in the turn order draws 3 times). |
| Interaction/power-up system ("todo later" in original spec) | **Decided: scope now, build later** | Architecture designed in [Section 6](#6-interaction--power-up-system-scoped-not-yet-built); no code yet. |

## 2. Game flow overview

```
Main Menu
  ├─ Host a Room ──────────────┐
  ├─ Join a Room (by code)     │
  ├─ Browse Rooms              │
  └─ Select Avatar             │
                                ▼
                        Room (Lobby / Settings)
                  - Host configures: round count, round duration
                  - Free chat between all players
                  - Host clicks "Start Game"
                                │
                                ▼
                     Countdown (10 → "Go!")
                     Chat is cleared on session start
                                │
                                ▼
              ┌─────────────────────────────────┐
              │   Drawing round loop (per turn)  │
              │  1. Active drawer chosen from    │
              │     shuffled turn order          │
              │  2. Secret word sent to drawer   │
              │     only                          │
              │  3. Drawer draws on shared canvas │
              │  4. Guessers type answers in chat │
              │  5. First correct guesser + the   │
              │     drawer both score a point     │
              │  6. Round ends on correct guess    │
              │     or timer expiry                │
              │  7. Advance to next player in      │
              │     turn order                     │
              └─────────────────────────────────┘
                   Repeats until every player has
                   drawn `rounds` times
                                │
                                ▼
                  Summary / Scoreboard (shown 60s)
                                │
                                ▼
              Vote / auto-return to Room Settings
           (or Main Menu if a player chooses to leave)

Special cases (handled continuously, not just at end-of-flow):
  - Host disconnects  → new host chosen at random from remaining players
  - All players leave → room is destroyed
```

## 3. Networking backend decision

Given the confirmed **4–12 player** range, here is the trade-off, to be
revisited once a decision is made:

| | **GodotSteam** | **GD-Sync** |
|---|---|---|
| Cost at 4–12 players | **Free** — Steam lobbies support far more than 12 members by default | Free tier caps at 4 players. The 12-player upper bound requires the **Advanced plan ($35/month)**, since the mid tiers only cover up to 10. |
| Host migration ("host leaves → random player becomes host") | Steam lobby *ownership* reassigns automatically, but the game's network authority (peer ID 1 in Godot's high-level multiplayer) must be manually rebuilt around the new owner — extra engineering work | **Built-in, native feature** — directly matches the spec with minimal extra code |
| Local dev/testing | Needs multiple Steam accounts or the shared "Spacewar" test AppID (480) to test P2P locally | No Steam account needed at all — faster iteration during development |
| Fit for a low-cost multi-game pipeline | Reusable for free across every future title in the pipeline | Recurring subscription; likely needed at the $35/mo tier for this title given the 12-player max |
| Steam-native integration (achievements, rich presence, overlay) | Native, first-class | Also offers Steam integration alongside its own relay, per their docs, but Steamworks features would layer on top of a non-Steam-relay netcode stack |

**Recommendation (unchanged pending a decision):** prototype with GD-Sync
for development speed (native host migration, no multi-account test
friction), then decide before Steam release whether to keep paying for
GD-Sync or port the finished game logic onto GodotSteam to keep this
title's running cost at $0, consistent with the rest of the pipeline.

Either backend is used through Godot's standard `MultiplayerPeer` interface,
so gameplay code (RPCs, `MultiplayerSynchronizer`, `MultiplayerSpawner`)
does not need to change based on which is chosen — only the lobby/session
setup code differs. This keeps the decision low-risk to defer.

## 4. Core systems

| System | Responsibility | Notes |
|---|---|---|
| **Lobby/Room manager** | Create/join/browse rooms, room codes, player list, avatar selection | Backed by GodotSteam `ISteamMatchmaking` lobbies or GD-Sync `lobby_create`/browsing, per Section 3 |
| **Chat system** | Free chat in lobby (cleared on session start); in-round chat doubles as the guess-input channel | `RichTextLabel` + `LineEdit`; guesses are parsed from the same input stream during drawing rounds |
| **Room settings** | Host-configurable round count, round duration (30–90s) | Synced dictionary/Resource, host is source of truth, broadcast via `@rpc` on change |
| **Turn/round manager** | Shuffles player order once at game start, tracks whose turn it is, tracks how many times each player has drawn, drives the state machine | Host-authoritative state machine: `LOBBY → COUNTDOWN → DRAWING_ROUND → ROUND_TRANSITION → SUMMARY → POST_GAME` |
| **Word bank** | Supplies random secret words | Bundled JSON/CSV resource; delivered only to the active drawer via `rpc_id(drawer_peer_id, ...)` |
| **Drawing canvas sync** | Real-time shared canvas, one active writer | Stroke points broadcast via `@rpc("call_local")`, `unreliable_ordered` transfer mode for in-progress points, reliable RPCs for stroke start/end; reference implementation: [GodotMultiplayerDrawingDemo](https://github.com/Goldenlion5648/GodotMultiplayerDrawingDemo) (MIT, Godot 4) |
| **Scoring** | Awards a point to the first correct guesser and to the drawer on each correct guess | Validated **host-side only** (never trust client-reported correctness) to prevent cheating and to keep the secret word off the wire to non-drawers |
| **Round timer / countdown** | Drives the 10-second start countdown and the 30–90s per-round guess timer | Synced target timestamp (`Time.get_unix_time_from_system()`) rather than a replicated decrementing counter, to avoid drift and reduce bandwidth |
| **Host migration** | Reassign host if the current host disconnects; end the room if everyone leaves | Native in GD-Sync; manual reconnect logic required with GodotSteam (see Section 3) |
| **Interaction/power-up system** | Lets guessers apply positive/negative effects to the active drawer | Scoped in Section 6; not implemented yet |

## 5. Tools & libraries

Carried over from Stage A research:

- **Engine:** Godot 4.7+ (free, MIT, CLI-exportable)
- **Networking:** GodotSteam or GD-Sync (decision pending, see Section 3)
- **Drawing canvas reference:** [GodotMultiplayerDrawingDemo](https://github.com/Goldenlion5648/GodotMultiplayerDrawingDemo)
- **UI:** Godot `Control` nodes (`Button`, `OptionButton`, `RichTextLabel`, `LineEdit`, `ItemList`, `GridContainer`)
- **Settings persistence:** Godot's built-in `ConfigFile`
- **Avatar/UI art:** [Kenney.nl](https://kenney.nl) CC0 asset packs, to avoid needing custom art for an MVP
- **Fonts:** Free playful fonts such as Google Fonts "Fredoka" or "Baloo 2" to fit the party-game tone
- **Local multiplayer testing:** Godot 4.3+ editor's built-in "Run Multiple Instances" debug feature

## 6. Interaction / power-up system (scoped, not yet built)

This formalizes the "todo later" note from the original spec (example:
"guessing player can request for drawer not to use colors, only black
pencil available for 15 seconds. Interactions can be negative or
positive.") into an architecture that the core systems above are designed
to accommodate, so it can be added later without reworking the netcode.

### Design principles

1. **Data-driven, not hardcoded.** Interactions are defined as data entries
   (Resource or JSON), not one-off scripts, so new interactions can be added
   without touching core game/network code.
2. **Host-authoritative.** Only the host validates whether an interaction
   can be triggered (cooldown, cost, timing window), applies it to shared
   round state, and broadcasts the effect to all peers — including the
   drawer.
3. **Enforced in two places.** The drawer's client restricts input locally
   for good UX (e.g., disables the color picker), but the **host also
   validates incoming stroke RPCs against the currently active
   restriction(s)** before relaying them to other peers. This prevents a
   modified client from ignoring a negative interaction.
4. **Start simple on cost/cooldown.** MVP uses a **per-player cooldown**
   (e.g., one interaction use per round) rather than a currency system.
   A points-based "Interaction Points" currency (earned from correct
   guesses) is a natural v2 addition once the cooldown-only version is
   proven.

### Data schema (per interaction definition)

| Field | Type | Example |
|---|---|---|
| `id` | string | `"black_pencil_only"` |
| `display_name` | string | `"Black Pencil Only"` |
| `polarity` | enum | `negative` |
| `target` | enum | `drawer` \| `self` \| `all_guessers` |
| `effect_type` | enum | `restrict_palette` \| `restrict_brush_size` \| `time_bonus` \| `time_penalty` \| `reveal_letter` \| `screen_shake` \| `blur_canvas` |
| `effect_params` | dict | `{"allowed_colors": ["black"]}` |
| `duration_seconds` | float | `15.0` |
| `cost` | int or null | `null` (cooldown-based in MVP) |
| `cooldown_seconds` | float | round-scoped in MVP (one use per player per round) |

### Networking flow

1. Guesser client → host: `request_interaction(interaction_id, target_player_id)` RPC
2. Host validates eligibility (cooldown/cost, correct game phase, valid target)
3. Host → all peers: `apply_interaction(interaction_id, params, start_time, end_time)` broadcast, using the same synced-timestamp pattern as the round timer
4. Host-driven expiry → all peers: `interaction_expired(interaction_id)` broadcast
5. Drawing canvas sync system checks active restriction state before accepting/relaying stroke input from the drawer while an interaction targeting them is active
6. If an interaction modifies round time (`time_bonus`/`time_penalty`), it adjusts the synced round-end timestamp used by the round timer system

### UI

An "interactions panel" visible to non-drawing players during a drawing
round, showing available interactions, per-player cooldown state, and
(later) cost — separate from the chat/guess input so it doesn't interfere
with typing guesses.

## 7. Suggested implementation checklist (for when development starts)

1. Decide the networking backend (Section 3) — blocking decision.
2. Stand up the Godot project skeleton with the state machine (Section 4)
   and empty scenes for each screen (Main Menu, Room/Lobby, Drawing Round,
   Summary).
3. Implement lobby create/join/browse + avatar selection against the chosen
   backend.
4. Implement chat (lobby + in-round) and room settings sync.
5. Implement the turn/round manager and word bank with host-only secret
   delivery.
6. Implement the shared drawing canvas and stroke sync, using the
   referenced open-source demo as a starting point.
7. Implement host-side guess validation and scoring.
8. Implement the countdown/round timer using synced timestamps.
9. Implement host migration / room teardown on disconnects.
10. Implement the summary screen, play-again vote, and return-to-settings
    flow.
11. Only after the above is stable: implement the interaction/power-up
    system per Section 6.
