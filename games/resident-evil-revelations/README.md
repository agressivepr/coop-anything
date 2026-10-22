# Resident Evil Revelations (2012, PC port 2017)

| | |
|---|---|
| Engine | MT Framework (`KATARIBE`, project tree `bhr\biorevhd`) |
| Versions tested | PC/Steam appid **222480**, build 13298487; `rerev.exe` 14,260,576 bytes, SHA-256 `1573B79EB921B1E7F571E6DEB71C39961C7F1940FAD6D19283FC3AEA7FCA48AA`; x86, image base 0x400000, no ASLR |
| Projects | `re1coop` (in progress, unpublished) — probe only; no public co-op project found |
| QA level | none |
| Last updated | 2026-10-22 |

## Where things stand

**This game cannot be played co-op in its campaign, because it has no campaign
co-op.** The PC release ships exactly one two-player mode, and it is not
hidden: **Raid Mode**, the second entry of the main menu
(`КАМПАНИЯ` / `РЕЖИМ «РЕЙД»` / `ПАРАМЕТРЫ`). The game's own description of that
entry is *"Режим совместного прохождения, в котором вы боретесь с врагами
разной сложности."* — co-op play, against enemies of varying difficulty. The
campaign is single player and its pause menu has no invite path
(`ЗАГРУЗКА` / `ПАРАМЕТРЫ` / `ВЫЙТИ`).

An earlier version of this write-up claimed the opposite: that the PC build
contains a complete two-player campaign with its netcode, closed only by a
`mode` field that is always `0`. **That was wrong**, and the section
[How the game works (for co-op)](#how-the-game-works-for-co-op) now records
what replaced it and how it was measured. The `re1coop` mod that was built on
the claim is kept as instrumentation and as a record; it does not add co-op.

## Existing work

| project | state |
|---|---|
| `re1coop` (in progress, unpublished) | proxy `dinput8.dll` that loads into the game and reports its state (mode, game-system instances, network manager fields, pad index). It used to write the mode field; that is now disabled. No netcode. |
| No other project found | — |

Searched on 2026-10-22: Steam community, GitHub, general web search turned up
nothing for the PC release. Nexus Mods and ModDB refused the fetcher with
HTTP 403 and the Steam community answered 429, so those are only partly
covered; Reddit and YouTube were not checked.

## How the game works (for co-op)

### The mode field — what it actually says

One `int32`:

```c
mode = *(int32 *)(*(int32 *)(sGameSystems + 0x24) + 0x20)
```

`sGameSystems` is a global pointer at RVA `0x0D69108`, `+0x24` is `getData()`
and `+0x20` inside the game data is the mode. Measured **in a running game**
with the probe in `re1coop`:

| where | `mode` |
|---|---|
| title screen / main menu | `0` |
| Raid Mode lobby | `1` |
| Raid Mode gameplay | `1` |

So `0` = campaign/neutral and **`1` = Raid Mode**. The same probe shows the
network manager's `playmode` going `-1` (menus) → `0` (Raid lobby) → `1`
(Raid gameplay). No value other than `0` or `1` was observed for the mode.

What this kills: the claim that `1` is a dormant campaign co-op and `2` is
Raid Mode. `1` is the mode the game already uses for its shipped two-player
mode.

### Why the assets looked like campaign co-op

All of these exist in the shipped build, and all of them belong to Raid Mode /
the network layer that Raid Mode uses:

* `game\scene_coop\...` — a parallel scene tree for every episode id
  (`01`…`f1`), 81 `*_event.arc` files;
* `scene_start_list_coop.ssl`, `coop_table\*.lvt`;
* `_coop` variants of scene actors, swapped in by the scene loader
  (`FUN_0041b710` at RVA `0x0001B710`);
* `cCoopSyncData` and the per-object sync payloads (position, damage, item
  pickup, doors, elevators, `uCoopChainLock`), `network::sNetworkManage`
  (`DAT_01169398`), `nNetwork::Session`, `SteamNetworking005`;
* a second game-system instance (`cGameSystemDouble`, `(&DAT_0116a344)[1]`),
  read by `FUN_0041ba70` when the mode is `1`.

`FUN_0042bfc0` (RVA `0x0002BFC0`) does pick between
`archive\game\{scene|scene_coop}\XX\XX_event` from the mode — that part of the
old write-up is accurate. It is the Raid/co-op route of a mode that the game
already ships and enters, not a hidden campaign route.

### How this was measured

No memory pokes: the game was driven through its own menus with injected keys
(arrows need `KEYEVENTF_EXTENDEDKEY`), the screen was read back with OCR, and
the probe read the mode from process memory. The menu contents above are what
the game renders, not what the executable seems to imply.

### A pointer trap worth knowing

The globals at `0x0D69108`, `0x0D69398`, `0x0D6A344` live in the image, but the
objects they point to live on the heap (`0x220A…`). A guard that accepts only
image addresses silently rejects every game object. That bug made `re1coop`
report `mode=-1` for the life of the project, and made its mode write a no-op.

## What we changed

* **`re1coop`: the mode probe never worked.** `currentGameData()` /
  `currentMode()` rejected heap pointers with an image-range test. Replaced with
  `readableMemory()` (`VirtualQuery` → committed, not `PAGE_NOACCESS` /
  `PAGE_GUARD`). The probe now reports `0` at the menus and `1` in Raid Mode.
  Unit tests cover the helpers; the value was confirmed in a running game.
* **`re1coop`: the mode override is disabled.** Writing `1` into a campaign
  would point the engine at Raid scene data for a campaign chapter. The
  function logs once and returns `false`. The enum/comments that claimed
  `1 = co-op`, `2 = raid` are corrected.
* **Two start-up crashes fixed earlier** (kept, tested): linking the proxy
  without `/guard:cf` leaves an empty CFG valid-target table (`0xc0000409`
  at start-up); using the CRT stream layer inside the DLL makes a narrow
  `fprintf` on a wide stream raise an invalid parameter, which the same CFG
  gap turns into the same fast-fail. The logger uses `CreateFileW`/`WriteFile`.
  `tests/load_config_check.cpp` and `tests/log_test.cpp` catch both.

## QA

- `build.bat` in the `re1coop` repo: configure, build, `ctest`.
- Last suite result: **8/8 passed**, 2026-10-22.
- In-game: repeated sessions reached the main menu and Raid Mode and played;
  `re1coop.log` showed `mode=0` → `mode=1` on entering Raid Mode; the game's
  save file was byte-identical (SHA-256) before and after every run.
- QA level stays `none`: there is no co-op feature to test.

## How to play

There is nothing to install for co-op in the campaign; it does not exist.
Raid Mode co-op is the game's own, over Steam, unchanged.

`re1coop` can be built and dropped in as a diagnostic (`dinput8.dll` +
`re1coop.ini`), but it changes no gameplay.

## Known issues

- The mod's readme and earlier hub notes described a "dormant campaign co-op";
  those claims are withdrawn here and in the mod's own README.
- The old theory's origin is not fully traced: `scene_coop` asset names match
  campaign episode ids, which is probably what made the inference look safe.
  The runtime measurement overrides the inference.

## Next steps

Two honest options for the next agent:

1. **Stop.** The campaign has no co-op to unlock, and adding one would mean
   writing a new mode (actors, spawn, camera, save, cutscenes) on top of a
   single-player campaign — a ground-up mod, not an unlock.
2. **Raid Mode quality of life.** The game has working two-player Raid Mode
   over Steam. The open question is whether two players can be brought together
   without Steam matchmaking (direct IP / relay) by working on
   `network::sNetworkManage`, `nNetwork::Session` and `SteamNetworking005`.
   That is a different project from the one this folder started.

Do not re-run the "flip the mode" experiment expecting a different result;
the measurement above is reproducible and the mode field is read, not guessed.
