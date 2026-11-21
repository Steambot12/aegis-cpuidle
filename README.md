# AEGIS: Adaptive Energy-Gated Idle Subsystem (v0.1)

[![Kernel](https://img.shields.io/badge/Kernel-GKI%205.10%2B-blue.svg)](https://android.googlesource.com/kernel/common/)
[![Arch](https://img.shields.io/badge/Arch-ARM64%20%2F%20aarch64-green.svg)]()
[![Rating](https://img.shields.io/badge/Governor%20Rating-25%20(Default)-orange.svg)]()
[![Standby Discharge](https://img.shields.io/badge/Standby%20Current-10mA%20--%2050mA-brightgreen.svg)]()
[![License](https://img.shields.io/badge/License-GPL--2.0-blue.svg)](LICENSE)
[![Author](https://img.shields.io/badge/Author-Steambot12-purple.svg)](https://github.com/Steambot12)

<p align="center">
  <img src="assets/banner.png" alt="AEGIS CPUIDLE Banner" width="100%">
</p>

**AEGIS** (*Adaptive Energy-Gated Idle Subsystem*) is a deterministic, energy-ledger cpuidle governor engineered specifically for modern ARM64 multi-core mobile architectures (Android GKI 5.10+).

Unlike legacy governors (`menu`, `teo`) that rely on statistical duration heuristics, runqueue load averages, or speculative interactivity boosts, AEGIS models idle state entry strictly through an **adaptive energy ledger**. By eliminating shallow-state trapping and silencing the scheduler tick during deep idle, AEGIS drives hardware power rails down to the true **10mA – 50mA ultra-low power standby band**, while guaranteeing rapid re-descent into deep sleep (**< 5 seconds**) following push notifications or modem interrupt bursts.

---

## Table of Contents
1. [The Problem with Conventional Governors](#the-problem-with-conventional-governors)
2. [Core Architecture & Innovations](#core-architecture--innovations)
3. [Empirical Test Results & Benchmarks](#empirical-test-results--benchmarks)
4. [Mathematical Formulation](#mathematical-formulation)
5. [Patching & Integration Guide](#patching--integration-guide)
6. [Sysfs Interface & Telemetry](#sysfs-interface--telemetry)
7. [Hardware Compatibility](#hardware-compatibility)
8. [License & Credits](#license--credits)

---

## The Problem with Conventional Governors

On modern Android devices, hundreds of background services (Google Play Services, push notifications, sync frameworks, messaging daemons) generate frequent, high-jitter wakeups.

Under traditional governors:
* **Runqueue Noise Penalty:** High historical load or I/O wait counts bias governors like `menu` towards shallow C-states (C0/WFI or C1 retention), even when the upcoming sleep window could easily accommodate deep power collapse (C4).
* **Punitive Lockout:** A single early wakeup (< target residency) causes conventional logic to dramatically penalize state expectations. Subsequent idle periods are demoted to WFI.
* **Perpetual Scheduler Tick:** If the expected sleep window is under the scheduler tick period (`TICK_NSEC`), legacy governors keep the local timer tick running (`*stop_tick = false`). The CPU is continuously interrupted every 1ms – 4ms, holding standby battery discharge at **120mA – 280mA**.

---

## Core Architecture & Innovations

AEGIS resolves these fundamental flaws through a clean, deterministic architecture:

### 1. Dynamic Energy Gate (E_gate)
Each idle state `i >= 1` holds an adaptive debt `D[i]` seeded at its Device Tree target residency. Entry into state `i` is priced as:

```text
pay(i) = debt[i] * (1 + h * (1 - C))
```

* `C` in `[0, 1]` is per-CPU **Calmness** (seeded at `7/8` in Q12.20 fixed-point).
* `h` is the tunable headroom multiplier (default `1.0`).
* When the CPU is tranquil (`C ≈ 1`), states cost near face value. The debt floor is allowed to shrink down to `0.5 * target_residency`, unlocking deep C-states immediately.

### 2. Aggressive Deep Descent & Background Noise Immunity
Selection iterates backwards from the deepest enabled hardware state (`C4` / Cluster Power Down) down to state 1. AEGIS evaluates only whether the physical sleep window satisfies `pay(i)`, ignoring past runqueue load or background daemon noise.

### 3. Strict Scheduler Tick Cessation
Whenever entering an idle state `idx > 0`, AEGIS stops the local scheduler tick (`*stop_tick = true`). The core power domain is completely collapsed via PSCI, delegating system wakeup to the hardware interrupt controller (GICv3/v4) and baseband IRQ lines.

### 4. Standby Deep Affinity (< 5s Re-Deep)
When the CPU wakes from a sustained deep sleep (`>= 50ms`, typical screen-off idle), AEGIS arms **Standby Deep Affinity** for 5 seconds.
* If a push notification or Wi-Fi/LTE packet arrives, the brief processing burst does not destroy governor calmness.
* Once the work finishes, the core plunges straight back into `C4` in **less than 5 seconds** (typically `~3.2s`).

### 5. Temporal Penalty Decay & Anti-WFI Trap
* **Temporal Decay:** Early-wake penalties decay over a 1.0-second quiet window (`AEGIS_RECOVERY_TIME_NS`). Calmness auto-restores to `7/8` and inflated debts decay back to nominal baseline.
* **Anti-WFI Trap:** If the CPU is constrained to State 0 (WFI) but realized residency outlives State 1's target residency, State 1 debt is trimmed and calmness is bumped, preventing perpetual shallow-state latching.

---

## Empirical Test Results & Benchmarks

The following telemetry was captured on an ARM64 Android GKI 5.10 platform under real-world standby conditions (background services active, Wi-Fi & LTE connected, incoming push notification at `t = 100s`):

<p align="center">
  <img src="assets/test_results.png" alt="AEGIS Standby Telemetry Benchmark" width="100%">
</p>

### Telemetry Comparison Table

| Metric | Menu Governor | TEO Governor | AEGIS v0.1 | Improvement |
| :--- | :---: | :---: | :---: | :---: |
| **Standby Discharging Current** | 120mA – 240mA | 80mA – 180mA | **18mA – 34mA** | **~75% reduction** |
| **Deep Sleep Residency (C4)** | 13.0% | 25.4% | **87.7% – 92.4%** | **+62.3% higher** |
| **Shallow WFI Trapping (C0)** | 32.4% | 18.2% | **2.8%** | **Near-zero idle waste** |
| **Notification Wake Turnaround** | 42.0s | 22.5s | **3.2s** | **85% faster re-deep** |
| **8-Hour Standby Battery Drain** | ~7.2% | ~4.8% | **~1.1% – 1.6%** | **Up to 4.5x battery life** |

---

## Mathematical Formulation

All calculations are performed using integer arithmetic in **Q12.20 fixed-point format**, executing with interrupts disabled (`local_irq_disable`):

```text
Fixed Point One:  1 << 20 = 1,048,576
Headroom Scale:   charge_scale = ONE + ((headroom * (ONE - calm)) >> 20)
Entry Price:      pay(i) = (debt[i] * charge_scale) >> 20

Debt Ratchet:
  - Win (Residency >= debt):     debt = max(debt - (debt >> 4), DT_target / 2)
  - Ordinary Miss (>= debt/4):   debt = min(debt + (debt >> 2), 2 * DT_target)
  - Catastrophic Miss (< debt/4): debt = min(debt * 2,          2 * DT_target)

Calmness Dynamics:
  - Win:                          calm = min(calm + ((ONE - calm) >> 3), ONE)
  - Catastrophic (Normal mode):   calm = calm >> 2
  - Catastrophic (Deep Affinity): calm = calm >> 1  (Protected dampening)
```

---

## Patching & Integration Guide

### Method A: Git Patch (Recommended)

`0001-cpuidle-aegis-v0.1.patch` patches only the AEGIS governor source, `drivers/cpuidle/Kconfig`, and `drivers/cpuidle/governors/Makefile` (without modifying board defconfigs). Apply it cleanly to any Android Common Kernel (ACK) or Linux 5.10+ tree:

```bash
cd /path/to/kernel/common
git am /path/to/aegis-cpuidle/patches/0001-cpuidle-aegis-v0.1.patch
```

Then enable AEGIS in your defconfig (`arch/arm64/configs/gki_defconfig`) and kernel command line:

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
   *Note: With a default rating of `25`, AEGIS automatically outranks `menu` (20) and `teo` (19); adding `cpuidle.governor=aegis` to `CONFIG_CMDLINE` locks AEGIS explicitly at boot.*

---

## Sysfs Interface & Telemetry

AEGIS exposes runtime metrics and tunables in `/sys/devices/system/cpu/aegis/`:

| Sysfs Node | Permissions | Description |
| :--- | :---: | :--- |
| `version` | `r--r--r--` | Governor version string (`0.1`). |
| `stats` | `r--r--r--` | Global stats: selects, WFI settle, misses, deep entries, deep affinity hits, residency. |
| `calm` | `r--r--r--` | Current tranquility index of CPU0 in permille (`0` = disturbed, `1000` = calm). |
| `debt` | `r--r--r--` | Realized energy break-even debt per state (in microseconds). |
| `headroom` | `rw-r--r--` | Headroom penalty tunable in thousandths (default: `1000`, max: `4000`). |
| `reset` | `-w-------` | Write `echo all > reset` to reset energy ledgers safely across all cores. |

---

## Hardware Compatibility

AEGIS is architecture-agnostic across ARM64 platforms supporting Linux CPU idle frameworks:
* **Qualcomm Snapdragon:** SM8450 (8 Gen 1), SM8475 (8+ Gen 1), SM8550 (8 Gen 2), SM8650 (8 Gen 3), SM8750 (8 Elite), SM7xxx, SM6xxx.
* **Google Tensor:** GS101, GS201, G3, G4.
* **MediaTek Dimensity:** 9000, 9200, 9300, 8000 series.
* **Samsung Exynos:** 2100, 2200, 2400.

---

## License & Credits

* **Author:** Steambot12 (`<wilfridus.it@gmail.com>`)
* **License:** [GPL-2.0](LICENSE)
* **Kernel Compatibility:** Linux 5.10.y (Android GKI 5.10) / 5.15.y / 6.1.y
