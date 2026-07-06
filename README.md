**IMPORTANT DISCLAIMER**

> [!CAUTION]
> This software is provided for testing and educational purposes only. Use at your own risk. The developers are not responsible for any damage, data loss, or issues that may occur. Please ensure you have proper backups before installation.

# MomenToMoiX GKI Kernel

**By:** @koneko_dev [t.me/Koneko_dev](https://t.me/Koneko_dev)

> Releases are named after Malice Mizer songs and appended to the kernel version string (e.g. `5.10.198-android14-MomenToMoiX-Gardenia`). Check the [Releases](../../releases) page for available builds, or run `uname -r` after flashing to confirm the exact build you're running.

# Features
- [KernelSU-Next](#kernelsu-next)
- [SUSFS](#susfs)
- [MomenToMoiX Driver](#momentomoix-driver)
- [Governor Reflex](#governor-reflex)
- [Governor Vorpal](#governor-vorpal)
- [Memory Management](#memory-management)
- [Power Management](#power-management)
- [Scheduler & I/O](#scheduler--io)

## [KernelSU-Next](https://github.com/pershoot/KernelSU-Next)

A kernel-based root solution for Android devices.

> [!WARNING]
> This release uses the [pershoot/KernelSU-Next](https://github.com/pershoot/KernelSU-Next) fork. The fork maintainer has said it is not ready for production use, so treat it as use at your own risk.

## [SUSFS](https://gitlab.com/simonpunk/susfs4ksu)

A KSU addon for hiding root using kernel patches and a userspace module.

## MomenToMoiX Driver

A kernel-level CPU/IO optimization driver that reacts to screen state, charging status, and thermal state, with suspend-awareness and hotplug-safe QoS handling.

**How it works:**

- **Screen state detection:** Polls `DPMS` first, falling back to backlight brightness, at an interval of `poll_interval_ms` (default 1000 ms), starting `boot_delay_ms` (default 40 s) after boot to let the display driver finish probing. Both the DPMS and backlight node are resolved from a **candidate list** at boot rather than a single hardcoded path, so the same build works across devices with different sysfs layouts without a rebuild — including `leds/lcd-backlight` panels alongside the more common `backlight` class.
- **Screen OFF:**
  - Isolates the `system-background` and `background` cpusets to CPU 0.
  - Switches the I/O scheduler to `none` (the previously active scheduler is auto-detected and saved so it can be restored later).
  - Applies a CPU frequency bias: both the min and max frequency QoS are pinned to the same target, calculated as `min_freq + (max_freq - min_freq) * bias%`. Charging and non-charging use **different bias percentages** — `charging_freq_bias_percent` (default 20%) applies while charging, `doze_active_freq_bias_percent` (default 10%, i.e. closer to the floor / more aggressive throttling) applies otherwise.
  - Scans every available thermal zone (up to 15) and takes the **highest** reported temperature. If it's at or above `thermal_threshold_mc` (default 65 °C), a thermal hold is armed for when the screen turns back on.
- **Screen ON:**
  - Cpusets and I/O scheduler are restored immediately.
  - If a thermal hold was armed, the device's current temperature is rechecked **immediately** on wake — if it has already cooled below `thermal_threshold_mc - thermal_hysteresis_mc` (default 60 °C) while the screen was off, frequencies are released right away instead of waiting out a stale hold. Otherwise the screen-off bias is held for `thermal_hold_ms` (default 3 s) and rechecked each poll cycle until it cools down.
  - If no thermal hold is active, CPU frequencies are restored to stock and a **Wake Boost** is triggered (see below).
- **Wake Boost:** on every screen-on that isn't blocked by a thermal hold, a configurable frequency floor (`wake_boost_percent`, default 100%) is held for `wake_boost_ms` (default 1200 ms) via a dedicated workqueue, so the unlock animation and first app launch feel snappier before control hands back to the stock governor. If the screen turns off again before the window expires, the boost is cancelled cleanly instead of racing with the screen-off bias.
- **Suspend-awareness:** a PM notifier tracks true suspend (`PM_SUSPEND_PREPARE`/`PM_POST_SUSPEND`) versus just screen-off-but-awake. During an actual suspend cycle the watcher now parks itself instead of touching cpuset/I-O-scheduler/cpufreq sysfs nodes mid-transition, and forces a full display-state resync right after resume so it never acts on stale state.
- **Hotplug-safe QoS:** frequency QoS requests are kept in sync with live cpufreq policies via a policy notifier, so CPUs/clusters that come back online after being hotplugged off are picked back up automatically instead of being silently excluded from throttling until the next reboot.
- **Runtime tunable:** every threshold above (`poll_interval_ms`, `boot_delay_ms`, `thermal_threshold_mc`, `thermal_hysteresis_mc`, `thermal_hold_ms`, `charging_freq_bias_percent`, `doze_active_freq_bias_percent`, `wake_boost_percent`, `wake_boost_ms`) is a writable module parameter — no rebuild needed to tune behavior.

**Requirements:**

Screen state detection needs one of:
- DPMS: `/sys/class/drm/card0-DSI-1/dpms` (additional DPMS candidates can be added to the driver's candidate list)
- Backlight, tried in order:
  - `/sys/class/backlight/panel0-backlight/brightness`
  - `/sys/class/leds/lcd-backlight/brightness`
  - `/sys/class/backlight/panel0/brightness`

Charging-aware behavior additionally looks for one of:
- `/sys/class/power_supply/battery/status`
- `/sys/class/power_supply/bms/status`
- `/sys/class/power_supply/BAT0/status`

Thermal-aware behavior reads `/sys/class/thermal/thermal_zone0/temp` through `thermal_zone14/temp` and uses whichever zones are actually present.

If a node isn't found on your device, the corresponding feature is disabled/skipped automatically and logged — the rest of the driver keeps working normally.

**Verify it's running:**

```bash
su -c 'dmesg -w | grep momx'
```
after the device finishes booting.

**Tune it live:**

```bash
su -c 'ls /sys/module/momx/parameters/'
su -c 'echo 60000 > /sys/module/momx/parameters/thermal_threshold_mc'
```

## Governor Reflex

A schedutil-based governor extension that blends real idle-time CPU busy% (measured from kcpustat counters) with PELT utilization to get a faster-reacting "hispeed floor" without abandoning PELT's proportional scaling.

Frequency scaling itself is identical to stock schedutil, including the 1.25× DVFS headroom — only the hispeed blend differs:

- On each observation window, actual CPU busy% is measured from kcpustat idle-time accounting (not PELT), giving an immediate, non-decayed read of current load.
- This reading is blended with PELT util using exponential decay tied to PELT's own 32 ms half-life:

  ```
  blended = pelt + (hispeed - pelt) >> half_lives
  ```

- Every 32 ms, the hispeed contribution halves while PELT fills in the same gap, so total coverage stays at ~100% throughout the transition.
- After roughly 320 ms, the hispeed contribution is negligible and PELT-based proportional scaling has taken full control.

The effect is a governor that reacts instantly to a real load spike (via the idle-time-based hispeed reading) while smoothly handing off to PELT's steady-state behavior within a third of a second — avoiding both the lag of pure PELT and the overshoot of a naive instant-boost.

Reflex Governor by Masahito Suzuki

## Governor Vorpal

**Vorpal v2.0 — Perfect Gaming & Thermal Edition**, a schedutil-derived governor built around switching cleanly between sustained gaming load and everyday power-efficient use, with per-cluster tuning on tri-cluster (Little/Big/Prime) SoCs.

- **Dual-Profile Operating Modes** — a Gaming profile that locks into a high-frequency band for consistent frame timing, and a Daily profile tuned for power efficiency.
- **Tri-Cluster Topology Awareness** — Little, Big, and Prime clusters are tuned independently rather than sharing one global policy.
- **Directional EMA Util Smoothing** — an exponential moving average filter that rises fast on load spikes but decays slowly, avoiding frequency "yoyo-ing" between samples.
- **Dynamic Capacity Headroom** — OPP headroom above the measured load scales with how loaded the CPU already is, rather than a fixed margin.
- **Proactive Thermal Step Controller** — frequency caps ramp down in small ~2% steps and back up in ~1% steps, smoothing thermal response instead of hard-clamping.
- **Thermal Zone Integration** — reads hardware thermal sensors with a userspace-fallback path if a zone is unavailable.
- **Frame Pacing & Miss Recovery** — detects overruns against a 120fps budget and applies a bounded floor boost to recover pacing.
- **Global Frame Boost** — a dropped-frame event synchronizes a boost across all clusters, not just the one that missed.
- **Touch Input Responsiveness Boost** — lifts the frequency floor for a 220 ms window after touch input for immediate UI responsiveness.
- **UI Ramp-Assist / Render Burst** — detects sharp utilization ramps typical of animations/scroll bursts and responds ahead of the normal control loop.
- **Adaptive Floor (Idle/Busy)** — Prime and Little clusters switch between idle and busy frequency floors dynamically.
- **Directional Rate Limiting** — separate up/down rate-limit gates per cluster instead of one shared rate limit.
- **IOWait Performance Boost** — retains schedutil's legacy IOWait boost handling.
- **Deadline Bandwidth Awareness** — SCHED_DEADLINE task bandwidth can bypass normal frequency selection when needed.
- **Jank Telemetry & Statistics** — tracks and reports frame/jank ratios for tuning and debugging.
- **Deferred IRQ-Work Frequency Commit** — an async path for platforms without fast-switch support.
- **Global Policy State Reset** — cleanly resets all boost/state on leaving the Gaming profile.
- **GKI 5.10 Util Interface** — uses `rfx_get_util_gki510` / `rfx_dl_bw_exceeded_gki510` for utilization and deadline-bandwidth queries on the GKI 5.10 ABI.
- **Scheduler Coupling** — exposes `sched_gaming_active` so BORE/CFS scheduler biases can react to the active gaming profile.

**Usage:**

Set the governor on the cluster(s) you want, then flip the gaming switch on the **Prime cluster only** — that's the cluster the governor auto-detects as highest-capacity, and it's the only one exposing the gaming/thermal/frame attributes.

```bash
# set governor on all clusters
for p in /sys/devices/system/cpu/cpufreq/policy*/; do
  echo vorpal > "${p}scaling_governor"
done

# find which policy is Prime — it's the one with the full attribute set
for p in /sys/devices/system/cpu/cpufreq/policy*/vorpal/; do
  ls "$p" | grep -q gaming_mode && echo "Prime cluster: $p"
done

# toggle gaming mode (replace policyN with the Prime cluster found above)
su -c 'echo 1 > /sys/devices/system/cpu/cpufreq/policyN/vorpal/gaming_mode'   # gaming ON
su -c 'echo 0 > /sys/devices/system/cpu/cpufreq/policyN/vorpal/gaming_mode'   # back to Daily profile

Vorpal Governor by Templar Dev (Steambot12).

## Memory Management

- MGLRU (Multi-Generational LRU)
- ZRAM with ZSTD compression support
- ZSMALLOC
- KSM (Kernel Samepage Merging)
- Compaction & Migration
- Transparent Huge Pages (THP)

## Power Management

- TEO Idle Governor
- WQ_POWER_EFFICIENT
- SCHED_MC
- NO_HZ_IDLE
- Strict 100 max wakelocks limit with automated GC
- MomenToMoiX charging-aware and thermal-aware throttling (see above)

## Scheduler & I/O

- Full Preemption (CONFIG_PREEMPT)
- BFQ I/O Scheduler (with Group IOSCHED)
- MQ-Deadline
- TCP FastOpen
- Governor Reflex schedutil/PELT hispeed blend (see above)
- Governor Vorpal gaming/thermal profile switching (see above)

## Other Features

- Full LTO (Link Time Optimization) builds
- Google Common Kernel LTS tracking

## Big Thanks

https://github.com/LoggingNewMemory — Tenebrion logic (Kanagawa Yamada) and SELinux rules

Masahito Suzuki — Reflex Governor

Templar Dev (Steambot12) — Vorpal Governor

## Recommended Tools

[Kernel Flasher](https://github.com/fatalcoder524/KernelFlasher)
- Recommended flashing utility

[PixelFlasher by badabing2005](https://github.com/badabing2005/PixelFlasher)
- Pixel phone flashing GUI utility with features.

## Installation Instructions

### Prerequisites
- Unlocked bootloader.
- Backup your current boot image.
- Have root access using Magisk / KernelSU / Apatch (Any forks).

### Via Kernel Flasher
Download the correct AnyKernel3 ZIP for your device.
If you previously used another root method, clean it up first:
a. Magisk: perform a complete uninstall after flashing the AnyKernel3 ZIP.
b. KSU LKM (boot/init_boot/vendor_boot‑patched): Flash back the stock boot/init_boot/vendor_boot depending on what you patched.
c. KSU GKI: if you are 100% sure you already flashed stock init_boot/boot/vendor_boot, no action is needed; otherwise, follow the same steps as KSU LKM.
d. APatch: remove /data/adb contents to avoid leftover root conflicts after flashing the AnyKernel3 ZIP.
Flash the ZIP to the active slot using Kernel Flasher.
Install the KernelSU‑Next Manager APK, same version as mentioned in the release notes.
Open the KernelSU‑Next app.
Reboot the device if you performed any cleanup in step 2

---

MomenToMoiX Kernel is based on and developed from [WildKernels/GKI_KernelSU_SUSFS](https://github.com/WildKernels/GKI_KernelSU_SUSFS)
