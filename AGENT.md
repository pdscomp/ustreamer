# AGENT.md

Scratch note for ongoing RAM-reduction work on `ustreamer`. Do not commit this file.

## Repo / branch state

- Current branch intended for this work: `paul/reduce-ram`
- Branch base commit: `88460b72e191035d04355e25106af817cbfe069e`
- That commit is the one pinned by Cosmos in:
  - `/home/paul/sandbox/cosmos/meta-opencentauri/recipes-apps/ustreamer/ustreamer_6.55.bb`
- Current source state on the branch:
  - `dc5c80d` = `Trim uStreamer memory overhead`
  - `4ef9593` = `Reduce uStreamer frame backlog`
  - `64f7fd6` = `Lower uStreamer thread stack cap`
  - `2db118c` = `Trim remaining frame overhead`
  - `117a314` = `Add RAM usage log`
- Scratch-only files that should stay uncommitted:
  - `AGENT.md`
  - `gen-cosmos-patch.sh`

## User goal

The user is running:

```sh
/usr/bin/ustreamer -r 864x480 -c HW -m MJPEG -b 1 -w 1 -f 8 -l --host=0.0.0.0 --allow-origin=*
```

They report about 10 MiB RAM total and want to know whether low-hanging changes could significantly reduce memory usage, ideally something like a 2x cut, without a full rewrite.

## Important conclusion up front

`ustreamer` is already a single-process design for the main streaming path. It is not using `fork()` for streaming clients.

The most important architectural facts:

- Main thread creates only two top-level threads:
  - stream loop
  - HTTP loop
  - See `src/ustreamer/main.c:105-110`
- Capture already waits with `select()`
  - See `src/libs/capture.c:504-528`
- HTTP already runs on a libevent event loop, not a process-per-client model
  - See `src/ustreamer/http/server.c:137-140`

So "replace fork with threads" is not the answer here. The memory opportunities are elsewhere.

## What threads actually exist

### Always present in normal `ustreamer`

- main thread
- stream loop thread
- HTTP event-loop thread

### Created inside stream loop

See `src/ustreamer/stream.c:170-203`.

For each capture session, the stream loop creates:

- one releaser thread per V4L2 capture buffer
  - `n_releasers = cap->run->n_bufs`
  - `src/ustreamer/stream.c:176-186`
- one JPEG worker-thread context
  - `src/ustreamer/stream.c:197`
- optional RAW thread if `raw_sink != NULL`
  - `src/ustreamer/stream.c:198`
- optional H264 thread if `h264_sink != NULL`
  - `src/ustreamer/stream.c:199`
- optional DRM thread if V4P/DRM output is enabled
  - `src/ustreamer/stream.c:200-201`

### Encoder worker pool

The JPEG worker context itself feeds jobs into a worker pool. For HW JPEG input, worker count collapses to 1:

- `src/ustreamer/encoder.c:126-130`

This matters because the user is running `-c HW -m MJPEG -b 1 -w 1`, so the active thread count is already near the minimum practical design for this codebase.

With that exact config, expected active threads are roughly:

- main
- stream loop
- HTTP loop
- 1 releaser thread
- 1 JPEG worker-context thread
- 1 encoder pool worker

No RAW/H264/DRM threads in that config unless additional sinks/features are enabled.

## Why the user may think there are "forks"

Inside the code we inspected, the main streamer path does not `fork()`.

If the user's memory accounting is "across all the fork()s", they are probably observing:

- multiple threads and treating them as tasks/processes in a system monitor, or
- additional service-manager/helper processes outside this repo, not internal `ustreamer` worker processes

For the work inside this repo, assume one process plus multiple threads.

## Memory model by subsystem

### 1. Capture buffers

Capture buffer count defaults to `us_get_cores_available() + 1`:

- `src/libs/capture.c:148`

But the user explicitly runs `-b 1`, so they already force one capture buffer.

For MMAP capture:

- V4L2 device buffers are requested from the kernel
  - `src/libs/capture.c:921-937`
- Each buffer is mmapped into userspace
  - `src/libs/capture.c:965-976`

Important nuance:

- these mapped buffers may show up in memory accounting
- but they are not heap copies introduced by `ustreamer`
- with `-b 1`, this part is already minimized

For USERPTR capture, the app would allocate user buffers directly:

- `src/libs/capture.c:1009-1018`

But the current default and user config are MMAP, not USERPTR.

### 2. Thread stacks

This is the cleanest low-risk place to cut overhead.

Before the local patch, all internal threads used:

```c
pthread_create(..., NULL, ...)
```

through `US_THREAD_CREATE`, which historically meant default libc stack size:

- original macro location: `src/libs/threading.h`

On glibc systems, default thread stack reservation is commonly large, often 8 MiB virtual per thread. Even when actual RSS is much lower, this can distort memory accounting depending on what the user is measuring:

- `VSZ` and some task views will look large
- `RSS` may be much smaller
- `PSS` depends on the tool

Current local patch:

- replaces default `pthread_create(..., NULL, ...)`
- uses `pthread_attr_setstacksize()`
- sets `US_THREAD_STACK_SIZE` default to 512 KiB

This patch is in `src/libs/threading.h`.

Expectation:

- likely helpful if the user's "10 MiB" includes thread stack mappings or task totals
- unlikely by itself to reliably halve true resident heap usage

### 3. JPEG frame copies

This is the strongest likely source of real userspace memory overhead in the user's configuration.

The frame abstraction is heap-backed and grows as needed:

- `us_frame_init()` starts at 32 KiB
  - `src/libs/frame.c:35-40`
- `us_frame_set_data()` reallocates and `memcpy()`s
  - `src/libs/frame.c:55-58`
- `us_frame_copy()` is a full deep copy
  - `src/libs/frame.c:68-70`

In the HW MJPEG path, even though the camera already supplies JPEG, the code still creates multiple userspace copies as the frame moves through the HTTP pipeline.

#### Copy path in the user's config

Likely path for one live JPEG frame:

1. V4L2 MMAP capture buffer contains camera-provided MJPEG
2. HW encoder worker copies from capture buffer into `job->dest`
   - HW encoder still calls `us_frame_set_data()`
   - see `src/ustreamer/encoders/hw/encoder.c:67-72`
3. `_stream_expose_jpeg()` copies `job->dest` into a slot of the HTTP JPEG ring
   - `src/ustreamer/stream.c:716-732`
   - actual deep copy at `src/ustreamer/stream.c:726-727`
4. HTTP refresher eventually copies ring frame into `server->run->exposed->frame`
   - server allocates `exposed->frame` at init
   - `src/ustreamer/http/server.c:112-116`
   - updated by deep copy in `_expose_frame()`
   - `src/ustreamer/http/server.c:954-985`

So for live JPEG data, you should assume at least:

- 1 kernel/MMAP capture buffer
- 1 encoder worker destination frame
- 1 HTTP ring slot holding the current frame
- 1 HTTP "exposed" frame

In addition, the ring has capacity 4 by default:

- `US_RING_INIT_WITH_ITEMS(http->jpeg_ring, 4, us_frame_init);`
- `src/ustreamer/stream.c:107`

Those slots start at 32 KiB each but grow to peak frame size and then stay allocated. Even if only one is current, previous slots may remain expanded.

This is one of the best real-RSS reduction targets.

### 4. Blank-screen canvas

This was the newest low-risk memory win.

- `us_blank_draw()` renders the "no live video" text into a full RGB canvas
- the canvas is only needed long enough to JPEG-encode the blank image
- after encoding, the canvas buffer is now released with `us_frame_release_data()`
- this avoids keeping a full-resolution RGB scratch buffer resident between redraws

Why this matters:

- the blank canvas is sized to the configured camera resolution
- at `864x480`, the RGB24 scratch buffer is about 1.2 MiB
- even when live video is running, the old code still held that memory

If blank redraws are needed later, the buffer is recreated on demand by `us_frametext_draw()`.

Implementation note:

- because the raw canvas can now be released, `us_frametext_draw()` must not skip
  a redraw just because the dimensions and text match
- the early-return guard now also requires the backing buffer to still exist

Latest measured result on `root@10.10.10.141`:

- commit `2be4fad` (`Cap USERPTR capture buffers at observed MJPEG size`)
- command: `/user-resource/ustreamer -r 864x480 -c HW -m MJPEG -b 1 -w 1 -f 8 -l --host=0.0.0.0 --allow-origin=*`
- sample method: wait 5 seconds, then read `/proc/<pid>/smaps_rollup`
- result: `Total: 3.74414 MB, Pss: 1.30664 MB, Rss: 2.43750 MB`

Operational path note:

- the live target root filesystem is squashfs read-only
- `/usr/bin/ustreamer` cannot be replaced in place
- the current service baseline on the box was restarted from `/usr/bin/ustreamer` and measured at `Total: 10.52246 MB, Pss: 4.67871 MB, Rss: 5.84375 MB`
- for the low-RAM binary, a file bind-mount works as the operational workaround:
  - `mount --bind /user-resource/ustreamer /usr/bin/ustreamer`
  - after that, the normal init script can still start `/usr/bin/ustreamer`
  - latest measured result with that bind mount and `-f 8`: `Total: 3.75098 MB, Pss: 1.30957 MB, Rss: 2.44141 MB`
- runtime conclusion on `root@10.10.10.141`:
  - `USERPTR` fails during `VIDIOC_QBUF` on this device and causes the capture loop to flap
  - `MMAP` works and produces live frames
  - the build should prefer `USERPTR` for the RAM win, but automatically retry with `MMAP` when `USERPTR` is not supported
  - `v4l2-ctl --all` on `/dev/video0` reports `Size Image: 1843200` for `864x480 MJPG` and `Read buffers: 0`
  - that means there is no supported `read()` capture path on this device, and shrinking `MMAP` buffers below the driver-reported size is not safe
  - the only way to make `USERPTR` work here would be to allocate at least driver `sizeimage`, which erases the RAM savings we were chasing
- YUYV mode test on the same camera:
  - `v4l2-ctl --try-fmt-video=width=864,height=480,pixelformat=YUYV` reports `Size Image: 829440`
  - running `/user-resource/ustreamer -r 864x480 -c HW -m YUYV -b 1 -w 1 -f 8 -l --io-method MMAP --host=0.0.0.0 --allow-origin=*`
  - measured `Total: 5.66309 MB, Pss: 2.25293 MB, Rss: 3.41016 MB`
  - snapshot payload size was `32499` bytes
  - this is a real RAM reduction versus MJPEG/MMAP, but it switches the encoder to CPU mode

## Automation / reproduction flow

This is the workflow that has been working end-to-end:

1. Make the local source change in `/home/paul/sandbox/ustreamer`.
2. Commit the source change on `paul/reduce-ram`.
3. Regenerate the Cosmos patch into the active Cosmos tree with:

```sh
COSMOS_DIR=/home/paul/carbon/cosmos ./gen-cosmos-patch.sh
```

4. Rebuild the CC2 USB image:

```sh
./rc.build.cc2.usb
```

5. Copy the rebuilt binary from the Yocto workdir to the target:

```sh
scp -p /home/paul/carbon/cosmos/build/tmp/work/cortexa7t2hf-neon-vfpv4-poky-linux-gnueabi/ustreamer/6.55/image/usr/bin/ustreamer \
  root@10.10.10.141:/user-resource/ustreamer
```

6. Launch the target binary, wait 5 seconds for it to settle, then sample memory and stop it:

```sh
ssh root@10.10.10.141 'nohup /user-resource/ustreamer -r 864x480 -c HW -m MJPEG -b 1 -w 1 -f 10 -l --host=0.0.0.0 --allow-origin=* >/tmp/ustreamer.log 2>&1 & echo $! >/tmp/ustreamer.pid'
ssh root@10.10.10.141 'sleep 5; grep -E '\''Pss:|Rss:'\'' /proc/$(pgrep -n ustreamer)/smaps | awk '\''{sum+=$2; label=$1} END{print label, sum/1024 " MB"}'\''; kill $(cat /tmp/ustreamer.pid)'
```

Preferred benchmark command:

```sh
ssh root@10.10.10.141 'nohup /user-resource/ustreamer -r 864x480 -c HW -m MJPEG -b 1 -w 1 -f 10 -l --host=0.0.0.0 --allow-origin=* >/tmp/ustreamer.log 2>&1 & echo $! >/tmp/ustreamer.pid; sleep 5; awk '\''/^Pss:|^Rss:/ {sum[$1]+=$2} END{printf "Total: %.5f MB, Pss: %.5f MB, Rss: %.5f MB\n", (sum["Pss:"] + sum["Rss:"]) / 1024, sum["Pss:"] / 1024, sum["Rss:"] / 1024}'\'' /proc/$(cat /tmp/ustreamer.pid)/smaps_rollup; kill $(cat /tmp/ustreamer.pid)'
```

Preferred runtime loop:

1. Start `ustreamer` in the background on the target.
2. Wait 5 seconds.
3. Sample RAM.
4. Leave the process running after the sample so the version has time to settle.
5. When a new binary needs to be copied, kill the running process first.
6. `scp` the new binary into `/user-resource/ustreamer`.
7. Restart the process and give it another settle window before the next measurement.

This is the command shape I want to use for a long-lived test run:

```sh
ssh root@10.10.10.141 'nohup /user-resource/ustreamer -r 864x480 -c HW -m MJPEG -b 1 -w 1 -f 10 -l --host=0.0.0.0 --allow-origin=* >/tmp/ustreamer.log 2>&1 & echo $! >/tmp/ustreamer.pid'
```

To refresh the binary:

```sh
ssh root@10.10.10.141 'kill $(cat /tmp/ustreamer.pid)'
scp -p /path/to/new/ustreamer root@10.10.10.141:/user-resource/ustreamer
ssh root@10.10.10.141 'nohup /user-resource/ustreamer -r 864x480 -c HW -m MJPEG -b 1 -w 1 -f 10 -l --host=0.0.0.0 --allow-origin=* >/tmp/ustreamer.log 2>&1 & echo $! >/tmp/ustreamer.pid'
```

Important note:

- The `grep -E 'Pss:|Rss:'` command also matches `SwapPss:` lines, so the output can print `SwapPss:` instead of `Pss:` or `Rss:`.
- The benchmark should be taken after a 5 second settle period, as requested by the user.
- `RAMUSE.md` should be kept in the repo, but excluded from the Yocto patch payload. The scratch generator now filters it out.
- For iterative testing, prefer restarting the *same* long-lived target process only when swapping binaries; otherwise leave it running after the snapshot.

## RAM log

Keep the history in `RAMUSE.md`. Current entries:

- `be84c43` -> `10.319 MB`
- `dc5c80d` -> `9.946 MB`
- `4ef9593` -> `9.946 MB`
- `64f7fd6` -> `SwapPss: 9.44434 MB`
- `2db118c` -> `Pss: 4.15527 MB / Rss: 5.29688 MB`

### 4. Blank frame storage

The blank path also keeps:

- a raw frametext buffer
- a JPEG-encoded blank frame

These are not likely the dominant cost during normal live streaming, but they exist:

- `src/ustreamer/blank.c`

### 5. Shared memory sinks

Shared memory sinks are only relevant if enabled.

In the user's command line, no JPEG/RAW/H264 sink flags are present, so these should not contribute.

If enabled, they allocate a full shared-memory region sized for the frame payload plus metadata:

- sink open/truncate/map in `src/libs/memsink.c`

For this user case, treat sinks as irrelevant unless the deployment adds them elsewhere.

## What is already minimized by the user's flags

The user has already done several good things:

- `-b 1`
  - one capture buffer instead of multiple
- `-w 1`
  - one encoder worker
- `-c HW -m MJPEG`
  - avoids CPU re-encoding from raw formats
- `-f 10`
  - limits encode/send cadence, though it does not directly eliminate all buffers

This means the remaining easy wins are limited.

## Best implementation candidates

### Candidate A: cap pthread stack size

Status:

- already implemented locally in `src/libs/threading.h`
- uncommitted

Pros:

- very low risk
- centralized
- no pipeline redesign
- can reduce apparent memory meaningfully if thread stacks dominate measurement

Cons:

- may help `VSZ` more than `RSS`
- may not deliver a 2x reduction in real resident memory

Notes:

- if this is kept, it should probably become configurable by macro or build flag
- 512 KiB is a reasonable first value, but validating against deepest call chains is still prudent

### Candidate B: shrink the HTTP JPEG ring from 4 to 2 or 1

Current code:

- `src/ustreamer/stream.c:107`

Pros:

- simple
- directly reduces number of retained expanded `us_frame_s` buffers
- likely real RSS reduction

Cons:

- ring depth probably exists to absorb producer/consumer timing mismatch
- going too low may increase drop/stall sensitivity under slow clients or event-loop jitter

Suggested implementation shape:

- start with capacity 2 instead of 4, or
- make it configurable behind a compile-time macro or CLI option

This is probably the lowest-effort path to a real heap reduction beyond thread stacks.

### Candidate C: remove one JPEG deep-copy stage

Most interesting targets:

- `_stream_expose_jpeg()` copy into ring
  - `src/ustreamer/stream.c:726-727`
- `_expose_frame()` copy into `exposed->frame`
  - `src/ustreamer/http/server.c:985`

Pros:

- best chance of reducing real resident memory in the live HTTP path

Cons:

- higher correctness risk than ring-size or stack-size tweaks
- ownership and lifetime become the hard problem
- current design uses copies to decouple capture/encode timing from HTTP client timing

Practical direction if someone implements this:

- do not try to point HTTP directly at the V4L2 MMAP buffer
- do not try to reuse the encoder worker buffer without a real ownership model
- a safer approach is to reduce the number of durable heap copies, not eliminate copying entirely

Example partial redesign:

- keep a tiny ring of owned immutable JPEG frames
- let HTTP expose reference a ring slot instead of deep-copying into `exposed->frame`
- ensure ref/lifetime rules prevent slot reuse while HTTP is still emitting from it

This is no longer "tiny tweak", but still much smaller than a rewrite.

### Candidate D: merge the releaser model into the main loop

Current design creates one releaser thread per capture buffer:

- `src/ustreamer/stream.c:176-186`

With `-b 1`, that is only one extra thread, so the payoff is small in the user's exact config.

Pros:

- could eliminate a thread and its stack

Cons:

- not worth much with `-b 1`
- likely more invasive than stack cap or ring reduction
- release timing is currently decoupled from worker completion via refcounts

This is not the first thing to do for the user's setup.

### Candidate E: force a single-process / no-thread model

Not recommended.

Reasons:

- the code is already one process
- select/libevent are already in place where they matter
- the remaining work would mostly be converting thread handoff/lifetime logic, not removing forks
- poor effort/reward for this goal

## Likely memory accounting breakdown in the user's case

This is inference, not measured data.

For `864x480` MJPEG at low FPS with `-b 1 -w 1 -c HW`:

- one capture MMAP buffer: size depends on driver and worst-case JPEG buffer sizing
- one encoder worker destination JPEG buffer: grows to peak seen frame size
- up to four ring-slot JPEG buffers: each may remain grown to prior peak size
- one HTTP exposed JPEG buffer
- several thread stacks / stack mappings
- libevent / HTTP bookkeeping

The relative winner depends on what the memory tool reports:

- if it is mostly `VSZ` or task-based totals:
  - thread stack cap may move the number a lot
- if it is mostly `RSS`:
  - reducing durable JPEG copies is the more promising path

## Suggested order of attack

### Lowest-risk first pass

1. Keep or refine the thread stack cap patch
2. Reduce HTTP JPEG ring capacity from 4 to 2
3. Measure again

This is the best "small changes first" plan.

### If more reduction is needed

4. Rework HTTP exposure so it does not make an extra deep copy into `exposed->frame`
5. Only after that, consider more structural thread/lifetime changes

## Things that are probably not worth time for this exact request

- replacing `select()` with another polling primitive
- hunting for `fork()` removal in the streaming path
- removing RAW/H264/DRM support paths for this config
- USERPTR experiments unless there is a driver-specific reason

## Build / verification notes

- Full `make` in this workspace failed for an environment issue:
  - missing `jpeglib.h`
- The touched thread helper compiled through a targeted object build:
  - `make -C src build/libs/logging.o`

So another agent should not assume full local builds are currently available without fixing dependencies.

## Current local patch summary

The current uncommitted patch in `src/libs/threading.h`:

- adds `US_THREAD_STACK_SIZE` default of `512 * 1024UL`
- wraps thread creation in `us_thread_create()`
- applies `pthread_attr_setstacksize()`

This patch was intentionally chosen because it is:

- centralized
- low risk
- the most plausible "small patch" memory win without redesigning the frame pipeline

## If implementing next step right now

If I were continuing from here, I would do this next:

1. Keep the current `threading.h` patch as one isolated change.
2. Leave the ring size alone for now; the live measurement shows the capture backend is the dominant resident mapping.
3. If the user wants more RAM reduction, prefer a capture-path change over more HTTP-copy work.
4. If you need a concrete next experiment, try USERPTR capture without pre-faulting the buffers and measure `/proc/<pid>/smaps_rollup`.

## Quick reference to code hotspots

- top-level threads
  - `src/ustreamer/main.c:105-110`
- capture uses `select()`
  - `src/libs/capture.c:504-528`
- default capture buffer count
  - `src/libs/capture.c:148`
- V4L2 MMAP buffer mapping
  - `src/libs/capture.c:921-976`
- stream helper thread creation
  - `src/ustreamer/stream.c:170-203`
- JPEG worker handoff
  - `src/ustreamer/stream.c:329-391`
- HTTP ring allocation
  - `src/ustreamer/stream.c:107`
- copy into HTTP ring
  - `src/ustreamer/stream.c:716-732`
- HTTP server exposed frame allocation
  - `src/ustreamer/http/server.c:112-116`
- initial blank copy into exposed frame
  - `src/ustreamer/http/server.c:198`
- copy into exposed live frame
  - `src/ustreamer/http/server.c:954-985`
- generic frame deep copy
  - `src/libs/frame.c:55-70`
- HW encoder still copies JPEG payload
  - `src/ustreamer/encoders/hw/encoder.c:67-72`
- HW input forces single encoder worker
  - `src/ustreamer/encoder.c:126-130`

## Latest experiment notes

- `--io-method=USERPTR` did not meaningfully change the measured total while the buffers were still pre-faulted.
- The next candidate was to stop touching the USERPTR buffer pages up front, so they can stay demand-paged.
- Debug sampling on the target showed actual MJPEG `bytesused` around 44-45 KB at 8 FPS.
- The current USERPTR cap is 256 KB to keep plenty of headroom while avoiding the 1.9 MB driver `sizeimage`.
