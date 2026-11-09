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
| 8 | codec: spudec: use the exact DVD tick for control sequence delays | `phase4c-a-spudec-dvd-clock.patch` (A) | `0bafceb424acbb69137a371ab1b1e0fa0e0b25bcd156bfc34434fc9ee55c4b72` |
| 9 | vout: drop undated pictures on a full flush | `phase4c2-vout-flush-undated.patch` (C) | `87e106dfdcef25a97635b6c1e4ec1b0175e7d3cfec161f28ee6b281ee2e50897` |
| 10 | demux: mkv: restore a subtitle already on screen after a seek | `phase4c-b-mkv-seek-active-subtitle.patch` (B) | `07ef166fee9631f6feec4fcc824d7f539fb7f8bdfa79522794d43dd363b2b940` |

Commits 1 to 4 are the Phase 2 series, applied with `git am` from the
supplied patch files, which keeps their author, dates and messages. Their
commit IDs differ from the supplier's original commits, the last of which
was `a58c08adb0af56cee557e963b5cef26cbd17edb6`, because the committer is
different. The resulting source files are identical: all eight changed files
match the Phase 2 source hashes recorded in the project's reconstruction lock.
After commit 5, the three changed files match the lock's Phase 3 source hashes.
After commit 7, `modules/demux/mkv/virtual_segment.cpp` has SHA-256
`c64bd8db17bf13f6232ea39108f40cd0f8bd539776393d2be1ca193d0f43ce16`.

Commits 8 to 10 are the Phase 4C corrections. Each commit is exactly its
patch file. Source SHA-256 after them:

| File | Before | After |
|---|---|---|
| `modules/codec/spudec/parse.c` (8) | `96ee4626…a427` | `4c13c80b2090dde44d8841921f353fbd3380001f4de488e7e9b8ec821534aa43` |
| `src/video_output/video_output.c` (9) | `9b7a49f108565f2b49d591a2bd58bc2e5ae6a0691183d86cef19e038cc739256` | `2459f1d0ed98e1607bfefea8691ffb6bbe0855358c1f916c79906cff48a3057b` |
| `modules/demux/mkv/matroska_segment.cpp` (10) | `fb68e2e3…5e0c` | `78b93a70f1b74e90d9124797b1d1ee46dd82dd202114a339dffbc07745ebe222` |
| `modules/demux/mkv/matroska_segment.hpp` (10) | `21f8387d…2a12` | `513154220783085b43a25161202bfba331313952461afc4c0334be4c59373415` |
| `modules/demux/mkv/util.cpp` (10) | `2ba5b92d…18cd39` | `af467f62b469308c2402dedd1e9b8a0540d67ed5fa48649ee02ea8d9aa29d874` |

Dependencies: commit 8 (A, DVD clock) and commit 9 (C, full-flush undated
pictures) are each usable on their own within their qualified profile.
Commit 10 (B, active-subtitle restoration) requires commit 9 for the
accepted configuration; without it, a picture from before the seek can be
shown composited with the restored subtitle. Commit 9 is placed before
commit 10 so the branch never contains B without C.

Commits 3 and 9 change the video output core and commit 8 the DVD subtitle
decoder; the rest is in the Matroska demuxer. A build needs the whole tree;
the Matroska plugin alone on a stock core is not equivalent.

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

- **DVD subtitle clock (commit 8).** spudec converted control-sequence
  delays as 11 ms per tick instead of 1024/90000 s, so starts, stops and
  changes inside a subtitle packet came 3.3% early. It now uses the exact
  tick, rounded up to the microsecond.
- **Full flush (commit 9).** After a seek, a picture decoded from the old
  position could be put as the forced first picture with no valid date, and
  a full vout flush kept undated pictures. It was then shown after the flush
  with the new position's subtitles. A full flush now discards it too.
- **Seek into a subtitle on screen (commit 10).** On an accurate seek, each
  selected subtitle track starts at its latest block starting at or before
  the target, looking back up to 10 s and never before the current ordered
  chapter's start. VobSub blocks now carry their BlockDuration as length, so
  the core keeps a subtitle still active at the target and drops one that
  has ended.

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

### Phase 4C: subtitle clock, seek restoration and stale pictures, commits 8 to 10

Same harness, extended with fixtures that put subtitle starts and stops
exactly on, and a few milliseconds around, picture times and ordered-chapter
transitions (T, T3), forced display (T2), and seeks into active subtitles
(S): into shared and alternative intervals, backward, exactly on a start,
after a subtitle ended, chapter navigation, within one chapter, repeated
targets with identical timestamps, three seeks about 200 ms apart, and into
a position with no subtitle. Ordered and plain files, first Cluster offsets
0, 33 and 500 ms, default and zero caching. A per-picture metric counts
every display event, including repeated renders.

- Linux x64 (GCC 13, FFmpeg 6.1), producer evidence:
  - Commit 8: the 120-case ownership matrix stays exact, and T2, T3 and S
    match the exact DVD clock on every picture (72 of 72).
  - Commit 10, with or without commits 8 and 9: after a seek, no subtitled
    destination picture is missing its subtitle beyond the first one or two.
    In the same seek sets, stock VLC, commit 6 and commit 7 lost 976, 992
    and 2,360 subtitled destination pictures.
  - Commit 10 without commit 9 showed a pre-seek picture composited with the
    restored subtitle in about 15% of seeks (29 pictures in 15 of 98 runs,
    21 in 13 of 70 on a second fixture set). Commit 9 brings this to 0 in
    all 238 seek runs per build, with and without commit 8. In traced runs,
    pictures rendered after their own generation's flush went from 51 to 0.
  - Whole-picture hashes, picture order, decoded audio tones and sample
    counts are identical with and without commits 8 to 10 on the control
    files. No subtitle from another edition or angle appeared in any run.
- Windows x64 (MSYS2 MINGW64), reported by the MovieCRITIC Phase 4C.3
  qualification and not rerun for this document:
  - 1,080 completed subtitle ownership and clock cases across five patched
    configurations, and 1,428 completed seek cases across six builds.
  - 120 of 120 ownership and 72 of 72 non-overlapping timing cases for each
    relevant candidate.
  - In the matched diagnostic set, commit 9 took commit 10's stale pre-flush
    renders from 46 to 0 and stale pixel composites from 27 to 0.
  - 72 exact video and audio control comparisons, 16 generated regressions
    of earlier-phase content, ordinary generated DVD and Matroska lifecycle
    controls, and 49 focused tests passed.
  - Completed cases are not all passes: near-end seeks with default caching
    that produce no destination video are failures, not successful seeks.

This is experimental qualification on generated material. It is not a claim
of general subtitle timing, general seek reliability, full-title DVD
correctness or production readiness.

## Phase 4C conditions on hold

- The T fixture's overlapping subtitles are not normative for a DVD
  decoder, and the SPU decoder thread holds a decoded subtitle until 500 ms
  before its start, which can delay an earlier-starting subtitle that
  arrives after it.
- After a seek, the first destination picture can be shown before its
  subtitle: the input's buffering does not wait for subtitle decoders, and
  the first video picture is previewed at once. Pre-existing in stock VLC.
- A rare race in the other direction: a new-position picture composited
  with the previous position's subtitle (1 of 168 runs on a build without
  commit 9; not fixed by commit 9, by construction).
- Seeks near the end of a title with default caching can fail to produce
  destination video.
- Commit 10 restores only subtitles that started at most 10 s before the
  target, and only VobSub blocks with a BlockDuration; otherwise behavior
  is unchanged.
- Stream output (transcode, remux) with commit 10's block lengths is
  untested.
- Other subtitle codecs, full-title authoring, Windows 7, interlaced
  material and physical discs are not qualified.

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
- VobSub timing (11 ms ticks) and seeking into a subtitle already on screen
  were limits of commits 1 to 7; commits 8 and 10 address them within the
  conditions listed above.
- A subtitle spanning a join that takes the fallback path is flushed with
  the decoders, as on stock VLC.
- Exact-stop coverage is incomplete: only a few stops that land exactly on a
  chapter end or join were tested.
- Forced subtitles, DVD menu highlights, several subtitle tracks at once, PGS
  and text subtitles, real disc-derived subtitles, Windows 7 and other codecs
  were not qualified.
- Not submitted upstream.
