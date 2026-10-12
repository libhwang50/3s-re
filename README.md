# 3s-re — reverse engineering *Sabin Sound Star: Renascence Burst* chart files

Notes on the on-disk format of the chart files (`.sss`) used by the abandonware
Korean arcade rhythm game **Sabin Sound Star: Renascence Burst** (3S), plus a
reference Python codec.

3S's chart container is a thin wrapper around an **EZ2AC v6 (EZFF) payload**, so
it is directly comparable to [freem/ez2stuff](https://github.com/freem/ez2stuff).
The interesting 3S-specific parts are the obfuscation layer, the text
instrument table that precedes the EZFF block, and the 64-slot track layout the
game's three chart types share.

## Status

Everything documented here is confirmed by static analysis of `3s.exe` and by
byte-exact round-trip of 36 shipped charts (4 songs). Items that are inferred
rather than proven are flagged inline.

| Area | Status |
| --- | --- |
| `.sss` obfuscation | solved, key recovered |
| Text instrument/sample header | solved |
| EZFF footer (header, tracks, notes) | solved, byte-exact round-trip |
| Track-slot usage per chart type | characterised empirically |
| Difficulty / chart-type naming | solved from the exe's own tables |
| `.rnk` high-score files | partially characterised |
| `.wav` sample obfuscation | **not solved** (different keystream) |
| `.3s` container | **not found on disk** (referenced by the loader) |

## Layout

```
FORMAT.md            the .sss container + EZFF payload spec
docs/TAXONOMY.md     filename grammar, difficulty ladder, chart types
docs/LANES.md        what the 64 track slots mean per chart type
docs/REVERSE.md      how this was recovered: functions, addresses, method
tools/sss.py         decoder / encoder / round-trip verifier (stdlib only)
```

## Using the tool

`tools/sss.py` needs Python 3, no dependencies.

```sh
# human-readable dump (measure:beat notation, sample names resolved)
python3 tools/sss.py decode path/to/kamui_1n.sss

# raw decrypted bytes instead of a dump
python3 tools/sss.py decode path/to/kamui_1n.sss --raw

# lossless JSON, and rebuild a .sss from it
python3 tools/sss.py json   path/to/kamui_1n.sss -o chart.json
python3 tools/sss.py encode chart.json rebuilt.sss

# regression test: verify byte-exact re-encoding over a set of charts
python3 tools/sss.py roundtrip 'path/to/*.sss'
```

`roundtrip` is the regression test. It checks two independent paths —
`parse -> build` and `parse -> JSON -> build` — and each must reproduce the
input byte-for-byte. It currently reports all 36 sample charts identical.
Run it after any change to the codec.

## Not included

No game content is committed. You need your own copy of the game for the
`.sss` / `.exe` files; the `.gitignore` keeps `data/`, `*.7z`, decrypted output
and build droppings out of the index.

## Legal note

3S is abandonware but not free software. The binaries, music and charts are
still copyrighted. This repository contains only format documentation and
clean-room tooling written from observation of the file layout.
