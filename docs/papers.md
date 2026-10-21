# Assessed research papers & sources

Format: ⭐ = must-read / central, ★ = supporting, 🔹 = background/industry.
Type notes whether it is peer-reviewed, a workshop, an RFC, or vendor/industry material — be
honest about source quality in your literature review.

---

## ⭐ Core / directly on our gap

1. **Scopelliti, G., Baumann, C., Mühlberg, J. T.** — *Understanding Trust Relationships in
   Cloud-Based Confidential Computing.* SysTEX'24 (Workshop on System Software for Trusted
   Execution), 2024.
   - **What it says:** Analyzes CVM attestation from commercial CSPs; shows many attestation
     steps can only be *partially* verified by the CVM owner and still require *trust in the CSP*
     to deploy secure software and report honestly. "Complete protection from infrastructural
     threats is thus not provided."
   - **Why we cite it:** It is the honest backbone for our threat-model calibration — proves you
     cannot naively trust the provider, which is exactly why posture must be scoped and why we
     fall back to quote-only under a fully malicious provider.

2. **Ruffin, M., Wang, C., Almasi, G., Adebayo, A., Franke, H., Wang, G.** — *Towards Continuous
   Integrity Attestation and Its Challenges in Practice: A Case Study of Keylime.* DSN 2025.
   - **What it says:** Empirical study of Keylime (CNCF continuous attestation). False positives
     from unscheduled OS updates; proposes dynamic policy generation. Finds 5 false-negative
     evasion paths where attackers defeat detection.
   - **Why we cite it:** Keylime is our closest continuous-attestation prior art; this paper gives
     us the *honest limitations* to contrast against and shows the open problem of balancing
     detection vs. false alerts — a sub-problem our trust-score policy must address.

3. **Benedictis et al.** — *TPM-Based Continuous Remote Attestation and Integrity Verification for
   5G VNFs on Kubernetes.* arXiv:2510.03219 (2025).
   - **What it says:** Continuous, pod-level RA with a Trust Monitor, heterogeneous "attestation
     drivers," and **policy-driven remediation** (evict/restart on violation).
   - **Why we cite it:** Demonstrates the "attestation → automated response" pattern and the
     vendor-agnostic verifier idea we reuse; shows continuous RA is an active, current research area.

---

## ★ Supporting (confidential computing landscape)

4. **"When Agents Handle Secrets: A Survey of Confidential Computing for Agentic AI."**
   arXiv:2605.03213 (2026). Survey.
   - TEE taxonomy across SGX/TDX/SEV-SNP/TrustZone/CCA/H100 CC; agent-centric threat model; six
     open challenges incl. *compound attestation for multi-hop chains* and *GPU-TEE overhead*.
   - Use for the "state of the field" section and to show attestation composition is unsolved.

5. **Wilke, L., Scopelliti, G.** — *SNPGuard: Remote Attestation of SEV-SNP VMs Using Open Source
   Tools.* EuroS&P Workshops (EuroSPW), 2024.
   - Practical open-source SEV-SNP attestation tooling; shows the multi-platform attestation
     complexity our C2 normalization layer targets.

6. **CENSUS Labs** — *Operationalizing Confidential Computing: Verifiable Attestation in Practice.*
   2025 (industry/whitepaper, cites IDC 2025 survey).
   - "Attestation is the control plane for confidential computing"; top adoption hurdle = validating
     attestation chains (84.5%). Great industry-evidence citation for "the problem is real."

---

## 🔹 Background / standards / surveys

7. **IETF RATS — RFC 9334** (Architecture) and **RFC 9711** (Entity Attestation Token, EAT).
   - The vocabulary (Attester/Verifier/Relying Party, evidence/result, EAT). Use to frame the
     system in standard terms — signals competence.

8. **Kamhoua et al.** — *Trusted Cloud Computing Architectures for IaaS: Survey and Systematic
   Literature Review.* Computers & Security (ScienceDirect), 2018.
   - Foundational survey: binary vs property-based vs behavior-based RA, TPM/PCA/DAA, vTPM/TVMM.
   - Use for the historical RA taxonomy and to show the field is 20+ years old.

9. **"A Survey on Hardware Approaches for Remote Attestation in Network Infrastructures."**
   arXiv:2005.12453 (2020).
   - Single-platform (TPM, DAA/EPID) vs group attestation; good for the crypto-attestation primer.

10. **Wiz CNAPP / CSPM explainers** (wiz.io; audityour.app, 2026).
    - Agentless model, Security Graph, attack-path/toxic-combination correlation. **Industry, not
      peer-reviewed** — cite as "representative commercial approach," not as a research contribution.

11. **AWS** — *Building secure, verifiable blockchain key management on AWS Nitro Enclaves at
    Turnkey* (AWS Blog) + *Validate a NitroTPM Attestation Document* (AWS Docs).
    - Real-world Nitro attestation + quorum deployment + exact PCR semantics and root-cert
      fingerprint. Primary source for our v0 implementation details.

---

## How to use this list
- Your **literature review** = papers 1–9 (peer-reviewed / RFC), framed as: (a) TEEs exist,
  (b) attestation standards exist, (c) continuous attestation exists for *nodes*, (d) posture
  tools exist but ignore attestation, (e) **nobody fuses them** → our gap.
- Papers 10–11 are **implementation/industry** references — keep them separate from the academic
  claims.
- Before submission, run a novelty search on "attestation + cloud posture + key release" to confirm
  no 2025–2026 paper pre-empts Thread A (we found none as of this writing, but verify).
