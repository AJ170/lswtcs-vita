# LEGO Star Wars: TCS (PS Vita) — audio, input and build fixes

Write-up of a debugging session on [gm666q/lswtcs-vita](https://github.com/gm666q/lswtcs-vita).
Starting complaint: *"most of the audio sounds and music is not present, and no audio in cutscenes"*.
That turned out to be four separate bugs, all in the OpenSL ES implementation
([gm666q/opensles](https://github.com/gm666q/opensles), `lswtcs` branch) rather than in the loader.

Everything below was verified on real hardware (3.60 enso), and the reasoning behind each fix is
included so you can disagree with it rather than just take it.

**Result:** all audio works — SFX, music, cutscenes — with correct pitch, no dropouts, and no
degradation over a play session. Diagonal analog movement also fixed.

---

## TL;DR — the headline bug

`SL_PLAYEVENT_HEADATEND` **was never fired**. In `libopensles`, that constant appears in exactly one
place: the argument validation inside `IPlay_SetCallbackEventsMask`. Nothing in the
buffer-queue / OutputMixExt path ever raises it.

The game registers `NuVoiceAndroid::PlayerCallback(SLPlayItf const*, void*, unsigned int)` and waits
for that event to learn a sound has finished so it can return the voice to its pool. On Android the
event arrives; on Vita it never did. So **every voice leaked**, the pool drained after ~18 sounds,
and the game stopped requesting audio entirely — which is why "most audio is missing" and why it got
worse the longer you played.

Telemetry that proved it (counters compiled into a throwaway build):

```
try=18  fail=0   players=18-1=17   tracks=17/32   qState=0   qPos=26316   qBQ=26316   cb=49(nocb=49)
```

* `try` frozen at 18 with `fail=0` — the game **stopped asking** for players; it wasn't being refused.
* `qState=0` — it never calls `GetPlayState`, so forcing players to STOPPED achieves nothing.
* `nocb == cb` — every buffer completes with **no buffer-queue callback registered**; it doesn't use that route.
* `qPos == qBQ` climbing in lockstep (~920/s) — it polls `GetPosition` + queue count every frame per voice.

The binary at `ux0:data/lswtcs/libTTapp.so` is **not stripped**. Disassembling it is far more
productive than guessing — that's how the callback was found.

---

## 1. `IBufferQueue_Enqueue` — resampler (silence + wrong pitch)

**Was:**

```c
int num_cycles = (44100) * 1000 / this->samplerate;   /* integer division */
/* ... then duplicate each sample num_cycles times ... */
```

Failure modes:

| Source rate | `num_cycles` | Result |
|---|---|---|
| 48000 Hz | **0** | `mSize == 0` → silent, and a zero-length buffer at the queue head stalls the mixer |
| 32000 / 24000 Hz | 1 (truncated from 1.38 / 1.84) | plays **too fast** |
| 44100 / 22050 / 11025 Hz | exact | fine — which is why *some* audio always worked |

**Now:** a 32.32 fixed-point resampler — linear interpolation for 16-bit, nearest for 8-bit —
handling any ratio up or down, mono→stereo and 8→16-bit.

One subtlety worth keeping if you rewrite this:

```c
/* (l1 - l0) spans +/-65535 and frac spans 0..65535, so this product overflows a
   signed 32-bit multiply on loud material and wraps into audible clicks. */
*dst++ = (int16_t)(l0 + (int32_t)(((int64_t)(l1 - l0) * frac) >> 16));
```

## 2. Conversion-buffer lifetime (use-after-free / silence)

**Was:** one global ring, shared by every sound in the game:

```c
#define NUM_BUFFERS 32
void *avail_buffers[NUM_BUFFERS] = {NULL};
int avail_buffers_idx = 0;
```

Buffers were freed and reused blind while still queued, and the index was mutated from multiple
threads with no synchronisation. Long sounds (music, cutscene streams) sit in the queue while the
ring wraps underneath them, so their memory was freed and `calloc`-zeroed mid-playback — silence
for streams, mostly-OK for short SFX. Also a plausible contributor to the "intermittent crashing"
in the README.

**Now:** ownership moved onto each queue slot —

```c
typedef struct {
    const void *mBuffer;
    SLuint32 mSize;
    void *mConverted;   /* implementation-owned; NULL if mBuffer is the app's own buffer */
} BufferHeader;
```

freed when the buffer is retired by the mixer, when the slot is refilled, and in
`IBufferQueue_Destroy`. `IEngine.c` switched `malloc` → `calloc` for `mArray` so `mConverted`
starts NULL. Freeing on retire matters: vitaGL leaves only ~18 MB of user RAM outside its own
pools, and an 11 kHz mono sound expands **8×** when resampled to 44.1 kHz stereo.

## 3. Leaked audio thread per engine (permanent audio loss)

The game **destroys and rebuilds its whole OpenSL ES engine** on room transitions. In `Vita.c`:

```c
CEngine_Realize()  -> SDL_open()  /* spawns a for(;;) playback thread + opens an audio port */
CEngine_Destroy()  -> SDL_close() /* was completely empty */
```

Nothing ever stopped the old thread or closed its port, so every rebuild added another. They all
read the global `slEngine` and call `IOutputMixExt_FillBuffer` on the same mix — and FillBuffer
*consumes* queued buffers, so the extra threads eat the audio the real one should play. Worse, if
the extra port fails to open, `sceAudioOutOutput` stops blocking and that thread spins flat out.

**Fixes:** one playback thread per process (later engines just re-point `slEngine`);
`SDL_close(IEngine *)` clears `slEngine` only if it is the engine being destroyed (signature change,
`CEngine.c` + header updated); the thread re-reads `slEngine` every cycle and tolerates NULL; it
exits instead of spinning if `sceAudioOutOpenPort` fails.

Diagnostic signature: two `CreateOutputMix ... connected to engine` events with **no** intervening
destroy ⇒ `mOutputMix` was reset by `IEngine_init` ⇒ a second engine exists.

## 4. `SL_PLAYEVENT_HEADATEND` (the voice-pool leak)

In `track_check`, when a playing track's queue has been empty for `LSWTCS_STARVED_CYCLES`, fire the
player's callback with `SL_PLAYEVENT_HEADATEND`. `Track.mEndReported` makes it fire once per
playback run, cleared whenever fresh data arrives.

Two things that matter:

* **Capture the callback under the object lock, invoke it after unlocking** — the game's handler
  re-enters OpenSL ES.
* **The debounce must be expressed in milliseconds, not mixer cycles.** It was originally a raw
  cycle count, so its real duration silently changed every time the buffer size was retuned
  (46 ms → 186 ms → 93 ms). That matters more than it sounds:

| Debounce | Symptom |
|---|---|
| 93 ms | looping sounds audibly restart; **crash** in `SetLevelSfxBits` at level transition |
| 46 ms | no crash, loops still audibly restart |
| 12 ms | current setting |

The delay is pure latency — the game re-arms looping sounds from this event, so it is dead air at
every loop point, and it is also how long a finished voice sits unreported in the game's own tables.
The 93 ms crash was a NULL deref of one of the game's SFX tables, presumably from that stale state.

```c
#define LSWTCS_STARVED_MS 12u
#define LSWTCS_STARVED_CYCLES ((LSWTCS_STARVED_MS * 44100u / 1000u) / (SndFile_BUFSIZE / 4u))
```

## 5. Output-mix teardown (two latent hangs)

Both under `#ifdef LSWTCS`:

* `IOutputMixExt_FillBuffer`'s destroy path set `engine->mOutputMix = NULL` and explicitly declined
  to connect another mix ("we don't attempt to connect another output mix, even if there is one"),
  while the audio thread emits pure silence whenever it is NULL. If the app builds its replacement
  mix *before* destroying the old one, audio is dead permanently. Now hands over to any other live
  mix (scan `engine->mInstances` for `mClass->mObjectID == SL_OBJECTID_OUTPUTMIX`).
* `COutputMix_PreDestroy` blocked unconditionally for a mixer acknowledgement that only ever arrives
  for the engine's **current** mix — destroying any other mix hangs that thread forever. Now skips
  the handshake when there is no mixer on the other end.

## 6. Buffer size / thread priority (clicks vs stutter)

A click exactly once per second turned out to be **output starvation**, not clipping and not the
resampler (the music is 44100 Hz stereo and never touches it; SFX ratios are exact 4×/2×). The
mixer had to deliver every **2.9 ms** with only 5.8 ms of slack, at default thread priority — so the
game streaming the next second of music off the memory card blew the deadline.

There is a genuine trade-off here, because `CAudioPlayer_PreDestroy` **blocks the calling thread**
until the mixer's next wake, and the Vita's audio port only ever queues two grains, so headroom and
wake-interval are coupled:

| Grain | Port headroom | Worst `Destroy` stall | Cost at ~7 destroys/sec |
|---|---|---|---|
| 128 frames (stock) | 5.8 ms | 2.9 ms | 2% — but clicks |
| 512 frames | 23.2 ms | 11.6 ms | **8% — visible stutter** |
| **256 frames (current)** | 11.6 ms | 5.8 ms | 4% |

Note the destroy stall only became significant *after* the HEADATEND fix, because that's what made
the game actually recycle voices (~7 per second during action).

Raising the audio thread priority also fixes clicks, but the game's HEADATEND handler runs **on that
thread**, so an elevated priority lets audio work preempt rendering and visibly costs frames. It is
deliberately left at `SCE_KERNEL_DEFAULT_PRIORITY_USER`.

**The proper fix, not attempted:** make `PreDestroy` not block — unlink the track from the app
thread under the output-mix lock, taking locks in `FillBuffer`'s order (outputMix → player) to avoid
deadlock. That would decouple buffer size from stutter entirely and allow a larger buffer, killing
the residual cutscene clicking. It's the obvious next improvement for anyone continuing this.

## 7. Also in `libopensles`

* Mixer adds now saturate instead of wrapping int16 (`saturate16`), so overlapping loud tracks clip
  rather than invert.
* `IPlay_GetPosition` — **keep the wall-clock hack.** Replacing it with a frames-mixed position is
  the "obviously correct" change and it **hangs the game**: the engine polls this to drive cutscenes,
  and when the stream's queue drains at the end of one, a frame-based position stops advancing so
  the cutscene never terminates. The only change kept was making PLAYING→PAUSED *accumulate* into
  `mPosition` instead of overwriting it.

---

## Loader change (`lswtcs-vita`)

Only one, in `source/main.c` — diagonal analog movement made the character walk instead of run.

`NuInputDevicePS::ReadAnalogValuesPS` (libTTapp.so `0x5b7dc8`) computes `analog + d-pad` per axis and
clamps **each axis independently** to ±1. The d-pad supplies ±1 per axis, so a d-pad diagonal is
`(1,1)` — the game expects a **square** range. The Vita's round stick gate only reaches ~`(0.7,0.7)`
at full diagonal, which reads as a partial push.

Radial/Euclidean normalisation does **not** fix this (tried first — unit *length*, but only 70% on
each axis, and the game never measures length). The fix is a radial deadzone followed by mapping the
circle onto the unit square, scaling the direction vector by `1/max(|x|,|y|)`:

| Direction | Before | After |
|---|---|---|
| Full up | `(0.00, -1.00)` | `(0.00, -1.00)` |
| Full diagonal | `(0.71, -0.71)` — walks | `(1.00, -1.00)` — runs |
| Centred | `0.030` drift | `0.000` |

Tunables `ANALOG_DEADZONE 0.18` / `ANALOG_SATURATION 0.72`. The deadzone is applied to the vector
length, so stick drift is dead in every direction rather than only near the axes.

---

## Build environment (this took a while to get right)

The port needs a **softfp** SDK. A hardfp vitasdk cannot link it at all.

| Component | Pin |
|---|---|
| vitasdk-softfp | autobuilds `master-win-v2.18` (2024-04-25, GCC 10.3) |
| vitaGL | `176a4a0` (2024-10-09), built **`SOFTFP_ABI=1 HAVE_SBRK=1 NO_DEBUG=1`** |
| vitaShaRK | `5d99fe81` |
| libsndfile | 1.0.28 (the old VITABUILD recipe) |
| kubridge stubs | the ones vendored in `lib/kubridge/` |

**`SOFTFP_ABI=1` is mandatory and is not in the vdpm package recipe.** Without it you get a black
screen with working audio and input, and it is not obvious why: it defines `HAVE_SOFTFP_ABI`, which
swaps `sceGxmSetViewport` for the `sceGxmSetViewport_sfp` assembly thunk. `sceGxmSetViewport` is a
**hardfp firmware function** that reads its floats from VFP registers; softfp-built vitaGL passes
them in integer registers, so without the thunk the viewport gets garbage and the GPU draws
everything off-screen. Audio and input are unaffected because they don't pass floats.

Other traps:

* Build from an msys2 login shell. Exports from Git Bash don't reach msys2 children and gcc then
  fails with `Cannot create temporary file in C:\Windows\`.
* `shaders/gxp/` is not in the repo — extract it from a release vpk. The 412 GXP files are portable
  across the vitaGL versions tested (v0.1 and a 2026 community build ship byte-identical sets).
* libsndfile 1.0.28's configure mis-detects the cross target on msys2; `sf_count_t` /
  `SIZEOF_SF_COUNT_T` must be hand-fixed in the generated `src/sndfile.h` and `src/config.h`.

---

## Findings that are not fixes, but will save you time

**Don't hook a per-draw function with `SO_CONTINUE`.** It rewrites the target's instructions on
every call (not thread-safe, not reentrant) *and* calls through an unprototyped
`((type(*)())addr)(...)` pointer, so a `float` parameter is default-promoted to `double` and shifts
every argument after it. Hooking `GameAnimSet_DrawReflection` crashed instantly at level load. It's
fine for the once-per-thread functions `patch.c` already hooks.

**`nativeSetCaps` is not a graphics switch.** The relocation shows it writes `g_flashAvailable`
(Adobe Flash availability). Passing 0 is correct.

**Reflections — investigated, no bug found.** The engine has a full reflection pipeline and
`Reflections_On` (exported, 1 byte) defaults to 1. A one-shot probe confirmed
`GameAnimSet_DrawReflection` **is reached** and `glCopyTexImage2D` **is called** — in a *window*
context, so the `is_pbuffer` skip in `glCopyTexImage2D_soloader` is dead code and re-enabling it
changes nothing. Every shader has a compiled GXP (no `glsl/` dumps on the card), all 26 JNI natives
are called, and there is no device/GPU quality tiering in the binary. Most likely the Android build's
level data simply has no reflective surfaces. **Confirm the stock release vpk shows reflections
somewhere before spending time on this.**

**Engine render globals are exported and writable** from the loader via
`so_symbol(&so_mod, ...)` after `so_file_load` — plain data stores, no hooking needed:
`Reflections_On`, `CHARSHADOWS_ON`, `COMPLEXSHADOWS`, `debris_detail_level`, `JointRotation_On`.
A settings framework already exists (`source/utils/settings.c`, `ux0:data/lswtcs/config.txt`), but
nothing ever calls `settings_save()`, so the file is never created — the `-config` LiveArea hook
launches `app0:configurator.bin`, which was never written.

---

## Files touched

`libopensles` (334 insertions, 81 deletions):

```
 CEngine.c              |   4 +      SDL_close(&this->mEngine)
 COutputMix.c           |  13 +      skip handshake for non-current mix
 IBufferQueue.c         | 159 +--     resampler + per-slot buffer ownership
 IEngine.c              |  13 +-     calloc for mArray
 IOutputMixExt.c        | 108 ++-    HEADATEND, saturating mix, free-on-retire, mix handover
 IPlay.c                |   9 +-     pause accumulates into mPosition
 OutputMixExt.h         |   4 +      Track.mEmptyCycles / mEndReported
 Vita.c                 |  68 ++-    single audio thread, SDL_close(IEngine*), grain, port check
 sles_allinclusive.h    |  37 ++-    BufferHeader.mConverted, SndFile_BUFSIZE, debounce macros
```

`lswtcs-vita`: `source/main.c` only (analog stick + `power_callback` returning `int`).

## Known remaining issues

* Faint clicking during cutscenes — streamed audio starving at the current buffer size. Fixing it
  properly means the non-blocking `PreDestroy` described in §6, not a bigger buffer (that trades it
  straight back for stutter).
* Looping sounds may still be slightly audible restarting; the debounce is a single tunable
  (`LSWTCS_STARVED_MS`).
* Reflections do not appear — believed to be absent from the Android data rather than a port bug.
