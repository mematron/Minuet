# ARMINTA (formerly Minuet)

### Autonomous Causal Discovery Agent for Linux

ARMINTA is a Python-based autonomous agent that treats the host operating system as an **interactive substrate**. Rather than serving as a passive monitor, it views the OS as a causal field to be interrogated through planned intervention. It observes, acts, measures effect, and updates a live causal graph to resolve system-level performance bottlenecks.

As of the latest telemetry, ARMINTA has completed **92,000+ live steps** on target hardware. It has constructed a private causal world model derived entirely from empirical measurement; no pretrained weights, no external knowledge, and no simulation. Every edge in its graph has been earned through direct intervention and observation.

> Source is closed. This repository documents the architecture, design philosophy, and version lineage of the Arminta engine.

---

## Core Operational Loop

ARMINTA operates as a root-privileged background process. Every 0.8 to 2.5 seconds (utilizing an adaptive step rate), it executes the following cycle:

1.  **Sampling**: Collects ~28 system metrics across CPU, memory, thermals, network, I/O, swap, PSI, and IRQ states.
2.  **Classification**: Derives the current "Session Geometry": a workload fingerprint based on resource ratios rather than process names.
3.  **Cognitive Selection**: Utilizes a high-level **Q-learning Mode Controller** to select a cognitive posture (`OBSERVE`, `INVESTIGATE`, `OPTIMIZE`, `DREAM`, or `SELF_ASSESS`).
4.  **Action Execution**: Within the chosen mode, selects and executes an action via the causal graph and learned confidence scores.
5.  **Measurement**: Captures the before/after delta across targeted metrics within a precise 300ms window.
6.  **Causal Update**: Updates the interventional edge for the `(action, metric)` pair, applying recency decay and confound filtering.
7.  **Episodic Logging**: Records the complete state (action, outcome, reward, and emotional affect) to a persistent SQLite database.

---

## Architecture

### Cognitive Hierarchy

ARMINTA employs a "double-loop" learning architecture. A high-level reinforcement learning agent manages the system's cognitive focus, while a lower-level causal engine manages system interventions.

```mermaid
graph TD
    %% Styling Configuration
    classDef default fill:#11111b,stroke:#a6adc8,stroke-width:1px,color:#cdd6f4;
    classDef memory fill:#1e1e2e,stroke:#cdd6f4,stroke-width:1px,color:#cdd6f4;
    
    ModeController["Mode Controller <br/> (Q-Learning Over Cognitive Postures)"]
    EpisodicMemory["EpisodicMemory <br/> (1,800+ Recorded Episodes in SQLite)"]:::memory
    BayesianPerception["BayesianPerception <br/> (Belief Updating & Noise Smoothing)"]
    WorldModel["WorldModel <br/> (State-Action Outcome Statistics)"]
    EmotionalState["EmotionalState <br/> (Affective Modulation: Calm, Bored, Stressed, etc.)"]
    HypothesisEngine["HypothesisEngine <br/> (Genetic Algorithm over Causal Nodes)"]
    MetaCognition["MetaCognition <br/> (AST-Based Source Code Rewriting)"]
    
    %% Interconnections
    ModeController -->|Selects Mode| BayesianPerception
    BayesianPerception -->|Updates Belief| ModeController
    EmotionalState -->|Modulates Thresholds| ModeController
    ModeController -->|Triggers Dream| HypothesisEngine
    HypothesisEngine -->|Refines Edges| WorldModel
    WorldModel -->|Logs Anomalies| EpisodicMemory
    ModeController -->|Self-Assess| MetaCognition
    MetaCognition -->|Rewrites Constants| ModeController
```

---

### TrueCausalGraph & Poison Registry

The reasoning engine is strictly interventional, utilizing the distinction between observation and intervention (do-calculus). 

*   **Interventional Edges**: Every `(action, metric)` pair is stored as a distribution of normalized deltas. Confidence is weighted by sample count.
*   **Poison Edge Registry**: To prevent "confound poisoning," the agent maintains a hard-coded registry of structurally impossible causal paths. For instance, `renice_ksoftirqd` is prohibited from being credited with changes in `temp_c` or `net_recv_kbps` within the 300ms window, ensuring the graph remains grounded in physical reality.
*   **Reward-Discount Layer**: If an action's metric effects appear positive but its rewards are consistently negative (a selection-bias signature), the graph's recommendation is discounted proportionally.

### Advanced Metacognition (Self-Rewriting)

Unlike traditional agents, ARMINTA possesses the ability to modify its own source code. In `SELF_ASSESS` mode, the **MetaCognition** module can perform **AST-based rewriting** of the script's own constants (e.g., `STEP_RATE`, `PSI_THRESHOLDS`). This process includes:
1.  **Validation**: Syntax and linting checks via `ast.parse`.
2.  **Atomic Commit**: Safe replacement of the running script on disk.
3.  **Automated Backups**: Retention of versioned `.bak` files for recovery.

### Affective Modulation (Emotion Model)

The agent's internal state is modulated by seven continuous affects: `calm`, `curious`, `stressed`, `confident`, `frustrated`, `bored`, and `focused`. 
*   **Boredom**: Emerges from sustained periods of high `calm` and low `stressed`, triggering increased `curiosity` and proactive probes.
*   **Frustration**: Accumulates from failed interventions, leading to shifts in cognitive mode to `INVESTIGATE` or `DREAM`.

---

### System Integration Details

*   **PSI Safety Interlock**: ARMINTA utilizes Linux **Pressure Stall Information**. A hard interlock (`PSI_MEM_DROP_CACHES_SUPPRESS = 40.0`) prevents the agent from triggering `drop_caches` when memory stalls are active, as evicting clean pages during a stall would exacerbate system pressure.
*   **Session Geometry**: Six continuous features (e.g., `sess_net_vs_disk`, `sess_proc_cpu_dilution`) allow the agent to learn context-specific behaviors. It understands that a high CPU load during a "compile" session requires different handling than the same load during a "streaming" session.
*   **Browser Taxonomy**: A brand-agnostic classifier identifies browser processes by architectural flags. It specifically targets **Extension Renderers** (Priority 1) for escalation, as they can be killed and auto-restarted with zero impact on the user's active tabs.

---

## Persistence & Progress

ARMINTA carries its entire learned history across sessions via a unified state pickle and a dedicated episodic database:
*   **92,063 Steps** of experience on target hardware.
*   **1,805 Episodes** logged, documenting every major hypothesis and self-modification.
*   **Version-Agnostic Migration**: Automatic state upgrading from prior versions back to v86.

---

## Version Lineage (Recent Highlights)

| Version | Milestone |
|---|---|
| **Minuet v105** | Introduction of full cognitive layer (Emotion, Self-Model, Episodic DB). |
| **Minuet v106** | Terminal corruption prevention and final Minuet stability release. |
| **Arminta v2** | **Extension Renderer Sweep**: Implementation of Priority-1 targeting, allowing surgical intervention in browser-heavy workloads with zero user-visible impact. |

---

## Relationship to SUKOSHI

ARMINTA is the local substrate predecessor to [SUKOSHI](https://ardorlyceum.itch.io/sukoshi), a browser-native causal entity built on Paramorphic Learning and genetic algorithm hypothesis evolution.

---

## Part of the BIOS of Being Framework

ARMINTA exists within a larger system. See: [ardorlyceum.itch.io](https://ardorlyceum.itch.io) · [mematron.hearnow.com](https://mematron.hearnow.com) · [keygentia.netlify.app](https://keygentia.netlify.app)
