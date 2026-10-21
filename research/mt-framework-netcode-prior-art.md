# MT Framework netcode and modding prior art

Background notes gathered on 2026-10-22 while starting a campaign co-op mod for
Resident Evil Revelations 2 (MT Framework, 32-bit x86). Everything here is
engine-level prior art: no game files are reproduced, only the structure of
public projects and the file formats they document.

It applies to **Resident Evil Revelations 1** as well, and one of its claims
has been confirmed on a second game since: Revelations 1 (Steam appid 222480)
also imports `XINPUT1_3.dll` **by ordinal**, and the export table of the
shipped `xinput1_3.dll` is `1=DllMain, 2=XInputGetState, 3=XInputSetState,
4=XInputGetCapabilities, 5=XInputEnable` — so the "the input seam is XInput,
not DirectInput" finding in §2.4 holds there too, and so does the choice of
`dinput8.dll` as the injection carrier rather than the input seam. See
`games/resident-evil-revelations/NOTES.md`.

Three groups of sources are covered:

1. [888Onyx/umvc3-rollback](https://github.com/888Onyx/umvc3-rollback), a rollback
   netcode for Ultimate Marvel vs. Capcom 3 (MT Framework, x86-64).
2. The Resident Evil 4 co-op mods, which are the closest thing to "how people
   actually add co-op to an MT Framework campaign".
3. The MT Framework `.arc` container format, documented by
   [gibbed/Gibbed.MT](https://github.com/gibbed/Gibbed.MT).

**Correction to an earlier belief in this repo:** `umvc3-rollback` is *not* closed
source. The README phrase "No source, no symbols, no debug build" refers to the
game, not the mod. The repository publishes 63 translation units and roughly 26,000
lines of C++ under `src/`, plus a vendored MinHook. It is the single most detailed
public description of how MT Framework allocates and updates state.

## 1. What the RE4 co-op mods actually do

The two repositories the community usually names are not the interesting ones.

| Repository | What is in it |
|---|---|
| [`GaryFrazier/RE4MP`](https://github.com/GaryFrazier/RE4MP) (redirects to `Grep5268/RE4MP`) | `master` has **no networking at all**: a `while(true) { Sleep(10) }` loop that reads arrow keys and moves a second player. The DLL plus an external injector plus a prebuilt `detours.lib`. |
| [`googprojects/remp`](https://github.com/googprojects/remp) | A 50-byte README and nothing else. No code, no licence. |
| [`goog-projects/remp4`](https://github.com/goog-projects/remp4) | `LICENSE` and a 1,010-byte README. All four commits contain only those two files; the code was never committed. |

The real prior art is two sibling repositories plus an SDK:

- [`Grep5268/ResidentEvil4Coop`](https://github.com/Grep5268/ResidentEvil4Coop),
  GPL-3.0. The actual netcode: TCP on port 5555, JSON packets, Hamachi VPN.
- [`w1nsid/RE4MP`](https://github.com/w1nsid/RE4MP), GPL-3.0 per the README (no
  LICENSE file in the tree). The best design of the lot: UDP, 40-byte packed
  binary packets, batched 32 entities per datagram, a dedicated C relay server on
  port 27015, AOB signature scanning, host-authoritative enemies.
- [`nipkownix/re4_tweaks`](https://github.com/nipkownix/re4_tweaks), zlib-style
  licence, the most reusable asset in the family. It is a `dinput8.dll` proxy with
  a hand-written SDK header set under `dllmain/SDK/`: `cUnit.h`, `cCoord.h`,
  `model.h`, `em.h`, `player.h`, `cManager.h`, `global.h`, `roomdata.h`, `pad.h`,
  `camera.h` and more.
- [`ChokoMenta/re4coop`](https://github.com/ChokoMenta/re4coop) also ships a
  `dinput8.def` proxy. Its contents were not read, and its licence is unknown, so
  do not reuse it until the licence is confirmed.

### 1.1 Injection: external injector versus proxy DLL

Both approaches exist and both are simple.

**External injector** (`RE4MPInjector.cpp`): find the window with
`FindWindowA`, get the PID with `GetWindowThreadProcessId`, `OpenProcess`,
`VirtualAllocEx`, `WriteProcessMemory`, then `CreateRemoteThread(LoadLibraryA)`.
The `w1nsid` fork hardens it: it tries five window titles, falls back to
`CreateToolhelp32Snapshot` on the process name, enables `SeDebugPrivilege`, accepts
`--pid`, and checks that `LoadLibrary` returned non-NULL (with an error message
that says the DLL "must be 32-bit/x86").

**Proxy DLL** (`re4_tweaks`): a `dinput8.dll` next to the game exe that forwards
`DirectInput8Create` to the real system DLL. The README notes the collision rule:
rename it to `winmm` or `xinput1_3` if another mod already owns `dinput8`.

The injector route is import-agnostic and is what both RE4 co-op mods use. The
proxy route gives you a `DllMain` that runs before `WinMain`, which is what makes
an early arena reservation possible.

### 1.2 Hooking

- **Microsoft Detours 3.0** (`DetourTransactionBegin/UpdateThread/Attach/Commit`),
  with a prebuilt x86 `detours.lib` in the repo.
- **Hand-rolled inline JMP**: `Hook()` in `inject/common.cpp` is `memset(0x90, len)`
  plus `E9 rel32`, with no external library at all.
- **NOP patches** to disable engine behaviour, applied only when a signature scan
  confirms the expected build. `re4_tweaks` refuses to run on an unknown version.

### 1.3 The MT Framework class vocabulary

`re4_tweaks`'s SDK is the reference. The class names and field semantics are
MT Framework level and will be present in Revelations 2; only the offsets change.

| Type | Size | Key members |
|---|---|---|
| `cUnit` | 0x0C | `be_flag_4` (ALIVE 0x1, TRANS 0x2, DEAD 0x200, DESTRUCT 0x400, MOVE 0x20, DISABLEMOTION 0x4000), `pNext_8` |
| `cCoord : cUnit` | 0xC4 | `world_70`, `world_old_7C`, `pos_94`, `ang_A0`, `scale_AC` |
| `cModel : cCoord` | 0x324 | `r_no_0_FC`, `id_100`, `speed_104`, `pos_old_110`, `Motion_1D8` |
| `cEm : cModel` | 0x408 | `hp_324`, `hp_max_326`, `emListIndex_3A0`, `m_StatusFlag_3CC`, `flags_3E8` |
| `cEm10 : cEm` | 0xC64 | `Route_target_42C`, `Sub_pos_450`, `Go_pos_45C`, `MOTION_INFO Sub_mot_B94` |
| `cPlayer : cEm` | 0x814 | `m_SubMot_474`, `m_Frame_544`, `m_CmdTimer_54C` |
| `cManager<T>` | 0x1C | `m_Array_4`, `m_nArray_8`, `m_blockSize_C`, `m_pAlive_14`, `m_pAliveBack_18` |
| `MOTION_INFO` | 0xD0 | `Mot_frame_24`, `Seq_AC`, `Seq_frame_B8`, `Seq_speed_C0` |
| `GLOBAL_WK` | 0x8690 | `Rno0_20` (Init/StageInit/RoomInit/MainLoop/Doordemo/Ending/Option), `curGameTime_5C`, `deltaTime_70`, `AreaNo_4F38`, `curRoomId_4FAC`, `playerHpCur_4FB4`, `flags_STATUS_0..3_501C..5028`, `frameCounter_530C`, `gameDifficulty_847C`, `SaveKind_8480` |

`cManager<T>` is the container pattern worth stealing: `get(i) = (T*)((uint8_t*)m_Array_4
+ i * m_blockSize_C)`, and `m_pAlive_14` / `m_pAliveBack_18` are a linked list of
live entries. Walking the array by index and walking the alive list give different
sets, and `emListIndex_3A0` is not the same thing as an array slot.

### 1.4 What the RE4 mods had to invent, and why that does not apply here

RE4 has no second campaign player at all, so the mods **invent** one:

1. Force the partner into existence in every area by setting a status bit:
   `GLOBAL_WK->flags_STATUS_3[3] |= 0x04000000` (`STA_SUB_ASHLEY`, bit 101 of a
   128-bit field), plus `subCostume_4FCB = 2`.
2. NOP out the partner set and partner movement code (`base+0x4E8A41`,
   `base+0x35E9FA` and neighbours, `base+0x35EB4C`, `base+0x2C520B`).
3. Re-purpose the partner entity as the remote player: push the remote position
   into the partner, and re-target the `RouteCkToPos` detour so the AI "follows"
   the remote player instead of the local one.
4. Disable collision asymmetrically so the two avatars do not fight.

**Revelations 2 is the opposite case.** It already ships a real second local
player and an online Raid mode. The work is *feeding remote input into an
existing second-player controller* and possibly reusing the engine's own session
layer, not synthesising a player.

### 1.5 Bugs worth knowing before writing any of this again

Documented or visible in the RE4 prior art:

- A single global sequence counter silently dropped valid packets from other
  streams. Fixed by accepting every valid packet and filtering per stream.
- `RE4MP`'s `GetCEmPos` adds `0x91` to an `int*`, which is `+0x244` bytes, not the
  `+0x94` the SDK documents. Every position read was wrong.
- One project reads rotation at `+0xA4` (only the Y component) where the SDK says
  `ang_A0` is a three-float vector.
- The "interpolation" is `previous + (new - previous)` with no time factor, so it
  snaps to the newest packet; and the same duration variable is compared against
  `std::clock()` ticks in one place and seconds in another.
- `send()` is called with a fixed 32,768-byte length regardless of the JSON size.
- `GetEnemyData()` leaks a `new Enemy()` chain on every send tick.
- Entity identity is index-based, so if two clients allocate enemies in different
  orders, entities are mismatched or dropped silently.
- Partner animation is never synced, only position and rotation, so the partner
  teleports and does not play locomotion animation. The workarounds are telling:
  copy `pos_old_110` to stop footstep sounds, and zero `speed_104` so the engine
  does not infer velocity from AI locomotion.
- Cutscene mitigation is one line: exclude area IDs 288 and 256 from sync.
- Nothing in any of these projects touches the pause flag, the cutscene sequencer,
  the save system or frame timing. Those remain unsolved by the prior art.

## 2. umvc3-rollback: the rollback netcode for an MT Framework game

Full source at <https://github.com/888Onyx/umvc3-rollback> (created and last pushed
2026-09-09). The design journal (`PATCHNOTES.md`, 832 KB) and the Ghidra project
are excluded by `.gitignore` and are only available on request.

### 2.1 The central idea

> "An emulator gets rollback for free: it owns the machine, so a savestate is the
> whole machine. A native game gives you none of that. This project took the other
> road: **Own the memory instead of the tick.**"

Four properties follow: every byte of engine state sits inside a range that can be
snapshotted; every allocation is intercepted so an object map
(`addr -> {size, birth_frame, death_frame, generation}`) is free; a freed slab is
not returned to a free list until its death frame is past the rollback horizon;
and a single object can be reverted instead of a whole page.

### 2.2 The arena

`src/arena.h` and `src/arena.cpp`:

```
PREFERRED_ADDR    = 0x30000000      ARENA_SIZE   = 2 GB
FALLBACK_ADDR     = 0x40000000      PAGE_SIZE    = 4096
ALLOC_GRANULARITY = 64 KB           MAX_PAGES    = 262144
HEAP_ZONE_SIZE    = 256 MB          RING_SLOTS   = 60
```

`arena::early_init()` runs from `DllMain(DLL_PROCESS_ATTACH)`, before the game's
first instruction: it reserves the 2 GB arena with `MEM_RESERVE | MEM_WRITE_WATCH`
and installs IAT hooks. It intercepts only `VirtualAlloc` calls with
`flProtect == PAGE_READWRITE`; new allocations are carved out of the arena and the
real `VirtualAlloc` is only asked to `MEM_COMMIT` at the arena address.
`VirtualFree(MEM_DECOMMIT)` is faked with a memset so pages stay committed for
snapshots, and `MEM_RELEASE` returns the range to the in-arena free list.
`HeapAlloc` is redirected to a 256 MB bump allocator at the top of the arena, with
a 1 MB committed guard because the engine's autonomous ogg decoder over-reads a
few KB past the zone top.

The hooks are hand-rolled IAT patches, not MinHook. MinHook is used separately for
inline hooks on the game's own code.

**A pass-through is a permanent hole.** Anything the hook does not cover is logged
as `ARENA-PASSTHROUGH: FULL` and flagged as a breach. The comment is explicit that
such a slab "is out-of-arena and will never revert".

### 2.3 Save, load, rollback

`arena::save(frame)`: reset the write-watch, then for each dirty page record a
bitmap bit, copy 4,096 bytes into the slot, and append the page index to a dense
list. Sixty slots, round-robin; the evicted slot is folded into the baseline
(~1.6 ms memcpy) or deferred to a background worker. The scan is clamped to the
allocation watermark, because scanning the full 2 GB reserve cost a flat ~2.5 ms
per frame.

`arena::load(frame)`: find the slot for the target frame, build the revert set as
the OR of dirty bits over all slots with a newer frame, OR in a *non-destructive*
`GetWriteWatch(0, ...)` so that writes landing after the last save cannot smuggle
post-target mutations into the restored state, then walk the ring slots backwards
to find the newest source for each page. Pages with no source are zeroed
("orphan-zero"). Pages can be excluded from the revert set so the engine
re-derives them.

Two extra mechanisms exist because a page-blind revert is too faithful: it
reinstates bytes that were right at save time and are wrong now, such as a handle
to a resource the driver has since released. That produced most of the crashes, so
the project added scoped "coherent full" ranges (a linked list is a cross-page
coherence group, and a windowed revert can leave a node page live, which breaks
the chain and then the allocator free list) and a sparse override table
(`src/dynamic_restore.h`) consulted only where the blind revert is known to be
wrong.

The shipped `dynamic_restore` registry is a skeleton carrying three hand-written
fixes. The general object-granular restore it implies is not implemented, and
`idspine` (birth stamps on every allocation) is shadow-only: nothing restores by
stamp yet.

### 2.4 The input seam, the most transferable finding

> "The input seam is **XInput, not DirectInput**. `XINPUT1_3.dll` is delay-load
> imported *by ordinal* (2 `GetState`, 3 `SetState`, 4 `GetCapabilities`, 5
> `XInputEnable`), and DirectInput8 is only ever used to create `GUID_SysMouse`
> and `GUID_SysKeyboard`. A static import table cannot see a delay-load, which is
> how this was wrong for a long time."

The netcode presents a synthetic 12-byte `XINPUT_GAMEPAD` at the device seam and
lets the engine derive its own eleven-field input window. It does not write that
window.

Two consequences:

- **Button configuration is a determinism gate.** Two peers with different pad
  bindings are running two different games. The mod learns the engine's button
  permutation passively from single-button frames and hashes it onto every packet
  (`config_hash`) so a mid-session remap is caught.
- **Read the first connected XInput index**, not index 0. Reading a disconnected
  slot sends NEUTRAL forever, which on the wire looks exactly like "my inputs are
  not reaching him".

`dinput8.dll` was still chosen, but as the **injection carrier**, not as the input
seam. The real DLL is loaded from the system directory via `GetSystemDirectoryA`,
so the proxy does not recurse into itself.

### 2.5 Wire format

A single non-blocking UDP socket, pumped from the hooked main loop so it stays on
the deterministic thread. There is no network thread by design.

Control packet, 16 bytes: `u32 magic (0x33564D55, "UMV3")`, `u8 type`, `u8 pad[3]`,
`u32 nonce`, `u32 stamp_ms`.

Input packet: `u32 magic`, `u8 type`, `u8 count`, `u8 pad[2]`, `u32 epoch`,
`u32 want_epoch`, `i32 start_frame`, `i32 ack_frame`, `u32 cfg_hash`,
`i16 frame_adv`, `i16 pad2`, then `u8 blobs[32][12]`. Twelve bytes per frame, up
to 32 frames per packet, worst case about 460 bytes per datagram.

Message types: `SYNC_REQ`, `SYNC_REPLY`, `INPUT`, `INPUT_ACK`, `QUALITY_REQ`,
`QUALITY_REPLY`, `KEEPALIVE`, `BEGIN`, `EPOCH`, `ENGINE_ARM`.

Above the transport sit epoch agreement (so both peers share frame 0), input delay
sized from the handshake RTT, ack-based retransmission, GGPO-style TimeSync frame
advantage give-back, and a prediction barrier. Constants that matter:
`MAX_PREDICTION = 8`, `TS_RECOMMEND_INTERVAL = 60` frames, `STALL_MAX = 180` frames
(3 s at 60 fps), `NET_INPUT_REDUNDANCY = 32`.

The redundancy window was 4 and that was a bug: a loss burst longer than about four
packets (roughly 67 ms, routine on a relay) dropped those frames permanently, and
without rollback the peer then predicted them forever.

**Epoch agreement is not optional.** A purely local game event must never zero the
lockstep clock on one side only; that produced "peer A at frame 55, peer B at frame
55255", a permanent and silent desync. Local triggers *propose*, both peers
converge on `max(epoch)`.

### 2.6 Match-start state sync

> "Decided by binary RE and the GGPO literature (both unanimous): NO state transfer
> (impossible cross-process, pointers plus ASLR); instead force both peers' next
> round to an identical RNG seed and let the game's OWN broadcaster fan it into
> MAIN plus stage plus every gameplay object's per-context stream. Requires
> identical builds; ASLR may stay on."

The seed choke point is hooked and the game's own broadcaster does the rest.

### 2.7 Documented limitations

- Rollback fires in live netplay at depths 4 to 6.
- **No cross-machine desync detection.** The CRC covers 19 curated fighter segments
  and no spawned-layer state, so it cannot see the desyncs that actually happen.
- **Resimulation costs about 12.7 ms per frame.** The author's own verdict: "That
  is the number that decides whether this is a real netcode or a demo, and it is
  currently too slow."
- Peer drift causes one-directional input loss, reproducible around stage select.
- Hole punching fails on symmetric NAT; the relay is the path that works.
- The dynamic restore registry is a skeleton.

> "The gap between 'rolls back' and 'holds a match together' is desync detection and
> resim cost, in that order."

### 2.8 Pitfalls recorded in the source

- **SteamStub**: the retail exe is packed; `.bind` decrypts `.text` in place at
  runtime. The DLL polls a 16-byte signature at `base+0x1000` *and* checks that
  the hook targets are in executable pages, because SteamStub `VirtualProtect`s
  `.text` to read-write, decrypts, then flips it back to execute-read as its final
  step before jumping to the original entry point. Hooking during the read-write
  window returns `MH_ERROR_NOT_EXECUTABLE`. Gate on executable, not on signature.
- **Boot stall**: freezing the main loop at boot skips the game's staged in-main-loop
  subsystem construction and ends in a null-vtable tick crash. You cannot freeze or
  roll back during scenario loading. Sync belongs at character select and at
  match start, never at boot.
- **Game-flow phase is unusable**: the same field reads 5 during character select
  in solo play and 0 in a netplay match.
- **Direct connection must be a preference, not a latch**: trusting the first
  packet from the peer's address forever strands a one-way connection.
- **Allocator critical-section hazard**: a worker froze mid two-part write while
  holding the allocator lock, so the rollback defers rather than restores.
- **Window-thread straddle**: the engine's window thread can be mid-render holding
  the CRT heap lock, so it has to be quiesced through the engine's own
  deactivation flags.
- **`wmgr+0x2a98` may be the main thread** on some builds, so a self-suspend
  deadlock is avoided with a thread-id check.
- **Freezing alone takes about 180 ms**, during which the window thread pumps
  frames itself, so the frame-driver hook bails out when it is off the main thread.

### 2.9 32-bit versus 64-bit

The project targets x86-64 only, and `build.sh` compiles only the 64-bit MinHook
backend. The 32-bit `hde32.c` is vendored but unused. Addresses use a
`0x140000000`-based convention that is rebased at runtime:

```c
inline void init() { g_base = (uintptr_t)GetModuleHandleA("umvc3.exe"); }
inline uintptr_t resolve(uintptr_t ida_addr) { return g_base + (ida_addr - 0x140000000ULL); }
```

The rebasing mechanism is bitness-agnostic: a 32-bit build only changes the
convention base to `0x00400000`. The arena approach is portable in principle because
it uses only Win32 APIs, but the injected DLL must be 32-bit for a 32-bit game, and
the 2 GB arena and 60-slot ring would need re-tuning.

## 3. The MT Framework `.arc` container

Source of truth: `projects/Gibbed.MT.FileFormats/ArchiveFile.cs` in
[gibbed/Gibbed.MT](https://github.com/gibbed/Gibbed.MT) (zlib licence).

### 3.1 Header

```
Offset Size  Field
0x00    4     magic    'ARC\0' (0x00435241) or 'ARCC' (0x43435241)
0x04    2     version  u16, must be 7, 8 or 17
0x06    2     count    u16, number of index entries
0x08  80*N    index
```

Endianness is self-describing: read the first four bytes little-endian; `41 52 43 00`
means little-endian, `00 43 52 41` means big-endian. `ARCC` means encrypted.

Index entries are exactly 80 bytes:

```
Offset  Size  Field
0x00    64    Name            char[64], ASCII, NUL padded, no path separator, no extension
0x40     4    TypeHash        u32
0x44     4    CompressedSize  u32
0x48     4    flags           u32
0x4C     4    Offset          u32, absolute offset in the whole file
```

The data region starts at `8 + 80 * count`. Entries are stored back to back with no
padding.

The `flags` bitfield **differs by endianness**, and this is the easiest thing to
get wrong:

| Field | Little-endian | Big-endian |
|---|---|---|
| `UncompressedSize` | `(flags >> 0) & 0x1FFFFFFF` (29 bits) | `(flags >> 3) & 0x1FFFFFFF` |
| `Quality` | `(flags >> 29) & 0x7` | `(flags >> 0) & 0x7` |

`Quality` runs 0 Lowest, 1 Low, 2 Normal, 3 High, 4 Highest, 5 StreamLow,
6 StreamHigh, 7 Invalid. The packer always writes Normal. For a PC `.arc` it is
opaque metadata.

### 3.2 Compression

The scheme is **not stored in the archive**. It is a per-project setting:
`archive_endian`, `archive_version` (default 7), `archive_compression_scheme`
(`None`, `Zlib`, `ZlibHeaderless`, `LZX`, `XCompress`), `archive_same_size_is_uncompressed`
and `archive_crypto_key`.

The critical rule:

> **If `CompressedSize == UncompressedSize`, the entry is stored raw even in a zlib
> archive.**

You cannot decide whether an entry is compressed from the header alone. A naive
"always zlib-decompress" unpacker fails on archives like this.

`LZX` is enumerated but not implemented. `XCompress` is unpacker-only and is the
Xbox 360 scheme.

### 3.3 Encryption

When the magic is `ARCC`, the **whole index block** (80 x count bytes) is encrypted
as one ECB stream. Each entry's data is then encrypted individually, and the
compressed blob is padded to a multiple of 8 bytes first, so `CompressedSize` is
the padded length and `UncompressedSize` is the only reliable one.

The cipher is Blowfish in ECB mode, 8-byte blocks, no IV. The key is the raw ASCII
bytes of the `archive_crypto_key` project setting, used directly as the key
material: no KDF, no salt, no hashing. The key is not stored in the archive.

### 3.4 Type hashes

`TypeHash = CRC32(typeName) & 0x7FFFFFFF`, where CRC32 is the reflected
`0xEDB88320` variant seeded with `0xFFFFFFFF` **and with no final XOR**, so it is
`zlib.crc32(data) ^ 0xFFFFFFFF` masked to 31 bits.

The type table file format is `<HASH> <EXTENSION> <TYPENAME>`, with the extension
and type name optional and `#` starting a comment. The tool validates that the
parsed hash equals `CRC32(name) & 0x7FFFFFFF`, so a table can be generated from
names alone.

An unknown type is unpacked as `name.UNK#XXXXXXXX`, and that extension is parsed
back as a raw hash on repack. **This is the safe round trip for a file type whose
format has not been reverse engineered yet.**

### 3.5 What is not published for Revelations 2

Gibbed.MT ships six project profiles: Dragon's Dogma, Dragon's Dogma (360),
Dragon's Dogma (PS3), Dragon's Dogma Dark Arisen, Dragon's Dogma Online and
Resident Evil 5. **There is no Revelations 2 profile**, and nothing in the repo
mentions "Revelations" at all.

The Resident Evil 5 profile is the closest published reference for an RE-family PC
title:

```xml
<setting name="archive_endian">Little</setting>
<setting name="archive_version">7</setting>
<setting name="archive_compression_scheme">Zlib</setting>
<setting name="archive_same_size_is_uncompressed">True</setting>
```

The RE5 type table has 65 entries and **no `.tbl` entry**, so it says nothing about
how MT Framework table resources are typed. The Dragon's Dogma Online profile's
type table is three times longer than the others, which suggests it is the profile
that carries the table-format types that a `.tbl` would need.

### 3.6 A Python implementation is about 300 lines

The container is trivial in Python. The hard parts are the entry *contents*, and
in particular the `.tbl` resource format, which nothing public describes.

Recommended approach: write a small Python reader and writer against the spec
above, use Gibbed.MT as a cross-check oracle (unpack with both, diff byte for
byte), and for repacking prefer an **in-place patcher** that preserves the
original index, entry order, offsets, type hashes and quality, changing only the
bytes of the entries you modify. The Gibbed packer rebuilds the archive from
scratch, and whether the retail loader is strict about index order or layout is not
documented.

Blowfish has no standard library module. Either `pip install pycryptodome`
(verify against the standard vector `key = 0, plaintext = 0 -> 0x4EF997456198DD78`)
or implement it from the published constants, which are the standard ones.

## 4. What this means for a campaign co-op mod

Ordered by how much it changes the plan.

1. **Do not synthesise a player.** Revelations 2 already has a second local player
   and an online Raid mode. The reverse problem applies: feed remote input into
   the existing second-player controller.
2. **The input seam is XInput, and it is delay-loaded by ordinal.** A static import
   table will not show it. Check with a delay-load aware dumper before assuming
   anything about the input path.
3. **Button configuration is a determinism gate.** Two friends with different pad
   bindings on different machines will desync immediately. Hash the permutation
   onto the wire and refuse to start if it differs.
4. **The engine's own session layer is worth investigating before writing a new
   transport.** Revelations 2 ships online Raid co-op, so there is a real protocol
   in the binary. Whether it can carry campaign scene flow is the single biggest
   open question, and no prior art answers it.
5. **Scenario-start state sync is unsolved by the prior art.** A fighter round has a
   clean RNG seed. A campaign has a position, an inventory, a set of flags and a
   story progress state. UMvC3 has no answer for this and it is the biggest
   unsolved piece.
6. **Do not lockstep during scene loading.** The boot-stall lesson applies: the
   engine builds subsystems inside the main loop, and freezing it there crashes.
7. **Desync detection is the number one missing piece** according to the author of
   the only public MT Framework netcode. For a campaign it would have to cover the
   AI and enemy state, not a handful of curated fighter segments.
8. **A rollback model is a poor fit for a campaign.** The arena approach costs
   12.7 ms per frame to resimulate in a fighter, and a campaign's AI and enemy
   state would be worse. Actor ownership (the host authoritative for enemies, the
   clients report their own damage) is the model the RE4 prior art converged on
   and it scales better for a story mode.
9. **Budget for the failure modes the RE4 mods hit**: entity identity across
   clients, cutscene areas, partner animation, teleport artefacts, and per-stream
   packet sequencing.
10. **Keep the data side in mind.** The co-op rules in these games live in data
    (`coop_table.arc`, `scene_coop`, `coop_enemy.arc`, per-enemy
    `*_global_coop.tbl` and `*_local_coop.tbl`), not in code. A campaign co-op mod
    that only moves bytes may be able to reuse the engine's own co-op scenarios.

## Sources

- <https://github.com/888Onyx/umvc3-rollback> and its `src/` tree
- <https://github.com/Grep5268/RE4MP>
- <https://github.com/Grep5268/ResidentEvil4Coop>
- <https://github.com/w1nsid/RE4MP>
- <https://github.com/nipkownix/re4_tweaks>
- <https://github.com/ChokoMenta/re4coop>
- <https://github.com/googprojects/remp>
- <https://github.com/goog-projects/remp4>
- <https://github.com/gibbed/Gibbed.MT>
- <https://github.com/gibbed/Gibbed.IO>
- <https://github.com/gibbed/Gibbed.ProjectData>
- <https://github.com/gibbed/XCompression>
- <https://en.wikipedia.org/wiki/MT_Framework>
