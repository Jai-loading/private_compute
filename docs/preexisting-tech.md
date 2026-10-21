# Pre-existing technology — what already exists, and where the gap is

This document maps the landscape so you can position your work honestly. Every claim here is
something an examiner may already know; citing it *first* makes your contribution credible.

---

## 1. TEE platforms at a glance

| Platform | Isolation unit | Attestation model | TCB size | Notes |
|---|---|---|---|---|
| Intel SGX | process/enclave | EPID or ECDSA (DCAP) quote over MRENCLAVE | smallest | Hard to port apps; phased out on consumer CPUs |
| Intel TDX | VM (CVM) | DCAP, TD_MEASUREMENT | medium | Protects whole VM from hypervisor |
| AMD SEV-SNP | VM (CVM) | ARK/ASK/VCEK certs | medium | Encrypted mem + integrity vs hypervisor |
| AWS Nitro Enclaves | VM subset | NSM → COSE_Sign1 doc, AWS Nitro PKI | small-ish | **Our v0 target**; no own network/storage |
| ARM TrustZone | secure world | vendor-specific | varies | Mobile/edge |
| ARM CCA | 4-world | Realm management | small | Armv9-A |
| NVIDIA H100 CC | GPU region | Compute Protected Regions | large | ML inference at scale |

**Takeaway for examiner:** the *hardware primitive* is mature and multi-vendor. The problem is not
"build a TEE" — it is everything *around* the TEE (verification, portability, key release, posture).

---

## 2. Attestation: the real standards

- **IETF RATS (RFC 9334)** defines the architecture we should speak in: **Attester → Verifier →
  Relying Party**, with *evidence* and *attestation results* as the artifacts. **EAT (RFC 9711)**
  is the portable token format. Using these terms shows you know the field's lingua franca.
- **TPM model (Trusted Computing Group):** PCRs extended via `H(old||measurement)`; a *quote*
  signs selected PCRs + nonce with an AIK certified by the manufacturer. Privacy via DAA/EPID.
- **SGX DCAP (Datacenter Attestation Primitives):** moves attestation verification on-prem via a
  PCK cert chain to Intel's root — important because it removes the online dependency on Intel.
- **AWS Nitro attestation:** the NSM measures the enclave into PCRs and emits a CBOR **attestation
  document** signed (COSE_Sign1, ES384) by the AWS Nitro Attestation PKI. Anyone can verify it
  offline against Amazon's published root cert.

---

## 3. Continuous / runtime attestation (prior art we MUST cite)

- **Keylime (CNCF, backed by IBM/Red Hat)** — the dominant open-source continuous-integrity system.
  Uses TPM 2.0 + Linux **IMA** (Integrity Measurement Architecture, extends file hashes into
  PCR 10) for *measured boot* and *continuous* re-attestation of a node. A verifier compares
  quotes against whitelists and alerts on drift. **Limitation (Ruffin et al., DSN 2025):** it
  suffers false positives from OS updates and has real false-negative evasion paths — i.e. it
  watches the *node's own* integrity, not the *environment's* exploitability.
- **5G VNF continuous RA (arXiv 2510.03219, 2025)** — pushes continuous RA down to the Kubernetes
  pod level with policy-driven *remediation* (evict/restart on integrity violation). Shows the
  "attestation → automated response" pattern we borrow.
- **PQ RA in Keylime (ML-DSA-87)** — shows the field is already planning for post-quantum.

**Where these stop:** continuous attestation research is about the *prover's own* integrity over
time. None of it fuses the prover's enclave quote with the *surrounding cloud environment's
exploitability* to make a dynamic key-release decision. That is our lane.

---

## 4. Cloud posture / Wiz (prior art we MUST cite)

- **Wiz (CNAPP)** — agentless: connects via cloud **control-plane APIs** (read-only), snapshots
  disks/VMs, and builds a **Security Graph** correlating identities, network exposure,
  vulnerabilities, permissions and data into ranked **attack paths**. 2800+ config rules, 250+
  compliance frameworks, optional eBPF runtime sensor.
- **The assumption baked in:** Wiz trusts the *cloud provider's control plane* — it is a tool a
  tenant uses to defend *their own* tenancy. It has **zero notion of TEE attestation**.

**Where this stops (and why it can't just be bolted on):** if the *provider itself* is the
adversary (our threat model), the control-plane data Wiz reads is only as trustworthy as that
adversary. Scopelliti et al. (SysTEX'24) show that even commercial CVM attestation can only be
*partially* verified by the owner and "requires trust in the CSP." So we cannot claim "Wiz makes
the environment safe." We use posture as a *conditional signal* in a clearly-scoped threat model.

---

## 5. The gap we are filling (state this verbatim to examiners)

> Confidential computing gives us a hardware-isolated enclave plus a point-in-time attestation
> quote. Continuous-attestation research protects a node's *own* integrity over time. Cloud
> posture tools (Wiz et al.) rank *environmental* risk but ignore attestation entirely. **No
> existing work fuses a TEE attestation quote with surrounding cloud-security posture to make a
> dynamic, context-aware key-release decision on infrastructure the data owner does not fully
> trust.** That fusion — a *trust broker* that gates secret release on (fresh quote AND acceptable
> posture) — is our contribution.

This is defensible because (a) it cites real prior art, (b) it names exactly what each prior
system does *not* do, and (c) it scopes the adversarial assumption honestly.

---

## 6. Supporting industry evidence (for "is the problem real?")

- **IDC 2025 survey (via CENSUS Labs):** 75% of 600+ IT leaders are adopting confidential
  computing (18% in production, 57% piloting); the **top adoption hurdle is validating attestation
  chains (84.5%)** — i.e. the verification/usability problem is real and unsolved at scale.
- **CENSUS Labs (2025):** "Remote attestation is the control plane for confidential computing…
  policy-driven key release, measurement-bound identity, and continuous assurance are what turn a
  TEE into an enforceable security control." This directly supports our thesis.
