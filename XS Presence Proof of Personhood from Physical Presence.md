# XS Presence: Proof of Personhood from Physical Presence

*A litepaper extending the XS ID draft*

Oct 17, 2026 · @hitesh malviya

## Abstract

We propose XS Presence, a proof-of-personhood protocol that derives evidence of humanness from physical check-ins already recorded by visitor management systems (VMS). Instead of deploying new biometric hardware, the protocol turns each existing gate — a guard tablet in a housing society, a lobby kiosk in an office, or a single shopkeeper's phone — into a validator that signs presence attestations co-witnessed by people who saw a real person arrive.

Attestations remain in the visitor's own wallet. The chain stores only hourly Merkle commitments per site, stake, and nullifiers. A personhood score accumulates from attestations across many independent sites, weighted by site rarity, operator independence, witness strength and recency. Three assurance levels result: L1 Present, L2 Rooted, and L3 Unique, the last adding a zero-knowledge proof over a government credential to enforce one identity per person.

The paper makes four contributions. First, it reframes the XS ID draft's "VMS partners as miners" idea from identity issuance to presence attestation, which removes the honeypot and single-issuer forgery risks of the original. Second, it specifies the architecture, data model and protocol. Third, it gives a Sybil cost model showing that physical presence converts identity fraud from a marginal-cost-zero software attack into a linear-cost physical one. Fourth, it describes a four-tier validator network and SDK that admits sites with as little as one visitor a day, and a fee-funded sharing economy that pays gates, VMS vendors and verified humans.

## 1. Introduction

The internet has crossed a threshold: in 2024, automated bots generated 51% of all web traffic, the first time in a decade that machines outnumbered people online ([Imperva Bad Bot Report 2025](https://lenz.io/q/imperva-bad-bot-report-bot-traffic-findings)). Malicious bots alone were 37% of traffic. Large language models make each bot more convincing and cheaper to run, so the question "is this a real, unique human?" is becoming the most valuable signal a platform can buy.

Existing identity systems answer a different question. Passwords, phone numbers and email addresses prove control of an account, not the existence of a person behind it. The XS ID draft notes the cost of this gap: identity theft is among the fastest-growing crimes in the United States, at 9.9 million incidents a year, with 29% of affected victims spending a month or more on repair. Government ID verification answers the personhood question, but it centralises sensitive data and excludes people the issuing state does not serve well.

Dedicated proof-of-personhood networks have emerged in response. The most prominent, World, has verified more than 18 million humans with its custom iris-scanning Orb across three years of operation, and processed more than 475 million World ID proofs ([World Foundation, July 2026](https://hackernoon.com/world-foundation-raises-$525-million-to-scale-proof-of-human-as-ai-transforms-the-internet)). Its central constraint is physical: every verification needs a person to stand in front of a purpose-built device.

### 1.1 The overlooked network

A second physical network already exists at far greater density. Every apartment complex, office tower, hospital and campus runs some form of visitor management, where a person is seen, logged and admitted. In India alone, one vendor, Mygate, serves more than 27,000 residential communities and 5.7 million homes ([Entrepreneur India, June 2026](https://india.entrepreneur.com/business-news/mygate-bags-inr-225-crore-from-dharana-capital)), and was processing 3.8 million daily approval requests as early as 2021 ([HRKatha](https://www.hrkatha.com/?p=30893)). Globally, Envoy reports more than 100,000 visitor sign-ins a day across its customer workplaces ([Envoy](https://Envoy.com/blog/why-you-should-invest-in-a-visitor-management-system)). The VMS market is projected to grow from US$2.0 billion in 2023 to US$5.7 billion by 2030 ([Global Industry Analysts via GII](https://m.giikorea.co.kr/report/go1594876-visitor-management-systems.html)).

Each of these check-ins is a weak but real proof of personhood: a human guard or host saw a body arrive at a specific place and time. Today that signal is trapped in vendor databases and discarded after a retention period.

### 1.2 Contributions

This litepaper specifies a protocol to make that signal portable, private and Sybil-resistant. We contribute:

1. A threat model and requirements for presence-derived personhood (Section 2).
2. A competitive analysis of seven existing approaches (Section 3).
3. A layered architecture and attestation data model in which no personal data reaches the chain (Section 4).
4. A protocol for witnessing, batching, scoring and recovery, with three assurance levels (Section 5).
5. A four-tier validator network and SDK that includes single-footfall sites (Section 6).
6. A security analysis and Sybil cost model (Section 7), an economic model (Section 8), and a go-to-market plan (Section 9).

## 2. Problem statement

The problem is to let any application ask "is this account controlled by a real human, and by a human who holds no other account here?" and receive a verifiable answer that reveals nothing else. We call this Sybil-resistant, privacy-preserving proof of personhood (PoP).

### 2.1 Requirements

A deployable PoP system must satisfy six requirements at once. Existing systems typically trade one or more away.

| Requirement | Definition | Why it is hard |
| --- | --- | --- |
| R1 Sybil resistance | The marginal cost of an additional fake identity is high and grows with scale | Software identities have near-zero marginal cost |
| R2 Uniqueness | At the highest assurance level, one person maps to one identity | Requires deduplication against everyone already enrolled |
| R3 Privacy | Verifiers learn only a yes/no and a per-app pseudonym | Deduplication usually needs a central database |
| R4 Inclusivity | Enrolment works without a smartphone-heavy flow, special hardware or a specific nationality | Most strong methods depend on one of these |
| R5 Decentralisation | No single operator can forge or revoke identities | Issuance tends to centralise |
| R6 Cost and reach | Enrolment cost per person is low and coverage is dense | Dedicated hardware is expensive to deploy |

### 2.2 Threat model

We consider five adversaries. The **bot farm** creates identities at scale with scripted devices and AI-generated faces. The **colluding site** is a building whose guard, owner or VMS account fabricates check-ins. The **colluding operator** is a VMS vendor that injects fake attestations across its fleet. The **surveillance adversary**, which may be a platform, a vendor or a state, tries to reconstruct where a person has been. The **key thief** steals a visitor's phone to impersonate them.

We assume that most guards and hosts act honestly most of the time, that VMS vendors are economically rational, and that standard cryptographic primitives (ECDSA or EdDSA signatures, Poseidon hashing, Groth16 or PLONK proofs) are secure. We do not assume any single vendor is trustworthy.

### 2.3 Where the XS ID draft falls short

The XS ID draft correctly identifies VMS partners as the natural "miners" of an identity network. Five of its design choices, however, conflict with the requirements above.

1. **Identity data on a sidechain** (violates R3). Names, documents and face data written to any chain are permanent and replicated, recreating the breached-database problem the draft opens with.
2. **VMS partners create IDs** (violates R5). If one partner can issue an identity, a compromised partner can issue unlimited fake ones.
3. **Government ID, social media, email and phone collected at enrolment** (violates R3 and R4). This builds a honeypot and excludes people without documents or social accounts.
4. **BTC as the monetary token** (violates R6). Volatile value and high fees make sub-cent payouts to small sites impractical.
5. **Recovery by face detection and document re-upload** (violates R1). Recovery becomes the weakest link, the same trade-off the draft itself warns about.

XS Presence keeps the draft's network insight and replaces these five mechanisms.

## 3. Related work and competitive landscape

Existing proof-of-personhood systems fall into four families: dedicated biometrics, aggregated credentials, social graphs, and government-credential proofs. None uses existing physical access infrastructure as its source of evidence, which is the gap XS Presence fills.

### 3.1 Dedicated biometric hardware

**World** (formerly Worldcoin) scans irises with a custom Orb, converts them to an iris code, and issues a World ID used through zero-knowledge proofs. It is the scale leader, with more than 39 million users and 18 million Orb-verified humans as of July 2026 ([HackerNoon](https://hackernoon.com/world-foundation-raises-$525-million-to-scale-proof-of-human-as-ai-transforms-the-internet)), and has signed integrations with Tinder, Zoom and Docusign ([Decrypt](https://decrypt.co/364774/sam-altman-world-zoom-tinder-better-verify-humans-ai-age)). Its strengths are strong uniqueness and mature ZK tooling. Its constraints are hardware throughput, the cost of deploying Orbs, and public resistance to iris scanning.

**Humanity Protocol** uses palm-vein biometrics captured by smartphone or dedicated scanners, paired with ZK proofs ([Alea Research](https://alearesearch.io/deep-dives/humanity/)). It reports more than 6 million Human IDs created ([Alea Research](https://alearesearch.io/?p=6699)) and has explicitly targeted building access and hotel check-ins as use cases ([Blockmedia](https://www.blockmedia.co.kr/archives/888959)). It is therefore the closest competitor in spirit, but it still asks buildings to install new palm scanners rather than reusing what they run.

### 3.2 Aggregated credentials

**Human Passport** (formerly Gitcoin Passport, acquired by Holonym Foundation in December 2024) aggregates "Stamps" from many verifiers into a Unique Humanity Score ([human.tech](https://human.tech/blog/proof-of-personhood-human-passport)). At acquisition it had more than 2 million users and 35 million credentials ([Cointelegraph via TradingView](https://tr.tradingview.com/news/cointelegraph:c7467773e094b:0-zk-identity-project-holonym-acquires-gitcoin-passport-for-10m)). Its scoring model is the direct ancestor of ours. Its weakness is that most Stamps are online signals that bots can farm with enough accounts.

### 3.3 Social graphs and synchronous tests

**BrightID** infers uniqueness from a web of mutual verifications between people. **Proof of Humanity** combines a video submission, vouching by an existing member and a dispute court. **Idena** requires every participant to solve AI-hard puzzles at the same global moment. These designs avoid biometrics but suffer from low throughput, onboarding friction and vulnerability to coordinated rings.

### 3.4 Government-credential proofs

**Anon Aadhaar**, an open-source project, lets a holder of India's Aadhaar prove facts from its signed QR code in zero knowledge. Similar circuits exist for e-passports. These give strong uniqueness within one jurisdiction, but exclude non-holders and depend on the issuer's own integrity. XS Presence uses this family only at its top assurance level.

### 3.5 Comparison

| System | Evidence of personhood | New hardware | Uniqueness | Personal data exposure | Cost per enrolment | Main weakness |
| --- | --- | --- | --- | --- | --- | --- |
| World | Iris scan at Orb | Yes, custom Orb | Strong | Iris code held by network | High (device fleet) | Hardware throughput, public trust |
| Humanity Protocol | Palm vein scan | Phone or palm scanner | Strong | Palm template, ZK-wrapped | Medium | New scanners needed for physical sites |
| Human Passport | Aggregated online Stamps | No | Medium | Per Stamp | Low | Online signals are farmable |
| BrightID | Social graph | No | Medium | Social links | Low | Rings, slow onboarding |
| Proof of Humanity | Video plus vouch | No | Medium | Public video | Low | Public face exposure |
| Anon Aadhaar | Government QR in ZK | No | Strong within India | None revealed | Very low | Excludes non-holders |
| **XS Presence** | Co-witnessed physical check-ins | **No, reuses VMS** | Medium (L2), strong (L3) | None on chain; wallet-held | Near zero marginal | Collusion at sites; needs density |

The last row summarises the claim of this paper: presence attestation is the only approach whose enrolment cost is already paid by building security budgets.

## 4. System architecture

XS Presence separates evidence into three zones — the physical gate, the visitor's private wallet, and a public settlement chain — so that no single zone holds both identity and location.

&#91;embedded content: XS Presence architecture · three zones, four message flows\]

A gate signs an attestation and hands it to the wallet (1), then commits only a hash of it inside an hourly Merkle root (2). The wallet later proves a level to a verifier in zero knowledge (3), and the verifier checks the root and nullifier on chain and pays its fee (4). Payouts flow from the fee pool back to gates, vendors and wallets each epoch.

### 4.1 Actors

| Actor | Holds | Trusted for | Not trusted for |
| --- | --- | --- | --- |
| Visitor | Identity secret, own attestations | Keeping their key | Anything about other people |
| Guard and host | Witness keys bound to the site | Seeing a live person | Uniqueness across sites |
| Gate (site) | Device key in secure enclave, leaf hashes | Signing honestly, under stake | Identity issuance |
| Gate Hub (VMS vendor) | Aggregated roots for its fleet | Relaying and fleet fraud checks | Minting attestations |
| Anchor set | Settlement chain state | Ordering roots, audits, governance | Seeing any personal data |
| Verifier | A level and a per-app nullifier | Paying fees | Learning sites, times or counts |

### 4.2 Attestation data model

An attestation is a small signed record created at the gate and stored only by the visitor:

```json
{
  "v": 1,
  "visitor_key": "k = H(s)",
  "site_id": "rotating per-epoch site identifier",
  "operator_id": "VMS vendor identifier",
  "time_bucket": "2026-10-17T10:00Z (hour granularity)",
  "witness": {"guard": "sig", "host": "sig or null"},
  "witness_class": "lite | guard | guard_host",
  "liveness_hash": "on-device hash, photo never sent",
  "nonce": "from the visitor QR, 30 s validity",
  "gate_sig": "device-key signature over all fields"
}
```

The leaf committed on chain is H(attestation ‖ salt), where the salt is known only to the gate and the visitor. Hour-level buckets limit timing precision.

### 4.3 What lives where

| Data | Gate | Wallet | Chain |
| --- | --- | --- | --- |
| Name, phone, photo | Vendor's existing records only | No | No |
| Full attestation | No (leaf hash only) | Yes | No |
| Hourly Merkle root per site | Computes | No | Yes |
| Site stake and slashing state | No | No | Yes |
| Per-app nullifiers | No | Derives | Yes |
| Personhood level | No | Proves | Verified, not stored |

### 4.4 Settlement layer

The settlement chain needs cheap writes, a mature ZK verifier and stable-unit payments. An Ethereum layer-2 rollup meets all three today. Roots, stakes and nullifiers are its only state, so the protocol can migrate or post to several chains without moving personal data.

## 5. Protocol

The protocol has five phases: enrolment, check-in and witnessing, batching, scoring, and proving. Recovery and revocation run alongside them.

### 5.1 Enrolment

Enrolment is the creation of a key pair in the visitor's wallet. Nothing else is collected. The wallet derives a long-term identity secret *s* and a public visitor key *k* = H(*s*). Per-application pseudonyms are derived later as H(*s*, app\_id), so two apps cannot link the same person. A visitor without a smartphone receives a printed QR card whose key is held by a custodial guardian wallet of their choice, such as a family member or an NGO.

### 5.2 Check-in and witnessing

A check-in proceeds in four steps, each mapped to an existing VMS action:

1. **Present.** The visitor shows the XS QR instead of typing a name. The QR carries *k* and a fresh nonce, refreshed every 30 seconds to prevent replay.
2. **Gate signature.** The site's device key, held in a secure enclave, signs the attestation body.
3. **Guard co-signature.** The guard confirms a live person stood at the gate. The SDK records a liveness hash computed on device; the photo is never transmitted.
4. **Host co-signature.** When the VMS asks a resident or employee to approve the visit, that approval becomes a third signature from an independent key.

The resulting attestation *a* is returned to the visitor's wallet over the same QR channel or by local push. The gate keeps only a salted leaf hash for batching and audit.

### 5.3 Batching

Each hour, a site computes a Merkle root over its leaf hashes and posts it to the Commitment Registry. Gate Lite sites without connectivity queue leaves locally and submit through their neighbourhood pool, which posts a root of roots. A root costs one transaction regardless of the number of visitors, so on-chain cost per attestation falls as volume rises.

### 5.4 Personhood scoring

The score for visitor key *k* is a weighted sum over distinct sites *s* in the visitor's attestation set:

```latex
P(k) = \sum_{s \in S(k)} w_{div}(s) \cdot w_{ind}(op(s)) \cdot w_{wit}(s) \cdot 2^{-\Delta t / \tau}
```

Here w\_div rewards rare sites, so a clinic with one visitor a day weighs more than the fiftieth visit to one mall. w\_ind falls when sites share an operator, owner, network range or device batch. w\_wit is 0.4 for a sticker-only Lite gate, 1.0 for a guard-confirmed check-in and 1.2 with host approval. The decay term halves weight every τ = 180 days so dormant or sold keys fade. Only the first attestation per site per day counts, which stops repeated check-ins at a single colluding site from inflating the score.

### 5.5 Assurance levels

| Level | Rule | Suitable for |
| --- | --- | --- |
| L1 Present | 3 distinct sites | Comment sections, ticket queues, rate limits |
| L2 Rooted | 5 sites, at least 3 independent operators, spread over at least 30 days | Airdrops, marketplaces, dating, community governance |
| L3 Unique | L2 plus a ZK proof over a government credential yielding a global uniqueness nullifier | Benefits, voting, universal basic income, one-person-one-share |

Thresholds are initial testnet parameters to be tuned against measured attack data.

### 5.6 Proving

When an application requests verification, the wallet builds a zero-knowledge proof that (a) it knows *s* behind *k*, (b) each attestation it uses is included under a posted root, (c) the weighted sum meets the requested level, and (d) the per-app nullifier H(*s*, app\_id) has not been used for this action before. The verifier learns the level and the nullifier only. It never learns which sites, how many visits, or when.

### 5.7 Recovery and revocation

Recovery replaces the XS ID draft's face-and-document flow with a **return-to-place** rule. A visitor who loses a device generates a new key and checks in at two sites that hold attestations for the old key within the past 180 days, plus one approval from a pre-registered guardian. The protocol then publishes a key-rotation record and migrates the score. A stolen device can be frozen by any two guardians, after which its pending attestations are rejected.

## 6. Validator network and SDK

Every gate that signs attestations is a validator, and the network has four tiers so that a 40-tower township and a single-room clinic can join on terms that fit them. Validators do not order transactions in the general-purpose sense; they validate one claim — that a person was physically present — and accept economic penalties if that claim is false.

### 6.1 Node tiers

| Tier | Typical operator | Hardware | Responsibilities | Stake and penalty |
| --- | --- | --- | --- | --- |
| Gate Lite | Clinic, shop or small office with 1–10 visitors a day, no VMS | Printed XS sticker and the owner's phone | Co-signs check-ins; submits through a neighbourhood pool of about 50 Lite gates | No upfront stake; rewards held 30 days as a bond and forfeited on proven fraud |
| Gate Node | Housing society, office or clinic running any VMS with the SDK | The guard tablet or kiosk already in use | Signs attestations with guard and host witnesses; posts hourly roots; answers audits | Small stake, funded from the site's fee share; slashed for fake attestations |
| Gate Hub | A VMS vendor such as Mygate, Envoy or a regional peer | Cloud full node | Aggregates customer roots; runs fleet-level anomaly detection; distributes payouts | Significant stake; slashed for fleet-wide fraud or ignoring alerts |
| Anchor | Consortium of hubs, large campuses, civic bodies and the foundation | Validator set on the settlement chain | Orders roots; runs random presence audits; governs weights and thresholds | Highest stake; governance accountability |

### 6.2 Including a one-footfall building

Gate Lite is designed for the smallest possible site: a building with one visitor a day. Its owner prints an XS sticker containing the site's public key and registers it from a phone app in under two minutes. A visitor scans the sticker with their wallet, the owner taps confirm, and both signatures form an attestation with witness weight 0.4. The phone queues leaves offline and forwards them to the neighbourhood pool, which posts a combined root once an hour.

The economics favour these sites by design. A bot farm can rent desks in one coworking space and inflate its traffic, but it cannot easily control hundreds of independent corner clinics, tailors and chemists, each with a different owner. Because w\_div is highest for low-traffic sites (×3.0 at up to 10 visitors a day, against ×1.0 above 1,000), a single-footfall gate contributes more Sybil resistance per visit than a busy lobby. Its income stays small — a few US cents a month in the model in Section 8 — so the operator's practical benefit is a free digital visitor register and a share in the network.

### 6.3 Audits and slashing

Anchors issue random **presence challenges**. A challenged site must, within 24 hours, prove inclusion of a sampled attestation and produce the matching guard and host signatures. A challenged visitor may be asked to re-attest at any other site within seven days. Failure triggers stake slashing for the site and a score freeze for the key. Hubs are slashed when more than a set share of their fleet's sampled attestations fail, which makes VMS vendors responsible for policing their own customers.

### 6.4 SDK

The SDK is a thin library that sits behind the check-in screen a VMS already ships. It is available for Android, iOS, web and server environments, and exposes seven calls.

| Call | Purpose |
| --- | --- |
| `init()` | Registers the site, binds a device key in the secure enclave, sets the tier |
| `attest()` | Signs a presence record for a scanned visitor key and returns a copy to the wallet |
| `cosign()` | Adds guard or host signatures from existing VMS approval flows |
| `batch()` | Builds and posts the hourly Merkle root; handles offline queues and retries |
| `challenge()` | Responds to Anchor audits with inclusion proofs and witness signatures |
| `collide()` | On-device check that flags one face behind two keys at the same site; templates deleted after 30 days |
| `payout()` | Claims the site's share each epoch; Lite pools split automatically |

A minimal integration looks like this:

```javascript
import { XSGate } from "@xs/gate-sdk";

const gate = await XSGate.init({
  siteKey: secureEnclave.key("xs-site"),  // never exported
  operator: "your-vms-id",
  tier: "node"                              // "lite" | "node" | "hub"
});

// inside the existing check-in handler
await gate.attest({
  visitorKey: visit.xsQr.pubKey,             // no name or phone number
  witnesses: [guard.sign(), host.approval()],
  liveness: localHash(visit.photo)           // photo stays on device
});

gate.on("challenge", audit => audit.respond());
```

The integration target is one engineering sprint for an existing VMS vendor. No change to the vendor's visitor database is required, because the SDK keeps its own leaf store.

## 7. Security analysis

Presence attestation does not make Sybil attacks impossible; it makes each fake identity cost a physical action, which turns a near-zero-marginal-cost software attack into a linear-cost physical one. This section quantifies that cost and walks through the threat model from Section 2.2.

### 7.1 Sybil cost model

Let *n* be the number of distinct site visits needed for a level (5 for L2), *c\_v* the cost of one visit (travel plus time), and *N* the number of fake identities an attacker wants. Without collusion the attack cost is:

```latex
C_{honest}(N) = N \cdot n \cdot c_v
```

With an assumed *c\_v* of US$1.50 for an urban trip and 20 minutes of labour, one L2 identity costs about US$7.50 and 1,000 identities cost about US$7,500, plus 30 days of calendar time per identity. By contrast, a scripted phone-verified account typically costs cents. The figure is an illustrative assumption, but the structure matters more than the constant: cost grows linearly with *N* and cannot be amortised by software.

An attacker can lower *c\_v* by colluding with sites. Because only one attestation per key per site per day counts, and L2 requires three independent operators, the attacker needs at least five colluding sites spread over three operators. If each colluding site posts stake *σ* and is audited with probability *p* per sampled attestation, the expected loss from minting *m* fake attestations at one site is roughly σ · (1 − (1 − p)^m). With σ = US$200 and p = 1%, a site that mints 500 fake attestations faces a 99% chance of losing its stake, which exceeds any plausible bribe for 100 fake identities.

### 7.2 Attacks and mitigations

| Attack | Description | Mitigation |
| --- | --- | --- |
| Bot farm with AI faces | Scripted wallets present QR codes | A guard or host must co-sign; Lite gates require the owner's tap on a physical device |
| One person, many keys | A real human enrols several keys and visits sites with each | Linear cost per key; `collide()` flags one face behind two keys at one site; L3 enforces global uniqueness |
| Colluding site | A guard or owner fabricates visits | Per-site daily cap, independence weights, random audits, stake slashing |
| Colluding operator | A VMS vendor mints across its fleet | w\_ind caps one operator's share of any L2 score; hub stake; cross-hub audits |
| Key theft | A stolen phone is used to impersonate | Device binding, guardian freeze, return-to-place recovery |
| Replay | An old QR is re-shown | 30-second rotating nonce; attestation bound to site and time bucket |
| Sybil sites | Attacker registers many fake Lite gates | New sites start at low weight and must be vouched by an existing hub or 3 nearby gates; weight grows with honest audit history |

### 7.3 Privacy analysis

The most sensitive artefact in the system is a person's location history. XS Presence protects it in four ways. First, raw attestations live only in the visitor's wallet. Second, the chain holds only salted leaf hashes inside hourly roots, which reveal neither visitor keys nor counts per visitor. Third, site identifiers inside proofs are rotated per epoch, so a verifier cannot accumulate a site list across sessions. Fourth, proofs disclose a level and a per-app nullifier, never sites, times or counts.

Residual risks remain. A VMS vendor still sees its own check-ins, exactly as it does today, so XS Presence adds no new exposure at the vendor but does not remove the existing one. Timing correlation between a check-in and a later on-chain root is possible for very small sites; Lite pools mitigate this by mixing roots across about 50 gates.

### 7.4 Regulatory compliance

India's Digital Personal Data Protection Act 2023 requires specific, informed consent for processing personal data. The protocol is opt-in at each QR scan, never on by default, and the existing name-entry flow must remain available. Because no personal data is written on chain, the right to erasure is satisfied by deleting the wallet and the vendor's ordinary records.

## 8. Economics and the sharing economy

The network is funded by verifiers paying per proof, not by token issuance, and those fees are shared among the gates, vendors and humans who produced the evidence. This avoids the funding gap of systems that subsidise enrolment with newly minted tokens before any external revenue exists.

### 8.1 Fee flow

An application — a social platform, marketplace, bank, ticketing system or AI agent platform — pays a fee for each verified proof it accepts, denominated in a stable unit and settled on a low-fee chain. Fees accrue to an epoch pool and are distributed monthly.

| Recipient | Share | Basis of distribution |
| --- | --- | --- |
| Gates | 40% | Pro rata to weighted attestations that were used in proofs during the epoch |
| Humans | 25% | Per proof in which the human's own attestations were used, paid to their wallet |
| VMS vendors | 15% | Pro rata to attestations from their fleets; compensates SDK integration and hub operation |
| Treasury | 20% | Audits, fraud bounties, free Lite stickers, grants and reserves |

The human share is the sharing-economy element the XS ID draft gestures at: a person's presence is their data, so they are paid when it is used. Gates that sign attestations that are never used in a proof earn nothing, which discourages padding.

### 8.2 Modelled scenarios for India

All inputs below are assumptions chosen to show the shape of the economy, not forecasts.

| Scenario | L2 humans | Proofs per human per month | Fee per proof (US$) | Annual fees (US$) | Gate pool (US$) | Human pool (US$) |
| --- | --- | --- | --- | --- | --- | --- |
| Conservative | 2 million | 2 | 0.02 | 0.96M | 0.38M | 0.24M |
| Base | 10 million | 4 | 0.03 | 14.4M | 5.76M | 3.6M |
| Ambitious | 50 million | 6 | 0.04 | 144M | 57.6M | 36M |

The base case's 10 million L2 humans compare with the 45 million people living in Indian gated communities as of 2019 ([Mygate Series B release](https://stg.primevp.in/news-and-announcement/mygate-raises-56-million-series-b-further-enhance-security-convenience-gated-premises)).

### 8.3 What a site earns

If the base-case gate pool is spread over about 3.8 billion weighted attestations a year, each weighted attestation is worth about US$0.0015. Two illustrative sites:

- A 500-flat housing society with 300 visitors a day, guard and host witnesses (w\_wit 1.2) and w\_div 1.2: about 158,000 weighted attestations and **US$236 a year**.
- A clinic with one visitor a day using Gate Lite (w\_wit 0.4, w\_div 3.0): about 440 weighted attestations and **US$0.66 a year**.

The per-human dividend in the base case is about US$0.36 a year. These amounts are deliberately honest: XS Presence is not an income scheme. Its value to sites is security tooling paid for by the network, and its value to humans is a portable, private credential with a small usage dividend. Larger flows depend on proof volume from AI-agent platforms and financial services, which is the main commercial uncertainty.

### 8.4 Token design

The protocol needs no speculative token to operate. Fees and payouts use a stable unit. A governance and staking token may be introduced at the Anchor tier once fee revenue exists, with emissions capped by realised fees, so that rewards never run ahead of external demand.

## 9. Competitive edges, go-to-market and roadmap

XS Presence competes on distribution rather than on cryptography: its zero-knowledge stack is comparable to peers, but its enrolment network is already installed, paid for and visited by millions of people every day.

### 9.1 Competitive edges

1. **Zero hardware capex.** World and Humanity Protocol must manufacture and place devices. XS Presence reuses guard tablets, kiosks and phones already funded from building security budgets.
2. **Enrolment happens as a side effect.** People do not travel to a verification point; they are attested while visiting a friend, a doctor or an office. Enrolment friction approaches zero.
3. **No biometric database.** The only biometric processing is the optional on-device `collide()` check, deleted after 30 days. This avoids the regulatory and public-trust barriers that iris and palm systems face.
4. **Continuous, not one-time, evidence.** A one-off scan proves a human existed at enrolment. Ongoing presence proves the key is still held by an active human, which defeats the resale of verified accounts.
5. **Vendor-aligned incentives.** VMS vendors gain a new revenue line (15% of fees) and a differentiator against competitors, so they distribute the SDK themselves.
6. **Density in the Global South.** Gated communities and staffed reception desks are common across Indian cities, where dedicated-hardware networks are thin.
7. **Composable with others.** XS Presence attestations can be issued as Human Passport Stamps or combined with an Anon Aadhaar or World ID proof, so it can grow as a complement before it competes head-on.

### 9.2 Go-to-market

The launch sequence follows the density of check-ins. Phase one targets one VMS vendor and one metro region — for example Thane and Mumbai — where housing societies are dense and a single vendor can deliver thousands of sites. Phase two adds office and clinic VMS vendors and opens Gate Lite to shops and small clinics through local partners. Phase three signs demand-side verifiers: dating apps, classifieds and resale marketplaces, ticketing for concerts and cricket, and airdrop or grant programmes that already buy Sybil resistance.

The first demand product is a hosted verifier API with a free tier, priced per proof above it. The first supply incentive is a free digital visitor register and a fee share for any site that enables attestations.

### 9.3 Roadmap

The roadmap is gated by evidence rather than dates: each phase starts only when the previous phase proves both adoption and fraud resistance.

&#91;embedded content: Roadmap · four phases, three evidence gates\]

Dates and gate thresholds are proposals for planning and will be revised against testnet results.

## 10. Limitations and conclusion

XS Presence's central limitation is that presence prices Sybil identities rather than eliminating them; any use that needs strict one-person-one-share must require L3.

### 10.1 Open questions

- **Weight calibration.** The diversity, independence and witness weights in Section 5.4 are design guesses. They need a testnet with red-team attackers to calibrate.
- **Guard fatigue.** Guards already handle heavy traffic. If co-signing adds even two seconds per visitor, adoption may stall. The SDK must make co-signing part of the existing approve tap.
- **Rural and low-density coverage.** People who rarely enter staffed buildings will struggle to reach L2. Gate Lite, community centres, ration shops and post offices are candidate sites, but coverage outside cities is unproven.
- **Vendor concentration.** If one VMS vendor dominates a region, the independence requirement becomes hard to meet. Lite gates and cross-vendor hubs are partial answers.
- **Demand-side volume.** Fee revenue depends on how many proofs verifiers buy. AI-agent authorisation and financial onboarding are the most promising sources, but neither is contracted.
- **Legal treatment of attestations.** Whether a presence attestation is personal data under the DPDP Act and GDPR, even when held only by the subject, needs formal legal review.

### 10.2 Conclusion

The XS ID draft identified a real and overlooked asset: the visitor management systems that already check whether a person is standing at a gate. This paper keeps that insight and rebuilds the system around it. Gates attest instead of issuing. The chain holds commitments instead of identities. Personhood accumulates from many independent witnesses over time. A single-footfall clinic can participate, and is rewarded for being hard to fake.

In an internet where bots now outnumber humans, the scarcest resource is evidence that a real person showed up somewhere. Buildings have been collecting that evidence for decades. XS Presence proposes to let people keep it, prove it privately, and share in its value.

### References

1. XS ID draft whitepaper (XS\_Draft.pdf), supplied by the authors.
2. [Imperva (Thales) Bad Bot Report 2025, summary](https://lenz.io/q/imperva-bad-bot-report-bot-traffic-findings).
3. [World Foundation raises $52.5M on third anniversary — HackerNoon](https://hackernoon.com/world-foundation-raises-$525-million-to-scale-proof-of-human-as-ai-transforms-the-internet), July 2026.
4. [World ID upgrade with Tinder, Zoom and Docusign — Decrypt](https://decrypt.co/364774/sam-altman-world-zoom-tinder-better-verify-humans-ai-age), April 2026.
5. [Mygate raises INR 225 crore — Entrepreneur India](https://india.entrepreneur.com/business-news/mygate-bags-inr-225-crore-from-dharana-capital), June 2026.
6. [Mygate team expansion and daily approvals — HRKatha](https://www.hrkatha.com/?p=30893), 2021.
7. [Mygate Series B release — Prime Venture Partners](https://stg.primevp.in/news-and-announcement/mygate-raises-56-million-series-b-further-enhance-security-convenience-gated-premises), 2019.
8. [Why invest in a visitor management system — Envoy](https://Envoy.com/blog/why-you-should-invest-in-a-visitor-management-system).
9. [Global Visitor Management Systems market — Global Industry Analysts via GII](https://m.giikorea.co.kr/report/go1594876-visitor-management-systems.html).
10. [Humanity Protocol deep dive — Alea Research](https://alearesearch.io/deep-dives/humanity/) and [report](https://alearesearch.io/?p=6699).
11. [Humanity Protocol DePIN use cases — Blockmedia](https://www.blockmedia.co.kr/archives/888959).
12. [Human Passport: proof of personhood — human.tech](https://human.tech/blog/proof-of-personhood-human-passport).
13. [Holonym acquires Gitcoin Passport — Cointelegraph via TradingView](https://tr.tradingview.com/news/cointelegraph:c7467773e094b:0-zk-identity-project-holonym-acquires-gitcoin-passport-for-10m).
