# The Learning Engine — Architecture V5
*The honest version. Built on what the research actually says.*

---

## What V4 Got Wrong

V4 was a significant improvement. The Student Intelligence File, the Relevance Engine, the Minimal Intervention Principle — those are right. But Scout's critique exposed three things that need to be fixed before this architecture is real:

**1. The architecture is a sophisticated textbook, not a tutor.**
There is no pathway by which a student can communicate confusion. Wrong answer → reteach with different format. That is a 1950s teaching machine with better scheduling. Bloom's tutors had conversations. This system doesn't. That gap is the gap between g = 0.35 and g = 2.0.

**2. The self-reference effect is being implemented wrong.**
The research (Symons & Johnson, d=1.11) is for tasks where the student explicitly evaluates a concept against themselves — active self-judgment. The architecture was delivering a passive analogy. That's closer to an elaborative interrogation effect (d=0.40), not the full SRE. The correct implementation: ask the student to generate the connection, not receive it.

**3. The ZPD is a binary gate when it needs to be a challenge gradient.**
No target error rate. No within-session difficulty management. No explicit flow zone targeting. A student sailing through at 95% correct is bored, not learning. A student failing at 60% is frustrated, not learning. 20-30% error rate is the optimal zone per IRT (maximum information) and desirable difficulties research. It is not tracked anywhere.

V5 fixes all three.

---

## The Alpha School Insight (Properly Understood)

Alpha School compresses learning by removing passive time — students aren't sitting through lectures on content they already know or aren't ready for. Two hours of active, mastery-gated learning replaces six hours of passive instruction + review.

This product doesn't replace lectures. It optimizes what happens after them. The correct framing: eliminate wasted review time by ensuring every minute of study is spent at the right concept, in the right format, at the right difficulty, with retrieval forced. That is meaningful compression — not 10x, but real. The Bastani study (PNAS, 2025) showed students with unrestricted AI scored 17% lower. Students with structured retrieval practice outperform them. That delta is the product's claim.

The bigger vision — what Quinn is gesturing at — is that this architecture could eventually replace passive instruction entirely, not just optimize review. That requires the dialogue layer. Which is why it's priority one.

---

## The Full Architecture

### Layer 0: Student Intelligence File (unchanged from V4, correct)

Maintained across all sessions. Tracks: academic context, inferred interests (never presumed), format response history (empirical, not VARK), analogy hit rate, engagement patterns, prior knowledge cross-course, time pressure, depth goal. Built from behavior, not forms.

Nothing changes here. V4 got this right.

---

### Layer 1: The Dialogue Layer (new — highest priority)

This is the single highest-leverage addition. No ML stack changes required. Adds bidirectional communication at two points.

**Point A: Checkpoint failure dialogue**

When a student fails a checkpoint (wrong answer), instead of immediately serving a reteach, the system asks one question:

> "Before I explain — tell me in your own words what you think the right answer is, and where you got stuck."

The student types freely. A classifier (fine-tuned on educational misconception data, or a prompted LLM call) maps the response to one of five categories:

```python
class ConfusionType(Enum):
    PREREQUISITE_GAP   # Student's answer reveals a gap in prerequisite knowledge
    MISCONCEPTION      # Student holds a specific wrong belief about this concept
    TERMINOLOGY_ERROR  # Understood the concept, used the wrong term
    PARTIAL_KNOWLEDGE  # Got part of it right, missing one sub-step
    COMPLETE_CONFUSION # No coherent understanding present
```

Each confusion type maps to a different repair:

```python
REPAIR_STRATEGIES = {
    ConfusionType.PREREQUISITE_GAP: RepairAction(
        target_concept="the prerequisite they're missing",
        approach="teach the prerequisite first, then return"
    ),
    ConfusionType.MISCONCEPTION: RepairAction(
        approach="name the misconception explicitly, contrast with correct understanding",
        note="hypercorrection effect: high-confidence wrong beliefs correct fastest when named"
    ),
    ConfusionType.TERMINOLOGY_ERROR: RepairAction(
        approach="confirm their understanding is correct, clarify the term",
        note="don't reteach the concept — just bridge the vocabulary"
    ),
    ConfusionType.PARTIAL_KNOWLEDGE: RepairAction(
        approach="affirm what they got right, teach only the missing sub-step",
        note="don't repeat what they already know — restart from their stopping point"
    ),
    ConfusionType.COMPLETE_CONFUSION: RepairAction(
        approach="rebuild from scratch with different format and worked example",
        note="full reteach warranted here, not elsewhere"
    )
}
```

This is the difference between a tutor and a textbook. The student who misread "ADP" as "ATP" gets a 10-second vocabulary correction. The student whose prerequisite is missing gets routed back to the prerequisite. The student with a specific misconception gets a targeted correction. None of these three students receive the same reteach — which is what V4's binary "failed → different format" would have given all three.

**Point B: Student-initiated questions**

At any point during a lesson, the student can ask a question. Bounded interface — a text field with the placeholder "Confused about something? Ask here." Not a full chatbot. A bounded retrieval-augmented response:

```
Student asks: "Why does glycolysis happen in the cytoplasm and not the mitochondria?"
→ Retrieve: relevant concept nodes (glycolysis, mitochondria, cellular location)
→ Retrieve: student's current mastery of those nodes
→ Generate: targeted answer using student context, grounded in source material
→ Constraint: one to three sentences. Link to relevant concept node for deeper reading.
```

This is not a general-purpose AI assistant. It is a question-answering layer over the concept graph, constrained to the source material, aware of the student's full learning history. The student cannot go off-topic. The student can get unstuck.

**What this changes:**

The product stops being a sophisticated textbook and starts being a tutor. A limited tutor — it can't notice that a student is frowning or pick up on a confused sigh — but one that can receive student-expressed confusion and respond appropriately. That is the core gap between this system and human tutoring. Not the ML sophistication. The ability to say "I don't get it" and have something happen.

---

### Layer 2: The Relevance Engine (V4, corrected)

The three-gate system from V4 is right. One correction: the self-reference effect implementation.

**V4:** System generates an approved connection and delivers it as one sentence.

**V5:** When a structural connection exists, the system asks the student to generate it first.

```
Before teaching glycolysis, the system detects: structural connection available
(invest resources upfront to get more back — known to map to student's background
in economics from prior upload with 0.82 familiarity score)

Instead of: "Think of it like investing — you spend to earn more back."

V5 prompt: "Before we get into the mechanism — can you think of any situation 
in your own life where you put something in upfront to get more back later? 
Take 10 seconds."

Student types: "Studying before an exam"
System: "Exactly. Glycolysis works the same way — invests 2 ATP in the first 
phase to harvest 4 ATP in the second. Net gain: 2."
```

This activates both the self-reference effect (student evaluated a concept against their own experience) and the generation effect (student produced the bridge themselves). Both effects are active simultaneously. The system contributed one sentence to confirm and connect. The student did the encoding work.

When no structural connection exists: NONE. Same as V4. The system does not manufacture one.

The 0.65 structural isomorphism threshold remains — but is now treated as a hypothesis to be validated. After 500 uses of each approved connection type across students, the behavioral data (did the student who generated a connection in this domain score higher on 7-day retrieval?) validates or invalidates the threshold. It gets tightened or loosened empirically.

---

### Layer 3: The Challenge Gradient (new)

V4's ZPD was a binary gate: prerequisite mastery > 0.75 and target mastery < 0.75 = teachable. Binary. Wrong.

V5 replaces this with an explicit challenge management loop targeting a 20-30% error rate throughout the session.

**Why 20-30%?**

Two converging evidence bases:
- IRT maximum information: Fisher Information I(θ) is maximized when P(correct) ≈ 0.50, but for sustained learning sessions (not single assessments), the optimal range is 0.65-0.80 correct to maintain engagement while ensuring genuine difficulty
- Desirable difficulties (Bjork): optimal difficulty for retention is higher than optimal difficulty for performance. Students performing at 80% correct are at the edge of their ZPD and encoding more deeply than students at 95% correct
- Flow state (Csikszentmihalyi): the flow zone is challenge slightly exceeding skill — which maps to approximately 70-80% success rate in educational contexts

**Target: 70-80% checkpoint accuracy per session. Track it in real time.**

```python
class ChallengeManager:
    TARGET_ACCURACY_MIN = 0.70
    TARGET_ACCURACY_MAX = 0.80
    WINDOW = 5  # rolling window of last 5 checkpoints
    
    def adjust(self, session: Session) -> ChallengeAdjustment:
        recent_accuracy = session.rolling_accuracy(self.WINDOW)
        
        if recent_accuracy > self.TARGET_ACCURACY_MAX:
            # Too easy. Student is bored, not encoding deeply.
            return ChallengeAdjustment(
                increase_bloom_level=True,
                reduce_scaffold=0.15,
                prefer_application_questions=True,
                note="Increase difficulty — student is in comfort zone, not ZPD"
            )
        
        elif recent_accuracy < self.TARGET_ACCURACY_MIN:
            # Too hard. Student is frustrated, not learning.
            return ChallengeAdjustment(
                increase_scaffold=0.20,
                prefer_lower_bloom=True,
                add_worked_example=True,
                note="Reduce difficulty — student is above frustration threshold"
            )
        
        else:
            # In the zone. Maintain.
            return ChallengeAdjustment(maintain=True)
```

**The Worked Example → Faded Example → Generation progression:**

For concept types where the student's accuracy on first encounter is < 0.50 (deep in the frustration zone), the system uses Sweller's scaffolding sequence instead of direct instruction:

1. **Worked example** (full solution shown, student follows): reduces extraneous load, allows schema formation
2. **Faded example** (partial solution, student fills in steps): bridges worked example to independent problem-solving
3. **Generation** (student produces the solution): full retrieval practice, maximum encoding

The transition point between stages is mastery-gated. The student moves to the faded example only when they can correctly predict the next step in the worked example. They move to generation only when they complete faded examples at > 0.75 accuracy. This implements the expertise reversal effect automatically — the scaffolding fades as competence grows.

**Engagement signal (improved from V4):**

V4 used response time alone — too noisy. V5 uses three signals:

```python
def engagement_score(session: Session, window_min: int = 3) -> float:
    # Signal 1: Response time trend (increasing = dropping engagement)
    rt_trend = session.response_time_trend(window_min)  # positive = increasing
    
    # Signal 2: Answer attempt quality
    # Did the student type something or just click an option?
    # Free-recall: length and coherence of typed answer
    attempt_quality = session.mean_attempt_quality(window_min)
    
    # Signal 3: Re-read rate
    # Did the student tap "show again" on the teach card before checkpointing?
    reread_rate = session.reread_rate(window_min)  # High reread = struggling
    
    # Weighted composite
    return (
        (1 - normalize(rt_trend)) * 0.40 +
        attempt_quality * 0.40 +
        (1 - reread_rate) * 0.20
    )
```

When engagement score drops below 0.35 for 3 consecutive minutes: session compression begins. Not a popup. The next card is shorter, the checkpoint is more direct, and the session is guided toward a natural close. If the student is genuinely engaged but struggling (low accuracy + high attempt quality), compression does not trigger — that's the right state to be in.

---

### Layer 4: The Session Architecture (V4 corrected)

The session graph from V4 is right in structure. Add the challenge gradient loop and the dialogue layer. The graph now looks like:

```
[Diagnostic: 1-3 questions]
    ↓
[Challenge Manager calibrates target difficulty]
    ↓
[Concept node selected by scheduler]
    ↓
    ┌──── Student asks question? → [Dialogue: retrieve + respond] ────┐
    │                                                                   │
[Format Reasoner selects format]                                       │
    ↓                                                                   │
[Content Pipeline generates card]                                       │
    ↓                                                                   │
[Relevance Engine: student generates connection if approved]            │
    ↓                                                                   │
[Checkpoint]                                                            │
    ↓                    ↓                                              │
[Pass: FSRS update]   [Fail: dialogue slot → confusion classification]  │
    ↓                    ↓                                              │
[Next concept]       [Targeted repair → checkpoint retry]              │
    ↓                                                                   │
[Challenge Manager: adjust difficulty based on rolling accuracy] ←──────┘
    ↓
[Continue or close based on time pressure + engagement]
```

The dialogue layer runs as a parallel track, always available, non-blocking.

---

### Layer 5: What the ML Stack Actually Does (unchanged from V3/AI-SYSTEMS)

The DKT-GAT, FSRS, IRT, three-phase scheduler — all correct and unchanged. The dialogue layer adds:

**Confusion classifier:** Fine-tuned on educational misconception data. Input: student's free-text explanation of their reasoning. Output: ConfusionType enum. This is a 5-class classification problem. Can be built with a prompted Claude Haiku call at low cost, then upgraded to a fine-tuned smaller model when enough labeled examples exist.

**Dialogue retrieval:** KNN search over the concept graph's embedded nodes using the student's question as the query. Returns the 2-3 most relevant concept nodes. Uses those as grounding context for a response. Already built (Qdrant + concept embeddings) — just not exposed to the student in V4.

**No new training required.** The dialogue layer is retrieval-augmented generation over existing infrastructure, plus a classifier. Neither requires changes to DKT-GAT, FSRS, or IRT.

---

### Layer 6: The Honest Compression Model

What this system actually compresses and by how much:

| Source of waste | Conventional study | This system | Estimated gain |
|---|---|---|---|
| Reviewing known material | Student re-reads everything | Skip to frontiers only | 30-40% time reduction |
| Passive re-reading instead of retrieval | Most study time | Retrieval-only sessions | 20-30% quality gain |
| Studying at wrong intervals | Random cramming | FSRS-optimized | 40% retention improvement at same time |
| Wrong format for this concept | Luck-based | Format-reasoned | 10-20% efficiency gain |
| Concept confusion undetected | Student moves on | Dialogue catches it | Eliminates silent confusion |
| Wrong concept studied | No sequencing | Prerequisites-first | 25% faster to mastery |

**Honest total:** Approximately 2-3x efficiency in terms of learning outcome per unit study time. Not 10x. Not "complete core subjects in 2 hours." The Alpha School 2-hour model compresses instruction time by replacing passive lectures with active mastery work — a different lever. This product compresses review time and eliminates wasted study effort. Both are real. They are different.

The claim that is defensible: "Students using this system achieve the same exam outcomes in roughly half the study time compared to conventional review methods." That is probably true for students who currently use passive review (rereading, highlighting). The Bastani PNAS data supports it.

---

### Layer 7: The Long Game (School Integration)

Quinn is right about the larger possibility. The architecture as specified is a study supplement. But its components — student intelligence file, adaptive sequencing, mastery-gated progression, dialogue layer — are exactly the components of a complete instructional system.

The path from study supplement to full instructional replacement:

**Phase 1 (now):** Study supplement. Student uploads professor's slides and uses the system to prepare for exams. Solves the review problem.

**Phase 2 (post-IRB study):** Course companion. Faculty integrate the tool into the course structure. Students are expected to use it between lectures. The system knows the course syllabus, the exam date, the expected Bloom's level for each concept. The dialogue layer handles questions students would normally bring to office hours.

**Phase 3 (institutional):** Partial lecture replacement. For concepts where the student has strong prerequisite mastery, the system teaches directly and the lecture is optional. For concepts where the whole class is weak, the lecture still happens. The system identifies which is which and tells the professor.

**Phase 4 (Alpha School model):** Full instructional replacement. The system handles all primary instruction for a subject. Two hours per day, mastery-gated, fully personalized. The teacher's role shifts to mentorship, project guidance, and social-emotional support. This is what Alpha School is doing — just with less sophisticated adaptive technology.

Phase 4 is where this becomes a genuinely revolutionary education product. The architecture described here — V5 — is directly extensible to Phase 4. The dialogue layer needs to become richer. The student intelligence file needs to track more longitudinal context. But the core systems are the same.

The IRB study at UCLA is not just a marketing move. It is the evidence base that makes Phase 2 and Phase 3 possible. "We showed in a randomized controlled trial that students in the treatment group outperformed controls by 0.5 SD on their actual exam" is the sentence that gets this into institutions.

---

## Summary: What Changed Across Versions

| Version | Core insight | What was wrong |
|---|---|---|
| V1 | Seven-layer system, LLM as component | Too generic, not differentiated |
| V2 | Learning science theory, moat logic | Still a GPT wrapper at heart |
| V3 | Precise ML architecture, DKT-GAT | Missing the product layer |
| V4 | Student Intelligence File, Relevance Engine | Sophisticated textbook, no dialogue |
| **V5** | **Dialogue layer + challenge gradient + generation-first SRE** | **This is the architecture** |

## What V5 Actually Builds

A system where:
- The student model tracks *who they are* (V4), *what they know* (V3), and *where their confusion lives* (V5)
- Every minute of study time is at the right concept, in the right format, at the right difficulty
- When a student is confused, they can say so and get a targeted response
- When a student understands something, they are pushed harder until they don't
- The lesson includes a personal connection only when the student generates one themselves
- The predicted exam score is real — calibrated against actual exam outcomes — and drives the student back to the product

What the student notices: it's fast, it adapts, it seems to know where they're stuck, it doesn't waste their time. What they don't notice: the DKT-GAT updating their mastery probability, the FSRS computing optimal review intervals, the challenge manager keeping them in the flow zone, the relevance engine rejecting four candidate connections before approving none.

The product is invisible because it works.

---

*V5 supersedes all prior architecture versions. The ML implementation is in `AI-SYSTEMS.md`. The research basis is in `docs/research/`. Scout's full critique is at `/tmp/scout-critique.md`.*
