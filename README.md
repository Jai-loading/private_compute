# private_compute — Living Project Doc & Progress Tracker

> Evolved from the original `ChipOath` design spec (Phase −1, README-only) into a scoped
> **degree major-project research build**. This repo is now the working notebook: decisions,
> explainers, assessed papers, and progress. Treat `docs/` as the knowledge base.

---

## 0. What this project is (one paragraph)

We are building a **context-aware continuous attestation layer** for confidential computing on
infrastructure the data owner does *not* trust. Today a hardware enclave (e.g. AWS Nitro) can
cryptographically prove *which code* is running on *genuine hardware* at launch — but that proof
is point-in-time and ignores whether the *environment around* the enclave is exploitable. We add a
**trust broker** that, on every key-release, demands *both* a fresh enclave attestation quote *and*
a posture attestation of the surrounding cloud environment, computes a composite trust score, and
releases (or scopes/defers/refuses) decryption keys accordingly. Outcome target: **a working
prototype + a research paper and/or patent**.

---

## 1. Chosen thread (decided 2026-10-21)

**Thread A — Context-aware continuous attestation** (fusing TEE quote + cloud posture for dynamic
key-release). Selected over B (portable verifier only), C (confidential ML), D (measurement
transparency log), E (side-channel tooling), F (key custody) because it is the most novel and
defensible, and it directly incorporates the user's Wiz/env-scanning idea *without* contradicting
the untrusted-provider threat model. See `docs/preexisting-tech.md §5` for the exact gap statement.

---

## 2. Elevator pitch (memorize this for the examiner)

> "Confidential computing lets a sensitive job run inside a hardware enclave on someone else's
> cloud, and the enclave can prove its identity with a cryptographic quote. But that quote is a
> snapshot at launch and says nothing about whether the *environment around* the enclave — its
> network exposure, vulnerabilities, over-privileged roles — can be used to steal the data anyway.
> Meanwhile, cloud posture tools like Wiz rank that environmental risk but completely ignore
> attestation. My work fuses the two: a trust broker that only releases decryption keys when the
> enclave quote *and* the environment posture both pass policy, continuously and fail-closed. The
> contribution is the fusion method and the honest threat-model calibration for partially-trusted
> infrastructure."

---

## 3. Explainer map (read in this order)

| Doc | Purpose | Read when… |
|---|---|---|
| [`docs/glossary.md`](docs/glossary.md) | Every term, plain-English, viva-ready | you need to explain any word |
| [`docs/preexisting-tech.md`](docs/preexisting-tech.md) | TEEs, attestation standards, Keylime, Wiz, the gap | positioning vs prior art |
| [`docs/low-level.md`](docs/low-level.md) | The actual protocol, byte-level, our addition | "how exactly does it work?" |
| [`docs/papers.md`](docs/papers.md) | Assessed papers + honest source-quality notes | writing the literature review |

---

## 4. Decisions log

| Date | Decision | Rationale |
|---|---|---|
| 2026-08-31 | Original `ChipOath` spec written (design only, no code) | Problem framing |
| 2026-10-21 | Confirmed problem is real; uniqueness claim needs calibration vs Veraison/MarbleRun/Wiz | Honesty for thesis |
| 2026-10-21 | Added Wiz/env-scanning idea → reframed as continuous context-aware attestation (not "bolt Wiz on") | Avoids threat-model contradiction |
| 2026-10-21 | Chose Thread A as the degree project; scoped to trust broker + verifier + negative tests | Buildable on laptop, defensible |
| 2026-10-21 | Created glossary + preexisting-tech + low-level + papers docs | So user can explain to team/examiner |

---

## 5. Progress tracker

| # | Work item | Status | Owner | Notes |
|---|---|---|---|---|
| 1 | Research TEE & attestation fundamentals | ✅ done | — | SGX/TDX/SEV/Nitro, RATS, COSE_Sign1 |
| 2 | Research continuous attestation & posture | ✅ done | — | Keylime, Wiz, 5G VNF paper |
| 3 | Read & assess research papers | ✅ done | — | 11 sources in `docs/papers.md` |
| 4 | Write glossary + explainers | ✅ done | — | 4 docs under `docs/` |
| 5 | **Write project proposal / scope doc** | ⬜ next | — | Problem, prior art, method, milestones, evaluation |
| 6 | Draft patent-claim sketch (method claims) | ⬜ | — | After proposal |
| 7 | Scaffold verifier (Rust) + Nitro doc parser | ⬜ | — | Hardware-light, use sample docs |
| 8 | Implement trust broker + posture signal | ⬜ | — | Core contribution |
| 9 | Negative-test suite (fail-closed) | ⬜ | — | Replay / wrong measurement / stale TCB / toxic combo |
| 10 | Evaluation: latency + detection benchmarks | ⬜ | — | Comparison vs naive gate |
| 11 | Novelty / patent search confirmation | ⬜ | — | Verify no 2025–26 pre-emption |

---

## 6. Open questions (to resolve with advisor)

1. **Degree level?** BSc / MSc / PhD changes scope & originality bar.
2. **Timeline & cloud budget?** Real Nitro enclaves need paid EC2; prototype can be laptop-only.
3. **Patent vs paper vs both?** Patent needs a novelty search + method claims; not a lawyer here.
4. **Measurement granularity** (exact PCR pin vs signer+SVN pin) — operational trade-off.
5. **Posture source in prototype** — real Wiz API (needs account) vs synthetic emitter we control.

---

## 7. Explaining to your team & examiner — FAQ

**Q: Isn't this just Wiz?**<br>
A: No. Wiz ranks *environment* risk but has zero notion of TEE attestation, and it trusts the
provider's control plane. We fuse Wiz-style posture *with* an enclave quote into one key-release
decision, and we scope the posture signal honestly for a partially-trusted provider.

**Q: Why not just use the enclave and ignore the environment?**<br>
A: The quote is a snapshot at launch. If a critical, network-exposed vulnerability later opens a
path to the enclave's memory, the static quote is blind. Continuous, context-aware gating catches that.

**Q: What exactly is your novel contribution?**<br>
A: The *method* — composing a TEE attestation quote with surrounding cloud-security posture into a
single dynamic, fail-closed key-release decision on untrusted infrastructure, with an explicit
degradation path when the provider is fully malicious.

**Q: What about side channels (Spectre, SGX-Step)?**<br>
A: Attestation does not prevent them; that is a known, stated limitation. Our trust boundary is the
silicon vendor + defense-in-depth (constant-time code, data-oblivious access). This is the current
state of the art, not a solved problem, and we say so.

**Q: What if the cloud provider is fully malicious?**<br>
A: Then the posture channel (which rides the provider's control plane) is untrustworthy, so we
explicitly fall back to quote-only attestation. Stating this is a strength, not a weakness.

**Q: How do you prove it works?**<br>
A: A negative-test suite where replayed quotes, wrong measurements, stale TCB, and toxic-combo
posture *all* fail closed, plus latency benchmarks and a comparison against a naive single-quote gate.

---

## 8. Next step (recommended)

Write the **proposal/scope doc** (item 5): problem statement, honest prior-art survey, the trust-broker
method, scoped milestones (verifier → broker → tests → eval), and an evaluation plan. Then a
patent-claim sketch. Say the word and I'll draft it.
