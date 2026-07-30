# Friday : An Ambient Intelligence


* **Problem Number** — 3
* **Problem Statement** — AI off the Grid
* **Team name** — Team Stack Overflow
* **Team members** — Soumya Gupta, Nirvik Goswami
* **Institute** — Vellore Institute of Technology Chennai

## Project Overview

**FRIDAY** is an ambient intelligence layer for Android that detects cognitive overload and burnout signals in real time — and responds by doing less, not more. Rather than adding another notification to an already overwhelmed user, FRIDAY enforces silence when it matters most.

The system captures passive behavioral signals (typing speed, error rate, notification backlog, time-of-day, session length) and produces a continuous urgency score. When that score drops below a threshold, the system enters **empathetic silence** — deferring non-critical prompts, suppressing low-value alerts, and reducing interface friction.

### Key technical differentiator

A fine-tuned RoBERTa regression model replaces heuristic scoring entirely. Trained on synthetic Android telemetry mapped to a `[0, 1]` burnout score, the model runs on-device via ONNX Runtime — no cloud call, no latency, no data exfiltration. The decision to act or stay silent happens in under 80 ms.

The backend intelligence layer is powered by **Google Gemma 4** (E4B variant) running locally via Ollama — a fully open-weight model from the Gemma 4 family, providing multimodal reasoning capabilities with up to 256K context windows while keeping all inference on the local network.


## Project Artefacts

* **Technical Documentation** — [docs/](./docs/) — architecture, agent design, API reference, and user guides
* **Agentic Setup (read this first)** — [docs/ax.md](./docs/ax.md) — LangGraph orchestration, memory strategy, open-weight model choices, and implementation notes
* **Source Code** — [src/](./src/) — Android app, FastAPI backend, Chrome Extension

### Models used

| Model | Role |
|---|---|
| Fine-tuned RoBERTa (ours) | On-device burnout regression (ONNX) |
| **Gemma 4 E4B** (Google, via Ollama) | Backend orchestration, empathetic response generation, and summarisation |
| Phi-3 Mini | Offline mobile reasoning fallback (ONNX) |
| whisper.cpp | Zero-latency voice transcription (C++ binding) |
| Coqui TTS | Empathetic voice output |
| all-MiniLM-L6-v2 | Semantic embeddings for context retrieval (ChromaDB) |

### Datasets used


**Dataset published**: [Rabbit-bot/FRIDAY-Synthetic-Burnout-Telemetry](https://huggingface.co/) — Apache 2.0

#### Reference Dataset

| Dataset | Used for |
|---|---|
| WESAD | Physiological stress baseline |
| StudentLife | Longitudinal burnout pattern reference |
| SWELL-KW | Cognitive load and task-switch modeling |
| ExtraSensory | Activity and context signal calibration |
| FRIDAY Synthetic Telemetry (ours) | RoBERTa fine-tune training set |

 #### Due to permission requirement and dataset size we were unable use the above dataset.


---

## Architecture

FRIDAY is a three-layer distributed system designed so the critical path never leaves the device.

### Layer 1 — Signal capture (Android)

`NotificationListenerService` and `AccessibilityService` collect raw behavioral events. The AccessibilityService uses targeted View ID lookups only (browser URL bars, YouTube titles) — no recursive UI tree traversal. App switch counts are sourced from the system `UsageStatsManager` API. A local Room Database buffers all signals to prevent data loss during network drops. No raw data is ever transmitted off-device.

### Layer 2 — Intelligence (Compute Hub)

A FastAPI backend runs a multi-agent pipeline with specialized agents: Emotion, Memory, Decision, Wellbeing, Burnout, Context, and Notification. Each agent processes its domain independently; a routing-based orchestrator combines their outputs into a single scored intervention decision.

**Scoring formula** — each candidate response is scored using five weighted dimensions:

```
SCORE = (Emotional_Relevance × 0.30)
      + (Timing             × 0.25)
      + (Memory_Alignment   × 0.20)
      + (Action_Quality     × 0.10)
      + (¬Intrusiveness     × 0.15)
```

Responses below a threshold of 40 are silenced (empathetic silence). Weights are adjusted dynamically via an **RLHF-lite feedback loop** — user reactions (helpful / dismissed / ignored) shift the scoring weights in real time, making FRIDAY adaptive rather than static.

**LLM backbone** — Gemma 4 E4B runs locally via Ollama, generating empathetic response candidates using turn-based prompting. The system is model-agnostic by design: ChromaDB semantic memory, SQLite KPI logging, and the scoring formula all consume plain text output regardless of which LLM sits in the slot.

### Layer 3 — Experience (Cross-device)

A Chrome Extension connects to the Compute Hub over secure WebSockets. It uses an isolated Shadow DOM so it never interferes with the host page's styles or scripts. When the hub is unreachable, Phi-3 Mini activates locally via ONNX Runtime within 200 ms.


## Measured performance

| Metric | Result |
|---|---|
| Burnout model inference latency | < 80 ms on-device |
| Offline fallback activation time | < 200 ms |
| Cross-device workflow handoff | < 1.2 s |
| Additional battery consumption | < 3% |
| RAM footprint (Android service) | ~45 MB |

> *Figures measured on a Galaxy S23 (Snapdragon 8 Gen 2) running Android 14 with the compute hub on a local LAN.*


## Key features

**Privacy-first by design** — AES-256 encrypted local storage. Behavioral telemetry never leaves the local network. The on-device model means the most sensitive inference path has no network dependency at all.

**Empathetic silence engine** — instead of firing another notification, FRIDAY suppresses low-value interruptions when the user's burnout score exceeds a threshold. The system actively reduces cognitive load rather than contributing to it.

**Semantic workspace memory** — ChromaDB stores vectorised session context so FRIDAY can restore a complex multi-tab workflow after a break without asking the user to reconstruct it manually.

**Dynamic power states** — three operating modes (Ghost, Aware, Active) trade capability for battery life based on user activity. Ghost Mode draws negligible power while still logging passive signals.

**RLHF-lite adaptive weights** — user feedback shifts the scoring weights in real time. Helpful responses reinforce emotional relevance and memory; dismissed responses reduce the intrusiveness penalty. Weights are renormalized every 10 interactions to prevent drift.

## Innovation and impact

### Why this is technically novel

Most ambient intelligence systems are cloud-dependent — they offload inference to keep the device lightweight, at the cost of latency and privacy. FRIDAY inverts this: the critical scoring model runs entirely on-device via ONNX, while the local backend handles reasoning tasks using Gemma 4 — Google's open-weight model family. The result is a system that can make real-time decisions even in airplane mode.

The Gemma 4 integration is model-agnostic by design: the scoring formula, memory retrieval, and KPI logging pipeline are all LLM-independent. Swapping to any Ollama-compatible model requires changing a single configuration string.

Using `whisper.cpp` C++ bindings eliminates Python GIL bottlenecks, achieving sub-second voice round-trips that pure Python implementations cannot match on mobile hardware.

### Alignment with Samsung's ecosystem

FRIDAY integrates naturally with Samsung Health SDK (physiological baselines), Galaxy Continuity (cross-device handoff), and Knox (enterprise security policy). The edge-compute architecture scales with device hardware — no server costs grow with user count.

### Market context

Digital wellness and productivity tooling for Gen Z is a $10B+ market with no dominant platform-level player. FRIDAY's competitive position is the combination of privacy-by-design and hardware-optimized local inference — neither of which generic cloud API products can replicate.


## Attribution

Built from scratch using the following open-source technologies:

* **Google Gemma 4** — local LLM reasoning (Apache 2.0)
* **ChromaDB** — local vector memory
* **Ollama** — localized LLM serving
* **whisper.cpp** — high-performance audio transcription
* **ONNX Runtime** — on-device model inference
* **Android Jetpack** — Room, WorkManager, Compose


## License

Apache License 2.0 — see [LICENSE](./LICENSE) for details.




FRIDAY is a proof that ambient intelligence does not require surveillance. By keeping the decision loop on-device and enforcing silence as a first-class response, it demonstrates that empathetic AI means knowing when not to act — not just when to act faster.
