# esp-pbft — Documentation Index

> **Status:** 📝 Master index
> **Last updated:** 2026-06-27
> **Audience:** contributors, integrators, reviewers

This is the master index for the esp-pbft design and implementation documentation. Read in order, or jump to a specific topic.

---

## 1. Reading order

```
   ┌──────────────────────────────────────────────────────────┐
   │  Day 1: Project overview                                 │
   │  ── HANDOVER.md         (1-page summary, key decisions)  │
   │  ── DESIGN-AUDIT.md     (issues found in current design) │
   └──────────────────────────────────────────────────────────┘
                              ↓
   ┌──────────────────────────────────────────────────────────┐
   │  Day 2: Architecture                                     │
   │  ── ARCHITECTURE.md     (system overview, layering)      │
   │  ── MEMORY.md           (static memory layout)           │
   │  ── POWER.md            (power state machine)            │
   └──────────────────────────────────────────────────────────┘
                              ↓
   ┌──────────────────────────────────────────────────────────┐
   │  Day 3-4: Protocol                                       │
   │  ── CONSENSUS.md        (3-phase PBFT, state machine)    │
   │  ── VIEW-CHANGE.md      (primary rotation)               │
   │  ── CHECKPOINT.md       (GC + catch-up)                  │
   │  ── PROTOCOL.md         (wire format + transport)        │
   └──────────────────────────────────────────────────────────┘
                              ↓
   ┌──────────────────────────────────────────────────────────┐
   │  Day 5: Crypto                                           │
   │  ── CRYPTO.md           (Pattern Y, key lifecycle)       │
   └──────────────────────────────────────────────────────────┘
                              ↓
   ┌──────────────────────────────────────────────────────────┐
   │  Day 6: Integration                                      │
   │  ── API-REFERENCE.md    (public C API)                   │
   │  ── FAILURE-MODES.md    (Byzantine scenarios)            │
   │  ── DEPLOYMENT.md       (provisioning + OTA)             │
   └──────────────────────────────────────────────────────────┘
                              ↓
   ┌──────────────────────────────────────────────────────────┐
   │  Day 7: Quality                                           │
   │  ── CODING_STANDARD.md  (C style + static analysis)      │
   │  ── TEST-PLAN.md        (test tiers + CI gate)           │
   │  ── ROADMAP.md          (implementation plan)            │
   └──────────────────────────────────────────────────────────┘
```

---

## 2. Document catalogue

| Doc | Size | Purpose | Status |
|-----|------|---------|--------|
| [HANDOVER.md](./HANDOVER.md) | 24 KB | 1-page project overview, key decisions, code structure preview | ✅ Done |
| [DESIGN-AUDIT.md](./DESIGN-AUDIT.md) | ~10 KB | Cross-doc consistency check; 25 issues identified | ✅ Done (2026-06-27) |
| [ARCHITECTURE.md](./ARCHITECTURE.md) | 30 KB | System overview, boot sequence, layering, data flow | ✅ Done |
| [MEMORY.md](./MEMORY.md) | ~15 KB | Per-module memory layout, static_assert catalog, NVS | ✅ Done (v0.1) |
| [POWER.md](./POWER.md) | ~15 KB | Power state machine, per-mode current, battery math, Wi-Fi PS modem | ✅ Done (v0.2) |
| [CONSENSUS.md](./CONSENSUS.md) | 22 KB | 3-phase PBFT state machine, quorum, log management | ✅ Done |
| [VIEW-CHANGE.md](./VIEW-CHANGE.md) | ~25 KB | Primary rotation, V-set/O-set, New-View verification | ✅ Done (v0.1) |
| [CHECKPOINT.md](./CHECKPOINT.md) | ~20 KB | Stable checkpoint, GC, state transfer | ✅ Done (v0.1) |
| [PROTOCOL.md](./PROTOCOL.md) | 20 KB | Wire format, transport abstraction, 7 message types | ✅ Done |
| [CRYPTO.md](./CRYPTO.md) | 17 KB | Pattern Y handshake, Y-5 re-gen, per-message MAC | ✅ Done |
| [API-REFERENCE.md](./API-REFERENCE.md) | ~25 KB | Complete public C API contract | ✅ Done (v0.1) |
| [FAILURE-MODES.md](./FAILURE-MODES.md) | ~20 KB | Byzantine scenarios, network failures, recovery | ✅ Done (v0.1) |
| [DEPLOYMENT.md](./DEPLOYMENT.md) | ~18 KB | Provisioning, NVS image, OTA, Y-5 runbook | ✅ Done (v0.1) |
| [CODING_STANDARD.md](./CODING_STANDARD.md) | 48 KB | Zephyr MISRA-C subset + CERT C + ESP-IDF style | ✅ Done |
| [TEST-PLAN.md](./TEST-PLAN.md) | ~18 KB | Unit / integration / Byzantine / perf / soak tests | ✅ Done (v0.1) |
| [ROADMAP.md](./ROADMAP.md) | ~12 KB | Phased implementation plan (P1-P5 + P6) | ✅ Done (v0.1) |
| [INDEX.md](./INDEX.md) | (this) | Master index | ✅ Done |

**Total: ~338 KB of design documentation** across 16 documents.

---

## 3. Topic index

### 3.1 PBFT protocol

- 3-phase state machine → [CONSENSUS.md](./CONSENSUS.md) §3-4
- Quorum math (n=7, f=2, quorum=5) → [CONSENSUS.md](./CONSENSUS.md) §7
- Pre-Prepare / Prepare / Commit handlers → [CONSENSUS.md](./CONSENSUS.md) §4.2-4.4
- View-change (primary rotation) → [VIEW-CHANGE.md](./VIEW-CHANGE.md)
- Checkpoint / GC → [CHECKPOINT.md](./CHECKPOINT.md)
- Catch-up / state transfer → [CHECKPOINT.md](./CHECKPOINT.md) §8

### 3.2 Crypto (Pattern Y)

- Key generation → [CRYPTO.md](./CRYPTO.md) §4
- Hello exchange + TOFU → [CRYPTO.md](./CRYPTO.md) §6
- ECDH + HMAC key derivation → [CRYPTO.md](./CRYPTO.md) §4.1
- Per-message MAC → [CRYPTO.md](./CRYPTO.md) §5
- Y-5 periodic re-gen → [CRYPTO.md](./CRYPTO.md) §7
- Threat model → [CRYPTO.md](./CRYPTO.md) §10 + [FAILURE-MODES.md](./FAILURE-MODES.md) §5

### 3.3 Transport

- ESP-NOW vs Wi-Fi UDP comparison → [PROTOCOL.md](./PROTOCOL.md) §3
- ESP-NOW impl → [PROTOCOL.md](./PROTOCOL.md) §2.2
- Wi-Fi UDP impl + power-save → [PROTOCOL.md](./PROTOCOL.md) §2.3 + [POWER.md](./POWER.md) §12
- ESP-NOW 250 B size limit + UDP fallback → [PROTOCOL.md](./PROTOCOL.md) §8
- Wire format / byte layout → [PROTOCOL.md](./PROTOCOL.md) §4-6
- MAC binding (view+seq+type+digest+payload) → [CRYPTO.md](./CRYPTO.md) §5.1

### 3.4 Memory

- Static allocation principle → [ARCHITECTURE.md](./ARCHITECTURE.md) §11
- Per-module layout → [MEMORY.md](./MEMORY.md) §2
- ESP-NOW build budget (~117 KB) → [MEMORY.md](./MEMORY.md) §3.1
- Wi-Fi UDP build budget (~177 KB) → [MEMORY.md](./MEMORY.md) §3.2
- NVS layout → [MEMORY.md](./MEMORY.md) §4
- Static_assert catalog → [MEMORY.md](./MEMORY.md) §8

### 3.5 Power

- Power state machine → [POWER.md](./POWER.md) §2
- Per-mode current → [POWER.md](./POWER.md) §3
- Battery-life math → [POWER.md](./POWER.md) §4
- Light-sleep wake sources → [POWER.md](./POWER.md) §2.2
- Wi-Fi PS modem mode → [POWER.md](./POWER.md) §11
- Boot staggering → [POWER.md](./POWER.md) §5.2

### 3.6 API

- Lifecycle (`pbft_init` etc.) → [API-REFERENCE.md](./API-REFERENCE.md) §2.3
- Error codes → [API-REFERENCE.md](./API-REFERENCE.md) §3
- `pbft_submit` + tx flags → [API-REFERENCE.md](./API-REFERENCE.md) §4
- Commit callback → [API-REFERENCE.md](./API-REFERENCE.md) §5
- State inspection → [API-REFERENCE.md](./API-REFERENCE.md) §7
- Metrics → [API-REFERENCE.md](./API-REFERENCE.md) §9
- Thread-safety → [API-REFERENCE.md](./API-REFERENCE.md) §8
- Multi-primary forwarding (v2) → [API-REFERENCE.md](./API-REFERENCE.md) §10

### 3.7 Failure modes

- Threat model → [FAILURE-MODES.md](./FAILURE-MODES.md) §1
- Primary misbehavior → [FAILURE-MODES.md](./FAILURE-MODES.md) §2.1
- Replica misbehavior → [FAILURE-MODES.md](./FAILURE-MODES.md) §2.2
- Combined Byzantine → [FAILURE-MODES.md](./FAILURE-MODES.md) §2.3
- Network partitions → [FAILURE-MODES.md](./FAILURE-MODES.md) §3.3
- Retransmit policy → [FAILURE-MODES.md](./FAILURE-MODES.md) §3.4
- Recovery playbook → [FAILURE-MODES.md](./FAILURE-MODES.md) §9

### 3.8 Testing

- Test tiers (T1-T7) → [TEST-PLAN.md](./TEST-PLAN.md) §1
- Unit tests → [TEST-PLAN.md](./TEST-PLAN.md) §2
- Host integration → [TEST-PLAN.md](./TEST-PLAN.md) §3
- Cluster integration → [TEST-PLAN.md](./TEST-PLAN.md) §5
- Byzantine injection → [TEST-PLAN.md](./TEST-PLAN.md) §6
- 24-hour soak → [TEST-PLAN.md](./TEST-PLAN.md) §7
- Performance benchmarks → [TEST-PLAN.md](./TEST-PLAN.md) §8
- CI gate → [TEST-PLAN.md](./TEST-PLAN.md) §9

### 3.9 Deployment

- NVS image baking → [DEPLOYMENT.md](./DEPLOYMENT.md) §2.1
- Per-device `my_node_id` → [DEPLOYMENT.md](./DEPLOYMENT.md) §2.2
- Physical-security requirement (TOFU) → [DEPLOYMENT.md](./DEPLOYMENT.md) §3.1
- Power-on sequence → [DEPLOYMENT.md](./DEPLOYMENT.md) §3.2
- OTA strategy → [DEPLOYMENT.md](./DEPLOYMENT.md) §5
- Y-5 runbook → [DEPLOYMENT.md](./DEPLOYMENT.md) §6
- Production topology → [DEPLOYMENT.md](./DEPLOYMENT.md) §7

### 3.10 Implementation

- Phase plan (P1-P5) → [ROADMAP.md](./ROADMAP.md) §2-6
- Coding standards → [CODING_STANDARD.md](./CODING_STANDARD.md) §0-15
- Anti-patterns → [CODING_STANDARD.md](./CODING_STANDARD.md) §B (banned patterns)
- Compiler flags → [CODING_STANDARD.md](./CODING_STANDARD.md) §16

---

## 4. Glossary

| Term | Meaning | Reference |
|------|---------|-----------|
| **Byzantine** | Arbitrary / malicious failure mode | [FAILURE-MODES.md §1](./FAILURE-MODES.md) |
| **Checkpoint** | Stable snapshot at `K × n` sequences | [CHECKPOINT.md §1](./CHECKPOINT.md) |
| **Cluster** | The 7 PBFT nodes | [HANDOVER.md §2.1](./HANDOVER.md) |
| **Commit** | Replica confirms it has reached PREPARED | [CONSENSUS.md §3](./CONSENSUS.md) |
| **ECDH** | Elliptic-curve Diffie-Hellman key agreement | [CRYPTO.md §4.1](./CRYPTO.md) |
| **ECDSA** | Elliptic-curve DSA (used only for keygen, not runtime signing) | [CRYPTO.md §3](./CRYPTO.md) |
| **ESP-NOW** | Espressif connectionless Wi-Fi protocol | [PROTOCOL.md §2.2](./PROTOCOL.md) |
| **HMAC** | Hash-based Message Authentication Code (RFC 2104) | [CRYPTO.md §5](./CRYPTO.md) |
| **HKDF** | HMAC-based Key Derivation Function (not used; SHA-256 substituted) | [CRYPTO.md §4.2](./CRYPTO.md) |
| **Kconfig** | ESP-IDF compile-time configuration system | [API-REFERENCE.md §11](./API-REFERENCE.md) |
| **NVS** | Non-Volatile Storage (ESP-IDF key-value store) | [MEMORY.md §4](./MEMORY.md) |
| **O-set** | Set of Pre-Prepares to re-propose after view-change | [VIEW-CHANGE.md §6](./VIEW-CHANGE.md) |
| **Pattern Y** | Self-gen TRNG + ECDH-boot + HMAC-runtime auth scheme | [HANDOVER.md §3.5](./HANDOVER.md) |
| **PBFT** | Practical Byzantine Fault Tolerance (Castro & Liskov 1999) | [HANDOVER.md §3.1](./HANDOVER.md) |
| **Pre-Prepare** | Phase 1 — primary assigns sequence & broadcasts request | [CONSENSUS.md §3](./CONSENSUS.md) |
| **Prepare** | Phase 2 — replicas echo agreement on (view, seq, digest) | [CONSENSUS.md §3](./CONSENSUS.md) |
| **Prepared** | State after observing 2f+1 matching Prepare certificates | [CONSENSUS.md §3](./CONSENSUS.md) |
| **Primary** | The node currently proposing requests (rotates per view) | [CONSENSUS.md §2](./CONSENSUS.md) |
| **PSA Crypto** | Platform Security Architecture Crypto API (Arm standard) | [CRYPTO.md §3](./CRYPTO.md) |
| **Quorum** | 2f+1 matching votes required to advance state | [CONSENSUS.md §7](./CONSENSUS.md) |
| **Replica** | All non-primary nodes in current view | [CONSENSUS.md §2](./CONSENSUS.md) |
| **Sequence** | Monotonic TX order position assigned by primary | [CONSENSUS.md §2](./CONSENSUS.md) |
| **STS** | Station-to-Station protocol (not used; TOFU substituted) | — |
| **TOFU** | Trust On First Use | [CRYPTO.md §6.2](./CRYPTO.md) |
| **TRNG** | True Random Number Generator (hardware) | [CRYPTO.md §4.1](./CRYPTO.md) |
| **V-set** | Set of VIEW-CHANGE messages proving a view-change | [VIEW-CHANGE.md §4.1](./VIEW-CHANGE.md) |
| **View** | Configuration number; changes when primary rotates | [CONSENSUS.md §2](./CONSENSUS.md) |
| **View-Change** | Protocol to rotate primary on suspected failure | [VIEW-CHANGE.md §1](./VIEW-CHANGE.md) |
| **Wi-Fi PS modem** | Power-save mode that sleeps between beacons/DTIMs | [POWER.md §12](./POWER.md) |
| **Y-5** | Pattern Y variant with periodic key re-gen (every 24 h) | [CRYPTO.md §7](./CRYPTO.md) |

---

## 5. Audit status

The [DESIGN-AUDIT.md](./DESIGN-AUDIT.md) document identified 25 issues:

| Severity | Count | Status |
|----------|-------|--------|
| 🔴 Protocol-breaking (P0) | 7 | All addressed in new docs (VIEW-CHANGE, CHECKPOINT, MEMORY) |
| 🟠 Behavior-changing (P1) | 11 | Documented; mitigations in place; some deferred to v2 |
| 🟡 Documentation/clarity (P2) | 14 | All addressed in new docs |
| 🟢 Nits | 8 | Style cleanups; will be applied during implementation |

See [DESIGN-AUDIT.md §F](./DESIGN-AUDIT.md) for the issue list.

---

## 6. Conventions

### 6.1 Status icons used in docs

- 🟡 Draft — work in progress
- ✅ Done — reviewed and approved
- 📝 Drafting — actively being written
- 📅 Planned — scheduled but not started
- 📅 Final — written last

### 6.2 Severity icons (DESIGN-AUDIT)

- 🔴 Protocol-breaking
- 🟠 Behavior-changing
- 🟡 Documentation/clarity
- 🟢 Nit

### 6.3 Document version

Each document ends with its version string, e.g., "v0.1 draft". Major docs version in sync with `PBFT_VERSION_*` macros (API-REFERENCE §1).

### 6.4 Cross-references

All cross-references use relative paths (`./CONSENSUS.md`) so docs can be browsed on GitHub without the index file.

---

## 7. References

### 7.1 External

- **Castro & Liskov, "Practical Byzantine Fault Tolerance"** (OSDI 1999) — http://pmg.csail.mit.edu/papers/osdi99.pdf
- **NIST SP 800-56A Rev. 3** — ECDH key agreement
- **NIST FIPS 186-4** — ECDSA
- **NIST SP 800-107** — HMAC security
- **NIST SP 800-22** — TRNG tests (ESP32-C3 TRNG compliance)
- **RFC 2104** — HMAC
- **RFC 5869** — HKDF (not used)
- **RFC 768** — UDP
- **RFC 1112** — IP multicast / IGMP
- **PSA Crypto API spec** — https://arm-software.github.io/psa-api/crypto/
- **ESP-IDF v6.0.1 docs** — https://docs.espressif.com/projects/esp-idf/en/v6.0.1/
- **ESP-NOW API** — https://docs.espressif.com/projects/esp-idf/en/v6.0.1/esp32c3/api-reference/network/esp-now.html
- **Mbed TLS 4.0** (bundled with ESP-IDF v6.0.1) — https://github.com/Mbed-TLS/mbedtls

### 7.2 Related projects

- **Hyperledger Fabric Orderer** — production PBFT with ECDSA identity
- **BFT-SMaRt** — Java PBFT reference implementation
- **Tendermint** — PBFT-like BFT consensus (different trade-offs)

---

## 8. Contributing

### 8.1 Edit process

1. Open a PR with doc changes
2. Reference the design-audit issue if applicable
3. At least 1 reviewer required for content, 1 for typos
4. CI must pass (`./tools/check_docs.sh` validates cross-references)

### 8.2 Doc style

- ASCII diagrams preferred over images (renders in any terminal)
- Code blocks with language tags (```c, ```kconfig, etc.)
- Cross-references use relative paths
- Section numbering: `§N.M` (e.g., `§3.2` for CRYPTO key derivation)
- Tables for structured comparisons
- Avoid emojis in technical prose (keep for headers/status)

### 8.3 Naming

- Files: `SCREAMING_SNAKE_CASE.md`
- Identifiers in code: `snake_case` (per CODING_STANDARD.md)
- Type names end in `_t`
- Constants in `UPPER_SNAKE_CASE` with `PBFT_` prefix

---

**End of INDEX.md (v0.1 — master index for esp-pbft docs)**