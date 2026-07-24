# Changelog

All notable changes to this kernel will be documented in this file.

## [Zeta] - 2026-07-24

### Added
- **Scheduler:** Implemented asymmetric CPU capacity wakeup scan and aggressive load balancing for SCHED_IDLE CPUs.
- **Memory:** Added automatic memory compaction mechanism to reduce fragmentation.
- **Security/Exec:** Added node tampering blacklist function for `power@2.0` and `IOP`.
- **Core:** Implemented `rcu_work` and migrated `cgroup` and `fs/aio` to use it instead of explicit rcu and work items.
- **Security:** Implemented `thread_struct` whitelist for hardened usercopy on arm64.

### Changed (Scheduler & CPU)
- **Defconfig:** Switched default scheduling algorithm from WALT to PELT (`xiaomi/miatoll_defconfig`).
- **Defconfig:** Enabled `CONFIG_SCHED_AUTOGROUP` by default.
- **Task Placement:** Configured `top-app` tasks to prefer exclusive mid-cluster CPU placement.
- **Task Placement:** Refined scheduler logic to avoid placing tasks on little CPUs blindly due to sync wakeups.
- **Load Tracking:** Separated capacity margins for boosted tasks and updated scheduler to use actual CPU capacity to calculate boosted utilization.
- **CPUFreq:** Exposed `schedutil` limit tunables cluster-wise and replaced the global stats lock with atomics.
- **Thermals:** Optimized `step_wise` thermal mitigation evaluation, stable trend handling, and cleared mitigations upon reaching safe thresholds.

### Changed (Locking & Core Kernel)
- **Locking (rwsem):** Massive rewrite and optimization of read-write semaphores (`rwsem`). Implemented a new locking scheme, optimized for uncontended lock acquisition, enabled time-based optimistic spinning, and implemented lock starvation handoff.
- **Locking (Spinlocks):** Replaced arm64 ticket lock implementation with `qspinlock`.
- **Locking (Mutex):** Optimized `__mutex_trylock_fast()`.
- **Libraries (Bitmap):** Extensive micro-optimizations across `lib/bitmap.c` and `linux/bitmap.h`, replacing slow array operations and simplifying page buffer prints.
- **Libraries:** Inlined `memzero_explicit()` and optimized `rbtree` to avoid generating duplicate cached versions.
- **RCU:** Restricted expedited grace periods (GP) to allow only one to run concurrently with wakeups.

### Changed (GPU, Display & Memory)
- **Memory (ION):** Enabled ION pool auto-refill and set fill mark to 100MB in defconfig.
- **Memory (ION):** Optimized refill behavior (shrink lower order pools first, skip order 0, defer on low zone watermarks, and removed CPU binding of refill kthreads).
- **DRM/Display:** Sped up interrupt processing upon commit.
- **GPU (KGSL):** Prevented dynamic allocation of memory for temporary/small command buffers to reduce overhead.
- **GPU (KGSL):** Fixed sysfs to report the correct GPU frequency.

### Fixed
- **Scheduler:** Reverted buggy userspace affining threads and use-after-free issues in `is_sched_lib_based_app()`.
- **Scheduler:** Fixed `is_min_capacity_cpu()` implementation and reduced softirq conflicts with RT tasks.
- **Locking:** Fixed implicit declarations, error arithmetic on incomplete types, and removed unnecessary `BUG_ON()` calls in `qspinlock`.
- **Build/Compilation:** Fixed build breakages when `CONFIG_SCHED_WALT=n` and `PELT` is enabled, and resolved C99 extension compilation warnings.
- **Misc:** Cleaned up excessive hyphens in Kconfigs, removed unneeded includes, and fixed checkpatch errors across staging drivers.

---

## [Epsilon] - 2026-07-16

### Build Variants Added
- **Scheduling:** Shipped separate builds utilizing `uclamp` and `stune` (Schedtune) for user preference.
- **Root:** Shipped separate variants with KernelSU (KSU) inline and without KSU (Non-KSU).

### Added
- **CPU:** Added bi-cluster API to affine IRQs and kthreads to fast CPUs.
- **CPU:** Added cpumask for big and LITTLE CPU clusters.
- **Scheduler:** Added API to migrate the current process to a given cpumask.
- **Filesystem:** Added support in f2fs for reporting a fake kernel version to fsck.

### Changed (Performance & Scheduler)
- **Scheduler (Core):** Optimized `ttwu()` spinning on `p->on_cpu` and `try_to_wake_up()` for local wakeups.
- **Scheduler (Core):** Skipped rq lock in `try_to_wake_up()` when WALT is disabled.
- **Scheduler (Core):** Offloaded wakee task activation if the wakee is descheduling.
- **Scheduler:** Changed default `SCHED_RR` timeslice from 100 ms to 1 jiffy and replaced `SCHED_FIFO` with `SCHED_RR` for all users.
- **Scheduler:** Reverted Google's capacity margin hacks and retained important task migration logic in PELT.
- **SMP:** Optimized `send_call_function_single_ipi()`.
- **Crypto:** Used `__cacheline_aligned` for AES data.
- **Affinity:** Affined hwcomposer, DRM important kthreads, and focaltech touchscreen IRQs strictly to the big CPU cluster.
- **Affinity:** Affined unbound workqueues and RCU nocb kthreads to little CPUs by default.
- **Clock:** Set each CPU clock to its maximum upon waking up (`clk-cpu-osm`).

### Changed (GPU & Display)
- **KGSL / DRM:** Drastically reduced latency while processing non-blocking commits and atomic ioctls.
- **KGSL / GPU:** Avoided busy waiting for fenced GMU writes.
- **KGSL / DRM:** Increased worker thread priority and optimized worker affinity for di-cluster setups.
- **Display:** Skipped heavy autorefresh checks and prefill calculations on command mode panels.
- **GPU:** Fixed target frequency calculations for high refresh rates (`adreno_tz`) and forced GPU idle timeout to 58 ms.

### Changed (Memory & Power Management)
- **Memory (Core):** Rewrote locking/atomics and `asm-generic/bitops/atomic.h` using `atomic_*()` APIs.
- **Memory (Core):** Forced PID map reads to occur on the little CPU cluster for efficiency.
- **LMK:** Increased default minimum memory to reclaim in `simple_lmk` and moved its kthread to the big CPU cluster.
- **Memory:** Disabled SLUB per-CPU partial caches (`miatoll_defconfig`).
- **Power:** Killed userspace CPU boosting entirely and changed input boost duration to 58ms.
- **Power:** Forced the LITTLE cluster to strictly utilize the default governor.
- **Idle:** Marked CPUs idle as late as possible to avoid unneeded IPIs and allowed `IPI_WAKEUP` outside of ACPI parking protocol.

### Fixed
- **Scheduler Races:** Fixed data-race in wakeup, race against `ptrace_freeze_trace()`, loadavg accounting race, and `ttwu()` race.
- **Scheduler:** Fixed `rq->nr_iowait` ordering, preempt warning in `ttwu`, and rq clock warning in `sched_migrate_to_cpumask_end()`.
- **Compilation:** Fixed smb5-lib macro compilation error in power supply.
- **Compilation:** Fixed undeclared identifier and missing member name errors in `kernel/irq`.
- **Compilation:** Fixed missing symbol on `ld.lld` for unaffine, reaffine, and set_perf irqs in `irq/manage`.
- **Compilation:** Fixed unterminated conditional directive error in `include/linux/cpuidle` and Clang `-Wbraced-scalar-init` warnings in `PM_QOS`.
- **Logs:** Silenced `ibat_ua` log spam in `power/supply/qcom`.
- **Logs:** Demoted verbose non-critical log spam across drivers to keep dmesg clean.
- **Misc:** Fixed load balancing for big tasks in `sched: fair`.
- **Misc:** Ordered thread flag tests with respect to flag mutations in `thread_info`.
- **Note:** Added and subsequently reverted releasing the spinlock on `zap_pte_range` in `mm`.
