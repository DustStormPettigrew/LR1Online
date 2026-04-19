# LR1Online — Claude Implementation Plan

## Feasibility Assessment: External Manipulation vs Executable Modding

### The short answer

Almost everything can be achieved through the existing external memory manipulation approach. No permanent on-disk modification of LEGORacers.exe is required. However, several features require **new runtime patches** — writing new instructions into the running process at startup — which the project already does for other purposes (NOP-ing out RRB loading, detouring AI powerup code, patching the background-run check). The technique is proven; it just needs to be applied to more areas.

A small number of features push toward **DLL injection** (loading a custom .dll into the game process via `CreateRemoteThread` + `LoadLibrary`). DLL injection is still runtime-only with no permanent exe change, but it provides cleaner integration for hooks that need to live inside the game's own call stack. The existing `CreateRemoteThread` infrastructure supports this transition naturally.

### What the project can already do today

The codebase demonstrates all the building blocks:

- Read/write arbitrary game memory (`ReadProcessMemory`/`WriteProcessMemory`)
- Allocate executable memory inside the game process (`VirtualAllocEx`)
- Inject and execute x86 assembly (`CreateRemoteThread`)
- NOP out instructions to disable game behavior (`SetLoadRRB`)
- Detour functions with JMP-to-cave patterns (`SetAIUsePowerUps`)
- Call native game functions with crafted register state (`UsePowerUp`, `GotoMenu`)
- Overwrite in-game text strings (`RemoveMenuButtons` modifying menustrings.srf)

These capabilities are sufficient for every feature in the end-state goal. The gap is not in the manipulation technique — it is in **undiscovered memory addresses and game functions** that need reverse engineering.

---

## Feature-by-Feature Breakdown: What Needs What

### Features achievable with current external approach (no new patches)

| Feature | How |
|---|---|
| 6-player slot mapping | Remove the `break`, index remote participants to `drivers[1..5]` |
| UDP coordinate broadcast to all players | Fix server UDP routing (store per-player endpoints) |
| TCP message framing | Length-prefix packets, fix 64-byte buffer assumption |
| Float locale fix | Use `CultureInfo.InvariantCulture` everywhere |
| Thread safety | Lock the participant list or use `ConcurrentDictionary` |
| Lobby state machine | Pure networking/UI, no game interaction |
| Race selection broadcast | Already works via `PacketType.Race` |
| Circuit scoring | Server-side tracking, overlay or lobby UI display |
| Kick/disconnect handling | Mostly works, needs cleanup for slot vacating |

### Features requiring NEW runtime patches (same technique, new addresses)

These use the same `WriteProcessMemory` approach the project already uses to NOP instructions and detour functions. They require reverse engineering to find the right addresses.

| Feature | What to patch | RE difficulty |
|---|---|---|
| **Disable AI movement for network slots** | Find the AI update function, NOP or detour the movement code for driver indices controlled by network. Similar pattern to the existing `SetAIUsePowerUps` detour — compare driver index, skip if network-owned. | Medium. The AI powerup detour already identifies the local player by comparing `ecx` to `esi`; the movement function likely follows a similar structure. |
| **Disable local brick pickup collision** | Find the brick collision handler, detour it to a cave that checks a flag before allowing pickup. The flag is set/cleared by the external process to implement server-authoritative pickups. | Medium-Hard. Need to find the collision detection function and understand its parameters. |
| **Read/write race timer** | Find the timer address in the race state structure. The `INRACE_BASEADDRESS` pointer chain is already known; the timer is likely at a nearby offset. | Easy-Medium. Systematic memory scanning during a race with Cheat Engine can find this quickly. |
| **Read/write lap/checkpoint counters** | Find per-driver lap count and checkpoint index. Likely stored in the same driver structure that holds position (the large struct at ~0xD58+ offsets). | Medium. Need to identify the checkpoint system's memory layout. |
| **Control AI driver count precisely** | `AIDriversAmount` already exposed. Needs to be set to match online player count before race start, and AI pathfinding must be fully disabled for network slots. | Easy (count) + Medium (pathfinding disable). |
| **Force-set brick inventory from server** | Already possible via `Driver.Brick` and `Driver.WhiteBricks` setters. The server just needs to send authoritative state and the client writes it. | Already done, just needs wiring. |

### Features requiring DLL injection (recommended but not strictly necessary)

These CAN be done externally but would be significantly cleaner and more reliable as an injected DLL.

| Feature | Why DLL injection helps | External fallback |
|---|---|---|
| **Frame-synced position interpolation** | Hook the game's main update/render loop to write interpolated positions at exactly the right moment each frame. Eliminates mid-frame tearing from async writes. | Write at high frequency (1ms) from external process. Approximate but functional. The game runs at ~30fps, so 1ms writes land roughly 1 write per frame with some waste. |
| **Player name rendering** | Hook DirectDraw `Flip`/`Blt` or Direct3D `Present` to inject text drawing after the game renders but before the frame displays. | External transparent overlay window positioned on top of the game window. Fragile (breaks on alt-tab, resolution changes, fullscreen) but functional for windowed mode. |
| **Intercepting brick pickup at the function level** | Hook the pickup function itself to call back to the external process for server validation before allowing the pickup to complete. Provides true server-authoritative pickups. | Poll brick state at high frequency and revert unauthorized pickups after-the-fact. Introduces a brief flicker where the player sees the brick then loses it. Acceptable for v1. |
| **Intercepting power-up activation** | Hook the activation function to validate with server before firing. | Detect activation via polling (existing `CheckPowerUp`), send to server, let server broadcast `UsePowerUp` to all clients. Activation delay equals round-trip time. |

### Features requiring significant reverse engineering (game knowledge, not code technique)

These don't need new patching approaches — they need someone to find the memory addresses.

| Feature | What needs to be found | Approach |
|---|---|---|
| **Kart build configuration per slot** | Where the game stores the car model, chassis parts, driver identity, and color per driver slot. Need read access on local player and write access on AI slots. | Cheat Engine: change car parts in the build screen, scan for changed values. Cross-reference with the driver structure base addresses already known. |
| **Checkpoint volumes per track** | Either find the game's internal checkpoint list in memory, or define them externally by analyzing track geometry (track data is parsed by LibLR1). | Two paths: (1) RE the game's checkpoint system, or (2) extract checkpoint positions from track files and do server-side crossing detection using player coordinates. Path 2 avoids RE entirely. |
| **Projectile and hazard entity list** | Where active projectiles/hazards are tracked in memory. Needed to replicate their state across clients. | Hard. Alternatively, rely on the existing `UsePowerUp()` injection which calls the game's native function — this spawns the correct visual effect. The gap is that hit detection is still local, so the server would need to arbitrate hits based on position proximity. |
| **Brick spawn positions and state** | Where the game tracks which colored bricks are on the track, their positions, and their cooldown timers. | Medium. Bricks are world objects. Could be found by scanning memory near known object structures, or by extracting spawn positions from track data and tracking state server-side. |
| **Race result / finish state** | Where the game stores "this driver finished" and final time. | Medium. Likely near the race state structure. May fire an event or set a flag at the same base as `INRACE_BASEADDRESS`. |

---

## Architecture: Dedicated Server with Phased Authority

The existing server process is the right foundation. The plan expands its authority in phases rather than rebuilding.

### Authority model

```
Server is authoritative for:          Client is responsible for:
- Lobby state & slot assignment        - Launching & attaching to game process
- Race selection & countdown           - Reading local player state from memory
- Race clock & start tick              - Writing remote players into driver slots
- Checkpoint & lap progression         - Local input & rendering
- Race placement                       - Visual interpolation/smoothing
- Brick pickup ownership               - Applying server-approved state
- Power-up inventory truth             - Power-up visual effect triggering
- Power-up activation approval         - Overlay rendering (names, HUD)
- Projectile/hazard lifecycle
- Circuit scoring
- Version ruleset enforcement
```

### Protocol redesign

Replace the current text-based `PacketType|Content` format with binary length-prefixed messages:

```
[2 bytes: message length][1 byte: message type][N bytes: payload]
```

Position updates should be binary-encoded floats (4 bytes each), not string-serialized. This cuts coordinate packet size from ~200+ bytes (ASCII) to ~52 bytes (binary: 1 byte type + 1 byte slot + 12 floats x 4 bytes + 2 bytes sequence + 4 bytes timestamp).

UDP packets should include a sequence number and client timestamp. Server stamps outgoing packets with server tick.

---

## Implementation Phases

### Phase 0: Stabilization

Fix the bugs that would make everything built on top unreliable.

**0.1 — Fix critical networking bugs**
- **UDP routing for multiple players** (`Server.cs` UdpListener): Store each participant's UDP endpoint when they first send coordinates. Send responses back to the correct endpoint, not just `ipEndPoint` which gets overwritten.
- **TCP message framing**: Implement length-prefixed reads. Current 64-byte buffer will truncate any packet with 6 players' worth of data.
- **Float parsing locale**: Add `CultureInfo.InvariantCulture` to every `float.Parse()` call in `ClientForm.cs` and `Server.cs`. Better yet, move to binary encoding.
- **Thread safety**: Wrap `participants` list access in `lock` blocks or switch to `ConcurrentBag`/`ConcurrentDictionary`.

**0.2 — Fix API bugs**
- **Blue powerup address**: `Driver.UsePowerUp()` line 312 maps `Brick.Blue` to `POWERUP_RED_ADDRESS`. Change to `POWERUP_BLUE_ADDRESS`.
- **NewMemory contention**: The single 1024-byte injection buffer is shared across all operations. Either allocate separate buffers per operation, or add a mutex to serialize injections.

**0.3 — Add version reporting**
- Client sends its game version (1999-nodrm, 2001) to server during Connect.
- Server stores version per participant.
- Server rejects connections from mismatched versions (configurable: strict or permissive).

### Phase 1: Full 6-Player Slot Mapping

**1.1 — Server assigns slot indices**
- On successful Connect, server assigns each participant a slot index 1-5.
- Slot 0 is always the local player on each client.
- Server broadcasts the slot map to all clients.
- Disconnecting a player frees their slot; new players can take freed slots.

**1.2 — Client writes all remote slots**
- Remove the `break` in `ClientForm.client_PacketReceived()`.
- Map each remote participant to their assigned `drivers[N]` slot.
- Only write to slots that have active remote players.

**1.3 — AI management**
- Set `AIDriversAmount` to match the number of connected players minus one (so the game creates the right number of driver structures).
- Disable AI pathfinding for all slots via `LoadRRB = false` (already done).
- **New patch needed**: Find and NOP/detour the AI movement update function so AI code doesn't overwrite network-written positions. Pattern: similar to `SetAIUsePowerUps` — inject a check at the top of the AI movement function that skips execution if the driver index is in the network-controlled set.

**1.4 — Handle joins and disconnects mid-race**
- Mid-race join: assign the new player an empty slot, set AI count, write their initial position.
- Mid-race disconnect: stop writing to that slot. The kart will freeze in place (acceptable for v1). Optionally teleport it off-track or make it invisible if a "driver visible" flag can be found.

### Phase 2: Transform Smoothing

**2.1 — Add sequence numbers and timestamps to position packets**
- Client includes a monotonically increasing sequence number and a local millisecond timestamp with each coordinate update.
- Server stamps outgoing broadcasts with its own tick counter.

**2.2 — Client-side interpolation buffer**
- On the receiving client, buffer the last 2-3 position snapshots per remote driver.
- Instead of writing the latest snapshot directly, interpolate between the two most recent snapshots.
- Display remote karts at a fixed delay behind real-time (e.g., 50ms behind). This provides a smooth window to interpolate within.

**2.3 — Dead reckoning**
- When no new packet has arrived for a remote driver within the expected interval, extrapolate using their last known velocity vector.
- Cap extrapolation at ~200ms to avoid runaway drift.
- When a new packet arrives, blend from the extrapolated position to the real position over 2-3 frames.

**2.4 — Write frequency**
- The interpolation layer should write to game memory at the game's effective frame rate (~30fps = ~33ms intervals) or faster.
- Move the write loop to a high-priority timer thread rather than a `Thread.Sleep(10)` loop.

**2.5 — (Optional, Phase 2+) Frame-sync via DLL injection**
- If external high-frequency writes prove insufficient (visible jitter), inject a DLL that hooks the game's main loop and performs the position write at the exact right moment each frame.
- This is an optimization, not a blocker.

### Phase 3: Lobby and Race Start Synchronization

**3.1 — Lobby state machine**

States for each participant:
```
Connected -> InLobby -> KartSelected -> Ready -> Loading -> WaitingForStart -> Racing -> Finished -> InLobby
```

Server manages transitions and broadcasts state changes.

**3.2 — Ready-up flow**
- Server admin selects track (already works via `NewRaceForm`).
- Server broadcasts track selection.
- Each client calls `SetupRace()` to navigate to the race.
- Each client reports "loaded" when `IsRaceRunning` becomes true.
- Server waits for all clients to report loaded.

**3.3 — Synchronized start**
- Once all clients report loaded, server sends a "start at server_tick + N" message (N chosen to account for worst-case RTT among connected clients).
- Each client enters the race paused (`Paused = true` — already implemented).
- Each client calculates the local time corresponding to the server's start tick and unpauses at that moment.
- This gives a synchronized "GO" within one network round-trip of accuracy.

**3.4 — Race clock authority**
- Server maintains the authoritative elapsed race time starting from the start tick.
- Server broadcasts race clock periodically (every ~1s).
- Clients can display a locally-ticking approximation but defer to server for official times.
- **RE needed**: Find the race timer memory address to write the server-synced value into the game's HUD. If not found, display in overlay instead.

### Phase 4: Checkpoint, Lap, and Placement Authority

**4.1 — Two approaches, pick based on RE success**

**Approach A: Game-memory checkpoint reading (preferred)**
- Reverse-engineer the per-driver checkpoint index and lap counter from the driver structure. These are likely at fixed offsets from the same base address used for position/brick data.
- Client reads local player's checkpoint/lap from memory and reports to server.
- Server validates progression (checkpoints must advance in order, lap increments only after all checkpoints).
- Server broadcasts authoritative placement order.

**Approach B: Server-side geometric checkpoint detection (fallback)**
- Define checkpoint trigger volumes per track using positions extracted from track data (LibLR1 can parse track files).
- Server uses player positions (already received via coordinate packets) to detect checkpoint crossings.
- No game memory RE needed beyond what's already known.
- Downside: requires manually defining checkpoints for all 13 tracks (+ mirrors).

**4.2 — Placement calculation**
- Server computes placement based on: lap count (primary), checkpoint index (secondary), distance to next checkpoint (tertiary).
- Server broadcasts placement order with each coordinate update response.
- **RE needed**: Find the "current place" display value in game memory to write the server's authoritative placement. If not found, display in overlay.

**4.3 — Finish detection**
- Server detects when a player completes the final lap (crosses last checkpoint with lap count = target).
- Server records finish order and times.
- Server broadcasts race results to all clients.

### Phase 5: Power-Up Inventory Authority

**5.1 — Client reports pickup attempts**
- The existing `PickedUpBrick` event fires when the local player picks up a brick.
- Client sends a `PickupRequest` packet to server: `{brick_type, white_brick_count, position, timestamp}`.

**5.2 — Server validates and approves**
- Server checks: is this brick type available at this position? Has another player already claimed it?
- If approved: server updates authoritative inventory, broadcasts `PickupApproved` to requester and `InventoryUpdate` to all.
- If denied: server sends `PickupDenied` to requester. Client writes `Brick = Brick.None` and `WhiteBricks = 0` back into game memory, reverting the local pickup.

**5.3 — Enforcing inventory on clients**
- On every coordinate update response, server piggybacks each player's authoritative brick type and white brick count (already partially in the packet format).
- Clients write the server's values into each remote `Driver.Brick` and `Driver.WhiteBricks`.
- Local player's inventory is also validated against server state; corrections applied if diverged.

**5.4 — Brick poisoning rule**
- In 1999: picking up a different colored brick replaces your current one (intended mechanic).
- In 2001: brick poisoning behavior may differ.
- Server declares the active ruleset at session start. The server's pickup approval logic uses the declared rules. Clients accept the result regardless of their local version's behavior.

### Phase 6: Pickup Conflict Resolution

**6.1 — Server-side brick tracking**

Two sub-approaches:

**6.1a — Passive (poll-based, no new patches)**
- Server doesn't track individual brick spawns.
- When two clients report picking up a brick at nearly the same time and position, server uses first-timestamp-wins with distance-based tiebreaker.
- Loser gets their inventory reverted.
- **Weakness**: brief visual flicker on the losing client (they see the brick, then lose it).

**6.1b — Active (requires RE + runtime patch)**
- Find the brick entity list in game memory — positions and cooldown states for all colored bricks on the track.
- **New runtime patch**: Detour the brick pickup collision function. When a player touches a brick, instead of immediately picking it up, the detoured code sets a "pending" flag and the external process sends a server request. The brick is only awarded when the server approves.
- **This eliminates the flicker entirely** but requires finding and hooking the collision function.
- **RE difficulty**: Hard. Recommend starting with 6.1a and upgrading to 6.1b later.

**6.2 — Brick pickup radius reconciliation**
- 1999 has a different (larger? smaller?) brick pickup radius than 2001.
- In a same-version session: not an issue, all clients agree.
- In a mixed-version session (if ever supported): server must define a canonical pickup radius. The runtime patch (6.1b) would use the server's radius rather than the game's built-in value.
- For v1: enforce same-version sessions and defer this.

### Phase 7: Power-Up Activation and Effects

**7.1 — Activation authority**
- When the local player uses a power-up, the `PowerUpUsed` event fires.
- Client sends `ActivateRequest{brick_type, white_bricks, position, forward_vector, timestamp}` to server.
- Server validates (does this player actually have this brick? are they in a valid state to use it?).
- Server broadcasts `ActivateEffect{participant_id, slot_id, brick_type, white_bricks, position, direction}` to all clients.

**7.2 — Remote effect triggering**
- Receiving clients call `Driver.UsePowerUp(brick, whiteBricks)` for the appropriate driver slot. This injects assembly to call the game's native power-up function, spawning the correct visual effect (projectile, shield, speed boost, trap, etc.) from that driver's current position.
- This already works in the codebase; it just needs to be wired to the activation broadcast instead of `MessageBox.Show()`.

**7.3 — Effect categories and their sync requirements**

| Effect type | Examples | Sync approach |
|---|---|---|
| **Self-applied** | Green speed boost, Blue shield | Server approves, each client applies to the correct driver slot. Visual is local. |
| **Projectile** | Red cannonball, Yellow oil slick launcher | Server approves activation. Projectile spawns from the driver's current position/direction on each client via `UsePowerUp()`. Hit detection is local per-client (acceptable for v1). |
| **Placed hazard** | Oil slick on track, planted trap | Server records placement position. Broadcasts spawn to all clients. Each client spawns the hazard via the game's native function. |
| **Targeted** | Homing missile (red+3 white) | Server approves. Target is chosen locally by the game's own targeting logic. If targeting logic diverges across clients, server can mandate the target (requires RE to find the target assignment). |

**7.4 — Hit detection**
- For v1: let each client's local physics determine hits. If your local game says the cannonball hit you, you got hit. Report the hit to the server.
- Server can validate plausibility (was the projectile near the victim? was the timing reasonable?) but doesn't need to simulate projectile physics.
- For later: server-authoritative hit detection based on positions and timing. Requires defining hitboxes and projectile trajectories server-side.

### Phase 8: Kart Configuration Sync

**8.1 — RE the kart build data**
- In the build screen, the player selects chassis, engine, driver model, and colors.
- This configuration must be stored somewhere in memory when entering a race, because the game renders the correct model.
- **RE approach**: Build different cars, enter a race, scan for values that differ between builds. The driver structure is large (~0xD58 bytes for 1999); the build data is likely within it.

**8.2 — Serialize and transmit**
- Client reads its kart config from game memory after entering the ChooseRacer screen.
- Client sends kart config to server.
- Server stores it per participant and broadcasts to all clients when the race loads.

**8.3 — Apply to remote slots**
- Before the race starts (during the Loading/WaitingForStart state), each client writes the remote players' kart configs into the appropriate driver slot memory.
- **Risk**: The game may construct the 3D model from the build data only once during race load. If so, the config must be written BEFORE the game initializes driver rendering. This may require finding the model construction function and triggering it after writing the config, or hooking the load sequence to inject the config at the right time.
- **Fallback**: If per-slot model injection proves too complex, assign each remote player one of the game's built-in AI racer presets. Not ideal but functional.

### Phase 9: Nameplates and UI Overlay

**9.1 — External overlay (v1)**
- Create a transparent, topmost, click-through WinForms window positioned over the game window.
- Use GDI+ or Direct2D to draw player names at screen positions calculated from 3D world positions.
- **Requires**: A projection function that converts (X, Y, Z) game coordinates to (screenX, screenY) pixels. This needs the game's camera matrix, which must be found in memory.
- **Limitation**: Only works reliably in windowed mode. Fullscreen may require a different approach.

**9.2 — DirectDraw hook (v2, DLL injection)**
- Inject a DLL that hooks `IDirectDrawSurface::Blt` or `Flip`.
- After the game renders a frame, draw names and HUD elements onto the back buffer before it flips to screen.
- Works in fullscreen.
- **This is the one feature that most benefits from DLL injection over pure external manipulation.**

**9.3 — What to display**
- Player nickname above kart (fade in when close, fade out when far)
- Server-authoritative placement (1st, 2nd, etc.)
- Synchronized race clock
- Connection quality indicator (ping)
- Power-up inventory if the base game HUD becomes unreliable under server corrections

### Phase 10: Mini-Map Accuracy

**10.1 — Likely already works**
- The game's mini-map reads driver positions from the same memory structures that LR1Online writes to. If all 6 driver slots have correct positions, the mini-map should display all 6 dots.
- **Test this early** (during Phase 1). If it works, no additional effort needed.

**10.2 — If it doesn't work**
- The mini-map may use a separate position cache or only update for locally-simulated drivers.
- Find the mini-map's data source via RE and ensure network-controlled drivers are included.
- Alternatively, draw a custom mini-map in the overlay (Phase 9).

### Phase 11: Circuit Mode and Scoring

**11.1 — Server-side circuit session**
- Server defines a circuit as an ordered list of tracks (using the existing `Circuit` class data).
- After each race finishes, server collects results, updates points, and advances to the next track.
- Server broadcasts the next track selection, triggering `SetupRace()` on all clients.

**11.2 — Scoring rules**
- Points per placement: 1st=10, 2nd=8, 3rd=6, 4th=4, 5th=2, 6th=1 (or configurable).
- Server tracks cumulative points across races.
- After the final race, server declares circuit winner.

**11.3 — The `SetupRace()` menu navigation challenge**
- `SetupRace()` navigates through the menu system with `Thread.Sleep(30)` delays between transitions. This is fragile.
- For circuit mode, after a race ends the game returns to a results screen. Navigating from results → menu → next race requires understanding the post-race menu flow.
- **RE needed**: Find the post-race state and menu transitions, or find a more direct way to load a track (e.g., by writing the track ID directly into the race-load structure and calling the load function).

---

## 1999 vs 2001 Compatibility Strategy

### Enforce same-version sessions for v1

- Server stores each client's version.
- Server rejects connections from clients running a different version than the first connected client (or a server-configured target version).
- This sidesteps all behavioral differences for the initial release.

### Known behavioral differences requiring eventual reconciliation

| Difference | Impact | Resolution |
|---|---|---|
| **Brick pickup radius** | Two clients may disagree on whether a player is close enough to pick up a brick. | Server-authoritative pickup with server-defined radius. For v1 with same-version enforcement, not an issue. |
| **Brick poisoning** | Picking up a brick of a different color — does it replace or stack? Behavior may differ. | Server-authoritative inventory. Server decides the result of a pickup based on declared ruleset. |
| **White brick offset** | 1999 uses offset `0xD58`, 2001 uses `0x870`. | Already handled by version-specific subclasses. No additional work. |
| **PauseGame assembly** | 2001 requires an extra `push 00`. | Already handled in `GameClient.PauseGame()`. |
| **All memory addresses** | Every address differs. | Already handled by `Client_1999NoDRM` and `Client_2001` subclasses. Any new addresses found during RE must be added to BOTH subclasses. |

### For future mixed-version support (post-v1)

- Server declares a canonical ruleset (e.g., "1999 rules" or "2001 rules").
- Pickup radius, poisoning behavior, and any other mechanical differences are enforced server-side.
- Clients accept server corrections even when they conflict with local game behavior.
- Any runtime patches that modify game behavior (e.g., overriding pickup radius) must have version-specific implementations.

---

## Pieces That Specifically Need Executable-Level Work

To be clear: none of these require permanent on-disk modification. All are runtime patches applied after the external process attaches.

### Must-have runtime patches (required for end-state)

1. **AI movement suppression for network slots**
   - Find the AI steering/movement update function.
   - Inject a detour that checks the driver index against a "network-controlled" flag array stored in the allocated memory block.
   - Skip AI movement for flagged indices, allowing network writes to persist without fighting.
   - Pattern: identical to existing `SetAIUsePowerUps` detour.
   - Needed in: BOTH `Client_1999NoDRM` and `Client_2001` with their respective addresses.

2. **Background execution patch**
   - Already implemented (`RunInBackground` flips a `JNE` to `JMP`).
   - Essential for multiplayer — the game must keep running when alt-tabbed.

3. **RRB (AI path) loading suppression**
   - Already implemented (`SetLoadRRB` NOPs out the loading code).
   - Without AI paths loaded, AI movement is disabled at the data level. But the movement CODE may still execute and produce garbage movement. Patch #1 above is the belt to this suspender.

### Nice-to-have runtime patches (improve quality significantly)

4. **Brick pickup interception (for flicker-free authoritative pickups)**
   - Detour the brick collision function to check an approval flag before granting the pickup.
   - Without this, server-authoritative pickups work but with a brief flicker on denied pickups.

5. **Power-up activation interception (for server-approved activation)**
   - Detour the powerup activation function to check server approval before firing.
   - Without this, the local player's powerup fires immediately and the server must validate after-the-fact.

6. **Direct track loading (bypass menu navigation)**
   - Find the function that loads a track directly and call it, bypassing the fragile `GotoMenu` chain.
   - Massively simplifies race transitions for circuit mode.

### Recommended DLL injection (for highest quality, not strictly required)

7. **DirectDraw/Direct3D render hook**
   - For player name rendering and HUD overlay in fullscreen.
   - Inject a DLL via `CreateRemoteThread` + `LoadLibraryA`.
   - Hook the flip/present function.
   - The DLL communicates with the external process via shared memory or named pipe.

---

## Reverse Engineering Roadmap

Ordered by importance to the end-state goal.

| Priority | Target | How to find it | Used by |
|---|---|---|---|
| 1 | AI movement update function address | Set breakpoint on driver position write, trace up to the function that moves AI drivers. Compare with local player movement path. | Phase 1 (AI suppression) |
| 2 | Race timer address | Scan for a float or int that increments during a race and resets between races. Likely at an offset from `INRACE_BASEADDRESS`. | Phase 3 (clock sync) |
| 3 | Per-driver lap count and checkpoint index | Scan for values that change at lap/checkpoint boundaries. Likely in the driver structure (offsets from driver base). | Phase 4 (placement) |
| 4 | Camera view/projection matrix | Scan for a 4x4 float matrix that changes with camera movement. Needed to project 3D positions to screen coordinates. | Phase 9 (nameplates) |
| 5 | Kart build configuration per driver slot | Change car parts, scan for values. Cross-reference with the driver structure. | Phase 8 (kart config) |
| 6 | Brick collision function address | Set breakpoint on driver brick-type memory, trace the function that writes it during pickup. | Phase 6 (pickup interception) |
| 7 | Brick spawn position list | Scan for known brick positions (can be estimated from track geometry). | Phase 6 (server-side brick tracking) |
| 8 | Race finish flag per driver | Scan for a flag that changes when a driver completes the race. | Phase 4 (finish detection) |
| 9 | Current placement value per driver | Scan for values 1-6 that change during position swaps. | Phase 4 (writing authoritative placement to HUD) |
| 10 | Direct track-load function | Trace the call chain from menu selection to actual track loading. Find the inner function that accepts track ID. | Phase 11 (circuit mode) |

All items above must be found for BOTH 1999 NoDRM and 2001 versions.

---

## Testing Strategy

### Phase 0-1 testing: Single machine
- Run two instances of LEGO Racers (may require renaming the exe or using sandboxing).
- Run server locally.
- Verify both clients connect, receive slot assignments, and write to different driver slots.

### Phase 2 testing: LAN
- Two machines on the same network.
- Measure round-trip time and verify interpolation smoothness.
- Intentionally introduce packet loss (e.g., `clumsy` tool on Windows) to test dead reckoning.

### Phase 3+ testing: Simulated internet
- Use traffic shaping to add 50-150ms latency and 1-5% packet loss.
- Verify race start sync, clock drift, and placement accuracy under realistic conditions.

### Version matrix
- 1999 NoDRM vs 1999 NoDRM: primary target.
- 2001 vs 2001: secondary target.
- Mixed: explicitly blocked in v1, tested when ready.

---

## Priority Order

```
Phase 0: Stabilization (bugs, protocol, threading)          — CRITICAL, do first
Phase 1: 6-player slot mapping + AI suppression              — CRITICAL, enables core goal
Phase 2: Transform smoothing                                 — HIGH, makes it playable
Phase 3: Lobby + synchronized race start                     — HIGH, makes it usable
Phase 5: Power-up inventory authority                        — HIGH, core gameplay
Phase 4: Checkpoint/lap/placement                            — MEDIUM, competitive play
Phase 6: Pickup conflict resolution                          — MEDIUM, fairness
Phase 7: Power-up effects and projectiles                    — MEDIUM, full gameplay
Phase 8: Kart configuration sync                             — LOW, cosmetic
Phase 9: Nameplates and overlay                              — LOW, polish
Phase 10: Mini-map verification                              — LOW, likely free
Phase 11: Circuit mode                                       — LOW, extension
```

### Minimum viable "real online race"

The project reaches its first meaningful milestone when:

- 6 racers appear correctly (local + 5 remote)
- Remote movement is smooth enough for racing
- Server starts the race at an agreed time
- Server owns lap, placement, and timer
- Power-up pickups are server-arbitrated
- Activated power-ups are applied consistently on all clients
- Disconnects don't corrupt slot ownership

That is Phases 0-5 plus the foundation of Phase 7. Everything after that is refinement toward the full end-state.
