# The Learning Engine — Architecture V4
*Built around knowing the student. Not performing that it knows the student.*

---

## What Changed Between V3 and V4

V3 was architecturally correct about the ML. It was wrong about the product.

The student model in V3 tracked mastery. That's a probability vector — it knows Quinn can recall glycolysis with 72% confidence. What it doesn't know is that Quinn runs track, that he understood oxidative phosphorylation immediately when it was framed as energy for sprinting, that he glazes over at walls of text but stays engaged with diagrams, that he has 30 minutes before his next class, that he's studying this for a multiple-choice midterm and not for genuine long-term understanding.

Those facts produce a completely different lesson than the mastery vector does. And the lesson that uses those facts produces a different outcome — not because it's more "personalized" in the marketing sense, but because of a precise mechanism: the **self-reference effect**.

When information is encoded in relation to the self — connected to something the student knows, does, or cares about — it is remembered significantly better than information encoded abstractly. A 2025 Psychonomic Bulletin & Review study showed self-reference promotes vocabulary retention specifically. Symons & Johnson (1997) meta-analysis: d=1.11 for self-referential encoding vs. neutral. This is not learning-styles mythology. It is a structural feature of how human memory works: information with more retrieval hooks — more connections to existing knowledge — is easier to retrieve.

But Quinn named the constraint precisely: **forced context is not self-referential encoding. It's a seductive detail.** Mayer's coherence principle: adding interesting but non-essential material to a lesson harms learning by increasing cognitive load, disrupting the mental model being built, and activating irrelevant prior knowledge schemas. The paragraph saying "this is like track!" when the connection is thin and manufactured is not a memory hook. It's noise that increases extraneous load and reduces the signal.

The difference: genuine relevance versus performed relevance. The architecture must be built to produce the first and never produce the second.

V4 is built around that distinction.

---

## The Central Reframe: What "Knowing the Student" Actually Means

The student model in V3 answered: *what does this student know?*

The student model in V4 answers: *who is this student, and what is the best way to teach this specific concept to this specific person right now?*

That's a much harder question. It requires:

1. **A student intelligence file** — not a mastery vector, but a structured understanding of the person
2. **A relevance evaluator** — a system that can distinguish genuine connection from forced association, and invoke context only when the connection is structural and real
3. **A format reasoning system** — that knows whether this concept needs a diagram, a video segment, a worked example, or plain prose — not based on learning-style mythology, but based on what the concept actually is and what has worked before
4. **A minimal intervention principle** — that adds complexity only when complexity adds value, and defaults to the simplest thing that produces understanding

The system's core discipline is restraint. The best lesson is often the leanest one.

---

## Layer 0: The Student Intelligence File

This is the file the system maintains on each student across all sessions, all uploads, all interactions. It is not a mastery vector. It is a structured portrait.

```python
@dataclass
class StudentIntelligenceFile:
    student_id: UUID

    # ── Academic context ──────────────────────────────────────
    year: Optional[str]                  # "sophomore", "junior", etc.
    major: Optional[str]                 # "Cognitive Science"
    institution: Optional[str]           # "UCLA"
    active_courses: List[CourseContext]  # What they're currently studying
    prior_uploads: List[UploadRecord]    # All past material + outcomes

    # ── Life context (used sparingly, only when structurally relevant) ─
    stated_interests: List[str]          # ["basketball", "music production", "startups"]
    inferred_interests: List[str]        # From language patterns, upload topics
    background_knowledge_domains: List[str]  # Strong domains from prior uploads
    # e.g. ["AP Chem — strong", "statistics — weak", "biology — intermediate"]

    # ── How they learn ───────────────────────────────────────
    # NOT learning style in the VARK sense (debunked).
    # What actually works for THIS student on THESE concept types.
    
    format_response_history: Dict[ContentType, FormatPerformance]
    # e.g. {ContentType.MECHANISM: FormatPerformance(
    #           diagram=0.81_checkpoint_accuracy,
    #           prose=0.62_checkpoint_accuracy,
    #           analogy=0.74_checkpoint_accuracy
    #       )}
    
    analogy_hit_rate: float              # Proportion of analogies that clicked
    analogy_domains: List[str]           # What analogy domains have worked
    # e.g. ["sports", "economics", "cooking"] — NOT used unless checkpoint data backs it
    
    engagement_dropoff_pattern: EngagementPattern
    # e.g. {session_min_10: high, session_min_20: moderate, session_min_30: low}
    # Used to shape session length and pacing, not content
    
    # ── Current context ─────────────────────────────────────
    time_pressure: Optional[TimePressure]  # "30 minutes", "exam tomorrow", "3 days"
    depth_goal: DepthGoal                  # "pass the test" vs "actually understand this"
    exam_format: Optional[ExamFormat]
    
    # ── What hasn't worked ──────────────────────────────────
    failed_explanation_types: List[ExplanationAttempt]
    # Specific explanations that produced low checkpoint accuracy for this student
    # Never repeat these. Use the inverse as a signal.
    
    # ── Prior knowledge cross-course ────────────────────────
    cross_course_knowledge: Dict[str, float]
    # concept_canonical_id → estimated_mastery from prior uploads
    # Feeds directly into cold start of new uploads
```

**What this is not:**

It is not a profile that says "Quinn is a visual learner." Learning styles are a debunked neuromyth — there is no evidence that matching presentation format to a student's stated preference produces better outcomes (Pashler et al., 2008; PMC11270031, 2024 meta-analysis). The format_response_history tracks empirical format effectiveness per concept type, not a fixed preference.

It is not a profile that drives every lesson to force a personal connection. Stated interests are used only when a structural relevance exists — when the real-world application is not manufactured but genuine and material to understanding.

**How it gets built:**

- At onboarding: 5 questions. Major, year, time pressure, why they're studying this, one thing they know well outside school.
- Each session: format performance is updated from checkpoint accuracy per content type. Analogy hit rate is updated (did the analogy produce correct retrieval or wrong retrieval). Engagement dropoff is inferred from response time patterns in the second half of a session.
- Each upload: prior knowledge cross-course is updated from the concept linking layer.
- Never: the student is never asked to take a learning styles inventory or fill out a preferences form. Everything is inferred from behavior.

---

## Layer 1: The Relevance Engine

This is the new layer. It solves the problem Quinn named: distinguishing genuine connection from forced association.

### The test for genuine relevance

Before any contextual connection is included in a lesson, it must pass three gates:

**Gate 1: Structural isomorphism**
Is the real-world concept structurally similar to the academic concept — same mechanism, not just similar surface features?

"Glycolysis is like a factory assembly line that breaks down materials into usable parts" — Structural. Both involve sequential transformation of inputs into outputs with intermediate products.

"Glycolysis is like running a race" — Surface-level. Running uses ATP, but the mechanism of glycolysis is not structurally similar to the act of running. This is a seductive detail. Do not use it.

"Glycolysis is the energy accounting before the real work" — Contextual but not structural. May help motivate but does not aid understanding of the mechanism. Use only to frame relevance briefly, never as the explanation itself.

**Gate 2: Prior knowledge exists**
Does the student demonstrably know the domain being used for the analogy?

If the student listed "basketball" as an interest but has never uploaded anything biology-adjacent, there is no evidence they understand the metabolic demands of basketball in enough detail for an analogy to map correctly. Use only domains where the student has demonstrated knowledge — either through prior uploads, through their own language, or through explicit background knowledge questions.

**Gate 3: It fits without padding**
Does the connection require extra prose to set up? If the analogy takes 40 words to establish before it pays off, the cognitive overhead likely exceeds the encoding benefit.

The rule: a genuine connection can be stated in one sentence and should make the student think "oh, I get it" rather than "I see what you're doing there."

### Implementation

```python
class RelevanceEngine:
    def evaluate_connection(
        self,
        concept: ConceptNode,
        proposed_connection: str,
        connection_domain: str,
        student: StudentIntelligenceFile
    ) -> RelevanceVerdict:
        
        # Gate 1: Structural isomorphism check
        concept_mechanism = concept.mechanism_summary
        connection_mechanism = self.extract_mechanism(proposed_connection)
        structural_similarity = self.mechanism_similarity(
            concept_mechanism, connection_mechanism
        )
        # Uses LLM with explicit structural mapping prompt:
        # "List the functional components of X. List the functional components of Y.
        #  Do they map onto each other in a way that preserves relationships?
        #  Not surface features — structural relationships."
        if structural_similarity < 0.65:
            return RelevanceVerdict.REJECT_NOT_STRUCTURAL
        
        # Gate 2: Prior knowledge check
        domain_familiarity = student.background_knowledge_domains.get(
            connection_domain, 0.0
        )
        analogy_domain_history = connection_domain in student.analogy_domains
        if domain_familiarity < 0.4 and not analogy_domain_history:
            return RelevanceVerdict.REJECT_NO_PRIOR_KNOWLEDGE
        
        # Gate 3: Conciseness check
        setup_words = self.estimate_setup_length(proposed_connection)
        if setup_words > 35:
            return RelevanceVerdict.REJECT_TOO_MUCH_OVERHEAD
        
        # Check historical performance of this connection type
        if self.similar_connection_failed_before(concept.id, connection_domain, student):
            return RelevanceVerdict.REJECT_HISTORICALLY_INEFFECTIVE
        
        return RelevanceVerdict.APPROVED
```

**The default is no connection.** The relevance engine is called only when there is a candidate connection to evaluate. If the scheduling engine determines that the concept should be taught with plain explanation — because the student's checkpoint accuracy for prose on mechanism concepts is 0.77 — it sends the content pipeline a spec with no connection requested. The relevance engine is never used to manufacture a connection because one should exist.

---

## Layer 2: Format Reasoning

This is also new. The content pipeline in V3 produced text cards. The format reasoning layer decides what form the teaching should take before the content pipeline is called.

**The research:**

Learning styles (VARK) are a myth — matching presentation modality to stated preference does not produce better outcomes. What does produce better outcomes:

- Words + pictures > words alone for complex mechanisms (Mayer multimedia principle, d=1.67 for well-designed cases)
- Video is powerful for processes, demonstrations, dynamic systems — not for facts or definitions
- Diagram + brief prose > diagram alone for spatial relationships
- Worked example > explanation for procedural content when the student is a novice
- Plain prose is often the most efficient for definitions, principles, and edge cases

The format depends on the concept type and the student's track record, not on a preference label.

```python
class FormatReasoner:
    def select_format(
        self,
        concept: ConceptNode,
        student: StudentIntelligenceFile,
        time_budget_min: float,
        scaffold_level: float
    ) -> FormatSpec:
        
        # Base format by concept type
        concept_format_map = {
            ContentType.MECHANISM: self.mechanism_format(concept, student),
            ContentType.DEFINITION: FormatChoice.PROSE,
            ContentType.FACT: FormatChoice.PROSE,
            ContentType.PROCESS: self.process_format(concept, student, time_budget_min),
            ContentType.PRINCIPLE: FormatChoice.PROSE_WITH_EXAMPLE,
            ContentType.EDGE_CASE: FormatChoice.CONTRAST_PAIR,
            ContentType.DIAGRAM_CONCEPT: FormatChoice.DIAGRAM_FIRST,
        }
        
        base_format = concept_format_map[concept.content_type]
        
        # Override based on student history
        if concept.content_type in student.format_response_history:
            history = student.format_response_history[concept.content_type]
            # If prose has consistently outperformed diagram for this student
            # on this concept type, use prose even for mechanism concepts
            if history.best_format_accuracy_delta > 0.15:
                base_format = history.best_format
        
        # Time constraint override
        if time_budget_min < 5:
            # Strip everything down to minimum viable format
            # No video (requires loading time + watch time)
            # No extended diagrams — text + one key image if critical
            base_format = self.compress_format(base_format)
        
        return FormatSpec(
            primary=base_format,
            include_diagram=self.diagram_adds_value(concept, base_format),
            include_video=self.video_justified(concept, student, time_budget_min),
            include_analogy=self.analogy_justified(concept, student),
            include_worked_example=scaffold_level > 0.5
        )
    
    def video_justified(
        self, concept, student, time_budget_min
    ) -> Optional[VideoSpec]:
        # Video is only justified if:
        # 1. Concept involves a dynamic process (cell division, chemical reaction)
        # 2. Student has ≥ 8 minutes available (< 3 min video + teach + checkpoint)
        # 3. A high-quality video segment exists and has been pre-indexed
        # 4. The student's video engagement rate is > 0.6 (they actually watch)
        
        if concept.content_type != ContentType.PROCESS:
            return None
        if time_budget_min < 8:
            return None
        segment = self.video_index.find_segment(concept.id, max_duration_sec=180)
        if not segment or segment.quality_score < 0.7:
            return None
        if student.video_completion_rate < 0.6:
            return None
        
        return VideoSpec(url=segment.url, start=segment.start_sec, end=segment.end_sec)
```

**The video index:**

The system maintains a curated index of video segments from YouTube, Khan Academy, Crash Course, and other sources — timestamped and tagged to concept IDs. This is not "search YouTube for a video about glycolysis." That produces inconsistent quality. The index is built once per subject domain, curated for quality, and updated quarterly.

A concept has a video segment in the index only if:
- A qualifying video exists (educational, accurate, appropriate length)
- The segment timestamp is identified and verified
- The concept tagging is accurate

Concepts with no indexed video simply do not get video. The format reasoning layer handles this gracefully.

---

## Layer 3: The Student-Aware Content Pipeline

The content pipeline in V4 takes three inputs instead of one:

V3: `CardSpec → LLM → Card`

V4: `CardSpec + StudentIntelligenceFile + FormatSpec + RelevanceVerdict → Content Pipeline → Card`

The LLM now has full context about the student when generating content. This changes what it generates — not because it forces a connection, but because it can make intelligent decisions about framing, vocabulary, and emphasis.

### What student context actually changes in generation

**Vocabulary calibration:**
A student who lists "cognitive science" as their major gets different vocabulary on a biology concept than a pre-med student. Not different content — same accuracy — but different framing of the conceptual entry point.

**Prior knowledge leverage:**
If the student's file shows they have strong chemistry background and the concept involves a chemical reaction, the prompt tells the LLM: "This student understands chemical equilibrium well. You can assume that and build from it rather than starting from scratch."

**Exam-format alignment:**
The student's exam is multiple choice. Every checkpoint is formatted as MC with distractors that are real misconceptions. The elaboration questions focus on discrimination (what makes this different from the wrong answers) rather than recall.

**Genuine connection — when approved:**
If the relevance engine approved a connection, the LLM receives it and weaves it into the explanation at the natural moment. One sentence. It doesn't restructure the lesson around it.

**What student context does NOT change:**
- The pedagogical structure (checkpoints still required before explanation)
- The successive relearning requirement (3 correct recalls to graduate)
- The interference scheduling
- The fact-checking pass

The student intelligence file shapes the form, not the function.

### The generation prompt structure (V4)

```
SYSTEM:
You are a teaching engine. Your job is to fill this card specification exactly.
You do not decide what to teach, in what sequence, or whether to include a checkpoint.
Those decisions have already been made. You execute them.

STUDENT CONTEXT:
- Year: {year}, Major: {major}
- Time available: {time_budget_min} minutes
- Exam format: {exam_format}
- Background strengths: {background_knowledge_domains}
- What's worked before: format={best_format}, analogy_domains={analogy_domains}
- Prior knowledge on related concepts: {prior_knowledge_summary}

CONCEPT:
{concept.name} — {concept.definition}
Source material: {concept.source_text}

CARD TYPE: {card_type}
SCAFFOLD LEVEL: {scaffold_level} (0=minimal, 1=maximal)
FORMAT: {format_spec}

APPROVED CONNECTION (use only if present, one sentence max):
{approved_connection or "NONE — do not manufacture one"}

CONSTRAINTS:
{card_constraints}

Generate the card. Stay within the constraints. Do not add anything not specified.
```

The `APPROVED CONNECTION` field is `NONE` the majority of the time. The LLM does not search for connections. It only uses one if the relevance engine approved it upstream.

---

## Layer 4: The Minimal Intervention Principle

This is the governing principle of the whole system, and the one most likely to be violated in implementation.

**The principle:** Add an element to a lesson only if removing it would produce a measurably worse learning outcome. If the outcome is the same without it, remove it.

This runs counter to the instinct of every designer and every LLM. The instinct is to add — more context, more explanation, more engagement, more connection. The research says the opposite. Mayer's coherence principle (d=0.86 for concise vs. expanded versions of the same lesson) shows that adding interesting but non-essential material reliably harms learning.

**Implementation: the lesson leanness score**

After content generation, before delivery, the card is scored:

```python
def leanness_score(card: GeneratedCard, spec: CardSpec) -> float:
    # Proportion of content directly necessary for the pedagogical goal
    necessary_elements = identify_necessary_elements(card, spec.concept, spec.card_type)
    all_elements = identify_all_elements(card)
    
    element_necessity_rate = len(necessary_elements) / len(all_elements)
    
    # Word count relative to information density
    info_per_word = spec.concept.information_density / len(card.word_count)
    
    # Absence of seductive details
    seductive_detail_flag = detect_seductive_details(card, spec.concept)
    
    return element_necessity_rate * info_per_word * (0.0 if seductive_detail_flag else 1.0)
```

Cards below a leanness threshold are automatically revised — the LLM is asked to remove what isn't load-bearing. The revision prompt is explicit: "Remove any content that isn't necessary for a student to understand {concept.name}. Every sentence should either teach the concept or test retrieval of it. Nothing else."

This means the system actively trims itself. The self-reference connection that seemed relevant gets cut if it doesn't pass the leanness check. The interesting historical anecdote gets cut. The extended analogy gets cut to its first sentence if that's all that's load-bearing.

---

## Layer 5: Session Architecture

The session in V4 is not a fixed sequence of cards. It's a directed graph of learning units, where the path taken depends on the student's performance.

### Session graph structure

```
[Entry] → Assessment of prior knowledge (0-2 questions)
              ↓
    [Fast path: strong prior]       [Standard path: weak prior]
              ↓                               ↓
    [Brief activation]         [Worked example + teach]
              ↓                               ↓
    [Checkpoint]                    [Checkpoint]
         ↓        ↓                    ↓          ↓
    [Pass]      [Fail]             [Pass]       [Fail]
       ↓           ↓                  ↓             ↓
  [Elaborate]  [Reteach with    [Elaborate]   [Reteach with
                different               different format]
                format]
```

The graph has 4-8 nodes. The path is determined in real time. A student who nails the prior knowledge questions skips the full teach and goes to a more demanding checkpoint. A student who fails the checkpoint gets a reteach — different format, determined by which format they haven't seen yet for this concept.

The session graph ensures: every student does retrieval practice. The path to that retrieval practice varies.

### Time-pressure adaptation

The student said "30 minutes." The session graph knows this.

```python
def adapt_for_time_pressure(graph: SessionGraph, 
                             time_budget_min: float,
                             student: StudentIntelligenceFile) -> SessionGraph:
    
    if time_budget_min >= 45:
        return graph  # No adaptation needed
    
    elif 20 <= time_budget_min < 45:
        # Remove elaboration nodes (those are nice-to-have, not essential)
        # Compress teach nodes (use more direct explanation, less scaffolding)
        graph.remove_node_type(NodeType.ELABORATE)
        graph.compress_teach_nodes(factor=0.7)
    
    elif 10 <= time_budget_min < 20:
        # Collapse to: one activation → one checkpoint → one teach (if failed)
        # Prioritize highest-exam-weight concepts only
        graph = graph.minimal_viable_session()
    
    elif time_budget_min < 10:
        # Retrieval practice only. No teaching. 
        # If a student has 8 minutes, the most valuable use is reviewing
        # concepts at highest risk of forgetting before the exam.
        # No new material.
        graph = graph.retrieval_only_session()
    
    return graph
```

The 8-minute session looks completely different from the 45-minute session. The 8-minute session is: here are the three concepts most likely to fail you on tomorrow's exam. Get them correct once each. Come back tomorrow. That's it.

---

## Layer 6: The Student Intelligence Engine

This is the system that builds and maintains the Student Intelligence File.

### Passive inference (most of it happens here)

The student is not surveyed. The system infers from behavior:

**Format preference inference:**
After every concept, the system has: (concept_type, format_used, checkpoint_accuracy). After 20 interactions, a format_response_history exists. This is what drives format selection — not a form the student filled out.

**Engagement pattern inference:**
Response time within a session. If response times start increasing at minute 15, the student is losing engagement. The session graph can detect this and either compress or close the session. Not via a popup ("Are you still there?") — via the next card being shorter and the checkpoint being more direct.

**Analogy hit rate:**
Every time an approved connection is used, the subsequent checkpoint accuracy is tracked. Did the analogy produce correct retrieval or wrong retrieval? If an analogy domain consistently correlates with incorrect retrieval (the student is applying the analogy incorrectly), that domain is de-prioritized in future relevance evaluations.

**Prior knowledge cross-course:**
When a new upload is processed and concepts are linked to the cross-course concept graph, the student's prior mastery transfers automatically. The Student Intelligence File knows: this student had strong mastery of protein structure from their Biochemistry upload. When a Cell Biology upload introduces "protein folding," the file pre-seeds mastery at 0.65 rather than starting at 0.10.

### Active inference (minimal)

At onboarding: 5 questions. Maximum. Answers are optional.
- What are you studying for? (course name, if they want to share)
- When is the exam / deadline?
- How deep do you want to go? (pass the test / actually understand it)
- What do you already know well that might be related?
- How much time do you have right now?

That's it. Everything else is inferred.

**The ethical constraint:** The Student Intelligence File is not used to build a psychological profile of the student. Its only purpose is to produce better lessons. It is never used for targeting, advertising, or inference beyond educational context.

---

## What the Product Actually Feels Like (V4)

Quinn uploads 30 minutes of bio slides before a midterm. He has 30 minutes.

The system knows: cognitive science major, junior, UCLA. Has studied chemistry before. Track runner. Has 30 minutes. MC exam tomorrow.

The session begins. The graph detects he already knows ATP from his chemistry upload — skips the ATP teach node, opens with a quick activation question to confirm. He gets it right. Fast path.

Glycolysis: the system checks the format history. Mechanism concepts with Quinn have gotten 0.81 checkpoint accuracy on diagrams vs. 0.62 on prose. The format reasoner selects diagram-first. The relevance engine checks: is there an approved connection for glycolysis + track? It runs the structural isomorphism check — "energy investment phase before energy payoff" has structural similarity to "slow miles before the kick." Passes all three gates. One sentence, woven in at the natural moment after the diagram is established: *"The investment phase costs 2 ATP before returning 4 — think of it as the first mile that sets up the finish."* Then the checkpoint. Then the elaboration if needed.

At minute 22, response times start climbing. The session graph detects engagement dropoff, compresses the remaining nodes. Two more concepts, no elaboration, direct checkpoints. Session ends at minute 28.

Predicted score updates: 64% → 78%.

What Quinn notices: the lesson was short. It felt like it understood what he needed. The track sentence made sense — it wasn't a stretch. He didn't feel like the system was trying to make learning fun. He felt like it was fast and he understood it.

What happened underneath: the Student Intelligence File drove format selection. The relevance engine approved one connection that passed structural isomorphism. The session graph compressed on engagement dropoff. The time pressure adapter kept the session to 28 minutes. The successive relearning scheduler queued 3 concepts for review in 2 days.

No part of this was performed. The system had the data to make the right call.

---

## The Architecture Tables

### What each layer does

| Layer | What it does | When it runs |
|---|---|---|
| Student Intelligence File | Maintains the student portrait across all sessions | Updated after every interaction |
| Relevance Engine | Evaluates candidate connections; rejects forced ones | Per concept, at spec generation time |
| Format Reasoner | Selects format based on concept type + student history | Per concept, at spec generation time |
| Session Graph | Determines lesson path based on real-time performance | Per session, dynamically |
| Student-Aware Content Pipeline | Generates content using full student context | Per card |
| Minimal Intervention Validator | Strips non-essential content after generation | Per card, post-generation |
| Student Intelligence Engine | Infers format history, engagement pattern, prior knowledge | Continuously, passive inference |

### What the LLM does and doesn't decide

| Decision | Decided by |
|---|---|
| What to teach | Scheduling engine (algorithmic) |
| In what order | Session graph (deterministic, guided by student performance) |
| In what format | Format reasoner (student history + concept type) |
| Whether to include a connection | Relevance engine (structural isomorphism gate) |
| What words to use | LLM |
| How to frame for this student's background | LLM (with student context in prompt) |
| Whether to include a connection | **NOT THE LLM** — it receives the approved connection or NONE |
| Whether to add an interesting aside | **NOT THE LLM** — minimal intervention validator removes it |

---

## The Honest Assessment of What This Builds

This is not a course generator. It doesn't produce a beautiful 10-module experience with curated videos and interactive exercises.

It is a teaching system that knows the student and makes intelligent decisions about how to teach each concept to that person right now.

The intelligence lives in the restraint — in the relevance engine that rejects 70% of candidate connections, in the minimal intervention validator that strips what isn't load-bearing, in the engagement detector that compresses the session rather than powering through. The features you don't notice are the features that make it work.

The goal is that Quinn finishes a session and thinks: "That was short. I actually understood it. How did it know to use that analogy?" Not: "It really tried to personalize it."

The system that produces the first response knows the student. The system that produces the second one is performing that it knows the student.

V4 is the first architecture that's built to produce the first response.

---

*The research basis for every decision in this architecture is documented in `/docs/research/`. The AI/ML implementation is in `AI-SYSTEMS.md`. The full prerequisite graph and scheduling logic is in `ARCHITECTURE-V3.md`.*
