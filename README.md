# INFO: task hung in gfs2_glock_nq (fs/gfs2/glock.c:1518, v7.4-rc2)

Syzkaller-found hang bug on Linux v7.4-rc2, reproduced and verified by the
fuzzing pipeline's state-primed reproducer machinery on freshly replayed
VMs. This is the hang-form sibling of the `kernel BUG in gfs2_glock_nq`
crash reported separately (both fire from the same statfs slow path).

## Summary

A syz-executor task blocked for more than 140 seconds acquiring a glock
during a directory lookup:

```
INFO: task syz-executor:14887 blocked for more than 143 seconds.
      Tainted: G L 7.4.0-rc2-kernsight-kasan #1
task:syz-executor state:D stack:22792 pid:14887
Call Trace:
 schedule+0xec/0x2d0 kernel/sched/core.c:7481
 ...
 gfs2_glock_nq_init fs/gfs2/glock.h:253 [inline]
 __gfs2_lookup+0x10f/0x2a0 fs/gfs2/inode.c:1006
 __lookup_slow+0x15b/0x330 fs/namei.c:1925
 lookup_slow+0x59/0x80 fs/namei.c:1942
 walk_component fs/namei.c:2288 [inline]
 lookup_last fs/namei.c:2795 [inline]
```

The task waits in `gfs2_glock_nq()` (fs/gfs2/glock.c:1518) via
`gfs2_glock_nq_init()` during a lookup (`__gfs2_lookup()`,
fs/gfs2/inode.c:1006) on a gfs2 mount: the glock holder it needs is held
by another task (or the wait queue ordering deadlocks), so it sleeps in D
state past the 143-second hung-task threshold. The guest runs with
`hung_task_panic=0`, so this is a hang report rather than a panic.

## How to reproduce

Environment: Debian trixie x86_64 guest (6 GiB RAM, 4 vCPU, KVM), kernel
built from v7.4-rc2 with CONFIG_KASAN=y, CONFIG_PROVE_LOCKING=y,
CONFIG_GFS2_FS=y, booted with `oops=panic`, `panic_on_warn=1` and
`hung_task_panic=0`. See `environment.txt` for the exact QEMU command line
and `kernel.config` for the configuration.

The hang depends on glock-holder state accumulated by the preceding
programs, so the reproducer is an ordered program set:

```sh
# in the guest, as root:
syz-execprog -procs=8 -threaded -repeat=0 -executor ./syz-executor repro.kernsight.prog
```

`reproducer/repro.kernsight.prog` is the pipeline-verified reproducer
(state-reconstruction tail + triggering lookup, in execution order). In
the fuzzing pipeline it was reproduced by the state-primed machinery on
freshly replayed VMs (multiple verified sessions; the raw verification
material is preserved in the pipeline artifacts).

`reproducer/repro.prog` is the minimal triggering testcase extracted by
the pipeline; it depends on the state above and does not reproduce alone.
A C reproducer has not been included yet.

## Files

- `kernel.config` — kernel configuration used (also required to interpret addresses)
- `environment.txt` — kernel image info (sha256), QEMU command line, syzkaller commit, reproducer command
- `reproducer/repro.kernsight.prog` — pipeline-verified reproducer (ordered program set)
- `reproducer/repro.prog` — minimal triggering testcase (state-dependent; does not reproduce alone)
- `reproducer/repro.priming.prog` — state-priming program set (pipeline repro recipe)
- `logs/log-*.txt`, `logs/report-*.txt` — original syzkaller execution logs and crash reports from the fuzzing pipeline (24 hangs of this title)
- `logs/machineInfo-*.txt`, `logs/title-stat.json` — guest environment info and title statistics

## Notes for maintainers

- The blocked path is a lookup (`__gfs2_lookup()`) acquiring a glock via
  `gfs2_glock_nq_init()`; identifying the conflicting holder from the full
  console log (available in the pipeline artifacts on request) is the
  natural next step.
- The same statfs/lookup slow paths also produced the
  `kernel BUG in gfs2_glock_nq` recursive-holder crash (reported
  separately).
- We can provide the kernel image, additional crash instances or raw
  artifacts on request.
