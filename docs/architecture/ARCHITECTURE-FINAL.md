# The Learning Engine — Final Architecture
*Everything. Clean. No filler.*

---

## The One Idea

Every prior architecture document was building toward this:

**The gap between studying and learning is retrieval.** Students confuse recognition for knowledge. Reading a clean explanation feels like learning. It isn't. At exam time, the only thing that exists is your ability to retrieve — under pressure, from a partial cue, surrounded by interference. The fluency illusion (Koriat & Bjork, 2005) is why students who studied hard still fail: they optimized for the feeling of understanding, not for the act of retrieval.

This product enforces retrieval at every step. That is not a feature. It is the architecture.

Everything else — the student model, the knowledge graph, the dialogue layer, the relevance engine — exists to make that retrieval practice as efficient and targeted as possible for this specific student, right now.

---

## What the System Actually Is

Five engines working together. Each one can fail independently. Each one makes the others better.

```
┌──────────────────────────────────────────────────────────┐
│  ENGINE 1: Document Intelligence                          │
│  Slides → Structured knowledge graph                     │
└────────────────────────┬─────────────────────────────────┘
                         │
┌────────────────────────▼─────────────────────────────────┐
│  ENGINE 2: Student Model                                  │
│  Who is this person + what do they know                  │
└──────────┬──────────────────────────────┬────────────────┘
           │                              │
┌──────────▼──────────┐    ┌─────────────▼──────────────┐
│  ENGINE 3: Scheduler │    │  ENGINE 4: Content Pipeline │
│  What to teach,      │    │  How to teach it to         │
│  when, at what       │───►│  this specific person       │
│  difficulty          │    │                             │
└──────────────────────┘    └─────────────────────────────┘
                                          │
┌─────────────────────────────────────────▼──────────────┐
│  ENGINE 5: Population Flywheel                          │
│  Every student makes every future student's lessons     │
│  better                                                 │
└────────────────────────────────────────────────────────┘
```

---

## Engine 1: Document Intelligence

### What actually happens when you upload

The system runs two processes in parallel on your slides.

**Structural extraction** (MinerU, open-source, CVPR 2025 benchmarked): reads the document's architecture — heading hierarchy, bullet nesting, formula notation, table structure. Output is structured markdown with positional metadata. Not a text dump. A document with preserved meaning.

**Visual analysis** (Claude 3.5 Sonnet as VLM): each slide is rendered as a 2048px image and analyzed for what can't be extracted as text: diagram content and spatial relationships, what concepts the diagram is actually teaching, what the professor visually emphasized (circled terms, bold, arrows), and the pedagogical narrative — what argument this slide is making. The VLM output includes a list of concepts visible only in the image layer, plus relationship triples extracted from the diagram structure.

The two outputs are merged into a **Document Object** — not text, but a structured pedagogical understanding of each slide.

**Content classification**: every content block is tagged by type — definition, mechanism (multi-step process), fact, principle, worked example, edge case, diagram concept, or exam signal. The type determines how the concept is later taught. A mechanism gets a different lesson structure than a fact.

**Quality scoring**: each slide gets an extraction confidence score. Slides below 0.55 are flagged visibly in the UI — "We had trouble reading this slide's diagram. Consider cross-referencing your notes." Transparency over false confidence.

**Processing time**: 45–90 seconds for 30 slides.

### The Knowledge Graph

From the document object, the system builds a **Directed Pedagogical Knowledge Graph (DPKG)**. Not a vector store — a graph. The difference matters: a vector store retrieves by similarity; the graph encodes dependency. Glycolysis and gluconeogenesis are semantically similar but pedagogically ordered.

Every concept is a node:
```
ConceptNode {
  name: "Glycolysis"
  definition: formal one-sentence definition
  bloom_level: UNDERSTAND (6-level taxonomy)
  content_type: MECHANISM
  difficulty_estimate: 0.62 (population-updated)
  common_misconceptions: ["net ATP = 4 (wrong — 2 net)", "occurs in mitochondria (wrong)"]
  visual_available: true
  exam_weight: derived from slide emphasis + behavioral data
}
```

Every prerequisite relationship is an edge with strength (HARD/SOFT), evidence source (LLM inference/ontology/behavioral), and semantic distance (used for interference scheduling).

**Prerequisite detection runs from three sources**, in order of reliability:
1. LLM pairwise reasoning: "Is concept A structurally required to understand concept B?" — ~80% accurate on hard prerequisites
2. Domain ontology grounding: UMLS (biology/medicine), ACM CCS (CS), MathSciNet (math) — cross-validates LLM inference
3. Behavioral validation (after data): P(fail B | fail A) measured across students who've studied both — overtakes the other two after 50+ students

The graph refines itself. After behavioral data accumulates, LLM-inferred edges that don't hold empirically get downgraded. Edges the LLM missed get discovered. The graph becomes genuinely accurate over time.

**Cross-document synthesis**: when a student uploads their third lecture, the system checks every new concept against their existing graph. "Protein folding" from Cell Bio maps onto "protein secondary structure" from their prior Biochem upload (0.89 cosine similarity). Mastery transfers. The student doesn't re-learn from zero.

---

## Engine 2: The Student Model

### Two parallel models

**What they know (DKT-GAT)**: Deep Knowledge Tracing with Graph Attention. A transformer that processes every interaction the student has had — (concept, correct/wrong, response_time, confidence, card_type) tuples — and outputs a probability of mastery for every concept in their graph simultaneously. Not just sequence modeling: a Graph Attention Network propagates information along prerequisite edges, so mastering concept A shifts the prior for dependent concepts before they're even attempted.

Pretrained on 4M+ student interactions (ASSISTments + EdNet). Fine-tuned weekly on the product's own accumulating data via LoRA (0.5% of parameters updated — prevents catastrophic forgetting while specializing). Cold start: 3–5 diagnostic questions before the model has meaningful signal; uses population priors until then.

**Who they are (Student Intelligence File)**: Not a mastery vector. A structured portrait. Updated across every session and upload.

```
StudentIntelligenceFile {
  // Academic context
  year, major, institution, active_courses, prior_uploads

  // Inferred behavioral patterns (never self-reported labels)
  format_response_history: {
    MECHANISM: {diagram: 0.81, prose: 0.62, analogy: 0.74}
    DEFINITION: {prose: 0.84, diagram: 0.71}
  }
  analogy_hit_rate: 0.68 (proportion of analogies that produced correct retrieval)
  analogy_domains: ["economics", "sports"] // only if checkpoint data backs them
  engagement_dropoff_curve: {min_0: 1.0, min_10: 0.88, min_20: 0.71, min_30: 0.52}

  // Current context
  time_pressure: "30 minutes"
  depth_goal: PASS_THE_TEST | ACTUALLY_UNDERSTAND
  exam_format: MULTIPLE_CHOICE

  // What hasn't worked
  failed_explanation_types: [{concept: "glycolysis", format: "prose", accuracy: 0.31}]

  // Cross-course knowledge
  cross_course_mastery: {"protein_folding_canonical": 0.74, "ATP_canonical": 0.89}
}
```

What this is not: a learning styles profile. VARK is a debunked neuromyth (Pashler et al., 2008; 2024 meta-analysis confirms). The format_response_history tracks empirical format effectiveness per concept type, measured from checkpoint accuracy. Not preference. Evidence.

### IRT at the question level

Item Response Theory runs parallel to DKT-GAT. For every question in the pool, once it has 100+ responses:
- **a parameter** (discrimination): how sharply does this question separate students by ability?
- **b parameter** (difficulty): what ability level gives 50% chance of correct response?

The scheduler uses Fisher Information to select the most informative question at every checkpoint — the question where the student has ~50–65% chance of success, maximizing information gain while keeping them in the productive challenge zone.

### What the student model does with confidence and response time

Every checkpoint includes a confidence rating (Sure / Unsure). This is not UX decoration — it's a model feature.

The system tracks calibration per student: when they say "Sure," how often are they right? A well-calibrated student saying "Sure, wrong" is a stronger not-mastered signal than a chronically overconfident student saying the same. The DKT-GAT adjusts its mastery update weights based on per-student calibration coefficients.

Response time: classified against the student's personal baseline. Fast + correct = automatic retrieval, genuine mastery. Slow + correct = worked it out, mastery is fragile, review sooner. Fast + wrong = probably guessing, discount the signal. Slow + wrong = genuinely tried and failed, strong not-mastered signal.

---

## Engine 3: The Scheduler

### What it decides

Every session, the scheduler answers: given this student's full knowledge state, their exam date and format, their time budget, and the interference map — what is the optimal set of concepts to cover, in what sequence, at what difficulty?

This is a constrained optimization problem. The objective: maximize expected exam score. The constraints: time budget, interference avoidance, ZPD (can't teach what prerequisites haven't unlocked), engagement (can't sustain sessions where difficulty is too high or too low).

### Priority function

```python
def concept_priority(concept, student) -> float:
  mastery = dkt_gat.p_mastered(student, concept)
  stability = fsrs.stability(student, concept)
  days_to_exam = student.days_until_exam

  # Marginal gain: how much does studying this today improve predicted exam score?
  p_without = mastery * exp(-days_to_exam / stability)
  p_with = fsrs.predict_post_study(student, concept, days_to_exam)
  marginal_gain = (p_with - p_without) * concept.exam_weight

  # Adjustments
  due_boost = 2.0 if fsrs.is_due(student, concept, today()) else 1.0
  blindness_boost = 1.0 + max(0, student.stated_confidence(concept) - mastery)
  zpd_pass = float(zpd_gradient(concept, student) > 0.4)  # Gradient, not binary
  interference_penalty = compute_interference(concept, student.recent_reviews)

  return (marginal_gain * due_boost * blindness_boost * zpd_pass) - interference_penalty
```

**ZPD is a gradient, not a binary.** The "prerequisite mastery > 0.75" gate from V4 is replaced with a smooth function: how ready is this student to learn this concept right now, from 0 to 1? A student with 0.6 prerequisite mastery isn't blocked — they're at 60% readiness, which might be acceptable for a soft prerequisite. The scheduler uses this gradient to rank frontier concepts, not gate them.

**Interference scheduling**: semantically adjacent concepts (glycolysis/gluconeogenesis, mitosis/meiosis) are separated in the review queue using embedding similarity + recency weighting. Reviewing them back-to-back increases interference and reduces net retention. The LECTOR formulation (arxiv 2508.03275): effective stability is reduced by Σ(semantic_similarity × recency_weight) for all recently reviewed concepts.

**Interleaving**: after a concept has been taught once (initial session), future reviews interleave it with semantically adjacent concepts. Blocked practice feels easier but produces weaker discrimination. Interleaving forces the student to actively distinguish between glycolysis and gluconeogenesis on every review — which is exactly what the exam will require.

**Deadline compression**: as exam approaches, the scheduler solves a marginal gain per minute optimization. It explicitly deprioritizes concepts where there isn't enough time to reach exam-level mastery — and tells the student: "With 2 days left, we're skipping these 3 concepts. Focusing here moves your score the most."

**Successive relearning enforcement**: concepts graduate only after 3 correct recalls across separate sessions. One right answer doesn't count. The FSRS stability parameters are fit per student per concept from response history — their personal forgetting rate, not a generic curve.

### The challenge gradient

Target: 70–80% checkpoint accuracy throughout the session. Managed in real time.

```python
class ChallengeManager:
  WINDOW = 5  # rolling last 5 checkpoints
  
  def adjust(self, session) -> Adjustment:
    accuracy = session.rolling_accuracy(self.WINDOW)
    
    if accuracy > 0.82:  # Too easy — comfort zone, not ZPD
      return Adjustment(increase_bloom=True, reduce_scaffold=0.15,
                        prefer_application=True)
    elif accuracy < 0.68:  # Too hard — frustration zone
      return Adjustment(add_worked_example=True, increase_scaffold=0.20,
                        reduce_bloom=True)
    else:  # In zone — maintain
      return Adjustment(maintain=True)
```

**Worked example → faded example → generation** for concepts where first-encounter accuracy is below 50%:
- Worked example: full solution shown, student follows and predicts next step
- Faded example: partial solution, student fills in missing steps
- Generation: student produces the solution independently

Transition is mastery-gated: student moves to faded example when they predict correctly on the worked example, moves to generation when faded accuracy exceeds 0.75. Scaffolding fades as competence grows — Sweller's expertise reversal in automatic practice.

---

## Engine 4: The Content Pipeline

### What the LLM does (and doesn't) decide

The LLM fills slots. Every decision about what to teach, in what format, at what difficulty, whether to include a connection — those are made upstream. The LLM receives a fully-specified CardSpec and generates text that satisfies it.

Swap Claude for GPT-5: nothing about how the product teaches changes. That's the test.

### Format reasoning

Format is selected from the student's empirical history, not from learning style labels.

```python
def select_format(concept, student, time_budget_min) -> FormatSpec:
  # Base by concept type + student's measured response history
  history = student.format_response_history.get(concept.content_type)
  
  if history and history.accuracy_delta > 0.15:
    # Student shows consistent format advantage — use it
    base = history.best_format
  else:
    # No clear signal — use concept-type default
    base = CONCEPT_FORMAT_DEFAULTS[concept.content_type]
  
  # Video: only if process concept + ≥8 min + indexed segment exists + student watches
  include_video = (
    concept.content_type == PROCESS and
    time_budget_min >= 8 and
    video_index.has_segment(concept.id) and
    student.video_completion_rate > 0.60
  )
  
  return FormatSpec(primary=base, include_video=include_video,
                    include_diagram=diagram_adds_value(concept, base))
```

The video index is curated — timestamped segments from YouTube, Khan Academy, Crash Course — tagged to concept IDs, verified for accuracy, quality-scored. A concept only gets a video if a qualifying segment exists. No live YouTube search. Consistent quality.

### The self-reference effect, done correctly

When a structural connection exists (passes the 3-gate relevance engine: structural isomorphism, prior knowledge exists, setup under 35 words), the system doesn't deliver it. It asks the student to generate it.

*"Before we get into the mechanism — can you think of a situation from your own experience where you invest something upfront to get more back? 10 seconds."*

Student generates a connection. System confirms in one sentence and bridges to the concept.

This activates the self-reference effect (student evaluated concept against their own experience) and the generation effect (student produced the bridge) simultaneously. The d=1.11 Symons & Johnson meta-analysis is for explicit self-judgment tasks, not passive analogy delivery. V4 was delivering analogies. V5 elicits self-generation.

When no structural connection passes the gates: NONE. The system never manufactures one. The threshold (0.65 structural isomorphism) is treated as a hypothesis — validated empirically from 7-day retention data after 500+ uses.

### The dialogue layer

Two-way communication. Not a chatbot — a bounded retrieval system over the concept graph.

**After checkpoint failure**: one question before any reteach.

*"Before I explain — tell me in your own words what you think the answer is and where you got stuck."*

Student types freely. A 5-class classifier (fine-tuned on educational misconception data, or prompted Haiku at launch):

| Confusion type | What it means | Repair |
|---|---|---|
| PREREQUISITE_GAP | Reasoning reveals a missing prereq | Route to prereq; return to concept after |
| MISCONCEPTION | Holds a specific wrong belief | Name the misconception explicitly, contrast |
| TERMINOLOGY_ERROR | Understood the concept, wrong word | 10-second vocabulary correction, move on |
| PARTIAL_KNOWLEDGE | Got part right, missing sub-step | Confirm what's right, teach only the gap |
| COMPLETE_CONFUSION | No coherent understanding | Full reteach, different format, worked example |

The student who misread "ADP" as "ATP" gets a 10-second fix. The student missing a prereq gets routed to it. The student with an outright misconception gets it named and corrected. None of them get the same reteach. That's the difference between a tutor and a textbook.

**Student-initiated questions**: text field, always available. Student asks anything about the material. The system does KNN search over the concept graph (Qdrant, already built), retrieves 2-3 most relevant concept nodes, generates a targeted 1-3 sentence answer grounded in the source material. Not a general assistant — a bounded retrieval system. The student can get unstuck. They cannot go off-topic.

### Constraint enforcement

Every card is validated after generation before delivery:

```python
class CardValidator:
  def validate(self, card, spec) -> ValidationResult:
    # Structural: does it match the spec type?
    structural = card.type == spec.card_type
    
    # Length: within word count for this card type?
    length_ok = len(card.words) <= CARD_LIMITS[spec.card_type]
    
    # Pretest: does the question hint at its own answer?
    # Embedding similarity between question text and correct answer
    if spec.card_type == PRETEST:
      hint_score = cosine_similarity(embed(card.question), embed(card.correct_answer))
      no_hint = hint_score < 0.72
    
    # Seductive detail check: any content unrelated to this concept?
    # NER on card text, check each entity against concept vocabulary
    lean = not has_seductive_details(card, spec.concept)
    
    # Fact check: is every factual claim supportable from source material?
    grounded = fact_checker.verify(card.content, spec.concept.source_text)
    
    if not all([structural, length_ok, no_hint, lean, grounded]):
      return ValidationResult.FAIL  # Triggers Instructor retry
    return ValidationResult.PASS
```

Instructor + Pydantic handles retry automatically on validation failure. Max 3 retries; escalate to Sonnet on persistent failure. ~20–30% of cards require one retry. <2% require escalation.

### Cost architecture at scale

```
Simple cards (FORWARD_TEST, PRETEST, CHECKPOINT, VOICE):
  Model: claude-haiku-3-5 → ~$0.0001/card

Teaching cards (population pool HIT):
  Model: None (cached) → $0.00/card

Teaching cards (new concept):
  Model: claude-sonnet-4-6 → ~$0.003/card

Fact check pass:
  Model: claude-haiku-3-5 → ~$0.00005/card

Confusion classification (dialogue):
  Model: claude-haiku-3-5 → ~$0.00008/card
```

At 10K daily active students × 5 sessions × 8 cards: 400K cards/day.
- Before population pool: ~$90-120/day
- After 6 months (~60% pool hit rate): ~$40-55/day
- After 12 months (~80% hit rate): ~$25-35/day

The pool is simultaneously a quality improvement and a cost reduction. The best explanation for a high-frequency concept costs zero to serve.

---

## Engine 5: The Population Flywheel

Every student interaction is a data point. Nightly, the flywheel runs.

### The explanation tournament

For every concept with 50+ exposures and multiple explanation variants:

1. Compute 7-day retention per variant (delayed outcome — requires students who came back)
2. Where 7-day data is thin, use proxy: 0.4 × checkpoint_accuracy + 0.6 × day3_retention
3. Wilson confidence interval per variant (conservative lower bound)
4. Champion = variant with highest lower CI bound, above 0.65 threshold
5. Retire variants with upper CI below 0.55

The champion explanation for a high-frequency concept is served to future students before any LLM is called. After 100K students have studied cell biology, the system has empirically found the best explanation for every concept in the course. That took years of student data. It cannot be replicated by calling an API.

### Misconception detection

Wrong answers are embedded and clustered (DBSCAN). When a cluster represents 25%+ of wrong answers for a concept, it's a universal misconception — something about how the concept is commonly taught produces a predictable wrong belief. The system:
- Adds the misconception to the distractor pool for all future MC questions on this concept
- Builds a targeted correction card for students who haven't seen the correction yet
- Flags it in the concept node for the content pipeline

### Graph refinement

Nightly: for all concept pairs (A, B), compute P(fail B | fail A) from accumulated interactions. Compare to existing edge weights. Upgrade soft edges that exceed 0.70 empirical failure rate. Downgrade hard edges that fall below 0.30. Add new edges where no edge exists but P > 0.50.

Over time: the prerequisite graph moves from LLM-inferred (80% accurate) to behaviorally grounded (95%+ accurate). Every student is improving the graph for future students.

### Score model calibration

When students report actual exam scores (the product asks after every exam, frames it as helping future students), the predicted score model calibrates:
- Concept weights (which concepts actually appeared on the exam for this course type)
- Prediction accuracy (was the predicted score within 5%?)
- Forgetting curve parameters (did the stability estimates match actual retention?)

After 5,000 ground truth observations in a subject, the predicted exam score for that subject is genuinely predictive. That trust cannot be reverse-engineered by a new competitor.

---

## The Predicted Score

The home screen number. The behavioral hook that brings students back.

```
P(correct on concept C at exam time) = 
  P(mastered now) × exp(-days_to_exam / personal_stability(C))

Predicted exam score = 
  Σ_C [P(correct at exam | C) × exam_weight(C)]

Displayed as: "Predicted: 74% (C)"
```

It drops every day you don't study — the forgetting curves are running in real time. It climbs after sessions.

Below the number: **"What moves this most today"** — the 3 concepts where the marginal gain per study minute is highest. One tap. That's the session.

The number is calibrated against real exam scores. V1: rough estimate from mastery × slide emphasis weights. V2 onward: Bayesian update from every student who reported back their actual grade. The longer the product runs, the more accurate the number becomes. That accuracy is what produces the viral story: "It said 74%. I studied for 8 days. Got a 78%."

---

## The Notification System

Notifications are not reminders. They are alerts about the forgetting curve.

When a concept's retrievability (P(correct on surprise test right now)) drops below 0.65 — which FSRS can compute in real time — the system sends one notification:

*"Glycolysis is fading. 3 other concepts dropped below 65% today. Your predicted score fell from 78% to 74%."*

Not a streak reminder. Not a "You haven't studied in 2 days!" message. A factual update about the student's actual knowledge state, tied directly to the number they care about. The score fell. Here's why. Here's how long it will take to fix it.

Students respond to loss aversion (watching the number fall) more reliably than to streak incentives (Duolingo). The notification is engineered to trigger that loss aversion through genuine information, not manufactured urgency.

---

## The Session Flow, End to End

A student opens the app. Exam tomorrow. 25 minutes.

1. **Score update**: "Predicted: 71% (C-). 4 concepts need review." System detected their stability decaying overnight.

2. **Session start**: Scheduler runs the portfolio optimization. 25 minutes, MC exam. Selects 2 review concepts and 1 new concept. Explicitly skips 3 frontier concepts — "not enough time to master these today."

3. **Challenge calibration**: system checks their last session's rolling accuracy. They were at 84% — slightly too easy. Challenge manager flags: increase difficulty, prefer application questions.

4. **Review concept 1 (glycolysis)**: IRT selects the most informative question at their current ability estimate. Application-level question (not recall — they're past that). They answer. Sure. Correct. FSRS updates stability, schedules next review in 9 days.

5. **Review concept 2 (Krebs cycle)**: IRT selects question. They answer. Sure. Wrong. Dialogue prompt: *"Tell me what you think the answer is and where you got stuck."* They type: "I thought it was 2 NADH not 3." Classifier: TERMINOLOGY_ERROR. System: "You're thinking of glycolysis — that produces 2 NADH. The Krebs cycle produces 3 NADH per turn because it runs twice per glucose. Quick fix." 15 seconds. Retry. Correct.

6. **New concept (oxidative phosphorylation)**: Relevance engine runs — student has strong economics background from prior upload. Structural isomorphism check: "investment phase before payoff" maps. Gate 3: setup under 35 words. Approved. System: *"Before I explain — can you think of a situation where you make an upfront investment to earn more back?"* Student types: "compound interest." System: "Exactly. Oxidative phosphorylation works the same way — the electron transport chain invests in building a proton gradient to earn a net 32-34 ATP at the end." Diagram loads. Student reads. Checkpoint follows.

7. **Engagement check at minute 18**: rolling accuracy is 75%. Challenge manager: in zone, maintain. Response time trend: flat. Student is still engaged.

8. **Session end at minute 23**: two concepts reviewed, one taught. Score updates: 71% → 76%. System: "3 days until your review on oxidative phosphorylation. Check back tomorrow for glycolysis."

What the student experienced: a fast, adaptive session that felt like it knew what they needed. What happened underneath: 6 AI systems made 40+ decisions. None of them were visible.

---

## What This Architecture Beats, and What It Doesn't

**Beats**: every current study app (StudyFetch, Knowt, Gizmo, HyperKnow, Vaia, Quizlet) — all of which are tool generators that let the student decide what to do. Beats passive study methods (rereading, highlighting, summary notes) by approximately 0.5 SD in exam performance on comparable time budgets, based on retrieval practice research.

**Approaches but doesn't reach**: Bloom's 2-sigma. A 1:1 human tutor achieves +2 SD. The dialogue layer closes part of this gap — a confused student can now say so and get a targeted response. But the system cannot notice a student's expression change, cannot carry multi-turn reasoning through complex problem-solving, cannot pick up on the half-finished thought that reveals where comprehension broke. A mediocre tutor still outperforms this system on interaction quality. The system outperforms a mediocre tutor on scheduling, consistency, and population data.

**Honest compression estimate**: approximately 2x study efficiency vs. passive review for students who currently study by rereading and highlighting. Not 10x. The Alpha School 2x claim is about replacing passive lectures with active mastery — a different lever. This product compresses the review phase, not the instruction phase.

---

## The School Integration Path

This architecture is directly extensible to classroom replacement, not just study supplement:

**Phase 1 (now)**: Student uploads professor's slides. Uses the system to prepare for exams.

**Phase 2**: Canvas LTI integration. Student clicks "Study for this" next to any assignment. Syllabus pre-loaded, exam date pre-set, format configured. Zero friction. The system knows the course before the student logs in.

**Phase 3**: Partial lecture replacement. For concepts where the class has strong prerequisites, the system teaches directly and the lecture is optional. For concepts where the class is uniformly weak, lecture still happens. The system identifies which is which.

**Phase 4**: Alpha School model. Core academics in 2 hours per day. Student model has enough longitudinal data to sequence a full curriculum, not just a single course. Teacher's role: mentorship, projects, social-emotional support. The system handles instruction.

Phase 1 is what's built now. Phase 4 is the same architecture with a richer student model, a longer time horizon, and more longitudinal data. The path is direct.

---

## Final Table: What Each System Does

| System | Decision made | Timescale | When it updates |
|---|---|---|---|
| Document Intelligence | What concepts exist + how they depend on each other | At upload (once) | Graph refined nightly from behavioral data |
| DKT-GAT | What does this student know right now | Per interaction (<50ms) | Weights fine-tuned weekly |
| FSRS | When will this student forget each concept | Per concept | Parameters fit nightly |
| IRT | Which question maximizes information at this student's ability | Per checkpoint | Parameters fit nightly |
| Student Intelligence File | Who is this student, what works for them | Per session | Updated after every session |
| Scheduler | What to teach, when, at what difficulty | Per session (20ms) | Phase 1: deterministic; Phase 2+: learned |
| Challenge Manager | Is the student in the productive zone right now | Per 5 checkpoints | Real-time rolling update |
| Format Reasoner | What form should this concept take | Per concept | Driven by SIF format history |
| Relevance Engine | Is this connection real enough to use | Per candidate connection | Threshold validated from behavioral data |
| Content Pipeline | What words should appear on the card | Per card (300-1200ms) | Champion pool grows nightly |
| Dialogue Layer | What kind of confusion is this, what's the targeted repair | Per failed checkpoint | Classifier improves with labeled data |
| Population Flywheel | Which explanation actually works, what misconceptions exist | Nightly batch | Compounds with every student |
| Exam Readiness Model | What score will this student likely get | Live (continuous) | Calibrated from reported exam scores |

**The LLM appears in exactly one row**: Content Pipeline. It generates words. Every other decision in this table is made by a different system. That is why this is not a GPT wrapper.

---

*This is the architecture. Everything prior was a draft.*
