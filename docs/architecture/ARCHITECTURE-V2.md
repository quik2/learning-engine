# The Learning Engine — Deep Architecture
*JARVIS. March 2026. Second pass. Harder.*

---

## What Changed From V1

V1 was a competent seven-layer system. It was correct. It was also something a senior engineer at a well-funded startup could have written on a whiteboard. Good architecture. Not revolutionary.

The gap between a well-funded competitor and a generational company is not in the list of components — it's in the theory of what those components are actually doing and why. V2 starts there.

---

## The Real Problem, Stated Precisely

Every student who fails an exam had access to the same material as the student who passed. The bottleneck is not information access. It has never been information access.

The bottleneck is three things, precisely:

**1. Retrieval structure.** Knowledge encoded passively — through reading, watching, listening — forms shallow traces. Re-reading the clean explanation produces the fluency illusion: the material feels understood because recognition is triggered. But recognition is not retrieval. At exam time, only retrieval exists. Shallow traces cannot be retrieved under the time pressure, partial cue, and interference conditions of a real exam.

**2. Forgetting dynamics.** Memory is not a state — it's a process. A concept "learned" on Day 1 is mostly gone by Day 4 without reinforcement. Students don't know their own forgetting rates. They study a week before, feel ready, and discover two days before the exam that they've forgotten 60% of it. The problem isn't that they didn't study enough — it's that they studied at the wrong times.

**3. Metacognitive blindness.** Students don't know what they don't know. Worse, they think they know what they don't. The Dunning-Kruger effect is not a personality quirk — it's a structural feature of how knowledge representation works. You don't know you don't know something until you fail to retrieve it under pressure. Passive learning generates false confidence. The product's entire existence is justified by this single finding.

The thesis: **a system that solves all three simultaneously — for every student, for every concept, on any material they upload — does not exist yet.** Every existing tool solves at most one. This is the company.

---

## The Theoretical Foundation

Before architecture, theory. The system is built on four converging scientific traditions. Understanding them determines every architectural decision.

### 1. The Encoding-Retrieval Specificity Principle (Tulving, 1983)
Memory retrieval succeeds when the retrieval cue matches the original encoding context. This is why studying with flashcards and then being tested with multiple choice fails — different retrieval context. The practical implication: the format of practice must match the format of test. For every student, on every concept, the question format during practice should mirror the format they'll see on the exam. This is not optional. It's mechanistic.

**Architectural implication:** Test format (MC, short answer, free recall, application) is a first-class parameter throughout the entire system. It affects lesson structure, checkpoint format, and review question design. A student preparing for a MC exam and a student preparing for an essay exam on the same material should have fundamentally different lesson experiences.

### 2. The Testing Effect Is Not About Feedback
The dominant model says: test yourself → get feedback → learn from the feedback. This is wrong as a mechanism. The testing effect occurs even without feedback (Roediger & Karpicke, 2006). The act of retrieval itself strengthens the memory trace, independent of knowing whether you were right. Feedback amplifies the effect but does not produce it.

**Architectural implication:** Checkpoints cannot be optional, skippable, or replaced by "see the answer" buttons. The cognitive work of attempting retrieval — even a failed attempt — is the pedagogical mechanism. Any system that lets students skip to the answer is not using the testing effect. It's simulating it.

### 3. Desirable Difficulties Are Mechanism, Not Metaphor
The term "desirable difficulty" is often treated as a vague motivational concept. It is not. It refers to a specific class of encoding conditions that reduce performance during acquisition but increase durable retention. The mechanism is this: conditions that make retrieval harder during practice force a deeper and more elaborate memory encoding. The additional cognitive work at encoding creates more retrieval routes.

The conditions with strongest evidence:
- Spacing > massing (forces reconstruction from partial traces, not recognition)
- Testing > restudy (forces active retrieval rather than passive recognition)
- Interleaving > blocking (forces category discrimination, not just category identification)
- Generation > reading (self-generated information has more distinctive encoding context)
- Variation in retrieval format (prevents format-specific retrieval cues from dominating)

**Architectural implication:** All five of these must be non-optional structural features. Not options. Not features the student enables. The system must enforce them architecturally because the research is unambiguous: if you let students choose, they choose wrong.

### 4. The Expertise Reversal Effect (Kalyuga, 2003)
Instructional techniques optimal for novices become harmful for experts. A fully worked example helps a novice by reducing extraneous cognitive load. The same worked example harms an expert because it adds extraneous processing of already-known information, interfering with the formation of more abstract schemas. Scaffolding must fade proportionally with mastery.

**Architectural implication:** The amount of instructional support on a given concept must be a function of current mastery — not a static design decision. A student's first encounter with glycolysis requires maximal scaffolding: worked examples, analogies, step-by-step decomposition, prior knowledge activation. Their fifth encounter with glycolysis (review before the exam) requires minimal scaffolding: direct retrieval prompt, no prior knowledge activation, no hints.

The system must model mastery with sufficient precision to know where each student is in the novice-to-expert progression on each concept, and adjust scaffolding continuously.

---

## The Architecture

### Layer 0: The Epistemological Layer (What Makes This Not a Wrapper)

Before describing any component, the philosophical decision that makes this work:

**The LLM is not the teacher. It is the voice of the teacher.**

The teacher is the system. The teacher knows the student (Layer 3). The teacher decides what to teach and when (Layer 4). The teacher decides how hard to push (Layer 4). The teacher knows what has worked for 10,000 prior students on this exact concept (Layer 6). The LLM executes the teacher's decisions in language.

This is not a semantic distinction. It is the entire product difference. When a student pastes slides into ChatGPT, ChatGPT is both the teacher and the voice — it makes all decisions and it speaks them. It has no student model. It has no concept graph. It has no scheduling algorithm. It has no population-level data. It decides what to say based on what it predicts the next token to be, which is a function of its training corpus, not a function of this student's knowledge state right now.

When a student uses this system, the LLM receives a structured context packet assembled by four other layers. It does not decide what type of interaction to have. It does not decide what to cover. It does not decide whether to give away the answer. It fills a template whose structure enforces the pedagogy. Swap Claude for Gemini 2 Flash and nothing about the educational system changes. That is the proof.

---

### Layer 1: Document Intelligence Pipeline

**What competitors do:** PyPDF2 text extraction. Maybe some chunking. Into a vector store. Done.

**The problem with that:** A slide deck is not text. It is a visual spatial argument. The relationship between a diagram and its caption is a pedagogical unit. The hierarchy of heading → bullet → sub-bullet encodes the logical structure of an argument. The visual emphasis of bolded terms signals cognitive importance. Text extraction throws all of this away and treats slides as a bag of words.

**What this does:**

*Stage A — Parallel dual-modal extraction:*

Path 1 (Text): MinerU processes the raw PDF/PPTX. Produces structured markdown that preserves heading hierarchy, lists, tables, formulas (LaTeX), and footnotes. Removes headers/footers/slide numbers. Preserves semantic coherence of paragraphs. Output: `StructuredText` object with positional metadata.

Path 2 (Visual): Every slide rendered at 2048px. Claude 3.5 Sonnet (or Gemini 2.0 Flash for cost optimization) analyzes each slide image as a VLM. Not for summary — for structural analysis. Outputs a `SlideAnalysis` JSON:
```json
{
  "slide_number": 4,
  "visual_emphasis": ["glycolysis", "net ATP"],
  "diagram_present": true,
  "diagram_description": "Linear pathway diagram showing glucose→pyruvate, annotated with ATP investment and yield steps",
  "diagram_concepts": ["glucose-6-phosphate", "fructose-1,6-bisphosphate", "pyruvate", "ATP"],
  "spatial_relationships": [
    {"from": "glucose", "to": "pyruvate", "relation": "converts_to"},
    {"from": "investment_phase", "to": "yield_phase", "relation": "precedes"}
  ],
  "pedagogical_emphasis": "Visual weight on net ATP (circled, larger font) suggests this is the testable fact",
  "content_type": "process_diagram",
  "likely_misconception_sources": ["gross vs net ATP confusion"]
}
```

*Stage B — Content classification:*
Each content block (sentence, bullet, diagram unit, table row) is classified into a pedagogical content type:
- `definition` — a concept with its formal meaning
- `worked_example` — a concrete instance of an abstract principle  
- `analogy` — a structural comparison to prior knowledge
- `edge_case` — a condition where normal rules break down
- `fact` — a discrete retrievable fact
- `process` — a multi-step mechanism
- `principle` — an overarching rule that applies across instances
- `diagram_concept` — a concept only fully defined by the visual

Content type determines downstream treatment. A `definition` gets a different lesson card structure than a `worked_example`. An `edge_case` gets a different review schedule (needs more reinforcement — students consistently underlearn edge cases).

*Stage C — Quality scoring:*
The pipeline scores extraction confidence per slide. Confidence is a function of: text/visual alignment, formula extraction completeness, diagram concept coverage, heading structure completeness. Low-confidence slides are flagged in the student UI: "We had trouble reading slide 12 — the diagram may not be fully captured. Consider cross-referencing your notes." Transparency about quality is better than confident wrong content.

---

### Layer 2: The Concept Graph Engine

**What makes this different from a vector store:**

A vector store stores chunks and retrieves by similarity. It has no understanding of dependency, difficulty, or pedagogical ordering. Retrieval is purely semantic proximity. That's sufficient for a chatbot. It is not sufficient for a teaching engine.

This layer builds a **Directed Pedagogical Knowledge Graph (DPKG)** — not a knowledge representation in the philosophical sense, but a pedagogical graph: what needs to be understood before what can be understood.

**Node schema:**
```python
@dataclass
class ConceptNode:
    id: str                          # Stable UUID
    name: str                        # Canonical name
    definition: str                  # Formal definition
    aliases: List[str]               # Alternative names ("ETC", "electron transport chain")
    bloom_level: BloomLevel          # REMEMBER/UNDERSTAND/APPLY/ANALYZE/EVALUATE/CREATE
    content_type: ContentType        # definition/process/principle/etc.
    common_misconceptions: List[Misconception]  # Detected from population data over time
    difficulty_estimate: float       # 0-1, updated from population performance
    estimated_teaching_time_min: float
    visual_representation: Optional[str]  # Reference to diagram, if available
    source_slides: List[int]
    embedding: np.ndarray            # High-dim semantic embedding
    cross_course_uid: Optional[str]  # Links to same concept across different uploads
```

**Edge schema:**
```python
@dataclass
class PrerequisiteEdge:
    source_id: str
    target_id: str                   # source is prerequisite for target
    strength: PrerequisiteStrength   # HARD (can't understand without), SOFT (helps), CONTEXTUAL
    confidence: float                # How confident we are this relation holds
    evidence_source: EvidenceSource  # LLM_INFERRED, BEHAVIORAL, ONTOLOGY_GROUNDED
    behavioral_weight: float         # P(fail target | fail source) from population data
```

**Prerequisite detection — three signal fusion:**

Signal 1 (LLM inference): Pairwise prompting with concept definitions. "Given this definition of A and this definition of B, does understanding A help understand B? Is it required or just helpful?" Produces initial edge set. Accuracy ~80% for hard prerequisites, ~55% for soft.

Signal 2 (Ontology grounding): Cross-reference concept names against subject-domain ontologies. OpenStax textbook concept graphs (available via API). Wikipedia category hierarchies for the subject domain. Khan Academy skill prerequisites (scraped, structured). Subject-specific: UMLS for biology/medicine, MathSciNet for mathematics, ACM CCS for CS. Where the LLM-inferred graph conflicts with ontology, flag for verification.

Signal 3 (Behavioral validation): After 200+ students have studied the same concept set, compute P(fail target | fail source) for all concept pairs. This is behavioral prerequisite strength — it doesn't tell you *why* one is prerequisite to another, but it tells you the measured consequence. This is the ground truth. It supersedes LLM inference and ontology over time.

**Cross-document synthesis:**
A student who uploads three lectures gets one unified DPKG, not three separate ones. When Lecture 3 introduces glycolysis, the engine checks whether glycolysis exists in the graph from Lectures 1-2. If so, it retrieves the mastery state for this student (Layer 3) and does not re-teach from scratch. It only teaches the delta: what is new in this lecture's treatment of the concept.

**The interference map:**
Every concept pair gets a semantic distance score (cosine similarity on embeddings). High-similarity pairs (glycolysis/gluconeogenesis, mitosis/meiosis, proactive/retroactive interference) are flagged as potential interference risks. The interference map is consumed by Layer 4.

---

### Layer 3: The Student Model

**This is the most important layer. It is also where the most interesting ML happens.**

The student model maintains, at every point in time, a complete probabilistic belief state over the student's knowledge: for every concept in their graph, a distribution over mastery levels.

**The core representation:**

For each student S and each concept C, the model maintains:
- P(mastered | evidence) — probability of stable retrieval under exam conditions
- stability — FSRS stability parameter: expected interval before 90% retention drops to 50%
- difficulty — FSRS difficulty: how intrinsically hard this student finds this concept
- recent_responses — last N responses with timestamps (for DKT input)
- confidence_history — paired (stated_confidence, actual_correct) tuples
- response_time_history — milliseconds per question attempt
- error_patterns — which specific wrong answers have been chosen

**The model stack:**

*Primary: Deep Knowledge Tracing with Graph Attention (DKT-GAT)*

Standard DKT (Piech et al., 2015) uses an RNN over (concept, correct) sequences. State of the art in 2026 combines DKT with Graph Attention Networks to incorporate concept graph structure. The GNN propagates mastery information along prerequisite edges: if the model sees that a student has mastered all prerequisites of a concept, it adjusts upward its prior for that concept. If a student fails a concept, it propagates probability mass toward failure of concepts that depend on it.

Architecture: 
- Input: sequence of (concept_id, response, response_time, confidence, bloom_level) tuples
- GNN layer: propagates mastery over DPKG edges using attention weights proportional to prerequisite strength
- Transformer encoder: models temporal dependencies in response sequence
- Output head: P(correct on next attempt | context) for every concept

Pretrained on: ASSISTments (400K students, 4M interactions), EdNet (780K students, 130M interactions), open-sourced Duolingo language data. Fine-tuned on this system's accumulating data after 10K student-sessions.

*Secondary: Item Response Theory (2PL-IRT)*

IRT doesn't track sequences — it models the relationship between student ability and item difficulty. For each question in the question pool:
- `a` parameter: discrimination (how well this question separates students by ability)
- `b` parameter: difficulty (what student ability level gives 50% chance of correct response)

As the system accumulates responses, IRT parameters are fit per question. This gives the scheduler something DKT doesn't: a way to select the most informative question to show next, given current uncertainty about student ability on a concept. The scheduler uses Information Gain (selecting the question that maximally reduces uncertainty about mastery) when the student is near the mastery boundary.

*Response time as a second signal:*

Response time is radically underused in existing systems. The ALEKS system collects it; most others ignore it. 

Response time is a noisy but real signal about processing fluency:
- Very fast correct response → probably automatic retrieval, high confidence justified
- Very fast incorrect response → probably guessing (high slip probability)
- Very slow correct response → correct but effortful, fluency not achieved, reschedule sooner
- Very slow incorrect response → genuinely working through it, not guessing

A response time model (log-normal distribution per concept per student) is fit from historical data and used to adjust slip/guess probabilities in DKT, improving mastery estimation accuracy.

*Confidence calibration tracking:*

For every (stated_confidence, actual_correct) pair, the system maintains a calibration curve: how well-calibrated is this student? A student who says "sure" and gets 80% right is well-calibrated. A student who says "sure" and gets 40% right is systematically overconfident. The calibration coefficient is used to:
1. Adjust the mastery estimate for confidence-tagged responses (overconfident "sure/wrong" is a stronger signal than well-calibrated "sure/wrong")
2. Show the student their calibration explicitly: "When you say you're confident, you get it right 42% of the time. When you say you're unsure, you get it right 61% of the time." This is metacognitive feedback that research shows improves subsequent calibration and performance.

*The metacognitive mirror:*

The system knows what the student knows. More unusually, it knows what the student thinks they know. The gap between these two is the metacognitive blindness gap. When the gap is large (student confident on a concept where mastery is low), the system:
1. Schedules a surprise retrieval attempt — show the student they're wrong
2. Flags the concept as metacognitively-dangerous: the student will under-prioritize studying it
3. Surfaces it in the exam readiness report: "Warning: you're confident about mitosis but your retrieval accuracy is 34%"

This is a feature no competitor offers because it requires a student model sophisticated enough to track both objective mastery and subjective confidence independently.

---

### Layer 4: The Scheduling Engine

**Pure algorithm. No LLM. No ambiguity. Complete determinism.**

The scheduling engine takes the complete student model state and produces: (a) the optimal teaching sequence for today, and (b) the optimal review queue for today.

**Foundational algorithm: Successive Relearning with FSRS**

Successive relearning (Rawson & Dunlosky, 2013, 2022) is the highest-leverage learning technique with no adoption. The procedure:
1. Study until criterion (3 consecutive correct retrievals in a single session)  
2. Wait (days)
3. Re-study until criterion again
4. Wait (longer)
5. Repeat

The FSRS (Free Spaced Repetition Scheduler) implements the spacing component: after each session-level criterion, it computes the interval until the next required session using a model of memory stability and retrievability:

```
R(t) = e^(-t/S)  # Retrievability at time t given stability S
```

Where stability increases proportionally to retrievability at review time — you get more benefit from reviewing when you still remember it somewhat than when you've completely forgotten. FSRS fits S and D (difficulty) parameters per student per concept from their response history.

**Combined algorithm:**

```
Concept_State = {
  mastery_level: P(mastered),        # from DKT-GAT
  stability: S,                       # from FSRS  
  next_review_due: timestamp,
  times_to_criterion_achieved: int,  # how many complete relearning cycles
  current_session_correct_streak: int
}

Graduated = mastery_level > 0.85 AND 
            times_to_criterion_achieved >= 2 AND
            current_stability >= target_stability(days_until_exam)
```

A concept is not done when the student gets it right once. It is done when successive relearning criterion is met, at stability sufficient to survive until the exam.

**Interference-aware scheduling (LECTOR-extended):**

The LECTOR paper (arxiv 2508.03275, 2025) showed that scheduling algorithms that ignore semantic interference between concepts have systematically suboptimal review intervals. High-similarity concepts interfere with each other's consolidation.

This system implements an extended LECTOR formulation:

```
effective_stability(concept_i) = base_stability(i) × 
  Π(1 - α × semantic_similarity(i, concept_j) × recency_weight(j))
  for all j ≠ i in recent_review_window
```

Where:
- `semantic_similarity` is cosine distance between concept embeddings
- `recency_weight` decays exponentially with time since concept_j was reviewed
- `α` is a learned interference coefficient (fit from behavioral data)

In practice: glycolysis and gluconeogenesis should not be reviewed in the same session, or even on the same day. The scheduler inserts a minimum temporal distance between semantically adjacent concepts. This reduces confusion-driven errors by ensuring consolidation is complete before a confusable concept is introduced.

**Interleaving protocol:**

Research (Kornell & Bjork, 2008; Rohrer & Taylor, 2007; PMC12108632, 2025) shows that interleaved practice of confusable categories improves discrimination. The mechanism: interleaving forces students to notice *what distinguishes* concepts, not just *what characterizes* each one.

Scheduling rule: for concepts in a semantically similar cluster (mitosis/meiosis, oxidative/substrate-level phosphorylation, proactive/retroactive interference), after initial separate teaching, review sessions interleave them rather than reviewing each to criterion before the other. The scheduler identifies semantically similar clusters from the interference map and applies interleaving during review phases.

**Deadline compression:**

As exam date approaches, the scheduler solves an optimization problem: given N concepts at varying mastery levels, M minutes of available study time, and T days until the exam — what allocation maximizes expected exam score?

This is a variant of the Secretary Problem / optimal stopping problem. The solution is not trivially "start with the lowest-mastery concepts." Low-mastery concepts may take more time than available. The scheduler:
1. Projects expected score contribution of each concept: `P(appears on exam) × P(correct at exam time | current mastery, schedule)`
2. Optimizes allocation to maximize total expected score
3. Under strict time constraints, may explicitly deprioritize some concepts ("dropping these 3 concepts from your review schedule because you won't have time to master them; focusing here maximizes your expected score")

This is a decision that currently requires a human tutor. No app makes it.

**Adaptive scaffolding schedule:**

Per concept per student, the scaffolding level decreases with mastery, implementing the expertise reversal effect. The scaffold parameter controls:
- How much prior knowledge is activated before teaching (more for novices)
- Whether worked examples or just direct prompts are shown (worked examples for novices)
- How many hint levels are available at checkpoints (more for novices)
- Whether analogies are included (usually for novices; experts find them condescending)

```python
def scaffold_level(mastery: float, bloom_target: BloomLevel) -> ScaffoldConfig:
    base = 1.0 - mastery           # High mastery → low scaffolding
    bloom_factor = bloom_target.depth_factor  # Deeper Bloom levels get more scaffolding
    return ScaffoldConfig(
        activate_prior_knowledge = base > 0.6,
        include_worked_example = base > 0.5,
        include_analogy = 0.3 < base < 0.8,
        hint_levels = int(base * 3),
        reveal_answer_after_n_attempts = max(1, int(base * 4))
    )
```

---

### Layer 5: The Content Generation Architecture

**This is where the hardest engineering problems live.**

The problem with current LLM-based educational content: LLMs are trained to be helpful, which in educational contexts means they want to explain things, not force students to work. An LLM asked to generate a "Socratic question" will often generate a question that reveals the answer in the question itself. It is biased toward completion, not toward productive struggle.

**The solution: Structural constraints enforced in code, not in prompting.**

The content pipeline receives a `LessonSpec` from the scheduler:

```python
@dataclass  
class LessonSpec:
    student_id: str
    concept: ConceptNode
    session_type: SessionType    # INITIAL_TEACH, REVIEW, DIAGNOSTIC
    bloom_target: BloomLevel
    scaffold_config: ScaffoldConfig
    test_format: TestFormat      # MC, FREE_RECALL, SHORT_ANSWER, APPLICATION
    time_budget_min: float
    prior_errors: List[ErrorRecord]
    confidence_history: CalibrationHistory
    population_explanation_pool: List[ExplanationVariant]  # Winning explanations from cross-student data
    interference_context: List[ConceptNode]  # Recent semantically-adjacent concepts
    card_sequence: List[CardType]  # Fully determined by algorithm, NOT by LLM
```

The `card_sequence` is determined algorithmically. Every card type has hard constraints on what the LLM is permitted to generate:

```python
CARD_CONSTRAINTS = {
    CardType.FORWARD_TEST: {
        "must_not_contain": ["the answer", "glycolysis", "pyruvate"],
        "must_activate_prior_knowledge": True,
        "answer_must_not_be_inferrable_from_prompt": True,
        "max_words": 80
    },
    CardType.PRETEST: {
        "is_question": True,
        "must_not_hint_at_answer": True,
        "distractors_must_be_real_misconceptions": True,
        "confidence_required": True,
    },
    CardType.TEACH: {
        "max_concepts_per_card": 1,
        "must_not_include_tangential_interesting_facts": True,
        "seductive_details_prohibited": True,
    },
    CardType.CHECKPOINT: {
        "student_answers_before_seeing_correct": True,
        "cannot_be_skipped": True,
        "confidence_required": True,
        "distractors_must_differ_from_population_misconceptions": False,  # Use population misconceptions
    },
}
```

LLM output is validated programmatically against card constraints before display. If a generated pretest question hints at the answer in the question text (detectable via re-prompting: "does this question reveal its own answer?"), it is regenerated. This validation loop runs silently, adding ~200ms to generation but ensuring pedagogical integrity.

**Population explanation injection:**

Before generating any explanation for a concept, the pipeline checks the explanation pool (Layer 6). If concept C has accumulated enough behavioral data:
- Pool size ≥ 20 validated explanations
- Top-1 explanation has ≥ 200 student exposures and ≥ 65% checkpoint accuracy
→ The top explanation is injected directly into the card. LLM is not called. The historically best explanation for this concept is served.

This means over time: the LLM is called less and less for high-frequency concepts. The population's collective learning has already found the best explanation. The system serves it deterministically.

**Misconception-aware distractor generation:**

For multiple-choice questions, distractors are not randomly generated. They are drawn from two sources:
1. Population-validated misconceptions (concepts that share high embedding similarity with the correct answer and are empirically chosen wrongly)
2. Targeted student errors (wrong answers this specific student has chosen before)

This is not cosmetic. Distractors that target real misconceptions are 3-4x more effective at revealing knowledge gaps than random plausible wrong answers.

**Hallucination grounding:**

Every piece of factual content generated by the LLM is verified against the source document. Process:
1. LLM generates card content
2. Verifier model (second LLM call, or Claude's extended thinking for efficiency) checks: is every factual claim in this card supportable from the source document?
3. Claims that cannot be grounded are flagged; card is revised or the claim is removed
4. Cards that require domain-level knowledge beyond the source document (e.g., student asks about something not in the slides) are flagged as "not from your slides" and optionally blocked

---

### Layer 6: The Cross-Student Learning Engine

**This is the moat. This is what a competitor cannot build in a year.**

Every student interaction generates signal. The question is what to do with it. Every competitor throws it away.

**The behavioral data schema:**

```python
@dataclass
class StudentInteraction:
    student_id: str                  # anonymized
    concept_id: str
    session_id: str
    card_type: CardType
    explanation_variant_id: str      # Which explanation was shown
    response: str                    # Student's answer
    correct: bool
    confidence: ConfidenceLevel      # sure/not sure
    response_time_ms: int
    session_position: int            # 3rd card in session
    days_since_last_review: float
    preceding_concepts: List[str]    # Recent session context
    exam_date: Optional[date]        # Days until exam
```

**What is computed on this data:**

*Explanation tournament:*

For every concept that has accumulated ≥ 100 exposures across multiple explanation variants:
1. Compute checkpoint accuracy per variant: P(correct at checkpoint | shown variant V)
2. Compute 3-day retention per variant: P(correct at Day+3 review | shown variant V at initial teach)
3. Compute 7-day retention per variant (if data exists)
4. Bayesian Thompson sampling: variants with uncertain performance get exploration probability; variants with strong evidence get exploitation probability
5. Winner is promoted to the explanation pool; losers are retired below a performance threshold

The tournament runs weekly. After a year, the winning explanations for high-frequency concepts across STEM subjects represent the best-performing educational content ever assembled — generated by AI, validated by millions of students, continuously improving.

*Universal misconception detection:*

Algorithm: for each concept, cluster wrong answers by embedding similarity. If a wrong answer cluster has P(chosen | wrong) > 0.3 across the population, it is a universal misconception.

When a universal misconception is detected:
1. A dedicated misconception-correction card is added to the concept's card sequence
2. The misconception is added to the distractor pool for all future checkpoint questions on this concept
3. It is added to the `ConceptNode.common_misconceptions` field, visible to the content generation layer

Effect: the system learns what students consistently misunderstand, and proactively addresses it before students even encounter it. After 100K students study cell biology, the system knows every systematic error students make in cellular respiration and has built targeted corrections into the lesson.

*Prerequisite graph behavioral refinement:*

Nightly batch process:
1. For all concept pairs (A, B) in the graph: compute P(fail B | fail A, passed all other prerequisites of B)
2. If P > 0.6 and edge is currently SOFT → upgrade to HARD
3. If edge is currently HARD and P < 0.2 → downgrade to SOFT (maybe the LLM was wrong)
4. If no edge exists and P > 0.5 → create SOFT edge, flag for review

The graph learns from reality. Over time, the prerequisite graph becomes more accurate than any expert-authored curriculum.

*Cross-concept transfer detection:*

A student who has mastered concept A on one upload may automatically have partial mastery of concept B on a new upload, if A and B are highly semantically similar. The system detects this via embedding similarity and does not re-teach from scratch.

Example: A student who uploaded their Biochemistry notes and studied protein secondary structure. When they upload their Cell Biology notes, the system detects that "protein folding" (Cell Bio) shares a 0.87 cosine similarity with "protein secondary/tertiary structure" (Biochem) and initializes that concept with partial mastery instead of zero mastery.

This is a moat: the longer a student uses the system, the faster they learn new material, because the cross-course knowledge graph grows.

---

### Layer 7: The Exam Readiness Model

**Not a bar. Not a percentage. A probability distribution.**

The exam readiness number is the product's single most important output. It must be:
1. Legible (one number, always visible)
2. Accurate (if it says 74%, student should score around 74%)
3. Actionable (it must change based on what the student does)
4. Responsive (it must drop visibly when the student doesn't study — urgency)

**The model:**

Expected exam score = Σ over all concepts C:
```
P(concept C appears on exam) × P(student recalls C correctly at exam time)
```

where `P(student recalls C correctly at exam time)` is:

```
P_mastery(C, now) × e^(-decay(C, student) × days_until_exam)
```

where `decay(C, student)` is the personal forgetting rate, estimated from FSRS stability parameters.

This produces a live, decaying estimate. If the student doesn't study, every concept's mastery estimate decays toward zero over days. The number falls. If they complete a session, mastery updates upward. The number rises.

**Calibration:**

V1 (launch): concept weights estimated from: slide emphasis (more slides devoted to a concept → higher weight), explicit statement ("this will be on the exam"), Bloom's level of questions in the slide deck, and subject-domain typical weighting (for bio: mechanisms > definitions > historical facts).

V2 (after data): concept weights learned from ground truth. Students who report back their exam score and the exam's question distribution become training data. The model learns P(concept C appears on exam for course type X, professor style Y, test format Z). Over time, the exam prediction becomes the most accurate academic performance predictor available for college students.

**The action panel:**

Below the number: "What's pulling this down?"

The system identifies the top 3 concepts where:
`(exam_contribution_if_mastered - expected_contribution_at_current_mastery) × urgency_weight`

is highest. These are the highest-leverage concepts to study today. The student doesn't need to understand the scheduling algorithm. They just need to tap the button.

This creates the product behavior without requiring the student to understand or choose the correct behavior. The correct behavior is what the product makes easy.

---

### Layer 8: The Metacognitive Interface

**The part no competitor has considered.**

Most education apps hide their model from the user. The student presses a button, content appears. The student has no visibility into what the system thinks they know.

This system surfaces the model.

**The Knowledge Map:** A visual graph of all concepts in the student's material. Node size = exam weight. Node color = mastery level (red → orange → yellow → green). Edge = prerequisite. The student can see: which concepts they've mastered, which are next, which are blocking others.

**The Calibration Report:** "When you said you were confident last week, you got it right 61% of the time. When you said you were unsure, you got it right 44% of the time. This means your confidence is a real signal — trust it. [vs:] Your confidence is not calibrated — you're overconfident. Here are the 3 concepts where you're most overconfident."

**The Forgetting Projection:** For each mastered concept: "You last reviewed this 4 days ago. Based on your personal forgetting rate, your recall probability is now 64% and dropping. Review today or it'll be below 50% by tomorrow."

**Why this matters:** Students who understand why the system does what it does comply with the schedule more. More importantly, students who see their own forgetting curves start to viscerally understand why passive re-reading is insufficient. The metacognitive interface is not UX polish — it is behavioral change infrastructure.

---

## The Data Flywheel At Scale

After enough data, three things become true that cannot be achieved in any other way:

**1. The explanation library becomes a proprietary asset.** 10,000 candidate explanations for "glycolysis," tournament-tested against millions of student responses, with the winning explanation serving to future students at zero marginal cost and zero API call. This cannot be reproduced by calling GPT-5. It required the students.

**2. The prerequisite graphs become empirically grounded.** Every LLM-inferred prerequisite edge has been validated or refuted by behavioral data. The graph knows, with confidence derived from millions of student interactions, that if you don't understand substrate-level phosphorylation, you will fail chemiosmosis 73% of the time. That number came from students. No amount of LLM inference produces it.

**3. The exam readiness model becomes genuinely predictive.** After students report back 50,000 actual exam scores, the model can say: "students with this mastery profile in BIOC 2A at UCLA score 74% ± 6% on the midterm." That's a product capability that competes with tutoring, not with flashcard apps.

---

## What Makes This Revolutionary (The Honest Version)

The architecture above is sophisticated. But sophistication alone is not revolution. Revolution means: if this works, it changes something.

Here is what changes:

**The student model is the teacher.** Not the LLM. The LLM is infrastructure. The student model — the one that knows their forgetting rate, their metacognitive blindness, their response time patterns, their misconception history — that is the teacher. Building that model better is the product's core work. Every improvement to Layer 3 is directly a teaching improvement.

**The population is the curriculum designer.** The cross-student learning engine means the curriculum is not static. It is not designed once. It is continuously revised by the collective behavior of every student who has studied the same material. The explanation that works is the one that empirically produces retention, not the one that seems pedagogically sound to an expert. This is a fundamentally different epistemology for curriculum design.

**Scaffolding that fades automatically is the end of the textbook model.** Every textbook is written at one scaffolding level for one imagined reader. Every MOOC is filmed once and served forever. This system presents the same concept at 1000 different scaffolding levels, personalized to this student's expertise state right now, and fades the scaffolding automatically as they improve. This is what expert human tutoring does. No software has done it before because no software has maintained a student model precise enough to know when to fade.

**The metacognitive mirror is the end of the fluency illusion.** When the system tells you: "You think you know mitosis. Your confidence is 9/10. Your retrieval accuracy in our model is 32%,"  — the fluency illusion breaks. Students have never had access to this information before because no system has been tracking it. This feature alone produces better learning outcomes than the entire study session, because metacognitive calibration is a prerequisite for self-regulated learning.

---

## The Risks (The Real Ones)

**The slide quality problem is worse than it appears.**

Professor slides are often not just incomplete — they are pedagogically incorrect. A professor may teach a concept using a model that simplifies it for intro students in a way that is technically wrong. The system that builds a knowledge graph from those slides will teach the technically-wrong simplified model with confidence. The student will learn something false. When the exam question reveals the false teaching, the student blames the product. This is an existential risk at launch.

Mitigation: subject-domain ontology cross-validation. When a concept node is created from slides, its definition is compared to trusted domain sources (Wikipedia, OpenStax, Khan Academy). Significant disagreements are flagged. The student is told: "This definition comes from your slides. Other sources define it differently — see note." Not a product-killer if handled honestly.

**The data cold start problem is deeper than stated in V1.**

DKT-GAT pretraining on Assistments/EdNet gives the model knowledge tracing capability for K-12 math and science. It does not give it capability for second-year biochemistry at UCLA, or philosophy of mind at Johns Hopkins, or constitutional law at Harvard Law. Fine-tuning requires domain-matched behavioral data.

Mitigation: launch in courses where pretraining data is abundant (intro bio, calculus, physics, general chemistry). Build domain-specific fine-tuned models as behavioral data accumulates. Be explicit about which subjects are in beta and which are validated. Do not oversell capability in domains where the student model is weak.

**The Knewton ghost.**

Knewton raised $157M, promising adaptive learning for everything. It failed because: (a) the algorithms were only as good as the content, (b) students don't generate enough data in a semester for sophisticated adaptation, (c) the promises outran the evidence.

This system makes similar promises and faces similar risks. The critical difference: this system uses population-level data to compensate for sparse individual data. A student who has only done 50 interactions contributes to population statistics while receiving benefits from 100,000 other students' interactions. Knewton's algorithms tried to adapt purely on individual data — they starved.

But the ghost remains. Don't promise what the data doesn't support. If the student model is uncertain about a concept, say so. "We have limited data on your mastery of this concept — you may want to verify with other sources."

**The measurement problem is real and unsolved.**

If the product claims to improve exam scores, it needs to measure exam scores. Measuring exam scores requires students to report back. Most won't. Exam scores without content of the exam are a noisy signal. IRB approval is required for controlled studies. Educational RCTs take semesters.

The short-term answer: use proxy metrics that correlate with exam performance (checkpoint accuracy, retention rate at Day-3 and Day-7, session completion rate). The medium-term answer: design the product to make grade reporting feel like a feature, not a survey. The long-term answer: institution partnerships that provide grade data directly with student consent.

Without outcome measurement, the product is science-backed but not science-validated. That's an honest distinction. Don't collapse it.

---

## The Technical Stack (Final)

| Component | Technology | Justification |
|---|---|---|
| PDF/PPTX parsing | MinerU (OSS) | Best layout-aware extraction benchmarked at CVPR 2025 |
| Slide visual analysis | Claude 3.5 Sonnet (primary) / Gemini 2.0 Flash (cost optimization) | VLM capability needed for diagram comprehension |
| Concept graph storage | PostgreSQL + pgvector + Apache AGE (graph extension) | Relational + semantic + graph in one system; avoids Neo4j ops complexity |
| Concept embeddings | text-embedding-3-large (API) or Nomic-Embed-Text-1.5 (self-hosted, 768d, MIT license) | Nomic is SOTA open-source; reduces embedding costs as scale grows |
| Knowledge tracing | Custom DKT-GAT (PyTorch + PyG) | Graph attention over prerequisite structure; pretrained on Assistments+EdNet |
| Spaced repetition | py-fsrs (Python, MIT license) | Open-source FSRS implementation; most accurate algorithm available |
| IRT calibration | irt-py or Girth (Python) | 2PL IRT for item difficulty/discrimination fitting |
| Scheduling engine | Pure Python service | Deterministic; must never hallucinate |
| Content generation | Claude API (Haiku for simple cards, Sonnet for complex) | Tier by complexity to manage costs |
| Content validation | Claude API (extended thinking) or Sonnet as verifier | Cheaper than a separate model; same context window |
| Explanation pool | PostgreSQL JSONB | Simple query; no graph traversal needed |
| Behavioral warehouse | PostgreSQL + TimescaleDB extension | Time-series optimization for response sequence data |
| Analytics pipeline | Apache Airflow + dbt | Industry standard for data pipeline orchestration |
| ML training pipeline | PyTorch + Weights & Biases | DKT-GAT training and experiment tracking |
| Frontend | Next.js 15 (App Router) | SSR for initial load perf; React for interactivity |
| API layer | FastAPI (Python) | Matches ML service language; async-native |
| Auth | Clerk | Don't build this |
| File storage | S3 or R2 | Student uploads |
| Queue | Redis + BullMQ | Async document processing |
| Search | Typesense (self-hosted) or Algolia | Concept search for students |

**Local/OSS model strategy (progressive):**

Phase 1: Everything through Claude API. Fast to build, known quality.
Phase 2: Migrate content generation to fine-tuned Llama 3.3 70B (QLoRA on A100s via RunPod). Population explanation pool fills gaps; LLM is called for novel content only. Cost reduction 10x.
Phase 3: Fine-tuned teaching model. Llama or Phi base. Dataset: (concept, student_state, card_type) → (card_content, behavioral_outcome). Trained on proprietary interaction data. This model cannot be reproduced. It is the deepest moat.

---

## Build Sequence (The Real One)

**Phase 0 — Before writing a line of code (2 weeks):**
Talk to 30 students. Watch them study. Don't ask what they want — watch what they do. Confirm: do they paste slides into ChatGPT? What's the failure mode they actually experience? Validate one assumption: that students will do anything differently if the predicted exam score is visible. If this assumption is wrong, the entire behavioral design breaks.

**Phase 1 — The Engine Core (10-14 weeks):**
Document pipeline (MinerU + VLM analysis + concept extraction + graph construction). Student model (FSRS + basic DKT, no GNN yet). Scheduler (frontier detection, successive relearning, no interference weighting yet). Content pipeline (card sequence generation + LLM filling + constraint validation + fact-checking). Web app: upload → lesson → checkpoint → review. Nothing else. No predicted score, no knowledge map, no analytics.

Ship to 50 students at UCLA. Measure: Day-3 return rate, checkpoint accuracy, session completion. Run one course (most important), compare against ChatGPT control group.

**Phase 2 — The Student Model (10-14 weeks):**
DKT-GAT integration. Response time features. Confidence calibration tracking. Predicted exam score V1. Metacognitive mirror (calibration display). Interference-aware scheduling. Explanation pool infrastructure. A/B test framework.

**Phase 3 — The Flywheel (6+ months of operation):**
Explanation tournament begins accumulating data. Universal misconception detection runs. Behavioral prerequisite refinement runs. IRB study designed, submitted, approved. Exam score reporting flow built. Cross-course transfer detection. Exam readiness model calibration begins.

**Phase 4 — The Moat (12+ months):**
Fine-tuned teaching model training. Institution partnerships. Canvas LTI integration. Published validation study. Series A story: evidence-based, proprietary data, demonstrable outcome improvement.

---

*This is not the plan for something that ships in a week. This is the plan for something that, in five years, is the default way college students study.*

*The window is real. ChatGPT made retrieval easy. The research has now conclusively shown that easy retrieval harms learning. The students who figure out that their AI tool is making them worse are the early adopters. There are more of them every month.*

*The technology exists. The research is done. The architecture is coherent. The only question is execution.*
