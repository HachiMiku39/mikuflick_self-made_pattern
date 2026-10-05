# Miku Flick/02 — Custom Chart and Game Mechanism Research

Experimental research into Miku Flick/02 USM resources, lyric display, chart events, Japanese flick input, and possible custom song packs.

**Two modifications have been confirmed on a device: Chinese full-line lyrics appear during MV playback with lyrics enabled, and a one-byte chart edit removes one note from EASY while preserving it in the other difficulties. Complete custom charts, Chinese gameplay input, and newly registered song packs have not been validated.**

This is an English adaptation of the [USM research in the Chinese localization project](https://github.com/HachiMiku39/mikuflick_chinese_localization/blob/9eb9a65079c981342acc6abed55c0677ae227f1d/README.md#usm-解析与歌词修改实验miku-flick02). It covers game-mechanism research rather than the separate `Localizable.strings` localization or jailbreak setup instructions.

The findings are based on `hello_planet.usm` from **＊ハロー、プラネット。**, inspected resource/save data, and the maintainer’s gameplay observations and test feedback recorded on **September 14, 2026**. They should not be generalized to Miku Flick 1, every song, or every USM file.

## Current evidence

| Topic | Evidence and scope |
|---|---|
| Chinese full-line lyrics | Confirmed in MV playback with lyric display enabled |
| Five difficulty slots and input codes | All 29 inputs across the opening line’s five difficulties match the parsed records; a separate edit confirms disabling the EASY note |
| Removing the opening 「ル」 on EASY | Confirmed; the same note remains in the other difficulties |
| Interlude tap mode | Keyboard disappears; the first interlude’s 8 NORMAL taps match the event count |
| Arbitrary note insertion, direction changes, or timing changes | Not individually validated on a device |
| A new `Mov_18` song pack | Proposed only; registration, loading, and persistence after restarting remain untested |

## 1. What is inside the USM?

USM is a CRIWARE container for audiovisual streams and events. Media chunks are interleaved in playback order; the following tree shows the logical contents:

```text
hello_planet.usm
├── CRID: directory, stream information, source filenames
├── @SFV: one MV video stream
├── @SFA / channel 0: one audio stream
├── @SFA / channel 1: another audio stream
├── @CUE: lyric lines, per-character notes, and control events
└── Metadata: UTF tables, video seek indexes, lengths, and alignment
```

| Item | Parsed result for this sample |
|---|---|
| File size | 102,646,144 bytes; chunk parsing reaches the end of the file |
| Video | MPEG-1, 320 × 480, 20 fps, 4,075 frames, approximately 203.75 seconds |
| Both audio streams | ADX, 44,100 Hz, stereo, approximately 204 seconds each |
| Audio source filenames | `pv_083vo2.aif` and `pv_083ok2.aif`; the embedded encoding is ADX |
| Game events | 505 rows in the `CUEPOINT_INFO` table inside `@CUE` |
| Event fields | `name`, `time`, `cue_type`, `parameter` |
| Time base | `time_unit = 1000`: 1,000 units per second |
| Full-line lyric events | 26, including one line of music symbols |
| Per-character note events | 376 records containing kana and numeric parameters |

The `vo2` / `ok2` filenames strongly suggest a vocal track and a karaoke instrumental track, but they are not explicit role labels. There is only one video stream, consistent with switching audio while sharing the MV; this is not evidence of two separate MVs.

`@UTF` is CRI’s binary table format, while the lyric strings use UTF-8 encoding. **LRC, CSV, and JSON are exported editing representations; there is no standalone LRC file inside this USM.** This sample also has no conventional `@SBT` subtitle track. A normal media player or an FFprobe stream listing can therefore miss the lyrics and chart data in `@CUE`.

## 2. Confirmed experiment: Chinese lyrics during MV playback

The lyric experiment changes only events whose `parameter` begins with `3`: keep the prefix and timestamp, then replace the Japanese line. Of the 26 line events, 25 were translated into Chinese and the music-symbol line was preserved.

- **Device confirmation:** translated lyrics appear when playing the MV with lyric display enabled.
- This does not establish that gameplay’s per-character lyrics, input targets, or chart have been translated.
- The Japanese kana, five-slot parameters, and other control events were left unchanged in this lyric-only experiment.
- Video, both audio streams, and all event timestamps were preserved; no media re-encoding was performed.
- Font coverage and layout still need testing for each text and device. Success with one song does not establish support for every Chinese character.

Each replacement fit within its original byte allocation. It was written in place, zero-padded, and its length field updated. The file size, chunk positions, and video seek indexes did not move. Chunk comparison found changes in only one CUE metadata chunk; all other chunks were identical.

## 3. Per-character chart records and Japanese flick input

All container-level `cue_type` values in this sample are `0`. In the discussion below, **prefix means the first character of `parameter`**, not the `cue_type` field. Every event’s `name` is `note`; that does not make every event a playable note.

Prefix `0` records can be interpreted as:

```text
0 + kana character + five codes in this order:
EASY / NORMAL / HARD / EXTREME / BTL (Break the Limit)
```

All 376 records split into exactly five tokens using `11`, `0`, `2`, `3`, `4`, and `5`. **Treat `11` as one effective operation code, not as two independent difficulty slots.** This is a reproducible parsing rule; whether the game internally divides `11` into two subfields has not been established by disassembly.

| Code in a prefix `0` record | Corresponding opening-line behavior |
|---|---|
| `0` | No note in this difficulty; also confirmed by the EASY 「ル」 edit |
| `11` | Tap |
| `2` | Flick up |
| `3` | Flick right |
| `4` | Flick down |
| `5` | Flick left |

The key is explained by the kana row. The maintainer uses this numeric convention for the Japanese keypad:

| Key number | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 0 |
|---|---|---|---|---|---|---|---|---|---|---|
| Kana row | あ | か | さ | た | な | は | ま | や | ら | わ |

For example, 「ル」 is key 9 + up, 「め」 is key 7 + right, and 「お」 is key 1 + down. 「タ」, 「が」, and 「さ」 all use code `11`, but require tapping keys 4, 2, and 3 respectively. The suffix is therefore not the keypad number. The internal kana lookup and handling of voiced kana and special characters still need further analysis.

### Opening-line comparison

The original line is `シェルターのおと　ひとりめがさめた`. The following are **CUE timestamps**, not calibrated hit times. Note-display lead time and any media synchronization offset have not been measured.

| CUE time (s) | Original parameter | EASY | NORMAL | HARD | EXTREME | BTL |
|---:|---|---|---|---|---|---|
| 6.340 | `0シ00005` | 0 | 0 | 0 | 0 | 5 |
| 6.540 | `0ル22222` | 2 | 2 | 2 | 2 | 2 |
| 6.740 | `0タ000011` | 0 | 0 | 0 | 0 | 11 |
| 7.140 | `0の00004` | 0 | 0 | 0 | 0 | 4 |
| 7.340 | `0お00444` | 0 | 0 | 4 | 4 | 4 |
| 7.540 | `0と00004` | 0 | 0 | 0 | 0 | 4 |
| 7.940 | `0ひ00005` | 0 | 0 | 0 | 0 | 5 |
| 8.140 | `0と00044` | 0 | 0 | 0 | 4 | 4 |
| 8.340 | `0り00005` | 0 | 0 | 0 | 0 | 5 |
| 8.540 | `0め03333` | 0 | 3 | 3 | 3 | 3 |
| 8.740 | `0が000011` | 0 | 0 | 0 | 0 | 11 |
| 8.940 | `0さ0001111` | 0 | 0 | 0 | 11 | 11 |
| 9.140 | `0め00003` | 0 | 0 | 0 | 0 | 3 |
| 9.340 | `0た1111111111` | 11 | 11 | 11 | 11 | 11 |

The maintainer’s observed inputs match the table. In the next column, each number is a **key number**, not an event code; arrows indicate flick direction and “tap” means no flick.

| Difficulty | Notes in the opening line | Observed inputs |
|---|---|---|
| EASY | ル、た (2) | 9↑, tap 4 |
| NORMAL | ル、め、た (3) | 9↑, 7→, tap 4 |
| HARD | ル、お、め、た (4) | 9↑, 1↓, 7→, tap 4 |
| EXTREME | ル、お、と、め、さ、た (6) | 9↑, 1↓, 4↓, 7→, tap 3, tap 4 |
| BTL | シ、ル、タ、の、お、と、ひ、と、り、め、が、さ、め、た (14) | 3←, 9↑, tap 4, 5↓, 1↓, 4↓, 6←, 4↓, 9←, 7→, tap 2, tap 3, 7→, tap 4 |

The opening line’s `4ェ` and `4ー` records have no matching BTL inputs. This is consistent with supplementary display characters, but does not prove that all prefix `4` events are disposable decoration.

Counting nonzero tokens in all prefix `0` records gives **65 / 107 / 161 / 225 / 376** for the five difficulties. These are parsed per-character counts, excluding interlude and special events, not device-verified total note counts.

## 4. Confirmed one-byte edit: remove only EASY’s opening 「ル」

This test was built from the original Japanese USM, without applying the Chinese lyric patch:

```text
Event index: 6 (zero-based), time = 6540
Before: 0ル22222 -> EASY=2, NORMAL=2, HARD=2, EXTREME=2, BTL=2
After:  0ル02222 -> EASY=0, NORMAL=2, HARD=2, EXTREME=2, BTL=2
```

| Check | Result |
|---|---|
| Expected behavior | EASY loses 「ル」 and retains only the final 「た」 in the first line; other difficulties retain 「ル」 |
| Device result | **Matches the expectation** |
| Byte change | Offset `0x5C6C` (decimal 23660): ASCII `2` (`0x32`) → `0` (`0x30`) |
| Binary comparison | Still 102,646,144 bytes; every other USM byte is identical |
| Original SHA-1 | `5c3a4a663c97528b34fe509b94b087a17800d2f5` |
| Modified SHA-1 | `7562bb0d3a39f829b4874c99f8fbd4f86b30e923` |
| External verification | The USM entry in `verificationFile.dat` was updated |

This demonstrates **disabling an existing per-character note independently for one difficulty** and supports the five-slot order. It does not validate arbitrary record insertion, direction changes, hit-time changes, input-character changes, or new songs. The offset applies only to the identified original file; locate the event again in other versions.
## 5. Interlude tap mode and unresolved events

The maintainer observed that between `メモリのなかのキミに　オハヨーハヨー` and `しずかに　ねむる　きみをみた`, the Japanese keypad disappears and the game switches to simple tap notes. **NORMAL requires 8 taps in this section.**

The relevant event sequence is:

```text
98.140 s             2
98.940 s             3♪♪♪♪♪．．．
99.740–107.740 s     15 prefix 5 events
110.140 s            1
110.340 s            3しずかに… (next lyric line)
```

The candidate structure for prefix `5` is `5 + one state digit per difficulty`. **This is a different rule from the operation tokens in prefix `0` records.**

| Parameter | Occurrences in this section | EASY / NORMAL / HARD / EXTREME / BTL | Current interpretation |
|---|---:|---|---|
| `501111` | 8 | `0 / 1 / 1 / 1 / 1` | Ordinary tap candidates for NORMAL and above |
| `500111` | 3 | `0 / 0 / 1 / 1 / 1` | Ordinary tap candidates for HARD and above |
| `500011` | 3 | `0 / 0 / 0 / 1 / 1` | Ordinary tap candidates for EXTREME and BTL |
| `502222` | 1 | `0 / 2 / 2 / 2 / 2` | A non-ordinary-tap state, possibly an end marker; no isolated edit test yet |

The eight records with state `1` in NORMAL’s second slot occur at **99.740, 100.940, 102.540, 103.740, 104.540, 105.340, 106.540, and 107.340 seconds**. This matches the observed tap count. An additional `502222` occurs at 107.740 seconds, so counting every nonzero state as an ordinary tap would incorrectly produce nine. State `2` here must not be interpreted as “flick up” just because that code has that meaning in per-character records.

| Interlude, counting state `1` | EASY | NORMAL | HARD | EXTREME | BTL |
|---|---:|---:|---:|---:|---:|
| First | 0 | **8 (matches device observation)** | 11 | 14 | 14 |
| Second | 0 | 18 | 31 | 32 | 32 |

All counts except the first interlude’s NORMAL count are predictions from the parsed data. There are 48 prefix `5` events across both sections, including two `502222` records; this does not mean 48 playable notes.

The standalone `2` and `1` events around the first interlude support a section/keyboard-state transition hypothesis. Their precise roles still require isolated tests; event ordering alone does not establish causation.

### Event inventory

| Parameter prefix | Whole-song count | Contents and current status |
|---|---:|---|
| `0` | 376 | Per-character chart records; opening-line mapping and disabling one EASY note are verified |
| `3` | 26 | Full lyric lines / music symbols; Chinese MV display is verified |
| `4` | 37 | Small kana, long-vowel marks, etc.; full semantics unresolved |
| `5` | 48 | Interlude states; first NORMAL interlude matches 8 observed taps |
| `6` | 10 | Examples: `600010`, `600110`, `601110`; unknown special-event semantics |
| `1`, `2` | 3 each | Section/display transition candidates; not isolated experimentally |
| `7`, `8` | 1 each | `71900` and `8150` at time 0; initialization parameter candidates |

`8150` might encode BPM 150. Some opening-line events are 200 ms apart, consistent with eighth notes at that tempo, but there is no independent confirmation. The meaning of `71900` is unknown. An editor should preserve unknown events rather than treating these hypotheses as established rules.

## 6. Custom song-pack hypothesis: from resource replacement to Mov_18

**This is a proposed experiment, not a confirmed installation guide.** Existing results demonstrate editing CUE content for a registered song. Adding a pack also involves the song catalog, resource mappings, and the range of identifiers the game accepts. Creating `Mov_18` and `Thum_18` alone does not establish that the game will discover them.

### 6.1 Proposed directory structure

```text
<MikuFlick02 application data container>/Library/InstallData/
├── Mov_çnumber/
│   ├── songname.usm
│   ├── songname2.usm
│   ├── songname3.usm
│   ├── music_(2-digits-number)_01.png
│   ├── pv_xxx_lp.adx
│   ├── pv_yyy_lp.adx
│   ├── pv_zzz_lp.adx
│   └── verificationFile.dat
└── Thum_(2-digits-number)/
    └── thum_(2-digits-number).png
```

`xxx`, `yyy`, and `zzz` represent three distinct three-digit identifiers. The proposed convention also avoids identifiers already used by existing resources. Underscores are literal filename characters, without backslashes. Following the observed `thum_17.png` naming pattern, the candidate thumbnail filename is **`thum_18.png`**, not `thum18.png`. The application data-container path varies by system and installation.

Three songs per pack follows the observed DLC template: the parsed catalog contains 21 DLC packs with 3 songs each, plus base pack 0 with 11 songs. **This does not prove an engine-wide three-song limit.** Globally unique three-digit preview identifiers are also a conservative convention, not a verified engine requirement.

| Resource | Role / current understanding | Editing considerations |
|---|---|---|
| Three `.usm` files | Each song’s MV, audio, lyrics, and chart events | Preserve streams, CUE data, offsets, timing, and indexes; replacing media or inserting/deleting events needs separate tests |
| `pv_*_lp.adx` | Song-selection preview audio, referenced by `m_PreSoundFileName` | Keep each song’s mapping correct; separate from the full audio streams inside the USM |
| `music_18_01.png` | Proposed song atlas; the inspected `music_11_01.png` is 1024 × 1024 and contains logos, creator credits, and cover elements | Preserve atlas dimensions, element positions, and song order; this is not simply three independent covers |
| `thum_18.png` | Proposed thumbnail atlas; the inspected `thum_17.png` is 512 × 512 with gameplay screenshots and small covers | Follow the working atlas layout; automatic detection of new crop regions is not established |
| `verificationFile.dat` | Observed filename/SHA-1 manifest | Update every affected entry using actual filenames and file contents; old hashes cannot simply be copied |

open the `verificationFile.dat` shows the contents below:
```text(for example)
verification datas

7194d1a0a4948e9f38bdcda3a1da59cab3bcf5a6  colorful_melody.usm

67736bd1da56c04ff88f0850eb3029678e5efea1  kotti_muite_baby.usm

cd2c04f37c486a90ea417f47b9ff36882100138b  Yellow.usm

6e2085bb6c4b5ac8a8a84e53fb092c88b1e84299  pv_038_lp.adx

4f016831a168fe7d3c9bb1491df4d7047bd30c2f  pv_040_lp.adx

40ec3573288219bbe4b582af5017ace13dcf5f1a  pv_041_lp.adx

e969062d9ce33d2c5b61040d1b4913097cb283f0  music_**_01.png
eof
```

The atlases and `m_ArtWorkIndex` values 0, 1, and 2 support selecting artwork by song index, but the complete crop coordinates and loading rules remain unconfirmed. Start with the dimensions, positions, and ordering of a working pack.

Seems that there is another encryption of soundpack. It's not OK to just change the package number, like `/Mov_18`,`music_18_01.png`, or package numbers in `verificationFile.dat`

## Follow-up Research: `MikuFlick2.dat` and Music Pack Registration

Further analysis of the general game save file, `MikuFlick2.dat`, shows that it contains a complete music pack catalog and per-song registration data.

This is important for custom pack research because it suggests that simply creating a new resource directory such as `Mov_18` is not sufficient. The game appears to maintain a separate catalog layer that determines which music packs and songs exist.

### File Structure

`MikuFlick2.dat` is not encrypted. It is a nested Apple property list archive using `NSKeyedArchiver`.

The structure is approximately:

```text
MikuFlick2.dat
└── NSKeyedArchiver
    └── StatusData
        ├── game settings / statistics / progress data
        └── m_MusicPackArray
            └── NSMutableData
                └── bplist00
                    └── NSKeyedArchiver
                        └── NSMutableArray
                            ├── MusicPack
                            ├── MusicPack
                            ├── ...
                            └── MusicPack
```

Each `MusicPack` contains another archived data object:

```text
MusicPack
├── m_PackID
├── m_PackName
├── m_Date
├── m_IsBuy
├── m_IsInstalled
└── m_MusicDataArray
    └── NSMutableData
        └── bplist00
            └── NSKeyedArchiver
                └── NSMutableArray
                    ├── MusicData
                    ├── MusicData
                    └── MusicData
```

Therefore, the save file contains both:

1. a music pack catalog, and
2. the song records belonging to each pack.

---

### Registered Pack IDs

The analyzed save contains 22 `MusicPack` objects.

The registered pack IDs are:

```text
0
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
96
97
98
99
```

The important result is that:

```text
Pack ID 18 does not exist in the save file.
```

This means that creating:

```text
Mov_18/
music_18_01.png
verificationFile.dat
```

does not automatically create a `MusicPack` entry for Pack 18.

The game therefore does not appear to discover all installed music packs by simply scanning `Mov_*` directories.

A more likely loading model is:

```text
MikuFlick2.dat / internal music catalog
        │
        ├── Does this MusicPack exist?
        ├── Is the pack purchased?
        ├── Is the pack installed?
        │
        ▼
     Pack ID
        │
        ▼
   corresponding resources
        │
        ├── USM
        ├── ADX
        ├── artwork
        └── verificationFile.dat
```

---

### Pack State

The analyzed save contains catalog entries for Packs `0-17` and `96-99`, even when the corresponding pack is not marked as installed.

For example, Pack 17 contains fields similar to:

```text
m_PackID        = 17
m_PackName      = "Rock_Pack01"
m_Date          = "13/12/20"
m_IsBuy         = true
m_IsInstalled   = false
m_MusicDataArray = <nested archive>
```

In this save:

```text
Pack 0:
    m_IsInstalled = true

Packs 1-17 and 96-99:
    m_IsInstalled = false
```

All registered packs have:

```text
m_IsBuy = true
```

This strongly suggests that pack existence, purchase state, and installation state are stored separately.

Therefore:

```text
Directory exists
```

does not necessarily mean:

```text
MusicPack exists in the catalog
```

and:

```text
MusicPack exists
```

does not necessarily mean:

```text
MusicPack is installed
```

---

### `MusicData` Records

Each song has its own `MusicData` object.

Relevant fields include:

```text
m_Index
m_Title
m_Artist

m_MovieDataFileName
m_PreSoundFileName

m_ArtWorkFileName
m_ArtWorkIndex

m_BPM
m_InputTiming
m_NoteDelay

m_DifficultNum0
m_DifficultNum1
m_DifficultNum2
m_DifficultNum3
m_DifficultNum4

m_PreInstall
m_PVListEnable
m_PVUnlock
m_PVView
```

There are also many progress-related fields, including values for:

```text
clear state
unlock state
top scores
flick results
flick success counts
replay data
```

Therefore, `MusicData` is not only static song metadata. It also acts as part of the per-song save state.

---

### Example: Pack 17

Pack 17 contains three songs:

| Global Index | Title | Movie data | Preview audio | Artwork | Artwork Index |
|---:|---|---|---|---|---:|
| 59 | 孤独の果て -extend edition- | `kodokunohate` | `pv_085_lp` | `music_17_01` | 0 |
| 60 | ローリンガール | `rolling_girl` | `pv_091_lp` | `music_17_01` | 1 |
| 61 | 透明水彩 | `toumei_suisai` | `pv_210_lp` | `music_17_01` | 2 |

The resource filenames are stored without file extensions.

For example:

```text
m_MovieDataFileName = "rolling_girl"
```

rather than:

```text
Mov_17/rolling_girl.usm
```

Similarly:

```text
m_PreSoundFileName = "pv_091_lp"
```

instead of:

```text
Mov_17/pv_091_lp.adx
```

This suggests that the directory name and file extension may be generated by the game code.

A possible implementation would be conceptually similar to:

```text
Pack ID 17
    ↓
Mov_17/
    ↓
rolling_girl
    ↓
rolling_girl.usm
```

This exact path-generation logic has not yet been confirmed.

---

### Global Song Index

The analyzed save contains 74 songs with global indexes:

```text
0-73
```

The indexes are continuous across all music packs.

For example:

```text
Pack 0  -> indexes 0-10
Pack 1  -> indexes 11-13
Pack 2  -> indexes 14-16
...
Pack 17 -> indexes 59-61

Pack 96 -> indexes 62-64
Pack 97 -> indexes 65-67
Pack 98 -> indexes 68-70
Pack 99 -> indexes 71-73
```

This confirms that:

```text
m_Index
```

is a global song ID rather than a song number local to each pack.

If a three-song Pack 18 can be added successfully, the most natural candidate indexes are therefore:

```text
74
75
76
```

However, unused indexes do not automatically prove that the game accepts additional entries.

---

### Artwork Atlas Behavior

The save provides additional evidence for the artwork atlas hypothesis.

Songs in a normal three-song DLC pack share the same artwork file:

```text
m_ArtWorkFileName = "music_17_01"
```

while selecting different artwork entries through:

```text
m_ArtWorkIndex = 0
m_ArtWorkIndex = 1
m_ArtWorkIndex = 2
```

For example:

```text
Song 1:
    m_ArtWorkFileName = "music_17_01"
    m_ArtWorkIndex = 0

Song 2:
    m_ArtWorkFileName = "music_17_01"
    m_ArtWorkIndex = 1

Song 3:
    m_ArtWorkFileName = "music_17_01"
    m_ArtWorkIndex = 2
```

No explicit crop coordinates were found in the corresponding `MusicData` records.

This suggests that the crop layout is probably fixed elsewhere, for example:

```text
m_ArtWorkIndex
        ↓
predefined atlas slot
        ↓
fixed crop rectangle
```

The exact crop coordinates and loading implementation remain unconfirmed.

---

### No `Mov_*` or Verification Paths in the Save

Searching the save file did not reveal strings such as:

```text
Mov_
Thum_
verification
music_18
checksum
signature
```

The save stores resource base names, but not complete filesystem paths.

For example:

```text
m_MovieDataFileName = "kodokunohate"
m_PreSoundFileName  = "pv_085_lp"
m_ArtWorkFileName   = "music_17_01"
```

This suggests that resource paths may be generated using the `m_PackID`.

The save file appears responsible for:

```text
Pack registration
Song registration
Resource base names
Purchase state
Installation state
Song metadata
Song progress
```

while directory mapping and actual file loading are probably handled elsewhere in the game.

---

## Implications for Custom Pack 18

Earlier testing showed that simply cloning a working pack and changing values such as:

```text
Mov_17 -> Mov_18
music_17_01.png -> music_18_01.png
verificationFile.dat filenames / hashes
```

was not sufficient to make a working new pack.

This should not currently be described as evidence of "sound pack encryption."

A more accurate conclusion is:

> There appears to be an additional pack-level catalog, registration, or validation mechanism.

`MikuFlick2.dat` now provides direct evidence for at least one such mechanism.

A custom `Mov_18` directory may contain valid resources, but the game may never attempt to load them unless a corresponding `MusicPack` object exists.

The current model is:

```text
                   MikuFlick2.dat
                        │
                  MusicPack 18
                        │
             ┌──────────┴──────────┐
             │                     │
        Pack metadata         MusicData array
                                   │
                             Songs 74 / 75 / 76
                                   │
                                   ▼
                              m_PackID = 18
                                   │
                                   ▼
                               Mov_18/
                                   │
                     ┌─────────────┼─────────────┐
                     │             │             │
                    USM           ADX           PNG
                     │
                     ▼
             verificationFile.dat
```

Previous Pack 18 experiments mostly tested the lower half of this chain.

The catalog layer now needs to be tested as well.

---

## Proposed Minimal Pack 18 Test

The next experiment should avoid introducing new songs, new charts, or modified media.

Start from a known working pack, such as Pack 17.

### Step 1: Clone the Pack 17 catalog entry

Create a new `MusicPack` object based on Pack 17:

```text
m_PackID        = 18
m_PackName      = "Custom_Pack01"
m_IsBuy         = true
m_IsInstalled   = true
```

The remaining fields should initially stay as close as possible to a known working pack.

### Step 2: Clone its three `MusicData` objects

Assign new global indexes:

```text
74
75
76
```

Keep the original Pack 17 resource names initially:

```text
kodokunohate
rolling_girl
toumei_suisai

pv_085_lp
pv_091_lp
pv_210_lp
```

Only the artwork name may need to change to:

```text
music_18_01
```

with:

```text
m_ArtWorkIndex = 0
m_ArtWorkIndex = 1
m_ArtWorkIndex = 2
```

### Step 3: Copy the original resources byte-for-byte

Create:

```text
Mov_18/
```

and copy the known working files without modifying their contents.

For example:

```text
kodokunohate.usm
rolling_girl.usm
toumei_suisai.usm

pv_085_lp.adx
pv_091_lp.adx
pv_210_lp.adx
```

Generate a corresponding `verificationFile.dat` using the actual hashes of those copied files.

This test intentionally avoids custom USM construction.

Its only purpose is to answer:

> Can the game register and load an additional music pack when both the catalog entry and resource directory exist?

---

## Interpreting the Result

Different failure modes would point to different parts of the loading system.

### Pack 18 does not appear at all

Likely areas to investigate:

```text
MusicPack catalog serialization
Pack ID whitelist
Pack count limit
internal DLC table
catalog reconstruction at startup
```

### Pack 18 appears, but artwork or preview audio fails

Likely areas:

```text
resource path generation
artwork atlas rules
filename mapping
Pack ID -> directory mapping
```

### Pack 18 appears, but selecting a song fails

Likely areas:

```text
USM lookup
song index limits
verificationFile.dat
additional resource validation
hardcoded music tables
```

### Pack 18 works until the game restarts

Likely areas:

```text
save regeneration
internal master catalog
catalog versioning
download / installation metadata
```

---

## Other Save-Level Fields

The outer `StatusData` also contains fields including:

```text
version = 27
m_InfoSerial = 26
```

Their exact purpose is currently unknown.

There is not yet enough evidence to modify them.

However, they should be investigated if a manually inserted Pack 18 entry is:

```text
accepted temporarily
```

but later:

```text
removed or overwritten after restarting the game
```

Such behavior could indicate that the game rebuilds `m_MusicPackArray` from another internal catalog.

---

## Current Conclusion

The evidence now supports the following:

### Confirmed

- `MikuFlick2.dat` is an `NSKeyedArchiver`-based save file.
- It contains `m_MusicPackArray`.
- It contains registered `MusicPack` objects.
- Registered Pack IDs are `0-17` and `96-99`.
- Pack 18 is not present.
- Each `MusicPack` contains its own `MusicData` array.
- `MusicData.m_Index` is a global song index.
- Existing song indexes cover `0-73`.
- Resource base names are stored in `MusicData`.
- Full `Mov_*` filesystem paths are not stored in the save.
- `m_ArtWorkIndex` values `0`, `1`, and `2` correspond to different entries in a shared artwork atlas.

### Strongly supported

- The game does not discover music packs solely by scanning `Mov_*` directories.
- A valid music pack probably requires both catalog registration and filesystem resources.
- Pack directory names are probably derived from `m_PackID`.
- Artwork crop positions are probably selected using a fixed atlas layout.

### Not yet confirmed

- Whether Pack ID 18 is accepted by the executable.
- Whether the number of `MusicPack` objects is hardcoded.
- Whether song indexes above 73 are accepted.
- Whether the game rebuilds the music catalog during startup.
- Whether `version` or `m_InfoSerial` participate in catalog synchronization.
- Whether another internal DLC table exists.
- Whether any additional validation exists beyond `verificationFile.dat`.
- Whether custom Pack 18 resources can be loaded successfully.

The current evidence does **not** establish that the missing mechanism is encryption.

For now, the more accurate description is:

> **an additional music-pack catalog, registration, or validation mechanism remains to be reverse-engineered.**
## `verificationFile.dat` Runtime Test

`verificationFile.dat` contains SHA-1 hashes for the files inside each music pack.

For example, the original `hello_planet.usm` entry in `Mov_11` was:

```text
5c3a4a663c97528b34fe509b94b087a17800d2f5  hello_planet.usm
```

After modifying the USM to display Chinese MV lyrics, the actual SHA-1 became:

```text
d82d241bf52a5c3b3d43510ac7c656588191f7aa  hello_planet.usm
```

However, in a later test, only the modified `hello_planet.usm` was imported. `verificationFile.dat` was left unchanged and still contained the original SHA-1.

The game still loaded and played the modified song normally.

### Result

This shows that the SHA-1 values in `verificationFile.dat` are **not enforced during normal playback of an already installed music pack**.

The file may instead be used for another purpose, such as:

- DLC download verification
- installation integrity checks
- repair or re-download checks
- installer-side validation

Its exact purpose is still unknown.

Therefore, `verificationFile.dat` should currently be treated as a low-priority part of the custom pack investigation.

In particular, failure of a custom `Mov_18` pack is now less likely to be caused by SHA-1 mismatch and more likely to involve:

```text
MusicPack registration
MusicData registration
Pack ID handling
internal DLC catalog
installation state
hardcoded pack limits
```

Further testing should include deleting or deliberately corrupting `verificationFile.dat` to determine whether it is required at all after installation.

### 6.2 Pack 17 reference and sample provenance

The previously inspected pack 17 inventory and registration records correspond as follows:

| Original song title | Global `m_Index` | USM | Preview audio | `m_ArtWorkIndex` |
|---|---:|---|---|---:|
| 孤独の果て -extend edition- | 59 | `kodokunohate.usm` | `pv_085_lp.adx` | 0 |
| ローリンガール | 60 | `rolling_girl.usm` | `pv_091_lp.adx` | 1 |
| 透明水彩 | 61 | `toumei_suisai.usm` | `pv_210_lp.adx` | 2 |

The pack also contains `music_17_01.png` and `verificationFile.dat`. Its thumbnail is `Thum_17/thum_17.png`, and its registered pack name is `Rock_Pack01`.

**The files supplied for the latest discussion were a mixed sample, not a complete pack 17:** `thum_17.png` belongs to pack 17, while `music_11_01.png` and the supplied `verificationFile.dat` correspond to pack 11 (`double_lariat`, `from_y_to_y`, `hello_planet`, with preview IDs 046, 057, and 083). That pack 11 manifest must not be used unchanged as a pack 17 or 18 manifest. The table above comes from the earlier pack 17 inventory and registration inspection, not solely from those three supplied files.

### 6.3 Song registration is another required layer to investigate

The inspected `MikuFlick2.dat` uses NSKeyedArchiver. Its root object’s `m_MusicPackArray` contains pack objects, and each pack’s `m_MusicDataArray` contains song objects. This is not a plain JSON catalog that can be generated from filenames alone; writing it back must preserve archive object references, including the nested archives.

| Level | Observed fields | Questions for a new pack |
|---|---|---|
| Pack identity | `m_PackID`, `m_PackName`, `m_Date` | Will the game accept the new identifier, name, and date? |
| Pack state | `m_IsBuy`, `m_IsInstalled` | Do recorded states agree with available resources? Setting “installed” does not install files |
| Pack songs | `m_MusicDataArray` | How are the three song objects organized and referenced? |
| Song identity | `m_Index`, `m_Title`, `m_Artist` | Global unique song indices and song-selection text |
| Resource mapping | `m_MovieDataFileName`, `m_PreSoundFileName`, `m_ArtWorkFileName`, `m_ArtWorkIndex` | USM, preview, artwork basename, and atlas index; inspected resource references omit file extensions |
| Difficulty and progress | `m_DifficultNum0` through `m_DifficultNum4`, plus unlock/score records | Complete difficulty display and initial state; changing a displayed difficulty value does not generate a chart |

`m_LogoFileName` is null in the sample. Its name alone does not establish that it is the entry point for `music_18_01.png`. The game may also rebuild the catalog from an internal source at startup, so persistence must be checked after restarting.

The 22 parsed pack IDs are **0–17 and 96–99**, and global song indices span **0–73**. Pack ID 18 is unused in this sample, but that does not prove the engine accepts it. Packs 96–99 already occupy song indices 62–73: **do not start a new pack at index 62**. Indices **74, 75, and 76** are candidates for this sample only; check the target device’s latest save before creating records.

### 6.4 Suggested validation stages

1. **Edit an existing song.** Continue isolated tests of note enablement, direction, and timing in a working pack. Only disabling one EASY note has been demonstrated so far.
2. **Test new-pack registration.** Copy a working three-song template under new resource names, add candidate pack/song records, and update the manifest. Initially keep the media and event content unchanged. Check whether the pack appears, whether titles/artwork/previews map correctly, and whether MV playback and all difficulties can be entered. This stage has not been completed.
3. **Test persistence.** Fully quit and restart the game. Check whether the pack remains available and whether catalog rebuilding, overwritten records, or identifier limits cause problems.
4. **Introduce custom content gradually.** Once registration works, separately test replacement media, full-line lyrics, per-character notes, and interlude events. Change one factor at a time.

The best-supported chart-design model is a **shared per-character timeline with separate operation codes for five difficulties**, alongside lyric-line and interlude events controlling display/input sections. Whether additional metadata is needed for total note counts, scoring, combos, progress, synchronization, and results after adding events remains unresolved. New packs may also be constrained by hardcoded catalogs, identifier limits, or other directory files. A folder on disk is not proof of a working custom pack.
## 7. Encryption, verification, and editing workflow

The sample’s lyrics are directly readable and its media can be decoded without supplying a key. No decryption step was needed in these experiments. USM supports optional encryption/masking, so files from other games or sources may still require a key.

The inspected `verificationFile.dat` is a filename/SHA-1 manifest, not an encryption key or a digital signature. Update the affected digest after editing a USM. Whether additional runtime checks exist must be established on a device.

For lyric-only edits to this sample:

1. Back up the original USM and its matching `verificationFile.dat`.
2. Parse the CUE/UTF table and export lyrics while preserving all original events and timestamps.
3. Edit the translation, retain the parameter prefix, and check the **UTF-8 byte length**, not just the character count.
4. If the replacement fits the original allocation, overwrite in place, zero-pad, and update the relevant length field. In this sample the field is big-endian and the original parameter length includes the terminating `00` byte.
5. Parse the modified table again, confirm that only intended lyrics changed, and compare media, other events, and indexes.
6. Update that USM’s SHA-1 in the manifest.
7. Fully quit the game, replace the USM and manifest together in the original pack location, then test MV playback with lyric display enabled.

Longer text, inserted/deleted events, or replacement media can require rebuilding affected UTF offsets, USM chunk lengths/alignment, directory information, and seek indexes. Do not insert arbitrary bytes using a text editor or place an LRC beside the USM expecting it to replace the embedded event table.

For the single-note test in section 4, the replacement occupies exactly the same number of bytes, so no length or offset changes are needed. Binary comparison must still confirm that only the intended byte changed, and the external manifest must be updated.

## 8. Windows and macOS tools

| Task | Windows | macOS |
|---|---|---|
| Inspect/extract CRI containers | CriStudio / CriCodecs | CriStudio / CriCodecs |
| Inspect media streams and extract audio/video | FFmpeg / FFprobe | FFmpeg / FFprobe |
| Inspect and overwrite binary bytes | HxD | Hex Fiend |
| Edit exported lyrics and JSON | VS Code or another text editor | VS Code or another text editor |
| Parse and write game events precisely | Python scripts | Python scripts |

These experiments used Python to parse and patch data in place. General-purpose tools can assist inspection, but CriStudio’s save workflow has not been verified to preserve this game’s custom CUE events completely. The tested FFmpeg build can read USM and extract its media, but has no USM output muxer; an ordinary transcode is not a replacement for rebuilding a playable game resource. Use overwrite mode in a hex editor and understand the relevant length and offset fields.

To calculate the modified file’s SHA-1:

Windows PowerShell:

```powershell
Get-FileHash "hello_planet.usm" -Algorithm SHA1
```

macOS Terminal:

```bash
shasum -a 1 hello_planet.usm
```

Write the resulting 40-digit hexadecimal digest to the `hello_planet.usm` entry in the manifest, preserving entries for unchanged files.

## 9. References and evidence boundaries

- [Chinese source research and device-test record](https://github.com/HachiMiku39/mikuflick_chinese_localization/blob/9eb9a65079c981342acc6abed55c0677ae227f1d/README.md#usm-解析与歌词修改实验miku-flick02)
- [CRI official Sofdec encoder documentation: multiple audio streams and cuepoints](https://game.criware.jp/manual/native/sofdec2/latest/usr_console_encoder_main.html)
- [PyCriCodecsEx: USM chunk definitions](https://mos9527.com/PyCriCodecsEx/_modules/PyCriCodecsEx/chunk.html)
- [PyCriCodecsEx: UTF table parser](https://mos9527.com/PyCriCodecsEx/_modules/PyCriCodecsEx/utf.html)
- [CriStudio / CriCodecs](https://github.com/Youjose/CriCodecs)
- [HxD](https://mh-nexus.de/en/hxd/) · [Hex Fiend](https://hexfiend.com/)

The external references explain the general container format. Sample statistics, parameters, and registration fields come from file inspection; game behavior comes from the maintainer’s observations and the one-byte EASY edit. Other direction/timing edits, special-event meanings, and custom-pack registration remain explicitly unverified. This document is a research record, not a complete format specification or a finished chart editor.

# MikuFlick2 1.1.5 Game Mechanics Reverse-Engineering Notes

> Based on static analysis of the iOS version of `MikuFlick2` 1.1.5 (ARMv7) in Ghidra.  
> This document records only mechanics that have been confirmed from the binary or can be inferred with high confidence from the current analysis. Anything not fully traced is explicitly marked.

---

## 0. Target Binary and Confidence Labels

### Binary Information

- App: `MikuFlick2`
- Version: 1.1.5
- Main executable: `Payload/MikuFlick2.app/MikuFlick2`
- Mach-O: 32-bit ARMv7
- Universal/Fat binary: No, `armv7` only
- FairPlay:
  - `LC_ENCRYPTION_INFO`
  - `cryptid 0`
  - The code section in this sample is decrypted and can be analyzed directly
- Main implementation technologies:
  - Objective-C
  - C / C++
  - CRI Middleware (CRI Mana / CRI Atom)
- Reverse-engineering tool: Ghidra 12.1.4

### Confidence Labels

- **[Confirmed]**: Directly verified from functions, tables, or control flow.
- **[High-confidence inference]**: The relationship is very clear from code, but some surrounding logic has not been fully traced.
- **[Not fully investigated]**: Analysis has not yet been completed.

---

# 1. Main Game Loop

Core entry point:

```text
-[SceneGame_Exec]
```

Confirmed per-frame execution order:

```text
SceneGame_Exec
│
├─ Update elapsed/play time
├─ MikuFlickCriManager_Exec
├─ NoteManager_exec
├─ ReplayManager_Replay       (Replay Mode only)
├─ TouchManager_exec
├─ StageManager_exec
├─ WindowManager_exec
└─ EffectManager_exec
```

## 1.1 Note Update Order

`+[NoteManager_exec]` iterates over the current Note collection and, for each Note:

```text
note.exec()
setNextTarget(0)
adjust Z position depending on active state
```

It then iterates over the Notes again and marks the first active Note as:

```text
setNextTarget(1)
```

Therefore, `NextTarget` is effectively the current Note that should receive priority highlighting / guidance.

---

# 2. Timing System: Judgement Is Not Tied to Render Frames

This is one of the most important findings in the judgement system.

## 2.1 Playback Counter

`+[MikuFlickCriManager_Exec]`:

```c
playTime = CriManager::GetPlayTime();
PlayCnt = ceil(playTime * 30.0f);
```

Global playback counter:

```text
DAT_00164340
```

`+[MikuFlickCriManager_GetPlayCnt]` simply returns:

```c
return DAT_00164340;
```

## 2.2 Per-Note Time

Inside `-[NoteNormal_exec]`:

```c
m_Cnt = MikuFlickCriManager_GetPlayCnt() - m_BaseTime;
```

So Notes do not run on:

```text
one rendered frame → m_Cnt++
```

Instead:

```text
CRI audio playback time
        ↓
converted to a 30 Hz PlayCnt
        ↓
minus the Note's BaseTime
        ↓
current Note m_Cnt
```

### Conclusion

**[Confirmed]**

```text
1 tick ≈ 1 / 30 s ≈ 33.33 ms
```

The gameplay clock is therefore tied to the audio clock, not the display refresh rate.

This matters on 2012-era mobile hardware: visual frame drops should not directly cause judgement timing to drift.

## 2.3 CRI Time Source

Call chain:

```text
MikuFlickCriManager_Exec
↓
CriManager::GetPlayTime
↓
_criManaPlayer_GetTime
↓
CriMvEasyPlayer::GetTime
```

`CriManager::GetPlayTime()` ultimately returns:

```text
timeValue / timeBase
```

`CriMvEasyPlayer::GetTime()` also contains timing correction logic involving:

```text
0.0333667
```

which is approximately the frame interval of 29.97 fps.

---

# 3. Judgement Enumeration

From `NoteNormal_checkResult::::`, the result screen, and `StatusData` bookkeeping, the internal judgement IDs are fully confirmed:

| Internal Value | Judgement |
|---:|---|
| 0 | Internal invalid / no-result state |
| 1 | WORST |
| 2 | SAD |
| 3 | SAFE |
| 4 | FINE |
| 5 | COOL |

The result screen directly reads:

```text
GetTmpNoteResult(5) → Cool
GetTmpNoteResult(4) → Fine
GetTmpNoteResult(3) → Safe
GetTmpNoteResult(2) → Sad
GetTmpNoteResult(1) → Worst
```

Value `0` is not shown as one of the five visible judgement categories, but internal arrays and high-score storage keep six slots (`0~5`).

---

# 4. Normal Note Judgement Windows

Relevant tables:

```text
s_tblNoteNormal_Front @ 0x00139488
s_tblNoteNormal_Back  @ 0x001394B4
```

Definitions:

```text
Front = player input earlier than JustFrame
Back  = player input later than JustFrame
```

The code calculates:

```c
delta = JustFrame - TouchFrame;
```

Therefore:

```text
delta > 0 → early
delta < 0 → late
```

## 4.1 Early Judgement Table

`s_tblNoteNormal_Front`:

| Early Offset | Judgement | Approx. Time |
|---:|---|---:|
| 0 tick | COOL | 0 ms |
| 1 tick | COOL | 33 ms |
| 2 ticks | COOL | 67 ms |
| 3 ticks | FINE | 100 ms |
| 4 ticks | FINE | 133 ms |
| 5 ticks | FINE | 167 ms |
| 6 ticks | FINE | 200 ms |
| 7 ticks | SAFE | 233 ms |
| 8 ticks | SAFE | 267 ms |
| 9 ticks | SAD | 300 ms |
| 10 ticks | SAD | 333 ms |

## 4.2 Late Judgement Table

`s_tblNoteNormal_Back`:

| Late Offset | Judgement | Approx. Time |
|---:|---|---:|
| 0 tick | COOL | 0 ms |
| 1 tick | COOL | 33 ms |
| 2 ticks | COOL | 67 ms |
| 3 ticks | COOL | 100 ms |
| 4 ticks | FINE | 133 ms |
| 5 ticks | FINE | 167 ms |
| 6 ticks | FINE | 200 ms |
| 7 ticks | FINE | 233 ms |
| 8 ticks | SAFE | 267 ms |
| 9 ticks | SAFE | 300 ms |
| 10 ticks | SAD | 333 ms |

### Asymmetry

**[Confirmed]**

Late inputs receive roughly one extra tick of leniency compared with early inputs:

```text
Early COOL: 0~2
Late  COOL: 0~3

Early FINE: 3~6
Late  FINE: 4~7

Early SAFE: 7~8
Late  SAFE: 8~9
```

This is a very obvious touch-input compensation design.

---

# 5. TouchDown / TouchUp Combination Logic

A normal Flick Note does not judge only one touch timestamp.

`-[NoteNormal_checkResult::::]` calculates both:

```text
JustFrame - TouchDownFrame
JustFrame - TouchUpFrame
```

Both values are checked against the normal Note Front / Back judgement tables.

## 5.1 Basic Combination

**[Confirmed]**

When both TouchDown and TouchUp are within valid judgement ranges, the game generally takes the worse of the two results:

```text
TouchDown judgement
TouchUp judgement
       ↓
take the worse result
```

However, there is special handling for early touch-hold behavior.

## 5.2 Early Hold Leniency

If TouchDown happens early and the combined result would otherwise fall below SAFE:

```c
if (result < SAFE && TouchDown is before JustFrame)
    result = SAFE;
```

There is also a path where TouchDown is much earlier than the normal judgement window, but TouchUp is still timed correctly:

```text
final result is capped at SAFE
```

### Design Meaning

**[High-confidence inference]**

This allows the player to:

```text
place a finger on the screen early
↓
wait for the beat
↓
perform the Flick / release at the correct time
```

but prevents that strategy from easily earning FINE or COOL.

This is clearly a touch-oriented "pre-touch leniency" mechanism.

---

# 6. Flick Direction and BoardType

## 6.1 Wrong Flick Direction

The code compares:

```text
m_FlickResult
against
the expected Flick direction for the input
```

When the direction does not match:

```text
maximum judgement is capped at SAFE (3)
```

So an incorrect Flick direction cannot receive FINE or COOL.

## 6.2 BoardType Mismatch

If the Note's `m_BoardType` does not match the input Board:

```text
result = SAD (2)
```

This is a direct downgrade path.

## 6.3 Break The Limit Special Handling

When `Difficulty == 4` (Break The Limit), the normal judgement function contains a special branch:

```c
if (difficulty == 4) {
    if (result == SAD)
        result = 0;
}
```

This branch also skips the normal:

```text
NoteManager_AddTensionGauge
```

call.

Likewise, the normal `ResetCombo()` path used for low judgements is skipped in BTL.

### Current Conclusion

**[Confirmed code behavior / gameplay interpretation incomplete]**

BTL is clearly not just a normal difficulty with tighter numbers. It has separate handling for:

```text
Gauge
SAD
Combo Reset
```

The full BTL rules have not yet been fully traced.

---

# 7. WORST and Note Lifetime

`NoteNormal_exec` determines whether a Note is within its active judgement region based on `m_JustFrame`.

For normal difficulties:

```text
InsideJudge:
roughly JustFrame - 11
through
JustFrame + 11
```

However, the actual judgement tables only define normal results for `0~10 ticks`.

That leaves a one-tick edge zone that behaves as a buffer / no-valid-result area.

## 7.1 Automatic WORST

On normal difficulties, once:

```text
m_Cnt >= JustFrame + 12
```

the Note is deactivated and the game records:

```text
WORST
Gauge penalty
ResetCombo
miss/failure effect
```

This is approximately:

```text
12 ticks late ≈ 400 ms
```

## 7.2 Break The Limit

Table:

```text
s_tblDifficultEndTimeOfs
```

Values:

| Difficulty | Offset |
|---|---:|
| Easy | 0 |
| Normal | 0 |
| Hard | 0 |
| Extreme | 0 |
| Break The Limit | -2 |

Therefore, the BTL Note tail window is shortened by about:

```text
2 ticks ≈ 66.7 ms
```

---

# 8. Base Score

Table:

```text
s_tblAddScore @ 0x001394E4
```

Primary score group:

| Judgement | Base Stage Score |
|---|---:|
| Invalid / none | 0 |
| WORST | 0 |
| SAD | 30 |
| SAFE | 50 |
| FINE | 150 |
| COOL | 300 |

Therefore:

```text
COOL  = 300
FINE  = 150
SAFE  = 50
SAD   = 30
WORST = 0
```

## 8.1 Secondary Score Group

The same table also contains:

```text
0, 0, 30, 50, 150, 250
```

This group is selected through a `local_28` branch.

In the currently traced normal Flick-direction mismatch path, high judgements are already capped at SAFE, so the `150/250` entries are not reached in the observed flow.

**[Not fully investigated]**

This may correspond to another reachable condition or leftover logic. It is intentionally left unexplained for now.

---

# 9. Combo

## 9.1 Combo Continuation

**[Confirmed]**

```text
COOL / FINE → AddCombo
SAFE / SAD / WORST → ResetCombo
```

Equivalent rule:

```text
result >= 4 → continue Combo
result < 4  → break Combo
```

This applies to the normal difficulty path.

BTL skips the normal ResetCombo branch in this function.

## 9.2 Max Combo

Whenever Combo increases:

```text
if current Combo > MaxCombo
→ update MaxCombo
```

The result screen also reads `GetMaxCombo()`.

---

# 10. Combo Bonus Formula

After a judgement that continues Combo, the game calculates an additional Combo Score.

The compiler's magic-number division is equivalent to:

```text
floor((Combo + 5) / 10) × 50
```

with a hard cap of:

```text
500 points
```

Approximate behavior:

| Current Combo | Per-Note Combo Bonus |
|---:|---:|
| 1~4 | 0 |
| 5~14 | 50 |
| 15~24 | 100 |
| 25~34 | 150 |
| 35~44 | 200 |
| 45~54 | 250 |
| 55~64 | 300 |
| 65~74 | 350 |
| 75~84 | 400 |
| 85~94 | 450 |
| ≥95 | 500 |

The bonus accumulates into:

```text
TmpComboScore
```

The result screen total uses:

```text
TmpStageScore + TmpComboScore
```

---

# 11. Tension Gauge

Relevant data:

```text
s_tblTensionGauge @ 0x00138F4C
DAT_00163088       current Gauge
```

## 11.1 Initial Value and Maximum

Inside:

```text
+[NoteManager_Initialize]
```

the game sets:

```c
DAT_00163088 = 0x43000000;
```

Interpreted as IEEE-754 float:

```text
128.0
```

Inside `AddTensionGauge`, the maximum is:

```text
0x43800000 = 256.0
```

Therefore:

```text
Initial Gauge = 128
Maximum Gauge = 256
Starting Gauge = 50%
```

This matches the in-game UI, which starts at half gauge.

---

# 12. Gauge Judgement Weights

`s_tblTensionGauge`:

| Judgement | Weight |
|---|---:|
| Invalid / none | 0 |
| WORST | -10 |
| SAD | -5 |
| SAFE | 0 |
| FINE | +2 |
| COOL | +2 |

Actual Gauge changes are not direct integer additions.

## 12.1 Note-Count Normalization

`+[NoteManager_AddTensionGauge:]` calculates a coefficient based on the total number of Notes:

```text
GaugeCoefficient ≈ 64 / TotalNotes + 0.01
```

Actual update:

```text
Gauge += JudgeWeight × GaugeCoefficient
```

Then Gauge is clamped to:

```text
256
```

Therefore:

```text
COOL  +2 × coefficient
FINE  +2 × coefficient
SAFE   0
SAD   -5 × coefficient
WORST -10 × coefficient
```

### Design Meaning

Charts with fewer Notes:

```text
each judgement has a larger impact on Gauge
```

Charts with more Notes:

```text
each judgement has a smaller impact
```

This normalizes survival difficulty across charts of different lengths / Note counts.

---

# 13. Game Over Conditions

`AddTensionGauge` confirms two independent fail conditions.

## 13.1 Gauge Reaches Zero

If:

```text
Gauge <= 0
```

then:

```c
DAT_00163088 = 0.0;
StatusData_SetGameOver(1);
SceneManager_NextScene(8);
```

So:

```text
Gauge <= 0
→ Game Over
→ Gauge forced to 0
→ switch to Scene 8
```

## 13.2 Early Failure When 50% Success Is No Longer Mathematically Possible

The game calculates:

```text
successful judgements = SAFE + FINE + COOL
remaining Notes = TotalNotes - already judged Notes
```

It then evaluates:

```text
(successful judgements + remaining Notes) / TotalNotes
```

This represents:

> The maximum possible SAFE-or-better success rate if every remaining Note is hit successfully.

If that ratio becomes:

```text
< 50%
```

the game immediately triggers Game Over.

Therefore, even if Gauge is still above zero, the run ends once:

```text
it is mathematically impossible to finish with at least 50% SAFE-or-better
```

---

# 14. Gauge and BGM Volume

Gauge also directly affects the main audio volume.

The code is equivalent to approximately:

```text
volumeFactor = min(1.0, Gauge / 128 + 0.35)
```

Then:

```text
MainAudioVolume = user BGM volume × volumeFactor
```

Therefore:

- High Gauge: audio remains at 100%
- Lower Gauge: BGM becomes quieter
- Near zero Gauge: about 35% base multiplier remains
- On Game Over, Gauge is forced to zero

This is a classic danger-state audio feedback system.

---

# 15. Crimax System

Crimax is not a simple global mode toggle. It is applied to specific Notes through chart CuePoints.

Entry point:

```text
CuePointFunc_Crimax
```

## 15.1 Difficulty-Specific CuePoint Flag

Logic:

```text
difficulty = GetDifficulty()

read one character from:
CuePoint[difficulty + 1]
↓
convert with intValue
```

If the value is nonzero:

```text
GetLastCrimaxEnableNote()
↓
setCrimaxMode(1)
```

This means a single Crimax CuePoint can independently enable or disable Crimax for:

```text
Easy
Normal
Hard
Extreme
BTL
```

---

# 16. Crimax Target Note Selection

`+[NoteManager_GetLastCrimaxEnableNote]`:

```text
start from the end of the current Note list
↓
check up to the most recent 4 Notes
↓
return the first Note where isCrimaxEnable == 1
```

So the CuePoint does not need to align exactly with one Note. It can search backward across the most recent four Notes.

## 16.1 isCrimaxEnable

`-[NoteNormal_isCrimaxEnable]` effectively checks:

```text
m_OptCharIdx < 0
```

Therefore:

```text
m_OptCharIdx < 0  → Crimax allowed
m_OptCharIdx >= 0 → Crimax not allowed
```

---

# 17. Crimax Rainbow Effect

Table:

```text
s_tblRainbow @ 0x00139310
```

Size:

```text
84 floats
= 28 RGB entries
= 3 × float32 per color
```

The color values are stored in:

```text
0~255
```

and divided by:

```text
255.0
```

during rendering to produce normalized `0.0~1.0` color values.

## 17.1 Rainbow Activation Conditions

Inside `-[NoteNormal_render]`, all of the following must be true:

```text
m_OptCharIdx < 0
m_Active != 0
m_CrimaxMode != 0
Combo >= 100
```

Only then does the Note use rainbow rendering.

Otherwise it uses normal Note rendering.

## 17.2 Rainbow Index

The index is equivalent to:

```text
(m_CrimaxCnt + 10) % 28
```

Each RGB entry occupies:

```text
12 bytes
```

`m_CrimaxCnt` cycles through:

```text
0~27
```

during Note updates.

### Conclusion

The Crimax rainbow is not just decorative:

```text
Crimax flag
+
active Note
+
Combo ≥ 100
↓
rainbow Note
```

It is a direct visual indicator of the 100+ Combo Crimax bonus state.

---

# 18. Crimax Bonus Score

After a normal Note judgement, the code checks:

```text
m_CrimaxMode != 0
and
result > 4
```

Since the maximum judgement value is `5`, this means:

```text
COOL only
```

It then checks:

```text
Combo >= 100
```

If true:

```text
TmpStageScore +200
```

and triggers:

```text
StartSpScoreEffect
```

Therefore:

```text
Crimax Note
+ COOL
+ Combo ≥ 100
= +200 Stage Score
```

FINE does not receive this extra 200-point bonus.

---

# 19. ClearCrimax

`+[NoteManager_ClearCrimax]` does:

```text
iterate over every current Note
↓
setCrimaxMode(0)
```

So it clears Crimax from the entire active Note collection, not only one Note.

**[Not fully investigated]**

The real gameplay call site for `ClearCrimax` has not yet been reliably identified. One direct Ghidra XREF was confirmed to be a false reference inside CRI Atom code.

Therefore, the exact full-flow condition that ends Crimax has not yet been traced.

---

# 20. Interlude: Intermission Mini-Game

Interlude is confirmed to be the single-button rhythm mini-game played during instrumental/intermission sections.

CuePoint entry:

```text
CuePointFunc_Interlude
```

Like Crimax, it:

```text
gets current Difficulty
↓
reads CuePoint[difficulty + 1]
↓
converts it to int
↓
dispatches through a function table
```

## 20.1 Interlude Type Table

```text
s_tblInterludeType
```

Contents:

| Type | Function |
|---:|---|
| 0 | `InterludeType_None` |
| 1 | `InterludeType_Normal` |
| 2 | `InterludeType_FadeOut` |

---

# 21. Interlude Start and End

## 21.1 Normal

`InterludeType_Normal()`:

```c
WindowManager_StartInterludeMode();
NoteManager_AddNote::::(1, 0, 1, 0);
```

Meaning:

```text
enter Interlude Mode
↓
spawn one NoteInterlude
```

## 21.2 FadeOut

`InterludeType_FadeOut()`:

```c
WindowManager_EndInterludeMode();
```

So this type is effectively the end marker for an Interlude section.

---

# 22. NoteInterlude Judgement

`NoteInterlude` has a full set of methods:

```text
exec
render
setTouchDown
setTouchUp
checkResult::::
isJustFrame
```

but it does not use the normal Flick-direction gameplay. It is a simple timing-only input.

## 22.1 Timing Window

`NoteInterlude_checkResult::::` directly reuses:

```text
s_tblNoteNormal_Front
s_tblNoteNormal_Back
```

Therefore, Interlude timing uses the same:

```text
30 Hz tick
COOL / FINE / SAFE / SAD
```

timing windows as normal Notes.

## 22.2 Success Condition

An Interlude input counts as successful only when:

```text
result > 3
```

which means:

```text
FINE
or
COOL
```

SAFE / SAD do not count as Interlude success.

This matches the actual gameplay:

> During an interlude, only one button appears, and the player simply taps the Note on time.

---

# 23. Interlude Scoring

Each successful Interlude input adds:

```text
TmpInterludeCnt +1
TotalInterludeSuccess +1
TmpStageScore +1
```

If, for the current song/difficulty:

```text
TmpInterludeCnt >= TotalInterlude
```

meaning all Interlude Notes were hit successfully, the game adds:

```text
TmpStageScore +39
```

and also triggers:

```text
special sound effect
StartSpScoreEffect(39)
EffectManager_AddEffect2D(..., 7, ...)
```

Therefore, the final successful Interlude Note effectively awards:

```text
base +1
all-success bonus +39
total +40
```

---

# 24. Difficulty Enumeration

The result screen confirms:

| Internal Value | Difficulty |
|---:|---|
| 0 | Easy |
| 1 | Normal |
| 2 | Hard |
| 3 | Extreme |
| 4 | Break The Limit |

BTL has special handling in normal Note judgement, Gauge, Combo Reset, and Note lifetime.

---

# 25. Result Screen and Clear Rank

The result screen reads:

```text
Total Score
Stage Score
Combo Bonus
Max Combo
Total Notes
Cool
Fine
Safe
Sad
Worst
Interlude
Difficulty
```

Total score is:

```text
TmpStageScore + TmpComboScore
```

## 25.1 Preliminary Rank Rules

The result screen clearly uses:

```text
COOL + FINE + SAFE
```

relative to `TotalNotes` when determining Rank / Clear state.

The code contains thresholds including:

```text
70%
80%
95%
100%
```

In addition:

```text
GetMaxCombo() == TotalNotes
```

causes:

```text
SetPerfectClear(...)
```

**[Not fully investigated]**

The complete mapping between internal Rank values `0~6` and the exact in-game rank names has not yet been fully reconstructed, so this document does not assign names prematurely.

---

# 26. Simplified Model for a Modern 64-bit Reimplementation

If the goal is not to reproduce every line of legacy code but to rebuild the core behavior for a modern 64-bit version, the currently confirmed mechanics can be simplified as follows.

## 26.1 Timing

```text
playTick = ceil(audioPlayTimeSeconds × 30)
noteTick = playTick - note.baseTime
```

## 26.2 Judgement

```text
read TouchDown / TouchUp tick
↓
lookup Front / Back judgement table
↓
combine the two timing results
↓
apply early-hold correction
↓
check Flick direction
↓
check BoardType
↓
produce judgement enum 0~5
```

## 26.3 Result Enum

```text
5 COOL
4 FINE
3 SAFE
2 SAD
1 WORST
0 INVALID / NONE
```

## 26.4 Combo

```text
COOL/FINE → combo++
SAFE/SAD/WORST → combo reset
```

BTL is an exception.

## 26.5 Score

```text
COOL 300
FINE 150
SAFE 50
SAD 30
WORST 0

+ Combo Bonus
+ Crimax Bonus
+ Interlude Bonus
```

## 26.6 Gauge

```text
Start = 128
Max = 256

weight:
COOL  +2
FINE  +2
SAFE   0
SAD   -5
WORST -10

coefficient = 64 / TotalNotes + 0.01
```

## 26.7 Fail Conditions

```text
Gauge <= 0
OR
maximum theoretically achievable SAFE-or-better rate < 50%
→ Game Over
```

## 26.8 Crimax

```text
CuePoint
↓
find Crimax-enabled Note among the most recent 4 Notes
↓
m_CrimaxMode = 1
↓
Combo >= 100
↓
Rainbow

If the judgement is also COOL:
+200 Stage Score
```

## 26.9 Interlude

```text
Interlude Cue
↓
StartInterludeMode
↓
spawn NoteInterlude
↓
single-button timing input
↓
COOL/FINE = success
↓
each success +1
↓
all-success bonus +39
```

---

# 27. Areas Not Yet Fully Investigated

The following systems have visible entry points but have not yet been fully reverse-engineered:

- Complete Break The Limit rules
- Exact Clear Rank names and full threshold mapping
- Real call site / end condition for `ClearCrimax`
- Purpose of the second `s_tblNoteNormal_Front / Back` pair
- Reachability and meaning of the second `s_tblAddScore` high-judgement values
- Exact Replay playback behavior
- `NoteThrow`
- `NoteArrow`
- `NoteWait`
- `NoteInterlude.exec`
- Full `StageManager` state machine
- `WindowGameWindow` HUD rendering details
- Telop / SmallTelop
- Lyrics
- FadeIn / FadeOut
- SetBPM / SetDelay
- Chart file format and Note generation format
- Full TouchManager Flick gesture recognition algorithm
- Complete CRI audio/video ↔ chart synchronization correction logic

---

# 28. Confirmed Ghidra Symbols Quick Reference

```text
SceneGame_Exec
NoteManager_exec
NoteNormal_exec
NoteNormal_checkResult::::
NoteNormal_render
NoteNormal_isCrimaxEnable
NoteObjBase_isInsideJudgeFrame
NoteObjBase_setCrimaxMode:
MikuFlickCriManager_Exec
MikuFlickCriManager_GetPlayCnt
CriManager::GetPlayTime
CriMvEasyPlayer::GetTime
StatusData_AddFlickTypeNum:
NoteManager_AddTensionGauge:
NoteManager_GetLastCrimaxEnableNote
NoteManager_ClearCrimax
CuePointFunc_Crimax
CuePointFunc_Interlude
InterludeType_Normal
InterludeType_FadeOut
NoteInterlude_checkResult::::
WindowResult_loadTexture
```

Important tables:

```text
s_tblTensionGauge          @ 0x00138F4C
s_tblDifficultEndTimeOfs   @ 0x001392FC
s_tblRainbow               @ 0x00139310
s_tblNoteNormal_Front      @ 0x00139488
s_tblNoteNormal_Back       @ 0x001394B4
s_tblAddScore              @ 0x001394E4
```

Important globals:

```text
DAT_00163088 → Tension Gauge
DAT_00164340 → CRI-derived PlayCnt
```

---

# 29. Summary

The core mechanics of MikuFlick2 can currently be summarized as:

```text
CRI audio clock
↓
30 Hz gameplay tick
↓
TouchDown / TouchUp dual-timestamp judgement
↓
Flick direction and Board region correction
↓
COOL / FINE / SAFE / SAD / WORST
↓
Score + Combo + Gauge
↓
special systems such as Crimax / Interlude
```

By modern rhythm-game standards, the judgement granularity is coarse. However, the implementation itself is not simplistic.

The current reverse-engineering results show deliberate handling for:

- audio-clock synchronization
- asymmetric early/late timing leniency
- early touch-hold behavior before a Flick
- direction-mismatch downgrade
- Gauge normalization by chart Note count
- Gauge-linked BGM volume
- early failure when clearing becomes mathematically impossible
- Crimax 100-Combo rainbow feedback
- the single-button Interlude bonus mini-game

For a 2012 touchscreen Flick rhythm game, this is clearly a system tuned around practical touch behavior rather than a naive "timestamp difference → judgement" implementation.

---

*Prepared: 2026-10-05*  
*Sample: MikuFlick2 1.1.5 / ARMv7 / cryptid 0*  
*Status: Reverse engineering in progress*
