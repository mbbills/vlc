# Matroska ordered editions: seamless chapter continuation (VLC 3.0.24)

This branch changes how VLC 3.0.24 plays ordered Matroska editions whose
chapters jump within one segment. A typical example is two "angle" editions
that share most of the stored video and differ in one interval. These are
source changes only. No binaries, media or test captures are included.

Baseline: VideoLAN VLC 3.0.24, upstream commit
`6de05adcbaf2e8b85fe86aad4169393098628119`.

## Commits, in order

| # | Commit subject | Patch it corresponds to | Patch SHA-256 |
|---|---|---|---|
| 1 | demux: mkv: keep the title mirror in sync when selecting an edition | `title-fix.3.0.24.patch` | `d01a5e8424027ce91f44e1244c003705d4851edb0ba2dd7ac4fe4bdb1c23013f` |
| 2 | demux: mkv: continue ordered chapters without resetting the decoders | `continuation.3.0.24.patch` | `392ac44540dadefaa6af9dad7bef4386e6d43e098737e4a6ac1863d47ef11240` |
| 3 | vout: don't report empty while a picture waits for its date | `endpoint.3.0.24.patch` | `d05a691fe2048cd9fcdc72d079370168d1f3bbf84ee1a06ca106f16b51a62f4e` |
| 4 | demux: mkv: find the audio frame covering the start of a continued chapter | `0001-mkv-audio-seekpoint-at-continuation.patch` | `0d19f870f8dfbebe409d6a6df44460b5f356ff493e273af8bfaa17d492369f0b` |
| 5 | demux: mkv: compare ordered chapter limits on the block timeline | `phase3-block-timeline-on-phase2.patch` | `9f53b113b3b37b96e842be2d67344fd8d39870a4b3d91c9f4e8e0cc1374a54a9` |
| 6 | demux: mkv: prime seek DTS on the block timeline | `phase3b-seek-dts-timeline.patch` | `2237629c107652e14422e3397cabb9cbb8845c146e20aefd948965c79a3ab3f1` |
| 7 | demux: mkv: drop subtitle blocks stored ahead of a continued chapter | `phase4b-spu-continuation-ownership.patch` | `817c03092b9020e1acdbfcdbfcd2deaf28aa72d4591bf461542d1235760b53ea` |

Commits 1 to 4 are the Phase 2 series, applied with `git am` from the
supplied patch files, which keeps their author, dates and messages. Their
commit IDs differ from the supplier's original commits, the last of which
was `a58c08adb0af56cee557e963b5cef26cbd17edb6`, because the committer is
different. The resulting source files are identical: all eight changed files
match the Phase 2 source hashes recorded in the project's reconstruction lock.
After commit 5, the three changed files match the lock's Phase 3 source hashes.
After commit 7, `modules/demux/mkv/virtual_segment.cpp` has SHA-256
`c64bd8db17bf13f6232ea39108f40cd0f8bd539776393d2be1ca193d0f43ce16`.

Commit 3 is the only change outside the Matroska demuxer, in the video output
core. A build needs the whole tree; the Matroska plugin alone on a stock core
is not equivalent.

## What changed

- **Continuation instead of reset.** When the next chapter of an ordered edition
  continues in the same segment and starts on a random access point, the
  demuxer seeks there without `ES_OUT_RESET_PCR` and without preroll. The
  decoders and the clock keep running, so the end of the outgoing chapter is
  no longer flushed.
- **Admission.** Blocks past the end of a chapter that is followed by a jump
  are dropped. Blocks stored ahead of the continued chapter's start are
  dropped too. Audio frames belong to the chapter that holds their middle.
  The chapter is left once every selected audio/video track has passed its
  end, with a 2 s limit.
- **Fallback.** If the continued chapter doesn't start on a random access
  point, the stock reset and preroll path is used.
- **Title state.** `DEMUX_SET_TITLE` updates the title the demuxer reports
  after a chapter change.
- **End of stream.** `vout_IsEmpty()` also counts the picture waiting for its
  display date, so the last picture of the stream is shown.
- **Subtitle ownership.** Commit 7 applies the "stored ahead of the continued
  chapter" rule to subtitle blocks too. Without it, when the read resumes
  before the destination (for example at the Cluster holding the audio frame
  that straddles the join), a subtitle stored near the end of the skipped
  chapter, such as the other angle's line, is shown in the selected edition.
- **Segment start.** VLC shifts block dates by the first Cluster timestamp, so
  that playback starts at 0. Commits 5 and 6 put the chapter limits and the
  post-seek DTS on that same shifted timeline. Without them, a file whose first
  Cluster isn't at 0 shows wrong pictures at the first jump. With no input
  caching, a large start offset also loses pictures and audio after the jump.

## Building

Build the whole tree as usual for VLC 3.0.x, with the VLC contribs or
system libebml/libmatroska. Nothing in this branch changes the build system.
The changes were compiled and exercised on Linux (GCC, libavcodec decoding)
and on Windows x64 (MSYS2 MINGW64, MSVCRT).

## Qualification summary

The test player is a libvlc client using vmem and amem callbacks. For every
displayed picture it compares a per-picture identity code and a whole-picture
hash against a continuous decode. For audio it checks decoded PCM tone
identity and timestamp continuity. Runs cover both editions, the default
decoder and forced avcodec, and default and zero input caching.

- Synthetic MPEG-2 ordered-edition fixtures (closed GOPs, variable soft
  pulldown, AC-3) with first Cluster timestamps of 0, 33 and 500 ms. Joins on
  an I picture play exactly: every picture, no picture from the unselected
  edition, continuous audio.
- Joins on leading-B or mid-GOP chapter starts take the fallback path. About
  one second at the join is lost, the same as stock 3.0.24, and nothing wrong
  is shown.
- Normal seeks inside an edition and continuous playback without editions
  behave as on stock VLC.
- A real DVD-derived two-angle test file played exactly in both editions at
  default caching (with commits 1 to 5). The remaining open item is zero
  caching, covered under known limits.

### DVD subtitles (VobSub), commit 7

Synthetic VobSub fixtures: hand-built SPU packets whose bitmap is an identity
code, so the test reads, from the pixels of every displayed picture, which
subtitle was actually composed. Cases: subtitles inside one interval, spanning
a jump, angle-specific subtitles (including one 10 ms before an alternative
ends), a subtitle exactly at a jump destination, stops exactly at chapter
ends, bitmap changes and stops just before and after a jump, an open-ended
packet replaced by the next one, and seeks and chapter changes. Each case was
run with first Cluster timestamps of 0, 33 and 500 ms, two interleaves (the
audio frame straddling a join stored after or before the destination I
picture), default and zero caching, and both editions.

- Linux x64 (GCC 13, FFmpeg 6.1): with commits 1 to 6, 114 of 120 runs
  matched the authored timing on every picture. All 6 failures were the
  other angle's subtitle after the jump, with the straddling audio frame
  stored ahead. With commit 7, 120 of 120 matched. Picture hashes and decoded
  audio were identical with and without commit 7. A rerun on builds without
  any tracing reproduced both results.
- Windows x64: reported by the MovieCRITIC qualification (not rerun for this
  document) as 120 of 120 with commit 7, together with the original failure
  reproduced on the unpatched build and no video or audio regression.
  Qualified Windows `libmkv_plugin.dll` SHA-256:
  `33c249ffa98d7f4065ebd365ec665c770af986bbcabf06cd4d5effb63c4b85b6`.
- With stock 3.0.24, every jump flushes the decoders, so a subtitle spanning
  a jump disappears. With this branch it stays, and a stop or change command
  that falls after the jump still takes effect.

This qualifies subtitle ownership at seamless joins on synthetic material. It
is not a claim of general VobSub correctness or of full-title DVD
qualification. The fixtures and test harness are not part of this branch.

## Known limits

- Only frame-coded, closed-GOP MPEG-2 joins that start on an I picture are
  continued seamlessly. Field-coded and open-GOP joins use the fallback.
- With zero input caching, the video output may render the first picture of an
  ordered edition twice. It is the same decoded picture: the decoder's output
  count is exact. This comes from the vout refresh path, not from the demuxer.
- After a seek, one or two already-queued pictures can still be displayed.
  With zero caching, a single forced preview of the picture just before the
  seek target can also appear before playback resumes exactly at the target.
  This matches stock VLC.
- VobSub timing: VLC's spudec counts control-sequence delays in 11 ms units
  instead of 1024/90000 s, so subtitles end about 3.3% of their stop delay
  early. This is stock VLC behavior, unchanged here.
- Seeking into a subtitle that is already on screen does not show it until
  the next subtitle starts, because the seek resumes after the block that
  carries it. This matches stock VLC and a file without editions. How the
  subtitle state should be restored after a seek is unresolved.
- A subtitle spanning a join that takes the fallback path is flushed with
  the decoders, as on stock VLC.
- Exact-stop coverage is incomplete: only a few stops that land exactly on a
  chapter end or join were tested.
- Forced subtitles, DVD menu highlights, several subtitle tracks at once, PGS
  and text subtitles, real disc-derived subtitles, Windows 7 and other codecs
  were not qualified.
- Not submitted upstream.
