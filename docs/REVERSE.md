# How this was recovered

Method, the functions that matter, and what is still open.

## Inputs available

* `3s.exe` — PE32, x86, image base `0x00400000`. Renascence Burst build.
* `3S.dll` — a small shim DLL (image base `0x10000000`), mostly input-related
  strings.
* Charts for 4 songs (36 `.sss` files): `kamui`, `bleed`, `eraser`, `firestorm`.
* A full shipped file listing (20907 paths, 898 `.sss`) but not the files
  themselves.
* No source, no debug info, no symbols.

## Toolchain

* **Ghidra** via an MCP bridge: `decompile_function`, `read_memory`,
  `get_xrefs_to`, `list_open_programs` / `switch_program`. Full auto-analysis
  takes well over the MCP 60 s timeout but completes asynchronously — function
  count went 13 → 2026 → 2084. Re-run the analysis, then come back.
  `run_script_inline` was disabled (`GHIDRA_MCP_ALLOW_SCRIPTS` unset), so custom
  Java scripting was not available; `read_memory` + `search_instructions` had to
  do the job.
* **objdump** for raw disassembly when the decompiler's stack-slot naming was
  unhelpful:
  ```sh
  objdump -dM intel --start-address=0x406c30 --stop-address=0x407600 3s.exe
  ```
  Section map for translating VA → file offset:
  `.text` `0x401000`/`0x400`, `.rdata` `0x4c3000`/`0xc2000`,
  `.data` `0x4ec000`/`0xea800`, `.rsrc` `0xc7a000`/`0x101600`.
* **Python** for byte-level verification — hex dumps, statistical structure
  recovery, and the round-trip test that ultimately proved the format model.
* **curl** for the community wiki. NamuWiki is JS-rendered and defeats scrapers,
  but a browser User-Agent gets the full HTML:
  ```sh
  curl -sSL -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 \
    (KHTML, like Gecko) Chrome/124 Safari/537.36" \
    "https://namu.wiki/w/Sabin%20Sound%20Star%20:%20Renascence%20Burst"
  ```
  Then strip `<script>`/`<style>` and tags. undumped.miraheze.org is reachable
  directly but has almost nothing on this title.

## Key functions in `3s.exe`

| Address | Role |
| --- | --- |
| `0x00444ce0` | `.sss` decryptor. Holds the 128-byte keystream as immediate operands; writes the decrypted EZFF payload to `tmp.tmp`. |
| `0x004468b0` | EZFF payload loader (parses `tmp.tmp`). |
| `0x00446c60` | converts 11-byte v6 note records into the internal 0x24-byte note struct. |
| `0x004457a0` | filename construction + load orchestration. Source of the difficulty and chart-type tables and the `.3s`/`.sss`/`.ez`/`.ezi` fallback order. |
| `0x004456c0` | parses a note name (`C#3`) into a sample-table index. |
| `0x0040ad70` | main note processor: iterates 42 tracks × up to 1024 notes, branches on chart type in ~20 places. Where the lane map is actually applied. |
| `0x00406c30` | per-mode **keyboard binding** tables and per-player binding save/restore. *Not* a lane map — a dead end worth recording so nobody re-traces it. |
| `0x00411ff0` | song database: enumerates songs with per-mode difficulty ratings. |
| `0x00449c80`, `0x00449bf0` | `.rnk` high-score read/write. |

## Key globals

| Address | Meaning |
| --- | --- |
| `0x004f8098` | `PTR_DAT` → `{"e","n","h","s"}` difficulty labels, at `0x004cd298` |
| `0x004f80c0` | `PTR_DAT` → `{"3","1","2","4"}` chart-type characters, at `0x004cd2a8` |
| `0x00921068` | current difficulty index (0..3), written only by menu code |
| `0x0091cf30` | current chart-type index |
| `0x0091cdf0` | song-slot index |
| `0x00bdfcdc` | mode flag selecting the two filename-construction branches |
| `0x00bdfd04` | per-song chart-type remap table |
| `0x004c9f80`–`0x004ca940` | chart-database label strings (4-byte stride) |
| `0x004ca0c0`–`0x004ca6dc` | chart-database song keys (`d001`…`d050`, then names) |

## Order the format fell in

1. **Structure from statistics, not code.** All nine `kamui` charts share a
   3397-byte ciphertext prefix despite different plaintexts → a fixed keystream,
   not a per-file key. Aligning the prefix against the known plaintext of a text
   header gave the additive/subtractive form.
2. **Key recovery.** `FUN_00444ce0` contains the keystream as a run of `mov`
   immediates. Decryption is `plain[i] = (cipher[i] - K[i % 128]) & 0xFF`.
3. **Decrypted bytes revealed a hybrid.** ASCII sample definitions followed by
   `EZFF`/`EZTR` magic — i.e. a text container around an EZ2AC payload.
4. **Header offsets were nailed by hex dump, not by trusting the decompiler.**
   The field layout was wrong twice before being pinned. The trap: `0x05` is a
   one-byte *format version* (`6`), not the start of `name1`. Reading `name1`
   from `0x05` swallows that byte and shifts everything, which silently breaks
   the encoder while the decoder still looks plausible. The correct layout is
   `u8@4=0`, `u8@5=6`, `name1[64]@0x06`, `name2[64]@0x46`, then the numerics at
   `0x86`/`0x88`/`0x8c`/`0x8e`/`0x92`.
5. **The round-trip test is what made it trustworthy.** `parse → build → compare
   byte-for-byte` over all 36 charts caught two real bugs that a plausibility
   check would have missed:
   * the header field misalignment above;
   * type-2 (volume) events carrying meaningful bytes *after* the volume byte
     (`75 fb 12 00 01 00`), which a "volume then five zeros" rebuild corrupts.
     Hence the codec stores the 6 payload bytes verbatim in `raw`.
6. **The `0x92` float was mislabelled as a "second BPM"** for a while. It is
   constant per song but varies between songs, which looked like a tempo. It is
   derived: `total_ticks / ticks_per_measure * 4 * 60 / initial_bpm`, exact to
   float precision for all 36 charts.
7. **Difficulty is not in the file.** Confirmed by comparing charts of the same
   song at different difficulties: identical headers. Then `FUN_004457a0` showed
   the two tables, and the counts in the shipped listing explained SHD and the
   orphaned uppercase set.

## Partially characterised: `.rnk`

`.rnk` high-score files are **plaintext** — not obfuscated the way `.sss` is.
5604 bytes:

* `0x000`–`0x0ff` — 256 zero bytes
* `0x100` — song key string
* `0x140` — metadata
* `0x1a0` — title
* `0x1e0` — artist
* `~0x400` onward — 32-byte score records

Not yet decoded: the score-record field layout and where difficulty/level are
keyed within the record set.

## Not solved

* **`.wav` sample obfuscation.** Samples use a *different* keystream from
  `.sss`; decrypted attempts show a `WTH]` magic rather than `RIFF`. The key is
  presumably elsewhere in the exe (a second decryptor routine), and recovering it
  is the single most useful next step for anyone wanting the audio.
* **`.3s` container.** The loader tries `.3s` before `.sss`, but no `.3s` file
  exists in the 20907-path listing. Either a newer format that never shipped, or
  one produced by an authoring tool.
* **Exact slot→button map.** See `LANES.md`.
* **The text-header `flag` column** (0 or 1).
* **Note payload byte 3** (`unk`) — always 0 in this sample set; unknown whether
  it is used by other charts.

## Reproducing the analysis

```sh
# 1. format model regression test — must report all identical
python3 tools/sss.py roundtrip '/path/to/game/data/*/*.sss'

# 2. eyeball one chart
python3 tools/sss.py decode /path/to/game/data/kamui/kamui_1n.sss | head -60

# 3. confirm the derived duration field over a whole tree
python3 - <<'PY'
import sys, glob; sys.path.insert(0, 'tools')
import sss
for f in glob.glob('/path/to/game/data/*/*.sss'):
    h = sss.parse(open(f,'rb').read())['header']
    est = h['total_ticks'] / h['ticks_per_measure'] * 4 * 60 / h['initial_bpm']
    assert abs(est - h['duration']) < 1e-3, (f, est, h['duration'])
print('duration field is derived, all files consistent')
PY

# 4. confirm all tempo/time events sit at tick 0 and in slot 0
python3 - <<'PY'
import sys, glob, collections; sys.path.insert(0, 'tools')
import sss
pos = collections.Counter(); slots = collections.Counter()
for f in glob.glob('/path/to/game/data/*/*.sss'):
    for t in sss.parse(open(f,'rb').read())['tracks']:
        for e in t['notes']:
            if e['type'] in ('bpm', 'beats'):
                pos[e['pos']] += 1; slots[t['index']] += 1
print('positions:', dict(pos), 'slots:', dict(slots))
PY
```
