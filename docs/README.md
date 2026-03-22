# Documentation

This folder contains all research, product, and architecture documentation for The Learning Engine.

---

## Architecture

| File | Description |
|---|---|
| [`architecture/ARCHITECTURE-V3.md`](./architecture/ARCHITECTURE-V3.md) | **Current architecture. Start here.** Implementation-precise: full service diagram, exact data schemas, ML training pipelines, constraint validation loops, three-phase scheduling evolution (deterministic → contextual bandit → deep RL), ZPD targeting, deliberate practice loop, Qdrant integration, async ingestion pipeline. Precise enough to hire engineers into. |
| [`architecture/ARCHITECTURE-V2.md`](./architecture/ARCHITECTURE-V2.md) | Deep theoretical architecture. Strong on the *why* — the learning science theory, the moat logic, the honest risks. Read before V3 for context. |
| [`architecture/ARCHITECTURE-V1.md`](./architecture/ARCHITECTURE-V1.md) | First-pass overview. Good orientation. Superseded by V2 and V3. |
| [`architecture/AI-SYSTEMS.md`](./architecture/AI-SYSTEMS.md) | **The AI.** Every model, precisely. DKT-GAT architecture (HGT + transformer + cross-attention fusion), FSRS+IRT combination, POMDP formulation, confidence calibration as model feature, three-phase scheduler (deterministic → contextual Thompson sampling bandit → deep RL with hindsight credit assignment), constrained generation via Instructor+Pydantic, explanation tournament, model tiering, DPO+RLAIF fine-tuning pipeline, inference serving latency budget, weekly evaluation suite. |
| [`architecture/GTM-BUSINESS-MODEL.md`](./architecture/GTM-BUSINESS-MODEL.md) | Go-to-market strategy, business model, pricing, retention reframe, Canvas LTI play, FERPA/privacy compliance architecture, voice mode (Study Walk) full technical design, semantic answer evaluation pipeline, and IRB study design. |

## Research

| File | Description |
|---|---|
| [`research/product-document-final.md`](./research/product-document-final.md) | **Core product specification.** The full vision: what the product is, the core learning loop, competitive positioning, the Sarah walkthrough, technical build order. |
| [`research/scout-report-unbiased.md`](./research/scout-report-unbiased.md) | **Unbiased evidence review.** What the learning science actually supports, what has failed replication, honest assessment of ITS research including the Knewton failure and the Carnegie Learning RAND study. |
| [`research/case-against.md`](./research/case-against.md) | **The strongest case against.** Engagement trap, student behavior patterns, ITS graveyard, the fundamental tension between what works and what students will pay for. |
| [`research/competitive-landscape.md`](./research/competitive-landscape.md) | **Competitor deep-dives.** YouLearn ($70K MRR, YC), StudyFetch ($11.5M Series A, 6M users), HyperKnow, Vaia (40M users), Oboe ($16M a16z), Gizmo, Knowt — what each actually does, what it doesn't do, and why none of them are this. |
| [`research/evidence-base.md`](./research/evidence-base.md) | **Raw literature.** Primary sources, effect sizes, study populations, and honest caveats on publication bias. |

---

## Reading Order

**For a technical audience (engineers, CTOs):** V2 Architecture → V3 Architecture → Product Document

**For a product/investor audience:** Product Document → Scout Report → Case Against → V2 Architecture (Moat section)

**For a research audience:** Scout Report → Evidence Base → V3 Architecture (Layers 3 and 6)

**To understand the core idea in 10 minutes:** Product Document (What This Is + The Core Loop + The Sarah Walkthrough)
