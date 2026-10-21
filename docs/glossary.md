# Glossary — every term in the project, explained plainly

Audience: you (the student). Goal: after reading this you can explain any term to your team
and your examiner without hesitation. Each entry is written so you can *say it out loud* in a viva.

---

## 1. The core problem

- **Data-in-use** — data while a CPU is actively computing on it (in RAM), as opposed to
  *data-at-rest* (on disk) or *data-in-transit* (on the network). Encryption protects the
  first two; traditionally **nothing** protects the third, because the OS/hypervisor can read RAM.
- **Confidential computing (CC)** — running computation inside hardware that keeps code and data
  hidden from the privileged software (OS, hypervisor, co-tenants) around it.
- **Trusted Execution Environment (TEE)** — a hardware-isolated region of a CPU where code runs
  and data sits, unreadable even by the host OS/hypervisor. The "trust boundary" is the CPU package.
- **Un trusted infrastructure** — in *our* project, the cloud/partner where the job runs is assumed
  to be at least partially hostile. We do **not** trust the operator.
- **Harvest-now-decrypt-later** — an adversary collects encrypted/ in-use data today and waits for
  future (e.g. quantum) breakthroughs to decrypt. Enclaves limit plaintext lifetime to defeat this.

---

## 2. TEE hardware platforms (the "which chip" question)

- **Intel SGX** — process-level enclaves; smallest TCB; measures code into **MRENCLAVE**.
  Hard to use for whole apps.
- **Intel TDX** — Confidential VM (CVM): protects a whole VM from the hypervisor. Attestation
  borrowed from SGX DCAP; the VM's identity is **TD_MEASUREMENT**.
- **AMD SEV-SNP** — CVM with encrypted VM memory + memory integrity (prevents hypervisor from
  silently swapping pages). Certs: ARK/ASK/VCEK rooted at AMD.
- **AWS Nitro Enclaves** — isolated VM subset on AWS, no persistent storage/network of its own,
  talks to its parent over **vsock**. Attested by the **NSM** (Nitro Security Module). **This is our v0 target.**
- **ARM TrustZone / ARM CCA** — mobile/edge TEEs (TrustZone) and a 4-world model (CCA).
- **NVIDIA H100 CC** — extends the trust boundary into GPU memory for accelerated ML inference.

---

## 3. Attestation cryptography (the "how do we prove it" question)

- **Remote attestation (RA)** — a protocol where a remote party cryptographically confirms *what
  code is running, on what hardware, in what configuration*, before sending it secrets.
- **Attester / Verifier / Relying Party** — the three roles from **IETF RATS (RFC 9334)**.
  The attester produces *evidence*; the verifier *appraises* it into an *attestation result*;
  the relying party (e.g. a KMS) *acts* on it.
- **Quote** — the signed attestation statement an enclave produces. Contains measurements + a nonce.
- **Platform Configuration Register (PCR)** — a PCR is a hash "slot" in the TPM/NSM. You don't
  store a value, you *extend* it: `new = H(old || measurement)`. This makes it a tamper-evident,
  ordered log of what booted/loaded. For Nitro: PCR0=EIF image, PCR1=kernel+initramfs,
  PCR2=user apps, PCR3=host IAM role.
- **Measurement** — the hash (usually SHA-384) of a code/image layer that gets extended into a PCR.
- **Nonce (challenge)** — a random value the verifier sends so the enclave signs *this* request;
  prevents **replay attacks** (an attacker reusing an old quote).
- **MRENCLAVE / TD_MEASUREMENT** — the canonical measurement of the enclave/VM's code.
- **COSE_Sign1** — the CBOR/COSE signature structure Nitro uses to sign its attestation document
  (a single-signer variant of COSE_Sign). We verify an ES384 (ECDSA P-384 + SHA-384) signature.
- **Entity Attestation Token (EAT, RFC 9711)** — a standard CBOR/JSON format for attestation
  evidence/results; the portable alternative to vendor-specific quotes.
- **TCB (Trusted Computing Base)** — everything that must be trusted for the security claim to hold.
  Smaller is better. A measurement also encodes a *TCB level / security version number (SVN)*.

---

## 4. Keys & certificates (the "chain of trust" question)

- **Root of Trust (RoT)** — the anchor you ultimately trust. For Nitro it is Amazon's
  `aws.nitro-enclaves` root cert (valid to 2049).
- **Endorsement Key (EK) / Attestation Identity Key (AIK)** — TPM terms: EK is the hardware-unique
  manufacturer key; AIK is a signing key certified to belong to that TPM.
- **Direct Anonymous Attestation (DAA) / Intel EPID** — privacy schemes so a device can prove it is
  a genuine TPM *without* being linkable across attestations.
- **Certificate chain / cabundle** — the list of intermediate certs from the enclave cert up to the
  root. Verification = walk the chain and check every signature + expiry.
- **PKI (Public Key Infrastructure)** — the system of CAs that issue and vouch for certs.

---

## 5. Posture / "Wiz-style" (the environment-scanning side)

- **Cloud Security Posture Management (CSPM)** — continuously checking cloud configs for
  misconfigurations, excessive permissions, public exposure, etc.
- **CNAPP** — Cloud-Native Application Protection Platform: CSPM + CIEM (identity) + DSPM (data) +
  CWPP (workload) + IaC scanning, in one product. Wiz is the flagship example.
- **Agentless** — Wiz connects via the cloud provider's **control-plane APIs** (read-only) and
  snapshots VMs/disks. No software agent installed in your workloads. This is its key innovation.
- **Security Graph** — Wiz's correlation engine: it models resources, identities, network paths,
  vulnerabilities and data as a graph, then computes **attack paths / blast radius** ("toxic
  combinations" — e.g. internet-exposed VM + over-privileged role + reachable DB).
- **eBPF runtime sensor** — a lightweight kernel hook Wiz uses (Wiz Defend) for real-time threat
  detection without a heavy agent.

---

## 6. Our project's method (Thread A)

- **Static attestation** — the enclave proves its identity *once* at launch. Point-in-time.
- **Continuous / runtime attestation** — re-attesting *over time* (Keylime does this for whole
  nodes via TPM+IMA). We extend the idea to enclaves + their environment.
- **Context-aware attestation** — fusing the enclave's quote with the **environment's posture**
  (exploitability around the enclave) into one trust decision.
- **Trust broker** — our proposed component: sits between the data owner and the untrusted cloud;
  on each key-release it demands a fresh quote **and** a posture attestation, computes a composite
  trust score, and applies a policy (release / release-scoped / defer / revoke).
- **Key-release gating** — only handing decryption keys to an enclave whose measurement + environment
  satisfy policy. The whole point: secrets are never exposed to an unsafe state.

---

## 7. General

- **CBOR** — Concise Binary Object Representation; the binary encoding used by COSE/Nitro docs
  (compact, unambiguous, unlike JSON for crypto).
- **vsock** — a socket-like channel between an enclave and its parent VM on Nitro; the only way
  they talk, and it is itself untrusted.
- **Reproducible build** — building the same source twice yields a byte-identical binary, so the
  measurement hash is auditable (otherwise attestation degrades to "trust the developer").
- **Side channel** — leaking secrets not through crypto but through timing, cache, power, etc.
  (Spectre, Meltdown, SGX-Step). Attestation does **not** prevent these.
