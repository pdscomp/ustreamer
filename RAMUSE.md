# RAM Use Log

Living notes for uStreamer memory measurements on the CC2 image.

Measurement command:

```sh
grep -E 'Pss:|Rss:' /proc/$(pgrep -n ustreamer)/smaps | awk '{sum+=$2; label=$1} END{print label, sum/1024 " MB"}'
```

Notes:
- These numbers are from one active stream using:
  - `/user-resource/ustreamer -r 864x480 -c HW -m MJPEG -b 1 -w 1 -f 10 -l --host=0.0.0.0 --allow-origin=*`
- The values below are what I observed on `root@10.10.10.141`.
- The simple `grep -E 'Pss:|Rss:'` command also matches `SwapPss:` lines, so I am recording the output exactly as it printed.
- For the latest run, I used `/proc/<pid>/smaps_rollup` with exact `Pss:` and `Rss:` matching so the result is comparable.
- `Total` means `Pss + Rss` for the `smaps_rollup` entries, and the earlier grep-based logs were already total combined numbers.

| Git commit | Change | Total | Details |
| --- | --- | --- | --- |
| `2be4fad` | Cap USERPTR capture buffers at observed MJPEG size | `3.74414 MB` | `Pss: 1.30664 MB`, `Rss: 2.43750 MB` |
| `6941db5` | Release blank RGB canvas after encoding | `7.37598 MB` | `Pss: 3.11816 MB`, `Rss: 4.25781 MB` |
| `be84c43` | Cap internal thread stack size | `10.319 MB` | grep total |
| `dc5c80d` | Trim uStreamer memory overhead | `9.946 MB` | grep total |
| `4ef9593` | Reduce uStreamer frame backlog | `9.946 MB` | grep total |
| `64f7fd6` | Lower uStreamer thread stack cap | `9.44434 MB` | grep total; label printed as `SwapPss` |
| `2db118c` | Trim remaining frame overhead | `9.45215 MB` | `Pss: 4.15527 MB`, `Rss: 5.29688 MB` |
| `b1bd032` | Record USERPTR capture cap result | `3.75098 MB` | `/usr/bin/ustreamer` bind-mounted to `/user-resource/ustreamer`, `Pss: 1.30957 MB`, `Rss: 2.44141 MB` |
| `b1bd032` | YUYV @ 864x480 with MMAP | `5.66309 MB` | `Pss: 2.25293 MB`, `Rss: 3.41016 MB`, snapshot payload `32499` bytes |
