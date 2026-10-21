# Resident Evil Revelations - working notes

Session date: 2026-10-22. Phases covered: 0 (triage), 1 (research), 2 (workspace),
3 (understand), 4 (choose), 5 (build, first slice). The owner asked for a from-zero mod.

Nothing in the Steam install was modified. All work happens on a copy of the
game in a test folder (`<test copy>`; the path is in `local/`).

Sections below are in the order they were written. The findings of the second
part of the session (the mode field, the loaders, the save slots, the input
path and the first build of the mod) are in
[Second session: the mode field, the loaders and the first build](#second-session-the-mode-field-the-loaders-and-the-first-build)
at the end of the file. The findings of the third part (the two game-system
instances, the netcode that already ships with the game, and the second slice
of the mod) are in
[Third session: the game already ships a netcode](#third-session-the-game-already-ships-a-netcode)
further down.

## Game identity (recorded before following any write-up)

| | |
|---|---|
| Game | Resident Evil Revelations (PC/Steam, the 3DS original re-made) |
| Steam appid | 222480 |
| build id | 13298487 |
| Executable | `rerev.exe`, 14,260,576 bytes, SHA-256 `1573B79EB921B1E7F571E6DEB71C39961C7F1940FAD6D19283FC3AEA7FCA48AA` |
| Engine | MT Framework (the build tree calls itself `KATARIBE`, project code name `bhr\biorevhd`) |
| Architecture | x86, 32-bit, image base 0x400000 |
| Sections | `.text` 12,301,312 bytes (entropy 6.41, normal code), `.rdata`, `.data`, `.rsrc`, `.bind` 170,712 bytes (entropy 7.97, SteamStub) |
| Entry point | RVA 0xdf0310, inside `.bind` (SteamStub loader) |
| Owner install | `<game dir>` (read only; the path is in `local/`) |
| Test copy | `<test copy>` (the path is in `local/`) |
| Steam tags | Single-player, Multi-player, Co-op, full controller support, Valve Anti-Cheat enabled. No "Online Co-op" and no "Shared/Split Screen Co-op" tag. |

The `.text` section has normal code entropy, so the game code is statically
analysable; `.bind` is only the SteamStub wrapper.

## Tooling on this machine (each item checked, not assumed)

Paths are in `local/`; the versions and where each tool had to be found from
are what matter here.

| Tool | Version / path | Note |
|---|---|---|
| Python | 3.13.15, `%LOCALAPPDATA%\Programs\Python\Python313\python.exe` | not on PATH, `python` hits the Store stub |
| PyYAML | 6.0.2 | so `tools/build_directory.py` really runs |
| git | 2.55.0, `%ProgramFiles%\Git\cmd\git.exe` | not on PATH |
| JDK | Temurin 21.0.12.1, `%ProgramFiles%\Eclipse Adoptium\jdk-21.0.12.101-hotspot` | needed by Ghidra |
| Ghidra | 12.1.4 PUBLIC 20260921, under the workspace folder | installed this session on owner approval; SHA-256 `ddac49f9…87d2d4db` matched the hash published on the release page |
| MSVC | Visual Studio Build Tools 2022 17.14.42, VCTools workload, `vcvars32.bat` | installed this session on owner approval |
| GitHub CLI | **not installed** | checked `%ProgramFiles%\GitHub CLI`, no other copy found; needed before a PR can be opened |

Correction to the previous session's notes (they live in
`games/resident-evil-revelations-2/NOTES.md`): that file stated that GitHub CLI
2.102.1 and Ghidra 12.1.4 were already installed on this machine. Neither was
present when checked; both were installed only after the owner approved them
during this session.

## Phase 3: the shipped PC build already contains a campaign co-op layer

This is the central finding, and it changes the shape of the mod: the game was
built with a two-player campaign co-op implementation compiled in. The shipped
UI does not expose it, and the store page has no co-op tag, but the code and
the data are in the build.

Evidence taken from the executable (58,907 strings, 1,189 RTTI class names, 120
leaked source paths) and from the data files:

**A global co-op flag with an FSM setter**

- `sGameSystems::getInstance().getData().isCoop()`, `isCoopMode() == false`,
  `mIsCoop`, `mIsCoopData`
- `FsmSetCoopFlag` and `cFsmAction::cSetCoopFlagParameter` - the flag is set by
  the scene script, not by a menu
- `mCoopFlag`, `mCoopState`, `mCoopDifficulty`, `mCoopKeyNum`
- `cCoopManager` / `mCoopManager`

**A second player actor**

- `uPcsActor<Player2>`

**A per-object network sync framework, in campaign-level classes**

| Owner | Sync data |
|---|---|
| `uCharaBase` | `cPositionSyncData`, `cDamageSyncData` |
| `cNetEnemySync` | `cEnemySyncData` |
| `sItem`, `uItem` | `cDropItemSyncData`, `cPickupItemSyncData` |
| `uDoor2pBase` | `cDoor2pSyncData` (a two-player door) |
| `uLoadRoom`, `uElevatorBase` | load/elevator sync data |
| `sArrange` | arrange, montage, cut move, terminal and fall-out sync data |
| `sObjModel` | `cCoopChainLockSyncData`, `cDoor2pOpenCancelSyncData`, `cElevatorSwitchSyncData` |
| `uCoopChainLock` | `cCoopChainHostCheckSyncData`, `cCoopChainHostAnswerSyncData` |
| `cCoopSyncData` | lobby, ready, chara/equip/colour/mission changes, unlock stage, `cReNetworkDeadSyncData` |

**A co-op UI**

- `uGUI_CoopEnemyHP`, `uGUI_PartnerWarning`, `uCoopChainLock`
- `cCoopMenu::cRaidMenuState`, `cPlayerMatch`, `cCreateSession`, `cLobby`,
  `cSearchSession` - this is the *Raid Mode* online menu, not a campaign menu

**Enemy behaviour for two players**

- `cNpcThinkCoop_Move`, `cNpcThinkCoop_Combat`, `cNpcThinkCoop_SwimWS`,
  `cNpcThinkCoop_SwimUW`
- `mLocalCoopTableRes`, `mGlobalCoopTableRes`, `mChainCoopRes`

### The transport

`prog\game\re_net\ReNetOptional.h`, `sReNetAccessManager`, `mReNetManager`,
`SteamNetworking005`, `mAddressP2p.mPort.mData`, `mPeerId`, `mpP2p`,
`cReNetReliefAidTable`. The exe imports `WS2_32.dll` with a full client **and**
server socket stack (20 functions), `steam_api.dll` (12 functions) and
`XINPUT1_3.dll` (4 ordinals). So the session layer is native code in this
binary, on top of Steam's old networking API.

**Unknown:** whether the campaign co-op path can be pointed at a transport we
control, or whether it is hard-wired to Capcom's matchmaking service.

## Co-op data files in the shipped install

| Path | What it holds |
|---|---|
| `nativePC\game\episode\scene_start_list_coop.ssl` | the co-op episode -> scene table |
| `nativePC\game\episode\<ep>\<room>\texturedetail_coop.tde` | co-op texture list for nine story scenes (13_00, 32_06, 41_00, 51_02, a2_10, f1_00, f1_06, f1_0b, f1_10) |
| `nativePC\game\scene_coop\72\72_07\72_07_s108_coop_em_DARK.arg` | a co-op enemy arrangement |
| `nativePC\game\scene_coop\montage_setting.msd` | co-op montage settings |
| `nativePC\game\chara\em\emXXXX\emXXXX_global_coop.tbl` / `_local_coop.tbl` | per-enemy co-op tables |
| `nativePC\game\coop_table\*.lvt` | co-op rule tables (enemy level, player level, damage rate, dead table, …) |
| `nativePC\game\coop_table\emXXXX\*.lvt` | per-enemy co-op tables (hp, behaviour, tolerant, formidable) |
| `nativePC\game\system\coop_dead.sdl` | what happens when the partner dies |
| `nativePC\msg\Coop_*.gmd`, `epCOOP_*.gmd` | co-op menu text (Raid Mode online: `stage_no_01`…`stage_no_22`, `player_match_button`, `systemlink_button`, `xboxlive_party_button`) |
| `nativePC\id_bhr\32_coopchara` | co-op character portraits/icons |

### The co-op campaign has its own scene flow

`scene_start_list.ssl` and `scene_start_list_coop.ssl` are both 540 bytes,
33 records of 16 bytes, and differ in **18 of the 33 records**. The differing
field is the "next scene" link, so the co-op run visits a different scene
graph than the single-player run. The co-op data is not a bolt-on: it is a
second, authored campaign route.

## Formats decoded this session (all read-only, verified against the real files)

| Format | Layout |
|---|---|
| `.ssl` scene start list | `"SSL"`, u32 version (2), u32 count, then records of 8 u16 |
| `.tde` texture detail | `"XFS"`, quality/param-list/auto-delete/path/format/type fields, then model texture paths |
| `.lvt` value table | `"lvt1"`, u32 version (2), u32 flags, u32 columns, u32 rows, then column descriptors (u8 type + NUL-terminated name) and row-major cells (u8 type + payload: 1 = int32, 2 = uint32, 3 = float32, 4 = string) |
| `.arc` archive | `"ARC\0"`, u16 version (7), u16 entry count, then 80-byte entries (`name[64]`, u32 timestamp, u32 compressed size, u32 size, u32 offset), payloads zlib-compressed, data block starts at 0x8000 |

The `.lvt` parser was checked against all 188 `.lvt` files in the install (every
one parses to exactly its length), the `.arc` parser against all 3,088 archives
and 66,203 entries (every entry decompresses to exactly its declared size).

Reading the co-op tables: **the difficulty scaling is keyed by level, not by
player count.** No co-op table has a player-count column, so any two-player
adjustment is either hard-coded in the executable or applied through the chosen
level index. That is what the executable still has to be asked about.

## Open questions (phase 3, being worked on)

1. Where is the co-op flag set, and what starts a co-op campaign run? The
   `FsmSetCoopFlag` action and the `isCoop()` asserts are the handles.
2. How is the second player created and driven (`uPcsActor<Player2>`)?
3. Which transport does a campaign co-op session use, and can it be pointed at a
   relay we run?
4. Is there any entry point at all in the PC build, or was the campaign co-op
   cut and only the data left behind?

Questions 1 and 4 are answered by the second session below: the co-op
campaign is fully present in the build and the gate is the game mode field.
Questions 2 and 3 are still open and are the two things the next agent has to
resolve.

## Guesses (not verified, listed separately on purpose)

- The co-op layer is probably a leftover from the 3DS original, where the game
  had a two-player local mode; the PC port kept the code and the data but not
  the menu.
- The 18 changed records in `scene_start_list_coop.ssl` suggest the co-op route
  was authored for the full campaign rather than for a handful of stages.

## Source files for this session

- `local/tools/peinfo.py` - PE facts (sections, imports, entropy)
- `local/tools/stringscan.py` - ASCII/UTF-16 string dump
- `local/tools/scan_coop_strings.py` - buckets the string dump by topic
- `local/tools/rtti_paths.py` - pulls RTTI class names and leaked source paths
- `local/tools/ssl_dump.py` - `.ssl` parser and diff
- `local/tools/arc_probe.py` - `.arc` layout probe
- `local/tools/mt_tables.py` - `.lvt` and `.arc` readers (subagent, this session)
- `local/out/coop_tables_report.md` - the co-op table dump and schema
- `<ghidra scripts>/CoopScan.java` - Ghidra script that
  resolves string references and decompiles the functions that touch them

---

# Second session: the mode field, the loaders and the first build

Same day, later. Phases 3 (understand), 4 (choose) and the first slice of
phase 5 (build). The game was **not** launched — the owner deferred it
("Да, но только позже"), so everything below is static analysis plus host-side
tests.

Tooling added this session: Ghidra 12.1.4 (headless project in the workspace
folder), MSVC Build Tools 2022 with the VCTools workload, CMake + Ninja, and
Python 3.13 with PyYAML. 1,635 functions were decompiled into
`<ghidra out>/fn\` and indexed into `<ghidra out>/{xrefs,strings,callers,callees,refs}.tsv`.

## The co-op gate is one int32

The whole "where is co-op switched on?" question reduces to a single field:

```c
mode = *(int32 *)(*(int32 *)(DAT_01169108 + 0x24) + 0x20)
```

| piece | value | how it was confirmed |
|---|---|---|
| `DAT_01169108` | the global `sGameSystems` instance pointer | RTTI name `sGameSystems`; VA 0x01169108 → RVA 0x0D69108 (image base 0x00400000, no ASLR, DllCharacteristics 0x8100) |
| `+0x24` | `sGameSystems::getData()` → the game data pointer | matches the decompiled `getData()` and the `isCoop()` assertion sites |
| `+0x20` | the mode field inside the game data | the field every `isCoop()` / `isRaid()` branch reads |

Mode values:

| value | meaning |
|---|---|
| 0 | campaign (single player) |
| 1 | co-op — the dormant two-player campaign |
| 2 | raid — the online Raid Mode the PC release does expose |

`FUN_004373c0` (the episode-start flow state) reads exactly this expression
and then, in the co-op branch, switches the game system over:

```c
if (*(int *)(*(int *)(DAT_01169108 + 0x24) + 0x20) == 1) {
    ...
    *(undefined4 *)(*(int *)(in_ECX + 0x14) + 0x3c8) = 1;
}
...
if (mode != 1) goto LAB_00437b13;
FUN_0041a900(1);          // switch to the co-op game-system instance
```

The `unique_ptr` member at `+0x14` of that flow state points at the co-op
player/game system in co-op mode, so the second player is created by the
episode-start flow, not by a menu.

## The two functions that read the mode while loading

| function | RVA | what it does |
|---|---|---|
| `FUN_0042bfc0` | `0x0002BFC0` | builds `archive\game\{scene\|scene_coop}\XX\XX_event` and `archive\game\episode\XX\XX_episode` and loads them — **the point where the game picks the campaign or the co-op route** |
| `FUN_0041b710` | `0x0001B710` | loads `archive\game\{scene\|scene_coop}\XX\XX_%02x\XX_%02x_event` and swaps every actor that has a `_coop` resource to the co-op variant |

Forcing the mode to 1 in a detour around these two is what makes the game
load the co-op campaign.

## The mode also picks the save slot

This is the sharpest consequence of the mode override and it is easy to miss.

- `FUN_0041ba70` selects the game-system instance by mode: index **1** (the
  co-op instance) when `mode == 1`, otherwise index **0**. The instances live
  at `(&DAT_0116a344)[index]`.
- `FUN_006d0cb0` uses `this + 0x60 + mode * 0x24` as the per-mode save array.

So a forced co-op session reads and writes a **different save slot** than the
campaign. On a fresh PC save, slot 0 has the campaign progress and slot 1 is
empty. Mirroring slot 0 into slot 1 before the flip (or sharing one slot) is
an open item, and it is the first thing to look at when the co-op campaign is
finally launched and the game appears to "forget" the save.

## The co-op campaign data is complete

Re-checked against the shipped archives of the test copy: `game\scene_coop\`
exists for every episode from `11` to `f1`, together with
`scene_start_list_coop.ssl` and the `coop_table\*.lvt` rules. The co-op route
is an authored campaign, not a leftover subset.

## The input path: XInput (corrected)

> **Correction.** An earlier version of this section said the opposite, that
> the game reads pad state through DirectInput 8 and uses XInput for rumble
> only. That was wrong: it was based on the assumption that ordinal 1 of
> `XINPUT1_3.dll` is `XInputGetState`. The real export table of the shipped
> `xinput1_3.dll` was dumped and says otherwise (see below), and the mod and
> this note were fixed.

Import table of `rerev.exe`:

| import | detail |
|---|---|
| `DINPUT8.dll` | exactly **one** symbol: `DirectInput8Create` (import thunk at RVA `0x0B1E202`) |
| `XINPUT1_3.dll` | by **ordinal** 2, 3, 4 and 5 (thunks at RVA `0x0B1E208`, `0x0B1E20E`, `0x0B1E214`, `0x0B1E21A`) |

The export table of the system `xinput1_3.dll` (32-bit, `SysWOW64`), dumped
from the file:

| ordinal | export |
|---|---|
| 1 | `DllMain` |
| 2 | **`XInputGetState`** |
| 3 | `XInputSetState` |
| 4 | `XInputGetCapabilities` |
| 5 | `XInputEnable` |

So the four ordinals the game imports are exactly
`XInputGetState / XInputSetState / XInputGetCapabilities / XInputEnable`, and
**the game reads pad state through XInput**. DirectInput 8 is used too, but
only for the mouse and the keyboard.

The call sites (all in one small input region, RVA `0x5BD000`–`0x5C1000`):

| call | RVA of the call | the user index it asks for |
|---|---|---|
| `XInputGetState` | `0x005BDB4F` | `[esi+0x10]` of the pad object |
| `XInputGetState` | `0x005BEFF6` | `[esi+0x10]`, guarded by `cmp byte [esi+7], 4` |
| `XInputGetState` | `0x005C0423` | `[esi]`, guarded by `cmp byte [esi-9], 4` |
| `XInputGetState` | `0x005C0555` | `edi`, inside a loop over a table with stride `0x2CC` |
| `XInputGetState` | `0x005C0692` | `edi`, guarded by a byte at `[esi+7]` |
| `XInputSetState` | `0x005BDB67`, `0x005BF00E`, `0x005BF20B`, `0x005C0438` | rumble |
| `XInputGetCapabilities` | `0x005BDDC9` | `[esi+0x10]`, flags `1` = `XINPUT_GAMEPAD` |
| `DirectInput8Create` | `0x005C1F09`, `0x005CB0D7`, `0x005CBE75` | — |

The pattern at the call sites is `push &state; push index; call`, so the user
index is a **per-object field** (`[this+0x10]`), not a constant: the game
asks for "the pad that belongs to this player object". The loop at
`0x005C0555` walks a table of per-pad records with a stride of `0x2CC` bytes
and matches a record against a user index and a device type byte.

**What this buys the mod:** the input seam is a single function, and it can
be hooked through the game's import address table without touching the
game's own code. The mod now does exactly that (`src/game/pad_probe.cpp`):
it hooks `XInputGetState` and logs which user index the game polls. When the
game is finally run in co-op mode, that log answers the decisive question:

- if the game polls index **1** as well as index 0, player 2 is driven by a
  second local pad and the online half only has to replicate *state*;
- if it never polls index 1, the mod has to synthesise player 2's input.

The same hook is where remote input has to go (`setPadOverride()`).

**Still unknown:** how many pad records the game keeps and whether a pad
object for player 2 is created at all in co-op mode.

## Signature verification against the real executable

The mod finds the two loaders by byte signature and refuses to patch
anything that is not unique. Checked against the shipped `.text`:

| signature | matches | RVA of the match | expected |
|---|---|---|---|
| `81 EC 04 02 00 00 53 55 56 8B F1 8B 46 70` (episode loader) | **1** | `0x2BFC0` | `0x2BFC0` ✔ |
| `81 EC 28 02 00 00 53 55 56 8B F1 33 DB` (scene loader) | **1** | `0x1B710` | `0x1B710` ✔ |
| `81 EC 04 02 00 00` (episode prefix only) | 11 | — | ambiguous ✘ |
| `81 EC 28 02 00 00` (scene prefix only) | 2 | — | ambiguous ✘ |

The different stack-frame sizes (`0x204` vs `0x228`) are what make the long
signatures unique. The 6-byte prefix that the inline hook actually overwrites
is *not* unique, which is why the uniqueness check exists.

## The mod, first slice

Repository: `<mod repo>` (its own repo, first commit `082b787`).

- **Injection carrier:** a proxy `dinput8.dll`. The game imports exactly one
  symbol from `DINPUT8.dll`, so a proxy that forwards it is enough to get
  code running inside the game process. Everything else is forwarded to the
  system copy.
- **Hooks:** inline detours at the two loaders, overwriting the 6-byte
  `sub esp, imm32` prologue.
- **Behaviour:** force `mode = 1` before the original call and again after
  it (the loader may reset the mode itself), then a watcher thread re-asserts
  the mode every `coop_mode_poll_ms` (250 ms default) while armed.
- **Deliberate:** mode 2 (Raid) is never touched, and the mode is only
  written once the game data pointer is inside the game image.
- **Guarded:** a signature that matches anything other than exactly once, or
  whose address does not equal the expected RVA, aborts the patch and is
  logged instead.

Build and test run (host-side, no game):

```
=== running tests ===
    Start 1: hook_test ........................   Passed    0.01 sec
    Start 2: iat_test .........................   Passed    0.13 sec
    Start 3: coop_mode_test ...................   Passed    0.12 sec

100% tests passed, 0 tests failed out of 3
```

What the tests prove: the hook framework detours and restores correctly, the
IAT patcher works, the mode-field offsets are the ones the mod uses, the
read/write helpers only touch the mode field, and both loader signatures are
unique in a synthetic `.text` and match the documented prologues.

What the tests do **not** prove: that the game loads a co-op scene, that a
second player appears, or that the save slot behaves. That needs the game.

## The mod, second slice: the pad probe

After the first slice, the second session added `src/game/pad_probe.{h,cpp}`
to the mod: it hooks `XInputGetState` through the game's import address
table (`iat::findImportSlotByAddress` + `iat::patchImportSlot`, so the
game's own code is not touched at all) and logs

- which user index the game polls,
- the button mask, the triggers and both sticks,
- only on a *change*, so the log stays readable.

`setPadOverride()` is the seam for the network half: a callback that can
replace the state the game sees for one user index.

Build and test run after the change:

```
=== running tests ===
    Start 1: hook_test ........................   Passed    0.14 sec
    Start 2: iat_test .........................   Passed    0.12 sec
    Start 3: coop_mode_test ...................   Passed    0.12 sec
    Start 4: pad_probe_test ...................   Passed    0.12 sec

100% tests passed, 0 tests failed out of 4
```

## Still unknown (updated after the third session)

1. **Does the game create the second player at all** when the mode is co-op?
   The episode-start flow suggests yes, but nothing has been run.
2. **How is the second player driven** — a second local pad, or replicated pad
   data? The seam is now known (`XInputGetState`, see above), but whether the
   game polls a second user index in co-op mode is still unknown. The
   `cPlayerPadSyncData` class exists, but it is attached to the Raid-mode
   classes (`cCoopSyncData` → lobby/ready/chara), so it is probably
   raid-only. The campaign co-op of the 3DS original used system link, i.e.
   each system drove its own player locally and the network replicated
   *state*.
3. ~~**Is the co-op game-system instance created up front** or lazily by the
   co-op menu?~~ — **answered in the third session: up front.** Both slots are
   filled from start-up; see
   [the two game-system instances](#the-two-game-system-instances-both-slots-exist-from-start-up).
4. **The save slot** (see above).
5. **The transport.** Not "does one exist" any more — it does
   (`nNetwork::Session*`, `SteamNetworking005`, `WS2_32.dll`). What is still
   open: whether the campaign co-op path can be pointed at a transport we
   control, and whether the Steam session is the only way into a lobby.
6. **Where the co-op menu entry has to be re-added.** The PC interface exposes
   only Raid matchmaking; the co-op lobby states exist in the binary but no
   menu path to them was found yet.
7. **What the two non-`None` `network::PlayMode` values mean**, and which one
   is the campaign co-op. The strings name only `network::PlayMode::None`.

## Guesses (still not verified - listed separately on purpose)

- The campaign co-op of the 3DS original drove player 2 from a second
  system over local wireless (the `systemlink_button` string in the co-op
  menu text), which would mean the PC port expects player 2's input on a
  second local pad and replicates state over the network. That is the most
  convenient case for us, because it splits the problem into "route a pad"
  and "replicate state".
- `cGameSystemDouble` most likely means "the game system that can host two
  players", and the second slot is the partner's world state. The mode
  selects which world the local player is in, which is why the co-op world
  has its own save slot. Not proven - the class name is a guess from the
  leaked RTTI strings, and the second slot is created as a *child* of the
  first one rather than as a peer, which does not fit the "two peers"
  reading.
- The two non-`None` `network::PlayMode` values are probably Raid and
  campaign co-op, with the co-op one being the value the menu writes into
  `sNetworkManage + 0xF4`. Not proven.

## Files produced this session

The workspace paths are in `local/`; the layout is:

- `<mod repo>` — the mod (own git repo, three commits)
- `<ghidra scripts>/{CoopScan,DecompileByString,Callers,DecompileFuncs,RefFuncs}.java`
- `<ghidra out>/fn/` — 1,635 decompiled functions
- `<ghidra out>/*.tsv` — string/xref/caller indexes
- `<ghidra out>/_sigcheck.py` — the signature-uniqueness check
  (throwaway, reimplemented as a unit test in the mod repo)

## Third session: the game already ships a netcode

Same day, later part. The plan was to map the two game-system instances and
then work out what the mod still has to build. The mapping turned up something
that changes the design of the whole project:

> **The PC build contains a complete, host-authoritative network layer for
> the campaign co-op.** The mod does not have to write a netcode. It has to
> unlock the one that is already in the binary.

Everything below is split into what was **observed** in the shipped
executable, what was **derived** from those observations, and what is still
**unknown**.

### An RVA that was read wrong

Ghidra prints a data symbol with its *virtual address*. `PTR_FUN_0106f0b4` is
the address `0x0106F0B4`, so its RVA is `0x0106F0B4 - 0x400000 = 0x00C6F0B4`,
which is in `.rdata`. An earlier note in this file read it as RVA `0x06F0B4`;
that address is inside `.text` and was being disassembled as code, which is
why the vtable looked like garbage.

The section table of the shipped `rerev.exe` (raw offsets, for the record):

| section | RVA | virtual size | raw offset | raw size |
|---|---|---|---|---|
| `.text` | `0x00001000` | `0x00BBB27B` | `0x00000400` | `0x00BBB400` |
| `.rdata` | `0x00BBD000` | `0x00155A50` | `0x00BBB800` | `0x00155C00` |
| `.data` | `0x00D13000` | `0x000D5BE0` | `0x00D11400` | `0x00055E00` |
| `.rsrc` | `0x00DE9000` | `0x00006228` | `0x00D67200` | `0x00006400` |
| `.bind` | `0x00DF0000` | `0x00029AD8` | `0x00D6D600` | `0x00029AD8` |

With the corrected RVAs the two vtables read as pointers:

| vtable | RVA | entries |
|---|---|---|
| class A | `0x00C6F0B4` | `0x00721B40, 0x00F4E480, 0x00A055F0, 0x0042E750, 0x00402990, 0x004A7850, 0x004A7850, 0x0042E750, …` |
| class B | `0x00C6F158` | `0x007227F0, 0x00F4E480, 0x00A055F0, 0x0042E750, 0x00402990, 0x004A7850, 0x004A7850, 0x0042E750, …` |

They are identical from entry 1 on, which is the classic shape of a derived
class. The slot at `vtable - 4` is **not** a `COL` (there is no MSVC RTTI
there), so the class names could not be recovered from the RTTI; the names
below come from the leaked source paths and from the class names in the
assert messages instead.

### The two game-system instances: both slots exist from start-up

**Observed** (disassembly of `FUN_00723790`, `FUN_007236e0`, `FUN_00721ba0`,
`FUN_00721b40`, `FUN_00722790`):

| class | vtable RVA | constructor | destructor | own fields |
|---|---|---|---|---|
| `cGameSystem` (A) | `0x00C6F0B4` | `FUN_00721ba0(slot)` | `FUN_00721b40` | `m_index` at `+0x20` |
| `cGameSystemDouble` (B) | `0x00C6F158` | `FUN_00723790(desiredIndex)` | `FUN_00722790` | `+0x24` own slot, `+0x28` child pointer |
| class C (derived from B) | `0x00C6F17C` | `FUN_007236e0` | — | size `0x54`, scene object at `+0x40` |

`FUN_00721ba0` is the **only** writer of the instance array
`(&DAT_0116a344)[slot]` (one store site, `mov [edi*4+0x0116a344], esi` at RVA
`0x321C0A`), and it asserts that the slot is free and that
`slot < InstanceCount_` with `InstanceCount_ == 2`
(`cGameSystem.h:0x47`, "index < InstanceCount_"). It has exactly **one**
caller.

The constructor of the derived class, `FUN_00723790`, disassembles to:

```
slot = (instances[desiredIndex] == 0) ? desiredIndex : desiredIndex + 1;
cGameSystem::cGameSystem(slot);            // FUN_00721ba0: registers this
vptr = 0x106f158;                           // cGameSystemDouble
this + 0x28 = 0;
if (instances[desiredIndex + 1] != 0) {      // early return, no child
    this + 0x24 = desiredIndex + 1;
    return this;
}
this + 0x24 = desiredIndex;
child = new (0x54) classC;                  // FUN_007236e0
this + 0x28 = child;
```

and `FUN_007236e0` (the constructor of class C) calls `FUN_00723790(0)` with
`ECX` = its own `this`, so a class C *is* a `cGameSystemDouble` with extra
members.

The manager `FUN_00792020` (it sets `DAT_01169108 = this`, the
`sGameSystems` singleton) creates one class C at `+0x6C`. That instance
registers itself in slot 0 and then creates a child, which registers itself in
slot 1 (slot 0 is taken by then) and takes the early-return path, so it has
no child of its own. The manager's reset function `FUN_00792440` destroys the
instance at `+0x6C` and creates a new one, which refills both slots.

**Derived from the above:** the co-op game system — slot 1 — is already there
when the title screen appears. `FUN_0041a900(1)` therefore does **not** trip
its `"false"` assertion when the mode is flipped to 1. This settles the third
open question of the previous session.

**Derived:** slot 1 is created as a *child* of slot 0, and `+0x24` holds the
slot index whose scene object (`(&DAT_0117cc90)[index]`) the object owns.
So the two slots are not two equal peers; the second one belongs to the
first.

### The network layer that ships with the game

**Observed** (string literals in `.rdata`, cross-checked against the
decompilation):

`network::sNetworkManage` — the netcode singleton, global `DAT_01169398`
(RVA `0x0D69398`). Its constructor is `FUN_00880f10`; it asserts `"false"`
when the singleton already exists, exactly like `sGameSystems`, and it is the
only writer of the global (one store site, RVA `0x480F44`). Its source path
is `prog\game\network\sNetworkManage.cpp`. Fields that the constructor
initialises and that the code reads:

| offset | constructor writes | meaning |
|---|---|---|
| `+0x30` | `0` | flag bits |
| `+0xE4`, `+0xE8` | `0x80011010` | unknown |
| `+0xF4` | `-1` | the play mode the co-op menu writes |
| `+0xF8` | `-1` | the player index |

`isHost()` is used as an assertion all over the code:
`network::sNetworkManage::getInstance().isHost() == false` /
`!= false` appear in `FUN_006734a0`, `FUN_006735d0` and `FUN_00674970`.

`network::sCoopManager` (`prog\game\network\sCoopManager.cpp`,
`FUN_0087edc0`) with the nested classes `cStageInfo`, `cCoopSkill` and
`cSaveProc`; the network manager's constructor builds eight of them
(`FUN_0087dad0` in a loop of eight), i.e. one per player slot.

`cCoopSyncData` (`prog\game\network\cCoopSyncData.cpp`) is the **complete
campaign sync vocabulary**, and every message has a host-authoritative reply
(the `cRe…` twin):

| message | reply | what it carries |
|---|---|---|
| `cCreateTimeSyncData` | `cReCreateTimeSyncData` | session start time |
| `cLobbyInSyncData` | `cReLobbyInSyncData` | a player entered the lobby |
| `cLobbyReturnSyncData` | `cReLobbyReturnSyncData` | a player left the lobby |
| `cModifyStageSyncData` | `cReModifyStageSyncData` | **which episode/stage the partner is in** |
| `cModifyCharaSyncData` | `cReModifyCharaSyncData` | the partner's character |
| `cModifyEquipSyncData` | `cReModifyEquipSyncData` | the partner's equipment |
| `cModifyColorSyncData` | `cReModifyColorSyncData` | the partner's colour |
| `cModifyMissionSyncData` | `cReModifyMissionSyncData` | mission state |
| `cUnlockStageSyncData` | `cReUnlockStageSyncData` | stage unlock |
| `cPresenceSyncData` | `cRePresenceSyncData` | presence |
| `cKickSyncData` | `cReKickSyncData` | kick |
| `cReadyOn/OffSyncData` | `cReReadyOn/OffSyncData` | ready state |
| `cReadySyncEndSyncData` | `cReReadySyncEndSyncData` | ready-up finished |
| `cCutStartSyncData` / `cCutStartEndSyncData` / `cCutEndSyncData` | `cRe…` | cutscene start/end |
| `cArrangeEndSyncData` / `cArrangeSyncEndSyncData` | `cRe…` | enemy arrangement |
| `cContinueSyncData` | `cReContinueSyncData` | continue |
| `cRushStartSyncData`, `cRushReadyKillSyncData`, `cRushLoadEndSyncData`, `cRushArrangeEndSyncData`, `cRushSyncEndSyncData` | `cRe…` | the "rush" (timed) mode |
| `cReadyKillSyncData`, `cResultSyncData`, `cResultEndSyncData` | `cRe…` | results |
| `cDeathEnemySyncData`, `cNetworkDeadSyncData` | `cRe…` | deaths |
| `cWantedSyncData`, `cZZZSyncData` | `cRe…` | misc state |

Per-object sync payloads (`cNetSyncData` subclasses):

| object | payload |
|---|---|
| `uCharaBase` | `cPositionSyncData`, `cDamageSyncData` |
| `sItem` | `cDropItemSyncData`, `cEnemyDropSyncData`, `cPickupItemSyncData` |
| `uItem` | `cGetItemSyncData` |
| `uWeapon` | `cExplodeSyncData`, `cShootDownSyncData` |
| `uDoor2pBase` | `cDoor2pSyncData` |
| `uElevatorBase` | `cElevatorSwitchOnSyncData`, `cElevatorOutOfBlockSyncData`, `cElevatorDoorOpenSyncData` |
| `uCoopChainLock` | `cCoopChainHostCheckSyncData`, `cCoopChainHostAnswerSyncData` — the chain lock is a **host question / host answer** pair |
| `sActionCommand` | `cMediateData`, `cMediateResultData`, `cUnlockData` |
| `cCutRushState` | `cLoadArrangeSyncData`, `cSetArrangeSyncData`, `cActivateEnemySyncData` |

Transport and session layer: `nNetwork::Session`, `nNetwork::SessionDriver`,
`nNetwork::SessionDatabase`, `nNetwork::Route`, `nNetwork::Transport`,
`SteamNetworking005`, and `WS2_32.dll` with a full client and server socket
stack.

### The co-op menu

`prog\game\gui\revel\coop\cCoopMenu.cpp` and `cCoopMenuState.cpp`. The menu
states are: `cStateBase`, `cRaidMenuState`, `cQuickConfig`, `cPlayerMatch`,
`cCreateConfig`, `cCustomMatch`, `cSessionList`, `cLobbyBase`, `cLobby`,
`cWaitOtherReady`, `cCreateSession`, `cSearchSession`, `cShortCutMoving`,
`cEndErrorJump`.

`FUN_007cb7b0` (`cCoopMenuState.cpp`) reads a play mode out of the menu
object at `+0x408` and branches on it:

| value | behaviour |
|---|---|
| `0` | `network::PlayMode::None` → leave the menu (transition to state 6) |
| `1` | a co-op specific path; asks `DAT_01169110` about something (argument 10) and sets `menu + 0x3F2 = 1` |
| `2` | the same, and additionally sets `sNetworkManage + 0xF8 = 0` (the local player is player 0) |
| other | assertion `"mode != network::PlayMode::None"` (`cCoopMenuState.cpp:0x5a3`) |

and finally writes `sNetworkManage + 0xF4 = uVar5` (`0` or `2`).

**Unknown:** the two non-`None` values are not named anywhere in the strings;
Raid and campaign co-op are the obvious candidates but nothing confirms it.

### What this changes for the mod

The previous plan was "write a host-authoritative netcode for the mod". That
is now the wrong plan. The remaining work is:

1. **the mode gate** — done (the mod flips the mode field);
2. **a way into the co-op menu** — the PC interface exposes only Raid
   matchmaking, so the co-op lobby states have to be reachable somehow;
3. **reuse the game's own session and transport** instead of shipping one.

### The mod, second slice of the third session: the co-op state probe

The mod gained `src/game/coop_state.{h,cpp}` (commit `348ece6` in the mod
repo). It **only reads**:

```
mode           = *(int32  *)((uint8 *)gameSystems->getData() + 0x20)
instances      = (uint32_t *)(imageBase + 0x0D6A344)   // two slots
netManager     = *(uint32_t **)(imageBase + 0x0D69398) // network::sNetworkManage
netPlayMode    = *(int32  *)(netManager + 0xF4)
netPlayerIndex = *(int32  *)(netManager + 0xF8)
netFlags       = *(uint32 *)(netManager + 0x30)
```

and logs a line whenever one of them changes:

```
coop state: mode=1 instances=2/2 coop_instance=1 net=playmode=-1 player=-2 flags=00000000
```

`playmode` stays `-1` while the network manager has never started a session,
`player` is `-2` before the manager exists. If the mode flip alone moves
`playmode` away from `-1`, the game starts a session by itself and the missing
piece is only the menu entry; if it does not, the netcode needs to be woken up
by the mod.

The same commit **guards the mode flip**: the game reads instance slot 1 when
the mode is co-op and asserts `"false"` when that slot is empty, so the mod
now checks that the co-op instance is there before it writes the mode, and
says so in the log instead of writing. The check is redundant for this build
(both slots exist), but the mod must not depend on that.

Build and test run after the change:

```
=== running tests ===
    Start 1: hook_test ........................   Passed    0.14 sec
    Start 2: iat_test .........................   Passed    0.13 sec
    Start 3: coop_mode_test ...................   Passed    0.13 sec
    Start 4: pad_probe_test ...................   Passed    0.12 sec
    Start 5: coop_state_test ..................   Passed    0.13 sec

100% tests passed, 0 tests failed out of 5
```

What the new test proves: the instance slots are read without running off
the end of the array, the co-op instance is detected as slot 1, the network
manager fields are read at the documented offsets, a missing manager is not
read at all, and the log line is formatted safely. What it does **not**
prove: that the game reacts to the mode flip. That needs the game.

### Still open after this session

- the game has to be run at least once with `coop_state = 1` and
  `pad_probe = 1` to answer: does the co-op path react to the mode flip, does
  a second pad index get polled, and does a session ever start;
- the missing menu entry has to be found (which function builds the "Online"
  entry and why only Raid is offered);
- the save-slot question (slot 0 vs slot 1) is still open and only shows up
  in a real run.

