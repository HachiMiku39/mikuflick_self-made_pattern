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
├── Mov_18/
│   ├── songname.usm
│   ├── songname2.usm
│   ├── songname3.usm
│   ├── music_18_01.png
│   ├── pv_xxx_lp.adx
│   ├── pv_yyy_lp.adx
│   ├── pv_zzz_lp.adx
│   └── verificationFile.dat
└── Thum_18/
    └── thum_18.png
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

The atlases and `m_ArtWorkIndex` values 0, 1, and 2 support selecting artwork by song index, but the complete crop coordinates and loading rules remain unconfirmed. Start with the dimensions, positions, and ordering of a working pack.

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
