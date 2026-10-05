# Miku Flick/02 Custom Chart & Music-Pack Research

> Reverse-engineering notes for **MikuFlick2 1.1.5** focused on custom charts, CRIWARE USM events, song-pack registration, and the minimum game mechanics needed to author compatible content.
>
> This repository is intentionally narrower than the main localization/reverse-engineering repository. For Chinese localization, old-iOS installation notes, and the broader game-mechanics overview, see [`mikuflick_chinese_localization`](https://github.com/HachiMiku39/mikuflick_chinese_localization).

## Project status

The project is still experimental, but several important pieces are now confirmed on real hardware and through Ghidra analysis.

| Area | Status |
|---|---|
| Chinese full-line lyrics in MV mode | **Confirmed on device** |
| Five difficulty slots in per-character chart events | **Confirmed** |
| Tap / up / right / down / left operation codes | **Confirmed for tested records** |
| Disable one existing Note on only one difficulty | **Confirmed on device** |
| Interlude single-button events | **Partially confirmed** |
| Original judgement / score / Gauge behavior | **Largely reconstructed** |
| `MikuFlick2.dat` music-pack catalog | **Confirmed** |
| `verificationFile.dat` SHA-1 enforcement during normal playback | **Not enforced in tested installed-pack case** |
| Arbitrary Note insertion / deletion with rebuilt USM tables | TODO |
| Timing / direction edits tested one by one | TODO |
| Fully custom Pack 18 registration | TODO |
| Fully custom song with new media + chart + artwork + save registration | TODO |

---

## 1. Research target

Primary binary:

```text
MikuFlick2 1.1.5
Mach-O ARMv7
cryptid 0
Ghidra 12.1.4
```

Primary chart sample:

```text
＊ハロー、プラネット。
hello_planet.usm
```

The current findings combine:

- static analysis of the decrypted ARMv7 executable
- parsing of CRIWARE USM / UTF / CUE data
- inspection of `MikuFlick2.dat`
- comparison with installed DLC resources
- real-device gameplay tests
- binary-diff experiments where one variable is changed at a time

Unknown behavior is left as **TODO / UNKNOWN** instead of being promoted from a guess into a format rule.

---

## 2. What is inside a song USM?

For the tested `hello_planet.usm`, the logical structure is:

```text
hello_planet.usm
├── CRID                       container / stream metadata
├── @SFV                       one MV video stream
├── @SFA channel 0             audio stream
├── @SFA channel 1             audio stream
├── @CUE                       lyrics, chart events, control events
└── @UTF / seek / alignment    supporting metadata
```

Parsed sample data:

| Item | Result |
|---|---|
| File size | 102,646,144 bytes |
| Video | MPEG-1, 320×480, 20 fps, 4,075 frames, about 203.75 s |
| Audio | 2 × ADX, 44.1 kHz stereo, about 204 s |
| CUE rows | 505 |
| Event fields | `name`, `time`, `cue_type`, `parameter` |
| Time base | `time_unit = 1000` |
| Full-line lyric events | 26 |
| Per-character chart records | 376 |

The audio source names `pv_083vo2.aif` and `pv_083ok2.aif` strongly suggest vocal / karaoke roles, but they are not explicit semantic labels in the container.

`@CUE` matters. A normal media-player or FFprobe stream listing can expose the media while completely missing the gameplay chart and lyric events.

---

## 3. CUE parameter model

In this sample, container-level `cue_type` is `0` for the inspected rows. The important discriminator is the first character of the CUE `parameter` string.

Observed prefix inventory:

| Prefix | Count | Current interpretation |
|---:|---:|---|
| `0` | 376 | per-character playable chart records |
| `3` | 26 | full lyric lines / music-symbol line |
| `4` | 37 | supplementary kana / long-vowel display data; incomplete semantics |
| `5` | 48 | Interlude states / single-button section events |
| `6` | 10 | special events, still UNKNOWN |
| `1` | 3 | section/display transition candidate |
| `2` | 3 | section/display transition candidate |
| `7` | 1 | initialization parameter candidate |
| `8` | 1 | initialization parameter candidate |

`8150` may be related to BPM 150, but that remains a hypothesis. A chart editor should preserve unknown events byte-for-byte until their behavior is isolated experimentally.

---

## 4. Per-character chart records

Prefix `0` records can be parsed as:

```text
0 + kana character + EASY / NORMAL / HARD / EXTREME / BTL operation tokens
```

The five difficulty slots are ordered:

```text
Easy
Normal
Hard
Extreme
Break The Limit
```

Confirmed operation tokens:

| Token | Operation |
|---:|---|
| `0` | no Note for this difficulty |
| `11` | tap |
| `2` | flick up |
| `3` | flick right |
| `4` | flick down |
| `5` | flick left |

`11` must be treated as one effective operation token when parsing these records. Do not split it into two difficulty slots.

### Japanese keypad mapping used by the game

| Key | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 0 |
|---|---|---|---|---|---|---|---|---|---|---|
| Kana row | あ | か | さ | た | な | は | ま | や | ら | わ |

Examples:

```text
ル → key 9
め → key 7
お → key 1
```

The operation token encodes the gesture, not the keypad row. For example, `タ`, `が`, and `さ` can all use the tap token `11` while mapping to different keypad keys.

---

## 5. Opening-line validation

Test line:

```text
シェルターのおと　ひとりめがさめた
```

Representative records:

| CUE time | Parameter | EASY | NORMAL | HARD | EXTREME | BTL |
|---:|---|---:|---:|---:|---:|---:|
| 6.340 | `0シ00005` | 0 | 0 | 0 | 0 | 5 |
| 6.540 | `0ル22222` | 2 | 2 | 2 | 2 | 2 |
| 7.340 | `0お00444` | 0 | 0 | 4 | 4 | 4 |
| 8.540 | `0め03333` | 0 | 3 | 3 | 3 | 3 |
| 8.940 | `0さ0001111` | 0 | 0 | 0 | 11 | 11 |
| 9.340 | `0た1111111111` | 11 | 11 | 11 | 11 | 11 |

Observed inputs match the parsed records across all five difficulties.

| Difficulty | Notes in the tested line | Observed input sequence |
|---|---|---|
| EASY | 2 | 9↑, tap 4 |
| NORMAL | 3 | 9↑, 7→, tap 4 |
| HARD | 4 | 9↑, 1↓, 7→, tap 4 |
| EXTREME | 6 | 9↑, 1↓, 4↓, 7→, tap 3, tap 4 |
| BTL | 14 | 3←, 9↑, tap 4, 5↓, 1↓, 4↓, 6←, 4↓, 9←, 7→, tap 2, tap 3, 7→, tap 4 |

Counting non-zero operation tokens across all prefix `0` records produces:

```text
Easy     65
Normal  107
Hard    161
Extreme 225
BTL     376
```

These are parsed per-character counts only. They do not include Interlude events and should not automatically be treated as final in-game total-note counts.

---

## 6. Confirmed one-byte chart edit

One experiment disabled only EASY's opening `ル` while leaving every other difficulty unchanged.

```text
time = 6540

before: 0ル22222
after:  0ル02222
```

Result on device:

- EASY loses that `ル` Note
- NORMAL / HARD / EXTREME / BTL retain it
- the rest of the USM remains unchanged

Binary details for the tested original file:

```text
offset: 0x5C6C
0x32 ('2') → 0x30 ('0')
```

SHA-1:

```text
original: 5c3a4a663c97528b34fe509b94b087a17800d2f5
modified: 7562bb0d3a39f829b4874c99f8fbd4f86b30e923
```

This proves difficulty-specific Note disabling. It does **not** yet prove that arbitrary event insertion, event deletion, timing edits, or new songs work without rebuilding additional USM metadata.

---

## 7. Chinese full-line lyric experiment

Prefix `3` events contain full lyric lines.

The tested modification:

- preserved timestamps and the `3` prefix
- replaced 25 Japanese lyric lines with Chinese
- preserved the music-symbol line
- did not re-encode video or audio
- kept every replacement within the original allocated byte area
- updated the affected length field and zero-padded the remaining bytes

Result:

> **Chinese full-line lyrics display correctly during MV playback when lyric display is enabled.**

This does not imply that per-character gameplay input has been converted to Chinese.

---

## 8. Interlude events

The game includes an Interlude mode where the 9-key keyboard disappears and the player taps a single rhythm target.

For the first tested Interlude on NORMAL, the game requires **8 taps**.

Relevant event patterns:

| Parameter | EASY / NORMAL / HARD / EXTREME / BTL | Current meaning |
|---|---|---|
| `501111` | `0 / 1 / 1 / 1 / 1` | ordinary tap candidate for NORMAL+ |
| `500111` | `0 / 0 / 1 / 1 / 1` | ordinary tap candidate for HARD+ |
| `500011` | `0 / 0 / 0 / 1 / 1` | ordinary tap candidate for EXTREME / BTL |
| `502222` | `0 / 2 / 2 / 2 / 2` | non-ordinary state, exact meaning still unknown |

The first Interlude has exactly eight NORMAL-slot `1` records, matching the eight taps observed on device.

Do not reuse the prefix `0` gesture meanings here. In prefix `5`, state `2` is **not established as flick up**.

From Ghidra, the runtime Interlude implementation is also known:

```text
CuePointFunc_Interlude
↓
InterludeType_Normal
↓
StartInterludeMode
↓
spawn NoteInterlude
```

`NoteInterlude_checkResult::::` reuses the normal timing tables. Only FINE / COOL count as successful Interlude hits.

Scoring:

```text
each successful Interlude hit: +1 Stage Score
all Interlude hits successful: additional +39
```

Therefore the final successful hit in an all-success Interlude contributes `+40` total.

---

## 9. Original timing model relevant to custom charts

The original game does not judge Notes using the screen refresh rate.

```text
playTick = ceil(audioPlayTimeSeconds × 30)
noteTick = playTick - note.baseTime
```

So:

```text
1 gameplay tick ≈ 33.33 ms
```

This is critical for any editor or modern reimplementation. A 60 Hz or 120 Hz renderer should still reproduce the original **30 Hz audio-clock-based judgement model**.

### Judgement IDs

| ID | Result |
|---:|---|
| 0 | NONE / INVALID |
| 1 | WORST |
| 2 | SAD |
| 3 | SAFE |
| 4 | FINE |
| 5 | COOL |

### Early window

| Offset | Result |
|---:|---|
| 0–2 ticks | COOL |
| 3–6 | FINE |
| 7–8 | SAFE |
| 9–10 | SAD |

### Late window

| Offset | Result |
|---:|---|
| 0–3 ticks | COOL |
| 4–7 | FINE |
| 8–9 | SAFE |
| 10 | SAD |

The late side is approximately one tick more permissive.

Normal Flick Notes combine both TouchDown and TouchUp timing. The game also contains an early-hold leniency path that can cap such input at SAFE rather than simply failing it.

---

## 10. Score model relevant to chart authoring

### Base Stage Score

| Judgement | Score |
|---|---:|
| COOL | 300 |
| FINE | 150 |
| SAFE | 50 |
| SAD | 30 |
| WORST | 0 |

### Combo

Normal difficulties:

```text
COOL / FINE → Combo +1
SAFE / SAD / WORST → reset Combo
```

Per-Note Combo Bonus after increasing Combo:

```text
min(500, floor((Combo + 5) / 10) × 50)
```

### Crimax

When all are true:

```text
Crimax Note
+ COOL
+ Combo >= 100
```

the game adds:

```text
+200 Stage Score
```

and renders the Note using the 28-entry rainbow table.

### Total Score

Result processing uses:

```text
Total Score = TmpStageScore + TmpComboScore
```

No additional result-screen multiplier or generic clear bonus has been found.

---

## 11. Tension Gauge and chart length

Normal difficulties start with:

```text
Gauge = 128
Max   = 256
```

Judgement weights:

| Result | Weight |
|---|---:|
| COOL | +2 |
| FINE | +2 |
| SAFE | 0 |
| SAD | -5 |
| WORST | -10 |

The per-chart coefficient is approximately:

```text
64 / TotalNotes + 0.01
```

So individual mistakes hurt more in short charts and less in long charts.

Game Over can occur when:

```text
Gauge <= 0
```

or when even hitting every remaining Note at SAFE-or-better can no longer produce at least a 50% success rate.

This matters when inserting or removing Notes: total-note bookkeeping can affect survival behavior even if the timing data itself is valid.

---

## 12. Break The Limit differences

BTL is not just a harder copy of the normal rules.

In the currently reconstructed `NoteNormal` path:

| BTL judgement | Stage Score | Combo behavior |
|---|---:|---|
| COOL | +300 | +1 and Combo Bonus |
| FINE | +150 | +1 and Combo Bonus |
| SAFE | +50 | preserved, no increment |
| SAD | converted to internal 0 | preserved |
| timeout / miss | +0 | preserved |

BTL also skips the normal Tension Gauge path.

That means custom BTL authoring must not assume normal-difficulty Combo or failure semantics.

---

## 13. Result Rank mapping

Original UI assets and result control flow now establish:

```text
Rank 0 → Perfect!
Rank 1 → S
Rank 2 → A
Rank 3 → B
Rank 4 → C
Rank 5 → D
Rank 6 → E
```

Conditions:

| Rank | Condition |
|---|---|
| Perfect! | all Notes are COOL |
| S | COOL + FINE = 100%, but not all COOL |
| A | COOL + FINE >= 95% |
| B | COOL + FINE >= 80% |
| C | COOL + FINE + SAFE >= 70% |
| D | below 70% SAFE-or-better and not Game Over |
| E | Game Over |

`PerfectClear` in persistent song data is separate from the Perfect rank. The result code sets it when `MaxCombo == TotalNotes`, so it behaves more like a full-combo flag.

---

## 14. Song-pack registration lives in `MikuFlick2.dat`

Simply creating a new `Mov_18` directory is not enough.

`MikuFlick2.dat` is an `NSKeyedArchiver`-based save/catalog structure containing:

```text
StatusData
└── m_MusicPackArray
    ├── MusicPack
    │   └── m_MusicDataArray
    │       ├── MusicData
    │       ├── MusicData
    │       └── MusicData
    └── ...
```

Relevant `MusicPack` fields:

```text
m_PackID
m_PackName
m_Date
m_IsBuy
m_IsInstalled
m_MusicDataArray
```

Relevant `MusicData` fields:

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
m_DifficultNum0 ... m_DifficultNum4
m_PreInstall
m_PVListEnable
m_PVUnlock
m_PVView
```

The same object also carries per-song progress such as clear state, scores, flick statistics, unlocks, and replay-related data.

### Registered IDs in the inspected save

Pack IDs:

```text
0–17
96–99
```

Pack 18 is absent.

Song indexes:

```text
0–73
```

`m_Index` is global across packs, not local to a pack.

Possible candidate indexes for a test Pack 18 in this exact save are therefore:

```text
74, 75, 76
```

Unused IDs are only candidates. They do not prove that the executable accepts new entries.

---

## 15. Pack 17 reference

Known Pack 17 registration:

| Global index | Title | Movie basename | Preview basename | Artwork | Index |
|---:|---|---|---|---|---:|
| 59 | 孤独の果て -extend edition- | `kodokunohate` | `pv_085_lp` | `music_17_01` | 0 |
| 60 | ローリンガール | `rolling_girl` | `pv_091_lp` | `music_17_01` | 1 |
| 61 | 透明水彩 | `toumei_suisai` | `pv_210_lp` | `music_17_01` | 2 |

The save stores resource **basenames**, not complete paths or extensions.

For example:

```text
m_MovieDataFileName = "rolling_girl"
m_PreSoundFileName  = "pv_091_lp"
m_ArtWorkFileName   = "music_17_01"
```

This supports a model in which the game derives the actual pack path elsewhere using the Pack ID.

`m_ArtWorkIndex = 0 / 1 / 2` selects different items from the shared artwork atlas. The exact crop table is not stored directly in the inspected `MusicData` records.

---

## 16. Repository sample files and provenance

The repository currently includes several reference files, but they are **not one complete matching pack**.

| File | Provenance / role |
|---|---|
| `music_11_01.png` | Pack 11 artwork atlas |
| `pv_046_lp.adx` | Pack 11 preview audio |
| `pv_057_lp.adx` | Pack 11 preview audio |
| `pv_083_lp.adx` | Pack 11 preview audio |
| `verificationFile.dat` | Pack 11 sample manifest |
| `thum_17.png` | Pack 17 thumbnail atlas |

Pack 11 corresponds to the sample set containing:

```text
double_lariat
from_y_to_y
hello_planet
```

with preview IDs:

```text
046, 057, 083
```

`thum_17.png` belongs to Pack 17 and must not be treated as if it were part of that Pack 11 sample.

---

## 17. `verificationFile.dat` runtime finding

The manifest contains SHA-1 entries such as:

```text
<sha1>  filename
```

An important real-device test changed `hello_planet.usm` while leaving the old SHA-1 in `verificationFile.dat`.

The already-installed song still loaded and played normally.

Therefore:

> **The SHA-1 values are not enforced during normal playback of the tested already-installed pack.**

The manifest may instead participate in download, installation, repair, or re-download validation.

This means a custom Pack 18 failure should currently prioritize investigation of:

```text
MusicPack registration
MusicData registration
Pack ID handling
catalog reconstruction
installation state
hardcoded pack limits
```

rather than assuming a SHA-1 mismatch or unknown encryption.

---

## 18. Proposed minimal Pack 18 experiment

The cleanest next test should change as little as possible.

### Step 1: clone a known working catalog entry

Start from Pack 17 and create a candidate:

```text
m_PackID        = 18
m_PackName      = "Custom_Pack01"
m_IsBuy         = true
m_IsInstalled   = true
```

### Step 2: clone its three `MusicData` records

Candidate global IDs:

```text
74
75
76
```

Initially preserve the known-good media basenames.

### Step 3: copy media byte-for-byte

Create:

```text
Mov_18/
```

with known working files copied unchanged.

The purpose of this stage is **not** to test a custom chart. It asks only:

> Can the game register and load one additional pack when both the catalog entry and filesystem resources exist?

### Step 4: restart and test persistence

After first launch:

- fully quit the game
- restart it
- confirm Pack 18 still exists
- check whether the catalog is rebuilt or the inserted objects are removed

Only after this succeeds should media, charts, artwork, and metadata be customized one component at a time.

---

## 19. How to interpret Pack 18 failures

| Symptom | Most likely next area to inspect |
|---|---|
| Pack does not appear | MusicPack serialization, pack whitelist/count limit, startup catalog rebuild |
| Pack appears but art/preview fails | path generation, atlas mapping, Pack ID → resource directory |
| Pack appears but selecting a song fails | song index limits, USM lookup, internal music table |
| Pack works until restart | save regeneration, catalog versioning, internal master catalog |

Outer `StatusData` also contains fields such as:

```text
version = 27
m_InfoSerial = 26
```

Their relationship to catalog synchronization remains unknown and should not be modified without evidence.

---

## 20. Safe editing workflow

For in-place edits that do not grow a CUE string:

1. Back up the original USM.
2. Parse the CUE / UTF table.
3. Identify the exact event and difficulty token.
4. Modify only the intended bytes.
5. Preserve all offsets and table sizes.
6. Re-parse the modified file.
7. Binary-diff against the original.
8. Test on device.

For lyric strings:

- check **UTF-8 byte length**, not character count
- preserve the parameter prefix
- zero-pad unused bytes when staying inside the original allocation
- update the relevant length field

For operations that grow or shrink data, a real rebuilder is required. Inserting raw bytes with a text editor can invalidate UTF offsets, chunk lengths, alignment, and seek metadata.

---

## 21. Tooling

| Task | Windows | macOS |
|---|---|---|
| USM / CRI inspection | CriStudio / CriCodecs | CriStudio / CriCodecs |
| Media inspection | FFmpeg / FFprobe | FFmpeg / FFprobe |
| Hex editing | HxD | Hex Fiend |
| Event parsing / patching | Python | Python |
| ARMv7 reverse engineering | Ghidra | Ghidra |

Useful SHA-1 commands:

macOS:

```bash
shasum -a 1 hello_planet.usm
```

Windows PowerShell:

```powershell
Get-FileHash "hello_planet.usm" -Algorithm SHA1
```

---

## 22. Current next steps

The most useful experiments now are:

1. **Minimal Pack 18 registration** using unchanged known-good media.
2. **Restart persistence test** for the inserted `MusicPack` / `MusicData` objects.
3. **Direction-only Note edit** on an existing registered song.
4. **Timing-only Note edit** without inserting a new record.
5. **Event insertion test** after building a proper CUE / UTF rebuilder.
6. **Determine the semantics of prefixes `1`, `2`, `4`, `6`, `7`, and `8`.**
7. **Trace the executable's Pack ID → directory path generation and any internal DLC whitelist.**

---

## 23. Evidence boundaries

Confirmed:

- the tested USM structure and CUE inventory
- the five-difficulty order
- tested gesture tokens
- one-difficulty Note disabling
- Chinese MV lyric replacement
- first NORMAL Interlude count
- 30 Hz audio-clock judgement model
- core score / Gauge / Crimax behavior
- Rank mapping
- `MikuFlick2.dat` pack/song catalog structure
- registered Pack IDs and song indexes in the inspected save
- lack of SHA-1 enforcement during normal playback in the tested installed-pack case

Not yet confirmed:

- Pack ID 18 acceptance
- song indexes above 73
- arbitrary CUE event insertion
- arbitrary chart-length changes
- complete custom-song installation
- exact semantics of all event prefixes
- exact startup catalog reconstruction logic
- every purpose of `verificationFile.dat`

---

## 24. References

- [Chinese localization and broader MikuFlick2 reverse-engineering notes](https://github.com/HachiMiku39/mikuflick_chinese_localization)
- [CRI Sofdec documentation](https://game.criware.jp/manual/native/sofdec2/latest/usr_console_encoder_main.html)
- [PyCriCodecsEx USM chunk definitions](https://mos9527.com/PyCriCodecsEx/_modules/PyCriCodecsEx/chunk.html)
- [PyCriCodecsEx UTF parser](https://mos9527.com/PyCriCodecsEx/_modules/PyCriCodecsEx/utf.html)
- [CriStudio / CriCodecs](https://github.com/Youjose/CriCodecs)
- [Hex Fiend](https://hexfiend.com/)

---

## 25. Distribution note

This research concerns assets and behavior from a commercial game. Code, parsers, format notes, and reproducible research can be published independently, but original songs, MVs, artwork, and other commercial assets may have separate redistribution restrictions.

For a future public chart editor or 64-bit reimplementation, the cleaner architecture is to let the user import assets locally from a legitimately obtained original installation rather than bundling original commercial content by default.

---

*Updated: 2026-10-05*  
*Primary sample: MikuFlick2 1.1.5 / ARMv7 / cryptid 0*  
*Status: chart editing confirmed in limited cases; full custom pack still under investigation*