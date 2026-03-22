# The Learning Engine — Architecture Document
*Written by JARVIS, March 2026. This is the real plan.*

---

## The Problem With Every Existing Approach

Every competitor — YouLearn, StudyFetch, Mindgrasp, HyperKnow, Vaia — does the same thing: take uploaded content, run it through a language model, generate tools (flashcards, quizzes, summaries, chatbot), and hand them back to the student. The student decides what to do. The AI is the entire product.

That's the trap. The LLM is not the hard part. The hard part is knowing:
- What does this specific student actually know right now?
- Which concept should they study next, in what sequence, in what format?
- Which explanation of this specific concept produced the highest retention for the last 10,000 students who studied it?
- Given that their exam is in 8 days and they've mastered 40% of the material, what is the optimal session schedule to maximize their expected score?

No language model answers those questions. Those are systems problems. **The product that solves them is not a GPT wrapper. It is a learning engine that happens to use an LLM as one component.**

---

## What This Actually Is

A system with seven distinct layers, each solving a hard technical problem. The LLM lives in Layer 5. Layers 1-4 and 6-7 require no LLM at all.

The surface — the UI students interact with — is a scroll-through lesson experience. Clean cards. Forced checkpoints. No chatbot to hide behind. But the surface is the least interesting part.

---

## Layer 1: Document Intelligence

**The problem:** Professor slides are not structured educational content. They're speaker notes with diagrams. A raw text extraction misses the relationships, the diagrams, the hierarchy of what's being taught.

**What competitors do:** PyPDF2. Text dump. Done.

**What this does:**

*Step 1 — Raw extraction:* MinerU (open-source, CVPR 2025 benchmarked, handles multi-column, math formulas, tables, layout preservation). Produces structured markdown with positional metadata.

*Step 2 — Visual pass:* Each slide is also treated as an image. A vision-language model (Claude 3.5 Sonnet or locally-hosted Llama 3.2 Vision 90B via Ollama) reads it for: diagram content, concept emphasis via visual hierarchy, relationships implied by spatial layout, and any content visible only in the image layer. The VLM output is merged with the text extraction.

*Step 3 — Semantic enrichment:* For each extracted content block, classify: is this a definition? A worked example? An edge case? An analogy? A diagram label? Different content types get different pedagogical treatment downstream.

*Output:* Not text. A structured document object: slide metadata + concept candidates + content type classifications + visual element descriptions. This is the input to Layer 2.

**Why this matters:** The quality of everything downstream is bounded by this step. A richer document object means a richer knowledge graph, which means better lessons. This is where most competitors are weakest and where this system invests most in quality infrastructure early.

---

## Layer 2: Knowledge Graph Construction

**The problem:** A topic is not a list of facts. It's a web of concepts with dependencies. To understand chemiosmosis, you need to understand the electron transport chain. To understand the ETC, you need to understand redox reactions. Teaching chemiosmosis to someone who doesn't know redox is pedagogically backwards. Every static tool ignores this.

**The architecture:**

*Concept extraction:* LLM structured-output call on the document object. Produces concept nodes:
```json
{
  "id": "glycolysis",
  "name": "Glycolysis",
  "definition": "...",
  "bloom_level": "understand",
  "examples": [...],
  "common_misconceptions": ["4 ATP net (wrong — it's 2 net)", "occurs in mitochondria (wrong)"],
  "visual_available": true,
  "estimated_difficulty": 0.62,
  "related_vocabulary": ["pyruvate", "cytoplasm", "NAD+"]
}
```

*Prerequisite detection:* Using the ACE method (2025 JEDM paper): LLM pairwise comparison "is concept B a prerequisite of A?" across the concept set. Cross-validated against subject-domain ontologies bootstrapped from open educational resources (Khan Academy, Wikipedia category graphs, OpenStax concept indexes).

*Storage:* PostgreSQL with pgvector extension. Nodes stored with embeddings (text-embedding-3-large or Nomic-Embed-Text open source). Edges store: prerequisite type (hard vs. soft), confidence score, source (LLM-inferred vs. population-validated), semantic distance for interference modeling.

*Cross-document synthesis:* When a student uploads Lecture 12, the system checks if concepts already exist from Lectures 1-11. Mastery transfers. The student doesn't re-learn ATP in every lecture.

*Population validation:* After N students study the same material, prerequisite edges get behavioral weights. If 80% of students who fail concept B also fail concept A, that's a stronger prerequisite signal than any LLM inference. The graph gets better with data.

**Storage:** Neo4j for the graph layer (efficient traversal for prerequisite chains), PostgreSQL for the relational data, pgvector for semantic search.

---

## Layer 3: The Student Mastery Model

**This is the most important layer. And the one no competitor has.**

A mastery model answers one question at every moment: *For this student, what is the probability they can correctly retrieve each concept, right now?*

That probability determines everything: what gets taught next, how long until review, whether to scaffold or withdraw support.

**Architecture:**

*Deep Knowledge Tracing (DKT):* Transformer-based model, pretrained on open educational datasets (Assistments: 400K+ students, 4M+ interactions; EdNet: 780K students, 130M+ interactions). Fine-tuned on this system's accumulating response data. DKT takes a sequence of (concept, correct/incorrect) pairs and outputs P(mastery) for every concept in the graph. Proven to outperform classical Bayesian Knowledge Tracing on every benchmark.

*FSRS personalization:* Free Spaced Repetition Scheduler (open-source, used by 10M+ Anki users, outperforms SM-2 by ~30% on retention prediction). Each concept gets stability and difficulty parameters fit to the individual student's response pattern. The scheduler knows: this student retains calculus concepts well (high stability) but forgets biology terminology fast (low stability). Scheduling adapts.

*Confidence calibration:* Before every checkpoint, student rates confidence (sure/not sure). System tracks: when this student is confident and wrong (overconfidence), schedule review within 48-72 hours per hypercorrection effect. When this student is underconfident and right (imposter syndrome pattern), adjust difficulty downward. The confidence signal is a feature the student provides without knowing they're calibrating the model.

*Forgetting curve parameters:* Not generic. Personalized. The system learns your forgetting rate per concept category and adjusts review intervals accordingly.

*Cold start:* New student has no history. Initialization from upload analysis: quiz a few concepts across the graph, identify mastery frontiers, estimate prior knowledge. 3-5 diagnostic interactions before the model has meaningful signal.

**No LLM here.** Pure PyTorch inference, running as a Python microservice. Inference time: <100ms per request.

---

## Layer 4: The Scheduling Engine

**Deterministic. Algorithmic. No LLM. No hallucination possible.**

Given inputs:
- Knowledge graph (with mastery probabilities from Layer 3)
- Exam date + format + depth setting
- Available study time
- Cross-student interference weights (from Layer 6)

Produces:
- Ordered lesson plan: which concepts to teach today, in what sequence
- Review queue: which graduated concepts need retrieval practice today
- Session structure: estimated time, difficulty curve, checkpoints per session

**Algorithm:**

*Frontier detection:* Concepts where prerequisites are mastered but the concept itself is not. These are the teachable concepts. Ranked by: dependency count (teach concepts that unlock the most others first), exam weight (concepts that appear more on tests of this format), time pressure (how long until exam × current mastery × forgetting rate).

*Successive relearning scheduler:* Concept doesn't graduate until 3 consecutive correct recalls across spaced sessions. Most SRS tools (including Anki in practice) let students graduate a card after one correct answer. The Rawson & Dunlosky research is unambiguous: mastery-to-criterion across sessions produces dramatically higher retention at 1 month. This is implemented at the algorithmic level, not dependent on student discipline.

*LECTOR interference:* Scheduling adjusts intervals based on semantic similarity between concepts in queue. If glycolysis (just learned) and gluconeogenesis (coming up) are semantically adjacent, the scheduler doesn't present them back-to-back. It inserts an unrelated concept between them to reduce interference. Interval math uses the LECTOR formulation: effective_half_life = base_stability × (1 - α × semantic_similarity_to_recent_items).

*Deadline compression:* As exam approaches, stability requirements relax. A concept that normally needs 5 days between reviews gets reviewed in 3 days. Spacing is still present — it just compresses proportionally. Cramming mode still uses retrieval practice, still better than passive re-reading.

*Multi-armed bandit for question format:* For each concept, the system has a distribution over question formats (MC, fill-in-blank, free recall, application). Starts uniform. Updates based on: which format this student performs best on for this concept type × which format best predicts actual exam performance for this test format × exploration-exploitation tradeoff. The student always gets the format most likely to produce retention, not the one they'd choose for comfort.

---

## Layer 5: Content Generation Pipeline

**The LLM lives here. It fills slots in a structure it did not create.**

The scheduling engine decides: teach glycolysis to Sarah, depth "know it well," MC format exam, she's overconfident on cellular respiration but weak on glycolysis, 8 minutes available.

The content pipeline produces a 7-card lesson sequence. Each card has a type, determined algorithmically:

1. **Forward test card** — Activates relevant prior knowledge. "Your cells need ATP. Glucose has way more energy than one ATP can carry. How does the cell extract it?" (Generation effect, d=0.40)
2. **Pretest card** — Question before any teaching. Student guesses, rates confidence. Even wrong guesses prime encoding (pretesting effect, d=0.82-1.24). Confidence captured.
3. **Teach card(s)** — One concept per card. Visual if available. LLM writes to principles: segmenting (d=0.79), coherence (d=0.86), multimedia (d=1.67), personalization ("you/your," d=1.11). No seductive details.
4. **Checkpoint card** — Can't skip. Student answers before seeing explanation. Confidence rating. Distractors are real misconceptions, not random wrong answers.
5. **Trap-coaching card** — "Answer choices love to say '4 ATP' for glycolysis. That's gross, not net." Test-format-aware. For MC only.
6. **Elaborative interrogation card** — "Why does glycolysis happen in the cytoplasm and not the mitochondria?" Self-generated explanation produces better retention than reading one.
7. **Pretest for next concept** — Preview + conditional on performance. Interleaving preview.

**The LLM's constraints:** It receives a structured prompt with: concept JSON, student context (prior errors, confidence history, explanation styles that worked), test format, depth, card type, and the pedagogical principle to apply. It generates content that fills the card. It cannot change the card sequence. It cannot decide to skip a checkpoint. The structure enforces the pedagogy.

**Fact-check pass:** Separate LLM call. Grounded against the source document. Any claim in the generated card not supportable from the original slides gets flagged and either revised or removed.

**Content cache:** See Layer 6. Winning explanations are stored and served first. The LLM generates novel content less over time as the best explanations accumulate.

---

## Layer 6: The Cross-Student Learning Flywheel

**This is the actual moat. This is what makes this a billion-dollar company and not a study tool.**

Every competitor generates content fresh for each student and throws the behavioral signal away. This system accumulates it.

**What gets captured:** For every card shown to every student: response (correct/incorrect), confidence rating, time-on-question, whether they re-read the previous card, whether they completed the session, and (quarterly survey) their actual exam score.

**What gets computed:**

*Explanation variant performance:* The system generates multiple explanation variants for high-frequency concepts (A/B test automatically in production). Variant A of the glycolysis explanation produces 81% checkpoint accuracy. Variant B produces 67%. Variant A is promoted. Stored. Future students get Variant A by default. The LLM isn't called for glycolysis anymore — the system already knows the best explanation.

After 100,000 students have studied cell biology, the system knows the optimal explanation for every concept. That's not something you can replicate by calling Claude.

*Universal confusion detection:* If concept X has checkpoint accuracy <60% across students with strong prerequisites, it's a structurally hard concept. The system automatically: generates more scaffolded explanations, increases review frequency, adds a pre-concept diagnostic.

*Misconception mapping:* Wrong answers aren't random. When 40% of students select "4 ATP" for glycolysis net ATP, that's a structural misconception — probably induced by the way the topic is commonly taught. The system detects this pattern, adds a dedicated misconception-correction card, and reduces that wrong answer rate. This is learnable. Competitors can't learn it because they don't have the data.

*Prerequisite graph refinement:* If students who fail concept B also fail concept A at a rate higher than chance, that's an empirical prerequisite. Stronger than LLM inference. The graph gets better.

*Predicted score calibration:* Every student who reports back their actual exam score improves the prediction model. After enough data, the model knows: "students with this mastery map going into a Bio 202 MC exam typically score 74±6%." The number becomes genuinely predictive. This is the viral mechanism.

**Infrastructure:** PostgreSQL behavioral data warehouse. Nightly batch jobs: explanation variant analysis, confusion detection, prerequisite graph updates. Weekly: DKT fine-tune on accumulated data. Monthly: full system evaluation against collected exam scores.

---

## Layer 7: The Predicted Exam Score

**The home screen number. The hook. The day-2 retention mechanism.**

Not a percentage bar. A number with a letter grade: **"Predicted: 74% (C)"**

It drops when you don't study — the forgetting curves erode mastery on unreviewed concepts, and the model updates in real-time. It climbs when you complete sessions.

**Technical construction:**

*V1 (launch):* Weighted mastery average. Concept weights estimated from course structure + subject-domain knowledge. Not perfectly calibrated but directionally right. Shows the right trend.

*V2 (after data):* Bayesian model. P(exam score | mastery map, concept weights, test format). Concept weights learned from students who reported back actual scores. Cross-validated. Calibration error tracked and minimized. Each student who reports back makes the prediction better for every future student in the same course.

*The display:* Shows the number, the trend over the last 7 days, and "What's pulling it down" — the specific concepts with the biggest gaps between current mastery and exam-weighted importance. This becomes the study plan.

**Behavioral design:** Students don't need to want to do spaced repetition. They want the number to go up. The number going up requires the algorithm. The product design removes the need for students to understand or choose the correct behavior — it's built into the structure.

---

## The Stack

| Layer | Technology | Why |
|---|---|---|
| Document extraction | MinerU (open source) | CVPR 2025 benchmarked, handles slides, formulas, tables, layout |
| Visual understanding | Claude 3.5 Sonnet / Llama 3.2 Vision 90B | Diagram comprehension, visual hierarchy, slide semantics |
| Knowledge graph | PostgreSQL + pgvector + Neo4j | Relational + semantic + graph traversal |
| Concept embeddings | text-embedding-3-large or Nomic-Embed-Text | Semantic distance for interference calculation |
| Student model | DKT (PyTorch) + FSRS (Python, open source) | Best-in-class knowledge tracing + spaced repetition |
| Scheduling engine | Pure Python service | Deterministic, no LLM, sub-100ms |
| Content generation | Claude API (structured prompting) | Instruction following for structured template filling |
| Fact checking | Claude API (grounded against source) | Reduces hallucination risk on domain content |
| Content cache | PostgreSQL | Winning explanations stored, served before generation |
| Behavioral warehouse | PostgreSQL | Response data, confidence ratings, session completion |
| Analytics pipeline | Python batch jobs (nightly/weekly) | Explanation A/B, confusion detection, graph updates |
| Frontend | Next.js (web-first, then React Native) | Fastest iteration, SEO, server components |
| Backend | FastAPI (Python) + Node.js where appropriate | Python for ML services, Node for API layer |
| Auth | Clerk or Auth0 | Don't build this |

---

## The Three Hard Problems (Honest Assessment)

**1. Slide quality variance at launch**

Some professors' slides are incomplete, disorganized, or actively wrong. A system that builds a knowledge graph from bad slides teaches bad content confidently. This is the single biggest quality risk.

*Mitigation:* Extraction confidence scoring. Flag low-quality extractions before they become lessons. Build a lightweight human review interface for the first 100 uploads — QA by hand, learn what the edge cases are, use that to improve extraction. Don't promise quality you can't deliver. Better to say "this upload had low-quality slides, here's what we extracted" than to silently teach wrong content.

**2. Cold start on the student model**

DKT needs behavioral history to be accurate. A brand new student has none.

*Mitigation:* 3-5 diagnostic interactions across the knowledge graph frontier before the first lesson. Brief, framed as "let's see where you're starting from." Not a quiz — a calibration. The model gets enough signal to initialize meaningfully. First-session quality will be lower than steady-state. That's acceptable and honest.

**3. Predicted score calibration with sparse ground truth**

You need students to report actual exam scores to calibrate the prediction. Most won't.

*Mitigation:* Make it part of the session-end flow, not a separate action. "Your exam is tomorrow — come back after and tell us your score. It helps us get better at predicting." Incentivize with something real: extended prediction window, course-specific insights. Even 20% ground truth return rate is enough to calibrate if the student base is large enough. V1 doesn't need to be calibrated — it needs to be directionally right.

---

## Build Sequence (Real Company, Not MVP)

The research is already done. The product spec is already done. The competitive landscape is mapped. None of that work gets re-done.

**Phase 1 — The Engine (first 6-8 weeks):** Document intelligence pipeline. Knowledge graph construction. Basic scheduling (FSRS, no interference yet). Content generation pipeline with structured templates. Web app that can take an upload and produce a lesson. Ship to 50 students at one university. Validate one thing: do students get higher scores than ChatGPT control group? Everything else is secondary to this proof point.

**Phase 2 — The Student Model (weeks 8-16):** DKT integration. Personalized FSRS. Predicted exam score v1. Confidence tracking + hypercorrection scheduling. Session return logic ("coming back in 3 days" actually functions). A/B infrastructure for explanation variants.

**Phase 3 — The Flywheel (months 4-9):** Cross-student learning analytics. Explanation promotion system. Population-level confusion detection. Predicted score v2 with Bayesian calibration. LECTOR interference weighting in scheduler.

**Phase 4 — The Moat (12+ months):** Fine-tuned teaching model (7B, aligned to force retrieval, not answer). Canvas LTI integration (institution distribution). Voice mode (retrieval practice for AirPods). IRB research study → published paper → institution credibility → B2B sales motion. Series A story: "we published the first study showing LLM tutoring improves delayed exam performance, by 0.5 SDs. Here's the dataset. Here's the model."

---

## The Actual Moat (In Order of Defensibility)

1. **Behavioral data:** Response patterns, confidence ratings, session completion, exam outcome correlations. Proprietary. Compounds with every student. Cannot be replicated by calling GPT-4.

2. **Cross-student explanation optimization:** After 100K students, you know the best explanation for every concept in every common course. Costs nothing to serve. Impossible to catch up to without the student data.

3. **Calibrated predicted score:** Accuracy improves with ground truth feedback. A product that reliably predicts your exam score within 5 points has trust no newcomer can claim.

4. **Fine-tuned teaching model:** After Series A. A model trained specifically to teach, not to answer. Proprietary training set. Cannot be replicated without the data.

5. **Subject-domain prerequisite graphs (population-validated):** After N students per subject, the graphs are empirically grounded, not LLM-inferred. Better than any competitor can generate from scratch.

---

## Why This Is Not a GPT Wrapper

A GPT wrapper: input → LLM → output. The LLM makes every decision.

This system: The LLM fills structured slots in cards whose type, sequence, and timing are determined by four distinct algorithmic layers (document intelligence, knowledge graph, mastery model, scheduling engine). The LLM cannot change what type of card to show. It cannot decide to skip a checkpoint. It cannot choose not to enforce retrieval before explanation. These behaviors are architectural — not dependent on prompting.

The LLM is the text-generation subsystem in a much larger system. Swap Claude for GPT-5 and nothing structurally changes. That's the test. If swapping the LLM changes the core product behavior, it's a wrapper. This passes that test.

---

## What This Is Not

- Not a chatbot. The chatbot is available in the corner. The product is the curriculum.
- Not an AI tutor that answers questions. An AI tutor that forces you to answer questions.
- Not a school replacement. A study supplement that works because it fits into exam cycles.
- Not trying to teach everything. Focused on: uploaded course material, defined exam date, specific test format. Scoped intentionally.
- Not assuming sustained daily use. Designed for intense 2-6 week exam prep cycles. The academic calendar provides the use pattern structure.

---

## The Fundraising Story (When Ready)

**Seed:** "We built the first study tool that treats learning science as the product, not the marketing. Here is our 50-student RCT from UCLA showing a 0.5 SD improvement in exam scores over ChatGPT control. Here is our Day-14 retention rate (we expect 18-22%, vs. 2-3% industry average for education apps, because we're not trying to be daily — we're trying to be exam-essential). We have $X MRR from students who found us organically. We're raising to reach 10,000 students and run the validation study that produces the published paper."

**Series A:** "Here is the published paper. Here's the calibrated predicted score model. Here's the cross-student explanation optimization data showing 23% improvement in checkpoint accuracy for our high-frequency concepts. Here is the fine-tuned model we trained on our proprietary dataset. We are now a data company that teaches students, not an AI company that generates study tools."

---

*This document represents what this product could be if built with the discipline to resist shortcuts. The technology is available. The research is done. The architecture is coherent. The question is whether to build it right — slowly, carefully, with systems that compound — or to build something that ships in a week and doesn't matter.*

*One of these is interesting. The other one already exists.*
