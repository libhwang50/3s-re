# Track slots: what the 64 lanes mean

Every chart declares exactly 64 track slots (the `num_tracks` field is always
64), each with an `"EZTR"` header. Slot **index** is the only thing that carries
meaning — track names are empty in all shipped charts.

## Summary of slot ranges

| Slots | Role | Evidence |
| --- | --- | --- |
| 0 | control: BPM (type 3) and beats-per-measure (type 4) events | all 1586 type-3 and all 36 type-4 events in the sample set live here |
| 1, 2, 21 | unused | empty in all 36 charts |
| 3–21 | gameplay lanes | the notes the player actually hits |
| 22–40 | audio / keysound tracks | populated in every chart; 22–31 always, 32–39 often, 40 rarely |
| 41–63 | unused | empty in all 36 charts |

## Gameplay lanes, per chart type

Observed lane sets across all 36 sample charts (four songs). This is empirical:
it is what the shipped charts actually use, not what the code claims.

**Type `1`** (16 charts) — a stable core of 7 lanes:

```
3 4 5 6 7   8   11
^^^^^^^     ^   ^^
5 keys      |   one extra lane
            one extra lane
```

Slot 9 appears in 1 chart, slot 14 in 1 chart. Everything else is exactly
`3,4,5,6,7,8,11`.

**Type `2`** (9 charts) — the type-1 set plus a second block of five keys:

```
P1: 3 4 5 6 7   8  (9)   11
P2: 12 13 14 15 16   (17 18) 19
```

Slots 12–16 appear in 8 of 9 charts, 19 in 8, 9 in 2, and 10/17/18 in none.
The key lanes show a clean **+9 offset** between the two blocks
(`3..7` → `12..16`), which strongly suggests a two-player or two-hand layout.

**Type `3`** (11 charts) — two groups of three:

```
8 9 10   17 18 19
```

All six appear in all 11 charts. Slot 11 appears in 1 chart. Some type-3 charts
*also* carry key lanes: `firestorm_3n` uses
`3,4,5,6,7,8,9,10,17,18,19` and `firestorm_3h` adds slot 3.

## What the lanes play

Lane notes are not fixed-keysound triggers. The `sound` byte selects any sample
from the text-header table, and in practice the melody is distributed across the
playable lanes. In `kamui_1n.sss`:

| slot | notes | sample families used |
| --- | --- | --- |
| 3 | 188 | `drumloop_*`, plus `lead_*`, `s_*` |
| 4 | 174 | `drumloop_*`, `piano*`, `lead_*` |
| 5 | 130 | `drumloop_*`, `piano*`, `lead_*` |
| 6 | 154 | `drumloop_*`, `piano*`, `lead_*` |
| 7 | 147 | `drumloop_*`, `piano*`, `lead_*` |
| 8 | 299 | `voia_*`, `voib_*` (vocal chops) |
| 11 | 69 | `scrc_x12.wav`, `kickdeep.wav`, `deep0*.wav`, `effect_0*.wav` |
| 22–31 | 914–2965 | backing / BGM material |

So slots 3–7 carry pitched melodic material, slot 8 is heavily weighted to
vocal samples, and slot 11 is the only lane using scratch (`scrc_`), kick and
effect samples. Slots 22–40 hold the rest of the audio bed. This is the EZ2AC
design: the music is synthesised from the note stream, so "playable lane" and
"audio track" are the same mechanism, distinguished only by slot index.

## Mapping to physical controls — inferred, not proven

The community wiki describes 3S's modes as:

* **Arcade** — 5 keys + percussion + pedal
* **Professional** — 10 keys + 3 percussion + pedal
* **Percussion** — 3 percussion lanes
* **Enhanced Arcade** (Renascence Burst) — 3 percussion + 5 keys + 1 pedal = 9
  lanes, where the three percussion notes are separate lanes rather than one

Those lane counts line up with the three shipped chart types, so the working
interpretation is:

| filename char | internal index | mode | lanes |
| --- | --- | --- | --- |
| `1` | 1 | Arcade | 5 keys + 1 + 1 |
| `2` | 2 | Professional | 10 keys + 3 + 1 |
| `3` | 0 | Percussion | 3 + 3 |
| `4` | 3 | unshipped | — |

**Treat this as a hypothesis.** The specific assignment of slots 8/9/10/11 to
percussion vs pedal vs scratch is *not* confirmed. Two observations cut against
a naive reading:

* Type-2 charts use slots 8, 9 and 19 but never 10, 17 or 18 — so the "3
  percussion lanes" do not appear as a clean `8,9,10` group in type 2.
* Type-3 charts use `8,9,10` and `17,18,19`, i.e. two groups of three, which
  looks like two players' percussion rather than one player's.

## Why the exact map was not extracted statically

The obvious candidate, `FUN_00406c30` (called right after the chart is loaded,
and branching on `DAT_0091cf30`, the chart-type index), turned out to build
**keyboard scancode binding tables**, not lane→track maps. Its stack tables
contain values like `0x58,0x44,0x43,0x46,0x56,0x41` (`X D C F V A`) and
`0x57,0x45,0x4f,0x50` — default key assignments per mode, plus save/restore of
per-player binding arrays at `DAT_0091ce1c` / `DAT_0091ce6c` (4 players × 5
slots).

The lane map itself is applied inside the note processor `FUN_0040ad70`, which
iterates the track array (42 tracks × up to 1024 notes) and reads
`DAT_0091cf30` in ~20 places to select behaviour. Recovering the precise
slot→button mapping would require either tracing that function's per-mode
branches in detail or running the game and observing input handling.
