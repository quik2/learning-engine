# The Learning Engine

> *The student model is the teacher. The LLM is just its voice.*

A research-backed, systems-level approach to AI-powered education that is structurally different from every existing product in the space. Not a GPT wrapper. Not a flashcard generator. A learning engine — built on cognitive science, driven by a probabilistic student model, and compounded by cross-student behavioral data.

---

## The Problem

Every student who fails an exam had access to the same material as the student who passed. The bottleneck has never been information access.

The bottleneck is three things:

1. **Retrieval structure** — Knowledge encoded passively (reading, watching, summarizing) forms shallow traces. Re-reading produces the fluency illusion: the material *feels* understood because recognition is triggered. But recognition is not retrieval. Exam conditions require retrieval. Shallow traces fail under time pressure and interference.

2. **Forgetting dynamics** — Memory is a process, not a state. A concept "learned" on Day 1 is mostly gone by Day 4 without reinforcement. Students don't know their own forgetting rates. They study once, feel ready, and discover they've forgotten 60% of it.

3. **Metacognitive blindness** — Students don't know what they don't know. Worse, they think they know what they don't. The fluency illusion generates false confidence. The student who pastes slides into ChatGPT, reads a clean explanation, and closes the tab is more confident and knows less than they did before they opened it.

The research is unambiguous on this. Bastani et al. (PNAS, 2025): unrestricted GPT-4 users scored **17% lower** on real exams than controls. Barcaui (2025): ChatGPT group scored **11 points lower** on a 45-day retention test. 59-86% of college students use AI to study. They're getting worse.

A system that solves all three problems simultaneously — for every student, for every concept, on any material they upload — does not exist yet.

---

## What This Is

A multi-layer learning engine where:

- The **document pipeline** understands slides as pedagogical arguments, not bags of text
- The **knowledge graph** encodes concepts with prerequisites, difficulty, and misconceptions
- The **student model** tracks mastery, forgetting rate, metacognitive accuracy, and response patterns per student per concept
- The **scheduling engine** determines what to teach, when, at what scaffold level — deterministically, with no LLM involved
- The **content pipeline** uses the LLM to fill structured card templates whose type and sequence the LLM cannot change
- The **cross-student engine** runs nightly to tournament-test explanations, detect universal misconceptions, and refine prerequisite graphs from behavioral data
- The **exam readiness model** produces a calibrated, live, decaying predicted exam score

The LLM fills slots. The system decides what slots exist. Swap Claude for GPT-5 and nothing about the educational system changes. That is the test.

---

## Documentation

All research, architecture, and product documentation lives in [`/docs`](./docs/).

| Document | Description |
|---|---|
| [`docs/architecture/ARCHITECTURE-V2.md`](./docs/architecture/ARCHITECTURE-V2.md) | Full deep technical architecture — all 8 layers, algorithms, data schemas, ML stack |
| [`docs/architecture/ARCHITECTURE-V1.md`](./docs/architecture/ARCHITECTURE-V1.md) | First-pass architecture overview (foundational, less detailed) |
| [`docs/research/product-document-final.md`](./docs/research/product-document-final.md) | Complete product specification — the core loop, UX design, competitive differentiation |
| [`docs/research/scout-report-unbiased.md`](./docs/research/scout-report-unbiased.md) | Unbiased evidence review — what the learning science actually says, including the replication crisis |
| [`docs/research/case-against.md`](./docs/research/case-against.md) | The strongest case against building this — engagement trap, student behavior, ITS graveyard |
| [`docs/research/competitive-landscape.md`](./docs/research/competitive-landscape.md) | Deep analysis of every competitor: YouLearn, StudyFetch, HyperKnow, Vaia, Oboe, Gizmo, Knowt, and big tech threats |
| [`docs/research/evidence-base.md`](./docs/research/evidence-base.md) | Raw evidence and citations from the learning science literature |

---

## Core Thesis

**The student model is the teacher.**

Most AI education products treat the LLM as the teacher. This is architecturally wrong. The LLM has no knowledge of this student's forgetting rate, metacognitive blindness, error patterns, or exam format. It cannot know what to teach or when. It can only predict what tokens follow the prompt it received.

The teacher is the student model — the layer that tracks, with probabilistic precision, what this student knows, what they think they know, and when they'll forget it. The LLM executes the teacher's decisions in language.

**The population is the curriculum designer — continuously.**

Every existing adaptive learning company designs a curriculum once. This system's curriculum is revised every night by the collective behavior of every student who studied the same material. The explanation that wins is the one that empirically produced the highest 7-day retention across thousands of students — not the one a designer thought sounded good. This is a different epistemology for curriculum design, and it cannot be reproduced without the behavioral data.

**The metacognitive mirror breaks the fluency illusion.**

No product currently shows students both what the system thinks they know AND what they think they know, and surfaces the gap. When the system tells a student "You're 9/10 confident on mitosis. Your retrieval accuracy is 32%," the fluency illusion breaks. This is the feature that changes behavior.

---

## The Moat

The moat is not the technology. The technology is available. The moat is the **behavioral data that accumulates on top of the technology**:

1. **Explanation library** — Thousands of candidate explanations per concept, tournament-tested against real student outcomes. After 100K students study cell biology, the best explanation for every concept is known empirically. This cannot be reproduced by calling an API.

2. **Empirically-grounded prerequisite graphs** — Every LLM-inferred prerequisite edge is validated or refuted by behavioral evidence. The graph knows, measured from millions of interactions, that if you don't understand substrate-level phosphorylation, you fail chemiosmosis 73% of the time.

3. **Calibrated exam readiness** — After students report back enough actual exam scores, the predicted exam score becomes the most accurate academic performance predictor available for college students. That trust cannot be reverse-engineered.

4. **Fine-tuned teaching model** — A model trained specifically to teach, not to answer. Trained on proprietary interaction data. The deepest moat because the training set is the product.

---

## Status

This is a research and architecture repository. No code has been written. The research phase is complete.

**Next:** Phase 1 build — document pipeline, knowledge graph, basic student model, scheduling engine, content pipeline, web app. Ship to 50 students at one university. Measure: Day-3 return rate, checkpoint accuracy, actual exam performance vs. ChatGPT control.

---

## Research Basis

Key studies this architecture is built on:

- Bastani et al. (PNAS, 2025) — Unrestricted GPT-4 users scored 17% lower on real exams
- Rawson & Dunlosky (2022) — Successive relearning: 68% retention at 1 month vs. 11% control
- Roediger & Karpicke (2006) — Testing effect occurs without feedback; retrieval itself strengthens traces
- Ma et al. (2014, Journal of Educational Psychology) — ITS vs. individualized human tutoring: g = -0.11 (no significant difference)
- LECTOR (arxiv 2508.03275, 2025) — Interference-aware spaced repetition scheduling
- Kalyuga (2003) — Expertise reversal: techniques optimal for novices become harmful for experts
- Kestin et al. (Scientific Reports, 2025) — Structured AI tutor: effect size 0.73–1.3 SD in physics
- Ropovik et al. (PLOS ONE, 2021) — Publication bias inflates education research effect sizes 6-7x

The full literature review is in [`docs/research/scout-report-unbiased.md`](./docs/research/scout-report-unbiased.md).

---

*Built on the premise that the technology is ready. The science is done. The only question is execution.*
