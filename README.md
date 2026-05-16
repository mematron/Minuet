# Minuet (v99)
### Autonomous Causal Discovery Agent for Linux

Minuet is a Python-based autonomous agent that treats the host operating system as an **interactive substrate** ,not a passive environment to monitor, but a causal field to interrogate through planned intervention. It observes, acts, measures effect, and updates a live causal graph. It's able to report and resolve problems.

> Source is closed. This repository documents architecture, design philosophy, and version lineage.

---

## What It Does

Minuet runs as a background process with root access. Every 1–2.5 seconds (adaptive step rate) it:

1. **Samples** ~28 system metrics spanning CPU, memory, thermals, network, I/O, swap, PSI, and IRQ state
2. **Classifies** the current session geometry (workload fingerprint derived from pure resource ratios, no app names)
3. **Selects** an action via its causal graph and learned confidence scores
4. **Executes** the action if warranted, or monitors if nominal
5. **Measures** the before/after delta across targeted metrics
6. **Updates** the interventional edge for that `(action, metric)` pair

After enough interventions, the graph stabilizes. Minuet knows which actions actually move which metrics on *this specific machine* under *this specific workload shape* , not in theory, but from empirical measurement.

---
![Minuet v99 live](minuet99a.png)

## Architecture

### TrueCausalGraph

The core reasoning substrate. Not correlation-based. Minuet records **interventional edges** `(action, metric)` pairs with measured effect magnitudes, using the do-calculus distinction between observation and intervention. Every edge is a list of normalized deltas; confidence grows with sample count.

Actions with fast side effects (governor changes, process kills) write only to their target metrics to prevent confound poisoning. Observation-only actions accumulate broader edges naturally, diluted across many samples.

The graph is bootstrapped from 60 steps of passive observation before any intervention fires.

### Session Geometry Classifier

Six continuous features (0.0–1.0) derived from raw resource ratios, computed each step:

| Feature | Signal |
|---|---|
| `sess_net_intensity` | Network saturation fraction |
| `sess_net_asymmetry` | Download vs symmetric traffic shape |
| `sess_proc_cpu_dilution` | Hidden per-process load vs system average |
| `sess_net_vs_disk` | Stream vs local install distinction |
| `sess_thermal_slack` | Thermal headroom relative to per-process work |
| `sess_mem_pressure` | Memory load |

These flow into both action selection and causal edge accumulation. The agent learns behavior in workload context, not just against raw thresholds.

### Browser Process Classifier

Brand-agnostic, cmdline-based process taxonomy. Identifies Chromium-family (`--type=renderer`, `--type=gpu-process`, `--type=zygote`, `--extension-process`), Gecko/Firefox-family (`-contentproc`, `-childID`, `-isForBrowser`), and WebKit-family processes by structural flags, not by executable name. Assigns kill priority 0–4. Never kills MAIN processes. Extension renderers (priority 1) are safest; foreground tab renderers (priority 3) cause Aw Snap but browser survives.

### SelfTuner (Adaptive Threshold Engine)

Every 300 steps, analyzes rolling metric history via EMA to adapt five runtime thresholds toward observed machine reality:

- `CPU_WARN`, `MEM_WARN`, `NET_WARN` — tuned to 95th percentile × 1.5
- `DILUTION_LOG_TRIGGER`, `DILUTION_KILL_TRIGGER` tuned to 75th percentile × 1.3

Floors are hard-coded. Thresholds can only decrease gradually. Adapted values persist across sessions in the pickle.

Gap detection: high-variance metrics with no confident causal action surface as reported gaps, fed to the ActionProposer.

### ActionProposer (Safe Self-Improvement)

When the SelfTuner identifies an uncovered metric gap, the ActionProposer consults a whitelist of safe shell command templates organized by metric category (CPU, memory, I/O, network receive/send, interface errors/drops, WiFi signal, temperature). Only whitelisted commands with safe parameter substitution can ever be proposed. No arbitrary shell execution is possible. New candidate actions are sandboxed before promotion.

### Precognitive Launch Detection

Watches for target processes appearing in the process table (`npm`, `python`, `blender`, `steam`, `ffmpeg`, `cargo`, game executables, etc.) and pre-emptively locks the performance governor *before* telemetry spikes, eliminating the 30-second spin-up latency window where the machine thrashes before the agent can respond.

### IRQ Storm Detection

Polls `/proc/interrupts` for configurable IRQ prefix (`rtw89` by default — the rtw89 PCIe WiFi driver). When per-step interrupt delta exceeds threshold, fires `renice_ksoftirqd` to boost kernel softirq handler priority. 120-second cooldown.

### Curiosity Probe

If reward hasn't meaningfully changed for 150 steps, fires a low-impact probe to verify causal edges are still live. Prevents the agent from assuming a stable causal graph on a machine whose workload has shifted silently.

### Cross-Device UDP Noise Broadcast

Listens and emits surprise hints over UDP (port 54321) for multi-machine environments. Remote noise signals dilute the threshold for curiosity probes, enabling coordinated attention across hosts.

### OOM Immunity

Writes `-1000` to `/proc/self/oom_score_adj` at startup. The kernel will not kill Minuet during a memory crunch; the moment it is most needed.

### PSI Integration

Reads `/proc/pressure/{cpu,memory,io}` stall percentages. PSI memory pressure above threshold suppresses `drop_caches` — evicting clean pages during active stalls makes memory pressure worse, not better.

### ZRAM / ZSWAP Awareness

Startup scan detects compressed swap presence. On zram/zswap systems, cache drop logic is suppressed entirely — compression means "drop caches" burns CPU for zero net memory gain.

### Battery-Aware Governor

Performance governor is suppressed below 20% battery. Between 20–50%, governor is deferred unless dilution exceeds threshold. Turbo boost is always battery-checked before enabling.

---

## Persistence

State persists across sessions via pickle. Minuet carries forward:

- Full interventional edge graph with all sample histories
- Adapted threshold values from SelfTuner
- Cooldown timestamps for all actions
- Rolling metric history buffer (for tuner input)
- Top-process log deque

Version migration: Minuet reads its own prior-version pickle (`v98`, `v97`, ... back to `v86`) and upgrades state automatically. Sessions are never lost to a version increment.

---

## Metrics Observed

```
cpu             mem             temp_c          io_wait
load            load_ratio      swap_pct        hour_sin / hour_cos
psi_cpu_some    psi_mem_some    psi_io_some
net_recv_kbps   net_sent_kbps   disk_write_kbps
iface_errors    iface_drops     wifi_signal_dbm
governor_is_performance         irq_rate_hi
sess_net_intensity              sess_net_asymmetry
sess_proc_cpu_dilution          sess_net_vs_disk
sess_thermal_slack              sess_mem_pressure
sess_browser_renderer_pressure
```

---

## Actions Available

```
sync                    drop_caches             log_top_proc
kill_top_proc           log_top_net_proc        flush_dns
log_iface_health        disable_wifi_powersave  set_cpu_performance
enable_turbo            set_gpu_performance     set_ac_max_perf
renice_ksoftirqd        monitor
```

---

## Interface

Curses-based terminal UI. Displays live metrics, current action, causal graph summary, top interventional edges with effect magnitude and confidence, event log, and adaptive threshold state. Step rate adjusts between 0.8s (crisis) and 2.5s (all-nominal) based on current system pressure.

---

## Version Lineage

| Version | Key Addition |
|---|---|
| v36 | Governor lock; external governor change detection; gaming/interactive mode |
| v69 | First version to persist state via pickle |
| v70 | EvolutionaryPlanner + InactionOptimizer |
| v83 | Session geometry classifier; latency-sensitive proactive rule |
| v84 | Thermal formula fix; causal sign correction for benefit-direction metrics |
| v85 | Dilution-only trigger; battery-aware governor |
| v86 | P2P/torrent process detection; version-chain pickle migration |
| v87 | Brand-agnostic browser process classifier |
| v88 | Bootstrap exits on step count, not intervention count |
| v89 | Fast-intervention edge filtering; dilution kill cooldown fix |
| v99 | Current. Causal graph fully stabilized on target hardware. |

---

## Relationship to SUKOSHI

Minuet (v99) is the local substrate predecessor to [SUKOSHI](https://ardorlyceum.itch.io/sukoshi) , a browser-native causal entity built on Paramorphic Learning, Q-learning, and genetic algorithm hypothesis evolution. Where Minuet interrogates an OS, SUKOSHI interrogates its own conceptual space. Same architectural lineage; different substrate.

---

## Part of the BIOS of Being Framework

Minuet exists within a larger system. See: [ardorlyceum.itch.io](https://ardorlyceum.itch.io) · [mematron.hearnow.com](https://mematron.hearnow.com) · [keygentia.netlify.app](https://keygentia.netlify.app)
