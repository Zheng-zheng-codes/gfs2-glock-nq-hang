# INFO: task hung in gfs2_glock_nq (v7.4-rc3)

Syzkaller-found hang on Linux v7.4-rc3, reproduced and verified by the
KernSight state-primed pipeline.

## Summary

A syz-executor task blocks for more than 143 seconds in the GFS2 glock
lookup/wait path during `umount`:

```
INFO: task syz-executor:32513 blocked for more than 143 seconds.
      Tainted: G            EL      7.4.0-rc3-kernsight-kasan #1
task:syz-executor    state:D ...
Call Trace:
 __schedule
  bit_wait
   __wait_on_bit
    out_of_line_wait_on_bit
     wait_on_bit
      gfs2_glock_wait+0xaa/0x2e0 fs/gfs2/glock.c:1239
       gfs2_glock_nq+0xd0b/0x17b0 fs/gfs2/glock.c:1530
        gfs2_glock_nq_init
         __gfs2_lookup+0x10f/0x2a0 fs/gfs2/inode.c:1006
          __lookup_slow
           lookup_slow
            path_lookupat
             filename_lookup
              user_path_at
               ksys_umount
                __x64_sys_umount
                 do_syscall_64
```

The task is in D state waiting for a GFS2 glock while unmounting a crafted
GFS2 filesystem. The held locks include `i_mutex_dir_key` and
`DCACHE_PAR_LOOKUP`; the glock grant does not make progress.

## Verification

- Pipeline attempt 1: budget exhausted (72 replay programs).
- Pipeline attempt 2: `confirmed=true`, `status=verified`,
  `verification_scope=snapshot_restore`
- Time: `2026-11-16T13:02:12Z`
- 182 replay programs, 0 priming programs
- Evidence: `reproducer/repro.outcomes.jsonl`, `reproducer/repro.stats`,
  `reproducer/repro.report`

External fresh-VM verification (rc3):

- Command: `syz-execprog -procs=8 -threaded -repeat=0 -executor ./syz-executor repro.kernsight.prog`
- Run 1: kernel hung-task report at 331.48 s after boot
  (`blocked for more than 143 seconds`)
- Run 2: kernel hung-task report at 290.82 s after boot
  (`blocked for more than 143 seconds`)
- Both reports show the same `gfs2_glock_wait` -> `gfs2_glock_nq` ->
  `__gfs2_lookup` -> `ksys_umount` stack.
- Full serial logs: `logs/external-verification/`

The hang is detected by the kernel's hung-task detector (`khungtaskd`), not
by the test harness. The outer 600/700 s timeouts only stop the
`-repeat=0` loop; the success signal is the kernel's
`INFO: task ... blocked for more than ... seconds` message and stack.

## How to reproduce

Environment: Debian trixie x86_64 guest (6 GiB RAM, 4 vCPU, KVM), kernel
built from v7.4-rc3 with CONFIG_KASAN=y, CONFIG_PROVE_LOCKING=y,
CONFIG_GFS2_FS=y, booted with `oops=panic` and `panic_on_warn=1`. See
`environment.txt` and `kernel.config`.

```sh
# in the guest, as root:
syz-execprog -procs=8 -threaded -repeat=0 \
    -executor ./syz-executor repro.kernsight.prog
```

The recipe has 183 entries and builds the glock state by repeated replay.
Wait for the kernel hung-task detector to report
`INFO: task ... blocked for more than 143 seconds` with the
`gfs2_glock_nq` stack.

## Files

- `description` — syzkaller title
- `environment.txt` — kernel/guest image, QEMU command line, syzkaller commit
- `kernel.config` — 7.4-rc3 kernel configuration
- `reproducer/repro.kernsight.prog` — state recipe (183 entries)
- `reproducer/repro.priming.prog` — priming part
- `reproducer/repro.prog` — triggering testcase
- `reproducer/repro.report`, `repro.stats`, `repro.outcomes.jsonl`,
  `repro.context.json` — pipeline reproduction evidence
- `logs/log-0.txt`, `logs/report-0.txt` — original rc3 syzkaller log/report
- `logs/machineInfo-0.txt`, `logs/title-stat.json` — guest/title statistics
- `logs/external-verification/` — full rc3 serial logs and run records for
  the two external reproductions

## Notes

- Crash title: `INFO: task hung in gfs2_glock_nq`
- Subsystem: GFS2 glock/umount
- The repro depends on the GFS2 mount/inode state built by the recipe.
