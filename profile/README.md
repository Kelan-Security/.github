# Kelan Security

> **Zero-trust network enforcement — local AI, eBPF kernel control, post-quantum cryptography.**
> No cloud. No SaaS. No telemetry. Everything runs on your hardware.

[![CI](https://img.shields.io/github/actions/workflow/status/Kelan-Security/kelan-core/ci.yml?style=flat-square&labelColor=0D1117&color=0F6E56&label=CI)](https://github.com/Kelan-Security/kelan-core/actions)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-D22128?style=flat-square&labelColor=0D1117)](https://github.com/Kelan-Security/kelan-core/blob/main/LICENSE)
[![Rust](https://img.shields.io/badge/Rust-1.80+-CE422B?style=flat-square&labelColor=0D1117)]()
[![Python](https://img.shields.io/badge/Python-3.12-3776AB?style=flat-square&labelColor=0D1117)]()
[![Ollama](https://img.shields.io/badge/Ollama-Gemma%204-111111?style=flat-square)]()
[![eBPF](https://img.shields.io/badge/eBPF-Kernel%20Level-00C853?style=flat-square&labelColor=0D1117)]()
[![Status](https://img.shields.io/badge/Status-Active%20Development-2962FF?style=flat-square&labelColor=0D1117)]()
[![IEEE CNS 2026](https://img.shields.io/badge/IEEE%20CNS-2026%20submission-0F6E56?style=flat-square&labelColor=0D1117)](https://github.com/Kelan-Security/kelan-core)

---

## The problem with "zero-trust" today

Every enterprise zero-trust product shares the same hidden architecture: **the AI that decides allow or block runs in the vendor's cloud.** Your traffic metadata — connection patterns, session behavior, protocol anomalies — leaves your network on every single decision.

For regulated industries, air-gapped environments, or any organization serious about data sovereignty, that is a fundamental contradiction. You are paying for security by handing your network's behavioral fingerprint to a third party.

**Kelan inverts this.** The AI runs on your hardware. The enforcement happens in your kernel. Your keys never leave your machines.

---

## What is AITP?

**AITP — Adaptive Intent Transport Protocol** is the core innovation behind Kelan Security.

Traditional network security asks: *"Is this IP address allowed?"*
Signature-based systems ask: *"Does this packet match a known bad pattern?"*

AITP asks something fundamentally different: **"What is this session trying to do, and should I trust its evolving behavior?"**

```
Traditional:    Client ──► [ALLOW / DENY based on IP or signature] ──► Server

AITP:           Client ──► Intent Classification ──► Trust Scoring ──► [ALLOW / THROTTLE / DENY]
                                                           │
                                              Score evolves over session lifetime
                                              based on behavioral drift detection
```

A session that starts with normal behavior but gradually exhibits anomalous patterns — unusual timing, protocol confusion, behavioral fingerprint drift — sees its trust score drop in real time. Traffic gets throttled, then blocked, before a human analyst even sees an alert.

### Why AITP matters

| Approach | What it catches | What it misses |
|---|---|---|
| IP allowlists | Known bad IPs | Compromised trusted IPs, lateral movement |
| Signature detection | Known attack patterns | Zero-days, slow attacks, behavioral anomalies |
| Cloud ML zero-trust | Unknown patterns | Requires sending your metadata to a vendor |
| **AITP — Kelan** | **Unknown patterns + behavioral drift** | **Nothing leaves your network** |

---

## The stack

### Rust / eBPF enforcement layer

- XDP hook point — earliest possible packet interception, before the kernel TCP stack
- No kernel modules required — just BPF programs attached at the NIC
- Drop rate at near line speed
- `kelan-ebpf/` — BPF programs using the `aya` crate
- `kelan-ebpf-loader/` — userspace loader written in Rust

### Python / AI trust layer

- FastAPI server — AITP protocol implementation
- Ollama integration — local LLM trust evaluation
- `gemma3:latest` model runs entirely on-device
- Zero external API calls — fully air-gap compatible
- SQLite session state via async SQLAlchemy

### Post-quantum cryptography

- **ML-KEM-768** (CRYSTALS-Kyber) for key encapsulation
- **Ed25519** for session authentication signatures
- **X25519** for ephemeral Diffie-Hellman
- Safe against Harvest Now, Decrypt Later attacks
- Every AITP handshake is quantum-resistant today

### Observability

- Trust score timeseries per session
- Behavioral drift detection metrics
- Docker Compose monitoring stack (Grafana + Prometheus)
- Attack simulation suite included in `scripts/`
- `/api/v1/sessions` — real-time session state endpoint

---

## Quick start

```bash
# 1. Clone and install — sets up everything in one command
git clone https://github.com/Kelan-Security/kelan-core.git
cd kelan-core
bash install.sh
```

`install.sh` automatically:
- Detects compatible Python (3.10–3.12)
- Creates `.venv` and installs all requirements
- Checks for Rust, installs via rustup if missing
- Builds all Rust/eBPF components
- Verifies Ollama is installed and the model is available

```bash
# 2. Configure
cp .env.example .env
# Edit .env — set OLLAMA_HOST if Ollama runs on a different machine

# 3. Launch — verifies everything before starting
bash launch.sh

# Stop
bash launch.sh --stop
```

**Requirements:** Python 3.10–3.12 · Rust 1.77+ · Ollama with gemma3 · Linux kernel 5.10+ for eBPF enforcement · macOS works for the AI/server layer

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                           YOUR NETWORK                               │
│                                                                      │
│   Enforcement Node (Linux)            AI / Server Node               │
│   ┌────────────────────────┐         ┌──────────────────────────┐   │
│   │  kelan-ebpf-loader     │         │  FastAPI + AITP Server   │   │
│   │  (Rust, userspace)     │◄───────►│  POST /api/v1/evaluate   │   │
│   │          │             │  HTTP   │  GET  /api/v1/sessions   │   │
│   │   eBPF XDP hook        │         │          │               │   │
│   │  (kernel, no module)   │         │  Ollama  (gemma3)        │   │
│   │          │             │         │  local inference only    │   │
│   │  DROP / PASS / METER   │         │  zero cloud calls        │   │
│   └────────────────────────┘         └──────────────────────────┘   │
│            │                                      │                  │
│     Network interface                      SQLite session DB         │
│     (all traffic)                          Trust score history       │
└──────────────────────────────────────────────────────────────────────┘
```

The enforcement node and AI node can be the same machine or separate. A common self-hosted setup: a Raspberry Pi 5 as the enforcement node on the network edge, with a Mac or Linux desktop running Ollama. Configure via `OLLAMA_HOST` in `.env`.

---

## How an AITP session works

```
1.  CLIENT CONNECTS
    └─► AITP Handshake
          ├─ ML-KEM-768 key encapsulation
          ├─ Ed25519 session authentication
          └─ Session ID assigned, trust_score initialized at 0.5 (neutral)

2.  INTENT CLASSIFICATION  (evaluated every N packets)
    └─► AI evaluator receives:
          ├─ Packet timing fingerprint
          ├─ Protocol behavior signature
          ├─ Session history vector
          └─ Behavioral drift delta (change from established baseline)

3.  TRUST SCORING
    └─► LLM returns:
          ├─ trust_score: 0.0 – 1.0
          ├─ decision: allow / throttle / deny
          └─ reasoning: human-readable explanation stored in session log

4.  ENFORCEMENT  →  eBPF XDP executes immediately
    ├─ score > 0.7   →  PASS    (full bandwidth)
    ├─ score 0.4–0.7 →  METER   (rate-limited)
    └─ score < 0.4   →  DROP    (immediate block)

5.  SCORE EVOLVES OVER SESSION LIFETIME
    └─ Behavioral drift recalculates score in real time
       Normal session that turns malicious → caught before breach
```

---

## Who is this for?

**Self-hosters**
Run it on a Raspberry Pi on your home network. Real AI-powered intrusion detection with zero subscription fees. One `bash launch.sh` and it's running.

**Security teams**
Replace black-box cloud zero-trust with something you can audit, customize, and run in air-gapped environments. The AI trust prompt is fully configurable — tune it for your specific threat model.

**Researchers**
AITP is a new protocol with an IEEE CNS 2026 paper in progress. The implementation is fully open. Build on it, benchmark it, break it, improve it.

---

## Compared to alternatives

| | Kelan AITP | Cloudflare Zero Trust | Zscaler | Palo Alto Prisma |
|---|:---:|:---:|:---:|:---:|
| AI location | Your hardware | Cloudflare cloud | Zscaler cloud | Palo Alto cloud |
| Traffic metadata | Stays on-prem | Sent to vendor | Sent to vendor | Sent to vendor |
| Air-gap compatible | ✅ Yes | ❌ No | ❌ No | ❌ No |
| Post-quantum crypto | ✅ ML-KEM-768 | Partial | ❌ No | ❌ No |
| Open source | ✅ MIT | ❌ No | ❌ No | ❌ No |
| Session-aware trust | ✅ Evolving scores | Static rules | Static rules | Static rules |
| Kernel enforcement | ✅ eBPF / XDP | Network proxy | Network proxy | Network proxy |
| Cost | Infrastructure only | $7 / user / month+ | Enterprise pricing | Enterprise pricing |

> Kelan is the enforcement layer that ensures AI-based trust decisions happen inside your perimeter — not in a vendor's cloud.

---

## Research

Kelan's AITP protocol is the subject of a paper submitted to the **IEEE Conference on Communications and Network Security (CNS) 2026**.

The paper covers:

- Formal specification of the AITP trust scoring model
- Behavioral drift detection methodology
- Post-quantum handshake protocol design
- Empirical evaluation against simulated attack scenarios
- Comparison with signature-based and cloud-ML approaches

Pre-print will be available on arXiv after acceptance. The full implementation is available now in [`kelan-core`](https://github.com/Kelan-Security/kelan-core).

---

## Repositories

| Repository | Description | Status |
|---|---|:---:|
| [`kelan-core`](https://github.com/Kelan-Security/kelan-core) | Main implementation — Rust eBPF + Python AI server | Active |
| [`.github`](https://github.com/Kelan-Security/.github) | Org profile and community health files | Active |

---

## Contributing

We are early stage and actively want contributors. The codebase is clean, the architecture is documented, and PRs are reviewed within 72 hours.

**High-priority areas:**

- WSL2 support — eBPF on Windows Subsystem for Linux
- Additional LLM backends — llama.cpp, vLLM, LM Studio compatibility
- Kubernetes operator — Helm chart for multi-node deployment
- Grafana dashboards — pre-built trust score monitoring panels
- Integration tests — end-to-end AITP protocol test harness

Good first issues are tagged [`good first issue`](https://github.com/Kelan-Security/kelan-core/issues?q=is%3Aopen+label%3A%22good+first+issue%22) in the main repo.

See [`CONTRIBUTING.md`](https://github.com/Kelan-Security/kelan-core/blob/main/CONTRIBUTING.md) for the full guide.

---

## Community

| Channel | Link |
|---|---|
| GitHub Discussions | [kelan-core/discussions](https://github.com/Kelan-Security/kelan-core/discussions) |
| Bug reports | [Open an issue](https://github.com/Kelan-Security/kelan-core/issues/new/choose) |
| Security vulnerabilities | `security@kelan.io` — do not open public issues |
| Twitter / X | [@KelansSecure](https://twitter.com/KelansSecure) |
| LinkedIn | [Kelan Security](https://linkedin.com/company/kelan-security) |

---

## Security policy

Kelan is a security tool. We hold its own security to a high standard.

- All secrets live in `.env` (gitignored) — never hardcoded
- Pre-commit hooks block accidental secret commits
- GitHub Actions run TruffleHog secret scanning on every push
- `cargo audit` and `pip-audit` run weekly via Dependabot
- History verified clean with `git-filter-repo` before public release

**Found a vulnerability?** Email `security@kelan.io`. We respond within 48 hours and credit researchers in release notes.

---

## License

Core: [Apache License 2.0](https://github.com/Kelan-Security/kelan-core/blob/main/LICENSE) — free for personal, commercial, and enterprise use, with patent protection.

Enterprise features (SLA, priority support, advanced dashboards): see [`COMMERCIAL.md`](https://github.com/Kelan-Security/kelan-core/blob/main/docs/COMMERCIAL.md).

---

*Apache 2.0 licensed · Open source core · Research-backed protocol · Built for people who own their infrastructure*

*If Kelan is useful — a ⭐ on [kelan-core](https://github.com/Kelan-Security/kelan-core) directly helps the project reach more developers.*

---

> Copyright 2026 Kelan Security. Owner & Author: Tanush Jain (<https://github.com/Tanush-Jain>)
