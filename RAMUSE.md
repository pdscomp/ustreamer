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

| Git commit | Change | RAM |
| --- | --- | --- |
| `be84c43` | Cap internal thread stack size | `10.319 MB` |
| `dc5c80d` | Trim uStreamer memory overhead | `9.946 MB` |
| `4ef9593` | Reduce uStreamer frame backlog | `9.946 MB` |
| `64f7fd6` | Lower uStreamer thread stack cap | `SwapPss: 9.44434 MB` |
