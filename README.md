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

A kernel-level CPU/IO optimization driver that reacts to screen state, charging status, and thermal state, with a scaffold in place for true-suspend detection.

**How it works:**

- **Screen state detection:** Polls `DPMS` first, falling back to backlight brightness, at an interval of `poll_interval_ms` (default 1000 ms), starting `boot_delay_ms` (default 40 s) after boot to let the display driver finish probing.
- **Screen OFF:**
  - Isolates the `system-background` and `background` cpusets to CPU 0.
  - Switches the I/O scheduler to `none` (the previously active scheduler is auto-detected and saved so it can be restored later).
  - Applies a CPU frequency bias: both the min and max frequency QoS are pinned to the same target, calculated as `min_freq + (max_freq - min_freq) * bias%`. Charging and non-charging use **different bias percentages** — `charging_freq_bias_percent` (default 20%) applies while charging, `doze_active_freq_bias_percent` (default 10%, i.e. closer to the floor / more aggressive throttling) applies otherwise.
  - Scans every available thermal zone (up to 15) and takes the **highest** reported temperature. If it's at or above `thermal_threshold_mc` (default 65 °C), a thermal hold is armed for when the screen turns back on.
- **Screen ON:**
  - Cpusets and I/O scheduler are restored immediately.
  - If a thermal hold was armed: the screen-off frequency bias is kept in place for `thermal_hold_ms` (default 3 s), then temperature is rechecked. Frequencies are only released once the max zone temp drops below `thermal_threshold_mc - thermal_hysteresis_mc` (default 60 °C); otherwise the hold is extended each poll cycle until it cools down.
  - If no thermal hold was armed, CPU frequencies are restored to stock immediately.
- **Suspend-awareness (scaffold):** A PM notifier tracks whether the device is in a true suspend (`PM_SUSPEND_PREPARE`/`PM_POST_SUSPEND`) vs. just screen-off-but-awake, but this state isn't wired into any frequency decision yet — it's reserved for a future policy.
- **Runtime tunable:** every threshold above (`poll_interval_ms`, `boot_delay_ms`, `thermal_threshold_mc`, `thermal_hysteresis_mc`, `thermal_hold_ms`, `charging_freq_bias_percent`, `doze_active_freq_bias_percent`) is a writable module parameter — no rebuild needed to tune behavior.

**Requirements:**

Screen state detection needs one of:
- DPMS: `/sys/class/drm/card0-DSI-1/dpms`
- Backlight: `/sys/class/backlight/panel0-backlight/brightness`

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

Based on `https://github.com/LoggingNewMemory/SuiKernel-Release`'s Tenebrion logic and SELinux rules.

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

## Other Features

- Full LTO (Link Time Optimization) builds
- Google Common Kernel LTS tracking

## Big Thanks

https://github.com/LoggingNewMemory/SuiKernel-Release — Tenebrion logic and SELinux rules

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
