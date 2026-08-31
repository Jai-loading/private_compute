# ChipOath — Attested Private Compute (working title)

> **Status:** Phase −1 (design). This document is the initial problem/solution statement.
> It is intentionally incomplete and will be revised as design decisions are settled and
> code lands. Treat it as a living spec, not a finished report.
>
> Last updated: 2026-08-31 · See [`research_papers.md`](research_papers.md) for the literature base.

---

## 1. Problem statement

Sensitive workloads (health records, payroll, credit data, proprietary model weights)
must increasingly run on infrastructure the data owner does not control — public cloud,
a partner's datacenter, a contractor's cluster. On every mainstream platform today,
**data is protected at rest and in transit, but not in use**: to compute on it, the
plaintext must exist in the memory of a machine whose operator, hypervisor, and
host OS are all outside the data owner's trust boundary.

Concretely, the current failure modes are:

| Failure | Description |
|---|---|
| **Operator visibility** | A cloud administrator, a compromised host OS, or a co-tenant escape can read process memory, core dumps, swap, and register state. Disk/transport encryption is irrelevant once the plaintext is resident. |
| **No binding between code and output** | A consumer of a result has no cryptographic evidence of *which binary* produced it, on *which* hardware, under *which* patch level. Integrity today rests on access-control policy and audit logs — both controlled by the same operator being distrusted. |
| **Sovereignty and cross-border constraints** | GDPR, HIPAA, India's DPDP Act, and sectoral rules restrict processing by untrusted parties. The typical workaround — "don't send the data" — blocks the computation entirely. |
| **Blocked collaboration** | Multi-party analytics (cross-bank fraud rings, cross-hospital research) do not happen because no participant will expose raw inputs to another participant's infrastructure. |
| **Harvest-now-decrypt-later** | Long-lived ciphertext plus future cryptanalytic or key-compromise capability makes bulk collection of in-use data a rational adversary strategy. |

### 1.1 Threat model (v0)

**Adversary (in scope).** A *software* adversary with full control of the host stack:
host OS kernel, hypervisor, container runtime, orchestration layer, cloud control plane,
and any co-resident workload. Capabilities include arbitrary code execution on the host,
memory read/write outside the enclave, traffic interception and replay, and rollback of
persistent state. This is the standard confidential-computing adversary.

**Trust anchor.** The CPU package and the silicon vendor's attestation signing key.
Trust is placed in: (a) the hardware enclave primitive's memory isolation and sealing,
(b) the vendor root certificate used to validate attestation quotes, (c) measurement
correctness — i.e. that the signed measurement corresponds to the code that ran.

**Out of scope (v0, stated explicitly so they are not silently assumed away):**
- Physical attacks: decapsulation, bus probing, DRAM cold-boot, voltage/clock glazing.
- Microarchitectural side channels: speculative-execution (Spectre-class), page-fault and
  cache-line (SGX-Step, Prime+Probe), branch-shadow, and DVFS/thermal channels.
  Attestation does **not** mitigate these; see §7.
- Supply-chain compromise of the enclave source, the compiler, or the signing key.
- Denial of service and resource exhaustion by the host (always available to a host adversary).
- Malicious enclave code itself: an enclave is a *confidentiality* primitive. If the code
  inside is malicious, TEE guarantees do not help — hence measurement + verification policy.

---

## 2. Why existing approaches are insufficient

| Approach | What it gives | Why it is not sufficient alone |
|---|---|---|
| **Encryption at rest / in transit** | Protects storage and wire | Plaintext must be resident in RAM to be computed on |
| **FHE** (Gentry 2009) | Compute on ciphertext, no trusted hardware | 10³–10⁶× slowdown; impractical for general workloads and ML training |
| **MPC / secret sharing** | No single party sees inputs | High round-trip complexity; requires multiple non-colluding parties to be online; hard to generalize |
| **zk-SNARK / STARK** | *Correctness* proof, succinct verification | Proves integrity, not confidentiality; proving cost is minutes–hours for non-trivial circuits; does not hide inputs from the machine running the prover |
| **TEE alone** (SGX / TDX / SEV-SNP / Nitro) | *Confidentiality* of data-in-use + hardware-signed attestation | Attestation is necessary but not consumable: raw quotes are vendor-specific, hard to verify, and say nothing about *policy* or *result freshness* |
| **Vendor CC offerings** (Nitro Enclaves, Azure Confidential VMs, GCP Confidential Computing) | Infrastructure primitives | They expose the primitive, not a verifiable workflow; no portable verification layer, no developer-facing contract |

**Gap this project targets.** The hardware primitive exists and is commoditized; what is
missing is the **verification layer** — a portable, developer-facing service that turns
raw vendor attestation into an enforceable, checkable statement of the form:

> *"This specific, reproducible binary — hash `H` — ran on genuine vendor hardware at TCB
> level `L` or higher, processed input sealed to key `K`, and produced output `R` at time
> `T` in response to challenge `N`."*

---

## 3. Proposed solution

A **confidential compute execution and verification layer**: the workload runs inside a
hardware enclave; the enclave's identity is measured and signed by the CPU; decryption
keys are released *only* to an enclave whose measurement matches an expected value; and
every result is returned with a verifiable attestation bundle that any third party can
check offline against the vendor's public root.

### 3.1 Design invariants

1. **No plaintext outside the enclave boundary.** Inputs are sealed client-side to an
   enclave public key; the corresponding private key never exists outside the enclave.
2. **Key release is measurement-gated.** The KMS releases the data-unwrapping key only
   after successful remote attestation against an expected measurement (MRENCLAVE /
   PCR0–2 / SNP measurement), not to a host or a session.
3. **Freshness by challenge–response.** Every attestation quote binds a verifier-supplied
   nonce, defeating quote replay.
4. **Verifiability without trust in the operator.** Verification requires only the vendor
   root CA and the published measurement — never a call to the compute host.
5. **Reproducible builds are the unit of trust.** The measurement is only meaningful if
   it is reproducible from public source; the build pipeline is therefore part of the TCB.
6. **No durable state by default.** Enclaves are ephemeral; sealing to disk is explicit
   and policy-controlled, with rollback protection where the platform provides it.

### 3.2 Trust decomposition (why TEE + ZK, not either)

The two primitives solve orthogonal halves of the trust problem and are complementary
in this design, not competing:

- **TEE → confidentiality + environment authenticity.** "This ran in a genuine, sealed
  environment that the operator could not observe." Trust anchor is the silicon vendor.
  Runtime overhead ~2–10%; real-time. Failure mode: side-channel leakage of secrets.
- **ZK → execution integrity.** "This output is the correct result of program `P` on
  input `x`." Trust anchor is mathematics (plus, for SNARKs, a trusted setup).
  Verification is milliseconds. Failure mode: circuit bugs / setup compromise.

The composition — run in an enclave for secrecy, emit a proof for verifiability — yields
*private **and** provable*. ZK emission is deferred to a later phase (§6, Phase 5) and is
not required for the core value proposition.

---

## 4. Architecture

```
                    ┌──────────────────────┐
                    │   Verifier / Auditor │
                    │  (offline, no trust  │
                    │   in compute host)   │
                    └─────────▲────────────┘
                              │ 5. attestation bundle
                              │    (quote + cert chain + result)
                              │    verified vs vendor root CA
                              │
  ┌──────────────┐  1. sealed │
  │  Data owner  │  input +   │      ┌────────────────────────────────────┐
  │   (client)   │  nonce ────┼─────▶│        Compute host (untrusted)    │
  └──────────────┘            │      │  host OS · hypervisor · runtime    │
         ▲                    │      │  ┌──────────────────────────────┐  │
         │                    └──────┼─▶│        Enclave (TEE)         │  │
         │ 4. sealed result          │  │  · ephemeral keypair         │  │
         │    + signature            │  │  · RA report / quote         │  │
         │                           │  │  · decrypt → compute → seal  │  │
         │                           │  └──────────▲───────────────────┘  │
         │                           │             │ 3. release data key   │
         │                           │             │    only if measurement│
         │                           │             │    == expected         │
         │                           │      ┌──────┴───────────┐          │
         └───────────────────────────┼──────│  Attestation-    │          │
                                     │      │  gated KMS       │          │
                                     └──────│  (policy engine) │──────────┘
                                            └────────┬─────────┘
                                                     │ 2. fetch + verify quote
                                            ┌────────▼─────────┐
                                            │  Attestation svc │
                                            │  PCCS / Azure    │
                                            │  Attestation /   │
                                            │  Nitro PKI       │
                                            └──────────────────┘

   Trust boundary = the CPU package. Everything left of it is hostile by assumption.
```

### 4.1 Execution protocol (v0)

1. **Enclave launch.** Host instantiates the enclave image; the CPU measures the loaded
   code and produces the platform measurement (SGX `MRENCLAVE`/`MRSIGNER`, Nitro
   `PCR0/PCR1/PCR2`, SEV-SNP guest measurement).
2. **Local key establishment.** The enclave generates an ephemeral P-256/P-384 keypair
   inside the enclave. The **public** key is published; the private key is never
   materialized outside the enclave boundary and is never persisted.
3. **Remote attestation.** The enclave requests a hardware-signed quote over
   `(public_key ‖ verifier_nonce ‖ user_data)`. The quote chains to the vendor root
   (Intel SGX DCAP/EPID → Intel PCCS; Nitro → NSM COSE_Sign1 → AWS Nitro PKI; SEV-SNP →
   AMD ARK/ASK/VCEK).
4. **Measurement-gated key release.** The KMS verifies the quote — signature chain,
   TCB/SVN level against a minimum floor, and measurement equality against the expected
   value — and only then wraps the data-unwrapping key to the enclave public key
   (ECDH-ES + AES-256-GCM or HPKE). A mismatch produces a refusal and an audit event.
5. **Sealed computation.** Client encrypts input to the enclave public key (hybrid:
   ECDH-ES → HKDF-SHA384 → AES-256-GCM). Decryption occurs inside the enclave;
   the computation runs; the result is re-sealed to the client (or to a designated
   output recipient) before it leaves the enclave.
6. **Result attestation.** The enclave signs `(result_digest ‖ job_id ‖ nonce ‖ timestamp)`
   with the attested key and returns the bundle:
   `{ ciphertext, ephemeral_pubkey, quote, cert_chain, enclave_signature }`.
7. **Offline verification.** Any party recomputes: cert chain → vendor root; quote
   signature; `SHA-384(pubkey ‖ nonce)` ∈ quote user-data; measurement == published
   expected value; TCB level ≥ policy floor; signature over the result digest.
8. **TCB freshness (operational, not cryptographic).** Published measurements are
   rotated on rebuild; verification policy pins an explicit measurement or a signer
   identity plus minimum SVN, and rejects stale TCB levels.

### 4.2 Components

| # | Component | Responsibility | Notes / candidate implementation |
|---|---|---|---|
| C1 | **Enclave application** | Execute workload on plaintext inside the TEE; expose a minimal vsock/local-channel RPC surface | Rust; Gramine / Occlum (SGX), or Nitro Enclaves SDK |
| C2 | **Attestation library** | Acquire platform quote; normalize SGX/TDX/SEV-SNP/Nitro quote formats into one internal `Quote` type | `nitro-attestation`, Intel DCAP QVL, `sev` crate |
| C3 | **Verification library** | Offline verification of the bundle; no network dependency beyond fetching the vendor root/collateral | Pure Rust, embeddable; the piece a third party runs |
| C4 | **Attestation-gated KMS** | Policy engine: `expected_measurement`, `min_tcb`, allowed signers; releases keys only on policy match | Standalone service; later: Vault / AWS KMS plugin |
| C5 | **Client SDK + CLI** | Seal input, submit job, verify returned bundle | Rust core + thin CLI; language bindings later |
| C6 | **Build & measurement pipeline** | Reproducible builds → deterministic measurement → signed, published measurement manifest | Nix/Bazel; `reproducible-builds.org` methodology |
| C7 | **Transparency log** *(later)* | Append-only public log of published measurements and job commitments | Trillian / Sigstore Rekor-style |

Component C3 is the architectural center of gravity: it must be small, auditable, and
independently runnable by a party that trusts nobody in the deployment.

---

## 5. Platform selection

| Platform | Isolation unit | Evidence format | Trade-off |
|---|---|---|---|
| **AWS Nitro Enclaves** | Separate VM, no host OS access; dedicated cores/memory | COSE_Sign1 NSM attestation doc, PCR0–3 + cert chain | Simplest operational path (no BIOS/SGX driver work); AWS-only; requires enclave-capable EC2 |
| **Intel SGX** | Process-level enclave, EPC-protected | ECDSA-P256 quote (DCAP) / EPID; `MRENCLAVE`, `MRSIGNER` | Smallest TCB, most literature; ~EPC size limits, driver/BIOS friction on self-hosted hardware |
| **Intel TDX** | VM-level, encrypted memory + integrity | TD quote, `MRTD`, `RTMR` | VM-granular, no app rewrite; larger guest TCB than SGX |
| **AMD SEV-SNP** | VM-level, memory encryption + integrity | SNP attestation report, `VMPL`, `GUEST_POLICY` | Good lift-and-shift story; Azure/GCP availability |

**Decision for v0:** start on **AWS Nitro Enclaves** — the platform with the least
infrastructure friction and a cleanly verifiable COSE document — while keeping C2/C3
platform-agnostic so SGX/TDX/SEV-SNP backends can be added without changing the protocol.
Platform-specific quote parsing is confined to C2; everything above it operates on the
normalized `Quote` type.

---

## 6. Milestones

Each phase has an explicit **exit criterion** — a demonstrable artifact, not elapsed time.

| Phase | Deliverable | Exit criterion |
|---|---|---|
| **0 — Environment** | Enclave-capable host, toolchain, CI skeleton | Enclave image builds and launches reproducibly on a Nitro-capable instance |
| **1 — Hello enclave** | Minimal C1 | Process runs inside the enclave; host cannot read its memory (demonstrated) |
| **2 — Attestation E2E** | C2 + C5 | Hardware-signed quote obtained, bound to an ephemeral key + nonce, verified via attestation service |
| **3 — Sealed computation** | C1 + C4 + C5 | Input encrypted off-platform, decrypted only in-enclave, computed, re-sealed; key release refuses a mismatched measurement |
| **4 — Verification library** | C3 | A third party verifies a full bundle offline with no access to the compute host; negative tests (replayed quote, wrong measurement, stale TCB) all fail closed |
| **5 — ZK augmentation** *(optional)* | Enclave-emitted execution proof | STARK/SNARK proof of correct execution verified outside the enclave |
| **6 — Reproducible measurement pipeline** | C6 | Two independent builds produce an identical measurement; manifest published |
| **7 — Operator console** | UI | Job, measurement, and attestation-health visibility |

---

## 7. Known limitations (stated up front)

- **Attestation ≠ side-channel immunity.** A valid quote says the right code started on
  genuine hardware. It does not prevent Spectre-class, page-fault (SGX-Step), or
  cache-timing extraction of secrets from a running enclave. Mitigations are
  defense-in-depth: constant-time cryptographic code, data-oblivious access patterns,
  enclave-internal randomization, and minimizing secret lifetime in memory. This is an
  active research area, not a solved one.
- **Trust in the silicon vendor is irreducible.** The vendor root CA is the ultimate
  anchor. Multi-vendor verification (requiring two independent roots) is a possible
  future hardening, not a v0 feature.
- **The measurement is only as trustworthy as the build.** Attestation binds to a hash;
  if the build is not reproducible, the hash is unauditable and the guarantee degrades to
  "trust the developer." C6 is therefore required, not optional polish.
- **Host-controlled side effects remain.** I/O, scheduling, and any output the workload
  deliberately emits are observable to the host. Confidentiality covers data in use, not
  traffic shape or timing of I/O.

---

## 8. Non-goals (v0)

- Building a new TEE or a new attestation scheme — the hardware primitive is taken as given.
- Becoming a general-purpose serverless platform; the scope is a single verifiable job-execution path.
- Solving side-channel resistance at the microarchitectural level.
- Post-quantum attestation signatures (relevant future work: ML-DSA migration paths).
- Multi-party computation across mutually distrusting data owners (the current model is
  one data owner, one untrusted operator, many verifiers).

---

## 9. Open design questions

1. **Quote collateral distribution.** Vendor-hosted (PCCS / AWS PKI) vs. self-hosted
   caching collateral — availability vs. reduced external dependency during verification.
2. **Measurement pinning granularity.** Exact `MRENCLAVE`/`PCR0` pin (strictest, breaks on
   every rebuild) vs. signer-identity + minimum-SVN pin (operationally viable,
   weaker). Likely: both, selected per deployment policy.
3. **Key-release topology.** Dedicated KMS service vs. embedding the policy check in the
   client (client-side attestation gating removes a moving part but couples clients to
   verification logic).
4. **ZK backend.** STARK (transparent, larger proofs) vs. Groth16 (succinct, trusted
   setup) — relevant only if Phase 5 is attempted.
5. **State and rollback.** Whether to support durable enclave state at all, and which
   platform-provided monotonic counters to rely on where it is supported.

---

## 10. Glossary

| Term | Meaning |
|---|---|
| **TEE** | Trusted Execution Environment — hardware-isolated execution region |
| **Enclave** | The isolated workload instance inside a TEE |
| **Measurement** | Cryptographic hash of the loaded code/config; the enclave's identity |
| **Attestation quote** | Vendor-signed statement binding a measurement, a public key, and a nonce |
| **Remote attestation (RA)** | The protocol by which a remote party validates a quote |
| **Sealing** | Encrypting data so it can be unsealed only by a specific enclave measurement |
| **TCB** | Trusted Computing Base — everything whose compromise breaks the guarantee |
| **MRENCLAVE / MRSIGNER** | SGX: measurement of enclave code / of its signing key |
| **PCR0–3** | Nitro: measurements of enclave image, kernel, ramdisk, application |
| **MRTD / RTMR** | TDX: initial measurement / runtime-extendable measurement registers |
| **VCEK / ARK / ASK** | AMD SEV-SNP: the chip's attestation key and its vendor cert chain |
| **SVN / TCB level** | Firmware/microcode patch level reported in a quote |

---

## 11. References

Curated literature — foundational (solved, implementable) and open problems — including
the enclave side-channel canon that motivates §7: [`research_papers.md`](research_papers.md).

---

*Initial specification, 2026-08-31. Sections marked "later" or "optional" are scoped out
of the first implementation. This document is expected to change substantially as Phases
0–2 produce empirical constraints.*
