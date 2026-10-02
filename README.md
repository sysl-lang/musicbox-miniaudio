# musicbox-miniaudio

A [musicbox](https://github.com/sysl-lang/musicbox) score played through
[miniaudio](https://github.com/sysl-lang/miniaudio) — module `sh.sysl.musicbox_miniaudio`.

```hocon
dependencies {
  musicbox           { git = "github.com/sysl-lang/musicbox", version = "0.1.0" }
  musicbox-miniaudio { git = "github.com/sysl-lang/musicbox-miniaudio", version = "0.1.0" }
}
```

The module is `sh.sysl.musicbox_miniaudio` rather than `sh.sysl.musicbox.miniaudio` because one
package's module path may not nest inside another's.

## Playing a score

A C-major arpeggio, up and back down, on the default device. This is a whole program — a
`main.sysl` beside the manifest above, with no `module` line:

```sysl
import sh.sysl.musicbox.{score, note, pitch, instrument, Partial, adsr}
import sh.sysl.musicbox_miniaudio.player

val bell = [instrument([Partial(1.0, 0.6), Partial(2.0, 0.2), Partial(3.0, 0.1)],
                       adsr(0.01, 0.15, 0.6, 0.3)).unwrap()]

// C4 E4 G4 C5 G4 E4 C4, an eighth of a second apart.
val keys = [60.0, 64.0, 67.0, 72.0, 67.0, 64.0, 60.0]
val notes = [note(0.0, 0.3, pitch(keys[0])), note(0.125, 0.3, pitch(keys[1])),
             note(0.25, 0.3, pitch(keys[2])), note(0.375, 0.3, pitch(keys[3])),
             note(0.5, 0.3, pitch(keys[4])), note(0.625, 0.3, pitch(keys[5])),
             note(0.75, 0.6, pitch(keys[6]))]

val p = player(score(notes, bell).unwrap()).unwrap()
val hw = p.device().playback_hardware()

print(s"${hw.name}: ${p.format()} x${p.channels()} at ${p.sample_rate()} Hz")
p.play_until_done()
print(s"played ${p.played()} samples, ${p.underruns()} underruns")
```

On a MacBook Pro, through CoreAudio, it printed `MacBook Pro Speakers: f32 x2 at 48000 Hz` and
`played 78304 samples, 0 underruns` — 1.6 s, the last note's release included.

`play(score)` is the same in one call. A program with a loop of its own — a game, a UI — keeps the
handle instead:

```sysl
val p = player(sc)?
loop
    p.pump()                 // render ahead; once a frame is plenty
    if p.done() then break
    ...the program's own work...
```

## The API

| | |
|---|---|
| `player(score, backends = [], format = F32, channels = 0, monitor = None) -> Result[&Player, PlayError]` | open the device at **its own rate**, build the synth at that rate, fill the ring, start |
| `play(score, backends = []) -> Result[unit, PlayError]` | all of it, returning once the last note has died away |
| `p.pump()` | render as much as the ring has room for |
| `p.done()` | every sample has been played, or the player was stopped |
| `p.play_until_done()` | pump every 5 ms until done, then close the device |
| `p.stop()` | close the device now; the callback has stopped when it returns |
| `p.played()`, `p.underruns()` | samples played; frames that went silent because the ring ran dry |
| `p.sample_rate()`, `p.format()`, `p.channels()`, `p.device()` | what was opened |

`format` is `F32` or `S16` — miniaudio converts to whatever the hardware takes — or `Unknown` for the
hardware's own, refused as `Unsupported` unless that is one of the two. `channels = 0` is the
device's own count, and every channel gets the same mono sample. `monitor` is called on the audio
thread after each period is written, with the same `Frames`: a level meter, or a test reading what the
device was handed. Dropping the last `&Player` closes the device.

## Why the synth stays on the program's thread

miniaudio's data callback runs on its own audio thread, so it is a `&sync Fn` and the compiler checks
what it captures. **A `Synth` cannot be one of those things.** It holds its score, and a score holds
the caller's notes as views, which own their elements through a count that is not atomic:

```
error: '&sync Holder' may be reached from two domains at once, so every count inside it has to be
atomic — but its 's.score.notes' reaches a '[]const Note', which owns its elements through a count
that is not atomic.
```

That is the right answer rather than an obstacle, and the shape it leaves is the conventional one
for audio anyway: **a single-producer, single-consumer ring of samples.** The ring is a fixed array
of 16,384 `i16` (a third of a second at 48 kHz) and two `Atomic[u64]` counters in a `&sync` box —
nothing in it has a count that could race. `pump` renders the score into it on the program's thread;
the callback copies out, converting to the device's format and duplicating across its channels, and
writes silence for whatever the ring cannot supply. Each side reads the other's counter with
`Acquire` before touching a slot and publishes its own with `Release` after, so there is no lock and
nothing the audio thread can wait on; it never runs anything slower than a copy.

The callback owns a share of the ring, so the ring outlives the device whatever order a program
lets go of things in — there is no `Drop` to get wrong. Handing the callback a raw `*Synth` would also
have compiled (a `*T` carries no count, so the capture check looks through it), but it would put
rendering on the real-time thread and the synth's lifetime on the honour system.

## Tests

`sysl test .` — 11 tests, every device on miniaudio's **null backend** (a real audio thread paced by
the clock, playing into nothing), read back through the `monitor` seam:

- **What the device was handed is what musicbox renders**, sample for sample, for f32 stereo, s16
  mono, s16 on three channels and f32 on six — the reference rendered in blocks of a different size
  from the player's — with every channel of every frame equal and silence after the last sample.
- A score **longer than the ring** played to the end through `play_until_done`: wrapping, with no
  gap and no underrun.
- **Underrun is silence**: a ring filled once and never pumped again plays exactly its 16,384
  samples, then zeros — every slot by then holds a stale sample, so reading past the end would be
  loud — and `underruns` counts it.
- **Stop mid-score** closes the device at once: no callback after `stop`, and `pump` and a second
  `stop` do nothing. **Dropping** a playing player closes its device.
- The device's own format; an unsupported one refused; `play` returning once finished.

**The mutation pass**: reading past what was written (dropping the bound on a shortfall) reddens the
underrun test; writing only the first channel reddens all four multi-channel tests.

**AddressSanitizer and ThreadSanitizer** — both green over the suite, 11 passed and no reports, with
the instrumentation confirmed: `nm -u` of the ASan binary shows 41 asan imports including
`___asan_memcpy`, and of the TSan one 10 `__tsan_read`/`__tsan_write` imports and
`__tsan_atomic64_load`. miniaudio's vendored C is instrumented too; CoreAudio is not.

**Run each sanitizer against a package cache of its own** (`XDG_CACHE_HOME=<fresh dir>`). The
prebuilt standard module is cached under a key that does not include `SYSL_EXTRA_CFLAGS`, so a
sanitized run after an ordinary one links an *uninstrumented* standard module. Under TSan that is
not merely weaker but wrong: `Atomic[u64].load` is called out of line into it, so the acquire never
reaches TSan, and it reports the ring's correctly ordered slot accesses as races. The tell is an
`nm -u` with no `__tsan_atomic64_load`.

**`fill` copies into the ring by index, and that is the language's intended form.** A slice does not
record whether its owner's count is atomic, so storage inside a `&sync` box cannot be sliced
(`reference/arrays.md`) — `r.samples[a..<b]` on the ring is refused. The synth renders into a block
of `fill`'s own, and indexing the ring's array copies it across without taking a share of the box.
This is why the package states `sysl = "0.0.156"`, the release that states the rule.

## Licence

MIT.
