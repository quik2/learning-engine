# The Learning Engine — Architecture V3
*The version that's actually hard to build.*

---

## What V2 Got Wrong (Or Incomplete)

V2 was structurally correct. But it had three problems that this version fixes:

**1. It treated the scheduling engine as the end of the optimization story.** The scheduler in V2 is deterministic and rule-based — good rules, backed by research, but still handcrafted heuristics. The real system eventually needs to *learn* the optimal policy, not just follow rules derived from aggregate study effects. Deterministic is Phase 1. Learned policy is where it goes.

**2. It undersold the student model's architecture.** DKT-GAT was mentioned but the actual design — how the model handles cold start, how it represents concept-level versus skill-level knowledge, how it processes response time as a feature, how it fuses behavioral and semantic signals — was left vague. This document makes it precise.

**3. It treated the LLM content pipeline as a box.** The real architecture for how the LLM is constrained, validated, and progressively replaced needs to be specified with enough precision to actually build it. V2 described the idea. V3 describes the implementation.

---

## The Full System Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    INGEST LAYER                              │
│  [Upload] → [MinerU] → [VLM Slide Analysis] → [Content     │
│  Classification] → [Document Object]                        │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                  KNOWLEDGE GRAPH ENGINE                       │
│  [Concept Extraction] → [Prerequisite Detection] →           │
│  [Ontology Grounding] → [Embedding Index] →                  │
│  [Interference Map] → [DPKG]                                 │
└──────────────────────────┬──────────────────────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
┌───────▼───────┐  ┌───────▼───────┐  ┌──────▼──────────────┐
│  STUDENT      │  │  SCHEDULING   │  │  CONTENT PIPELINE   │
│  MODEL        │  │  ENGINE       │  │                      │
│               │  │               │  │  [Population Pool]   │
│  DKT-GAT      │  │  Phase 1:     │  │  → [Card Spec]       │
│  + FSRS       │  │  Deterministic│  │  → [LLM Generation]  │
│  + IRT        │  │               │  │  → [Validation]      │
│  + Confidence │  │  Phase 2:     │  │  → [Fact Check]      │
│  + Response   │  │  Contextual   │  │  → [Rendered Card]   │
│    Time       │  │  Bandit       │  │                      │
│               │  │               │  └──────────────────────┘
│               │  │  Phase 3:     │
│               │  │  Deep RL      │
└───────┬───────┘  └───────┬───────┘
        │                  │
┌───────▼──────────────────▼──────────────────────────────────┐
│                  CROSS-STUDENT FLYWHEEL                       │
│  [Interaction Warehouse] → [Explanation Tournament] →        │
│  [Misconception Detector] → [Graph Refinement] →             │
│  [Score Calibration] → [DKT Fine-Tune Pipeline]              │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                  EXAM READINESS MODEL                         │
│  [Mastery × Forgetting × Concept Weight] → [Score] →        │
│  [Calibration via Ground Truth] → [Action Panel]             │
└──────────────────────────┬──────────────────────────────────┘
                           │
┌──────────────────────────▼──────────────────────────────────┐
│                  METACOGNITIVE INTERFACE                      │
│  [Knowledge Map] → [Calibration Report] →                    │
│  [Forgetting Projection] → [Surprise Retrieval]              │
└─────────────────────────────────────────────────────────────┘
```

---

## Layer 1: The Ingest Layer — Precisely

### Why slide-as-text is catastrophically wrong

Consider a typical cell biology slide. It has: a header reading "Oxidative Phosphorylation," three bullet points with incomplete sentences, a diagram of the electron transport chain with embedded chemical notation, and a small footnote about an exam exception.

PyPDF2 extracts: "Oxidative Phosphorylation NADH → NAD+ ΔG° = –220 kJ/mol electrons passed via protein complexes." 

The causal relationship between the bullet points (the mechanism is that electrons pass through protein complexes, releasing energy used to pump protons, creating a gradient, which drives ATP synthase) is gone. The diagram — which is the actual lesson — is gone. The professor's visual emphasis — the thing circled in red that will be on the exam — is gone.

The lesson built from the text-only extraction teaches: "there are some things called protein complexes and the word NADH exists." That's not biochemistry. That's noise.

### The dual-path extraction pipeline

**Path A: Structural extraction (MinerU)**

MinerU performs layout-aware PDF analysis using a detection model for regions (text, formula, figure, table, header, footer), followed by specialized processors per region type:
- Text regions: OCR + structure preservation (heading hierarchy, list nesting)
- Formula regions: LaTeX extraction via a formula recognition model
- Figure regions: bounding box capture + figure caption association
- Table regions: cell extraction + header inference

Output per slide:
```json
{
  "slide_index": 4,
  "heading": "Oxidative Phosphorylation",
  "text_blocks": [
    {"type": "bullet_l1", "text": "NADH donates electrons to Complex I", "position": [120, 340]},
    {"type": "bullet_l2", "text": "Electrons flow down energy gradient", "position": [140, 380]},
    {"type": "bullet_l1", "text": "Proton gradient drives ATP synthase", "position": [120, 420]}
  ],
  "formulas": [
    {"latex": "\\Delta G^\\circ = -220\\, kJ/mol", "position": [400, 340]}
  ],
  "figures": [
    {"bbox": [100, 500, 700, 900], "caption_text": "Electron transport chain"}
  ],
  "footnote": "Focus on net ATP yield, not individual complex contributions"
}
```

**Path B: Visual-semantic analysis (VLM)**

Each slide rendered as 2048×1536px image. Prompt to VLM (Claude 3.5 Sonnet — chosen for diagram comprehension over Gemini 2 Flash after testing):

```
Analyze this educational slide as a pedagogical document. Return structured JSON with:
1. diagram_type: what kind of visual is present (pathway_diagram / graph / table / illustration / none)
2. concepts_in_diagram: list each distinct concept labeled or implied in the diagram
3. spatial_relationships: directed relationships between concepts visible in the diagram layout
4. emphasis_signals: what visual cues (bold, circles, arrows, color) indicate exam-important content
5. pedagogical_narrative: what argument or mechanism is this slide teaching, as a one-sentence summary
6. likely_misconceptions: what would a student plausibly misunderstand from this slide alone
7. prerequisite_implied: what concepts must the student already know for this slide to make sense
```

Output merged with Path A using positional alignment — figure captions from Path A are linked to concepts from Path B's diagram analysis.

**Path C: Content-type classification**

Per content block, classification model (fine-tuned DistilBERT or few-shot Claude) assigns:

| Type | Definition | Pedagogical treatment |
|---|---|---|
| `DEFINITION` | Formal concept definition | Pretest → Teach → Checkpoint |
| `MECHANISM` | Multi-step process | Worked example → Step-by-step retrieval |
| `FACT` | Discrete retrievable datum | Direct flash → FSRS |
| `PRINCIPLE` | Overarching rule | Varied application questions |
| `EXAMPLE` | Concrete instance of abstract rule | Transfer test: new instance |
| `EDGE_CASE` | Rule exception | Contrast question with main rule |
| `DIAGRAM_CONCEPT` | Concept defined by visual | Visual → Describe-from-memory |
| `EXAM_SIGNAL` | Instructor-flagged testable content | Highest review priority |

The `EXAM_SIGNAL` type is detected from: footnotes mentioning exams, bold/circled text in slides, explicit instructor language ("you need to know"), and behavioral signal after data accumulates (concepts with high checkpoint failure rate across students are retroactively upgraded to `EXAM_SIGNAL`).

**Quality scoring per slide (0.0–1.0):**
```python
def extraction_confidence(slide: SlideResult) -> float:
    text_coverage = len(slide.text_blocks) / expected_text_blocks(slide.heading)
    formula_completeness = all(f.latex for f in slide.formulas)
    diagram_coverage = slide.vlm_concepts_found / slide.vlm_concepts_expected  
    caption_alignment = slide.captions_matched / len(slide.figures)
    
    return weighted_mean([text_coverage, formula_completeness, 
                          diagram_coverage, caption_alignment],
                         weights=[0.3, 0.2, 0.35, 0.15])
```

Slides below 0.55 confidence are flagged in the UI. Not hidden — shown with a warning badge. The student sees: "Slide 12 was difficult to extract. The diagram content may be incomplete." This matters because content accuracy is an existential quality risk; transparency is better than silent failure.

---

## Layer 2: The Knowledge Graph — Precise Schema

### Why a graph and not a vector store

A vector store answers: "what is semantically similar to X?" The DPKG answers: "what must be known before X can be understood, and what does X enable?" These are different questions with different answers.

Semantic similarity and pedagogical dependency are not the same. "Glycolysis" and "gluconeogenesis" are semantically adjacent (high cosine similarity) but pedagogically ordered (glycolysis before gluconeogenesis in most curricula). A vector store cannot represent this ordering. A graph can.

### Full node schema

```python
@dataclass
class ConceptNode:
    # Identity
    id: UUID                              # Stable identifier
    canonical_name: str                   # "Glycolysis"
    aliases: List[str]                    # ["EMP pathway", "glycolytic pathway"]
    
    # Semantic representation  
    embedding: np.ndarray                 # 1536-dim (text-embedding-3-large)
    definition: str                       # Formal, one-sentence definition
    extended_description: str             # 2-3 sentence contextual description
    
    # Pedagogical metadata
    bloom_level: BloomLevel               # REMEMBER/UNDERSTAND/APPLY/ANALYZE/EVALUATE/CREATE
    content_type: ContentType             # DEFINITION/MECHANISM/PRINCIPLE/FACT/EDGE_CASE
    estimated_difficulty: float           # 0.0–1.0; initialized from VLM analysis, updated behaviorally
    estimated_teach_time_min: float       # How long initial teaching takes; behavioral update
    
    # Knowledge structure
    common_misconceptions: List[Misconception]  # Populated from population data
    known_confusables: List[str]          # Concept IDs that students confuse with this one
    prerequisite_ids: List[str]           # IDs of concepts that must be mastered first
    enables_ids: List[str]               # IDs of concepts this concept unlocks
    
    # Cross-document tracking
    cross_course_uid: Optional[str]       # Stable ID linking same concept across uploads
    source_documents: List[DocumentRef]   # Which uploads introduced this concept
    source_slide_indices: List[int]
    
    # Diagram reference
    has_visual: bool
    visual_description: Optional[str]     # VLM-generated description of associated diagram
    diagram_type: Optional[DiagramType]
    
    # Population statistics (updated nightly)
    mean_checkpoint_accuracy: float       # Across all students
    mean_sessions_to_criterion: float     # Mean successive relearning sessions needed
    misconception_rate: Dict[str, float]  # wrong_answer_id → P(chosen | wrong)
    population_coverage: int              # Number of students who have studied this concept
```

### Full edge schema

```python
@dataclass
class PrerequisiteEdge:
    source_id: UUID           # This concept must be mastered...
    target_id: UUID           # ...before this concept is teachable
    
    strength: EdgeStrength    # HARD (P(fail target | fail source) > 0.7)
                              # SOFT (P(fail target | fail source) 0.3–0.7)
                              # CONTEXTUAL (helps but not required)
    
    evidence: EdgeEvidence
    llm_confidence: float     # From pairwise prompting, 0.0–1.0
    ontology_validated: bool  # Cross-checked against domain ontology
    behavioral_p_fail: Optional[float]  # Empirical P(fail target | fail source)
    behavioral_n: int         # Sample size for behavioral estimate
    
    # Interference metadata
    semantic_distance: float  # Cosine distance between concept embeddings
    interference_risk: float  # Computed from semantic_distance + co-occurrence frequency
```

### Prerequisite detection — three-signal fusion in detail

**Signal 1: LLM pairwise prompting**

For every pair (A, B) in the concept set, prompt:
```
Concept A: {definition_A}
Concept B: {definition_B}

In a college course, would a student need to understand A to understand B?
Answer: HARD_PREREQUISITE (cannot understand B without A), SOFT_PREREQUISITE 
(A helps but isn't required), INDEPENDENT (no meaningful relation), or 
REVERSE (B is actually prerequisite to A).
Confidence: 0.0-1.0
Reason: one sentence.
```

Pairwise prompting is O(n²). For a 50-concept graph, that's 1,225 pairs — expensive but tractable. Above 100 concepts, use embedding distance to filter: only prompt pairs with cosine similarity > 0.4. This reduces calls to O(n × k) where k is the mean number of semantically related concepts.

**Signal 2: Domain ontology grounding**

Available ontologies per subject domain:
- Biology/Medicine: UMLS (Unified Medical Language System) — 3.6M concepts, 50+ semantic types
- Chemistry: ChemOnt — hierarchical chemical classification
- Mathematics: MathSciNet Mathematical Subject Classification (MSC2020)
- Computer Science: ACM Computing Classification System (CCS)
- Physics: Physics and Astronomy Classification System (PACS)
- General STEM: Khan Academy skill graph (scraped, structured) + OpenStax concept maps

Process: match extracted concept names against ontology via embedding nearest-neighbor search. When an ontology edge exists between two concepts, its direction supersedes LLM inference with confidence 0.9. When ontology and LLM disagree, flag for human review (very rare; mostly occurs on highly course-specific jargon).

**Signal 3: Behavioral refinement**

Nightly batch job. For all (A, B) edge pairs with behavioral_n ≥ 50:
```python
def compute_behavioral_strength(source: ConceptNode, target: ConceptNode,
                                 interaction_log: InteractionLog) -> float:
    # Students who have studied both concepts
    students_studied_both = interaction_log.students_with(source.id, target.id)
    
    # Among those who failed source: what fraction also failed target?
    failed_source = [s for s in students_studied_both 
                     if s.mastery(source.id) < MASTERY_THRESHOLD]
    failed_target_given_failed_source = sum(
        1 for s in failed_source if s.mastery(target.id) < MASTERY_THRESHOLD
    ) / len(failed_source) if failed_source else 0.0
    
    return failed_target_given_failed_source
```

Edge updates:
- `behavioral_p_fail` > 0.70 → upgrade to HARD
- `behavioral_p_fail` 0.30–0.70 → set/maintain SOFT  
- `behavioral_p_fail` < 0.30 → downgrade or remove edge
- Required n ≥ 50 for any update (confidence threshold)

After behavioral refinement, the graph's edges are empirically grounded. LLM inference had ~80% accuracy for hard prerequisites. Behavioral refinement pushes this above 95% for concepts with sufficient population coverage.

### Cross-document concept linking

When a student uploads a new document, every extracted concept is compared against their existing graph:

```python
def link_cross_document(new_concept: ConceptNode, 
                        existing_graph: DPKG,
                        threshold: float = 0.92) -> Optional[str]:
    # Find nearest neighbor in existing graph by embedding
    nn_concept, similarity = existing_graph.nearest_neighbor(new_concept.embedding)
    
    if similarity >= threshold:
        # High confidence: same concept, different document
        # Don't create new node; link to existing
        new_concept.cross_course_uid = nn_concept.id
        # Merge source documents, preserve existing mastery state
        return nn_concept.id
    elif similarity >= 0.75:
        # Medium confidence: possibly same concept; create soft link
        # System notes the relationship but maintains separate mastery tracking
        return None
    else:
        # New concept; create node
        return None
```

This means a student who uploads three lectures on cellular respiration doesn't re-learn ATP each time. It means a student who studied biochemistry and then takes cell biology has their existing biochemistry mastery transfer automatically to overlapping concepts.

---

## Layer 3: The Student Model — Full Architecture

### The three-component model stack

**Component A: DKT-GAT (Deep Knowledge Tracing with Graph Attention)**

The architecture:

```
Input sequence: [(c₁, r₁, t₁, conf₁), (c₂, r₂, t₂, conf₂), ... (cₙ, rₙ, tₙ, confₙ)]
  where: c = concept_id, r = correct (0/1), t = response_time_ms, conf = confidence (0/1)

Step 1: Concept embedding lookup → E[c] ∈ R^d

Step 2: Interaction embedding
  x_i = concat(E[c_i], r_i, log(t_i)/normalize, conf_i, bloom_level(c_i))

Step 3: Graph attention message passing
  For each concept c in graph G:
    h_c = GAT(E[c], {E[neighbor] for neighbor in prerequisites(c)},
               attention_weight=prerequisite_strength)
  
  This propagates: if student masters prerequisites of c, prior for c increases
                   if student fails c, posterior of concepts depending on c updates

Step 4: Transformer encoder over interaction sequence
  H = TransformerEncoder(x_1, x_2, ..., x_n)  # Captures temporal dependencies
  
Step 5: Mastery prediction head
  P(correct on concept c | H, h_c) = sigmoid(W · concat(H[-1], h_c) + b)
  
  Output: mastery probability for every concept in graph, given this student's history
```

**Why transformer instead of LSTM?**

DKT-LSTM (the original DKT) processes responses sequentially and has weak long-range dependency modeling — it essentially forgets interactions from many steps ago. For a student who studied a concept 3 weeks ago, the LSTM has poor recall of that session. A transformer's attention mechanism can directly attend to the relevant prior interaction regardless of its position in the sequence.

The graph attention component is the key upgrade over standard DKT: mastery probabilities propagate along prerequisite edges. This means the model is learning within a structured space, not just pattern-matching on sequences.

**Pretraining strategy:**

Phase 1: Pretrain on ASSISTments 2017 (400K students, 4M interactions) + EdNet-KT (780K students, 95M interactions). These are open datasets with (student, problem, correctness, timestamp) tuples. The concept IDs map to math/science skills.

Phase 2: Fine-tune on the system's own interaction data as it accumulates. After 10K student-sessions, the first fine-tune runs. Weekly thereafter. The model gets better as the product grows.

Cold start: For a student with 0 interactions, the model outputs P(mastered) = 0.1 for all concepts (low prior, not zero). After 3–5 diagnostic interactions, the model has enough signal to meaningfully differentiate mastery levels.

**Component B: FSRS (Free Spaced Repetition Scheduler)**

FSRS models two parameters per (student, concept) pair:
- `S` (Stability): expected time for retrievability to decay to 90%
- `D` (Difficulty): intrinsic difficulty for this student on this concept

Update rules (from the FSRS paper):
```
After correct recall:
  S_new = S_old × e^(FACTOR × (11 - D) × (S_old/DECAY))  # Stability grows
  D_new = D_old - 0.1 × (correct - difficulty_target)     # Difficulty adjusts

After failed recall:
  S_new = S_old × FORGET_FACTOR                           # Stability drops
  D_new = D_old + 0.1 × (difficulty_target - correct)

Next review interval: I = S × log(RETENTION_TARGET) / log(0.9)
  where RETENTION_TARGET = 0.90 (default, adjusts as exam approaches)
```

FSRS parameters are fit per student per concept from their response history. A student who retains biology concepts well (high S_init) gets longer intervals. A student who consistently struggles with a specific concept (high D) gets shorter intervals and more scaffolding.

**Component C: Item Response Theory (2PL-IRT)**

IRT operates at the question level, not the concept level. For each question q in the system's question pool:

```
P(correct | student ability θ, item parameters a, b) = 
  1 / (1 + e^(-a(θ - b)))

where:
  θ = student latent ability on this concept (estimated from DKT)
  a = discrimination parameter (how well this question separates high/low ability)
  b = difficulty parameter (θ at which P(correct) = 0.50)
```

Parameters `a` and `b` are fit from population response data using maximum likelihood estimation once a question has ≥ 100 responses. Before that, the question uses population mean values.

IRT is used by the scheduler to select the most informative question to show next. The maximum information criterion selects the question with maximum:

```
I(θ) = a² × P(correct|θ) × (1 - P(correct|θ))
```

This is highest when the student is near the difficulty threshold of the question — when the outcome is most uncertain. Showing easy questions to a struggling student provides minimal information. Showing hard questions to a strong student provides minimal information. The most informative question is the one where the student has roughly 50% chance of success — precisely challenging without overwhelming.

### Response time as a second signal

Response time is available from the client (timestamps on question presentation and answer submission). It is almost universally ignored in educational software.

Response time carries real signal about processing fluency:

```python
def classify_response_type(response_time_ms: int, 
                            student_baseline_ms: float,
                            concept_baseline_ms: float) -> ResponseType:
    # Normalize by student baseline and concept baseline
    normalized_rt = response_time_ms / (student_baseline_ms * concept_baseline_ms)
    
    if normalized_rt < 0.4:
        return ResponseType.RAPID  # Too fast; likely guess or prior knowledge
    elif normalized_rt > 2.5:
        return ResponseType.SLOW   # Deliberate processing; effortful
    else:
        return ResponseType.NORMAL

def adjust_mastery_for_rt(base_p_mastered: float, 
                           response_type: ResponseType,
                           correct: bool) -> float:
    if correct and response_type == ResponseType.RAPID:
        # Fast correct: likely automatic retrieval; genuine mastery
        return min(1.0, base_p_mastered + 0.05)
    elif correct and response_type == ResponseType.SLOW:
        # Slow correct: worked it out; mastery not fluent; reschedule sooner
        return max(0.0, base_p_mastered - 0.05)
    elif not correct and response_type == ResponseType.RAPID:
        # Fast wrong: guessing; high slip probability; adjust DKT slip param
        return base_p_mastered  # Discount the wrong answer; likely noise
    elif not correct and response_type == ResponseType.SLOW:
        # Slow wrong: genuinely trying and failing; strong not-mastered signal
        return max(0.0, base_p_mastered - 0.10)
    return base_p_mastered
```

Response time baseline calibrates per student over the first 20–30 interactions. Early interactions use concept population baselines.

### The confidence calibration model

Every checkpoint question receives a confidence rating before the answer: **Sure / Unsure**.

This binary signal is rich. It allows construction of a personal calibration curve:

```
calibration_score(student) = |P(correct | said "Sure") - 0.9| + 
                              |P(correct | said "Unsure") - 0.5|

Perfectly calibrated student: says "Sure" and gets 90% correct; 
                               says "Unsure" and gets 50% correct
Overconfident student: says "Sure" frequently, gets 40-60% correct
Underconfident student: says "Unsure" frequently, gets 70-80% correct
```

The calibration score is computed from rolling 30-question windows. It is:

1. Shown to the student as part of the metacognitive interface ("When you said you were confident last month, you were right 58% of the time")
2. Used to adjust DKT mastery estimates: when an overconfident student says "Sure" and gets it right, the mastery update is dampened (their "Sure" is less reliable)
3. Used to trigger hypercorrection scheduling: when a student says "Sure" and gets it wrong, that concept is flagged for review within 48–72 hours

**The metacognitive blindness detector:**

```python
def detect_metacognitive_blindness(student_id: str, 
                                    concept_id: str) -> Optional[BlindnessAlert]:
    mastery_p = student_model.p_mastered(student_id, concept_id)
    calibrated_confidence = student_model.recent_confidence(student_id, concept_id)
    
    # Blindness: student frequently confident, but mastery is low
    if calibrated_confidence > 0.75 and mastery_p < 0.45:
        return BlindnessAlert(
            concept_id=concept_id,
            severity=BlindnessSeverity.HIGH,
            confidence=calibrated_confidence,
            actual_mastery=mastery_p,
            gap=calibrated_confidence - mastery_p
        )
    return None
```

High-severity blindness alerts are surfaced: in the metacognitive interface, in the exam readiness report as "⚠️ Confidence trap: You feel prepared on these concepts but our model predicts you will struggle," and as scheduling priority upgrades (the concept gets more aggressive review even if it's nominally in the "graduated" category).

---

## Layer 4: The Scheduling Engine — Three Phases

### Phase 1: Deterministic (Launch)

The scheduler in Phase 1 is entirely rules-based. No learned policy. This is intentional: you need behavioral data to learn a policy, and you don't have it at launch.

**The full decision procedure, every session:**

```python
def generate_session_plan(student: Student, 
                           session_duration_min: float) -> SessionPlan:
    
    # Step 1: Collect due reviews
    due_reviews = [c for c in student.graduated_concepts 
                   if fsrs.is_due(student.id, c.id, today())]
    
    # Step 2: Prioritize reviews by urgency
    # Urgency = (current_retrievability - retention_target) × exam_weight × blindness_flag
    review_queue = sorted(due_reviews, key=lambda c: urgency_score(student, c), reverse=True)
    
    # Step 3: Interference pruning
    # Remove concepts that were reviewed in the last 24h and have a 
    # semantically adjacent concept in the queue
    review_queue = prune_interference(review_queue, 
                                       student.recent_review_history,
                                       interference_map=graph.interference_map,
                                       min_distance_hours=24)
    
    # Step 4: Determine if we have budget for new teaching
    review_time = estimate_time(review_queue[:max_reviews])
    remaining_time = session_duration_min - review_time
    
    # Step 5: Find teachable concepts (prerequisites mastered, not yet taught)
    frontier = graph.frontier(student.mastery_map, threshold=MASTERY_THRESHOLD)
    
    # Step 6: Rank frontier by:
    # (exam_weight × dependency_count × inverse_time_to_deadline) / estimated_teach_time
    frontier_ranked = sorted(frontier, key=lambda c: teachability_score(c, student), 
                             reverse=True)
    
    # Step 7: Select concepts fitting time budget
    new_concepts = select_within_budget(frontier_ranked, remaining_time)
    
    # Step 8: Interleave reviews and new concepts
    # Rule: Never review two semantically adjacent concepts back-to-back
    # Rule: Alternate teach/review when both are present
    session = interleave(review_queue, new_concepts, 
                         interference_map=graph.interference_map)
    
    # Step 9: Successive relearning enforcement
    # Concept not graduated until 3 correct recalls across sessions
    # Correct-streak counter per concept is tracked in student state
    for concept in session:
        concept.graduation_check = student.correct_streak(concept.id) >= 3
    
    return SessionPlan(concepts=session, estimated_duration_min=session_duration_min)
```

**Deadline compression algorithm:**

As exam date approaches, the scheduler solves a portfolio optimization problem:

```python
def optimize_session_allocation(concepts: List[Concept], 
                                 time_budget_min: float,
                                 days_until_exam: int) -> List[Concept]:
    # For each concept: expected score contribution if studied vs. not
    contributions = {}
    for c in concepts:
        p_correct_now = student.p_mastered(c.id)
        p_correct_at_exam = fsrs.predict_at_time(student.id, c.id, days_until_exam)
        p_correct_if_studied = fsrs.predict_post_study(student.id, c.id, days_until_exam)
        
        marginal_gain = (p_correct_if_studied - p_correct_at_exam) * c.exam_weight
        time_cost = c.estimated_review_time_min
        
        contributions[c.id] = marginal_gain / time_cost  # Gain per minute
    
    # Greedy selection by gain-per-minute (approximates optimal portfolio)
    sorted_by_efficiency = sorted(concepts, key=lambda c: contributions[c.id], 
                                   reverse=True)
    
    selected, remaining_time = [], time_budget_min
    for c in sorted_by_efficiency:
        if c.estimated_review_time_min <= remaining_time:
            selected.append(c)
            remaining_time -= c.estimated_review_time_min
    
    return selected
```

The implicit behavior: when time is very short and some concepts are too far from mastery to reach by exam day, they are deprioritized. The student is told: "With 2 days left, we're focusing on these 8 concepts where studying now moves your predicted score the most. Leaving out 3 concepts you haven't studied — there isn't enough time to learn them well." This is a hard decision that human tutors make constantly and apps never do.

### Phase 2: Contextual Bandit (After 50K sessions)

When enough behavioral data exists, the scheduler introduces learned policies for one specific decision: **explanation selection**.

Every concept has multiple explanation variants in the pool. The question is: given this student's context (mastery history, learning style signals, confidence profile, time until exam), which explanation variant produces the highest checkpoint accuracy AND 7-day retention?

This is a contextual bandit problem:
- **Arms**: explanation variants for a concept
- **Context**: student feature vector (mastery profile, response time patterns, confidence calibration score, exam proximity, concept difficulty, Bloom target)
- **Reward**: checkpoint_accuracy × 0.4 + day7_retention × 0.6 (delayed reward)
- **Algorithm**: Thompson Sampling (demonstrated superior to UCB and ε-greedy in educational recommendation by INFORMS 2025)

```python
class ExplanationBandit:
    def __init__(self, concept_id: str, n_variants: int):
        # Beta distribution parameters per variant
        # α = successes + 1, β = failures + 1 (Beta(α, β))
        self.alpha = np.ones(n_variants)
        self.beta = np.ones(n_variants)
    
    def select(self, student_context: np.ndarray) -> int:
        # Thompson Sampling: sample from each arm's posterior
        samples = np.random.beta(self.alpha, self.beta)
        # Context-adjusted: scale samples by predicted student affinity
        context_weights = self.context_model.predict(student_context)
        return np.argmax(samples * context_weights)
    
    def update(self, variant_idx: int, reward: float):
        # Update posterior based on observed reward
        self.alpha[variant_idx] += reward
        self.beta[variant_idx] += (1 - reward)
```

The delayed reward problem: checkpoint accuracy is available immediately; 7-day retention is available only after 7 days. Solution: use a reward shaping model that predicts 7-day retention from immediate checkpoint signals. Trained from historical data where both are available.

### Phase 3: Deep RL Policy (After Series A)

The long-term trajectory: the scheduler becomes a learned policy trained to maximize expected exam score, not just to follow handcrafted rules.

Formulation as MDP:
- **State**: student knowledge state S_t = (mastery_vector, confidence_calibration, days_until_exam, session_history)
- **Action**: what to teach next (concept_id × card_type × scaffold_level)
- **Reward**: dense proxy rewards (checkpoint accuracy, session completion, confidence calibration improvement) + sparse terminal reward (actual exam score, when available)
- **Transition**: P(S_t+1 | S_t, a_t) is the student model (DKT-GAT)

The credit assignment challenge (exam score as terminal reward with 8-day horizon) is addressed using hindsight credit assignment — a technique from recent RL research that retroactively assigns credit to earlier actions when a sparse terminal reward is finally observed.

This is Phase 3 because: (1) it requires massive behavioral data to train reliably, (2) it requires calibrated exam score ground truth for the terminal reward, and (3) it requires regulatory/IRB consideration for using RL to directly optimize student outcomes. It is architecturally correct. It is not a Phase 1 move.

---

## Layer 5: The Content Generation Pipeline — Implementation

### The card specification system

The scheduler emits a `CardSpec`. The content pipeline's job is to fill it. The LLM cannot modify the spec. It can only satisfy it.

```python
@dataclass
class CardSpec:
    card_type: CardType             # FORWARD_TEST, PRETEST, TEACH, CHECKPOINT, TRAP, ELABORATE
    concept: ConceptNode
    student_context: StudentContext # Mastery, prior errors, confidence profile
    scaffold_level: float           # 0.0 (minimal) to 1.0 (maximal)
    bloom_target: BloomLevel
    test_format: TestFormat
    time_budget_sec: int
    
    # Constraint set — LLM must satisfy all
    must_not_reveal_answer: bool     # For FORWARD_TEST, PRETEST
    must_activate_prior: bool        # For FORWARD_TEST
    max_words: int
    requires_confidence_rating: bool
    distractor_pool: Optional[List[str]]  # For MC: use these wrong answers
    population_explanation: Optional[str] # If pool has winner, inject this
    
    # Validation criteria (checked after generation)
    answer_not_in_question: bool     # For pretests: regex check
    max_concepts_per_card: int = 1   # Coherence constraint
    seductive_details_prohibited: bool = True
```

### The generation loop

```python
def generate_card(spec: CardSpec) -> RenderedCard:
    # Step 1: Check population explanation pool
    if spec.card_type == CardType.TEACH and spec.population_explanation:
        # Use the empirically winning explanation; skip LLM
        return RenderedCard(content=spec.population_explanation, 
                           source=ContentSource.POPULATION_POOL)
    
    # Step 2: Build generation prompt
    prompt = build_prompt(spec)
    
    # Step 3: Generate (with cost-tiering)
    model = select_model(spec.card_type, spec.concept.difficulty)
    # Simple cards (FACT review, FORWARD_TEST): claude-haiku-3-5
    # Complex cards (MECHANISM TEACH, ELABORATE): claude-sonnet-4-6
    
    raw_output = llm.generate(prompt, model=model, 
                               response_format=CardOutput.schema())
    
    # Step 4: Structural validation
    validation_result = validate_structure(raw_output, spec)
    if not validation_result.passed:
        # Regenerate with violation feedback
        raw_output = llm.generate(
            prompt + f"\nPrevious output failed: {validation_result.failures}",
            model=model
        )
    
    # Step 5: Pedagogical constraint validation
    if spec.must_not_reveal_answer:
        reveal_check = llm.generate(
            f"Does this question hint at or reveal its own answer? Question: {raw_output.question}",
            response_format=BoolResponse,
            model="claude-haiku-3-5"  # Cheap; simple task
        )
        if reveal_check.answer_revealed:
            raw_output = regenerate_with_constraint(spec, "question must not hint at answer")
    
    # Step 6: Fact-checking
    fact_check = verify_against_source(raw_output, spec.concept.source_text)
    if fact_check.ungrounded_claims:
        raw_output = revise_removing_claims(raw_output, fact_check.ungrounded_claims)
    
    # Step 7: Seductive detail removal
    raw_output = strip_seductive_details(raw_output, spec.concept)
    
    return RenderedCard(content=raw_output, source=ContentSource.LLM_GENERATED,
                        generation_cost_usd=compute_cost(model, raw_output))
```

### Constraint enforcement: what "seductive detail removal" means technically

Seductive details (Mayer, 2005) are interesting but pedagogically irrelevant additions. They improve engagement ratings while harming learning outcomes. An LLM trained on engaging text is biased toward including them.

Detection: after generation, identify sentences that contain concepts not in `spec.concept.related_vocabulary` and not in the prerequisite chain. These are candidates for removal. Filter using concept embedding similarity: a sentence is seductive if it introduces concepts with cosine similarity < 0.6 to the target concept.

This is not perfect but is directionally correct. The goal is not to eliminate all tangential content but to prevent the LLM from padding an explanation of glycolysis with an interesting anecdote about Pasteur.

### The cost architecture

At scale, content generation cost is a major variable:

| Card Type | Frequency | Model | Cost/1K cards |
|---|---|---|---|
| FORWARD_TEST | High | Haiku 3.5 | ~$0.10 |
| PRETEST | High | Haiku 3.5 | ~$0.15 |
| TEACH (population pool hit) | High (grows over time) | None | $0.00 |
| TEACH (new concept) | Medium | Sonnet 4.6 | ~$3.00 |
| CHECKPOINT | High | Haiku 3.5 | ~$0.20 |
| ELABORATE | Low | Sonnet 4.6 | ~$2.00 |
| FACT_CHECK | Per-card | Haiku 3.5 (fast) | ~$0.05 |

At 10K daily active students with 5 sessions/day and 8 cards/session:
- 400K cards/day
- Before population pool: ~$80-120/day in API costs
- After 6 months of pool accumulation (est. 60% hit rate): ~$35-50/day
- After 12 months (est. 80% hit rate): ~$20-30/day

The population pool is both a quality improvement and a cost reduction. After sufficient data, the most common concepts are served at zero LLM cost — we already know the best explanation.

---

## Layer 6: The Cross-Student Flywheel — In Detail

### The interaction warehouse schema

```sql
CREATE TABLE interactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    student_id UUID NOT NULL,  -- anonymized
    concept_id UUID NOT NULL,
    session_id UUID NOT NULL,
    card_type VARCHAR(30) NOT NULL,
    explanation_variant_id UUID,
    
    -- Response data
    student_answer TEXT,
    correct BOOLEAN NOT NULL,
    confidence VARCHAR(10),  -- 'sure' | 'unsure' | null
    response_time_ms INTEGER,
    
    -- Session context
    session_position INTEGER,  -- nth card in session
    days_since_upload FLOAT,
    days_since_last_review FLOAT,
    current_mastery_estimate FLOAT,
    preceding_concept_ids UUID[],  -- last 5 concepts reviewed
    
    -- Exam context
    days_until_exam INTEGER,
    exam_format VARCHAR(30),
    
    -- Delayed outcomes (updated later)
    day3_retrieval BOOLEAN,     -- NULL until Day+3 review
    day7_retrieval BOOLEAN,     -- NULL until Day+7 review
    actual_exam_score FLOAT,    -- NULL until student reports back
    
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Timescale hypertable for time-series performance
SELECT create_hypertable('interactions', 'created_at');

-- Indexes for core queries
CREATE INDEX idx_concept_variant ON interactions (concept_id, explanation_variant_id);
CREATE INDEX idx_student_concept ON interactions (student_id, concept_id);
CREATE INDEX idx_correct_concept ON interactions (concept_id, correct, created_at);
```

### The explanation tournament — full algorithm

```python
class ExplanationTournament:
    def run_nightly(self, concept_id: UUID):
        # Get all variants with ≥ 50 exposures
        variants = ExplanationVariant.query.filter(
            concept_id=concept_id, 
            exposure_count >= 50
        ).all()
        
        if len(variants) < 2:
            return  # Not enough data for tournament
        
        for variant in variants:
            # Primary metric: 7-day retention (delayed outcome)
            day7_exposures = Interaction.query.filter(
                concept_id=concept_id,
                explanation_variant_id=variant.id,
                day7_retrieval.isnot(None)
            ).all()
            
            if len(day7_exposures) < 30:
                # Use proxy metric: checkpoint accuracy + day3 retention
                variant.performance_score = weighted_mean([
                    variant.checkpoint_accuracy,
                    variant.day3_retention
                ], weights=[0.4, 0.6])
            else:
                # Sufficient ground truth for direct measurement
                variant.performance_score = mean([e.day7_retrieval for e in day7_exposures])
            
            # Compute confidence interval
            variant.confidence_interval = wilson_ci(
                successes=sum(e.day7_retrieval for e in day7_exposures),
                n=len(day7_exposures)
            )
        
        # Winner: highest lower confidence bound (conservative)
        winner = max(variants, key=lambda v: v.confidence_interval.lower)
        
        # Retire losers below threshold (lower bound < 0.55)
        for variant in variants:
            if variant.id != winner.id and variant.confidence_interval.upper < 0.55:
                variant.status = VariantStatus.RETIRED
        
        # Promote winner to population pool if lower bound > 0.65
        if winner.confidence_interval.lower > 0.65:
            PopulationPool.upsert(concept_id=concept_id, 
                                   explanation=winner.content,
                                   performance_score=winner.performance_score)
```

### Universal misconception detection

```python
def detect_misconceptions(concept_id: UUID, min_sample: int = 100):
    # Get all wrong answers for this concept
    wrong_interactions = Interaction.query.filter(
        concept_id=concept_id,
        correct=False,
        card_type=CardType.CHECKPOINT
    ).all()
    
    if len(wrong_interactions) < min_sample:
        return
    
    # Cluster wrong answers by embedding similarity
    wrong_answers = [i.student_answer for i in wrong_interactions]
    embeddings = embed(wrong_answers)  # Batch embedding call
    
    clusters = cluster_embeddings(embeddings, method='DBSCAN', 
                                   eps=0.15, min_samples=5)
    
    for cluster_id, cluster_members in clusters.items():
        if cluster_id == -1:  # Noise
            continue
        
        cluster_rate = len(cluster_members) / len(wrong_interactions)
        
        if cluster_rate > 0.25:  # > 25% of wrong answers cluster here
            # Universal misconception detected
            representative_answer = cluster_members[most_central_index]
            
            Misconception.create(
                concept_id=concept_id,
                misconception_text=representative_answer,
                frequency=cluster_rate,
                detection_date=today(),
                suggested_correction_card=True  # Flag for content team review
            )
            
            # Immediately add to distractor pool for future MC questions
            DistractorPool.add(
                concept_id=concept_id,
                distractor=representative_answer,
                is_misconception=True,
                frequency_weight=cluster_rate
            )
```

After misconception detection: a dedicated correction card is queued for the next session for any student who hasn't yet encountered the correction. The card format: show the concept, present the common wrong answer, ask the student to explain why it's wrong before revealing the explanation. This activates the hypercorrection effect specifically for the identified misconception.

---

## Layer 7: The Exam Readiness Model — Full Implementation

### The live score model

```python
class ExamReadinessModel:
    def compute_score(self, student_id: UUID, exam_date: date) -> ReadinessScore:
        days_remaining = (exam_date - today()).days
        
        concepts = student.graph.all_concepts()
        
        total_weighted_score = 0.0
        total_weight = 0.0
        concept_contributions = []
        
        for concept in concepts:
            # Current mastery probability
            p_mastered_now = student_model.p_mastered(student_id, concept.id)
            
            # Decay to exam day
            stability = fsrs.get_stability(student_id, concept.id)
            p_mastered_at_exam = p_mastered_now * exp(-days_remaining / stability)
            
            # Concept exam weight
            weight = self.compute_concept_weight(concept, student.exam_format)
            
            # Contribution to predicted score
            contribution = weight * p_mastered_at_exam
            
            total_weighted_score += contribution
            total_weight += weight
            
            concept_contributions.append(ConceptContribution(
                concept=concept,
                current_mastery=p_mastered_now,
                predicted_at_exam=p_mastered_at_exam,
                weight=weight,
                contribution=contribution,
                marginal_gain=self.marginal_gain_if_studied(concept, student_id, days_remaining)
            ))
        
        predicted_score = (total_weighted_score / total_weight) if total_weight > 0 else 0.0
        
        # Sort by marginal gain for action panel
        actionable_concepts = sorted(concept_contributions, 
                                      key=lambda c: c.marginal_gain,
                                      reverse=True)[:3]
        
        return ReadinessScore(
            predicted_percentage=predicted_score * 100,
            predicted_grade=self.pct_to_grade(predicted_score),
            trending=self.compute_trend(student_id, lookback_days=7),
            top_leverage_concepts=actionable_concepts,
            days_remaining=days_remaining
        )
```

### Concept weight computation

V1 weights (at launch, before ground truth data):
```python
def compute_concept_weight_v1(concept: ConceptNode, exam_format: TestFormat) -> float:
    # Slide emphasis weight: proxy for professor importance
    slide_emphasis = concept.source_slide_count / mean_slide_count_per_concept
    
    # Bloom level weight: higher-order thinking = harder to master = higher stakes
    bloom_weight = {
        BloomLevel.REMEMBER: 0.6,
        BloomLevel.UNDERSTAND: 0.8,
        BloomLevel.APPLY: 1.0,
        BloomLevel.ANALYZE: 1.2,
        BloomLevel.EVALUATE: 1.3,
        BloomLevel.CREATE: 1.4
    }[concept.bloom_level]
    
    # Exam format modifier: MC rewards recognition; short answer rewards recall
    format_modifier = {
        TestFormat.MULTIPLE_CHOICE: 0.9,      # MC has guessing floor
        TestFormat.FREE_RECALL: 1.2,           # Free recall requires retrieval
        TestFormat.SHORT_ANSWER: 1.1,          # Similar to free recall
        TestFormat.APPLICATION: 1.3,           # Transfer required
        TestFormat.OPEN_NOTE: 0.7             # Less memory pressure
    }[exam_format]
    
    # Dependency count: concepts that unlock more are foundational
    dependency_weight = 1.0 + 0.1 * len(concept.enables_ids)
    
    return slide_emphasis * bloom_weight * format_modifier * dependency_weight
```

V2 weights (after ground truth accumulation):

When students report back exam scores and identify which concepts appeared on their exam, the weight model is recalibrated using maximum likelihood estimation. The weights that minimize prediction error on the ground truth set become the new weights. After 5,000 ground truth observations in a subject domain, V2 weights outperform V1 by 15–25% in prediction accuracy.

---

## Layer 8: The Deliberate Practice Loop

This layer was absent from V2. It is the bridge between the learning engine and Ericsson's deliberate practice framework.

Ericsson (2008): expert performance is produced by deliberate practice — purposeful training focused on specific weaknesses, with immediate feedback, operating at the edge of current ability. Three criteria: (1) tasks just beyond current skill level, (2) immediate feedback, (3) focused repetition of the specific weakness.

The system currently satisfies (2) and (3) through the checkpoint + FSRS loop. (1) is the gap: the current scheduler doesn't explicitly ensure tasks are "just beyond current skill level."

**The zone of proximal development (ZPD) targeting:**

Vygotsky's ZPD: the range between what a student can do independently and what they can do with guidance. Optimal learning occurs when tasks are in the ZPD — not too easy (boring, no growth), not too hard (overwhelming, discouraging).

```python
def is_in_zpd(student_id: UUID, concept: ConceptNode) -> bool:
    mastery = student_model.p_mastered(student_id, concept.id)
    prerequisites_mastery = mean([
        student_model.p_mastered(student_id, prereq_id)
        for prereq_id in concept.prerequisite_ids
    ]) if concept.prerequisite_ids else 1.0
    
    # ZPD condition:
    # - Prerequisites are well-mastered (can access prior knowledge)
    # - Concept itself is not yet mastered (there's room to grow)
    in_zpd = (prerequisites_mastery > 0.75) and (mastery < 0.75)
    
    # Difficulty targeting: aim for 70-85% success rate during practice
    # Too easy (>85% success): boring, minimal growth
    # Too hard (<50% success): overwhelming, motivation damage
    predicted_success = student_model.predict_success(student_id, concept.id, 
                                                       scaffold_level=current_scaffold(student_id, concept.id))
    in_difficulty_range = 0.55 <= predicted_success <= 0.85
    
    return in_zpd and in_difficulty_range
```

The scheduler uses ZPD targeting to select among frontier concepts. When multiple concepts are in the frontier (prerequisites mastered, not yet taught), it prefers those where the student's predicted success rate is in the 55–85% range. Too easy means the student is probably already familiar from other sources. Too hard means the lesson will fail and demoralize.

The scaffold level is adjusted to place the student in the ZPD:

```python
def find_scaffold_for_zpd(student_id: UUID, concept: ConceptNode) -> float:
    # Binary search over scaffold_level to find the setting that places 
    # predicted_success in [0.60, 0.80]
    low, high = 0.0, 1.0
    for _ in range(10):  # Convergence in ~10 iterations
        mid = (low + high) / 2
        predicted = student_model.predict_success(student_id, concept.id, mid)
        if predicted < 0.60:
            high = mid  # More scaffolding needed
        elif predicted > 0.80:
            low = mid   # Less scaffolding needed (student is stronger)
        else:
            return mid
    return mid
```

This is the implementation of expertise reversal at the decision level: every student on every concept gets exactly the scaffold level that targets deliberate practice, automatically.

---

## The Model Fine-Tuning Pipeline

### How the DKT-GAT gets better over time

```
Week 0-4: Pretrained on ASSISTments+EdNet. No fine-tuning. Cold start.

Week 5+: 
  1. Every Sunday at 2 AM: export interaction logs from past week
  2. Filter: concepts with ≥ 20 student interactions
  3. Convert to DKT training format: (student_id, [(concept_id, correct, timestamp)])
  4. Fine-tune DKT-GAT on proprietary data using AdamW, lr=1e-4, batch=64
  5. Evaluate on held-out 20% of student sessions (AUC-ROC on correctness prediction)
  6. If AUC > pretrained baseline: deploy new weights
  7. Log fine-tune metadata to W&B
```

### How the fine-tuned teaching model gets built (Phase 4)

The long-term technical ambition is a small model (7B params) that is trained specifically to generate pedagogically effective educational content — not to be a helpful general-purpose assistant, but to execute the card specification system with native understanding of the pedagogical constraints.

Training data: every (CardSpec → RenderedCard → InteractionOutcome) triple the system accumulates. The target: given a spec, generate content that maximizes checkpoint_accuracy × day7_retention. This is a conditional generation task where the condition is the full pedagogical context.

Training approach: 
1. Start from a general instruction-tuned base (Llama 3.3 70B Instruct or Phi-4)
2. SFT on high-quality (spec, card) pairs where outcome was positive (checkpoint_accuracy > 0.7, day7_retention > 0.65)
3. RLAIF: use Claude Opus as the reward model, scoring generated cards on pedagogical quality dimensions (retrieval-forcing, no seductive details, appropriate scaffold level)
4. DPO: use (positive_card, negative_card) pairs from the explanation tournament data

This model cannot be replicated by any competitor. Its training set is the accumulated interaction history of every student who has used the system. It knows, at inference time, what works — not because it was pretrained on general text, but because it was fine-tuned on outcome-validated educational content.

---

## Infrastructure and Operational Architecture

### Service decomposition

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Frontend        │    │  API Gateway    │    │  Auth Service   │
│  Next.js 15     │◄───►  FastAPI         │◄───►  Clerk         │
│  App Router     │    │                 │    │                 │
└─────────────────┘    └────────┬────────┘    └─────────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
┌─────────▼─────┐    ┌──────────▼──────┐    ┌────────▼────────┐
│  Document     │    │  Session        │    │  Interaction    │
│  Processor   │    │  Manager        │    │  Logger         │
│  Service     │    │  Service        │    │  Service        │
│  (Python)    │    │  (Python/Node)  │    │  (FastAPI)      │
│              │    │                 │    │                 │
│  MinerU      │    │  Scheduling     │    │  TimescaleDB    │
│  VLM API     │    │  Engine         │    │  writer         │
│  Concept     │    │  Content Gen    │    │                 │
│  Extraction  │    │  LLM API        │    └────────┬────────┘
└──────┬───────┘    └────────┬────────┘             │
       │                     │                      │
┌──────▼─────────────────────▼──────────────────────▼────────┐
│                        Data Layer                            │
│  PostgreSQL (main) + TimescaleDB ext + pgvector ext          │
│  Qdrant (vector search for concept nearest-neighbor)         │
│  Redis (session state, rate limiting, job queue)             │
│  S3/R2 (document storage)                                    │
└──────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────▼──────────────────────────────┐
│                     ML Pipeline                              │
│  Airflow DAGs:                                               │
│    - Nightly: explanation_tournament.py                      │
│    - Nightly: misconception_detection.py                     │
│    - Nightly: graph_behavioral_refinement.py                 │
│    - Weekly: dkt_gat_finetune.py                             │
│    - Weekly: irt_calibration.py                              │
│    - Monthly: score_model_recalibration.py                   │
│  W&B (experiment tracking)                                   │
│  GPU instances: RunPod A100 (on-demand for fine-tune only)   │
└────────────────────────────────────────────────────────────┘
```

### Qdrant for concept vector search

V2 suggested PostgreSQL + pgvector. This is correct for the relational data and moderate-scale vector operations. However, for the cross-document concept linking (nearest-neighbor search across the global concept pool, which grows to millions of concepts across all users' uploads), pgvector becomes a bottleneck.

Qdrant (Rust-based, open-source, ANN search) handles:
- Nearest-neighbor search for concept linking at ingestion time
- Interference map computation (find all concepts within embedding distance threshold)
- Cross-course transfer detection

Qdrant runs as a separate service alongside PostgreSQL. The concept graph structure (nodes, edges, mastery probabilities) stays in PostgreSQL. Concept vectors and nearest-neighbor search move to Qdrant.

### Asynchronous document processing

Document ingestion is expensive (MinerU + VLM API + concept extraction + graph construction). It cannot block the user's session.

```
Upload → S3 storage → BullMQ job → Document Processor Service
  → MinerU (parallel per-page)
  → VLM analysis (parallel per-slide, rate-limited to 50 concurrent)
  → Concept extraction (batched LLM calls)
  → Graph construction
  → Prerequisite detection (batched LLM + ontology lookup)
  → Embedding batch (Qdrant upsert)
  → Student session unlocked
  → Notification: "Your notes are ready. 47 concepts found."
```

Estimated latency for a 30-slide lecture PDF: 45–90 seconds end-to-end. Acceptable for an async flow. The UI shows progress per stage.

---

## The Engineering Principle That Governs Everything

The system has two modes of decision-making:

**Deterministic decisions** (no LLM, no ambiguity): What to teach. When to teach it. In what sequence. At what scaffold level. Whether the student has met criterion. These decisions use the scheduling engine, the student model, and the FSRS parameters. They are verifiable, debuggable, and explainable.

**Generative decisions** (LLM involved): How to say what has been determined to be taught. What words to use. Which analogy to select. How to phrase the question. These decisions are constrained by the card spec system.

The discipline of the system is that every deterministic decision remains deterministic. The moment the scheduler becomes "ask the LLM what to teach next," the pedagogical integrity of the system is compromised. The LLM wants to be helpful. Helpful means answering questions, explaining things, providing comfort. The scheduling engine needs to force retrieval, enforce successive relearning criterion, target the ZPD, and refuse to let the student skip. These goals are in direct tension with helpfulness.

The system resolves this tension architecturally: the LLM is never asked what to do. Only how to say what has already been decided.

That is the thing that is hard to build. That is the thing that makes it not a wrapper.

---

*This is V3. The architecture is now precise enough to hire engineers into.*
