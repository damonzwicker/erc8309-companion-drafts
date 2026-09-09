# Vantage-Authority Resolution Profiles for ERC-8309
## Companion Specification — Draft v0.2

**Status:** Draft v0.2 — supersedes v0.1. Review round complete; ratification record in §13.
**Author:** Damon Zwicker.
**Contributors (see §12):** Tiago Merlini, Jimmy Shi, Pavlo, Fede.
**Base:** ERC-8309 as amended (ethereum/ERCs#1826); reference implementation trustless-ai/ccip-router#3 (divergence preservation, `getRecordState`, `resolveDivergence`) and #4 (ATTEST-RETAIN, `getAttestations`, `getSignedAttestations`).
**Cut:** 2026-08-23.

---

## 1. Abstract

ERC-8309, as amended, preserves signed evidence and exposes genuine multi-vantage divergence as a first-class store state — `single | divergent(committed observation set) | absent` — and names an extension point (`resolveDivergence`) where a declared resolution policy MAY attach. It deliberately prescribes none.

This companion defines what attaches there: a common **resolution envelope** every policy must fill, a normative **output-state taxonomy**, a **default profile that resolves nothing**, and three opt-in profiles (quorum, registry-weighted, declared priority) expressed as declarations over established consensus models rather than new consensus machinery. No profile establishes external truth; each establishes at most agreement or authority *under its declared assumptions*, which travel with every verdict.

## 2. The Three Refusals (design principles)

1. **Evidence is not agreement.** The base's `divergent` state is preserved signed evidence, not a verdict.
2. **Resolution is not agreement.** A resolved divergence is a distinct top-level state — `resolved(policy, set, conditions)` — never folded into `agreement`.
3. **Judgment is not execution.** A signed resolution verdict is not the claim that any downstream action occurred as approved; execution is a separate, later fact, checked separately.

A conformant implementation makes the collapsed forms *unrepresentable*, not merely discouraged.

## 3. Terminology and fault taxonomy

**Observation vs. attestation:** an *observation* is identified by `(inputHash, namespace, value)`; an *attestation* is a distinct signed message carrying an observation plus attestation metadata. Observation identity governs the divergence taxonomy; attestation identity and retention are separate concerns (§4.2).

**A signed observation is not a consensus vote.** Honest gateways may produce different observations because their vantages differ:

| State | Definition | Fault? |
|---|---|---|
| **Honest vantage-conditioned divergence** | Differing values explained by declared conditioning variables | No |
| **Same-signer equivocation** | One signer, conflicting values for the same observation identity | Yes (Byzantine evidence) |
| **Unexplained divergence** | Differing values not attributable to any declared conditioning variable | Byzantine candidate; investigate, don't assume |

Profiles MUST NOT count honest vantage-conditioned divergence toward Byzantine fault bounds. The divergence≠equivocation boundary is test-locked in the reference implementation (ccip-router#3, 3ce2a76): the detector MUST warn on same-signer equivocation and MUST stay silent on multi-signer divergence.

## 4. Base dependencies and named extensions

### 4.1 Required base behavior (normative, by citation)
A conformant deployment runs above a base satisfying the #1826 amendment: observation-identical dedup only; divergence retained and expressible as `single | divergent(committed observation set) | absent`; never a silent discard; never represented as agreement; `resolveDivergence` as the attachment point.

### 4.2 Base extension: ATTEST-RETAIN — LANDED (ccip-router#4)

A signature-keyed attestation store populated on ingest, with a **two-level eligibility model** ratified this round:

- **Structural eligibility (base).** A record is a *signed attestation* iff its signature is a well-formed, non-empty 65-byte value. `getSignedAttestations(inputHash, namespace, value)` returns exactly this set; unsigned/dry-run records (`signature = "0x"`) remain observable via `getAttestations` but never enter it. Necessary, not sufficient.
- **Cryptographic eligibility (this companion).** Before minting signer weight or entering V2, a profile MUST additionally verify the signature is cryptographically valid over the observation and that the recovered signer binds to the declared source. The recovery/validity check is profile surface, per the layering.
- **Eligibility is recomputed at commitment time, normatively.** Ingest-side filtering and migration hygiene are never the guarantee: the store is evidence, not authority; the verdict verifies, it does not inherit. (Motivated by the v8-backfill finding: history cannot be assumed filtered.)
- **Canonical signature identity.** Signed attestation identity is the **low-s-canonical** (EIP-2) signature, enforced at ingest (aae4a34) with a load-bearing test: `(r, s)` and `(r, n−s)` of one signed message collapse to one attestation. This closes the malleability/storage-growth vector and is the stability basis for V2 (§5).
- **Unsigned fallback identity (ratified — Pavlo, verified independently — Fede, b9fc7e6):**

  ```
  attestation_id(unsigned) = "unsigned:" + SHA256( encodeJsonUtf8Lf({
      domain: "ccip.attestation.unsigned.v1",
      input_hash, namespace, key, value, timestamp }) )
  ```

  The preimage is a named canonical object (the family's one canonical form — sorted-key UTF-8 JSON, single trailing LF), never delimiter concatenation, which is not injective over unconstrained fields. `source_peer` and `signature` are excluded from identity **by enumeration**: transport metadata cannot redefine record identity. Distinct unsigned records of one observation stay distinct; two unsigned copies of one record from different peers collapse (transport isn't identity). Store PK: `(input_hash, namespace, value, attestation_id)`.
- **History closed by construction.** Migration reclassifies every existing row through the same identity function rather than raw-backfilling: legacy signed rows re-key on the low-s-canonical signature (historical malleated duplicates collapse on their own); legacy unsigned rows receive content identity and `signed = 0`. No cleanup pass required, and V2 does not depend on one.
- The signed envelope binds timestamp and payload today. Consensus-class profiles MAY additionally require bound `instanceId`, membership `epoch`, `round`, `phase`, `payloadHash` — tracked for if a consensus-class profile is specified.

Reference state: 76/76 on ccip-router's suite, `tsc` clean, fallback identity load-bearing (reverting it reds the distinct-unsigned test).

## 5. The resolution envelope

**Serialization (Q1, resolved):** the envelope and verdict are canonical JSON — sorted-key UTF-8 with a single trailing LF (`encodeJsonUtf8Lf` / the family's JCS gate) — carrying a `schema` + `version` field so every envelope self-identifies and can be pinned. The envelope is an off-chain declaration consumers recompute byte-for-byte; on-chain commitments may separately use typed encodings.

Every profile MUST publish a declaration containing:

- **E1 — Agreement object.** What is being agreed: the *observation set*, a *resolution verdict*, or an *action*. (Profile A agrees only on the observation set and resolves nothing.)
- **E2 — Membership.** Who counts; the signer→participant mapping (Sybil resistance); how membership changes bind (epoch, as-of).
- **E3 — Independence.** The vantage classes (§8) over which independence is assumed, per class — never globally.
- **E4 — Fault model and finality.** Tolerated taxonomy states and bounds; the established family whose safety argument is inherited, named rather than re-derived. **No evaluation carries finality unless the profile declares a finality rule here.**
- **E5 — Synchrony and evaluation identity.** The evaluator-local observation window/cutoff; late-evidence handling per §6a; timeout/round rules where applicable. Windows MUST NOT be defined over signed timestamps: a late-arriving attestation carrying an earlier signed timestamp is indistinguishable from early evidence — backdating and late arrival collapse. The window is evaluator-local; the digest binding (V2), not the store, is load-bearing.
- **E6 — Evidence requirements.** The named base extensions required (e.g. ATTEST-RETAIN for quorum-class profiles).

Every **verdict** MUST carry:

- **V1** — policy identifier and version;
- **V2** — the committed evidence-set commitment: a digest over the **sorted, low-s-canonical signatures of the cryptographically eligible attestation set** drawn from `getSignedAttestations`, eligibility recomputed at commitment time. V2's stability and ungameability rest on ATTEST-RETAIN's canonicalization (aae4a34), cited as its basis. A set commitment, never a count.
- **V3** — as-of bindings for any external state consulted (registry state, membership epoch, config version);
- **V4** — the vantage-class declarations under which independence was assumed;
- **V5** — resolution as a distinct state: `resolved(policy, committed set, conditions)` — never folded into `agreement`.

## 6. Output-state taxonomy (normative)

```
agreement                                   — genuine observation-level agreement over the committed set
divergence(committed attestation set)       — preserved, unresolved; the divergence itself is the verified fact
insufficient-observation(inspected-set)     — coverage limitation, with proof obligation (§9)
resolved(policy, committed set, conditions) — an opt-in profile's output; distinct from agreement by construction
```

**Resolved-state payload (Q2, resolved):** `resolved(...)` carries the committed-set **digest inline** and the **full set by reference**, with a retrievability obligation on the emitter. This imports a derived⇒retrievable invariant — the known IPFS failure shape — so retrievability MUST be **mechanically demonstrated, never merely declared**: the conformance vector for any resolving profile performs an actual retrieval of the referenced set and verifies it against the inline digest.

Two symmetric prohibitions:
- **No false green:** `agreement` MUST NOT be emitted for an auto-resolved disagreement; the taxonomy makes that state unrepresentable.
- **No false red:** profiles MUST NOT manufacture `divergence` from undeclared transformations (serialization, normalization, encoding); any transformation applied before comparison is declared and third-party recomputable.

### 6a. Evaluation identity, supersession, and finality (Q3, resolved)

**Evaluations are immutable but supersedable, and no evaluation carries finality unless a profile declares a finality rule.**

- Every evaluation binds, at production time, its evaluator-local window/cutoff and the exact committed evidence-set digest (V2). It is thereafter an **immutable historical claim**, verifiable forever under its original window and evidence.
- Late evidence never mutates an earlier verdict: the attestation store is append-only (`S0 ⊆ S1`), so new evidence produces a **new evaluation over a new committed set**, which MAY **supersede** the earlier one as the *current conclusion*. Historical claim and current conclusion are distinct statuses; supersession neither rewrites nor invalidates the superseded claim under its own binding.
- Division of labor (normative phrasing): **the base provides monotonic evidence retention; the companion provides historical evaluation identity.** `getAttestations` is a current-state read; the earlier evaluated set is recoverable only because the verdict bound it — old verdicts are *verifiable*, not *replayable*. (An `as_of` read parameter would be a convenience, explicitly not a correctness dependency.)
- Profile A's evaluations are trivially supersedable by construction. A consensus-class profile wanting a commit point MUST declare the finality rule and the family it inherits it from (E4) — never implied.

## 7. Profiles

### 7.0 Profile A — Divergence Surfacing (normative default)
Resolves nothing. Emits the §6 taxonomy minus `resolved`. E1 = observation set; no membership judgment, no independence assumption, no fault tolerance claimed; E6 = base only. A is the floor because it composes unconditionally: every downstream resolution layer can disclose exactly what it inherited. TSEI's fail-closed divergence state maps onto A directly.

### 7.1 Profile B — Quorum (opt-in)
**Requires ATTEST-RETAIN (E6), landed as ccip-router#4.** k-of-n agreement over the corroboration of each value, read via `getSignedAttestations` with cryptographic eligibility recomputed at commitment.
- **Counting rule (normative):** quorum counts **distinct signers per declared vantage class, never raw attestations.** Low-s canonicalization collapses the malleability of one signature, but a signer re-signing the same message under a fresh nonce mints a genuinely distinct attestation the base rightly retains — counting attestations would let one vantage inflate its own weight by re-signing. The base retains signed messages; the profile counts signers.
- Quorum is computed over the committed set (V2); quorum over a partial view is nonconformant.
- **Monoculture guard:** n counts *independent* vantages per the E3 declaration; independence is claimed per class (§8) and travels in the verdict (V4).
- Safety inherits quorum-intersection arguments from the declared family (E4); this profile adds no novel consensus claims.

### 7.2 Profile C — Registry-Weighted Authority (opt-in)
Declared vantage classes carry registry-assigned weight or rank. The registry MUST provide **verifiable history** (state-at-T provable, membership changes epoch-bound); absent that, this profile is nonconformant — a current-state read cannot establish issue-time validity (the certificate-revocation lesson). Every verdict binds registry state as-of evaluation time (V3); issue-time and read-time validity are distinct claims.

### 7.3 Profile D — Declared Priority (opt-in; kept per Q4)
A fixed ordering of vantage classes, set in deployment configuration.
- The config version is a conditioning variable and MUST be bound in the verdict (V3); config drift is a new policy version, never a silent change.
- **Any result Profile D produces remains a `resolved(policy, set, conditions)` divergence — never `agreement`** (V5 restated here deliberately, because D is the profile most tempted to launder ordering into consensus).
- Rationale for keeping D: deployments will do priority-ordering regardless; an explicit, declared, condition-carrying version of the rule is safer than an implicit authority rule. It remains the weakest profile; candidates SHOULD prefer A or B.

## 8. Vantage-class taxonomy

Independence is declared per axis, never assumed across all: **client class** (UA/library/TLS family), **network origin class** (ASN/region/egress), **operator class**, **design-context class** (shared design lineage — the third independence axis, per the affiliation-binding work). A profile claiming independence on an axis MUST state how the claim was established; the claim rides in V4. Extensible by new declared classes, never silent broadenings.

## 9. Insufficient-observation proof obligation

`insufficient-observation` MUST carry a committed enumeration of what **was** inspected. "Not found" without an inspected-set commitment is nonconformant. Where the store supports it, non-membership SHOULD be proven structurally (verifiable-map / sparse-tree non-inclusion) rather than asserted.

## 10. Conformance

- **Per-profile conformance vectors** follow the declared-expectation pattern: each vector declares the implementation under test and its expected conformance; a regressed implementation declared conformant goes RED.
- **Retrieval check (Q2):** the vector for any resolving profile MUST actually retrieve the referenced committed set and verify it against the inline digest — retrievability demonstrated, not declared.
- **Mutation-survival gate:** every normative MUST in this document maps to at least one mutant a conformant suite MUST KILL. The recipe is landed (recompute-kit#17: lineage + spec-MUST mapping) with a first worked example — `conformance/erc-8309/spec-mutations.json` maps the base's five §Deduplication MUSTs to guard mutants, 5/5 KILLED (ccip-router#3). This document's MUSTs receive an analogous `spec-mutations.json` alongside the first reference profile implementation.
- **Reference roles:** TSEI (producer of unresolved divergence; consumes Profile A as its defined resolution layer); Fede's /review verdict pipeline (reference consumer; validates A-plus-opt-in-B composition with disclosed-limitation verdicts and `execution_binding = external`).

## 11. Security considerations

- **Presence is not verification.** A row in the attestation store is not evidence of a signed attestation — the `"0x"` case is the live instance. Hence structural eligibility at the base, cryptographic eligibility at the profile, recomputed at commitment time (§4.2).
- **Signature malleability** — resolved at the base: low-s canonical identity enforced at ingest (aae4a34), load-bearing test. Remaining exposure: nonce re-signing mints distinct attestations by design; countered by the §7.1 signer-counting rule, not by storage.
- **Adversarial vantage selection (cloaking):** claims about serving behavior require multi-vantage observation across declared client and network-origin classes; single-vantage claims are conditioned claims and must say so.
- **Registry rewrite (C):** without verifiable history a registry operator can silently rewrite the past; hard conformance requirement in §7.2.
- **False-red economics:** alarms that fire on undeclared-transformation artifacts train operators to disable the mechanism; §6's symmetric prohibition is a security property.

## 12. Lineage and acknowledgments

- The base's recompute-and-compare mechanism generalizes ERC-8281's Verification Invariant from a single commitment edge to mesh observations; cited as lineage, not normative dependency.
- **Immutable-but-supersedable evaluations, no implied finality** (§6a): Jimmy Shi — along with the observation/attestation separation, the envelope bindings, the agreement-object trichotomy (E1), and the Q2 sharpening that retrievability be mechanically demonstrated.
- **Monotonic retention (base) / historical evaluation identity (companion)** (§6a), the **"0x" eligibility boundary**, the **preimage injectivity requirement**, and formal ratification of the unsigned-attestation identity: Pavlo.
- **Resolution is not agreement** (V5): Fede — who also independently verified b9fc7e6 against its description, and anchors the reference-consumer role.
- **§Deduplication amendment, reference implementation, structural/cryptographic eligibility split, low-s canonical identity, unsigned fallback identity implementation, the §7.1 signer-counting rule, divergence conformance vectors, and the mutation-gate recipe**: Tiago Merlini (ccip-router#3, #4; #1826; recompute-kit#17) — including the honest scoping that the live preimage exposure was narrower than the general argument, taken as future-proofing regardless.
- **Motivating fail-closed producer:** Pavlo's TSEI, whose refusal to invent this policy is why it exists as a declared layer.

## 13. Resolution and ratification record

| Item | Status | Record |
|---|---|---|
| Q1 — envelope serialization | **Resolved** | Canonical JSON + schema/version; Tiago (ccip-router side) |
| Q2 — resolved-state payload | **Resolved** | Digest inline + set by reference + demonstrated retrievability; explicit consensus: Fede, Jimmy, Pavlo; Tiago by proxy — full agreement, closed ahead of the 2026-08-24 04:00 UTC window |
| Q3 — evaluation identity / late evidence | **Resolved** | Immutable-but-supersedable, no implied finality; ratified by Jimmy |
| Q4 — Profile D | **Resolved** | Kept as explicit, condition-carrying opt-in; same full consensus as Q2 |
| Eligibility clause + unsigned preimage | **Closed** | Implemented Tiago (b9fc7e6); formally ratified Pavlo; independently verified Fede |
| ATTEST-RETAIN packaging | **Closed** | Landed as base amendment (ccip-router#4), per the data-model-lives-in-the-base argument |

**Next (v0.3 / implementation):** envelope schema JSON draft; reference Profile A implementation; `spec-mutations.json` for this document's MUSTs; Fede's /review use-case pass against the cut; Pavlo notified so TSEI can cite Profile A as its defined resolution layer — closing the loop that opened the thread.

---
*Draft v0.2 — cut 2026-08-23 against ccip-router#3/#4, ERCs#1826, recompute-kit#17. Every resolution above cites the message, commit, or ratification that produced it.*
