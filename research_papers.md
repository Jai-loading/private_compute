# Research Papers — Private Compute Project

Curated literature for the confidential-computing verification layer ("ChipOath").
Split into **foundational / solved** (the science exists — implement it) and **open problems**
(the hard edge — worth attacking). Papers marked with a ✓ were directly encountered in
2026 web research during project ideation; the rest are well-known canon in the field.

---

## Foundational / solved (implementable)

### Confidential computing & enclaves
- **Intel SGX Explained** — Costan & Devadas, 2016. The definitive architecture explainer for SGX enclaves. Start here.
- **A Formal Foundation for Secure Remote Execution of Enclaves** — Subramanyan et al., CCS 2017. The formal model behind remote attestation.
- **Verifying Computations without Re-executing Them** — Walfish & Blumberg, CACM 2015 ✓. The "don't trust, prove" framing in verifiable computation.

### Zero-knowledge & verifiable computation
- **The Knowledge Complexity of Interactive Proof Systems** — Goldwasser, Micali & Rackoff, 1985. Where ZK was born.
- **On the Size of Pairing-Based Non-interactive Arguments** — Groth, EUROCRYPT 2016 ✓. The Groth16 zk-SNARK — the practical standard.
- **Quadratic Span Programs and Succinct NIZKs without PCPs** — Gennaro, Gentry, Parno & Raykova, EUROCRYPT 2013 ✓. Circuit-to-SNARK pipeline.
- **Scalable, Transparent, and Post-quantum Secure Computational Integrity** — Ben-Sasson et al., 2018 ✓. STARKs — no trusted setup.
- **A Fully Homomorphic Encryption Scheme** — Gentry, Stanford PhD thesis, 2009 ✓. The theoretical ceiling (FHE), usually too slow in practice — relevant contrast to TEEs.

### Foundations
- **New Directions in Cryptography** — Diffie & Hellman, 1976 ✓.
- **A Method for Obtaining Digital Signatures and Public-key Cryptosystems** — Rivest, Shamir & Adleman, 1978 ✓.
- **The Algorithmic Foundations of Differential Privacy** — Dwork & Roth, 2014 ✓ (privacy lens for the analytics use case).

---

## Open problems (the hard edge — where this project can contribute)

### Side-channel attacks on enclaves (why attestation alone isn't enough)
- **Spectre Attacks: Exploiting Speculative Execution** — Kocher et al., S&P 2019 ✓. The class of attacks that broke enclave assumptions.
- **SgxPectre: Stealing Intel Secrets from SGX Enclaves via Speculative Execution** — Chen et al., S&P 2019. Spectre specifically against SGX.
- **SGX-Step: A Practical Attack Framework for Precise Enclave Execution Control** — Van Bulck et al., 2017. Precise step-by-step enclave control — shows the enclave is not a black box.
- **Practical Enclave Malware with Intel SGX** — Schwarz et al., DIMVA 2019. Enclaves hiding malware — why the layer must verify *what* runs, not just *that* an enclave ran.

### AI security & verifiable computation at scale
- **Artificial Intelligence Security and Privacy: A Survey** — He et al., Sci China Inf Sci, 2025 ✓. Concludes there is *no universal defense* — frames the whole landscape.
- **Agentic AI for Autonomous Defense in Software Supply Chain Security** — arXiv 2512.23480 ✓. Agentic + verifiable compute trend (MCP, ledger-audited actions) — a natural extension of this project.

### Prompt injection & agent security (context for the ZK augmentation path)
- **CaMeL: Defeating Prompt Injections by Design** — Debenedetti et al., 2025 ✓.
- **Task Shield: Enforcing Task Alignment Against Indirect Prompt Injection** — Jia et al., ACL 2025 ✓.
- **Awesome-Agentic-Security** (150+ paper index) ✓ — if the project grows toward verified AI agents.

---

## Reading order (if you're a beginner — start light)

1. **Intel SGX Explained** (skim) + the **armored vault analogy** in README.
2. **Verifying Computations without Re-executing Them** — the clearest "why".
3. **Groth16** — the ZK half (optional until Phase 5).
4. **SgxPectre / Spectre** — why "trust the chip" has caveats, and why defense-in-depth matters.

---

## How papers map to project phases

| Paper | Phase |
|---|---|
| Intel SGX Explained / Nitro docs | 0–1 (environment, hello enclave) |
| Formal Foundation of Secure Remote Execution | 2 (attestation model) |
| Groth16 / STARKs | 5 (ZK augmentation) |
| SgxPectre / SGX-Step / Enclave Malware | Threat model + blog-worthy analysis |
| AI Security survey | Positioning the bigger picture |
