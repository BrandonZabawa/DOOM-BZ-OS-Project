# Repository Learning + pthreads On-Ramp (Linux)

## 1) What CS topics this repository uses

The `linuxdoom-1.10/FILES2` index gives a very clear map of the core computer science areas in the engine:

- **Game loop and event processing** (`g_game.c`, `d_main.c`).
- **Rendering pipeline**: BSP traversal, clipping, texture cache, visplanes, sprite rendering (`r_bsp.c`, `r_main.c`, `r_plane.c`, `r_segs.c`, `r_things.c`).
- **AI/simulation**: enemy logic, movement/collision, LOS/REJECT checks, thinker/ticker model (`p_enemy.c`, `p_map.c`, `p_sight.c`, `p_tick.c`).
- **Networking**: higher-level net protocol (`d_net.c`) and legacy drivers (`ipx/`, `sersrc/`).
- **Memory management**: custom zone allocator (`z_zone.c`).
- **File formats / data-oriented programming**: WAD lump I/O (`w_wad.c`, `wadread.c`) and lookup tables (`tables.c`, `info.c`).
- **Systems programming**: Linux/X11 interfaces and sound device handling (`i_*.c`, `i_sound.c`, `sndserv/*`).

Good evidence files:
- module map: `linuxdoom-1.10/FILES2`
- engine intent and architecture comments: `README.TXT`
- explicit TODOs discussing algorithmic/rendering/network directions: `linuxdoom-1.10/TODO`

## 2) Best starter files for introducing pthread-based multithreading

For a first Linux pthread experiment, prefer **sound** over gameplay/rendering state, because gameplay and rendering share lots of mutable globals and pointer-heavy structures.

### Lowest-risk starting point

1. **`sndserv/soundsrv.c`**
   - Already a dedicated sound process with a command loop + mixing/update loop.
   - Natural split:
     - Thread A: command reader/parser (stdin protocol)
     - Thread B: audio mixer/output loop
   - This isolates race scope mostly to channel/mix data structures in this file.

2. **`linuxdoom-1.10/i_sound.c`**
   - Already has comments around synchronous vs asynchronous updates and timer experiments.
   - Can be used to compare against threaded soundserver behavior and simplify experimentation boundaries.

3. **`sndserv/linux.c`** (if needed for backend writes)
   - Keep I/O backend unchanged first; only thread orchestration around existing calls.

### Avoid initially

- `p_tick.c`, `g_game.c`, `d_main.c`: core simulation tick and gameplay are tightly coupled to global state.
- `r_main.c` and render modules as first target: rendering is heavily stateful and depends on ordering of shared buffers.

## 3) A concrete “first issue” to learn engine + pthreads

### Proposed issue title

**“Add optional pthread command/mix split to `sndserv` with baseline perf logging”**

### Scope (small but educational)

- Add `-pthread` build support for `sndserv/Makefile`.
- In `sndserv/soundsrv.c`, behind a compile-time flag (e.g. `PTHREAD_SOUNDSRV`):
  - Move command handling (`read/select` loop) to one thread.
  - Keep `updatesounds()` / mixing in another thread.
  - Protect shared channel state with `pthread_mutex_t`.
  - Use a minimal queue or shared command slot + condition variable.
- Add simple counters/timestamps:
  - commands processed/sec
  - mix iterations/sec
  - average command-to-audible latency proxy (timestamp at enqueue/dequeue)
- Preserve legacy single-thread mode as default so behavior is easy to compare.

### Learning outcomes

- Understand engine/runtime boundaries without destabilizing gameplay loop.
- Practice lock design around real-time-ish audio data.
- Establish a repeatable baseline to compare single-thread vs threaded sound path.

### “Definition of done”

- Builds both modes (`legacy`, `pthread`)
- Runs without deadlocks for a full demo or timed session
- Logs comparative metrics for both modes
- No gameplay code touched yet

## 4) Useful debugging + CLI toolchain for this repo

- **`gdb`**: crash/backtrace analysis (historically mentioned in project notes).
- **`valgrind`** (if available): memory errors/races (`helgrind`/`drd` for threading checks).
- **`perf`**: CPU hotspot and sampling comparison between single-thread and pthread modes.
- **`strace`**: inspect syscalls in sound server / I/O and timing behavior.
- **`time`**: quick wall-clock comparison.
- **`rg`**: fast symbol/topic discovery (preferred over recursive grep in large trees).
- **`grep`**: quick small-file filtering.
- **`git`**: commit discipline and patch isolation (`git add -p`, `git log --oneline`, `git diff --stat`).
- **`make`**: build system entry point.

Useful commands:

```bash
rg -n "D_DoomLoop|TryRunTics|P_Ticker|R_RenderPlayerView|S_UpdateSounds" linuxdoom-1.10
rg -n "pthread|threaded|sndserver|mix|select" sndserv linuxdoom-1.10
make -C linuxdoom-1.10
make -C sndserv
gdb --args linuxdoom-1.10/linux/linuxxdoom
perf stat -- ./linuxdoom-1.10/linux/linuxxdoom
```

## 5) Version-control style expected by this codebase

There is no modern `CONTRIBUTING.md`, but repository history and source metadata strongly indicate these conventions:

- **Historically CVS-oriented metadata** (`$Id$`, `$Log$`) is embedded in many source headers.
- **`ChangeLog`-style narrative entries** are used to explain changes and TODOs.
- Existing git history uses **short, imperative/statement-style commit subjects**.
- `ChangeLog` notes that obsolete CVS logs were removed to provide a clean slate, implying maintainers valued clean revision history.

Practical recommendations to avoid structural disruption:

- Keep commits small and subsystem-scoped (e.g., only `sndserv/*` for first pthread issue).
- Preserve existing file header/comment style when editing C files.
- Avoid large cross-module refactors in first threading PR; prefer feature-flagged incremental changes.
- Keep commit messages concise and specific, similar to existing git log style.

