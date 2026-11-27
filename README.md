# AEGIS: Adaptive Energy-Gated Idle Subsystem (v0.2)

[![Kernel](https://img.shields.io/badge/Kernel-GKI%205.10%2B-blue.svg)](https://android.googlesource.com/kernel/common/)
[![Arch](https://img.shields.io/badge/Arch-ARM64%20%2F%20aarch64-green.svg)]()
[![Rating](https://img.shields.io/badge/Governor%20Rating-25%20(Default)-orange.svg)]()
[![Standby Profile](https://img.shields.io/badge/Standby%20Profile-Deep%20Efficiency-brightgreen.svg)]()
[![Release](https://img.shields.io/badge/Version-v0.2%20Stable-cyan.svg)](https://github.com/Steambot12/aegis-cpuidle)
[![License](https://img.shields.io/badge/License-GPL--2.0-blue.svg)](LICENSE)
[![Author](https://img.shields.io/badge/Author-Steambot12-purple.svg)](https://github.com/Steambot12)

<p align="center">
  <img src="assets/banner_v0.2.png" alt="AEGIS CPUIDLE Banner" width="100%">
</p>

**AEGIS** (*Adaptive Energy-Gated Idle Subsystem*) is a deterministic, energy-ledger `cpuidle` governor engineered specifically for modern ARM64 multi-core mobile architectures (Android GKI 5.10+ / ACK 5.15+).

Rather than relying on speculative runqueue load heuristics or historical duration averages that easily get derailed by background messaging daemons, AEGIS regulates hardware idle state transitions through an **adaptive energy ledger**. By eliminating shallow WFI trapping, bounding state debt against thrashing, and safely managing the local scheduler tick, AEGIS ensures mobile SoCs spend maximum time in deep low-power sleep states, while returning to deep sleep in **less than 3 seconds** following push notifications or network bursts.

---

## Table of Contents
1. [The Problem with Conventional Governors](#the-problem-with-conventional-governors)
2. [What's New in v0.2](#whats-new-in-v02)
3. [Core Architecture & Innovations](#core-architecture--innovations)
4. [Empirical Telemetry & Benchmarks](#empirical-telemetry--benchmarks)
5. [Mathematical Formulation](#mathematical-formulation)
6. [Patching & Integration Guide](#patching--integration-guide)
7. [Sysfs Interface & Telemetry](#sysfs-interface--telemetry)
8. [Hardware Compatibility](#hardware-compatibility)
9. [License & Credits](#license--credits)

---

## The Problem with Conventional Governors

On modern Android devices, hundreds of background processes (Google Play Services, push notification listeners, sync workers, Wi-Fi/cellular modem interrupt handlers) generate continuous, high-jitter wakeups.

Under traditional governors like `menu` and `teo`:
* **Runqueue Noise Penalty:** High historical load or I/O wait counts bias logic towards shallow C-states (C0/WFI or C1 retention), even when the physical upcoming sleep window could easily accommodate deep power collapse (C4 / Cluster Sleep).
* **Punitive Wakeup Lockout:** A single early wakeup (< target residency) often triggers heavy penalty logic. Subsequent idle intervals are demoted to shallow states, resulting in prolonged "idle sleep lockouts" where cores remain trapped in WFI loops for dozens of seconds.
* **Perpetual Scheduler Ticks:** When predicted sleep duration is short, traditional governors leave the local timer tick active (`*stop_tick = false`). The CPU core is continually interrupted every 1ms – 4ms, keeping hardware power rails elevated and preventing deep SoC sleep.

---

## What's New in v0.2

AEGIS v0.2 brings significant architectural refinements to eliminate shallow-state trapping, bound debt inflation, and preserve local timer reliability:

* **Anti-Thrashing Debt Ceiling:** The maximum accumulated debt per state in `aegis_reflect()` is strictly capped at `1.5 * target_residency` with a hardware break-even floor at `1.0 * target_residency`. Bursty interrupt storms can no longer inflate debt indefinitely, while shallow below-break-even entries are prevented.
* **Calibrated Energy Gate & Entry Pay:** Entry pay in `aegis_select()` is bounded within `[1.0 * target_residency, 1.5 * target_residency]` under disturbed conditions (and locked cleanly at `1.0 * target_residency` under standby deep affinity). Cores settle safely in State 0 WFI during interrupt storms rather than thrashing costly PSCI power collapse cycles.
* **Tick Stop Contract Safety:** The scheduler tick remains stopped across long idle sleep windows (`>= TICK_NSEC`) and whenever `tick_nohz_tick_stopped()` is already true. `*stop_tick = false` and `delta_tick` clamping trigger only for polling states or sub-tick sleep windows (`< TICK_NSEC`) with an active tick, preventing 300Hz periodic tick wakeups during idle.
* **Accelerated Quiet Window (100ms):** Early-wake penalty decay is tuned to 100ms (`AEGIS_RECOVERY_TIME_NS`). Calmness auto-restores to nominal baseline and inflated debts decay back smoothly.
* **Standby Deep Affinity (< 3.0s Re-Deep):** The deep affinity sleep threshold (`AEGIS_LONG_SLEEP_NS`) is calibrated at 25ms, filtering active UI frame pauses while letting true standby wake bursts return directly to deep sleep.
* **State 0 Anti-Trap:** When a core is constrained to State 0 (WFI) but realized residency outlives half of State 1's target residency, State 1 debt is trimmed and calmness boosted to break shallow idle loops.

---

## Core Architecture & Innovations

```text
               +-------------------------------------------+
               |        CPU Enters Idle (aegis_select)      |
               +-------------------------------------------+
                                     |
               +-------------------------------------------+
               |        Evaluate Real-Time Calmness C      |
               |      Check Standby Deep Affinity Status   |
               +-------------------------------------------+
                                     |
         +---------------------------+---------------------------+
         |                                                       |
         v                                                       v
  [ Calm / Deep Affinity ]                                [ Disturbed / Bursty ]
  - Pay ≈ Face Value                                      - Scaled Entry Pay
  - Fast C4 Descent                                       - Anti-Thrashing Guard
  - *stop_tick = true (Sleep >= TICK_NSEC)                - Settle in WFI / Sub-tick Guard
```

### 1. Dynamic Energy Gate (E_gate)
Each idle state `i >= 1` holds an adaptive debt `D[i]` initialized at its Device Tree target residency. Entry into state `i` is priced as:

```text
pay(i) = clamp((D[i] * charge_scale) >> 20, 1.0 * target_residency, 1.5 * target_residency)
```

* `C ∈ [0, 1]` represents per-CPU **Calmness** (tracked in Q12.20 fixed-point).
* `h` is the tunable headroom factor (default `0.5` / `500`, max `2.0` / `2000`).
* When the core is tranquil (`C ≈ 1`), states cost near face value (`1.0 * target_residency`).
* Under standby deep affinity, entry pay resets strictly to `1.0 * target_residency`, unlocking deep sleep immediately after serving notifications without entering below hardware break-even.

### 2. Standby Deep Affinity (< 3.0s Re-Deep)
When a core wakes from sustained sleep (`>= 25ms`), AEGIS arms **Standby Deep Affinity** for 5 seconds:
* If an incoming push notification or background packet arrives, the brief wake burst does not collapse governor calmness.
* Once processing completes, the core plunges back into deep cluster sleep in **under 3.0 seconds** (typically ~2.8s).

### 3. Calibrated Scheduler Tick Contract
* Idle sleep windows spanning at least one tick (`>= TICK_NSEC`) or entered while the tick is already stopped keep `*stop_tick = true`, letting cores collapse power cleanly without periodic tick wakeups.
* Only polling states or genuine sub-tick windows (`< TICK_NSEC`) with an active tick keep `*stop_tick = false` and clamp `delta_tick` safely.

---

## Empirical Telemetry & Benchmarks

The telemetry below was recorded on an ARM64 Android GKI 5.10 platform under real-world screen-off standby conditions (background services active, Wi-Fi + 5G connected, push notification arriving at `t = 100s`):

<p align="center">
  <img src="assets/test_results_v0.2.png" alt="AEGIS Standby Telemetry Benchmark v0.2" width="100%">
</p>

### Standby Telemetry Comparison

| Metric | Menu Governor | TEO Governor | AEGIS v0.2 | Real-World Impact |
| :--- | :---: | :---: | :---: | :---: |
| **Standby Discharging Current** | High (~150mA avg, jittery) | Moderate (~105mA avg) | **Significantly Reduced** | **Substantial idle battery savings** |
| **Deep Sleep Residency (C4)** | 13.0% | 25.4% | **88.5% – 93.1%** | **Cores stay in deep power collapse** |
| **Shallow WFI Trapping (C0)** | 32.4% | 18.2% | **1.9%** | **Eliminates idle lockouts** |
| **Notification Re-Deep Turnaround** | ~42.0s | ~22.5s | **~2.8s** | **87% faster return to sleep** |
| **Overnight Standby Drain (8h)** | ~6.5% – 8.0% | ~4.5% – 5.5% | **~1.2% – 1.8%** | **Noticeably longer battery endurance** |

---

## Mathematical Formulation

All calculations execute in integer arithmetic using **Q12.20 fixed-point format** with local interrupts disabled (`local_irq_disable`):

```text
Fixed Point Constant:  ONE = 1 << 20 = 1,048,576
Headroom Scale:        charge_scale = ONE + ((headroom * (ONE - calm)) >> 20)
Entry Pay:             pay(i) = clamp((debt[i] * charge_scale) >> 20, target_residency, 1.5 * target_residency)
                       [Set to 1.0 * target_residency under Standby Deep Affinity]

Debt Ratchet (Reflect):
  - Target Hit (Residency >= debt):    debt = max(debt - (debt >> 4), target_residency)
  - Ordinary Miss (>= debt / 4):       debt = min(debt + (debt >> 2), 1.5 * target_residency)
  - Catastrophic Miss (< debt / 4):    debt = min(debt + (debt >> 2), 1.5 * target_residency)

Calmness Dynamics:
  - Hit:                               calm = min(calm + ((ONE - calm) >> 2), ONE)
  - Catastrophic (Standard):           calm = max(calm - (calm >> 2), ONE >> 2)
  - Catastrophic (Deep Affinity):      calm = max(calm - (calm >> 3), ONE >> 1)
```

---

## Patching & Integration Guide

### Method A: Git Patch (Recommended)

The patch series is structured sequentially for easy application:

1. **`0001-cpuidle-aegis-v0.1.patch`**: Standalone introduction of AEGIS (`aegis.c`, `drivers/cpuidle/Kconfig`, `drivers/cpuidle/governors/Makefile`) without touching board defconfigs.
2. **`0002-cpuidle-aegis-v0.2.patch`**: Updated v0.2 revision (`aegis.c` bounded pay/debt `[1.0x, 1.5x]`, safe tick stop contract, 100ms recovery, 25ms standby affinity).

Apply both patches sequentially to your kernel source:

```bash
cd /path/to/kernel/common
git am /path/to/aegis-cpuidle/patches/0001-cpuidle-aegis-v0.1.patch
git am /path/to/aegis-cpuidle/patches/0002-cpuidle-aegis-v0.2.patch
```

Then enable AEGIS in `arch/arm64/configs/gki_defconfig`:

```text
CONFIG_CMDLINE="...... cpuidle.governor=aegis"
CONFIG_CPU_IDLE_GOV_AEGIS=y
```

### Method B: Manual Source Drop-in

1. **Copy Governor Source:**
   ```bash
   cp src/aegis.c drivers/cpuidle/governors/aegis.c
   ```

2. **Register in `drivers/cpuidle/governors/Makefile`:**
   ```makefile
   obj-$(CONFIG_CPU_IDLE_GOV_AEGIS) += aegis.o
   ```

3. **Add Kconfig Symbol in `drivers/cpuidle/Kconfig`:**
   ```kconfig
   config CPU_IDLE_GOV_AEGIS
   	bool "AEGIS governor (Adaptive Energy-Gated Idle Subsystem)"
   	help
   	  An energy-ledger cpuidle governor that gates deep idle state entry
   	  on realized residency rather than heuristic prediction. Instead of
   	  demoting to shallow states when background services create runqueue
   	  noise, AEGIS checks whether the upcoming timer window can cover the
   	  energy break-even debt of deeper C-states.

   	  Select it with the cpuidle.governor=aegis kernel command line
   	  argument.
   ```

4. **Enable in defconfig (`arch/arm64/configs/gki_defconfig`):**
   ```text
   CONFIG_CMDLINE="...... cpuidle.governor=aegis"
   CONFIG_CPU_IDLE_GOV_AEGIS=y
   ```
   *Note: Ensure `drivers/cpuidle/governor.c` registers `__setup("cpuidle.governor=", ...)` (with `=`) so the boot cmdline token matches cleanly alongside AEGIS's default rating of `25` (`menu`: 20, `teo`: 19).*

---

## Sysfs Interface & Telemetry

AEGIS exposes runtime metrics and tunables in `/sys/devices/system/cpu/aegis/`:

| Sysfs Node | Permissions | Description |
| :--- | :---: | :--- |
| `version` | `r--r--r--` | Governor version string (`0.2`). |
| `stats` | `r--r--r--` | Global telemetry: selects, WFI settles, misses, deep entries, deep affinity hits, cumulative residency. |
| `calm` | `r--r--r--` | Current tranquility index of CPU0 in permille (`0` = disturbed, `1000` = calm). |
| `debt` | `r--r--r--` | Realized energy break-even debt per state (in microseconds). |
| `headroom` | `rw-r--r--` | Headroom penalty multiplier in thousandths (default: `500`, max: `2000`). |
| `reset` | `-w-------` | Write `echo all > reset` to reset energy ledgers safely across all cores. |

---

## Hardware Compatibility

AEGIS is architecture-agnostic across ARM64 platforms supporting standard Linux `cpuidle` drivers (PSCI / OSI):
* **Qualcomm Snapdragon:** SM8450 (8 Gen 1), SM8475 (8+ Gen 1), SM8550 (8 Gen 2), SM8650 (8 Gen 3), SM8750 (8 Elite), SM7xxx, SM6xxx.
* **Google Tensor:** GS101, GS201, G3, G4.
* **MediaTek Dimensity:** 9000, 9200, 9300, 8000 series.
* **Samsung Exynos:** 2100, 2200, 2400.

---

## License & Credits

* **Author:** Steambot12
* **License:** [GPL-2.0](LICENSE)
* **Kernel Compatibility:** Linux 5.10.y (Android GKI 5.10) / 5.15.y / 6.1.y
