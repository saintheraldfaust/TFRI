# Project Littleman — Engineering System Architecture

> Full-stack, engineering-level architecture reference for implementation teams.

---

## Table of Contents

1. [High-Level System Topology](#1-high-level-system-topology)
2. [Internal Data Flow](#2-internal-data-flow-event-lifecycle)
3. [Core Subsystems](#3-core-subsystems)
4. [Deployment Architecture](#4-deployment-architecture)
5. [Constraints & Engineering Challenges](#5-constraints--engineering-challenges)
6. [Earbud Interaction Protocol](#6-earbud-interaction-protocol)

---

## 1. High-Level System Topology

```
                   ┌────────────────────────────┐
                   │   External Hardware Layer   │
                   │  (Drones, Cameras, IoT)     │
                   └────────────┬───────────────┘
                                │
                       Bluetooth / WiFi
                                │
┌───────────────────────────────────────────────────────────┐
│                    ANDROID DEVICE (CORE)                  │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐  │
│  │                  Perception Layer                   │  │
│  │  • Audio Listener (Hotword + Streaming ASR)         │  │
│  │  • Camera Pipeline (Realtime Frames)                │  │
│  │  • Sensor Input (GPS, Motion, Device State)         │  │
│  └──────────────────────┬──────────────────────────────┘  │
│                         │                                 │
│  ┌──────────────────────▼──────────────────────────────┐  │
│  │               Interpretation Layer                  │  │
│  │  • On-device LLM (Gemma-class, quantized)           │  │
│  │  • Intent Classifier                                │  │
│  │  • Context Engine (state + memory)                  │  │
│  └──────────────────────┬──────────────────────────────┘  │
│                         │                                 │
│  ┌──────────────────────▼──────────────────────────────┐  │
│  │               Anticipation Engine                   │  │
│  │  • Trigger Detection (keywords + semantic)          │  │
│  │  • Mapping Engine (Trigger → Intent → Action Graph) │  │
│  │  • Priority Resolver (conflict handling)            │  │
│  └──────────────────────┬──────────────────────────────┘  │
│                         │                                 │
│  ┌──────────────────────▼──────────────────────────────┐  │
│  │                 Execution Engine                    │  │
│  │  • Script Executor (deterministic flows)            │  │
│  │  • Agent Executor (LLM reasoning)                   │  │
│  │  • Device Control Interface                         │  │
│  └──────────────────────┬──────────────────────────────┘  │
│                         │                                 │
│  ┌──────────────────────▼──────────────────────────────┐  │
│  │               Android Control Layer                 │  │
│  │  • Accessibility API (tap, scroll, type)            │  │
│  │  • Intent System (launch apps, services)            │  │
│  │  • Secure Storage (keys, configs)                   │  │
│  └──────────────────────┬──────────────────────────────┘  │
│                         │                                 │
│  ┌──────────────────────▼──────────────────────────────┐  │
│  │               Communication Layer                   │  │
│  │  • Bluetooth Manager (Earbuds, hardware)            │  │
│  │  • Encrypted Messaging (alerts/logs)                │  │
│  │  • Remote Sync (optional cloud relay)               │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

---

## 2. Internal Data Flow (Event Lifecycle)

```
[Audio Input / Sensor Event]
            │
            ▼
[Hotword Detection / Trigger Match]
            │
            ▼
[Intent Classification]
            │
            ▼
[Anticipation Engine Lookup]
            │
     ┌──────┴────────┐
     ▼               ▼
[Script Path]   [Agent Reasoning Path]
     │               │
     └──────┬────────┘
            ▼
[Execution Engine → Device Control]
            │
            ▼
[External Actions + Feedback Loop]
```

---

## 3. Core Subsystems

### 3.1 Audio Subsystem

Always-on, low-power design with a two-stage pipeline: keyword spotting wakes the system, then high-fidelity transcription takes over.

**Stack:**
- [Vosk](https://alphacephei.com/vosk/) or [Whisper](https://github.com/openai/whisper) (quantized for on-device)
- Custom hotword engine (Snowboy-style)

**Design goals:**
- Local-first processing (no network dependency for trigger detection)
- Low-power idle state between trigger events

---

### 3.2 Vision Subsystem

Frame sampling rather than a full continuous stream to preserve battery. Event-based activation (motion, anomaly detection) drives the pipeline.

**Functions:**
- Object detection
- Face recognition *(optional, user-configured)*
- Scene classification

---

### 3.3 Anticipation Engine

The critical core of the system. Each anticipation is defined as a JSON record mapping a trigger to an intent and an ordered action graph.

**Anticipation data model:**

```json
{
  "trigger": "KL45",
  "type": "explicit",
  "intent": "emergency_protocol",
  "priority": 10,
  "actions": [
    { "type": "call",     "target": "emergency_contact" },
    { "type": "send_sms", "payload": "HELP + GPS" },
    { "type": "record",   "mode": "video+audio" }
  ]
}
```

**Priority field** is used by the Priority Resolver to handle conflicting simultaneous triggers. Higher value wins.

---

### 3.4 Execution Engine

Two execution modes run in parallel, selected based on trigger type.

| Mode | Trigger Type | Latency | Notes |
|------|-------------|---------|-------|
| **Script Execution** | Explicit / pre-defined | <100ms | Deterministic, no LLM call |
| **Agent Execution** | Semantic / ambiguous | 500–2000ms | LLM builds action plan dynamically |

Agent execution is used when:
- The trigger is ambiguous or novel
- Context significantly affects the correct response
- No matching explicit anticipation exists

---

### 3.5 Device Control Layer

Full device orchestration via Android Accessibility Service. The system simulates human interaction at the OS level, enabling cross-app automation without requiring per-app integrations.

**Capabilities:**
- Simulated gestures (tap, swipe, long-press, type)
- App launch and navigation via Intent system
- Secure storage read/write (keys, configs, anticipation definitions)

> **Note:** Accessibility Service requires explicit user grant and surfaces a persistent system notification. This is a platform constraint, not a design choice.

---

### 3.6 Hardware Integration Layer

**Communication protocols:**

| Protocol | Use Case |
|----------|----------|
| BLE (Bluetooth Low Energy) | Earbuds, wearables, low-bandwidth sensors |
| WiFi Direct | High-bandwidth devices (drones, external cameras) |

**Command lifecycle:**

```
Device → Command → Acknowledgement → State Sync
```

---

## 4. Deployment Architecture

| Phase | Description | Cloud Dependency |
|-------|-------------|-----------------|
| **Phase 1 — MVP** | Fully on-device; all processing local | None |
| **Phase 2** | Secure relay server; remote monitoring dashboard | Optional relay |
| **Phase 3** | Distributed agent network; multi-device coordination | Required |

Phase 1 is the target for initial prototyping on the reference device (Tecno Camon 40 Pro Plus).

---

## 5. Constraints & Engineering Challenges

### 5.1 Android Background Execution

**Problem:** Android aggressively kills background processes and restricts microphone access for non-foreground apps.

**Solution:**
- Run core pipeline as a **Foreground Service** (displays a persistent notification — this is an OS requirement and cannot be suppressed without root)
- Use `RECORD_AUDIO` with foreground service declaration in `AndroidManifest.xml`

### 5.2 Battery Optimization

| Strategy | Description |
|----------|-------------|
| Event-driven activation | Full pipeline only runs on trigger; idle state uses minimal CPU |
| Model quantization | INT8 / INT4 quantized models (e.g., Gemma 2B via llama.cpp or MediaPipe LLM Inference) |
| Sensor fusion | Combine low-cost sensors to avoid redundant high-cost processing |

---

## 6. Earbud Interaction Protocol

The earbud is not a peripheral — it is the **primary command interface** for scenarios where screen interaction is unsafe or impossible.

### 6.1 Design Principles

1. **Stealth-first** — no visible interaction required
2. **Low cognitive load** — commands are simple and pre-learned
3. **Sub-second latency** — action must start within 1s of trigger
4. **Non-verbal fallback** — tap signals available when voice is unsafe

---

### 6.2 Interaction Modes

#### Mode A — Voice Command *(Primary)*

User speaks naturally; the system handles intent classification.

```
Earbud Mic → Phone → ASR → Intent Classifier → Anticipation Engine → Action
```

Examples:
- `"KL45"` — explicit code trigger
- `"Is that a black cat?"` — semantic trigger
- `"I don't feel safe"` — latent/reasoned trigger

#### Mode B — Tap Signals *(Silent Commands)*

| Pattern | Action |
|---------|--------|
| Double tap | Trigger primary alert |
| Triple tap | Execute fallback plan |
| Long press | Cancel pending action |

#### Mode C — Passive Ambient Listening

- Stress detection in ambient speech
- Keyword detection within normal conversation
- No explicit command required from user

#### Mode D — Whisper Mode

- Extremely low-volume input via close-mic detection
- For covert trigger scenarios

---

### 6.3 Feedback System

The user must know an action is executing **without any visible indication**.

#### Audio Micro-Feedback (Primary)

Subtle tones delivered privately via earbud:

| Tone Pattern | Meaning |
|-------------|---------|
| 1 short beep | Trigger detected |
| 2 short beeps | Action executing |
| 1 long beep | Action completed |
| Rapid beeps | Error |

#### No-Feedback Mode

For extreme stealth scenarios where even earbud audio is a risk.

#### Bone Conduction *(Future)*

Silent haptic-style confirmations with zero audible leakage.

---

### 6.4 Secure Command Validation

Multi-layer validation prevents accidental triggers:

1. Keyword/pattern match
2. Voice signature verification *(optional, user-configured)*
3. Context validation (location, time, prior state)

---

### 6.5 Anti-Coercion Design

For scenarios where the user is under duress and may be forced to "cancel" the system:

- **Fake confirmation signals** — system appears cancelled but continues executing
- **Delayed execution** — action fires after a configurable delay, allowing genuine cancellation via a separate secret gesture
- **Reverse triggers** — a covert phrase silently cancels everything without outward sign

---

### 6.6 Earbud ↔ Phone BLE Protocol

**Input message (earbud → phone):**

```json
{
  "type": "input",
  "mode": "tap",
  "pattern": "double",
  "timestamp": 1710000000
}
```

**Response message (phone → earbud):**

```json
{
  "type": "feedback",
  "signal": "short_beep"
}
```

---

### 6.7 Latency Targets

| Operation | Target |
|-----------|--------|
| Trigger detection | < 300ms |
| Action start | < 800ms |
| Feedback to user | < 200ms |

---

*The Faust Research Institute (TFRI) — v0.1 Foundational Draft*
