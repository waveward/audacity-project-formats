---
title: Looking inside an .aup3 or .aup4 with the sqlite3 command line
description: Open an Audacity project read-only with sqlite3, decode its version stamps, count its sample blocks and pull one block out as a WAV — with the WAL catch that makes sqlite3 -readonly fail.
---

# Looking inside an .aup3 or .aup4 with the sqlite3 command line

An Audacity project is an SQLite database (see the [main notes](./)), so the stock `sqlite3` shell can read one without Audacity. These are the commands, and the output they gave, on two public projects: `clips4.aup4` from Audacity's own test data and `power-plug.aup3` from the tau-OS hydrogen-sound-theme repository. Run on 12 October 2026 with sqlite3 3.54.0 and FFmpeg 7.1.1 on macOS. I build [Waveward](https://aup4.aivismonitor.com), a browser tool that reads these files; that is where the sample rate and durations quoted below were read.

**Work on a copy.** Nothing here writes to the project, but a copy costs nothing.

## Opening it read-only: the WAL catch

Audacity puts the database in write-ahead-log mode, and SQLite records that in the file itself: bytes 18 and 19 of the header are both 2.

```
$ xxd -s 18 -l 2 clips4.aup4
00000012: 0202
```

With no `-wal` or `-shm` file beside it, `sqlite3 -readonly` refuses such a file:

```
$ sqlite3 -readonly clips4.aup4 "PRAGMA user_version;"
Error in 3rd command line argument: unable to open database file
```

The `immutable` URI parameter opens it: it tells SQLite the file will not change, and SQLite's WAL documentation names an immutable database as one of the ways a read-only WAL database can be opened. Nothing is created next to the file.

```
$ sqlite3 "file:clips4.aup4?immutable=1" "PRAGMA application_id;" "PRAGMA user_version;" "PRAGMA page_size;"
1096107097
67108865
65536
```

One exception matters. If the project **does** have a `<name>-wal` file next to it — Audacity was killed, crashed or ran out of disk before closing it — `immutable` ignores that file, and the edits in it with it. In that case copy the project together with its `-wal` (and `-shm`) into a folder of their own and open the copy without `immutable`, so SQLite reads the log.

## The two numbers that identify it

- `application_id` 1096107097 is 0x41554459, the ASCII bytes `AUDY` — Audacity's tag.
- `user_version` is the Audacity version packed into four bytes, major to modification level:

```
$ python3 -c "print('.'.join(str(b) for b in (67108865).to_bytes(4, 'big')))"
4.0.0.1
```

`clips4.aup4` comes from a pre-release build (its document's `audacityversion` attribute is literally `..`). `power-plug.aup3` gives 50397184, which is **3.1.0.0** — although its document says `audacityversion="3.5.1"`. That is the pre-3.6.1 rule described in the main notes, on a real file: the stamp was set according to the features the project used, not by the version that saved it.

The page size differs too: 65,536 bytes in `clips4.aup4`, 4,096 in `power-plug.aup3`.

## Tables

```
$ sqlite3 "file:clips4.aup4?immutable=1" .tables
autosave     project     project_history     sampleblocks
$ sqlite3 "file:power-plug.aup3?immutable=1" .tables
autosave     project     sampleblocks
```

`project_history` is the Audacity 4 table of earlier saves; in `clips4.aup4` it exists and is empty. `autosave`, the crash-recovery copy, is empty in both. The saved document itself is two blobs, the name dictionary and the token stream (2,067 and 2,653 bytes in `clips4.aup4`): `SELECT length(dict), length(doc) FROM project;`.

## The audio

```
$ sqlite3 "file:power-plug.aup3?immutable=1" \
  "SELECT printf('0x%08X', sampleformat), count(*), sum(length(samples)), max(length(samples)) FROM sampleblocks GROUP BY sampleformat;"
0x0004000F|2|1276792|638396
```

`0x0004000F` is 32-bit float. `clips4.aup4` holds a single float block of 229,740 bytes.

## Pulling one block out as a WAV

`writefile()` is a function of the `sqlite3` shell (not of SQL itself) that writes a value to disk. A block is headerless samples, so FFmpeg has to be told the format, the rate and the channel count:

```
$ sqlite3 "file:power-plug.aup3?immutable=1" \
  "SELECT writefile('block.raw', samples) FROM sampleblocks ORDER BY blockid LIMIT 1;"
638396
$ ffmpeg -f f32le -ar 44100 -ac 1 -i block.raw -c:a pcm_f32le block.wav
$ ffprobe -v error -show_entries stream=codec_name,duration -of default=nw=1 block.wav
codec_name=pcm_f32le
duration=3.619025
```

638,396 bytes is 159,599 four-byte samples: 3.619 seconds at 44.1 kHz. The rate is not stored with the block; it is in the project document (44,100 Hz in this project). Leave out `-c:a pcm_f32le` and FFmpeg writes 16-bit PCM instead.

One thing this shows: **trimming hides audio, it does not remove it.** `power-plug.aup3` plays for 1.22 seconds, but that one block holds 3.62 seconds, and all of it comes back out with the two commands above. Blocks are immutable; a trim only changes what the document says to play.

## What this does not give you

One block is one channel's slice of one clip. The mix — clips placed at their offsets, trims, envelopes, gain and pan, silence encoded as non-positive block ids — needs the document decoded, and that is a binary token stream, not XML (see the main notes). That is the part [Waveward](https://aup4.aivismonitor.com) does in the browser: [.aup4 to WAV](https://aup4.aivismonitor.com/aup4-to-wav), [.aup3 to WAV](https://aup4.aivismonitor.com/aup3-to-wav), and, for a project that will not open, [recovery](https://aup4.aivismonitor.com/recover-audacity-project).

## Sources

- [SQLite: Write-Ahead Logging — read-only databases](https://www.sqlite.org/wal.html#readonly) and [URI filenames: immutable](https://www.sqlite.org/uri.html#uriimmutable)
- [SQLite database file format](https://www.sqlite.org/fileformat2.html) — header bytes 18–19 (2 = WAL), `user_version` at offset 60, `application_id` at 68
- [sqlite3 shell: writefile()](https://sqlite.org/cli.html#file_i_o_functions)
- The `AUDY` tag, the packed `user_version` and the sample-format codes: see the [main notes](./) and their sources

---

Licensed [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). "Audacity" is a trademark of Muse Group; these are independent notes, not affiliated with or endorsed by the Audacity project.
