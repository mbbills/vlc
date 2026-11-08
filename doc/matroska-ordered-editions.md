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

Commits 1 to 4 are the Phase 2 series, applied with `git am` from the
supplied patch files, which keeps their author, dates and messages. Their
commit IDs differ from the supplier's original commits, the last of which
was `a58c08adb0af56cee557e963b5cef26cbd17edb6`, because the committer is
different. The resulting source files are identical: all eight changed files
match the Phase 2 source hashes recorded in the project's reconstruction lock.
After commit 5, the three changed files match the lock's Phase 3 source hashes.

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
- Visible subtitles, Windows 7 and other codecs were not qualified.
- Not submitted upstream.
