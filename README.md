
A real-time N-source audio mixer over the UltraHighSpeecCodec stream format. C++17, Linux
x86-64, CMake + CTest, no system dependencies. Media import is sniffed by magic
(`bad`, `wav`, `mp3`, `flac`, `ogg`), never by extension.

Named for the pseudonym Morrissey used for the sped-up backing vocal on
*Bigmouth Strikes Again* — which is exactly what feed 1 does.

## What this is

Three independent feeds, each at its own sample rate and depth, are decoded into
a common int32 domain, resampled to a 48 kHz / 32-bit bus, volume-weighted,
summed, and written to a `.bad` output (and optionally a stereo WAV). Feed 1
carries a **+20% pitch shift** —it consumes its input
faster than real time and ends early, like a tape played fast.

The work is in the plumbing: a pull-based pipeline driven by an injected clock,
with four seams (`Source`, `Stage`, `Sink`, `ControlSource`) clean enough that
the file reader, the control input and the output sink can each be replaced
without the mixer noticing.

## Design documents

The full low-level design, implementation plan, appendices and plan review live
in `EnhancedDesign7/` (`LLD.md`, `ImplementationPlan.md`, `APPENDICES.md`,
`PlanReview.md`). The submission is complete at milestone M9 of the plan; the
optional tracks (telemetry, GUI control surface, runtime feed install) and the
media-import decoders (MP3/WAV/FLAC/Ogg) are also implemented.

## Build

```sh
cmake --preset default          # Debug, tests on, no GUI/FFmpeg
cmake --build build -j
ctest --test-dir build -j8      # 30 tests, incl. the bit-identical golden test
```

Other presets: `all` (Release + GUI + FFmpeg), `asan` (ASan+UBSan), `tsan`
(ThreadSanitizer). A clean TSan run is the acceptance gate for the thread model;
on kernels with high ASLR entropy TSan needs `setarch -R ctest --test-dir
build-tsan -j4`.

## Web console

The same engine behind a browser — no VNC, no display needed:

```sh
./build/webmixer --port 8080            # foreground, Ctrl-C to stop
./build/webmixer --port 8080 --daemon   # fork: fully detached background server
```

Open `http://<host>:8080` in any browser. The page has an HTML5 audio player
(play/pause, seek, volume, a Web Audio analyser) and a mix panel with **3
roles**: **1 · lead voice** (sets the mix length), **2 · background score**,
**3 · special effects**. Each slot has its own "add audio file" action
(`.bad`/`.wav`/`.mp3`/`.flac`/`.ogg`, sniffed by magic — never by extension)
plus per-slot volume/pitch and a reset. **Feeds 2–3 loop to fill feed 1 when
shorter than it** (a decoded-duration probe decides; if longer they play once
and extend the mix). The slots start **empty** — only what you put in them is
mixed. The **"load demo preset"** button fills them with the demo
test-tones, but **any file you upload auto-clears demo feeds from the other
slots**, so a stray demo tone can never sneak into your mix. **Mix** posts
`POST /api/mix`; the engine renders at full speed (SteppedClock) and the page
polls `GET /api/status` until done, then loads the WAV into the player
(cache-busted, `Cache-Control: no-store`, Web Audio graph created up-front so
playback never stops) with a download link.
A **page refresh is a fresh start**: `POST /api/clear` drops every uploaded
feed (files unlinked), removes the rendered mix, hides the download link, and
the player starts empty until you mix again. Uploads **persist across a
server restart** (`web-out/slots.json`, restored on startup), so a restarted
daemon keeps your lead/background/effects instead of failing a mix with
"no feeds configured".

**Live mixer controls.** The per-slot volume/pitch knobs and the master volume
slider are heard immediately. Each change is posted to `POST /api/live/control`
and the mixer applies it at the very next tick (~10 ms): control events are
stamped with the mixer's *own* stream position (a per-period atomic) rather
than a wall-clock estimate, so a knob turn is never deferred — even after a
long session, or if the real-time scheduler ever fell behind wall clock. The
master **volume** slider drives a single Web Audio gain node shared by the
offline player and the live pump (the media element's own volume stays at 1),
so the same control works in both modes.

The live audio path is hardened against playback gaps: the browser runs its
ScriptProcessor pump on a ~0.5 s buffer so a busy main thread or a clock
mismatch rarely starves it; on a brief underrun it fades the last sample out
instead of cutting straight to silence; if a long stall ever lets frames pile
past ~1.5 s it jumps cleanly to the live edge rather than trimming frame by
frame; and the visualiser runs at half refresh rate while live so the pump
stays first-class on the main thread. A knob change is therefore heard within
about half a second, never behind stale buffered audio.

**Live pause/resume.** The Play button pauses the live mixer in place: the
engine freezes its timeline (the feed sources are in-memory buffers, so the
whole pipeline stalls with no I/O) and Play resumes from the same stream
position — no restart from the top. If the session dies while paused (e.g. a
long idle tears the WebSocket down on either side), the next press of Play
reports it and automatically starts a fresh live session instead of leaving
the button stuck on "paused".

**Reset.** Each slot's reset button clears it (an uploaded file is unlinked,
a preset feed is simply dropped); during a live session it also removes that
feed from the running mix, so the slot's audio stops immediately even though
the engine snapshotted the feeds at start.

**Logging:** every request (with its response status), upload, reset and mix is
written (timestamped) to `web-out/logs/webmixer.log` *and* a bounded in-memory
ring, fetchable via `GET /api/logs?n=200` and shown in the page's Logs panel.
Mix entries list every feed (id, volume, pitch, source rate/channels) plus the
result, timing and output sizes; engine decode messages from the
mixer/sources land in the same stream, so failures from the last runs are
always recoverable. The high-frequency poll endpoints are omitted to keep the
tail readable.

The server is a minimal POSIX-socket HTTP/1.1 handler — no third-party code —
with Range support so seeking works, and one mix at a time (a finished job
thread is joined, not destroyed). The rendered files land in `web-out/mix.wav`
+ `web-out/mix.bad`.

## Demo

One command, headless or GUI:

```sh
./demo.sh          # configure + build + mix + render demo.bad / demo.wav
./demo.sh --gui    # the console window instead of a headless run
./demo.sh --clean  # remove the demo outputs
```

The manual form of the headless run:

```sh
./build/mixer --control data/control.txt --wav-out out.wav out.bad
./build/badwav out.bad out.wav        # or play out.wav directly
```

With a display (`-DWITH_GUI=ON`, the `all` preset), the same binary opens the
console — per-strip faders and pitch knobs, level meters with clip LEDs, a drop
zone that builds a feed mid-run (`queue_install`), and the run panel. The GUI is
a `ControlSource` and nothing more: widget moves post the same `ControlEvent`s
the mock file produces, into the same lock-free queue. `--no-gui` forces the
headless path; `--fast` mixes at full speed (useful for long files).

The demo defaults: feed 0 `data/feed0_lead.bad` (44100/16, vol 80), feed 1
`data/feed1_backing.bad` (16000/8, vol 55, **pitch +20%**), feed 2
`data/feed2_instrumental.bad` (96000/32, vol 90). The control file drives
`vol`/`pitch` events at their exact output-sample index. Feed 1 drops out
~20% early  

Regenerate the data (and the golden reference) at any time:

```sh
./build/badgen 44100 16 data/feed0_lead.bad --dur 2 --freq 440 --amp 0.25
./build/badgen 16000  8 data/feed1_backing.bad --dur 2 --freq 220 --amp 0.5
./build/badgen 96000 32 data/feed2_instrumental.bad --dur 2 --freq 330 --amp 0.3
./build/badref --out data/expected_mix.bad --control data/control.txt \
    --rate 48000 --depth 32 --period 10000 --master 100 \
    --volumes 80,55,90 --pitches 0,20,0 \
    data/feed0_lead.bad data/feed1_backing.bad data/feed2_instrumental.bad
```

`badref --check data/expected_mix.bad ...` recomputes the golden and diffs it
against the committed file, proving it is current. `ctest` does this for you.

## Tools

| Tool | Purpose |
|---|---|
| `badgen` | Synthesise test feeds (same writer the mixer's sink uses) |
| `baddump` | Inspect a stream; `--check` validates it and the worked-example constants |
| `badref` | A deliberately naive **second implementation** of the mix spec → the golden file |
| `badwav` | Render `.bad` → WAV for listening |

## Tests

One Catch2 v3 binary per file (25 test binaries + 5 fixture/check tests, +1 GUI
test with `-DWITH_GUI=ON`). The load-bearing ones: `test_interp`, `test_volume`,
`test_equation`, `test_normalize`, `test_ring`, `test_codec`, `test_late`,
`test_resample`, `test_final_sample`, `test_determinism`, `test_flac`,
`test_ogg`, and `test_mix_golden` — the last compares the real-time pipeline
against the offline reference **sample-for-sample and timestamp-for-timestamp
with zero tolerance** (`memcmp`-style). The golden data is committed,
regenerated into the build directory by a CTest fixture before the test, and
proven current by `badref --check`.

## Algorithmic details

The DSP is deliberately small and deliberately exact — every decision is a
worked example in `LLD.md` §3, and the golden test pins the arithmetic to the
bit. The pipeline per period is: *promote → resample → volume → sum → master →
clip → down-convert*.

**Sample domain.** Every depth is promoted into a signed 32-bit `Sample`;
summing happens in signed 64-bit `Accum` so the bus can never wrap. Promotion
is a **multiply, never a shift** — left-shifting a negative signed value is UB
in C++17, and roughly half of any signal is negative:
`8-bit × 2²⁴`, `16-bit × 2¹⁶`, `32-bit` identity. Down-conversion to the sink
depth is the mirror image: `sample / 2^(32−bits)` in int64, dividing toward
zero so negative samples gain no DC offset.

**Volume (Q15).** A 1%-quantised gain `g = round-half-up(percent × 32768 / 100)`,
then `out = (sample × g) / 32768` in int64 — integer division toward zero, not
`>>15`, which would floor negatives by 1 LSB. The brief's example closes
exactly: at 13%, `g = 4260` and `−200 × 4260 / 32768 = −26`. Gains are clamped
to ±int32 *after* the multiply, so boosts past 100% cannot overflow before the
mixer's final limiter.

**Summing bus.** Each feed contributes one `SamplePair` per output sample into a
two-lane int64 accumulator (mono feeds write `{s, s}` to both lanes). The master
gain is applied once: `(Σ · master_gain) / 32768`, then clamped to int32 with a
clip counter.

**Resampling & pitch.** The cursor advances by a per-feed step
`step = src_rate / out_rate × (1 + pitch/100)`; a `+20%` pitch is `step = 1.2` —
, which is why feed 1 ends ~20% early. For an output
position `pos = k + frac`, the window serves `a = s[k]` and `b = s[k+1]`
(`a` if the source has ended and `k+1` is past the end — last-sample hold) and
interpolates linearly in int64, truncating toward zero:
`out = a + (b − a) × frac`. The feed is a half-open interval `[first, end)`, so
a slow feed's final sample serves several output positions; the exhaustion test
is the single comparison `source_ended && pos ≥ end`. Pitch is clamped to
`[−90%, +400%]`.

**Late data.** A frame whose samples all fall behind the mixer's cursor is
dropped and counted (`dropped_samples`); a frame that straddles the cursor has
its past portion skipped. A gap reads as silence. This is how the file feeds
"arriving as if on a network" degrade under real-time scheduling.

**Stereo.** The bus is stereo whenever any live feed is stereo; all-mono runs
are **bit-identical** to the mono-only design because `L == R` through every
stage and `(2x)/2 == x` exactly. The `.bad` sink folds `(L+R)/2` in int64; the
WAV sink writes the bus at its own channel count.

**Real-time contract.** Between `start()` and `stop()` the mixer thread never
blocks and never calls `operator new` (all rings, the accumulator and the
pending/cut vectors are preallocated); allocation-freedom is enforced by a
thread-keyed counting `operator new` in `test_no_alloc`.

## Assumptions register

The format is under-specified; each gap is resolved deliberately:

- **Cross-depth normalisation** — per-depth full scale, promoted into int32 by
  multiply (never shift: left-shifting a negative signed value is UB).
- **Sample signedness** — signed two's complement at all depths, including 8-bit.
- **Endianness** — little-endian on disk; hex in the spec is MSB-first for
  reading only.
- **Channel count absent** — the container is mono and stays mono (the 16-byte
  header is not extended). Stereo exists *above* the sink: the bus is stereo
  whenever any feed is stereo, `WavSink` renders it, and the `.bad` sink folds
  `(L+R)/2` in int64. An all-mono run is bit-identical to the mono-only design.
- **Block time semantics** — the timestamp of the first sample in the block.
- **Gaps and overlaps** — gaps are silence; for overlaps, keep the earlier
  block and drop the overlapping region of the later one.
- **Stream time zero** — the earliest block timestamp across all feeds, sampled
  once at startup.
- **Output block timestamps** — normalised timeline, origin = stream zero.
- **No version field, EOS marker or checksum** — deliberately absent.

## Layout

```
src/core/       types, interfaces, spsc_ring, frame_pool, clock, config,
                stats, telemetry, feed_window, feed, mixer
src/codec/      UltraHighSpeedCodec_reader, UltraHighSpeedCodec_writer
src/filters/    volume, resample
src/io/         file_source, sine_source, UltraHighSpeedCodec_sink, wav_sink
src/control/    file_control
src/codecs/     registry, mp3_source, wav_source, flac_source, ogg_source
src/ui/         gui_control, widgets, feed_loader, console   (WITH_GUI)
src/web/        server.hpp/cpp, main.cpp — the browser hookup (webmixer)
src/app/        main.cpp — the only TU that names concrete types
tests/          23 Catch2 binaries + 5 golden fixture/check tests (+1 GUI test with WITH_GUI)
tools/          badgen, baddump, badref, badwav
data/           committed demo feeds, control file, golden reference
```

`main.cpp` is the only file that names concrete types. Everything below the four
seams in `src/core/interfaces.hpp` is interchangeable. The mixer publishes a
`Telemetry` snapshot every period (peak-hold preserved across dropped frames)
and supports runtime feed install/remove via a lock-free command ring.
