# Audacity project files: field notes on .aup3 and .aup4

Notes collected while implementing a reader, converter, recovery and downgrade tool for Audacity project files (for [Waveward](https://aup4.aivismonitor.com), a browser-based tool — disclosure: I build it). Every statement below was checked against the Audacity source of the version named, or against Audacity's own release notes and tools; the links are at the end. Corrections welcome — open an issue.

## The one-line version

| | .aup3 | .aup4 |
|---|---|---|
| Written by | Audacity 3.0 – 3.7 | Audacity 4.0 and newer |
| Opens in | Audacity 3 and 4 (4 converts it to .aup4 on open, one way) | Audacity 4 only — 3.7 refuses it |
| Container | SQLite 3 database | SQLite 3 database, same design |
| `user_version` (SQLite header) | the Audacity 3 build that saved it, e.g. 3.7.9 | the Audacity 4 build, e.g. 4.0.0 |
| Document `version` attribute | 1.3.0 — unchanged since the .aup era | 2.0.0 |
| Tables | `project`, `autosave`, `sampleblocks` | the same, plus `project_history` (the last ten saves — kept, never restored from) |
| Audio | immutable sample blocks: int16, int24-in-4-bytes, or float32 | identical bytes |
| Clip fields | offset, trims, envelope; stretch ratio and pitch shift since 3.4 | adds `groupId`, `clipTempo`, `clipStretchToMatchTempo`, `isSelected` |
| Colour | a palette index (`colorindex`) on the track | a palette index on every clip too |
| Unsaved working copy | `.aup3unsaved` | `.aup4unsaved` |

## It is a SQLite database, and that decides everything

- The file is a standard SQLite 3 database. Two header fields identify it: `application_id` at byte offset 68 holds Audacity's `AUDY` tag, and `user_version` at byte offset 60 holds the Audacity version that saved it, packed as four bytes — major, minor, revision, modification level.
- Three tables in Audacity 3, four in Audacity 4: `project` (the saved document), `autosave` (the crash-recovery copy), `sampleblocks` (the audio), and in 4.0 `project_history`, which keeps earlier saved documents, newest first, ten deep (`kProjectHistoryDepth`). Every save inserts the document into it and deletes all but the ten newest generations; as of 4.0.1 the only code that reads the table copies it along when a project is written to a new file, and there is no "revert to an earlier save" command — those ten documents are reachable only by reading the database directly.
- The document in `project` / `autosave` is not text XML. It is a dictionary blob plus a binary token stream (`ProjectSerializer`): a table of two-byte name ids, and 16 field types in Audacity 3, 17 in Audacity 4 (a blob type was added). Strings are stored in the writing platform's native wide-character width — two bytes per character from Windows, four from macOS and Linux — so a byte-level search for a track name has to know which platform saved the file.
- `sampleblocks` columns: `sampleformat`, `summin`, `summax`, `sumrms`, `summary256`, `summary64k`, `samples`. The three sample formats are int16 `0x00020001`, int24 `0x00040001` and float `0x0004000F`, with the byte width packed into the upper sixteen bits.
- Blocks are **immutable and never updated in place**: every edit writes new blocks and repoints the document. A block holds up to about 1 MB — roughly five seconds of mono audio at default settings — so block ids are not contiguous within a track. Silence is encoded as a **non-positive block id**, not as a block of zeros.

## Why Audacity 3.7 refuses an .aup4 — two gates, not one

1. **Gate 1, the SQLite header.** Audacity compares `user_version` with its own version and refuses anything newer: *"This project was created with a newer version of Audacity"*.
2. **Gate 2, inside the document.** The root `project` element carries a file-format `version` attribute, separate from the application version. Audacity 3.7 writes and accepts 1.3.0 (`AUDACITY_FILE_FORMAT_VERSION`); Audacity 4 writes 2.0.0. A higher value stops the load before any track is read: *"You may need to upgrade to a newer version to open this file"*.

Everything else is the same design in both versions. That is what makes a downgrade possible: rewrite the two stamps (`user_version` → 3.7.9, `version` 2.0.0 → 1.3.0, and the `audacityversion` attribute beside it), drop the four clip fields Audacity 3 has never seen, and copy the sample blocks byte for byte. Verified on `clips4.aup4` from Audacity's own test data: Audacity 3.7.9.0 opened the downgraded copy (six file handles including the WAL and SHM, an entry in the recent-files list) and rejected the untouched original.

Audacity itself offers no way back: 4.0 converts .aup3 to .aup4 on open and *"converted projects cannot be saved back to .aup3"* (release notes); an Audacity contributor's tracker issue from May 2025 states that backwards compatibility from 4 to 3 is not planned. Audacity 4.0.1 (30 September 2026) changed neither gate: `AUDACITY_FILE_FORMAT_VERSION` is still 2.0.0, and its release notes add no save or export to .aup3.

A note on the stamp's history, as reported by an Audacity team member on the official forum (May 2026): before 3.6.1 the project version was set according to which features a project used; from 3.6.1 it is simply the running Audacity version. So Audacity is strictly backwards-compatible (newer reads older) and no longer forwards-compatible.

## What is not in Audacity 4.0

From the 4.0.0 release notes: Time Tracks, Note/MIDI tracks, the Mixer, the Macro Manager and the scripting pipe, VAMP and LADSPA plugin hosting, and Play-at-speed. A project that depends on any of these is a reason to keep a 3.7 install — and to keep the .aup3.

Audacity 4.0.1 lists *Export multiple* — each track, or each labelled region, to its own file — among its new features; its release notes bring back none of the features listed above.

## When a project will not open but is not damaged

- **A non-ASCII path on Windows, in 4.0.0.** Audacity 4.0.0's command-line parser decoded the file path with the system's ANSI code page instead of UTF-8, so a project whose name or folder contains Cyrillic, Japanese, Hebrew or an accented letter (a parent folder such as `Téléchargements` is enough) was reported as unreadable or missing — an .aup3 opened in 4.0.0 failed the same way before it was even converted. Fixed in 4.0.1 (pull requests 11985 and 12115); on 4.0.0, move the project to a folder with plain ASCII names, or start Audacity with nothing open and use File → Open. macOS and Linux are not affected.
- **`-wal` and `-shm` files beside the project.** Audacity opens a project in SQLite's write-ahead-log mode with automatic checkpointing switched off (`wal_autocheckpoint = 0`) and copies the log into the project file from its own thread; closing the project forces the final copy, after which SQLite deletes both files. A `<name>-wal` next to a closed project means that close never finished — a crash, a kill, a full disk — and the log still holds edits the project file does not. Do not delete it, and move or copy it together with the project.

## When a project breaks

- Audacity surfaces two SQLite results by number. **Error code 11** (`SQLITE_CORRUPT`) means the database structure is damaged — a B-tree page that is not what it should be, typically after a crash mid-write, a full disk, or a drive disconnected before the write was flushed. **Error code 101** (`SQLITE_DONE`) is what you see when the structure is fine but a sample block the document points at is not there.
- Because blocks are immutable, the audio usually survives: blocks written successfully stay valid even when later writes failed. Recovery is a matter of finding a document that still decodes (`project`, then `autosave`, then — in an .aup4 — the `project_history` generations), listing which referenced blocks can still be read, and rebuilding a project from those. Blocks that were never written are silence; nothing recovers them.
- Audacity's own recovery utility, [audacity-project-tools](https://github.com/audacity/audacity-project-tools), is a terminal program. Points that matter in 2026: it has had a [macOS build since v1.0.5](https://github.com/audacity/audacity-project-tools/releases/tag/v1.0.5) (2026-01-27); its `MaxSupportedVersion` is 3.7.0.0, so an .aup4 is treated as a project with a broken `user_version`, clamped to 3.7, and any recovered copy is written as a 3.7.0 project ([AudacityDatabase.cpp](https://github.com/audacity/audacity-project-tools/blob/main/src/AudacityDatabase.cpp)); it reads `project` and `autosave` only, not `project_history`; and the [official support page](https://support.audacityteam.org/au3/troubleshooting/recovering-corrupted-projects) asks for about four times the project's size in free disk space (and, as of this writing, still says macOS is unsupported).

## Hands-on

- [Looking inside an .aup3 or .aup4 with the sqlite3 command line](inspect-with-sqlite3.md) — opening one read-only (and why `sqlite3 -readonly` fails), decoding the version stamps, counting sample blocks, and pulling one block out as a WAV; measured on two public projects.

## Sources

- [ProjectFileIO.cpp, Audacity 3.7.9](https://github.com/audacity/audacity/blob/Audacity-3.7.9/libraries/lib-project-file-io/ProjectFileIO.cpp) — the `AUDY` application id, the packed `user_version`, `AUDACITY_FILE_FORMAT_VERSION` 1.3.0, and both refusal messages.
- [ProjectFileIO.cpp, Audacity 4.0.0](https://github.com/audacity/audacity/blob/Audacity-4.0.0/au3/libraries/au3-project-file-io/ProjectFileIO.cpp) — `AUDACITY_FILE_FORMAT_VERSION` 2.0.0 and the `project_history` table (`kProjectHistoryDepth` = 10).
- [ProjectFileIO.cpp, Audacity 4.0.1](https://github.com/audacity/audacity/blob/Audacity-4.0.1/au3/libraries/au3-project-file-io/ProjectFileIO.cpp) — still 2.0.0; on every save the document is inserted into `project_history` and the table pruned to ten generations; the table is read only by `CopyTo`, which carries it into a new file, and never restored from.
- [DBConnection.cpp, Audacity 3.7.9](https://github.com/audacity/audacity/blob/Audacity-3.7.9/libraries/lib-project-file-io/DBConnection.cpp) — `journal_mode = WAL`, `wal_autocheckpoint = 0`, Audacity's own checkpoint thread, and the truncating checkpoint on close (the same code ships in 4.0.0 under `au3/libraries/au3-project-file-io`).
- [SQLite: Write-Ahead Logging](https://www.sqlite.org/wal.html) — the WAL is part of the database's persistent state; a database separated from its WAL can lose committed transactions.
- [ProjectSerializer.cpp, Audacity 4.0.0](https://github.com/audacity/audacity/blob/Audacity-4.0.0/au3/libraries/au3-project-file-io/ProjectSerializer.cpp) — the binary token stream, its dictionary, and the 17 field types.
- [WaveClip.cpp, Audacity 4.0.0](https://github.com/audacity/audacity/blob/Audacity-4.0.0/au3/libraries/au3-wave-track/WaveClip.cpp) — the clip attributes Audacity 4 writes, including the four that Audacity 3 does not read.
- [SqliteSampleBlock.cpp, Audacity 3.7.9](https://github.com/audacity/audacity/blob/Audacity-3.7.9/libraries/lib-project-file-io/SqliteSampleBlock.cpp) — the `sampleblocks` columns and silence as a non-positive block id.
- [SampleFormat.h, Audacity 3.7.9](https://github.com/audacity/audacity/blob/Audacity-3.7.9/libraries/lib-math/SampleFormat.h) — the three sample-format codes.
- [SQLite database file format](https://www.sqlite.org/fileformat2.html) — `user_version` at offset 60, `application_id` at offset 68 — and [SQLite result codes](https://www.sqlite.org/rescode.html) for 11 and 101.
- [Audacity 4.0.0 release notes](https://github.com/audacity/audacity/releases/tag/Audacity-4.0.0) — the .aup4 format, the one-way conversion, and the features not in 4.0.
- [Audacity 4.0.1 release notes](https://github.com/audacity/audacity/releases/tag/Audacity-4.0.1) — *Export multiple*, and the fix for opening projects and files with non-ASCII characters in their name or folder.
- Audacity pull requests [11985](https://github.com/audacity/audacity/pull/11985) (the command-line parser decodes paths as UTF-8) and [12115](https://github.com/audacity/audacity/pull/12115) (the other Windows code-page conversions) — both shipped in 4.0.1.
- [audacity-project-tools README](https://github.com/audacity/audacity-project-tools) — block size and the never-updated-in-place behaviour.
- [audacity/audacity issue #8841](https://github.com/audacity/audacity/issues/8841) — "backwards compatibility is not planned".

## Further reading

Longer write-ups of the above, kept current, on the Waveward site:

- [What is an .aup4 file](https://aup4.aivismonitor.com/what-is-aup4-file) — the format table by table, and how it differs from .aup3.
- [Why Audacity 3 won't open an .aup4, and the way back](https://aup4.aivismonitor.com/aup4-to-aup3) — the two gates and what a downgrade rewrites.
- [Recovering a corrupted Audacity project](https://aup4.aivismonitor.com/recover-audacity-project) — error codes 11 and 101, the `-wal` file, and rebuilding from the blocks that survive.
- Getting the audio out of a project without Audacity: [.aup4 to MP3](https://aup4.aivismonitor.com/aup4-to-mp3) · [.aup4 to WAV](https://aup4.aivismonitor.com/aup4-to-wav) · [.aup3 to MP3](https://aup4.aivismonitor.com/aup3-to-mp3) · [.aup3 to WAV](https://aup4.aivismonitor.com/aup3-to-wav) · [open an .aup4 or .aup3 in the browser](https://aup4.aivismonitor.com/).

---

Licensed [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — reuse freely with attribution. "Audacity" is a trademark of Muse Group; this is an independent set of notes, not affiliated with or endorsed by the Audacity project.
