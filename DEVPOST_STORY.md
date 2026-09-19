## Inspiration

A phone rings. The caller says he is from a national investigation agency: the victim's identity has been linked to a money-laundering case, she is under **"digital arrest"**, and she must not hang up. He keeps her on the line for hours. Eleven minutes after he orders a "verification transfer", **₹4,80,000 leaves her account**. Within thirty minutes it has been split across six mule accounts, layered, and withdrawn at ATMs across the city.

What struck us was not how clever this crime is. It is how **visible** it is — to everyone, separately.

```
   TELECOM          BANK            CAMERAS          PLATFORMS
  "a call"      "a transfer"       "a car"           "a post"
      |               |               |                  |
      +---------------+-------+-------+------------------+
                              |
                              v
          five true, ordinary facts in five buildings.
          the crime exists only in the JOIN -- and
          no single custodian can make it.
```

The telco sees a normal call. The bank sees a legal IMPS transfer. The camera sees a car at a junction. Investigators make the join by hand, with spreadsheets and weeks — while the money moves in minutes.

The second thing that drove us was AI itself. Everyone wants to point a language model at this problem. But an investigation tool that *hallucinates* a transaction is worse than no tool at all: it manufactures evidence against a real person. So we set ourselves a harder brief than "use AI":

> **Build an AI investigator that is structurally incapable of lying about the record — and make every one of its claims provable.**

## What it does

**TRACE X** is an evidence-grounded investigation workbench for financial cybercrime. It takes the raw exports investigators actually receive — call detail records, internet session records, bank statements, social posts and licence-plate reads — and:

1. **Hash-chains every record at the moment of ingest**, so tampering with a single rupee is detectable and localisable.
2. **Resolves identifiers into people** — phones, handsets, SIMs, accounts and handles collapse into real entities, with every merge justified by a named rule.
3. **Performs the join no custodian can** — call-to-debit coupling, money fan-out, SIM rotation, synchronised cash-outs, shared network infrastructure.
4. **Ranks people for review with a trained, calibrated, explainable model** — and shows *exactly* why each score is what it is.
5. **Argues for the defence** — every flagged person is tested against the innocent explanations that fit equally well.
6. **Lets autonomous agents investigate** with read-only tools — then **re-verifies every sentence they write** against the original hashed records before a human sees it.
7. **Reconstructs the crime in space and time** — a scrubbable 128-step map of the case, plus a live 3D globe for situational context.

```
+---------------------------------------------------------------------+
| PRESENTATION  Next.js 14 | 36 screens | d3 graphs | CesiumJS globe  |
+----------------------------------+----------------------------------+
                                   | REST + bearer identity
+----------------------------------v----------------------------------+
| API           FastAPI | 124 endpoints | role-based access | audit   |
+----------------------------------+----------------------------------+
                                   |
+----------------------------------v----------------------------------+
| ENGINE  dataset -> resolve -> correlate -> score -> graph / hunt    |
|         pattern-of-life | spatial | evidentiary | TRUTH GATE        |
+-----------+-----------------------+-----------------------+---------+
            |                       |                       |
+-----------v---------+ +-----------v---------+ +-----------v---------+
| MODEL LAYER         | | AGENT MESH          | | EVIDENCE LEDGER     |
| XGBoost | isotonic  | | 8-role pipeline     | | rows + SHA-256      |
| TreeSHAP | pinned   | | response agent      | | append-only chain   |
| artifacts           | | LLM tool-use agent  | | SQLite (WAL)        |
+---------------------+ +---------------------+ +---------------------+
```

On the seeded case, TRACE X turns **2,119 records** into **2,147 graph nodes and 3,397 relationships**, resolves them to **20 people**, surfaces **7 fast call-to-debit couplings, 7 fan-out patterns and 3 SIM-rotating handsets**, and finds **2 coordinated campaigns** — a 12-member mule network with ₹45.9 lakh exposure and a 5-member cluster with ₹13.1 lakh. A full cold rebuild (dataset, resolution, features and scores for everyone) takes **138 ms**; queries on a warm dataset take **4–5 ms**.

## How we built it

### 1. The evidence ledger — blockchain's core primitive, without the consensus layer

Every ingested row is serialised to canonical JSON (sorted keys, no whitespace) and hashed. Rows are then linked into an append-only chain, exactly the construction that makes a blockchain trustworthy:

$$h_0 = \mathrm{SHA256}(\varepsilon)$$

$$d_i = \mathrm{SHA256}\big(\mathrm{canon}(r_i)\big)$$

$$h_i = \mathrm{SHA256}\big(\mathrm{hex}(h_{i-1}) \mathbin{\Vert} \mathrm{hex}(d_i)\big), \qquad \text{chain head} = h_n$$

Each stored row keeps its own digest and chain value; each batch keeps its head.

**Proposition (tamper evidence).** Given an authentic chain head hₙ, any record sequence that differs from the original — by an edit, a deletion, an insertion or a reordering — yet re-hashes to the same hₙ yields an explicit SHA-256 collision.

*Proof sketch.* Walk both chains backwards from the shared head. Every link hashes a **fixed-width 128-character string** (two 64-character hex digests), so the concatenation is unambiguous, and two different inputs with one output are a collision by definition. So at each step, either we have found a collision, or the previous chain value and the row digest are equal — and if the rows differ while their digests are equal, that is itself a collision. If the sequences differ in length, the shorter chain reaches SHA-256(ε) while the longer produces that same value from a non-empty input: again a collision. ∎

With a 256-bit output, a generic collision search costs about 2¹²⁸ operations by the birthday bound. Verification re-hashes every row and re-walks every chain, then reports the **first breach** with its expected and actual hashes: `content_altered`, `link_broken` or `chain_recomputed`.

**Why no blockchain network?** Evidence in a law-enforcement deployment must stay inside the deployment. Replicating a victim's bank rows onto a public ledger would be a privacy disaster, and a permissioned chain across agencies that do not yet share infrastructure is a procurement problem, not a code problem. We built the integrity guarantee that is enforceable today. **We are explicit about its limit:** the proposition assumes an authentic head, and today the head lives in the same database as the rows. Anchoring heads externally is the first item on our roadmap.

**The tamper drill.** A supervisor types a confirmation phrase; the server rewrites one stored amount — ₹4,80,000 becomes ₹1 — without touching its hash. Re-run verification and watch it fail on that exact row. A restore token undoes it, and both actions land in the audit log. The page's subtitle is our philosophy: *"This page exists to be tested, not believed."*

The same SHA-256 primitive is reused five more times: citation verification, reasoning-ledger claims, case-reconstruction steps, file de-duplication, and **pinning the ML model's weights** (a swapped model file is refused at load).

### 2. From identifiers to people — union-find entity resolution

Records contain identifiers; investigations are about people. Four deterministic rules decide which identifiers belong together:

- **R1** — a phone number and the handset it was used in are one person (every CDR and IPDR pair)
- **R2** — an account belongs to its registered holder
- **R3** — identifier groups sharing a subscriber registration merge
- **R4** — a social handle belongs to the holder of the number it is linked to

The merges run on a disjoint-set forest with **union-by-size and full path compression**.

**Theorem (Tarjan, 1975).** With union-by-size (or rank) and path compression, any sequence of m operations on n elements runs in

$$T(m, n) = O\big(m \, \alpha(n)\big)$$

where α is the inverse Ackermann function — α(n) ≤ 4 for any n that fits in the physical universe. Resolution is effectively linear.

Every identifier stores **which rule attached it**, so an analyst can challenge *why* two identifiers became one person. Groups that match no registered subscriber become "Unregistered subscriber Pnnnn": never silently dropped, never silently merged.

### 3. The join — cross-source correlation

```
 t = 0          call    handler --> victim                 CDR00019
 t + 11 min     debit   victim  --> mule    Rs 4,80,000    TXN00086
                        ^-- DETECTOR 1: call -> debit coupling
 t + 11..41 min         mule --> 6 accounts (96% onward)
                        ^-- DETECTOR 2: fan-out / pass-through
                        accounts --> ATM cash-outs
 throughout             one handset, 4 SIMs
                        ^-- DETECTOR 3: SIM rotation
```

**Call-to-debit latency.** Latency runs from the end of the call, unless the money left while the caller was still on the line — then it runs from the start of the call, and that call supersedes all earlier explanations:

$$\ell = \begin{cases} t_{\mathrm{debit}} - t_{\mathrm{end}} & \text{if } t_{\mathrm{debit}} \ge t_{\mathrm{end}} \\ t_{\mathrm{debit}} - t_{\mathrm{start}} & \text{if } t_{\mathrm{start}} \le t_{\mathrm{debit}} < t_{\mathrm{end}} \end{cases}$$

which feeds a speed score over a 3-hour window, saturating at 10 minutes:

$$s = \min\left(1,\; 1 - \frac{\ell_{\min} - 600}{10800 - 600}\right)$$

**Pass-through ratio.** For a credit of amount a_c, over the debits D leaving the same account within 30 minutes:

$$\rho = \frac{\sum_{d \in D} a_d}{a_c}, \qquad \text{flagged when } 0.7 \le \rho \le 1.5$$

Money in, money straight out: the signature of a mule.

**Counterparty stability** protects busy, legitimate businesses:

$$\sigma = 1 - \frac{\lvert \text{distinct payees} \rvert}{2 \cdot N_{\mathrm{debits}}}$$

A bakery paying the same five suppliers every week scores high. A mule fanning out to fresh accounts scores low.

### 4. CGNAT — why an IP address is weak evidence

Mobile operators put thousands of subscribers behind a single public IPv4 address using **Carrier-Grade NAT**, drawn from the shared address space **100.64.0.0/10 (RFC 6598)**. A naive system that treats "two people shared an IP" as a link will accuse an entire neighbourhood.

Our own seed data shows it: the busiest "public" address in the IPDR, **100.91.29.149**, is a CGNAT address carrying 145 sessions from six different subscriber numbers.

TRACE X treats shared addresses accordingly:

- **Campaign hunt** weights an IP-sharing link by how crowded the address is. If k entities sit behind one address, each pairing is worth
$$w_{\mathrm{ip}} = \frac{1}{k - 1}$$
so a two-person address is strong evidence (1.0) and a six-person address is weak (0.2).
- **Cross-border exposure** geolocates public IPs against national allocations, but **explicitly skips private and CGNAT addresses**. A CGNAT address says nothing about where a person is.
- **The schema is CGNAT-ready.** Every IPDR row carries its private IP, public IP and NAT source-port block — the three fields an operator needs to attribute a CGNAT session to one subscriber.

We are honest about what is not done yet: port-level attribution (matching a destination-side timestamp and source port to one subscriber's NAT port block) is on the roadmap, and one scoring feature still counts shared addresses without discounting CGNAT (see *Challenges*).

### 5. The model stack — six models, one decision

| # | Model | Role |
|---|---|---|
| 1 | **XGBoost**, monotone-constrained | Primary risk classifier |
| 2 | **Isotonic regression** | Calibration: makes "0.87" mean 87% |
| 3 | **Logistic regression** | Baseline: proves the trees earn their complexity |
| 4 | **Hand-weighted rule scorer** | Transparent fallback when artifacts fail their integrity check |
| 5 | **TreeSHAP** | Exact per-entity attribution |
| 6 | **Majority class** | The floor, published to keep everyone honest |

Each entity is described by **18 engineered features**: identifier counts, shared IPs, SIMs per handset, call-to-debit coupling and speed, pass-through ratio, fan-out width and value, transaction counts, counterparty breadth and stability, velocity, and whether this person *called* people who then moved large sums.

**Objective.** Gradient-boosted trees minimise a regularised logistic loss:

$$\mathcal{L} = \sum_{i=1}^{n} \Big[ -y_i \log p_i - (1 - y_i) \log(1 - p_i) \Big] + \sum_{k=1}^{K} \Big( \gamma T_k + \tfrac{1}{2} \lambda \lVert w_k \rVert^2 \Big), \qquad p_i = \sigma\big(F(x_i)\big)$$

A 5-fold stratified grid search on log-loss selected **K = 320 trees of depth 4, λ = 1.0, learning rate 0.06** (cross-validated log-loss 0.0616 ± 0.0063).

**Monotone constraints — domain knowledge the data is not allowed to unlearn.** For six features (shared IPs, SIMs per handset, pass-through ratio, fan-out width, coercive calls placed, and money-moving callees) we enforce

$$x_j \le x'_j \;\;\text{and}\;\; x_{-j} = x'_{-j} \;\;\Longrightarrow\;\; F(x) \le F(x')$$

Without this, a quirk in any finite sample could teach the model that *more* pass-through makes someone *safer* — indefensible to a reviewer and exploitable by an adversary. Counts and velocity are deliberately left free, because a busy legitimate business looks busy too.

**Calibration.** Isotonic regression finds the best non-decreasing map from raw probability to observed frequency, solved exactly by the Pool Adjacent Violators Algorithm:

$$g^{\star} = \arg\min_{g \ \text{non-decreasing}} \; \sum_{i} \big( y_i - g(z_i) \big)^2$$

The served score is clipped so that no model of this kind ever claims certainty:

$$\hat{p}(x) = \mathrm{clip}_{[0.005,\, 0.995]} \Big( g^{\star}\big( \sigma(F(x)) \big) \Big)$$

**Proposition (monotonicity survives calibration).** σ is strictly increasing, g⋆ is non-decreasing (it interpolates a non-decreasing step table), and clipping is non-decreasing. A composition of non-decreasing functions is non-decreasing, so the **served, calibrated score** is still monotone in every constrained feature. ∎ A test in our suite checks this empirically.

**Explanations — exact Shapley values.** The Shapley value is the unique attribution satisfying efficiency, symmetry, dummy and additivity (Shapley, 1953):

$$\phi_j = \sum_{S \subseteq N \setminus \{j\}} \frac{\lvert S \rvert! \,\big(\lvert N \rvert - \lvert S \rvert - 1\big)!}{\lvert N \rvert!} \Big[ v\big(S \cup \{j\}\big) - v(S) \Big]$$

TreeSHAP computes these **exactly** for tree ensembles in polynomial time, O(T·L·D²) (Lundberg et al., 2020). Efficiency guarantees the explanation adds up to the model's actual output:

$$F(x) = \phi_0 + \sum_{j=1}^{18} \phi_j$$

Our test suite verifies this identity numerically. Every score on screen shows its six largest drivers.

**Measured results.** On a held-out split of 1,200 synthetic entities, evaluated once:

| Model | ROC-AUC | PR-AUC | Precision@0.33 | Recall@0.33 | Brier |
|---|---|---|---|---|---|
| **Trained XGBoost** | **0.998** | **0.994** | **0.944** | **0.981** | **0.018** |
| Logistic regression | 0.960 | 0.929 | 0.807 | 0.869 | 0.068 |
| Rule scorer | 0.776 | 0.506 | 0.422 | 0.939 | 0.299 |

$$\mathrm{Brier} = \frac{1}{n} \sum_{i=1}^{n} (\hat{p}_i - y_i)^2 \qquad\qquad \mathrm{ECE} = \sum_{b=1}^{10} \frac{\lvert S_b \rvert}{n} \, \big\lvert \bar{y}_b - \bar{p}_b \big\rvert = 0.0123$$

**How we read these numbers — and how you should.** There is no public labelled dataset of real mule accounts, and there should not be one in a public repository. So we trained on **8,000 entities drawn from 12 hand-authored archetypes**: mules, layering accounts, handlers, SIM farms and associates, alongside the legitimate look-alikes a naive detector flags — busy businesses, charities, households sharing a handset, roaming travellers, victims and bystanders. The distributions deliberately overlap. The table above measures fit to that sampler, so it is **optimistic by construction** and is not a real-world accuracy claim. Our only out-of-sampler check is the seeded case: on 18 graded entities the model reaches **ROC-AUC 0.958 with recall 0.50** — and we publish that number next to the flattering one.

**Integrity and reproducibility.** Artifacts are plain JSON (no pickle is ever deserialised), their SHA-256 digests are recorded in the model card and verified at load, and training is seeded: our tests retrain twice and require **bit-for-bit identical** artifacts. A runtime switch (`TRACEX_SCORER`) selects trained, rule-based or replayed scores, and a **shadow evaluation** scores every entity with the trained model beside whichever scorer is in force, so disagreement is visible instead of hidden.

### 6. The agent mesh — three agent systems

"Multi-agent" here means three genuinely different architectures, each chosen because the others would be wrong for the job.

**① The eight-role pipeline, with a critic built in.** *Planner → Investigator → Correlator → Analyst → Critic → Verifier → Responder → Auditor.* The Analyst opens a hypothesis for every flagged entity with its risk score as the prior. The **Critic's only job is to argue against the Analyst**: it runs the exculpatory checks and links every innocent explanation as contradicting evidence. Confidence moves in bounded steps:

$$c \leftarrow \mathrm{clip}_{[0,1]}\big( c \pm 0.2 \, w \big), \qquad \text{hypothesis refuted when } c < 0.2$$

A refuted subject is skipped by the Responder, so **the system can talk itself out of accusing someone**. If a language model writes the final report, the report is rejected if it contains any figure not present in the facts: *a model may choose words, never numbers.*

**② The response agent: arguing with itself on purpose.** OBSERVE → HYPOTHESIZE → INVESTIGATE → DECIDE → SCORE → PROPOSE. It holds **seven rival hypotheses at once**: mule, handler, SIM farm, **victim**, legitimate activity, shared household phone and bystander. It gathers evidence from **eight sources**, and a failing source is reported, never hidden. Crucially, a large debit minutes after an inbound call counts as evidence of being a **victim**, not a mule: the agent is designed to consider that the person may be the one being hurt.

It also computes a **"Fraud DNA"** fingerprint: twelve behavioural traits (burst intensity, night activity, fan-out breadth, repeat targeting, short-call ratio, URL density, urgency language, authority language, money requests, identifier rotation, geographic spread, pass-through), hashed into a `DNA-XXXXXXXXXX` signature. Two entities are linked to the same campaign when

$$\cos(u, v) = \frac{u \cdot v}{\lVert u \rVert \, \lVert v \rVert} \ \ge\ 0.82$$

which catches networks that **operate the same way while sharing zero identifiers** — exactly what happens after a ring burns its SIMs and opens fresh accounts. The agent then proposes six possible actions, each with its benefit, its **downside** and its reversibility. A freeze or a suspicious-activity report requires a supervisor, **enforced by the server**.

**③ The LLM tool-use investigator.**

```
 objective
    |
    v
 +--------------------------------+     unknown tool / bad args
 | model chooses tools (<=4/turn) |---> refused, counted, told why
 +---------------+----------------+
                 v
   9 READ-ONLY tools --> evidence --> injection strings REDACTED
                 |                          |
                 +<-------- fed back -------+    (<= 8 steps)
                 v
   answer with [record-id] citations after every factual sentence
                 v
          +-------------+   fails   +-------------------------+
          | TRUTH GATE  |---------->| exactly ONE repair round|
          +------+------+           +-----------+-------------+
            pass |                   still fails: reported as
                 v                   NOT GROUNDED, never hidden
   conclusion written to the reasoning ledger --> a human decides
```

- **Providers:** Claude (Messages API), a local Ollama model, or a deterministic planner that runs the same loop without a model. `auto` mode **never selects a cloud provider, even when an API key is present** (a test enforces this). Cloud use requires explicit opt-in, and the UI states that evidence then leaves the machine.
- **Zero blast radius:** an import-time assertion guarantees every registered tool is read-only and no tool name begins with a write verb. **The agent physically cannot act.**
- **Prompt-injection defence:** evidence contains text written by scammers. Instruction-like strings inside tool output are **redacted before the model sees them**, and counted.
- **Graceful failure:** if a provider errors mid-run, the run restarts on the deterministic planner and is labelled `degraded`.

### 7. The truth gate — hallucination defeated by arithmetic

Every checkable sentence is decomposed into its cited records R and the figures, identifiers and timestamps it asserts, Fig(c). The verdict is evaluated in order:

$$V(c, R) = \begin{cases} \textsf{fabricated} & \text{if some } r \in R \text{ was never ingested} \\ \textsf{tampered} & \text{if some } r \in R \text{ no longer re-hashes to its ingest digest} \\ \textsf{unsupported} & \text{if } \mathrm{Fig}(c) \not\subseteq \mathrm{Fig}(R) \\ \textsf{verified} & \text{otherwise} \end{cases}$$

No model sits in this path, and nothing leaves the deployment to check it. A citation that resolves to nothing is a **fabrication, not a judgement call**. Uncited sentences are marked *uncited* or *unverifiable*, never "false". And an honest *"I can't answer that from the data"* is scored **not_a_claim** instead of being penalised, because punishing refusals teaches an agent to guess.

The verifier grades itself in public: nine hand-built traps (fabrications, a tampered record, unsupported figures, faithful claims that must pass, and honest refusals). **9/9 correct, fabrication recall 1.00.** The chat screen has a **"Try a fabrication"** button, so a judge can attack it live.

### 8. The devil's advocate — the exculpatory engine

Before any conclusion about a person, three checks argue for the defence: *stable counterparties* (a business settling known suppliers), *fixed beneficiary, no onward movement* (a routine remittance), and *long-lived SIMs without money coupling* (a shared household phone). Each compares the entity to a legitimate reference pattern:

$$\mathrm{sim}(v, v^{\mathrm{ref}}) = \max\left(0,\; 1 - \frac{\lvert v - v^{\mathrm{ref}} \rvert}{v^{\mathrm{ref}}}\right), \qquad r_k = W_k \cdot \mathrm{fit}_k \cdot e_k, \qquad W = (0.30,\ 0.25,\ 0.20)$$

$$R = \min\Big(\sum_k r_k,\ 0.6\Big), \qquad \hat{p}_{\mathrm{adjusted}} = \hat{p} \, (1 - R)$$

The 0.6 cap is a design decision: *checks can lower a machine score but never zero it out. Final assessment requires human review.*

### 9. Campaign hunt — ranking rings, not individuals

A ring of ten individually quiet accounts never reaches the top of a risk queue. So the hunt builds an **evidence graph** — synchronised ATM withdrawals within 30 minutes, direct fund transfers, and shared addresses weighted by 1/(k−1) — and extracts connected components. Each campaign of three or more members gets:

$$\mathrm{confidence} = \gamma \cdot \frac{1}{\lvert E^{\star} \rvert} \sum_{(a,b) \in E^{\star}} \max_{\ell \in L_{ab}} w_\ell, \qquad \mathrm{cohesion} = \frac{\lvert E^{\star} \rvert}{\binom{n}{2}}$$

where γ = 0.7 when most links are standing payment relationships, and 1 otherwise. A standing relationship over four or more days is itself down-weighted to 0.35, because **routine commerce generates more transfers than fraud does**. A detector that forgets this accuses every small business in the city.

### 10. Space and time — Investigation Eye, pattern of life and God's Eye View

**Investigation Eye** reconstructs the case as **128 ordered steps** (61 controlling calls, 48 coerced transfers, 9 layering hops, 7 funds-received events and 3 cash-outs), each resolving to a hashed source row. A **time machine** slider lets you drag through the incident and watch it assemble on the map, with filters to follow the money or follow the device.

**Pattern of life** joins licence-plate reads to cash-outs, and judges each corroboration **against the plate's own baseline**: a car that passes that camera every day is *weak* evidence however close in time it sits. Impossible travel is detected with the haversine distance:

$$d = 2R \arcsin \sqrt{ \sin^2\!\left(\frac{\Delta\varphi}{2}\right) + \cos\varphi_1 \cos\varphi_2 \sin^2\!\left(\frac{\Delta\lambda}{2}\right) }, \qquad R = 6371.0088 \text{ km}$$

A speed above 200 km/h between two plate reads means a misread or a **cloned plate**. Above 900 km/h between cell sites is flagged as a contradiction.

**God's Eye View** is a live 3D globe built on **CesiumJS**:

- **Live aircraft** from OpenSky ADS-B, polled every 15 seconds. OpenSky's CORS policy blocks browsers, so we built a same-origin server proxy with a 9-second cache that respects its rate limit.
- **Satellites** from CelesTrak two-line elements, propagated **client-side with the SGP4 model** every second, converted from inertial to geodetic coordinates, and de-duplicated by NORAD catalogue number.
- **Esri World Imagery** as the base map.

Vessel and camera layers are registered but **deliberately empty**, labelled *"Needs a feed key this deployment does not hold."* We would rather show nothing than invent a ship.

Most importantly, **God's Eye View is walled off from evidence**. Its header says so plainly: *nothing on this globe is hash-chained or resolves to an ingested record, and no finding should rest on it.* A live globe is persuasive, but persuasive is not admissible, and blurring the two would teach investigators a dangerous habit.

### The stack

**Backend:** Python 3.12, FastAPI, Pydantic, SQLite (WAL), httpx, pypdf, openpyxl. **ML:** XGBoost, scikit-learn, NumPy, pandas, TreeSHAP. **Frontend:** Next.js 14, React 18, Tailwind CSS, D3.js, CesiumJS, satellite.js. **AI providers:** Claude API, Ollama. **Crypto:** SHA-256 hash chains, HMAC-SHA-256 session tokens. **Quality:** 51 automated tests plus a 573-response regression harness.

## Challenges we ran into

**Our first model cheated.** Version one of the sampler produced a near-perfect score — and ranked *transaction volume* as the most important feature. It had learned "busy means legitimate", because every synthetic business was busy and every synthetic mule was quiet. That is not just a shortcut; it is an **evasion route**, because a high-volume mule would walk straight through. We rebuilt the sampler with high-volume mules and pass-through-shaped legitimate businesses, reported the residual dependence in the model card, and **stopped tuning** once the numbers were honest rather than when they looked best.

**Train/serve skew we could measure but not erase.** Four of our eighteen features are approximations of definitions we could not fully reconstruct. Rather than hide it, the training run measures it: 17 of 20 entities have at least one differing feature, with a mean score change of 0.11. That gap is why the out-of-sample number (0.958) sits below the captured-vector number (1.000), and we publish both.

**A rounding error that would have been a fabrication.** Our deterministic answer engine once formatted ₹7,852.50 as ₹7,853. Harmless-looking — except the verifier then correctly marked the claim *unsupported*, because ₹7,853 appears in no record. Our own no-fabrication test caught it. Money is now formatted exactly, never rounded.

**The globe that never loaded.** God's Eye View hung on its spinner for three separate reasons: OpenSky blocks browsers via CORS, CelesTrak returns HTTP 403 on its heaviest catalogue endpoint, and an iframe's load event can be missed entirely. The fixes: a same-origin proxy, CelesTrak's curated visual group, and **three independent safeguards** (the iframe load handler, a 6-second fallback timer, and a 15-second in-page watchdog that explains the failure instead of spinning forever).

**CGNAT made one of our own features wrong.** `n_ips_shared` counts addresses a person shares with others — and it is monotone-constrained upward. Behind CGNAT, innocent strangers share addresses constantly, so in a real deployment that feature would over-flag. Our campaign hunt and geolocation already discount CGNAT. The scoring feature does not yet, and fixing it properly needs port-level attribution. We would rather name this than bury it.

**Writing the documentation broke our own claims — twice.** While documenting every screen against the code, we found the UI said *only a supervisor can close a case* while the server did not enforce it. We fixed it and added tests. While writing *this very story*, we found that our "O(α(n))" union-find claim was false: we compressed paths but did not union by size, which gives O(log n), not α(n). We added union-by-size and verified that the resolved output is **byte-identical before and after** (same SHA-256 fingerprint). Lesson learned: documentation is a test suite.

## Accomplishments that we're proud of

- **An integrity system that breaks itself on demand.** The tamper drill turns "trust us" into "watch it fail".
- **An AI that cannot lie about the record.** Fabrication recall of 1.00 on our trap set, a button that lets anyone attack it live, and read-only tools that make acting impossible, not merely discouraged.
- **A model we can defend line by line:** monotone-constrained, calibrated (ECE 0.0123), exactly explained, reproducible bit-for-bit, integrity-pinned, and shipped with a model card listing seven limitations.
- **Publishing the bad numbers.** Benchmark recall on the seeded case is **0.50**: six planted fraud entities are missed. It is on the benchmark page, in our README and in the model card. Zero of five planted decoy businesses were falsely flagged.
- **Scale of the build:** 124 API endpoints, 36 screens, 19 analytics modules, three agent systems, 125 compliance controls re-evaluated live on every page load, and **51 passing tests**.
- **Court-oriented by design:** an electronic-record certificate under Section 63 of the Bharatiya Sakshya Adhiniyam, supervisor-only evidence packages, dispositions that store **the score they were taken against** so later re-analysis cannot rewrite history, and an append-only audit log whose actor comes from the authenticated identity, never from the client.

## What we learned

- **Accuracy is not calibration.** A model that ranks well can still lie about probabilities. Isotonic calibration made "0.87" mean 87%, and it mattered more to investigators than another point of AUC.
- **Shortcut learning is a security vulnerability**, not just a statistics bug. If a model can be fooled by volume, a criminal can be too.
- **Grounding has to be arithmetic.** Asking a second model "is this true?" just moves the hallucination. Resolving the citation and re-hashing the record cannot be argued with.
- **An IP address is not a person.** CGNAT turns naive network evidence into collective suspicion. Evidence strength must shrink as an address gets crowded.
- **"AI proposes, humans decide" must be enforced in code.** Read-only tools, server-side role checks and an import-time assertion are what make the slogan true.
- **Honesty is a feature.** Every limitation we published made the rest of the system more believable.

## What's next for TRACE X — Evidence-Grounded AI for Cybercrime Investigation

- **Tamper-evident → tamper-proof.** Anchor each batch's chain head externally, via OpenTimestamps on Bitcoin and RFC 3161 timestamp authorities, so the head no longer lives beside the data it protects. Permissioned cross-agency anchoring is the one place a distributed ledger genuinely earns its cost.
- **Port-level CGNAT attribution.** Match destination-side timestamps and source ports to each subscriber's NAT port block, then rebuild the shared-IP feature on attributed sessions instead of raw addresses.
- **Graph learning.** Message-passing neural networks over the behaviour graph, benchmarked against XGBoost with the same honest external validation, and not shipped unless they win.
- **Temporal models** that learn the *order* of events, not just aggregate features.
- **Live LLM validation.** Our Claude and Ollama paths are tested against wire-protocol mocks. Next, we run them against live models and publish the measured grounding rate.
- **A multi-judge verifier** that reports disagreement between independent judges rather than averaging it away.
- **Governed real data** with a fairness and subgroup review, which synthetic data cannot support. Plus production SSO/OIDC and streaming ingest that preserves per-record chaining.

**Known limits, stated plainly:** all data here is synthetic; the model has never seen real casework; the hash-chain head is not yet externally anchored; authentication is a demo mechanism; and live LLM paths are protocol-tested only.

> *A score is a lead for a human reviewer, never a verdict.* Every component of TRACE X exists to make that sentence **structurally true** rather than merely printed in a disclaimer.

*AI disclosure: this project was built with substantial assistance from Claude (Anthropic) via Claude Code. The full disclosure is in the README, section 19.*
