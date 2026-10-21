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
two-player campaign. The co-op scene route (`game\scene_coop\` for every
episode from `11` to `f1`), the `_coop` variants of the scene actors, the
partner system (`cNpcThinkCoop_*`, `uGUI_CoopEnemyHP`, `uGUI_PartnerWarning`,
`mIsCallPartner`, `FsmSetPartnerAttackStatus`), a second game-system instance
(`cGameSystemDouble`) and dozens of gameplay branches that test the game mode
are all in the shipped build and in the shipped archives. What is closed is
the switch: the game mode field is always `0` (campaign) on PC, so none of
that code is reachable. The only online mode the release exposes is Raid
Mode.

The work in progress flips that field. Nothing has been launched yet: the
override is verified against the shipped binary and by host-side unit tests,
but the co-op campaign itself has never been run, so the QA level stays
`none`.

## Existing work

| project | state |
|---|---|
| `re1coop` (in progress, unpublished) | a proxy `dinput8.dll` that flips the game mode to co-op around the two loaders that read it, plus a probe of the pad input. No networking yet. |
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

**The transport.** The binary has one of its own
(`re_net\ReNetOptional.h`, `SteamNetworking005`, `WS2_32.dll` with a full
client and server socket stack), but it is the session layer of the *Raid*
mode. Whether the campaign co-op path can be pointed at a transport we run is
unknown.

**Tools and formats.** `.arc` is the MT Framework archive (`ARC\0`, u16
version, 80-byte index entries, zlib payloads, type hash = CRC32 of the type
name); `.ssl` is the scene start list; `.lvt` is a typed value table. All
three were decoded and checked against the whole install (3,088 archives,
66,203 entries, 188 tables, every one parsing to exactly its length).

## What we changed

Nothing in the game yet — the game has never been run with the mod. The work
so far is the reverse engineering above plus the first slice of the mod:

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
3. **The pad probe.** `XInputGetState` is hooked through the game's import
   address table and reports which user index the game polls. This is the
   measurement that decides the rest of the design: if the game polls index 1
   in co-op mode, player 2 is a second local pad and the online half only has
   to replicate state; if it does not, the mod has to synthesise player 2's
   input. The same hook is where remote input will be injected.

## QA

```
cd <mod repo>
build.bat
```

which runs CMake + Ninja and then `ctest`. Last result, 2026-10-22:

```
    Start 1: hook_test ........................   Passed    0.14 sec
    Start 2: iat_test .........................   Passed    0.12 sec
    Start 3: coop_mode_test ...................   Passed    0.12 sec
    Start 4: pad_probe_test ...................   Passed    0.12 sec

100% tests passed, 0 tests failed out of 4
```

These are host-side tests: the hook framework, the IAT patcher, the mode
field offsets and helpers, the uniqueness of both loader signatures, and the
`XINPUT_STATE` layout the pad probe depends on. **No test starts the game**,
so the QA level of this write-up is `none`.

## How to play

Not yet. The mod is not published and there is no release, no installer and
no way to join a friend's session.

## Known issues

- **The save slot.** Forcing co-op switches the game to a different save slot.
  On a fresh save that slot is empty, so a co-op session starts with no
  campaign progress and writing to it does not touch the campaign save.
- **The co-op campaign has never been launched.** The override is verified
  statically only.
- **No networking.** There is no transport, no session, no sync and no
  second player input source yet.
- **The mod ships as a proxy `dinput8.dll`**, which collides with any other
  mod that owns that name.

## Next steps

1. **Run the game with `coop_mode = 1` and `pad_probe = 1`.** That single run
   answers the three open questions at once: does the co-op campaign load, is
   a second player created, and is a second pad polled.
2. **Mirror or share the save slot** between campaign and co-op, otherwise
   every co-op session starts blind.
3. **Build the sync core against the fuzz harness** in `tools/fuzz` (host
   authoritative, epochs, item pickup races, world epoch on load) before it
   touches the game.
4. **Decide the input path** from the probe result: a second local pad (then
   only state has to be replicated) or a synthesised second pad (then the
   network has to carry the input).
5. **Decide the transport.** The binary's own `re_net` layer is worth
   investigating before writing a new one, but it belongs to Raid mode.

The working notes, with every evidence trail and the list of things that are
still unknown, are in [NOTES.md](NOTES.md).
