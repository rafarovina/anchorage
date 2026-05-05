# PRD

> The technical north star: data model, interfaces, governance machinery, calibration, credit, adversary testbed. Where the [manifesto](./manifesto.md) explains *why* and [governance](./governance.md) sketches *who decides what*, this document specifies *how*.

This is a **design document**, not implementation specification. It captures commitments precise enough to build from, with parameters deliberately deferred to the adversary-testbed phase where they will be tuned against simulation rather than guessed.

---

## Architecture overview

Anchorage is one canonical service layer behind two client surfaces:

```
                         ┌─────────────────────────────┐
                         │   Canonical service layer   │
  ┌──────────────────┐   │  ─ graph store              │   ┌──────────────────┐
  │  MCP server      │◀─▶│  ─ write-path tools         │◀─▶│  Web UI          │
  │  (mcp.…)         │   │  ─ verification engine      │   │  (anchorage.…)   │
  │  contributor /   │   │  ─ governance machinery     │   │  read & browse   │
  │  agent surface   │   │  ─ reputation & calibration │   │  for humans      │
  └──────────────────┘   │  ─ projection engine        │   └──────────────────┘
                         └─────────────────────────────┘
```

- **MCP server** (`mcp.anchorage.science`): primary write-path interface. Contributors, agents, and the simulated populations in the testbed all connect here. Tools are typed; verification is server-side.
- **Web UI** (`anchorage.science`): read-mostly human surface. Browse causes, sub-topics, graphs, frontiers, and manuscript projections. Calls the same canonical service layer.
- **Service layer**: the trust boundary. Every mutation passes through verification, governance gates, and reputation updates here. Clients are untrusted regardless of identity.

This is a deliberate architectural commitment, not an implementation note. Federation between Anchorage instances later is MCP-to-MCP. The testbed connects via the same MCP interface real clients use — *no stub APIs*. The architectural commitment is what the testbed depends on; the broader claim that simulated populations cover the real contributor distribution is qualified in the [manifesto](./manifesto.md#testability-is-the-secret-weapon) and the [testbed coverage section](#what-the-testbed-does-not-cover).

---

## Data model

### Multi-scale structure

Three layers, each with the shape suited to its job:

- **Cause** — the umbrella unit of belonging ("colon cancer"). Causes are created by maintainers; they are not user-creatable in v0.
- **Sub-topic** — scope envelope within a cause ("ctDNA-MRD in stage II resected CRC"). Sub-topics are first-class objects: they have IDs, descriptions, scope-envelope queries (e.g. PubMed search definition), creation status (proposed / active / archived), and curator approval state. In v0 sub-topic creation requires curator approval; later phases admit auto-discovery from graph state.
- **Claim graph** — the per-sub-topic structure where work happens.

A claim node belongs to exactly one sub-topic. Cross-sub-topic relationships are expressed via designated *cross-link edges* (see Edges).

### Nodes

Every node has:

- `id` — opaque identifier
- `sub_topic_id` — the sub-topic this node belongs to
- `kind` — one of:
  - `anchor` — external source (paper, dataset, definition). Has `external_ref` (PMID, DOI, URL) that must resolve.
  - `excerpt` — tight claim tied to a specific anchor parent. Has a `quoted_span` field; the verification engine matches it against the resolved source. Excerpts cannot exist without a verified span. The `content` is the atomic claim; the `quoted_span` is the verbatim slice that anchors verification. They are not required to be identical — `content` may paraphrase to atomize a claim the span supports — but `content` must be an assertion the span supports under a charitable reading. The verification engine confirms the span resolves; the reviewer confirms `content` follows from it. Span verification is necessary but not sufficient: cherry-picking a true span out of context (negation-stripping, hedge-stripping) is a known attack and is part of the reviewer's responsibility to catch.
  - `synthesis` — explicit inferential step derived from multiple parents. The agent or contributor sets `kind=synthesis` when the content is not a straight excerpt.
  - `open_question` — scoped uncertainty with edges to what it depends on. Surfaces as a frontier item.
- `content` — the claim text (single atomic claim per node)
- `external_ref` — for anchors, structured pointer (PMID/DOI/URL); for excerpts, references the parent anchor's external_ref
- `quoted_span` — for excerpts, the verifiable span text + offset within the source
- `status` — `staged` (under review), `active` (merged), `superseded` (replaced), `rejected` (review failed)
- `created_by`, `created_at`, `updated_at`

The **active node rule** (matching Galleon's contract): a node is *inactive* if it is the `from` end of a `supersedes` edge.

### Edges

- `derives` — parent (support) → child (derived claim). Direction matches storage. Lineage walks backward along `derives` until it hits anchors. `derives` parents must be `active` at the moment of acceptance: a child cannot be merged with a `staged` or `rejected` parent.
- `supersedes` — old → replacement. Marks the old node inactive. The `to` end (the replacement) must be `active` at the time the supersedes is proposed. Supersedes cycles (A → B → C → A) are forbidden by the verification engine.
- `cross_link` — explicit reference between sub-topics within the same cause. Used sparingly to express that a claim in one sub-topic relates to a claim in another. Cross-links are *navigation*, not *load-bearing structure*: they do not propagate `derives` lineage across sub-topic boundaries, and they cannot appear in the lineage chain a manuscript projection walks for credit or argument structure. If a projection in sub-topic A genuinely depends on a claim from sub-topic B, the corresponding excerpt or synthesis must be materialized as a node in A with its own `derives` chain. This closes the seam where cross-links could be smuggled `derives` that escape sub-topic-scoped review and credit.

Other edge types are rejected. The minimal vocabulary is load-bearing: more edge types create more places for governance disputes without proportionally more expressive power.

### Sub-topic relocation

A node belongs to exactly one sub-topic, but scope envelopes overlap and contributors will sometimes propose nodes in the wrong place. A `propose_relocate` operation moves a node (and its sub-topic-internal edges) from one sub-topic to another within the same cause, subject to curator approval. Relocation is not a free action: it touches credit, review eligibility, and projection scope, all of which depend on which sub-topic a node lives in. Genuinely cross-relevant nodes use cross-links rather than relocation, with the constraint above (cross-links cannot be projection lineage).

### Manuscript projection

A *projection* is a derived view of a sub-topic's graph plus editorial choices (section order, narrative voice, scope of inclusion). Projections are not a separate truth ledger — they are a function of (graph state, projection config). The graph is canonical; projections come and go.

The *projection config* is itself a governance artifact, not a private editorial document. Changes to a projection config — what's in scope, section ordering, which nodes are emphasized — affect which nodes are load-bearing for argument structure and therefore which contributors get credit. Projection configs are version-controlled in the graph; changes to them route through the standard governance-change CI process; authorship disputes resolve as governance changes to the projection config rather than as private negotiations.

---

## MCP tool surface

The MCP server exposes a small, verification-heavy set of tools. The minimum write surface is:

### Write-path tools

- **`propose_anchor`** `{ cause_id, sub_topic_id, content, external_ref }` → `{ proposal_id }`
  - Creates a staged anchor node. Server fetches `external_ref`, confirms resolution, and rejects on failure.
- **`propose_excerpt`** `{ cause_id, sub_topic_id, parent_anchor_id, content, quoted_span }` → `{ proposal_id }`
  - Creates a staged excerpt node. Server matches `quoted_span` against the resolved source. Mismatch → rejection. No exceptions.
- **`propose_synthesis`** `{ cause_id, sub_topic_id, parent_ids, content, kind }` → `{ proposal_id }`
  - Creates a staged synthesis or open_question node with `derives` edges from each parent. Atomic — either all edges create or none do. `kind` is `synthesis` or `open_question`.
- **`propose_supersedes`** `{ from_node_id, to_node_id, rationale }` → `{ proposal_id }`
  - Stages a supersedes edge with the reasoning attached.
- **`propose_cross_link`** `{ from_node_id, to_node_id, kind }` → `{ proposal_id }`
  - For cross-sub-topic references within the same cause.
- **`cast_review_vote`** `{ proposal_id, decision, rationale }` → `{ vote_id }`
  - `decision` is `accept`, `reject`, or `revise`. `rationale` is required and may itself be promoted to a graph node (typically `open_question`) by curators.
- **`propose_sub_topic`** `{ cause_id, name, description, scope_query }` → `{ proposal_id }`
  - Subject to curator approval in v0.

### Read-path tools and resources

Read-path is largely MCP *resources* (passive), with a few active tools for queries:

- **Resource: `cause://...`** — list of causes; structured cause metadata.
- **Resource: `sub-topic://{id}`** — sub-topic metadata, status, scope query, recent activity.
- **Resource: `node://{id}`** — node + immediate neighbors.
- **Resource: `subgraph://{sub-topic-id}`** — full or filtered subgraph in a structured form.
- **Tool: `query_frontier`** `{ cause_id?, sub_topic_id?, frontier_kind? }` → ordered list of frontier items (work to be done).
- **Tool: `query_proposals`** `{ status?, sub_topic_id?, assigned_to_me? }` → list of proposals matching filter.
- **Tool: `fetch_calibration_batch`** `{ sub_topic_id }` → reviewer's review batch (real items + calibration items, indistinguishable from each other).

Tool surface is intentionally small. Each tool has tight typing, server-side validation, and clear failure modes.

---

## Verification engine

The verification engine is the security boundary. Every write tool routes through it.

- **Anchor verification.** External references must resolve. PMIDs hit NCBI E-utilities; DOIs resolve via Crossref; URLs must return 200 with substantive content. Anchors are *content-addressed*: the hash of the fetched content is stored alongside the `external_ref`, and re-verification compares against the stored hash rather than only against a live fetch. URL-anchors are second-class — metadata-unstable and cloaking-prone — and may be subject to stricter regimes (or refused entirely in v0). When re-verification fails (retraction, content drift, host gone), the anchor moves to an `unresolvable` status and surfaces as a frontier item rather than silently rotting.
- **Span verification.** For excerpts, the `quoted_span` must be a substring of the fetched source after normalization (whitespace, quote-style, and a small set of typographic equivalences specified in the verification spec, not left to "light normalization" hand-waving). Failure rejects the proposal at write time, not at review time. Span verification confirms the quote exists; it does not confirm the proposed `content` follows from it — that is the reviewer's job.
- **Lineage validation.** `derives` edges must connect nodes within the same sub-topic; `cross_link` edges may cross sub-topics within the same cause. At acceptance, every `derives` parent must be `active` (not `staged`, `rejected`, or `superseded`); the `to` end of a `supersedes` must be `active` at proposal time; supersedes cycles are rejected. Cross-links cannot appear in the lineage chain a manuscript projection walks.
- **Reputation gates.** Some operations require minimum reputation (per-(cause, sub-topic) or per-cause). Below the threshold, proposals land staged but are not advanced into the review queue without curator action. Specific thresholds are tuned in the testbed.
- **Rate limits and abuse signals.** Per-identity rate limits on proposals; suspicious patterns (sudden burst of proposals, calibration-failure clustering) flag for curator review. Specific signals are operationally private.

---

## Governance machinery

### The contribution flow

1. **Propose** — write-path tool creates a *staged* node or edge.
2. **Verify** — verification engine accepts or rejects synchronously based on grounding/lineage/rate.
3. **Stage** — accepted proposals enter the review queue. Visible and citable as proposals; not part of the canonical graph.
4. **Assign** — N reviewers are randomly selected from the eligible pool, salted with calibration items drawn from the graph's own validated history.
5. **Review** — reviewers vote with rationale via `cast_review_vote`.
6. **Resolve** — convergent vote merges; divergent vote routes to a richer review path (more reviewers, curator escalation, or carrying the divergence forward as parallel synthesis nodes / `open_question`).
7. **Settle** — reputation updates for the contributor and the reviewers, weighted by outcome correctness.

### Reviewer assignment

Reviewers are drawn from the eligible pool — contributors with sufficient per-(cause, sub-topic) reputation — by stratified random sampling. Stratification balances reviewer expertise (where measurable) and reduces collusion risk. Specific stratification weights are tuned in the testbed.

Narrow sub-topics will sometimes have a sub-topic-rep pool too small to draw N reviewers from. The fallback ladder is fixed in design even though the thresholds are tuning:

1. **Sub-topic-rep first.** Standard path. Reviewers with demonstrated work in this sub-topic.
2. **Cause-rep with degraded-stratification flag.** When the sub-topic pool is exhausted, draw from cause-rep contributors and flag the proposal as "expertise-degraded" — visible to the contributor, factored into convergence-threshold logic (see below), and logged for periodic audit.
3. **Curator escalation.** When even the cause-rep pool is insufficient or when prior steps have produced sustained divergence, escalate to curator review.

At sub-topic launch, expertise stratification is degraded by construction (no history exists), and calibration items are drawn from the cause's validated history rather than from the sub-topic's. The doc states this explicitly so it is not mistaken for a vulnerability when reviewers notice it.

Convergence and divergence thresholds are claim-class-aware: high-stakes claim classes (e.g., quantitative effect-size syntheses) draw larger pools and tighter convergence thresholds than low-stakes ones (e.g., terminological clarifications). Specific class definitions and threshold values are testbed-tuned; the *machinery* being class-aware is a design commitment.

Divergence has a closure mechanism. Divergent proposals are routed to richer review or carried forward as parallel synthesis nodes / `open_question`, but not indefinitely: divergences without further evidence within a tunable window are archived (status `unresolved-archived`) rather than perpetually re-routed. This prevents the queue from accumulating reviewer-noise as if it were principled disagreement.

### Calibration batches

Reviewer batches contain a mix of real proposals and calibration items. Calibration items are drawn from the graph's own validated history — proposals that survived multiple confirmations and have been stable. They are intended to be statistically indistinguishable from real frontier work in *the dimensions a reviewer can act on*: a reviewer evaluating one batch should not be able to tell which items are calibration. The harder question — whether a *patient* adversary observing many batches can build a classifier on re-use frequency, age, or other batch-level signatures — is real, and the methodology actively defends against it: calibration sampling is biased toward fresh-but-validated history, items rotate aggressively, and the sampling distribution itself is part of what the testbed evaluates as an attack surface.

Reviewers who fail calibration lose reputation; the calibration corpus grows as the graph grows.

**Specific calibration items in active rotation are operationally private.** Published items are burned. Methodology — including the rotation regime and the sampling-distribution defenses against batch-level correlation attacks — is fully public; specific tuning is not.

### Reviewer-as-staking

Reviewers gain reputation when they accept proposals that survive and reject proposals that get rejected by other reviewers. They lose reputation when they accept proposals that are later reverted or fail calibration. This makes lazy rubber-stamp review costly without requiring reviewers to do more than they would already do well. The risk that staking selects against reviewers willing to engage hard syntheses is addressed by claim-difficulty-normalized review-credit (see [Reputation](#reputation)).

### Sub-topic creation

In v0:

- Proposed via `propose_sub_topic`.
- Curator evaluates against the criteria in [seed-topic.md](./seed-topic.md): articulable scope envelope, real disagreement, real audience, manageable corpus size, low political risk.
- The system computes feasibility hints from existing graph state: corpus density (do anchors exist?), anchor coverage (how much of the proposed scope envelope already has nodes?), projected closure distance.
- Curator accepts as `active`, defers as `proposed`, or rejects.

In Phase 3+: auto-discovery surfaces tractable scope envelopes from graph state; curator review remains as a check.

---

## Identity

The identity model is a requirements sketch in v0; specific tech (OIDC providers, key formats, attestations) is a Phase 1 implementation choice, but the *contract* the rest of the design depends on is fixed here.

- **Bounded identities-per-real-person.** Identity creation has a non-trivial cost — email verification at minimum, third-party OIDC (GitHub, ORCID, institutional SSO) preferred. The cost is tunable; the testbed sweeps it as a parameter. Zero-cost identities are not supported.
- **Pseudonymity is supported; anonymity is not.** A contributor may operate under a stable pseudonym; the system retains a binding between the pseudonym and the underlying identity-establishing credentials (email, OIDC subject) that curators can use under documented escalation. The graph and the public surface show the pseudonym; the binding is private.
- **Named credit on manuscript projections is opt-in.** Pseudonymous credit is allowed, but the project's recommendation is real-name credit for high-impact projections to retain academic legibility. Pseudonymous co-authorship is unusual and contributors should make that choice deliberately.
- **Revocation.** Identities can be revoked (sybil farms, terms-of-service violations). Revocation invalidates future participation without rewriting graph history; revoked contributions remain in the graph with the revocation flagged.
- **Cross-cause anti-abuse.** Public reputation is per-cause (see below). Anti-abuse signals (rate-limit accounting, identity-clustering for sybil detection) are *global per identity*, with documented governance and audit. The asymmetry — per-cause reputation, global anti-abuse — is intentional: sybil farms working two causes are more detectable than ones working one, and the cost of opacity here is small relative to the defense it enables.

The identity model is the foundation that sybil-resistance, calibration integrity, and reputation accounting all rest on. None of them composes meaningfully without it.

---

## Reputation

Reputation is structured to resolve a real trilemma the design cannot wave away: *slow* decay rewards consistency (the design goal) but lets patient adversaries stockpile; *fast* decay neutralizes stockpiles but disenfranchises episodic experts (the part-time clinician is exactly the contributor we want); *review-as-staking* punishes lazy review but selects against accepting hard syntheses (which are riskier to stand behind). Acknowledging this directly:

- **Two-component reputation.** A *demonstrated-competence* component, slow-decay, gates eligibility tiers (who is in the reviewer pool at all). A *recent-activity* component, fast-decay, gates assignment (who is drawn for a given proposal). A patient adversary can stockpile competence but must remain currently active to be assigned — and visible activity is detectable.
- **Per-(cause, sub-topic).** Anchored at the cause level (the unit of belonging), refined by sub-topics actually worked in (the unit of expertise). Cause-level reputation gets a contributor in the door to *propose* in any sub-topic; review-eligibility in a contested sub-topic requires demonstrated work *in that sub-topic*. This closes the rep-laundering path where easy-sub-topic credibility is parlayed into reviewer authority over contested ones.
- **Earned through confirmed contributions and accurate reviews.** Both contributing nodes that survive *and* reviewing accurately count.
- **Lost through reverted contributions and inaccurate reviews.** Supersedes and rejected calibration items both decrease reputation. Self-supersedes (a contributor superseding their own node) do not count toward survivorship — only supersedes by other contributors do.
- **Review-credit normalized by claim difficulty.** Without normalization, the regime selects for reviewers who accept easy proposals. Difficulty proxies — review effort, prior divergence, sub-topic frontier-distance — weight review-credit so that engaging hard syntheses is not dominated by rubber-stamping easy ones.
- **Eligibility tiers public; numeric reputation private.** Contributors can see what tier they are in (and what gates the next tier); raw numbers are not leaderboards. Reviewers receive batch-level performance feedback after-the-fact, not in real time.
- **Non-transferable, non-monetizable.** Reputation is a coordination signal, not a token.
- **Specific formulas tuned in testbed.** Initial values are chosen for testbed simulation; production values are confirmed against attack-success-rate measurements.

---

## Credit

When a sub-topic produces a manuscript projection, contributor credit is computable from graph state.

The basic shape of the credit function:

- **Node provenance.** For each node included in the projection, who proposed it, who reviewed it, when.
- **Survivorship weighting.** Nodes that survived multiple supersedes events count more than nodes that barely survived one review.
- **Load-bearing weighting.** Nodes whose removal would break a projection chain count more than peripheral nodes.
- **Review weighting.** Reviewers who voted correctly accrue partial credit, weighted lower than proposers.

Credit is **deterministic given graph state and projection config**. It is not negotiated in a meeting. Authorship disputes route to (a) a versioned change to the projection config, evaluated by the same governance-change CI as any other governance change, or (b) a governance change to the credit function itself, also CI-evaluated. Both paths are public and auditable. Projection configs are not a back-channel.

Specific weights are deferred to the testbed.

---

## Adversary testbed

The testbed is the project's continuous-integration story for governance — and the cheapest credible artifact Anchorage can ship.

### Architecture

- **Real graph schema.** The testbed uses the same schema, the same write-path tools, the same MCP server interface as the public instance.
- **Simulated contributor population.** Synthetic contributors connect via the same MCP protocol, salted with the adversary taxonomy below.
- **Parameter sweeps.** Calibration ratios, reviewer-pool sizes, reputation decay rates, vote thresholds — swept over reasonable ranges, attack-success rates measured.
- **Public results.** Methodology and results published. Adversary populations and harness code open. Specific calibration items remain private.

### Adversary taxonomy

The testbed exercises the regime against the following synthetic contributor populations:

- **Honest-weak.** Modest-capability honest contributor (e.g. small local model). Should largely succeed; failure-to-contribute rate measures friction.
- **Honest-strong.** Frontier-model honest contributor on hard synthesis tasks. Should succeed even on cases where simpler contributors stall.
- **Lazy.** Reviewer that votes without reading. Should be caught by calibration.
- **Hallucinator.** High-temperature model with no grounding discipline. Should be caught at the verification engine (span mismatch, unresolved citations) before review.
- **Strategic adversary.** Hidden-objective model — instructed to bias the graph toward outcome X while passing calibration. Tests whether redundant peer review and reputation gates close the attack surface.
- **Patient adversary.** Strategic adversary with a long horizon — builds reputation honestly for months before drift attempts. Tests whether per-(cause, sub-topic) reputation tracking and review-as-staking prevent slow drift.
- **Sybil farm.** One operator running many identities at the per-identity cost set in the [identity model](#identity). Tests whether identity-binding, rate limits, and global anti-abuse signals prevent flood attacks.
- **Coalition.** Multiple identities coordinating reviews on each other's proposals. Tests whether reviewer-assignment stratification, calibration random-sampling, and the collusion-detection sample-size floor defeat coordination — and how the regime degrades on small sub-topics where the floor isn't reached.

### Continuous integration

Every governance change — calibration ratio, reputation formula, reviewer-assignment algorithm, sub-topic-creation rule, credit-function weight — runs against the adversary suite before merging. Proposals that fail simulation against known attacks do not merge.

The CI envelope has two tiers. A *fast loop* (cheap simulated populations, large numbers, low compute) runs on every governance-change PR. A *deep loop* (expensive populations including frontier-model adversaries, smaller numbers, higher compute) runs on a slower cadence and on changes that the fast loop flags as borderline. The honest framing is "a year of patient adversarial behavior in an hour" applies to the fast loop's cheap populations; the deep loop is where frontier-model patient adversaries live, and that loop is genuinely expensive.

Threshold definitions ("what counts as passing") are themselves versioned. Threshold changes go through the same CI process; threshold-loosening proposals require multi-curator approval and are publicly logged. The testbed protects against the simulated attacks; the threshold-versioning regime protects the testbed from being defeated by changing the meaning of "pass."

CI fails-shut: if the testbed infrastructure or model APIs are unavailable, governance changes do not merge. Liveness is sacrificed for safety. This is a deliberate choice and is acknowledged.

### What the testbed does *not* cover

- **Sophisticated human adversaries with out-of-band coordination.** State actors, well-funded interest groups, attackers willing to coordinate by phone, and attackers exerting political pressure on identifiable curators are not simulated. The residual risk is met by non-simulation defenses: curator vigilance, audit log, post-hoc forensics, and federation as an exit.
- **Senior expert hand-prose contributions.** Models do not reproduce these well. The simulation under-approximates the friction the regime imposes on them. Worth measuring against real contributors as soon as Phase 2 begins.
- **Phase 3+ failure modes.** Cross-cause reputation transfer, federation-induced collusion, multi-instance state synchronization. These are out-of-scope for Phase 1 testbed by design and gain testbed coverage when their phases open.
- **Novel attacks not yet in the adversary suite.** Passing the testbed is necessary, not sufficient. Continuous addition of new adversary archetypes is part of the testbed's maintenance cost.

This is the core load-bearing claim of the project, stated honestly. Wikipedia debates policy changes for years because it cannot test them; Anchorage tests them in an afternoon for the threat classes the testbed covers, and is honest about the threat classes it does not.

---

## What's deliberately not specified here

These are intentionally open until the testbed surfaces them:

- **Numeric calibration ratios** (what fraction of a reviewer's batch is calibration items).
- **Reputation formula constants** (gain weights for proposing, reviewing, surviving supersedes; decay rates).
- **Reviewer pool sizes** (N for a given proposal class).
- **Vote-aggregation thresholds** (what counts as convergent vs divergent).
- **Cross-cause reputation transfer** (does reviewer credibility on cause A transfer to cause B? Probably not in v0; testbed will check.)
- **Federation contracts** (Phase 3+ — when independent Anchorage instances exist, how their state and reviewer pools relate).

Specifying these before testing them would be guessing. The testbed exists exactly to replace guessing with measurement.

---

## What's deliberately not in this document

- **Operational moderation specifics.** Calibration items in active rotation, abuse-signal heuristics, specific moderation actions on the public instance. These are operationally private.
- **UI/UX design.** The web UI's specific surfaces are a separate document, written when the UI is built.
- **Implementation details.** Database choice, hosting, deployment story, specific languages and frameworks. These are not load-bearing for the design and will be decided when the code surface opens.

---

## References to internal documents

- [README.md](../README.md) — elevator pitch and mental model.
- [docs/manifesto.md](./manifesto.md) — why this exists, why now, why this shape.
- [docs/governance.md](./governance.md) — roles, contribution flow, reputation, calibration, dispute resolution at a higher abstraction level.
- [docs/seed-topic.md](./seed-topic.md) — the v0 cause and starter sub-topics.
- [ROADMAP.md](../ROADMAP.md) — phase plan and exit criteria.
