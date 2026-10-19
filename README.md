# BattleBit Remastered — GameAssembly research notes

Notes from a private reverse-engineering pass on BattleBit's Unity IL2CPP `GameAssembly.dll`. This repo is documentation only. It does not include game binaries, metadata dumps, or a decompiled source corpus.

| | |
|--|--|
| Target studied (local) | `GameAssembly.dll` — Unity **2019.4.40f1** IL2CPP, PE ImageBase `0x180000000` |
| Not targeted | Easy Anti-Cheat / FACEIT anti-cheat |
| Publisher | OkiGames — [EULA](https://agreements.battlebit.cloud/eula.pdf) · [Steam SSA](https://store.steampowered.com/subscriber_agreement/english/) |
| Work completed | Full unique-RVA decompile (local), subsystem maps, deep pipeline writeups |
| Published here | This README (+ `NOTICE.md`, `LICENSE`) |

> Not affiliated with OkiGames or Valve. See [NOTICE.md](NOTICE.md).

## Table of contents

1. [Legal / IP](#legal--ip)
2. [What this is](#what-this-is)
3. [Method](#method)
4. [Coverage](#coverage)
5. [Evidence grades](#evidence-grades)
6. [Findings](#findings)
7. [Combat pipeline](#combat-pipeline)
8. [Gamemode lifecycle](#gamemode-lifecycle)
9. [Spawn / authority](#spawn--authority)
10. [Networking sketch](#networking-sketch)
11. [Vehicles / loadout / squad](#vehicles--loadout--squad)
12. [Key addresses](#key-addresses)
13. [Still unknown](#still-unknown)
14. [License](#license)

## Legal / IP

Steam SSA and the BattleBit EULA restrict reverse engineering, decompilation, and redistribution of game software/content (with limited "applicable law" carve-outs).

This public tree does not redistribute:

- `GameAssembly.dll`, `global-metadata.dat`, assets, maps, audio, or other install content
- Ghidra decompilation corpora
- Il2Cpp DummyDlls or full `script.json` dumps

Research artifacts stay local. The text below is a writeup of observations (addresses, call shapes, architecture). Identifier names and addresses are technical references from that private work.

EAC / anti-cheat reverse engineering is out of scope.

## What this is

A report on BattleBit's server-authoritative gameplay architecture inside `GameAssembly.dll`: combat routing, mode timers, spawn deny policy, and networking shape.

It is not reconstructed C#, a complete RPC opcode table, a cheat or AC-bypass guide, a dump of Unity/mscorlib, or a copy of the Steam install.

Most authored gameplay logic lives in `GameAssembly.dll`. Assets are data and were not the focus.

## Method

Private local workflow (not shipped here):

1. Il2CppDumper on a legitimate install's `GameAssembly.dll` + `global-metadata.dat` for a method name/signature index (~289k ScriptMethods).
2. Persistent Ghidra project on the DLL (ImageBase `0x180000000`).
3. Tiered mass decompile of unique non-framework, non-AC RVAs into a local corpus (~113k `.c` bodies).
4. Phase 4: full T1 bucket call-frequency maps per subsystem (combat, gamemode, spawn, squad, vehicle, loadout, network).
5. Phase 5: deep CFG / narrative writeups on combat, modes, spawn authority, and networking.

Many types are Unicode-obfuscated. Readable `$$Method` suffixes (`EnqueueKill`, `OnPlayerSpawn`, `ServerSide_*`) still show intent on important surfaces.

## Coverage

| Tier | Meaning | Count | Result |
|------|---------|------:|--------|
| T1 | Critical readable gameplay (keywords + core types) | 4,615 | 100% decompiled (local) |
| T2 | Other ASCII-readable game methods (delta) | 8,424 | 100% |
| T3 | Remaining unique non-framework / non-AC RVAs | 99,809 | 100% |
| Corpus | Unique game+SDK bodies on disk (local) | ~113,136 | complete |

Skipped by design: Unity/System framework (~84k methods) and AC-tagged methods (~847).

Many Il2Cpp names alias one RVA; unique addresses are about 113k versus ~289k ScriptMethod entries.

## Evidence grades

| Grade | Meaning |
|-------|---------|
| O | Observed directly in private decompile / metadata |
| I | Inferred from structure / naming / call patterns |
| U | Unknown / not established |

Claims below are tagged with these grades.

## Findings

1. **[O/I]** Server-authoritative monolith: spawn, damage policy, and many mode transitions are decided server-side. Clients get thinner `Received*` paths.
2. **[O]** The thin `ServerSide_ApplyDamage` helper only subtracts a float. Routing sits in a large orchestrator plus a target-resolution callee with a per-kind switch.
3. **[O]** HP math and kill reporting are separate. Kill enqueue builds a record and appends to a list; it does not set HP to 0.
4. **[O]** Modes often share a small timer: `endTime = duration + Time.time`. Special modes such as OneLife also iterate players.
5. **[O]** Spawn is a large policy function. Failures send numeric deny reason codes as network messages.
6. **[O/U]** Networking uses clear `Send*` / `Received*` / `*_serverSide` naming and shared send helpers. Full opcodes are still unknown **[U]**.
7. The mass decompile was raw material. The sections below are the distilled reading of it.

## Combat pipeline

### Flow **[O]**

```
damage context
    → FUN_180e81940          (infantry orchestrator, ~27KB)
        → FUN_181dfa5e0      (resolve / iterate hit targets, ~41KB)
            → switch(target kind)
                → optional FUN_1817984e0   (thin HP at this+0x30)
                → FUN_180fae540            (vehicle-style apply)
        → FUN_180b36490      (health notify / outbound message)

parallel: ExecuteDamage (FUN_181aaf4a0) → FUN_181c42500 (hitbox health)

kills: EnqueueKill* → FUN_182ff9c90 (growable list push)
```

### Step detail **[O]**

1. `FUN_180e81940` copies context locals, runs early validation and prep helpers (`FUN_181e02e90` / `FUN_1822d7d20`), then calls the target resolver.
2. `FUN_181dfa5e0` builds and iterates a hit target set (loop helper `FUN_1819686c0`), then `switch(targetKind)` with IL2CPP type checks. Infantry-like cases gate through helpers (`FUN_180fb1d90`, `FUN_182455950`, `FUN_182d770f0`), optionally thin HP apply, and vehicle-style apply.
3. Thin player apply `FUN_1817984e0` does `hp = max(0, hp - dmg)` at `this+0x30` only. It is not the combat brain.
4. Hitbox path: `ExecuteDamage` indexes a hitbox slot from `param_2`, then `FUN_181c42500(hitbox, health - damage, 0)`.
5. Vehicles: the orchestrator also reaches `FUN_180fae540`. Post-damage `FUN_180faf380` (~19KB) writes health (`+0x1e4`) and walks component lists for sync **[O/I]**.
6. Kill queue: `EnqueueKill` / `EnqueueAssistKill` / `EnqueueVehicleKill` allocate a record, fill attacker/victim/flags, push via `FUN_182ff9c90`, and bump a sequence counter (`+0x80`). That is bookkeeping, not HP math.
7. Noise: `FUN_1800af160` / `FUN_1800af130` are null/throw aborts everywhere. Ignore them as gameplay.

### Phase 4 combat map (T1 bucket, full 180/180) **[O]**

Hottest shared callees by frequency: `FUN_1800af160`, `FUN_1800af130`, `FUN_1808bb370`, `FUN_181710760`, `FUN_1822bb450`, `FUN_182e7cdb0`, `FUN_181c597d0`, `FUN_18101de30`, …

Largest T1 `Apply*` bodies are often settings helpers (`ApplyToSettings_*`, tens–100KB+), not the damage orchestrator (naming collision). Other large combat-sized surfaces include `EnterDeathMode`, `ExplosionSphere`, `KillAndSendToPool`, and `OnAPlayerKilledAnotherPlayer`.

### Open (combat) **[U]**

- Exact `switch` case labels (infantry vs vehicle vs prop / seat)
- Where `EnqueueKill` sits relative to HP reaching 0 (callers not fully traced)
- Full field layout of the damage context struct
- Il2Cpp name mapping onto `FUN_181dfa5e0` and `FUN_180b36490`

## Gamemode lifecycle

### Shared timer **[O]**

```
FUN_182efaad0:
  now = Time.time-like (FUN_1820e7960)
  *(float*)(mode + 0x14) = (float)*(int*)(mode + 0x10) + now
  // endTime = durationSeconds + now
```

Most simple modes' server start eventually call this (for example a nested timer at `mode+0x58` in OneLife).

### OneLife server start **[O]**

```
OnStartGameModeServerSide_OneLife (FUN_1810ae5c0)
  → FUN_182efaad0(timerObj)
  → iterate collection → per-slot setup (FUN_180ce7220)
  → dispose iterator
```

OneLife starts the timer and iterates actors. It is more than a timer wrapper.

### Mode taxonomy **[I]**

| Class | Start body shape |
|-------|------------------|
| Shared / objective-ish | Often thin → `FUN_182efaad0` only |
| OneLife | Timer + player iteration |
| Elimination | Extra helpers (`FUN_182c4d500` appears in samples) |
| GunGame FFA/Team | Distinct server start RVAs; small specialized bodies |
| Infected / VoxelFortify / TDM | Own `OnStart*` RVAs; client sides often stubs |

Round end: send/handle symbols exist (`SendEndOfRoundMessage`, match-lifecycle callbacks) **[O]**. The exact win-condition → end-of-round call graph was not fully walked **[U]**.

### Open (modes) **[U]**

- Who writes `mode+0x10` duration before timer start
- Per-mode win conditions → end-of-round graph
- Relationship between Elimination helpers and the kill queue

## Spawn / authority

### Model **[O/I]**

- Server decides whether a spawn proceeds (`OnPlayerSpawn` @ `0x180c3aed0`, ~65KB). This is a policy engine, not a teleport.
- Failures call `FUN_180ce6090` (SendDenySpawn) with immediate integer reason codes **[O]**.
- Clients have a smaller Received path (`0x18111d4e0`, ~5KB) for applying an allowed spawn **[O]**.

### Deny codes in `OnPlayerSpawn` **[O]**

| Code (hex) | Approx role |
|------------|-------------|
| `0xfa` | Early rejection gate |
| `0x2f5` | Secondary validation fail |
| `0x5cc` / `0x5f1` / `0x5fe` | Slot / loadout / state rejects **[I]** |
| `0x369` / `0x36a` | Paired related denies |
| `0x54f` | Terminal deny |

Exact string/enum labels need further enum recovery **[U]**.

### SendDenySpawn body **[O]**

```
FUN_180ce6090(ctx, reasonCode)
  → resolve net peer (FUN_181710760 …)
  → FUN_1822bd660(peer, reasonCode)
  → build/send packet (FUN_1808bb370 + FUN_1822baea0)
```

Deny goes out as a network message. It is not a silent local return.

### Open (spawn) **[U]**

- Deny code → player-facing string / enum map
- Which branches are rally / friend / vehicle spawn
- Field-by-field mirror of the success path in `OnPlayerSpawnReceived`

## Networking sketch

Pattern-level only. This is not a full opcode map **[U]**.

### Direction from names **[O]**

| Pattern | Role |
|---------|------|
| `Send*` / `*_server2Clients` | Server → clients |
| `Received*` / `OnRpcReceived` | Inbound handlers |
| `*_serverSide` | Server-authoritative action |
| `MessageToServer*` / `HandleMessageToClient*` | Bidirectional callbacks |

Phase 4 T1 network bucket: 516 readable methods scanned **[O]**.

### Shared plumbing **[O]**

Across Send/Received bodies, the same helpers dominate:

- `FUN_1800af160` / `FUN_1800af130` — null/throw aborts
- `FUN_1808bb370` — frequent in packet build (also SendDenySpawn)
- `FUN_181710760` — connection/peer resolve (arg often channel/type id)
- `FUN_1822b9ab0` / `FUN_1822b9ff0` / `FUN_1822bc4e0` / `FUN_1822bece0` — field writers on message objects

Inferred pattern **[I]**: resolve peer → alloc/fill message → shared send helpers.

### Concrete walked examples **[O]**

1. SendDenySpawn: reason → peer → send.
2. Health notify `FUN_180b36490`: builds a message, writes floats/flags via the `FUN_1822b*` family.
3. Kill enqueue: local list first; net fan-out is probably elsewhere **[U]**.

### Transport **[O]**

Steam Networking Sockets/Utils APIs appear in metadata names. FACEIT AC message callbacks exist and stay out of scope.

### Not claimed **[U]**

- Complete opcode ↔ handler table
- Wire format / serialization layout
- Anti-cheat channel semantics

### How an opcode map could be finished later (local)

1. List all `Send*` + `Received*` from metadata.
2. In each Send body, collect integer literals into peer-resolve / message constructors.
3. Cross-link Received handlers on the same ids.
4. Automate over a private decompile corpus (not shipped here).

## Vehicles / loadout / squad

- Vehicle post-damage `FUN_180faf380` is the rich sync path **[O]**. The T1 vehicle bucket covers get-in/out and health-changed surfaces **[O]**.
- Loadout: `SwitchLoadout` / `ReloadLoadout` showed up earlier **[O]**. The T1 loadout bucket is small (25 files) but complete in the local map **[O]**.
- Squad / team: Phase 4 maps catalog T1 `Spawn*` / `Squad*` / `Team*` surfaces **[O]**. Authority cues stay `Server_` vs `Received_` naming **[I]**.

## Key addresses

ImageBase `0x180000000`. RVA = VA − ImageBase.

| Role | VA |
|------|-----|
| Damage orchestrator | `0x180e81940` |
| Target set / iterate | `0x181dfa5e0` |
| Thin player ApplyDamage | `0x1817984e0` |
| ExecuteDamage (hitbox) | `0x181aaf4a0` |
| Hitbox health helper | `0x181c42500` |
| Vehicle apply | `0x180fae540` |
| Vehicle post-damage | `0x180faf380` |
| Health notify | `0x180b36490` |
| EnqueueKill | `0x1830832f0` |
| EnqueueAssistKill | `0x183082fb0` |
| Kill list push | `0x182ff9c90` |
| Mode timer start | `0x182efaad0` |
| OneLife server start | `0x1810ae5c0` |
| OnPlayerSpawn | `0x180c3aed0` |
| OnPlayerSpawnReceived | `0x18111d4e0` |
| SendDenySpawn | `0x180ce6090` |

## Still unknown **[U]**

- Exact combat orchestrator `switch` case semantics
- Spawn deny code → string/enum mapping
- Complete ClientRPC / message-id ↔ handler table
- Full field layouts of damage/spawn context structs
- Per-mode win condition → end-of-round call graphs

## License

Author-written documentation in this repository is offered under [LICENSE](LICENSE) (MIT), subject to [NOTICE.md](NOTICE.md).
