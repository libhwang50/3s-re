> **Superseded.** Everything in this repository — the `.sss` and `.wav` format
> documentation, the taxonomy and lane maps, the codec, and the renderer — has
> been merged into **`../ez2ac-re`**, where it sits alongside the EZ2AC
> (Endless Circulation and FINAL EX) writeups that it kept having to restate.
> The merged repository is `/home/hwang/Projects/ez2ac-re`: the 3S writeups are
> under `docs/3s/`, the tools under `tools/`, and this repository's full git
> history is reachable there through a merge commit.
>
> Nothing here needs updating any more. Make changes in `ez2ac-re`.

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
| `.wav` sample obfuscation | solved, key recovered; 577/577 samples round-trip |
| Note field semantics (velocity, pan, length) | solved — velocity and pan are dB |
| Per-instrument volume | none exists; the per-lane table is a constant 127 |
| Slot names and lane map | names read from the file; lane map extracted from the exe |
| Song audio renderer | working; onset alignment verified at lag 0 ms |
| `.3s` container | **not found on disk** (referenced by the loader) |

The EZ2AC half of this work lives in a sibling repository, `ez2ac-re`: the Endless
Circulation container codec, the `.ez` / `.ezi` formats, and a converter that
turns a `.sss` into an EZ2AC `.ez` + `.ezi` pair. A `.sss` is literally an EZ2AC
chart with those two halves concatenated — for Firestorm the 3S keysound table is
byte-identical to EC's `.ezi`.

## Layout

```
FORMAT.md            the .sss container + EZFF payload spec
docs/TAXONOMY.md     filename grammar, difficulty ladder, chart types
docs/LANES.md        what the 64 track slots mean per chart type
docs/WAV.md         the partial keystream over sample files
docs/RENDER.md       reconstructing the song audio from a chart
docs/REVERSE.md      how this was recovered: functions, addresses, method
tools/sss.py         chart decoder / encoder / round-trip verifier (stdlib only)
tools/wav.py         sample de-obfuscator (stdlib only)
tools/render_song.py song renderer (needs numpy + soundfile)
```

`render_song.py` is the only part with dependencies:

```sh
uv venv && uv pip install numpy soundfile
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

### Samples

The `.wav` files a chart references are only *partially* obfuscated — the first
256 bytes only. `tools/wav.py` undoes that:

```sh
python3 tools/wav.py info    'path/to/sound/*.wav'   # decoded WAV headers
python3 tools/wav.py decode  path/to/1-kick-02.wav -o kick.wav
python3 tools/wav.py roundtrip 'path/to/sound/*.wav'
```

Decoded output is ordinary PCM that `wave`/`ffmpeg` accept. See `docs/WAV.md`
for why the chart keystream must *not* be reused here.

### Rendering the song

There is no BGM file. The song is a render of the chart:

```sh
python3 tools/render_song.py path/to/kamui_1n.sss -o kamui_1n.flac
python3 tools/render_song.py path/to/songs --all -o rendered/
```

See `docs/RENDER.md` for what has to be right — in particular that velocity and
pan are dB values, not linear ones.

## Not included

No game content is committed. You need your own copy of the game for the
`.sss` / `.exe` files; the `.gitignore` keeps `data/`, `*.7z`, decrypted output
and build droppings out of the index.

## Legal note

3S is abandonware but not free software. The binaries, music and charts are
still copyrighted. This repository contains only format documentation and
clean-room tooling written from observation of the file layout.
