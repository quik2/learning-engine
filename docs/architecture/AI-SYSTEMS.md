# AI Systems Architecture
*The models, the math, the training pipelines, and exactly how they connect.*

---

## The Correct Mental Model

The system has three AI components that are fundamentally different in kind:

1. **The Student Model** — a probabilistic inference machine. It maintains a belief state over a hidden variable (what the student actually knows) and updates that belief on every observation. This is not a language model. It is a Bayesian filter running over a structured graph.

2. **The Scheduling Engine** — a decision function. It takes the belief state and outputs an action (what to teach next). In Phase 1 it is a deterministic policy. In Phase 2 it is a contextual bandit. In Phase 3 it is a learned policy from deep RL. The distinction matters because each phase requires different data, different training, and different trust.

3. **The Content Pipeline** — a constrained generative system. It takes a fully-specified instruction and produces text. The LLM here is not making pedagogical decisions. It is a high-quality text renderer.

These three components interact but are architecturally separate. A failure in the content pipeline (hallucinated explanation) does not corrupt the student model. A wrong pedagogical decision from the scheduler does not contaminate the LLM. This separation is intentional and load-bearing.

---

## Component 1: The Student Model

### The problem it solves, stated precisely

Student knowledge is a **hidden state**. We cannot observe it directly. We observe proxies: correct answers, wrong answers, response time, confidence ratings. The student model is a system that maintains a probability distribution over that hidden state and updates it as observations arrive.

This is formally a **Partially Observable Markov Decision Process (POMDP)** from the system's perspective:
- Hidden state S: actual knowledge for each concept
- Observations O: (correct/wrong, response_time, confidence, card_type)
- Belief state b(S): our probability distribution over S
- Transitions: learning (state changes when knowledge is acquired or forgotten)

The student model is the **belief updater**. Every observation triggers a Bayesian update of b(S). The scheduler then acts on b(S), not on any single observation.

### Architecture: DKT-GAT

**What standard DKT does:**

DKT (Piech et al., NeurIPS 2015) frames knowledge tracing as a sequence modeling problem. Given a student's interaction history h_t = [(c_1, r_1), (c_2, r_2), ..., (c_t, r_t)] where c_i is concept and r_i is correct/incorrect, predict P(correct on c_{t+1}).

The original used an LSTM. The hidden state h captures an implicit, undifferentiated representation of what the student knows. It works. It has two problems:

1. Knowledge of concept A is entangled in the hidden vector with knowledge of concepts B through Z. There is no interpretable per-concept mastery.
2. The model ignores the prerequisite structure of the domain. It doesn't know that failing concept B gives information about concept A.

**The DKT-GAT upgrade:**

We replace the LSTM with a combination of:
- A **Heterogeneous Graph Attention Network (HGT)** over the DPKG
- A **Transformer encoder** over the interaction sequence
- A **cross-attention fusion layer** that combines graph-informed concept representations with sequence-informed student state

```
Architecture detail:

Input layer:
  interaction_i = [concept_embed(c_i),    # 768-dim Nomic embedding
                   r_i,                    # binary
                   log_rt_normalized_i,    # log(response_time / baseline)
                   conf_i,                 # binary
                   bloom_level(c_i),       # 6-class one-hot
                   session_position_i]     # normalized 0-1

HGT over DPKG:
  For each concept node c in graph G:
    Neighbors: prerequisite concepts (HARD edges get weight 1.0, SOFT 0.5)
    Message function: m_{j→i} = W_k · h_j (typed by edge type)
    Aggregation: h_i = σ(Σ_j attention(i,j) · m_{j→i})
    
  Output: graph-informed concept embeddings G_embed ∈ R^{|concepts| × d}
  
  Key property: if student masters concept A (prerequisite of B),
  h_B updates to reflect higher prior for B — even before B is attempted.
  If student fails B, the failure signal propagates back along prerequisite
  edges, updating priors for A and siblings.

Transformer encoder over interaction sequence:
  Input: stack of interaction_i vectors (padded to max_seq_len=512)
  Architecture: 4 layers, 8 heads, d_model=256
  Output: H ∈ R^{seq_len × 256} (contextualized student state)
  
  Key property: full attention over entire history. Interaction from
  week 3 can directly inform prediction at week 8. No vanishing gradient
  from long sequences.

Cross-attention fusion:
  For each concept c we want to predict:
    query = G_embed[c]           # graph-informed concept representation
    keys = H                      # student interaction history
    values = H
    
    student_concept_state = CrossAttention(query, keys, values)
    # Attends to the relevant parts of history for THIS concept
    
  This asks: "given the graph's knowledge of concept c, what parts of
  this student's history are most relevant for predicting their mastery of c?"

Output head:
  P(correct | c, student_history) = σ(W · [student_concept_state; G_embed[c]] + b)
  
  Output: probability vector over ALL concepts simultaneously
  (not just the next concept — full mastery map update on every interaction)
```

**Why cross-attention is the right architecture here:**

Cross-attention allows the concept-side (graph) and the student-side (sequence) to be developed independently and then fused. This matters because:
- The graph structure changes (new uploads, behavioral refinement) independently of the student's history
- The student's history grows while the concept graph is mostly stable during a study session
- Attention weights are interpretable: we can visualize which past interactions most informed the mastery prediction for a given concept

**Implementation:**
- PyTorch + PyTorch Geometric (PyG) for the HGT
- Flash Attention 2 for the transformer encoder (memory efficient, 2-4x faster)
- Total parameters: ~12M (small enough for CPU inference in a pinch, fast on A10G GPU)
- Inference latency: <50ms on A10G for a student with 200 interactions and 50 concepts
- Training: AdamW, lr=2e-4 with cosine decay, batch=128 student sessions

### The FSRS layer

FSRS runs **on top of** DKT-GAT, not instead of it. The two systems serve different purposes:

- **DKT-GAT**: predicts P(correct on next attempt) given history
- **FSRS**: maintains per-concept stability and difficulty parameters; computes the optimal review interval

They are combined as:
```python
def mastery_estimate(student_id, concept_id) -> MasteryState:
    p_correct = dkt_gat.predict(student_id, concept_id)      # Sequence model
    stability = fsrs.get_stability(student_id, concept_id)    # Memory model
    difficulty = fsrs.get_difficulty(student_id, concept_id)  # Item model
    
    # P(correct at exam time) combines current state and forgetting
    days_to_exam = student.days_until_exam
    p_at_exam = p_correct * exp(-days_to_exam / stability)
    
    return MasteryState(
        p_correct_now=p_correct,
        stability=stability,
        difficulty=difficulty,
        p_correct_at_exam=p_at_exam,
        next_review_due=fsrs.next_review(stability, target_retention=0.90)
    )
```

FSRS parameters (stability S, difficulty D) are **fit per student per concept** from response history using the FSRS optimizer (open source, Rust-based via `fsrs-rs-python`). The optimizer runs nightly for each student against their accumulated history.

This means: two students who both get glycolysis correct three times may have very different stability values if one got it correct on first try (high stability) and another required multiple attempts (low stability, lower D suggests intrinsic difficulty).

### The IRT layer

2PL Item Response Theory operates at the **question level**, not the concept level.

For every question in the system's pool:
```
P(correct | θ_student, a_item, b_item) = 1 / (1 + exp(-a_item × (θ_student - b_item)))
```

Where:
- `θ_student` = DKT-GAT's latent ability estimate for this concept (mapped from 0-1 to logit scale)
- `a_item` = discrimination: how sharply this question separates students by ability
- `b_item` = difficulty: the θ at which P(correct) = 0.50

Parameters fit using marginal maximum likelihood (MML) via `girth` (Python IRT library) once a question has ≥100 responses.

IRT's role is **question selection**, not mastery tracking. The scheduler uses Fisher Information to select the maximally informative question:

```python
def information(theta, a, b):
    p = 1 / (1 + exp(-a * (theta - b)))
    return a**2 * p * (1 - p)

def select_question(concept_id, student_theta, question_pool):
    # Select question that maximizes information at student's current ability
    return max(question_pool[concept_id],
               key=lambda q: information(student_theta, q.a, q.b))
```

Maximum information is achieved when the question difficulty matches the student's ability — when P(correct) ≈ 0.50. This implements adaptive testing within each concept: easy students get harder questions, struggling students get easier ones, until ability is estimated with sufficient precision.

### Confidence calibration as a model feature

Confidence ratings (Sure/Unsure) are not just UX. They are a second signal fed directly into the model.

**The extended interaction encoding:**

```python
# Standard DKT input: (concept_id, correct)
# Our input: (concept_id, correct, response_time, confidence, card_type, bloom_level)

# Additional derived features:
calibration_error = abs(confidence_stated - P(correct | history))
  # High calibration_error = student's confidence is unreliable
  
slip_probability = P(incorrect | confidence="Sure")
  # Estimated from history: when this student says "Sure", how often are they wrong?
  # High slip_prob → discount confident correct answers

guess_probability = P(correct | confidence="Unsure", fast_response=True)
  # Fast + unsure + correct = likely guessing, not knowledge
  # High guess_prob → discount uncertain correct answers
```

These features are computed per student from their rolling 30-interaction window and fed as additional inputs to the DKT-GAT. The model learns to weight observations based on their reliability signal.

**The hypercorrection scheduler:**

```python
def schedule_hypercorrection(interaction: Interaction) -> Optional[ReviewJob]:
    if (interaction.confidence == "sure" and 
        not interaction.correct and
        interaction.calibration_error < 0.3):  # Student is usually well-calibrated
        
        # This is a genuine surprise — strong encoding of correction expected
        # Schedule review within 48-72 hours
        return ReviewJob(
            student_id=interaction.student_id,
            concept_id=interaction.concept_id,
            due_at=now() + timedelta(hours=random.uniform(48, 72)),
            priority=ReviewPriority.HYPERCORRECTION,
            include_misconception_correction=True
        )
    return None
```

### The cold start protocol

A new student has zero history. DKT-GAT has no signal. FSRS has no parameters. IRT cannot estimate θ.

**Three-stage cold start:**

Stage 1 (0 interactions): Initialize all concept mastery at prior = 0.10. Use population-mean FSRS parameters (S=7.0 days, D=5.0). Show diagnostic questions selected by concept graph centrality (concepts that, if mastered, give most information about the graph's prerequisite structure).

Stage 2 (3-10 interactions): DKT-GAT produces meaningful predictions. FSRS parameters begin fitting. Show questions targeting concepts where uncertainty in the mastery estimate is highest (maximum entropy selection).

Stage 3 (10+ interactions): Full model. IRT parameters begin accumulating (still at population mean per-question for IRT; per-student IRT ability estimated from DKT-GAT latent state).

---

## Component 2: The Scheduling Engine

### Phase 1: Deterministic Policy

The Phase 1 scheduler is a priority function, not a learned policy. Every session it scores all concepts and selects a session plan.

**The full priority function:**

```python
def concept_priority(concept: ConceptNode, student: Student) -> float:
    mastery = student_model.p_mastered(student.id, concept.id)
    stability = fsrs.get_stability(student.id, concept.id)
    days_to_exam = student.days_until_exam
    
    # Factor 1: Urgency — how much does NOT studying this hurt?
    forgetting_rate = exp(-days_to_exam / stability) if stability > 0 else 0
    p_at_exam_without_study = mastery * forgetting_rate
    p_at_exam_with_study = fsrs.predict_post_study(student.id, concept.id, days_to_exam)
    marginal_gain = (p_at_exam_with_study - p_at_exam_without_study) * concept.exam_weight
    
    # Factor 2: Feasibility — is this in the ZPD?
    prerequisite_mastery = mean([
        student_model.p_mastered(student.id, prereq_id)
        for prereq_id in concept.prerequisite_ids
    ]) if concept.prerequisite_ids else 1.0
    in_zpd = (prerequisite_mastery > 0.75) and (mastery < 0.75)
    feasibility = float(in_zpd)
    
    # Factor 3: Due — is this concept scheduled for review today?
    review_due = fsrs.is_due(student.id, concept.id, today())
    due_weight = 2.0 if review_due else 1.0
    
    # Factor 4: Metacognitive danger — student thinks they know it but don't
    blindness_gap = max(0, student.stated_confidence(concept.id) - mastery)
    danger_boost = 1.0 + blindness_gap  # Up to 2x for maximally blind student
    
    # Factor 5: Interference penalty — adjacent concept reviewed too recently
    interference_penalty = compute_interference_penalty(concept, student.recent_reviews)
    
    return (marginal_gain * due_weight * danger_boost * feasibility) - interference_penalty
```

**Interference penalty computation:**

```python
def compute_interference_penalty(concept: ConceptNode, 
                                  recent_reviews: List[ReviewRecord]) -> float:
    penalty = 0.0
    for review in recent_reviews:
        # Semantic similarity from Qdrant ANN search
        similarity = concept_similarity_cache.get(concept.id, review.concept_id)
        hours_since = (now() - review.timestamp).hours
        
        # Penalty decays over 24 hours; max at same-session review
        recency_weight = max(0, 1 - hours_since / 24)
        penalty += similarity * recency_weight * 0.3  # 0.3 = interference coefficient
    
    return min(penalty, 1.0)  # Cap at 1.0
```

### Phase 2: Contextual Bandit for Explanation Selection

This is the first learned component. It operates over one specific decision: which explanation variant to show for a given concept to a given student.

**State space:**
```python
@dataclass
class StudentContext:
    # Mastery profile (dim: concept_count, up to 200)
    mastery_vector: np.ndarray
    
    # Learning signals
    calibration_score: float          # How well-calibrated is this student?
    mean_response_time_ms: float      # Are they fast or slow processors?
    session_completion_rate: float    # Do they finish sessions?
    
    # Current context
    days_until_exam: int
    concept_difficulty: float
    bloom_target_level: int
    exam_format: int                  # MC=0, free_recall=1, etc.
    
    # Recent history features
    consecutive_correct: int          # Momentum signal
    hours_since_last_session: float
```

**Thompson Sampling implementation:**

```python
class ExplanationBandit:
    """
    Contextual Thompson Sampling for explanation variant selection.
    
    For each variant v, maintains:
      - Linear model: expected_reward = w_v · context + b_v
      - Uncertainty: Σ_v (posterior covariance of w_v)
    
    Thompson Sampling draws from posterior and selects argmax.
    """
    
    def __init__(self, n_arms: int, context_dim: int, lambda_prior: float = 1.0):
        self.n_arms = n_arms
        self.context_dim = context_dim
        
        # Prior: w_v ~ N(0, λ^{-1} I)
        self.precision = [lambda_prior * np.eye(context_dim) for _ in range(n_arms)]
        self.precision_weighted_mean = [np.zeros(context_dim) for _ in range(n_arms)]
        
    def select(self, context: np.ndarray) -> int:
        sampled_weights = []
        for arm in range(self.n_arms):
            # Posterior covariance: Σ = precision^{-1}
            cov = np.linalg.inv(self.precision[arm])
            mean = cov @ self.precision_weighted_mean[arm]
            
            # Sample from posterior
            w_sample = np.random.multivariate_normal(mean, cov)
            sampled_weights.append(w_sample @ context)
        
        return np.argmax(sampled_weights)
    
    def update(self, arm: int, context: np.ndarray, reward: float):
        # Bayesian update (Sherman-Morrison rank-1 update)
        self.precision[arm] += np.outer(context, context)
        self.precision_weighted_mean[arm] += reward * context
```

**Reward signal:**

The bandit's reward is the composite of immediate and delayed outcomes:

```python
def compute_reward(interaction: Interaction) -> float:
    immediate = float(interaction.correct) * 0.4
    
    # Delayed reward arrives when day3 or day7 retrieval is recorded
    # Use expected value (predicted from model) until ground truth arrives
    delayed_predicted = delayed_reward_model.predict(interaction)
    
    return immediate + delayed_predicted * 0.6
```

The delayed reward model is a gradient boosted tree trained on historical (immediate_features → day7_retention) pairs. It learns to predict 7-day retention from immediate response patterns.

### Phase 3: Deep RL Policy

The Phase 3 scheduler is a neural policy trained end-to-end to maximize expected exam score.

**Formal POMDP specification:**

```
State space S:
  s_t = (mastery_vector_t,          # P(mastered) for all concepts
          stability_vector_t,        # FSRS stability per concept
          days_until_exam_t,
          session_history_t[-10:],   # Last 10 interactions
          calibration_score_t)

Observation space O:
  o_t = (interaction: correct/wrong, response_time, confidence)
  — Noisy observation of true knowledge state

Belief state b_t:
  Maintained by DKT-GAT: b_t = P(s_t | o_1, ..., o_t)

Action space A:
  a_t = (concept_id, card_type, scaffold_level, question_format)
  
  For 50 concepts × 6 card types × 5 scaffold levels × 4 question formats:
  |A| ≈ 6,000 (discretized; continuous action approximation in practice)

Reward function r(s_t, a_t, s_{t+1}):
  Dense: +0.1 per checkpoint accuracy point above 0.5
  Dense: +0.05 per calibration improvement point
  Dense: -0.1 per session abandon event
  Sparse terminal: (actual_exam_score - baseline_predicted_score) × 10
    — Only available when student reports exam result
    — Hindsight credit assignment used to attribute this to earlier actions

Discount factor γ = 0.95 (care about future outcomes, but not infinitely)

Policy architecture:
  Input: b_t (belief state from DKT-GAT, dim=50×7=350 for 50 concepts)
  Hidden: 3-layer MLP, 512 units each, ReLU
  Output: Q(s, a) for all actions (DQN), or policy π(a|s) + value V(s) (PPO)
```

**The credit assignment problem:**

Exam score is observed ~8 days after the last session. Many teaching decisions contributed to it. Hindsight Credit Assignment (HCA) works by:

1. When terminal reward arrives, replay the trajectory that led to it
2. Use a hindsight model to estimate which actions at which timesteps contributed most to the final outcome
3. Assign dense pseudo-rewards retroactively to those timesteps

```python
def assign_hindsight_credit(trajectory: List[Transition], 
                             terminal_reward: float) -> List[float]:
    # Hindsight model: P(terminal_reward | trajectory[:t])
    # Trained on historical (trajectory, outcome) pairs
    
    baseline = [hindsight_model.predict(trajectory[:t]) 
                for t in range(len(trajectory))]
    
    # Credit = marginal contribution at each step
    credits = [baseline[t] - baseline[t-1] if t > 0 else baseline[0]
               for t in range(len(trajectory))]
    
    # Scale by terminal reward
    return [credit * terminal_reward for credit in credits]
```

**Why not Phase 3 at launch:**

Phase 3 requires:
1. Thousands of complete (session_sequence, exam_outcome) pairs to train the hindsight model
2. A stable base policy (Phase 1/2) to generate training trajectories
3. Careful evaluation: RL policies can find unexpected exploits (e.g., showing the easiest concepts always produces high checkpoint accuracy even if exam score doesn't improve)
4. Regulatory consideration: using RL to optimize student outcomes requires demonstrating the reward function is aligned with genuine learning, not proxies that look good but harm learning

Phase 3 is the 18-month goal. The architecture is specified now so the data collection is structured to enable it.

---

## Component 3: The Content Pipeline

### The core constraint: LLM as renderer, not reasoner

Every architectural decision in the content pipeline enforces one principle: the LLM fills a template it did not create. The moment the LLM is asked to decide what pedagogical move to make, the system has failed.

**Implementation with Instructor + Pydantic:**

```python
from instructor import from_anthropic, Mode
from pydantic import BaseModel, field_validator, model_validator
from typing import Literal, Optional

class TeachCard(BaseModel):
    """A single teaching card. LLM fills this; code defines its constraints."""
    
    concept_name: str
    explanation: str
    
    # Structural constraints enforced by Pydantic (before LLM output is accepted)
    @field_validator('explanation')
    @classmethod
    def validate_length(cls, v: str) -> str:
        word_count = len(v.split())
        if word_count > 120:
            raise ValueError(f"Explanation too long: {word_count} words (max 120)")
        return v
    
    @field_validator('explanation')
    @classmethod  
    def validate_single_concept(cls, v: str, info) -> str:
        # Check that only vocabulary from the target concept appears
        # (seductive detail prevention)
        # Implementation: NER + concept vocabulary intersection check
        return v

class PretestCard(BaseModel):
    question: str
    options: Optional[list[str]]    # None for free recall
    correct_answer: str
    distractors_are_misconceptions: bool
    confidence_required: Literal[True]  # Always required; can't be False
    
    @model_validator(mode='after')
    def answer_not_in_question(self) -> 'PretestCard':
        # Semantic check: does the question text hint at the answer?
        similarity = embed_similarity(self.question, self.correct_answer)
        if similarity > 0.75:  # Threshold from empirical testing
            raise ValueError("Question hints at answer (sim={:.2f})".format(similarity))
        return self

# Instructor client with automatic retry on validation failure
client = from_anthropic(anthropic.Anthropic(), mode=Mode.ANTHROPIC_TOOLS)

def generate_pretest(spec: CardSpec) -> PretestCard:
    # Instructor automatically retries up to max_retries on ValidationError
    return client.chat.completions.create(
        model="claude-haiku-3-5",  # Cheap; pretests are structurally simple
        response_model=PretestCard,
        max_retries=3,
        messages=[{
            "role": "user",
            "content": build_pretest_prompt(spec)
        }]
    )
```

When `PretestCard` validation fails (question hints at answer, explanation too long, etc.), Instructor automatically retries with the validation error message included. The LLM learns from its constraint violations within the same call. Max 3 retries; if still failing, escalate to Sonnet.

**Grammar-constrained decoding for local models (Phase 4):**

When the fine-tuned teaching model runs locally (Phase 4, post-Series A), we use XGrammar or Outlines for token-level constraint enforcement. This is stronger than post-hoc validation: the model cannot even generate tokens that would violate the schema.

```python
# With Outlines (local model serving)
import outlines
import outlines.models as models

model = models.transformers("teaching-engine-7b")  # Our fine-tuned model

@outlines.prompt
def pretest_prompt(spec: CardSpec):
    """Generate a pretest question for {{spec.concept.name}}.
    The question must NOT hint at the answer. Use these distractors: {{spec.distractors}}"""
    ...

generator = outlines.generate.json(model, PretestCard)  # Token-level JSON enforcement
card = generator(pretest_prompt(spec))
# card is guaranteed to be valid PretestCard — no post-hoc validation needed
```

### The population explanation pool

This is the most important component for long-term cost reduction and quality improvement.

**Data model:**

```sql
CREATE TABLE explanation_variants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    concept_id UUID NOT NULL REFERENCES concepts(id),
    content JSONB NOT NULL,              -- Full card content
    
    -- Performance metrics
    checkpoint_accuracy FLOAT,
    day3_retention FLOAT,
    day7_retention FLOAT,
    
    -- Statistical metadata
    n_exposures INTEGER DEFAULT 0,
    n_day7_measured INTEGER DEFAULT 0,
    confidence_lower FLOAT,              -- Wilson CI lower bound
    confidence_upper FLOAT,
    
    status VARCHAR(20) DEFAULT 'active', -- active | champion | retired
    created_at TIMESTAMPTZ DEFAULT NOW(),
    last_updated TIMESTAMPTZ DEFAULT NOW()
);

-- The champion is the one served to new students
CREATE UNIQUE INDEX idx_champion_concept 
  ON explanation_variants (concept_id) 
  WHERE status = 'champion';
```

**The tournament process:**

```python
class ExplanationTournament:
    MIN_EXPOSURES = 50            # Minimum to enter tournament
    MIN_DAY7_MEASURED = 30        # Minimum for delayed outcome analysis
    CHAMPION_LOWER_CI = 0.65      # Champion must have lower CI above 0.65
    RETIRE_UPPER_CI = 0.55        # Retire variants with upper CI below 0.55
    
    def run(self, concept_id: UUID):
        variants = ExplanationVariant.query.filter_by(
            concept_id=concept_id,
            status__in=['active', 'champion']
        ).all()
        
        # Score each variant
        for variant in variants:
            if variant.n_day7_measured >= self.MIN_DAY7_MEASURED:
                # Primary metric: 7-day retention (direct measurement)
                score = variant.day7_retention
                n = variant.n_day7_measured
            elif variant.n_exposures >= self.MIN_EXPOSURES:
                # Proxy metric: weighted combination of checkpoint + day3
                score = 0.4 * variant.checkpoint_accuracy + 0.6 * variant.day3_retention
                n = variant.n_exposures
            else:
                continue  # Not enough data
            
            # Wilson confidence interval
            lo, hi = wilson_ci(successes=int(score * n), n=n, confidence=0.95)
            variant.confidence_lower = lo
            variant.confidence_upper = hi
        
        # Sort by lower confidence bound (conservative selection)
        ranked = sorted(variants, key=lambda v: v.confidence_lower, reverse=True)
        
        if not ranked:
            return
        
        new_champion = ranked[0]
        
        # Promote champion
        if new_champion.confidence_lower >= self.CHAMPION_LOWER_CI:
            # Demote current champion
            for v in variants:
                if v.status == 'champion' and v.id != new_champion.id:
                    v.status = 'active'
            new_champion.status = 'champion'
        
        # Retire poor performers
        for variant in ranked[1:]:
            if variant.confidence_upper < self.RETIRE_UPPER_CI:
                variant.status = 'retired'
```

**The cache serving layer:**

```python
def get_explanation(concept_id: UUID, student_context: StudentContext) -> ExplanationContent:
    # Check population pool first
    champion = ExplanationVariant.query.filter_by(
        concept_id=concept_id, status='champion'
    ).first()
    
    if champion and champion.confidence_lower >= 0.65:
        # Serve champion to 90% of students (exploitation)
        # Reserve 10% for exploration (new variants)
        if random.random() < 0.90:
            return champion.content
    
    # Generate new variant (either cold start or exploration budget)
    return generate_new_variant(concept_id, student_context)
```

### Model tiering by card type

```
Card complexity → model selection → cost optimization

FORWARD_TEST, simple PRETEST:
  Model: claude-haiku-3-5
  Latency: ~400ms
  Cost: ~$0.0001/card
  Why: Simple task; structured; Pydantic enforces constraints

TEACH (population pool hit):
  Model: None (cache hit)
  Latency: <5ms (database read)
  Cost: $0.00
  
TEACH (new concept, first encounter):
  Model: claude-sonnet-4-6
  Latency: ~1200ms
  Cost: ~$0.003/card
  Why: Complex task; quality critical on first impression

CHECKPOINT with ELABORATE:
  Model: claude-sonnet-4-6
  Latency: ~1000ms
  Cost: ~$0.002/card

FACT_CHECK (validation pass):
  Model: claude-haiku-3-5
  Latency: ~300ms
  Cost: ~$0.00005/card

MISCONCEPTION CORRECTION:
  Model: claude-sonnet-4-6
  Latency: ~1000ms
  Cost: ~$0.002/card

Voice/audio question generation:
  Model: claude-haiku-3-5 (short, binary-evaluable questions)
  Latency: ~300ms
  Cost: ~$0.00005/card
```

---

## The Training Pipeline

### DKT-GAT training and fine-tuning

**Pretraining data:**
- ASSISTments 2017: 100 students × 4M interactions (math skills, K-12)
- EdNet-KT1: 780K students × 95M interactions (English exam prep)
- Open-spaced-repetition Anki revlogs: 10K users × 100M review logs

These datasets don't have our exact feature set (no confidence ratings, no response time in all cases). We train with available features and use learned feature imputation for missing ones.

**Training objective:**

Binary cross-entropy on next-interaction correctness prediction:
```
L = -Σ_t [y_t log(ŷ_t) + (1 - y_t) log(1 - ŷ_t)]

where ŷ_t = DKT-GAT output for concept c_t
      y_t = actual correctness of interaction t
```

**Evaluation:**

AUC-ROC on held-out 20% of student sessions. Target: AUC > 0.82 (state-of-art on ASSISTments is ~0.83-0.86 depending on model). Our graph attention component should close the gap.

**Fine-tuning protocol:**

```
Week 0-3: Pretrained weights only. No fine-tuning.
Week 4: First fine-tune run.
  Dataset: accumulated proprietary interactions (filter: concepts with ≥20 student interactions)
  Method: LoRA (Low-Rank Adaptation) — freeze base weights, train rank-8 adapters
    Why LoRA: Prevents catastrophic forgetting of pretraining knowledge
              Fast: only 0.5-1% of parameters trained
              Cheap: A10G, ~2 hours for 50K interaction updates
  Schedule: Weekly thereafter
  Deployment: New weights if AUC improvement > 0.5% on validation set
```

**Catastrophic forgetting mitigation:**

Standard fine-tuning on new data causes the model to forget previously learned patterns. We use:
1. LoRA (adapter-based: base weights frozen)
2. Experience replay: 20% of each fine-tuning batch from pretraining data (maintains general knowledge)
3. Elastic weight consolidation (EWC) for the non-LoRA layers: penalize updates to weights important for pretraining performance

### The fine-tuned teaching model (Phase 4)

**Architecture:**
- Base: Llama 3.3 8B Instruct (stronger instruction following, smaller than 70B)
- Training: SFT → DPO → RLAIF
- Target parameter count after fine-tuning: ~8B (no pruning; quality over size at this stage)

**Phase 4a: Supervised Fine-Tuning (SFT)**

```
Training data: (CardSpec, champion_explanation) pairs
  — Only pairs where champion_explanation has confidence_lower ≥ 0.70
  — Estimated data requirement: 100K high-quality pairs across concepts

Training:
  Method: QLoRA (4-bit quantized base + LoRA adapters)
  GPU: 2x A100 80GB via RunPod (~$8/hour)
  Duration: ~24 hours for first run
  Cost: ~$200 for initial training run
  
  Loss: Standard cross-entropy on explanation tokens
  Input format:
    [SYSTEM: You are a teaching engine. Generate the specified card type exactly.
     SPEC: {card_spec_json}
     PEDAGOGICAL_CONSTRAINTS: {constraints}]
    [ASSISTANT: {champion_explanation_content}]
```

**Phase 4b: Direct Preference Optimization (DPO)**

DPO trains the model to prefer high-retention explanations over low-retention ones, without requiring a separate reward model.

```
Training data: (CardSpec, winning_explanation, losing_explanation) triples
  — Winner: explanation with day7_retention > 0.70
  — Loser: explanation of same concept with day7_retention < 0.50
  — Requirement: ~20K triples minimum

DPO loss:
  L_DPO = -E[log σ(β(log π_θ(y_w|x)/π_ref(y_w|x) - 
                     log π_θ(y_l|x)/π_ref(y_l|x)))]
  
  where y_w = winning explanation
        y_l = losing explanation
        π_θ = model being trained
        π_ref = frozen SFT model
        β = 0.1 (KL penalty coefficient)
```

This teaches the model to generate explanations that look more like the ones that produced high retention and less like the ones that produced low retention — without any explicit reward model.

**Phase 4c: RLAIF (Reinforcement Learning from AI Feedback)**

```
Reward model: Claude Opus as judge
  Prompt: "Rate this educational explanation on these 5 dimensions:
    1. Retrieval-forcing: Does it withhold information before the checkpoint?
    2. Scaffolding appropriateness: Matches stated mastery level?
    3. Single-concept focus: No seductive details?
    4. Accurate: No factual errors relative to source?
    5. Pedagogically sound: Uses correct principles (generation effect, etc.)?"
  Output: 5 scores (0-10 each) → composite reward signal

PPO training:
  Use composite reward to improve beyond DPO
  KL penalty to prevent reward hacking (diverging too far from DPO model)
```

**What this model knows that Claude doesn't:**

A general Claude model is trained to be helpful. Helpful in educational contexts means explaining things clearly when asked. That is precisely wrong for retrieval practice — the model should withhold explanation until the student has attempted retrieval.

The fine-tuned teaching model is trained specifically on (spec → champion_content) pairs where champion_content was the explanation that forced retrieval and produced retention. It has learned, from behavioral outcome data, what pedagogically effective card content looks like. This is a capability no general model has, because no general model has been trained on behavioral outcome data from students.

---

## Inference Serving Architecture

### Latency budget per session interaction

```
Student submits answer →
  1. Answer evaluation: 50ms (embedding similarity) or 850ms (+ LLM borderline)
  2. Student model update: 45ms (DKT-GAT forward pass)
  3. FSRS update: 5ms (closed-form)
  4. Scheduler: 20ms (priority function over all concepts)
  5. Card spec generation: 10ms (deterministic)
  6. Content generation: 300-1200ms (depends on model tier)
  7. Fact check: 250ms (Haiku call)
  8. Database write: 10ms
  
Total: 700ms (easy card) to 2.4s (complex new concept)
Target: <2s for 90th percentile interaction
```

**Latency optimization techniques:**

1. **Prefetching**: After the student starts a session, prefetch the likely next 3 card specs and begin generating them speculatively. Most sessions follow predictable sequences.

2. **Model tiering**: Claude Haiku for high-frequency simple cards. Sonnet only for complex generation. Cost and latency both optimize together.

3. **Response streaming**: Stream card content token-by-token. The student sees the card begin appearing after ~200ms even if full generation takes 1200ms. UX perception of latency is dramatically reduced.

4. **Embedding cache**: Concept embeddings are static (only change when the graph updates, not per-session). Cache all 50K+ concept embeddings in Redis. Cosine similarity computation is then matrix multiplication, not an API call.

5. **DKT-GAT batching**: Multiple students' model updates can be batched together for GPU efficiency. In practice, student interactions are somewhat bursty (exam cycles), so batch size varies. Target: 32-64 student updates per GPU batch.

### GPU allocation strategy

```
Always-on (A10G, 1 GPU):
  - DKT-GAT inference (production)
  - FSRS parameter serving (CPU)
  - Qdrant (CPU with HNSW index)
  
Bursty (autoscale):
  - Card generation (Claude API — no GPU needed)
  - Answer evaluation (embedding + small LLM — Haiku, CPU-feasible)
  
Weekly (spot instances, A100):
  - DKT-GAT fine-tuning (LoRA, 2-4 hours)
  - Explanation tournament (batch compute, CPU)
  - FSRS optimizer runs (CPU, parallelizable)
  
Monthly (spot instances, 2×A100):
  - Teaching model training (Phase 4 only)
  - IRT parameter fitting (CPU)
```

---

## Observability and the ML Feedback Loop

### What gets logged (exhaustive)

```python
@dataclass
class InteractionLog:
    # Identity
    interaction_id: UUID
    student_id: UUID          # Anonymized UUID, no PII
    session_id: UUID
    
    # What was shown
    concept_id: UUID
    card_type: CardType
    explanation_variant_id: Optional[UUID]
    scaffold_level: float
    question_format: QuestionFormat
    
    # What the model predicted
    p_mastered_before: float      # DKT-GAT before this interaction
    ifo_score: float              # IRT information at this question
    scheduled_by: SchedulerPhase  # DETERMINISTIC | BANDIT | RL
    
    # What happened
    student_answer: Optional[str]
    correct: bool
    confidence: Optional[str]
    response_time_ms: int
    
    # Downstream outcomes (populated later)
    p_mastered_after: float       # DKT-GAT after update
    day3_correct: Optional[bool]  # Populated 3 days later
    day7_correct: Optional[bool]  # Populated 7 days later
    exam_score: Optional[float]   # Populated when student reports back
    
    # Model state snapshot
    dkt_model_version: str        # Which DKT weights were used
    fsrs_stability_at_time: float
    
    # Cost tracking
    llm_model_used: Optional[str]
    llm_cost_usd: Optional[float]
    content_source: ContentSource  # LLM_GENERATED | POPULATION_POOL
```

### The evaluation suite (automated, runs weekly)

```python
class SystemEvaluation:
    def run(self):
        results = {}
        
        # 1. DKT accuracy (knowledge tracing)
        results['dkt_auc'] = evaluate_dkt_auc(held_out_sessions)
        
        # 2. FSRS retention prediction accuracy
        results['fsrs_mae'] = mean_abs_error(
            predicted=[fsrs.predict_retention(s, c, t) 
                       for s, c, t in day7_review_pairs],
            actual=[day7_review_pairs[s, c].correct for ...]
        )
        
        # 3. Exam score prediction accuracy (where ground truth exists)
        results['score_prediction_mae'] = mean_abs_error(
            predicted=[exam_readiness_model.predict(s) for s in students_with_scores],
            actual=[s.reported_exam_score for s in students_with_scores]
        )
        
        # 4. Calibration: are confidence ratings meaningful?
        results['sure_accuracy'] = P(correct | confidence == "sure")
        results['unsure_accuracy'] = P(correct | confidence == "unsure")
        
        # 5. Champion explanation quality
        results['champion_7day_retention'] = mean([
            v.day7_retention for v in ExplanationVariant.query.filter_by(status='champion')
        ])
        
        # 6. Scheduler diversity (are we avoiding interference?)
        results['semantic_adjacent_consecutive_rate'] = measure_adjacency_rate(
            recent_sessions
        )
        
        # 7. Cost per session
        results['mean_cost_per_session_usd'] = mean([
            s.total_llm_cost_usd for s in recent_sessions
        ])
        
        # Alert if any metric degrades > 5% from baseline
        self.check_regressions(results)
        
        return results
```

---

## The Key Insight That Ties Everything Together

Every AI component in this system is solving a different problem at a different timescale:

| Component | Problem | Timescale | Update frequency |
|---|---|---|---|
| DKT-GAT | What does this student know right now? | Per interaction (seconds) | Inference: continuous; Weights: weekly |
| FSRS | When will this student forget? | Per concept per student | Parameters: nightly |
| IRT | What question maximizes information? | Per question selection | Parameters: nightly |
| Phase 1 Scheduler | What should they study next? | Per session | Deterministic; no updates |
| Phase 2 Bandit | Which explanation works best for this student? | Per explanation selection | Continuous Bayesian update |
| Phase 3 RL | What teaching policy maximizes exam score? | Offline, episodic | Monthly, post Phase 2 |
| Content Generator | How should this be said? | Per card | Weights: monthly (Phase 4) |
| Explanation Pool | Which explanation actually produces retention? | Nightly tournament | Promoted: when CI threshold met |

The system is an ensemble of specialized AI components, each operating at the timescale appropriate to its decision. The student model updates every second. The RL policy updates monthly. The explanation pool is evaluated nightly.

No single model makes all the decisions. No single model could — the decisions span too many timescales and too many objectives. The architecture's job is to keep these components correctly specialized and correctly integrated.

That is what makes it not a GPT wrapper.

---

*This document specifies every AI component with enough precision to implement. If anything is underspecified, open an issue.*
