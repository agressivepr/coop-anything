# Resident Evil Revelations - working notes

Session date: 2026-10-22. Phases covered: 0 (triage), 1 (research), 2 (workspace),
3 (understand, in progress). The owner asked for a from-zero mod.

Nothing in the Steam install was modified. All work happens on the copy in
`<test copy>`.

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
| Owner install | `<game dir>` (read only) |
| Test copy | `<test copy>` |
| Steam tags | Single-player, Multi-player, Co-op, full controller support, Valve Anti-Cheat enabled. No "Online Co-op" and no "Shared/Split Screen Co-op" tag. |

The `.text` section has normal code entropy, so the game code is statically
analysable; `.bind` is only the SteamStub wrapper.

## Tooling on this machine (each item checked, not assumed)

| Tool | Version / path | Note |
|---|---|---|
| Python | 3.13.15, `%LOCALAPPDATA%\Programs\Python\Python313\python.exe` | not on PATH, `python` hits the Store stub |
| PyYAML | 6.0.2 | so `tools/build_directory.py` really runs |
| git | 2.55.0, `%ProgramFiles%\Git\cmd\git.exe` | not on PATH |
| JDK | Temurin 21.0.12.1, `%ProgramFiles%\Eclipse Adoptium\jdk-21.0.12.101-hotspot` | needed by Ghidra |
| Ghidra | 12.1.4 PUBLIC 20260921, `<ghidra>` | installed this session on owner approval; SHA-256 `ddac49f9…87d2d4db` matched the hash published on the release page |
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
- `<ghidra scripts>\CoopScan.java` - Ghidra script that
  resolves string references and decompiles the functions that touch them
