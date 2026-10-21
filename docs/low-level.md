# How it works at the low level — so you can answer "how exactly?"

This is the document to study before your viva. It walks the real protocol end-to-end using
**AWS Nitro Enclaves** (our v0 target) and then shows **our Thread-A addition**.

---

## A. The baseline: a single attested, sealed computation (no posture yet)

### Step 0 — Build (reproducibly)
The enclave app is compiled into an **EIF** (Enclave Image File). Because builds are reproducible,
anyone can rebuild it and get the *same* PCR hash — the measurement is auditable, not "trust us."

### Step 1 — Launch & measure
When the enclave starts, the **NSM (Nitro Security Module)** measures the layers into PCRs:
- **PCR0** = hash of the EIF (your app + OS)
- **PCR1** = hash of kernel + initial RAM disk
- **PCR2** = hash of user applications
- **PCR3** = hash of the parent host's IAM role

### Step 2 — Ephemeral key + challenge
The enclave generates an ephemeral keypair **whose private key never leaves the enclave**. The
verifier/KMS sends a random **nonce** (challenge) to prevent replay.

### Step 3 — Quote (the attestation document)
The NSM produces a **CBOR-encoded, COSE_Sign1-signed attestation document**. Its payload contains:
`module_id, timestamp, digest (SHA-384 of PCRs), pcrs, certificate (enclave pubkey), cabundle
(chain to AWS root), public_key, user_data, nonce`.

The signature is **ES384** (ECDSA over P-384 with SHA-384).

### Step 4 — Verify offline (anyone, anywhere)
To verify, a verifier:
1. Decodes CBOR → COSE_Sign1 structure.
2. Reconstructs the signed bytes: `Sig_structure = ["Signature1", protected_header, external_aad="", payload]`.
3. Hashes that with SHA-384.
4. Verifies the 96-byte ES384 signature using the **device certificate**'s public key (from the payload).
5. Walks `certificate → cabundle → AWS Nitro root` (downloaded once, fingerprint
   `64:1A:03:21…BB:5B`, valid to 2049); checks every signature and every expiry.
6. Checks `nonce` matches (freshness) and `timestamp` is recent (anti-replay / stale-TCB).

### Step 5 — Key-release gating (the payoff)
Only after the quote verifies does the **KMS** compare the PCR values against the *expected*
measurement pinned in the key policy (e.g. `kms:RecipientAttestation:PCR0` must equal the known
good hash). **If and only if** the measurement matches does KMS decrypt and release the key — to
the enclave's attested public key, never to the host.

### Step 6 — Sealed compute
The enclave decrypts the input, computes, re-seals the output, signs the result with its attested
key. A third party later verifies the whole bundle offline.

> **One-line summary you can say:** "The CPU signs a statement that this exact code is running on
> genuine hardware; the KMS only releases the key if that signed statement matches policy. Nothing
> outside the enclave ever sees the plaintext or the key."

---

## B. Where baseline attestation is weak (our motivation)

1. **Point-in-time.** The quote proves the state *at launch*. If the host is later compromised or a
   critical CVE opens an attack path to the enclave's memory, the static quote says nothing.
2. **No environmental context.** The quote proves the *enclave's* identity; it says nothing about
   whether the *surrounding VM/network/IAM* is exploitable.
3. **Provider trust.** Per Scopelliti et al. (SysTEX'24), CVM attestation is only partially
   verifiable by the owner and still requires trusting the CSP to report honestly.

---

## C. Our Thread-A addition: context-aware continuous attestation

### New component — the **Trust Broker**
Sits between the data owner (relying party) and the untrusted cloud. On **every key-release** (and
periodically thereafter), it requires **two** fresh pieces of evidence:

1. **Enclave quote** (as above) — proves *code identity + TCB level + freshness*.
2. **Posture attestation** — a signed statement about the *environment around* the enclave:
   network exposure, vulnerabilities with known exploits, IAM blast-radius, co-tenant risk,
   egress paths. (Sourced from a Wiz-style, agentless control-plane scan; or, for our prototype,
   a synthetic/local posture emitter we control.)

### Composite trust decision
The broker computes a **trust score** and applies policy:

| Enclave quote | Environment posture | Decision |
|---|---|---|
| valid, expected measurement | clean / low risk | **release** |
| valid, expected measurement | critical exploitable exposure | **release-scoped** or **defer** (e.g. restrict output, require re-attest sooner) |
| valid but stale TCB | any | **defer / revoke** |
| wrong measurement | any | **refuse** (fail closed) |
| unverifiable quote | any | **refuse** (fail closed) |

This is the **novel method**: attestation is no longer a one-shot gate; it is a *continuous,
context-weighted* gate. That is the contribution a paper/patent can claim.

### Honest threat-model calibration (say this proactively)
- We scope the posture signal to a **honest-but-curious / tenant-scoped-compromise** provider: the
  provider won't forge attestation, but a misconfiguration, a co-tenant, or a vulnerable adjacent
  workload might. In that regime posture is meaningful.
- If the **provider is fully malicious**, the posture channel is untrustworthy (it controls the
  control plane). We explicitly *fall back to quote-only* and say so. This is not a weakness — it
  is the correct, honest statement of the trust boundary, and examiners respect it.

---

## D. How we answer "how are you filling the gap?" (cheat sheet)

- *What exists?* TEEs (SGX/Nitro/SEV) give isolation + a one-shot quote. Keylime gives
  continuous *node* integrity. Wiz gives *environment* risk — but ignores attestation.
- *What's missing?* Something that fuses the enclave quote **with** environment posture into a
  **single dynamic key-release decision** on infra the owner doesn't trust.
- *What do we build?* The **trust broker** + a portable verification path + the posture signal,
  with fail-closed negative tests (replayed quote, wrong measurement, stale TCB, toxic-combo
  posture all refuse the key).
- *Why is it hard / novel?* Cross-signal trust composition, freshness across two evidence types,
  and an honest degradation path when the provider is adversarial.

---

## E. Implementation reality check (for scoping)
- The **verifier + trust broker + negative-test suite** can be built in pure Rust on a laptop using
  the documented Nitro attestation-doc format and **published sample documents / test vectors** —
  no paid hardware required for the core.
- A **real** enclave (C1) and live cloud posture need Nitro-capable EC2 (paid) — treat as stretch.
- Reproducible builds (C6) are required for the measurement to be auditable; without them the
  guarantee collapses to "trust the developer."
