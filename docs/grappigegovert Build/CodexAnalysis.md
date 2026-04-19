# LR1Online Code Analysis

## Overall

`LR1Online` is a dedicated-server relay prototype built around live process memory editing, not a finished multiplayer simulation. The current code can identify and attach to `LEGORacers.exe`, read and write racer state, send local racer transforms over UDP, and tell clients which race to load. It does not currently implement a full 6-player authoritative race with synchronized pickups, projectiles, timing, standings, kart cosmetics, or circuit progression.

The key architectural point is that it is not using the game's built-in split-screen or LAN stack. It hijacks the normal single-player race state by reading the local driver from memory and overwriting opponent driver slots in memory. The relevant hook points are in [MemoryManager.cs](./LEGORacersAPI/MemoryManager.cs), [GameClient.cs](./LEGORacersAPI/GameClient.cs), [Driver.cs](./LEGORacersAPI/Driver.cs), and the version maps in [Client_1999NoDRM.cs](./LEGORacersAPI/Client_1999NoDRM.cs) and [Client_2001.cs](./LEGORacersAPI/Client_2001.cs).

## What It Actually Does

- The launcher starts `LEGORacers.exe`, then the API identifies the build by MD5 and selects hard-coded addresses for 1999 NoDRM or 2001. The 1999 DRM hash is explicitly mapped to the wrong client with a comment saying it will crash, so 1999 DRM is not really supported right now: [LauncherForm.cs](./Client/LauncherForm.cs), [WizardForm.cs](./Client/WizardForm.cs), [GameClientFactory.cs](./LEGORacersAPI/GameClientFactory.cs).
- The process hook is straightforward Win32 memory access: `OpenProcess(PROCESS_ALL_ACCESS)`, `ReadProcessMemory`, `WriteProcessMemory`, `VirtualAllocEx`, `CreateRemoteThread`. That is how it ties into the executable: [MemoryManager.cs](./LEGORacersAPI/MemoryManager.cs).
- `GameClient.Initialize()` creates six `Driver` wrappers, which clearly matches Lego Racers' local player plus five opponents model: [GameClient.cs](./LEGORacersAPI/GameClient.cs).
- The server is a separate executable on TCP `3031` and UDP `3030`, so the intended topology is server-based, not peer-to-peer: [Server.cs](./Server/Server.cs).
- TCP is used for connection and disconnection, join notifications, race selection, and nominal power-up messages. UDP is used for racer transform snapshots: [Server.cs](./Server/Server.cs), [Client.cs](./Client/Client.cs).
- Each client sends its own local driver's `X/Y/Z`, `SpeedX/Y/Z`, forward vector, and up vector every 10 ms: [ClientForm.cs](./Client/ClientForm.cs).
- The server stores that snapshot on the matching `Participant` and returns a full participant snapshot back to the sender. The snapshot model only contains nickname, transform, speed, orientation, and nominal power-up fields: [Server.cs](./Server/Server.cs), [Participant.cs](./Library/Participant.cs).
- On receive, the client updates the `Participant` objects for everyone, but then only writes the first remote player into `gameClient.drivers[1]` and immediately breaks. So the current playable injection path is effectively one visible remote racer only: [ClientForm.cs](./Client/ClientForm.cs).

## Player Count And Race Model

- The codebase intent is clearly up to 6 total racers.
- Evidence:
- `GameClient` allocates `Driver[6]`: [GameClient.cs](./LEGORacersAPI/GameClient.cs)
- `Driver.UsePowerUp()` has slot-specific logic for indices `0..5`: [Driver.cs](./LEGORacersAPI/Driver.cs)
- `powerUpsUsed` is sized for five opponents in client code: [ClientForm.cs](./Client/ClientForm.cs)
- The implemented behavior does not reach that intent.
- Server admission has an off-by-one. The accepted TCP client is added before the `participants.Count() >= 6` check, so the sixth joiner gets rejected and the live maximum is effectively 5 connected clients, not 6: [Server.cs](./Server/Server.cs).
- Even if that were fixed, the client currently maps only one remote racer into opponent slot 1: [ClientForm.cs](./Client/ClientForm.cs).
- This is not using the game's split-screen second-player path. It is overwriting AI and opponent slots in the normal race memory model. So the "one other racer" behavior is not "online split-screen"; it is "first remote player shoved into opponent slot 1."

## Power-Ups, Pickups, And Conflicts

- There is a usable low-level concept for power-up detection. `Driver.CheckPowerUp()` compares previous and current brick state and fires `PowerUpUsed` when a driver's brick drops to `None`: [Driver.cs](./LEGORacersAPI/Driver.cs).
- There is also a usable low-level injector for invoking a power-up on a given racer slot with a remote thread: [Driver.cs](./LEGORacersAPI/Driver.cs).
- But the feature is not actually wired end-to-end.
- The client subscribes to `gameClient.Initialized`, but `GameClient.Initialize()` directly sets `InitializedType = Both` and never calls `OnInitialized()`. So `drivers[0].PowerUpUsed += Player_PowerUpUsed` is never actually attached: [ClientForm.cs](./Client/ClientForm.cs), [GameClient.cs](./LEGORacersAPI/GameClient.cs).
- Even if a power-up packet arrives, the client only shows a message box. It does not apply the remote effect in-game: [ClientForm.cs](./Client/ClientForm.cs).
- The actual remote `UsePowerUp()` calls are commented out: [ClientForm.cs](./Client/ClientForm.cs).
- The server never updates `Participant.PowerUpType` or `PowerUpWhiteBricks`, so those fields in the UDP snapshot are dead data at present: [Server.cs](./Server/Server.cs), [Participant.cs](./Library/Participant.cs).
- Near-simultaneous pickup conflicts are not handled at all. There is no notion of pickup identity, spawn ownership, server timestamps, authoritative winner selection, or rollback. Right now, each client would locally decide pickup contact on its own simulation. For internet play, that guarantees divergence.
- The "freeze power-up / freeze white bricks" server controls are mostly administrative stubs. They serialize into UDP settings, but the client only deserializes them and never applies them back into the game state: [Settings.cs](./Library/Settings.cs), [ClientForm.cs](./Client/ClientForm.cs).

## What Is Missing For The End Goal

- `Local + 5 remote friends`: intended by data model, not implemented in the client injection path.
- Host architecture: the dedicated server model is the correct direction, but the current server is only a relay plus minimal admin UI. It is not authoritative for race state: [ServerForm.cs](./Server/ServerForm.cs).
- Positioning under latency: predictive handling is needed. Current behavior is direct memory overwrite of raw snapshots with no tick number, no interpolation, no dead reckoning, and no correction strategy. Because speed and orientation are already transmitted, dead reckoning is the obvious next step.
- Race clock and synchronized start: not implemented. The server can only tell clients which track to load; `SetupRace()` stops at the racer-selection flow, not a synchronized start signal: [ServerForm.cs](./Server/ServerForm.cs), [ClientForm.cs](./Client/ClientForm.cs), [GameClient.cs](./LEGORacersAPI/GameClient.cs).
- Player names over karts: not implemented. There is no overlay, 3D text, or render hook. Only some commented debug `DrawString` remnants exist: [ClientForm.cs](./Client/ClientForm.cs).
- Kart configuration sync: not implemented. No car body, driver, license, build, or cosmetic fields are replicated anywhere in `Participant`.
- Mini-map: there is no explicit mini-map code. If a remote is written into a real opponent slot, the vanilla UI might partially reflect that slot, but only for whichever slots are actually driven by network data.
- Place, order, laps, checkpoints, and circuit scoring: no code for them. Circuit support currently means only "server picks a track from a list." There is no championship state machine or scoring system: [Circuit.cs](./LEGORacersAPI/Circuit.cs), [NewRaceForm.cs](./Server/NewRaceForm.cs).
- Projectiles, placed 3D objects, sabotage effects: not implemented beyond the idea of sending a power-up-use message. There is no replicated object or effect lifecycle.

## Version-Specific And Other Gotchas

- 1999 NoDRM and 2001 have separate memory maps. That part is explicit and workable.
- 1999 DRM is knowingly unsupported and marked as crash-prone in the factory: [GameClientFactory.cs](./LEGORacersAPI/GameClientFactory.cs).
- There is no code accounting for 1999 vs 2001 behavioral differences like brick pickup radius or brick poisoning. Only address differences are modeled. If those mechanics diverge across versions, online state will drift even if transforms sync.
- `Driver.UsePowerUp()` has a likely bug: the `Blue` case calls `POWERUP_RED_ADDRESS` instead of `POWERUP_BLUE_ADDRESS`: [Driver.cs](./LEGORacersAPI/Driver.cs).
- White-brick writes are capped to `0..3` in the driver wrapper, but library settings allow `4`, which is inconsistent: [Driver.cs](./LEGORacersAPI/Driver.cs), [Settings.cs](./Library/Settings.cs).
- TCP has no framing protocol and uses a 64-byte read buffer, so packet boundaries are fragile: [Client.cs](./Client/Client.cs), [Server.cs](./Server/Server.cs).
- Numeric serialization is culture-sensitive because it uses plain string concatenation and `float.Parse` with current locale. Mixed decimal separators across machines would break packets.
- Disconnect cleanup is incomplete. `LastActivity` exists, but timeout cleanup is commented out, so ghost participants are possible: [ServerParticipant.cs](./Server/ServerParticipant.cs), [ServerForm.cs](./Server/ServerForm.cs).

## Detailed Architectural Notes

### How It Ties Into The Executable

The integration point is the `LEGORacersAPI` assembly.

- `GameClientFactory` identifies the running executable by MD5 and selects a version-specific client wrapper: [GameClientFactory.cs](./LEGORacersAPI/GameClientFactory.cs).
- `Client_1999NoDRM` and `Client_2001` define absolute addresses and pointer offsets for menu navigation, in-race structures, driver state, pause and unpause functions, AI count, power-up functions, and other controls: [Client_1999NoDRM.cs](./LEGORacersAPI/Client_1999NoDRM.cs), [Client_2001.cs](./LEGORacersAPI/Client_2001.cs).
- `MemoryManager` opens the process with full access, allocates executable memory inside the game process, writes custom machine code, and starts that code with `CreateRemoteThread`: [MemoryManager.cs](./LEGORacersAPI/MemoryManager.cs).
- `GameClient` and `Driver` use those facilities to:
- read and write racer transform and brick state
- patch instruction bytes for features such as `RunInBackground` and `LoadRRB`
- inject call stubs to invoke internal game functions such as menu transitions and power-up activation

This means the project is not modifying game files on disk during runtime. It is mutating live memory and calling internal game code.

### How Position Updates Work

Current data flow:

1. Local client samples `drivers[0]` from live memory every 10 ms.
2. It sends one UDP packet with nickname, transform, speed, and orientation.
3. Server stores that snapshot on the matching `ServerParticipant`.
4. Server sends the whole participant list back to the sender.
5. Client updates local `Participant` records from that payload.
6. Client writes the first remote participant into `drivers[1]`.

Important limitations:

- The server sends the snapshot only back to the UDP sender for that received packet. In practice each client gets updates as long as it keeps sending, but this is still a request-response style relay rather than a true server broadcast loop.
- There is no tick number, no sequence number, no time base, and no lag compensation.
- There is no interpolation buffer. Remote positions are hard-written directly into game memory.
- There is no reconciliation against the game's own simulation.

For a real online race, predictive code is almost certainly needed. The minimal next step would be:

- server-side sequence numbers
- client-side interpolation buffer for remote racers
- dead reckoning from transmitted velocity and orientation
- snap thresholds when positional drift exceeds a tolerance

### How Power-Up Pickup And Usage Should Work Versus What Exists

What exists:

- Brick state can be observed in memory.
- A local "used power-up" event can theoretically be derived.
- Remote power-up calls can theoretically be injected for driver slots.

What does not exist:

- authoritative pickup ownership
- authoritative white-brick count
- per-pickup entity ids
- projectile ids
- placed-object ids
- status effect duration sync
- server arbitration for same-time pickups
- client rollback or correction when pickup outcome differs from local prediction

For the end goal, the server should own pickup resolution. A workable model would be:

1. Each pickup pad or brick spawn gets a stable id.
2. Clients report attempted pickup with local tick and position.
3. Server chooses the winner based on authoritative timing and distance policy.
4. Server confirms winner and denies losers.
5. Clients update brick inventory to server state, including rollback if necessary.

Without that, 1999-era gameplay mechanics like near-simultaneous item contact will desync immediately online.

### Circuit Support And Lobbying

What exists:

- Server UI can select a circuit from a fixed list and send block plus race numbers to all clients: [ServerForm.cs](./Server/ServerForm.cs), [NewRaceForm.cs](./Server/NewRaceForm.cs), [Circuit.cs](./LEGORacersAPI/Circuit.cs).
- Client can receive the packet and call `SetupRace()` to navigate menus and choose that race: [ClientForm.cs](./Client/ClientForm.cs), [GameClient.cs](./LEGORacersAPI/GameClient.cs).

What does not exist:

- ready state
- car selection lock-in
- synchronized countdown
- host migration
- lobby chat
- race result screen coordination
- circuit standings and score accumulation

So "lobby hosting and joining" currently means only:

- server process running
- client enters IP and nickname
- server accepts or rejects connection
- server can tell clients which race to load

That is far short of a modern lobby system.

## Recommended Direction

If the goal is truly `1 local player + 5 remote players`, the best architectural direction is:

- keep the dedicated server architecture
- make the server authoritative over race-state decisions
- keep clients responsible for rendering and local input only
- map remote racers onto the five opponent slots
- replicate full race-state, not just transforms

Recommended implementation phases:

1. Fix the current foundation
- correct the join limit off-by-one
- actually map remote racers to `drivers[1]` through `drivers[5]`
- fix event wiring so power-up detection works
- fix the blue-power-up address bug
- add reliable packet framing and invariant culture float serialization

2. Add deterministic race synchronization
- assign server tick or sequence ids
- synchronize race start from server
- replicate checkpoint, lap, and finish progress
- make standings server-authoritative

3. Add inventory and power-up authority
- replicate current brick inventory and white bricks
- assign stable ids to pickups, projectiles, and placed hazards
- resolve pickup conflicts on the server
- broadcast effects and projectile lifecycle

4. Add polish and compatibility layers
- kart configuration sync
- nameplates or overhead labels
- mini-map synchronization
- 1999 versus 2001 compatibility rules for pickup radius, poisoning, and any other mechanics that differ by executable

## Bottom Line

The project is aimed at `1 local + up to 5 remote` by replacing the five opponent slots with network players, using a dedicated relay server. The current implementation only reaches a small subset of that: attach to the game, load a race, stream one racer's transform, and write one remote racer into one opponent slot. Everything that makes internet racing coherent rather than merely visible still needs authority, timing, reconciliation, and object and effect replication.

## Verification Notes

- Analysis was based on static inspection of the source files in this repository.
- I attempted a local build check, but `msbuild` is not installed in the current shell environment, so I could not verify compilation from the command line.
