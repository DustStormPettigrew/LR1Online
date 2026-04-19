# LR1Online — Detailed Architecture Analysis

## 1. How the Project Ties Into the Executable

LR1Online uses **external process memory manipulation** — it does NOT inject a DLL. The `MemoryManager` class (`LEGORacersAPI\MemoryManager.cs`) calls Win32 APIs directly:

| API Call | Purpose |
|---|---|
| `OpenProcess(PROCESS_ALL_ACCESS)` | Gets full read/write/execute handle to `LEGORacers.exe` |
| `ReadProcessMemory` / `WriteProcessMemory` | Reads/writes game state (positions, bricks, menu state) |
| `VirtualAllocEx` | Allocates 1024 bytes of RWX memory inside the game process |
| `CreateRemoteThread` | Executes injected x86 machine code in the game process |

**Game version identification** is done by MD5-hashing `LEGORacers.exe`:

| MD5 Hash | Version |
|---|---|
| `80c9577841476a26ed76749b8e4b4a9f` / `8c4bb866a9b5d313584831834aec8158` | 1999 NoDRM |
| `a0007cfe64097651e5505fba15ab5cc1` | 1999 DRM (noted in code as "this will crash") |
| `325cbbedc9d745107bca4a8654fce4db` | 2001 |

Two concrete subclasses — `Client_1999NoDRM` and `Client_2001` — provide version-specific memory addresses for every feature: driver positions, menus, powerups, pause state, AI count, etc. The offsets differ substantially (e.g., driver X coordinate is offset `0x518` in 1999 vs `0x528` in 2001).

The client app (`LauncherForm`) **launches** LEGORacers.exe itself, then passes the `Process` handle to `ClientForm`, which attaches via `MemoryManager.Initialize()`. Admin rights are required.

---

## 2. Player Count: 1v1, Not 6-Player

**The game engine supports 6 driver slots** (index 0 = local player, indices 1-5 = AI/opponents). `GameClient.Initialize()` creates all 6:

```csharp
this.drivers = new Driver[6];
for (int i = 0; i < 6; i++)
    this.drivers[i] = new Driver(this, i);
```

**However, the current client code is hardcoded for 1v1.** In `ClientForm.client_PacketReceived()` (lines 464-491), when processing UDP coordinate responses from the server, the code iterates remote players but writes ONLY to `drivers[1]` and then immediately `break;`:

```csharp
foreach (Participant enemy in participants.Where(p => p.Nickname != participant.Nickname))
{
    gameClient.drivers[1].X = enemy.X;
    gameClient.drivers[1].Y = enemy.Y;
    gameClient.drivers[1].Z = enemy.Z;
    // ... all 12 fields written to drivers[1] only ...
    break;  // <-- only first remote player processed
}
```

The **server** does cap at 6 participants (`participants.Count() >= 6` check in `Server.cs:137`), and the `Participant` data model supports N players. So the networking layer is architecturally ready for 6, but the game-side memory writing is strictly 1v1.

To support 5 remote racers, the `break` must be removed and a counter must map each remote participant to `drivers[1]` through `drivers[5]`.

---

## 3. Position Transmission & Ingestion

### Network Protocol

The project uses a **dual-protocol design**:

| Protocol | Port | Purpose |
|---|---|---|
| **TCP** (port 3031) | Reliable | Connect, Disconnect, Join, PowerUp, Race selection |
| **UDP** (port 3030) | Fast, lossy | Coordinate updates (position, speed, orientation) |

### The Update Loop

1. **Client to Server** (`ClientForm.UpdateClient()`, ~10ms loop):
   - Reads from game memory: X/Y/Z position, X/Y/Z speed, forward vector (3 floats), up vector (3 floats) — 12 floats total for `drivers[0]`
   - Sends UDP: `"Coordinates|nickname|X|Y|Z|SpeedX|SpeedY|SpeedZ|FwdX|FwdY|FwdZ|UpX|UpY|UpZ"`

2. **Server** (`Server.UdpListener()`):
   - Receives packet, stores 12 floats on the sender's `ServerParticipant`
   - Builds response containing ALL players' data: `"Coordinates|settings_string|nick1;X;Y;Z;SpeedX;SpeedY;SpeedZ;FwdX;FwdY;FwdZ;UpX;UpY;UpZ;PowerUpType;WhiteBricks|nick2;..."`
   - Sends response back to the sender via UDP

3. **Client from Server** (`ClientForm.client_PacketReceived()` for UDP):
   - Parses each player's data from the delimited string
   - Writes remote player positions directly into game memory via `Driver` property setters (which call `MemoryManager.WriteFloat`)

### What Position Data is Synced

Per driver, 12 floats are synced:
- **Position**: X, Y, Z
- **Velocity**: SpeedX, SpeedY, SpeedZ
- **Orientation**: Forward vector (X,Y,Z) and Up vector (X,Y,Z)

These are written directly to the corresponding offsets in the game's driver structure (e.g., for 1999: coordinate X at `baseAddress + 0x518`, speed X at `baseAddress + 0x3F0`, etc.).

### Predictive Code / Interpolation

**There is none.** Positions are written raw as received. Speed values ARE transmitted (which is good — the game engine uses them for physics), but there is no:
- Client-side interpolation between updates
- Dead reckoning / prediction during network gaps
- Timestamp-based extrapolation
- Jitter buffer

At 10ms client loop + network latency, remote karts will exhibit jittery, discrete teleporting movement rather than smooth motion.

---

## 4. Power-Up Pickup Tracking & Conflicts

### Detection Mechanism

The `Driver.CheckPowerUp()` method (called in a polling loop at the `RefreshRate` interval, default 10ms) compares the current brick type and white brick count against the last known values:

- **Brick type changed from X to None**: fires `PowerUpUsed` event (player activated their power-up)
- **Brick type changed from X to Y**: fires `PickedUpBrick` event (player picked up a new brick)
- **White bricks increased**: fires `PickedUpBrick` with `Brick.White`

### Transmission

When the local player's `PowerUpUsed` event fires, the client sends a TCP packet:
```
"PowerUp|brickType|whiteBricks"
```
The server relays this to all other connected clients (broadcast, excluding sender).

### Remote Power-Up Activation

`Driver.UsePowerUp()` injects x86 assembly into the game process to call the native power-up functions. Each brick color has its own function address, and each driver index (0-5) has different register setups:

```csharp
case 0: // Local player
    ebx = ecx - 0x498;
    edx = ecx - 0x444;
    break;
case 1: // Opponent 1
    edx = 0xD1;
    break;
// ... cases 2-5 ...
```

**Bug**: In `Driver.UsePowerUp()` line 312, `Brick.Blue` calls `POWERUP_RED_ADDRESS` instead of `POWERUP_BLUE_ADDRESS`.

### Conflict Resolution

**There is NONE.** The server blindly relays `PowerUp` packets. There is no:
- Server-side tracking of which bricks exist on the track
- Arbitration for near-simultaneous pickups
- Authoritative brick state on the server
- Rollback mechanism

Each client's game engine independently manages brick spawning and collision detection. If two clients pick up the "same" brick at nearly the same time, both will have it locally — the server has no mechanism to resolve this. This is a fundamental architectural gap for the 6-player goal.

### Server Power-Up Freeze Feature

The server admin can "freeze" power-ups and white bricks to specific values via the Settings menu. This 5-character settings string is piggy-backed on every UDP coordinate response. Clients deserialize it in `Library.Settings.Unserialize()`. However, **the client never actually enforces these frozen values** — the code to read the settings exists but no code writes them into game memory.

---

## 5. Architecture: Server-Based (Dedicated)

The architecture is **dedicated server** (server does NOT run the game):

- **Server app** (`Server` project): Standalone WinForms application. Manages connections, relays packets, provides admin controls (kick, race selection, power-up freeze). Does NOT run LEGO Racers.
- **Client app** (`Client` project): Launches LEGO Racers, attaches to its memory, connects to the server, sends/receives game state.

The server is authoritative for:
- Connection management (join, disconnect, kick, full-server rejection)
- Race selection (server admin picks the circuit, broadcasts to all clients)
- Nickname uniqueness
- Settings/freeze state

The server is NOT authoritative for:
- Game physics or positions (just relays what clients report)
- Power-up state (no server-side game state)
- Race timing or progress
- Anything requiring game knowledge — it's a relay

This is essentially a **peer-to-peer architecture with a relay server** for NAT traversal and coordination, rather than a true authoritative server.

---

## 6. Game Tick Alignment

**There is no game tick synchronization.** The 10ms `UpdateClient()` loop runs on wall-clock time, independent of the game's internal frame rate or tick system. The game's rendering and physics loops run independently on each client.

Position writes happen whenever the network delivers data, which is uncorrelated with the game's rendering frames. This means:
- A position write might happen mid-frame, causing visual tearing
- Two clients may be at different points in their physics step when exchanging data
- There's no concept of a shared "game tick" number

---

## 7. Player Names Overhead

**Not implemented.** The game has no native name rendering system for racers. There is no DirectX overlay, no hook into the rendering pipeline, and no mechanism to draw text above kart models. A commented-out block references `g.DrawString()` suggesting an earlier attempt at a GDI overlay, but it's not functional.

---

## 8. Online Player's Kart Configuration

**Not implemented.** There is no mechanism to:
- Read the local player's kart build configuration from memory
- Transmit it to the server
- Apply it to remote driver slots in the game

Remote players will appear as whatever model the game loads for that driver slot (likely the default AI racer).

---

## 9. Mini-Map

**No code addresses the mini-map directly.** However, the mini-map in the original game renders driver positions from the same in-memory driver structures that LR1Online writes to. So the mini-map *might* update correctly as a side effect of position writes — but this is untested and depends on whether the game re-reads position data during mini-map rendering vs caching it.

---

## 10. Race Placement & Clock Sync

**Not implemented.** There is:
- No race timer synchronization
- No shared start countdown
- No lap/checkpoint tracking
- No race position calculation
- No finish detection or results sharing

The `Race` packet type exists and the server can tell all clients to start a specific circuit (`"Race|blockNumber|raceNumber"`), which triggers `GameClient.SetupRace()` — this navigates the menu system via injected assembly to load the race. But there's no coordinated countdown or start signal. Each client's race begins whenever the game finishes loading, which varies by machine speed.

---

## 11. Activated Power-Up Display & Sync

**Partially implemented but incomplete.** The power-up usage event is detected and transmitted via TCP. `Driver.UsePowerUp()` can invoke the game's native power-up functions for any driver slot. However:
- The receiving client currently just shows a `MessageBox.Show()` with the packet content instead of actually activating the power-up
- Projectile trajectories, placed 3D models, and visual effects are not synced
- The native power-up activation would fire the effect from the AI driver's current position/orientation, which might be correct if positions are synced tightly enough

---

## 12. Circuit Support & Scoring

**Partial.** The `Circuit` class defines all 13 circuits (4 per block across 3 blocks + Rocket Racer Run), including mirrored versions. The server can broadcast a circuit selection. But:
- No multi-race circuit progression
- No scoring/points between races
- No tracking of race completion or results
- Circuit mode would require tracking which race in the circuit each player is on

---

## 13. Lobby

**Minimal.** The current "lobby" is:
- `ConnectForm`: Manual IP address + nickname entry
- Server shows a `DataGridView` of connected nicknames
- Server admin can kick players via right-click context menu
- No room browser, no matchmaking, no player list refresh

---

## 14. 1999 vs 2001 Version Differences

### What IS accounted for:
- Completely separate memory address tables for all features (every single address differs)
- Version detection via MD5 hash
- Settings UI to force a specific version
- 2001 PauseGame has an extra `push 00` instruction before calling the pause function

### What is NOT accounted for:

**Brick pickup radius difference**: No code addresses this. If the pickup radius differs between versions, two players on different versions racing together would disagree about whether a brick was picked up. Since there's no server-side brick state, this would cause silent desync.

**Brick poisoning**: Not addressed at all. The "brick poisoning" mechanic (where picking up a brick of a different color replaces your current one) — if this behaves differently between versions, it would cause power-up state desync between mixed-version clients.

**Cross-version play**: **There is no enforcement that all clients run the same game version.** The server has no knowledge of client versions. Two clients on different versions could connect to the same server and race together, but since all game mechanics are local, any behavioral differences between versions would cause silent state divergence.

---

## 15. Critical Edge Cases & Gotchas

### Race Conditions & Threading

1. **Participant list is not thread-safe**: `Server.cs` modifies `participants` from the TCP listener thread, UDP listener thread, and UI thread without any locking. This will cause `InvalidOperationException` or data corruption under load.

2. **UDP reply routing is broken for >2 players**: The server's `UdpListener()` overwrites `ipEndPoint` on every `Receive()` call, then sends the response to that endpoint. With multiple clients sending UDP simultaneously, only the **last sender** gets the coordinate response. This is a fundamental bug that prevents >2 player operation.

3. **Shared code injection buffer**: All injected code writes to `MemoryManager.NewMemory` (a single 1024-byte allocation). If `CheckPowerUp()` triggers `UsePowerUp()` while `GotoMenu()` is writing assembly to the same buffer, the injected code will be corrupted, likely crashing the game.

### Network Issues

4. **No TCP message framing**: TCP is a stream protocol, but the client reads into a fixed 64-byte buffer and assumes each read is exactly one complete packet. Large packets (like a coordinate response with 6 players' data) will be truncated. Back-to-back small packets may be coalesced into a single read.

5. **Float locale sensitivity**: `float.Parse()` is used without `CultureInfo.InvariantCulture` throughout `ClientForm.cs` for parsing network data. Any client in a locale that uses `,` as a decimal separator (most of Europe) will corrupt ALL position data, producing wildly wrong coordinates or parse exceptions.

6. **No keepalive/timeout**: The cleanup timer in `ServerForm` is commented out. If a client crashes without sending a Disconnect packet, their `ServerParticipant` persists forever, and their stale position data keeps being broadcast to all other clients.

7. **UDP receive timeout**: The client sets a 20-second UDP receive timeout. If no UDP data arrives for 20s, the receive throws a `SocketException` which is caught and the loop continues — but there's no reconnection logic or user notification.

### Game Integration Issues

8. **AI fights network writes**: Writing remote player positions to AI driver slots conflicts with the game's AI movement system. Even with `LoadRRB = false` (disables AI path loading), the AI movement code may still execute and overwrite the network-injected positions every frame, causing flickering between AI-computed and network positions.

9. **Game AI count mismatch**: The `AIDriversAmount` property controls how many AI drivers the game spawns. But there's no code that sets this to match the number of online players. If 3 players connect, the game still spawns its default number of AI drivers, and only slot 1 gets overwritten.

10. **Menu navigation timing**: `SetupRace()` uses hardcoded `Thread.Sleep(30)` delays between menu transitions. If the game takes longer to transition (e.g., loading assets), the menu navigation will fail silently.

11. **No game state validation**: The client sends coordinates even when `IsRaceRunning` might be false. The server has no concept of game state, so position data flows during menus, loading screens, etc.

### Missing Features for 6-Player Goal

12. **No brick spawn management**: The game spawns power-up bricks independently on each client. With 6 players, two clients might see a brick at the same location at the same time and both pick it up. There's no server-side brick inventory.

13. **No collision**: The game's collision detection for AI drivers is physics-based on the local simulation. Network-driven position overwriting bypasses the physics system entirely. Remote players will drive through walls, ignore obstacles, and not interact with track geometry correctly.

14. **No finish line detection**: There's no mechanism to know when a remote player crosses the finish line, so race results would only reflect the local player vs the game's internal AI state.

15. **Power-up effects not synced bidirectionally**: If a remote player fires a red brick projectile, the local client would need to spawn that projectile at the correct position with the correct trajectory. The current code only transmits "player X used brick type Y with Z white bricks" — not the position, aim direction, or timing needed to correctly reproduce the visual effect and hit detection.

---

## Summary: Current State vs End-Goal

| Feature | End-Goal | Current State |
|---|---|---|
| Player count | 1 local + 5 online | 1 local + 1 online (hardcoded `break`) |
| Position sync | Smooth, tick-aligned | Raw writes, no interpolation |
| Power-up conflicts | Server-authoritative | No conflict resolution |
| Race start sync | Countdown from host | No sync (load-time dependent) |
| Race clock | Synced from host | Not implemented |
| Placement tracking | Accurate positions | Not implemented |
| Mini-map | Updated for all players | Possibly works as side-effect |
| Player names overhead | Visible near kart | Not implemented |
| Kart configuration | Synced across clients | Not implemented |
| Power-up visuals | Full sync (projectiles, effects) | Detection + relay only, no visual reproduction |
| Circuit mode | Scoring across races | Circuit selection only |
| Lobby | Browse, host, join | Manual IP entry only |
| Version enforcement | Same version required | No enforcement |
| Brick pickup radius | Version-aware | Not addressed |
| Brick poisoning | Version-aware | Not addressed |

The project represents a solid proof-of-concept that successfully demonstrates external memory manipulation of the running game, reading/writing driver state, navigating menus via code injection, and basic client-server networking. The foundation for 6-player racing exists in the data model and driver array, but significant work remains in every category to reach the stated end-goal.
