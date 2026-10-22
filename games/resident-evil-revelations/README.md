# Resident Evil Revelations (2012, PC port 2017)

| | |
|---|---|
| Engine | MT Framework (`KATARIBE`, project tree `bhr\biorevhd`) |
| Versions tested | PC/Steam appid **222480**, build 13298487; `rerev.exe` 14,260,576 bytes, SHA-256 `1573B79EB921B1E7F571E6DEB71C39961C7F1940FAD6D19283FC3AEA7FCA48AA`; x86, image base 0x400000, no ASLR |
| Projects | no public co-op project found; `re1coop` is in progress and **not published** (see the notes) |
| QA level | none |
| Last updated | 2026-10-22 |

## Where things stand

Nobody can play the campaign together today, and no public project tries.

The interesting part is that the PC release already contains the whole
two-player campaign **and the network layer for it**. The co-op scene route
(`game\scene_coop\` for every episode from `11` to `f1`), the `_coop` variants
of the scene actors, the partner system (`cNpcThinkCoop_*`,
`uGUI_CoopEnemyHP`, `uGUI_PartnerWarning`, `mIsCallPartner`,
`FsmSetPartnerAttackStatus`), a second game-system instance
(`cGameSystemDouble`) and dozens of gameplay branches that test the game mode
are all in the shipped build and in the shipped archives — and so is a
host-authoritative netcode with a co-op lobby (see
[The network layer ships with the game](#the-network-layer-ships-with-the-game)).

What is closed is the switch: the game mode field is always `0` (campaign) on
PC, so none of that code is reachable. The only online mode the release
exposes is Raid Mode.

The work in progress flips that field. Nothing has been launched yet: the
override is verified against the shipped binary and by host-side unit tests,
but the co-op campaign itself has never been run, so the QA level stays
`none`.

## Existing work

| project | state |
|---|---|
| `re1coop` (in progress, unpublished) | a proxy `dinput8.dll` that flips the game mode to co-op around the two loaders that read it, plus probes of the pad input and of the co-op state. It does **not** contain a netcode — the game already has one. |
| No other project found | — |

Searched for public co-op mods on 2026-10-22: Steam community, GitHub and
general web search turned up nothing for the PC release. Nexus Mods and
ModDB refused the fetcher with HTTP 403 and the Steam community answered 429,
so those two are only partly covered; Reddit and YouTube were not checked.

The 3DS original of the same game shipped a two-player mode, and the PC build
still carries its data and its code — the co-op menu text still mentions
`systemlink_button` and `xboxlive_party_button`. The PC port kept the
campaign co-op and dropped the entry point, which is why this is a mod job
rather than a config change.

## How the game works (for co-op)

**The gate.** One `int32`:

```c
mode = *(int32 *)(*(int32 *)(sGameSystems + 0x24) + 0x20)
```

`sGameSystems` is a global pointer at RVA `0x0D69108`, `+0x24` is
`getData()` and `+0x20` inside the game data is the mode: `0` = campaign,
`1` = co-op, `2` = raid.

**The loaders.** Two functions read that field while loading:

| function | RVA | role |
|---|---|---|
| `FUN_0042bfc0` | `0x0002BFC0` | builds `archive\game\{scene\|scene_coop}\…` and the episode paths — picks the campaign or the co-op route |
| `FUN_0041b710` | `0x0001B710` | loads the per-scene files and swaps every actor that has a `_coop` resource to the co-op variant |

**The episode start.** `FUN_004373c0` switches to the co-op game-system
instance (`FUN_0041a900(1)`) when the mode is co-op, so the second player is
created by the episode flow and not by a menu.

**The save system follows the mode.** `FUN_0041ba70` picks the game-system
instance by mode (index 1 for co-op, index 0 otherwise, instances at
`(&DAT_0116a344)[index]`) and `FUN_006d0cb0` uses
`this + 0x60 + mode * 0x24` as the per-mode save array. **A forced co-op
session therefore reads and writes a different save slot than the campaign**,
and on a fresh PC save that slot is empty.

**The input path is XInput.** `rerev.exe` imports `XINPUT1_3.dll` by ordinal;
the shipped `xinput1_3.dll` export table is `1=DllMain, 2=XInputGetState,
3=XInputSetState, 4=XInputGetCapabilities, 5=XInputEnable`, so the game
reads pad *state* through `XInputGetState` and uses `XInputSetState` for
rumble. DirectInput 8 is imported too, but only `DirectInput8Create`, and it
is used for the mouse and the keyboard. The user index is a per-object field
(`[this+0x10]`), not a constant.

**The two game-system instances both exist from start-up.** The instances
live at `(&DAT_0116a344)[index]`, and there are exactly two slots — the
constructor asserts `index < InstanceCount_` with `InstanceCount_ == 2`.
The manager constructor creates one instance, which registers itself in slot 0
and then creates a child, which registers itself in slot 1. So the co-op game
system is already there when the title screen appears, and flipping the mode
does **not** trip the `"false"` assertion in `FUN_0041a900`. The mod checks
for the instance anyway before it writes the mode.

**Tools and formats.** `.arc` is the MT Framework archive (`ARC\0`, u16
version, 80-byte index entries, zlib payloads, type hash = CRC32 of the type
name); `.ssl` is the scene start list; `.lvt` is a typed value table. All
three were decoded and checked against the whole install (3,088 archives,
66,203 entries, 188 tables, every one parsing to exactly its length).

### The network layer ships with the game

This is the finding that changed the design of the project: there is no need
to write a netcode. The shipped binary contains

* `network::sNetworkManage` — the netcode singleton (global `DAT_01169398`,
  RVA `0x0D69398`, constructor `FUN_00880f10`) with `isHost()`, a play mode
  at `+0xF4` and a player index at `+0xF8`;
* `network::sCoopManager` with the nested classes `cStageInfo`, `cCoopSkill`
  and `cSaveProc`;
* `cCoopSyncData` — the **complete campaign sync vocabulary**, and every
  message has a host-authoritative reply (`cRe…` twin): `cModifyStage`
  (which episode the partner is in), `cModifyChara`, `cModifyEquip`,
  `cModifyColor`, `cModifyMission`, `cUnlockStage`, `cPresence`, `cKick`,
  `cReadyOn`/`cReadyOff`, `cCutStart`/`cCutEnd`, `cArrangeEnd`, `cContinue`,
  the `cRush*` family, `cResult`, `cDeathEnemy`, `cNetworkDead`, …;
* per-object sync payloads for everything that has to agree between two
  players: position and damage (`uCharaBase`), item drop and pickup
  (`sItem`, `uItem`), explosion and shoot-down (`uWeapon`), the two-player
  door (`uDoor2pBase`), the elevator (`uElevatorBase`), the co-op chain lock
  (a **host check / host answer** pair on `uCoopChainLock`), the action
  command mediator and the cutscene arrangement state;
* a session layer (`nNetwork::Session`, `SessionDriver`, `SessionDatabase`,
  `Route`, `Transport`), `SteamNetworking005` and `WS2_32.dll`;
* a co-op lobby: `cCoopMenu` with the states `cRaidMenuState`, `cQuickConfig`,
  `cPlayerMatch`, `cCreateConfig`, `cCustomMatch`, `cSessionList`,
  `cLobbyBase`, `cLobby`, `cWaitOtherReady`, `cCreateSession`,
  `cSearchSession`, `cShortCutMoving`, `cEndErrorJump`.

`FUN_007cb7b0` (`cCoopMenuState.cpp`) reads the play mode out of the menu
object (`menu + 0x408`) and writes the result into the network manager:
`0` is `network::PlayMode::None` (leave the menu), and the two non-`None`
values take a co-op path — the second one additionally sets the local player
index to `0`. The two non-`None` values are not named anywhere in the
strings; Raid and campaign co-op are the obvious candidates, but that is not
proven.

So the mod's remaining work is **not** "write a netcode" but:

1. the mode gate (done),
2. a way into the co-op lobby — the PC interface exposes only Raid
   matchmaking, so the co-op states have to be reachable somehow,
3. reuse the game's own session and transport.

## What we changed

Nothing in the game yet — the game has never been run with the mod. The work
so far is the reverse engineering above plus the first slices of the mod:

1. **The co-op mode override.** A detour around the two loaders writes `1`
   into the mode field before and after the original call, and a watcher
   thread re-asserts it while an episode runs. The mode is only written once
   the game data pointer is inside the game image, and Raid mode is never
   touched.
2. **Guarded patching.** Both loaders are found by byte signature and the
   patch is refused unless the signature matches exactly once *and* the found
   address equals the expected RVA. The 6-byte prologue the hook overwrites
   is *not* unique on its own (11 and 2 matches), which is why the check
   exists.
3. **A guard on the mode write.** The game reads the co-op game-system
   instance (slot 1) when the mode is co-op and asserts `"false"` when that
   slot is empty. The mod therefore only writes the mode once it can see the
   instance, and says so in the log instead of writing. The check is
   redundant for this build — both slots exist from start-up — but the mod
   must not depend on that.
4. **The pad probe.** `XInputGetState` is hooked through the game's import
   address table and reports which user index the game polls. This is the
   measurement that decides the rest of the design: if the game polls index 1
   in co-op mode, player 2 is a second local pad and the online half only has
   to replicate state; if it does not, the mod has to synthesise player 2's
   input. The same hook is where remote input will be injected.
5. **The co-op state probe.** A read-only probe of the co-op globals: the
   game mode, the two game-system instances, and the network manager with its
   play mode, player index and flag bits. It logs a line whenever one of
   them changes:

   ```
   coop state: mode=1 instances=2/2 coop_instance=1 net=playmode=-1 player=-2 flags=00000000 session=0 state=0 status=0
   ```

   `playmode` stays `-1` while the network manager has never started a
   session and `player` is `-2` before the manager exists. If the mode flip
   alone moves `playmode` away from `-1`, the game starts a session by itself
   and the only missing piece is the menu entry; if it does not, the mod has
   to wake the netcode up. One run answers that.
6. **The proxy was fixed so the game can load it at all.** The first launch
   attempt failed with `The procedure entry point DirectInput8Create could not
   be located in the dynamic link library …\rerev.exe`. The proxy exported the
   MSVC-decorated name `_DirectInput8Create@20`; the game imports the
   undecorated `DirectInput8Create`. A `.def` file now pins the export, and a
   test resolves the export by name so the regression cannot come back — see
   the [export fix](#the-proxy-export-fix) below.

The mod lives in its own repository, `re1coop`, and is not published yet.

### The proxy export fix

The game imports exactly one symbol from `DINPUT8.dll` — `DirectInput8Create` —
by its undecorated name. MSVC exports `__stdcall` functions decorated
(`_DirectInput8Create@20`) unless a module-definition file says otherwise, so
the first built proxy could not satisfy the import and the game refused to
start. Searching the DLL for the *string* `DirectInput8Create` said "found" and
was useless; only the **export name table** shows the decoration.

`dinput8.def` now pins the five exports to their undecorated names:

```
LIBRARY dinput8
EXPORTS
    DirectInput8Create
```

and `tests/export_check.cpp` loads the built DLL and calls
`GetProcAddress(module, "DirectInput8Create")`, which is exactly what the
game's import binder does.

**This applies to any proxy of a system DLL for this game**, not only ours: a
proxy that builds cleanly but makes the game report a missing entry point
almost certainly has this problem. Check the export name table
(`dumpbin /exports <dll>`), not the presence of the string.

## QA

```
cd <mod repo>
build.bat
```

which runs CMake + Ninja and then `ctest`. Last result, 2026-10-22:

```
    Start 1: hook_test ........................   Passed    0.01 sec
    Start 2: iat_test .........................   Passed    0.01 sec
    Start 3: coop_mode_test ...................   Passed    0.01 sec
    Start 4: pad_probe_test ...................   Passed    0.01 sec
    Start 5: coop_state_test ..................   Passed    0.01 sec
    Start 6: export_check .....................   Passed    0.12 sec

100% tests passed, 0 tests failed out of 6
```

These are host-side tests: the hook framework, the IAT patcher, the mode
field offsets and helpers, the uniqueness of both loader signatures, the
`XINPUT_STATE` layout the pad probe depends on, the co-op state helpers
(the instance slots, the network manager field offsets and the log format),
and the proxy's export table (`export_check` resolves `DirectInput8Create`
by name the way the game's loader does).
**No test starts the game**, so the QA level of this write-up is `none`.

## How to play

Not yet. The mod is not published and there is no release, no installer and
no way to join a friend's session.

## Known issues

- **The save slot.** Forcing co-op switches the game to a different save slot.
  On a fresh save that slot is empty, so a co-op session starts with no
  campaign progress and writing to it does not touch the campaign save.
- **The co-op campaign has never been launched.** The override is verified
  statically only. One launch attempt was made; it stopped at the loader with
  a missing `DirectInput8Create` export, which is now fixed, but the game has
  not been started since.
- **There is no way to start a co-op session.** The game has the netcode and
  the lobby states, but the PC interface only offers Raid matchmaking, so the
  co-op menu entry has to be found and re-added first.
- **No second player input source yet.** The pad probe knows which XInput user
  index the game polls, but nothing feeds a second player.
- **The mod ships as a proxy `dinput8.dll`**, which collides with any other
  mod that owns that name.

## Next steps

1. **Run the game with `coop_mode = 1`, `coop_state = 1` and
   `pad_probe = 1`.** That single run answers the open questions at once: does
   the co-op campaign load, is a second player created, is a second pad
   polled, and does the network manager ever leave `PlayMode::None`.
2. **Find the missing menu entry.** Which function builds the "Online" entry
   and why does it only offer Raid? The co-op lobby states are already in the
   binary.
3. **Mirror or share the save slot** between campaign and co-op, otherwise
   every co-op session starts blind.
4. **Decide the input path** from the probe result: a second local pad (then
   only state has to be replicated) or a synthesised second pad (then the
   network has to carry the input).
5. **Point the co-op path at the game's own session layer** instead of
   shipping a new transport. The `nNetwork::Session*` stack and the
   `cCoopSyncData` vocabulary are already there; what is not settled is
   whether a session can be created without the Steam matchmaking that the
   PC UI drives.

The working notes, with every evidence trail and the list of things that are
still unknown, are in [NOTES.md](NOTES.md).
