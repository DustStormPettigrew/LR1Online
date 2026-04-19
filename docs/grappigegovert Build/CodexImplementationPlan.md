# LR1Online Implementation Plan

## Goal

Move `LR1Online` from its current relay-plus-memory-write prototype to a practical online racing architecture that supports:

- 1 local player plus 5 remote players
- dedicated server preferred
- synchronized race start and race clock
- remote kart transform updates that remain stable under internet latency
- correct kart slot mapping for all 5 remote players
- synchronized power-up inventory, activation, projectiles, placed hazards, and applied effects
- authoritative handling of near-simultaneous pickup conflicts
- accurate mini-map and standings
- correct kart configuration for remote racers
- lobby hosting and joining
- at least a viable path to circuit progression and scoring
- compatibility handling for known 1999 vs 2001 gameplay differences where they affect multiplayer state

This plan assumes the current repository state as-is. It does not assume working full multiplayer code already exists.

## Current Reality

The codebase already has the right broad direction:

- a separate `Server` process
- a separate `Client` launcher and runtime
- `LEGORacersAPI` for version-specific process integration
- live racer memory access and remote thread injection
- a transform replication prototype

The codebase does not yet have:

- full 6-racer slot mapping on clients
- authoritative race-state handling
- reliable power-up replication
- lobby readiness flow
- checkpoint, lap, place, and timer authority
- object lifecycle replication for projectiles and hazards
- 1999/2001 rule reconciliation

The safest route is not a rewrite. The safest route is to stabilize the current architecture and then expand authority outward from the server in phases.

## Architecture Decision

Use a **dedicated server authoritative model**.

### Why this is the correct direction

- The current repository already has a standalone server process.
- Pickup conflicts, race starts, standings, lap progress, and power-up effects cannot be trusted to independent client-local simulations.
- Client-hosted authority would work for testing, but it would make desync and host migration more painful later.
- Lego Racers was not built for deterministic internet lockstep. The practical path is server arbitration plus client-side smoothing.

### Recommended authority split

Server owns:

- lobby membership
- race selection
- ready states
- authoritative race start tick and race clock
- remote player slot assignment
- checkpoint and lap progression
- current race place
- item and brick pickup ownership
- power-up inventory truth
- power-up activation truth
- projectile and hazard lifecycle
- effect application and expiration
- circuit scoring

Client owns:

- launching and attaching to the executable
- local input and local rendering
- reading local player state from game memory
- writing remote players into opponent slots
- visual smoothing for remote players
- displaying server-approved state in the local game session

## Core Constraints To Respect

### Constraint 1: Lego Racers only really gives you 6 in-race driver slots

That means the target is exactly:

- `drivers[0]` = local player
- `drivers[1]..drivers[5]` = five remote players or AI stand-ins replaced by remote data

Do not try to exceed this in the first serious implementation.

### Constraint 2: `LEGORacersAPI` is the critical seam

If a feature requires:

- reading or writing race state
- invoking game functions
- mapping racer slots
- handling 1999/2001 address differences

then it belongs in `LEGORacersAPI` first, not scattered in `ClientForm`.

### Constraint 3: Do not trust local collision or pickup results for multiplayer truth

The original game was never designed for online arbitration. If multiple players locally believe they picked up the same item, the server must resolve it.

## Phase 0: Stabilization And Instrumentation

### Objective

Stop building on broken assumptions. Fix the current prototype so later phases have reliable footing.

### Work items

1. Fix participant capacity logic in `Server`

- Current code accepts a TCP client and only then checks whether the participant list is full.
- Replace with a pre-admission model:
- accept socket
- validate room capacity and nickname
- only add to active participant list when accepted

2. Replace ad hoc packet parsing with framed messages

- Current TCP reads use a 64-byte buffer and assume complete packets.
- Define a simple framing protocol:
- length-prefixed binary packets, or
- newline-delimited UTF-8 packets if kept simple

Recommended: length-prefixed packets with explicit message type ids.

3. Stop using culture-sensitive float serialization

- Use `InvariantCulture` for all numeric serialization and parsing.
- Better: move to binary packet encoding for transforms and race-state.

4. Fix event wiring in `LEGORacersAPI`

- `GameClient.Initialize()` currently sets `InitializedType = Both` directly without firing initialization events in a meaningful way.
- Establish a real init sequence:
- core attached
- driver structures available
- in-race hooks ready when applicable

5. Fix `Driver.UsePowerUp()` blue power-up bug

- `Brick.Blue` currently maps to `POWERUP_RED_ADDRESS`.
- Correct it and add regression coverage around address mapping.

6. Add structured logging

- Server log:
- connection lifecycle
- packet receive and send counts
- race-state transitions
- slot assignment
- pickup arbitration
- Client log:
- attach status
- version detection
- packet timings
- slot mapping
- correction and smoothing stats

### Deliverables

- stable packet protocol
- stable server admission logic
- fixed power-up mapping bug
- stable initialization lifecycle
- useful logs for all later debugging

## Phase 1: Establish Proper Remote Slot Mapping

### Objective

Move from "one remote player written into `drivers[1]`" to true support for 5 remote racers.

### Work items

1. Add explicit server-assigned racer slots

Each connected participant should receive:

- stable participant id
- assigned in-race slot number `1..5`

The local player is always slot `0` on their own client, but the server should still track a global participant id so it can reason about standings and events.

2. Expand participant model

Add fields for:

- participant id
- assigned remote slot
- version type
- readiness state
- kart configuration id or payload
- ping and last acknowledged sequence

3. Move remote slot writing into a dedicated client runtime component

Do not leave this logic in `ClientForm`.

Introduce a `RaceReplicationController` or similar that:

- receives replicated remote states
- maps them to `drivers[1]..drivers[5]`
- applies smoothing and corrections
- handles slot vacancy and reconnect cases

4. Ensure AI count and slot ownership are controlled intentionally

The prototype already exposes AI count memory access. Use it deliberately:

- set AI count to the number needed for remote occupancy, or
- keep all opponent slots alive and overwrite all of them consistently

This needs testing on both 1999 NoDRM and 2001, because UI and race-state assumptions may differ.

### Deliverables

- all 5 remote racers can appear in-game simultaneously
- reconnect and disconnect do not scramble slots
- slot ownership remains stable through a race

## Phase 2: Transform Replication That Survives Real Network Conditions

### Objective

Make remote kart movement visually stable and resilient to latency.

### Server model

Server should receive regular state input packets containing:

- client sequence number
- local simulation timestamp or sampled tick
- position
- velocity
- forward vector
- up vector
- current local animation or state hints if needed later

Server should stamp outgoing packets with:

- authoritative server tick
- participant id
- slot id
- latest approved transform state

### Client model

Do not hard-snap remote transforms directly on arrival.

Implement:

1. Sequence-aware receive buffers

- drop out-of-order stale packets
- keep a short interpolation window

2. Interpolation for remote racers

- render remote karts slightly behind real time
- blend between approved states

3. Dead reckoning

- extrapolate briefly using velocity and orientation when packets are delayed

4. Correction thresholds

- soft-correct small drift
- hard-snap only when error exceeds a threshold

### Practical note

Because the game is not designed for external interpolation layers, the smoothing will still end in memory writes to remote driver slots. That is acceptable. The smoothing layer should simply decide what transform gets written this frame.

### Deliverables

- remote karts move smoothly at internet latency
- jitter is reduced significantly
- packet loss does not immediately cause visible teleporting

## Phase 3: Real Lobby Flow

### Objective

Move from "connect by IP and nickname" to a usable race session flow.

### Required lobby states

- connecting
- connected
- in lobby
- selected kart
- ready
- loading race
- countdown
- racing
- finished
- returning to lobby

### Work items

1. Formalize session and participant state machine on server

2. Add ready/unready flow

3. Add host or admin controls on dedicated server

- choose map
- choose mirrored map
- choose race mode
- kick player
- start countdown

4. Synchronize race load and countdown

- server sends selected race
- client acknowledges race-load readiness
- server waits for all ready players
- server sends authoritative countdown start tick

5. Add reconnect policy

Recommended first version:

- reconnect allowed in lobby
- mid-race reconnect optional and deferred unless needed immediately

### Deliverables

- full lobby -> race -> results -> lobby cycle
- deterministic synchronized race start

## Phase 4: Race Clock, Checkpoints, Laps, And Standings

### Objective

Make race progress authoritative and visible consistently across clients.

### Required server-tracked race progress

- race start tick
- elapsed race time
- checkpoint progression
- lap count
- finish state
- place order

### Implementation approach

1. Determine how much of checkpoint and lap state can be read from game memory

If the game exposes checkpoint and lap counters in reliable memory locations, add them to `LEGORacersAPI`.

2. If memory exposure is incomplete, derive race progress from track checkpoint geometry

This is heavier, but still practical:

- define checkpoint volumes externally per track
- server validates crossing order using participant positions

3. Make place order server-authoritative

Do not trust each local game session's place readout if remote karts are being externally written into slots.

4. Sync race clock from the server

- clients can show a local approximation visually
- standings and official elapsed time must come from the server clock

### Mini-map and current place

Mini-map may partially work automatically if remote racers are occupying real opponent slots.

Still verify:

- all five remote karts appear on radar
- ordering does not break when smoothing writes lag behind the server's official place

If the built-in place logic becomes unreliable, add a client overlay for authoritative standings rather than trying to patch every internal UI path immediately.

### Deliverables

- correct lap and checkpoint progression
- server-approved place ordering
- synchronized race timer

## Phase 5: Power-Up Inventory And Activation Authority

### Objective

Turn power-ups from local guesses into synchronized gameplay.

### Inventory model

Server tracks per participant:

- current colored brick
- current white-brick count
- inventory change sequence
- currently active effect state

### Work items

1. Extend `LEGORacersAPI` accessors

Expose consistent read and write APIs for:

- brick type
- white-brick count
- possibly active effect flags if discoverable

2. Detect local pickup attempts and local activations

Clients report:

- attempted pickup event
- attempted power-up use event

3. Server validates and approves

On approval:

- server updates authoritative inventory
- server broadcasts inventory and effect result

4. Client applies approved inventory

- local client corrects its own inventory if needed
- remote clients update visual state and effect state

### Important design rule

The server should own **whether the player actually has the power-up** and **whether the use actually succeeds**.

Without that, desync is guaranteed.

### Deliverables

- all clients agree on inventory state
- power-up activation is authoritative

## Phase 6: Pickup Conflict Resolution

### Objective

Handle near-simultaneous pickups cleanly.

### Required model

Each pickup source must have:

- stable pickup id
- position
- availability state
- respawn timing if applicable

### Arbitration rule

A simple initial rule is enough:

- first valid server timestamp wins
- tie-break by smaller distance to pickup center
- final tie-break by participant id

### Client behavior

Clients may predict local pickup for responsiveness, but they must accept rollback:

- if approved, keep it
- if denied, revert local inventory

### 1999 vs 2001 issue

Pickup radius and brick poisoning differences need to be treated as server-side rules. Do not let each client independently decide those mechanics.

Recommended approach:

- define a ruleset per session:
- `1999-nodrm-rules`
- `2001-rules`
- if mixed versions are ever supported in one session, the server must choose one canonical ruleset and clients must accept correction

### Deliverables

- same pickup cannot be permanently won by multiple players
- inventory conflicts are resolved consistently

## Phase 7: Projectiles, Hazards, And Applied Effects

### Objective

Make offensive and defensive power-ups visible and authoritative.

### This phase is where multiplayer becomes real gameplay instead of ghost racing.

### Required replicated entities

- projectile instance id
- owner id
- spawn tick
- transform
- target or guidance state if applicable
- hazard placement id and lifetime
- applied effect id and duration

### Recommended implementation path

1. Start with simple applied effects

- speed boost
- slowdown
- direct status effects with no free-flying world object

2. Then add placed hazards

- simpler than moving projectiles

3. Then add projectiles

- most stateful and most likely to expose engine integration gaps

### Key decision

Do not rely on every client invoking the original game effect locally based only on "player used red."

Instead:

- server decides the effect target and timing
- clients apply the exact approved result

Where possible, use original game internal functions through `LEGORacersAPI`.
Where those functions are too brittle or produce divergent local simulation, directly set the resulting state instead.

### Deliverables

- visible synchronized effects
- hazards and projectiles that behave consistently for all players

## Phase 8: Kart Configuration Sync

### Objective

Ensure remote players are represented with the right racer setup.

### Required replicated fields

- driver identity
- car build or parts
- cosmetic selections
- license or displayed name metadata if relevant

### Work items

1. Identify where selected kart configuration lives in memory or file-backed runtime state

2. Add a serializable config payload

3. Send it in lobby before race load

4. Apply it to remote slots before the race starts

### Risk

This may be harder than transform replication if the game strongly assumes only one local customized racer. If so, fall back to:

- a restricted supported kart subset first
- or a mirrored "display approximation" mode until full config injection is understood

### Deliverables

- remote racers visually match their configured build closely enough for actual online play

## Phase 9: Nameplates And UI Overlay

### Objective

Add missing UX without blocking core multiplayer.

### Nameplates

Possible approaches in order of pragmatism:

1. External overlay drawn by client app
- easiest to iterate
- least invasive

2. In-engine text or sprite injection
- cleaner visually
- higher reverse-engineering cost

Start with the external overlay.

### UI items worth overlaying first

- nearby player names
- authoritative place
- synchronized race clock
- connection quality
- current inventory if the base game UI becomes unreliable under net corrections

### Deliverables

- usable online HUD even if built-in UI elements are imperfect

## Phase 10: Circuit Mode And Scoring

### Objective

Support multi-race progression if the architecture allows it cleanly.

### Recommended approach

Do not force this early.

Single-race synchronization must work first. After that:

- server tracks points table
- server advances lobby to next circuit race
- clients load the next race through the existing `SetupRace()` path

### If the native circuit flow proves too brittle

Implement server-side "custom circuit session" that:

- chooses a sequence of tracks
- tracks points externally
- presents standings in overlay or launcher UI

That is acceptable and likely safer than trying to fully coerce the game's original circuit logic at first.

### Deliverables

- repeatable multi-race sessions with stable scoring

## 1999 And 2001 Compatibility Strategy

### Current state

The repository supports address maps for:

- 1999 NoDRM
- 2001

1999 DRM is not actually safe yet.

### Recommended strategy

1. Treat 1999 NoDRM and 2001 as separate supported targets

2. Add a server-declared session ruleset

- movement and pickup reconciliation rules
- power-up behavior differences if any
- brick poisoning behavior if relevant
- pickup radius values

3. Keep mixed-version sessions disabled until:

- remote slot mapping
- pickup arbitration
- effect replication
- and lap and place authority

are all stable enough to validate cross-version parity.

### Deliverables

- explicit supported version matrix
- no accidental false promise that all versions interoperate equally

## Suggested Refactor Targets

### Move logic out of forms

Current forms are doing too much coordination work.

Introduce service-level components:

- `SessionClient`
- `PacketCodec`
- `RaceReplicationController`
- `InventoryReplicationController`
- `LobbyStateMachine`
- `RaceStateMachine`

### Strengthen `LEGORacersAPI`

Add explicit APIs for:

- driver slot control
- lap and checkpoint reads
- inventory reads and writes
- racer config reads and writes
- effect application hooks

### Keep server clean

Server should separate:

- networking
- session management
- race authority
- event arbitration
- admin UI

## Testing Strategy

### Phase-by-phase testing

1. Single machine, multiple game instances if feasible

- fastest iteration for networking and slot mapping

2. LAN testing

- packet timing and race flow validation

3. High-latency simulation

- add artificial delay, jitter, and packet loss

4. Version-matrix testing

- 1999 NoDRM to 1999 NoDRM
- 2001 to 2001
- mixed-version only when explicitly ready

### Essential automated coverage

Not every part can be unit tested because much of this is process integration, but some parts can and should be:

- packet codec
- session state transitions
- pickup arbitration logic
- place ordering logic
- scoring logic

## Priority Order

If time is limited, the best sequence is:

1. Stabilize protocol and fix current correctness bugs
2. Map all five remote racers into slots reliably
3. Add smoothing and sequence-aware transform replication
4. Add real lobby and synchronized race start
5. Make lap, place, and timer authoritative
6. Make inventory and pickups authoritative
7. Add effects, hazards, and projectiles
8. Add kart config sync
9. Add overlays and polish
10. Add circuit scoring

## Minimum Viable "Real Online Race"

The project can be considered to have reached a meaningful first milestone when all of the following are true:

- 6 total racers appear correctly as local plus 5 remote
- remote movement is smooth enough for normal racing
- server starts the race at one agreed time
- server owns lap, place, and timer
- item pickups are server-arbitrated
- activated power-ups are applied consistently across all clients
- disconnects do not corrupt slot ownership

That is the real transition point from prototype to actual multiplayer game.

## Final Recommendation

Do not spend early effort on nameplates, circuit scoring, or polished UI until:

- slot mapping
- transform smoothing
- race-state authority
- and pickup and power-up authority

are in place.

The current codebase is already pointed at the right broad model. The real work is not inventing a new architecture. The real work is converting a relay prototype into an authoritative multiplayer session while preserving the game's 6-slot in-race structure and isolating version-specific executable behavior inside `LEGORacersAPI`.
