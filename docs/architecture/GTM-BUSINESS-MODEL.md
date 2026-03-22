# Go-To-Market, Business Model, and Growth Architecture
*The product is only worth building if it reaches students. This is how.*

---

## The Retention Problem Reframe

Every education app benchmarks Day-30 retention. The industry average is 2–3%. Duolingo, the best in class, hits 55% Day-1 and then falls off a cliff. Apps that set out to beat Duolingo's retention benchmark are solving the wrong problem.

This product is not trying to build a daily habit. It is trying to be indispensable during exam cycles.

**The academic calendar is the retention mechanism.** A student uploads their biology notes 10 days before the midterm. They use the product intensively for 10 days — the predicted score climbs from 62% to 81%. They ace the exam. Two months later, finals arrive. They're back.

The correct retention metric for this product is not Day-30. It is **semester-over-semester return rate** and **exams-per-active-user**. A student who uses the product for 3 exams per semester, across 4 semesters, is an excellent customer regardless of what their Day-30 number looks like.

This reframe has a second implication: the monetization model shouldn't be daily-habit-driven either. It should be exam-cycle-driven.

---

## Business Model

### Tier structure

**Free tier (permanent):**
- 1 upload per month
- Lessons up to 30 minutes total session time per day
- Predicted exam score (core hook — must be visible to free users or the viral mechanism doesn't work)
- Basic knowledge map
- 3-day review window (not full successive relearning schedule)

The free tier is real and useful. A student can genuinely prepare for one exam with it. But it runs out before they're done — after 30 minutes of session time, they've taught and reviewed maybe 12 concepts, and there are 35 in the upload. The upgrade moment is when the score number has climbed to 74% and they can see 8 more concepts they haven't studied yet.

**Exam mode ($4.99/exam cycle, 30 days):**
- Unlimited uploads and session time for 30 days
- Full successive relearning schedule
- Voice mode (Study Walk)
- Metacognitive calibration report
- Detailed forgetting projections
- Priority support

Exam mode is priced like a Netflix rental, not a subscription. $4.99 for the month before an exam. The framing: this is what a tutoring hour costs, except it's available at 2am and knows your specific slides.

**Semester plan ($14.99/semester, ~4 months):**
- Everything in Exam mode, for the full semester
- Multiple simultaneous uploads (all courses at once)
- Cross-course concept transfer (if you studied protein folding in biochem, cell bio inherits that mastery)
- Historical performance analytics

**Rationale for this structure:**
- Students don't want annual subscriptions for study tools. They think in semesters.
- Exam mode pricing removes the willingness-to-pay friction by anchoring to a concrete, high-stakes event
- Semester plan LTV is $30/year (2 semesters) per student — achievable without institutional revenue
- A student who gets from 64% to 84% on one exam has gotten $40+ of tutoring value from a $4.99 purchase

### Revenue model math

At 10,000 paying students:
- Assume 60% Exam mode ($4.99) × 2 exams/semester = $59,880
- Assume 40% Semester plan ($14.99) × 2 semesters = $119,920
- Total: ~$180K ARR

At 50,000 paying students: ~$900K ARR. At 200,000: ~$3.6M ARR.

The institutional track (Canvas LTI, university licensing) opens at Series A when outcome data exists. A department licensing the tool for a course is $2-5/student/course, but requires procurement cycles, IRB approval for data sharing, and administrative champions. It's not a first-year motion.

---

## Go-To-Market: Phase 1 (Months 1–6)

### The launch cohort: UCLA

Quinn is a UCLA student. This is not a coincidence — it's a distribution advantage.

**Why UCLA:**
- 47,000 undergraduates
- Heavy STEM enrollment (biology, chemistry, neuroscience, economics, psychology — all high-exam-pressure, fact-dense subjects)
- Personal network for initial outreach
- IRB access through UCLA's research infrastructure for eventual validation study

**The launch playbook:**

Step 1: Find 50 students in one course (Bio 7A, Chem 14A, or Psych 120B — intro STEM with large enrollment and multiple exams per semester). Offer free semester-plan access in exchange for feedback and consent to track their exam outcomes. This is not a research study yet — just a product pilot.

Step 2: The 50 students use the product for the midterm. Measure three things:
- Day-3 return rate (came back after first session)
- Predicted score accuracy (how close was it to actual grade)
- Qualitative: did it feel different from using ChatGPT?

Step 3: If Day-3 return rate > 40% and at least one student says "this is the first thing I've used that actually made me feel like I remembered it" — launch publicly within UCLA.

**The growth loop:**

Student A uploads bio notes → studies → score goes from 58% to 79% → tells Student B "it told me I was going to get a 79 and I got an 81" → Student B uploads before their next exam → repeat.

Word of mouth in university dorms is underrated. The predicted score is the word-of-mouth hook: it's specific, it's dramatic, it's verifiable. "I got exactly the grade it predicted" is a story worth telling.

**Reddit:**

r/UCLA, r/premed, r/learnprogramming, r/biology. Not promotional posts — genuine presence. If the product works, students will post about it. Seed one authentic "I used this for my bio midterm and went from a 68 to an 85, here's what it did differently" post and let it run.

The target: 500 active users at UCLA within 60 days of launch.

### Phase 2: University expansion (Months 6–18)

**The ambassador model:**

At each new university, find 3–5 students who care about grades and have social reach (pre-med students, study group leaders, CS students who will nerd out about the technology). Give them free semester plans. They post naturally if the product works.

Target universities: UC Berkeley, USC, Michigan, Georgetown, NYU. Large enrollment, high exam pressure, research-oriented for eventual IRB study.

**The Reddit flywheel:**

r/premed: 500K members. r/college: 500K members. r/MCAT: 200K members. r/learnprogramming: 4M members. These communities share study resources obsessively. One product that demonstrably moves grades belongs in all of them.

The strategy: ship fast, get the outcome data, then let the outcomes do the marketing. "App predicted I'd get a B-, I studied with it for 8 days, got a B+. Here's the breakdown" is more powerful than any ad.

### Phase 3: The Validation Study (Months 12–18)

This is the most important long-term GTM move. Not for growth. For credibility.

**The study design:**

Partner with a professor at UCLA or another university. IRB-approved. One section of a course (control, no product). One section (treatment, product freely available). Measure exam performance at midterm and final. Consent students to data sharing.

Timeline: IRB approval typically 4–8 weeks for expedited review. Study runs one semester. Analysis and write-up: 2 months. Total: 8–10 months from IRB submission to published preprint.

**Why this is worth doing before it seems necessary:**

The Bastani PNAS study (GPT harms learning) was done at a Turkish high school. It was published in a top journal and immediately became the proof point that "AI tutoring needs guardrails." That paper is cited in every serious edtech conversation now.

A study showing this product improves delayed exam performance — real exam, real students, randomized control — would be the first published evidence that a consumer AI learning tool actually works. That paper is the Series A pitch. It's also the proof point that every competitor lacks.

---

## The Canvas LTI Play (Phase 3+)

Canvas has 30 million+ users in higher education. An LTI 1.3 integration makes the product available without download, without signup friction, embedded directly in the course module.

**How it works:**

1. A professor adds the tool to their Canvas course
2. Students see a "Study for this assignment" button next to each module
3. Clicking it launches the product in an iframe with: the course syllabus pre-loaded, the assignment due date pre-set, the exam format pre-configured
4. Students don't need to upload anything — the course materials are already there

This is the zero-friction entry point that changes the adoption curve from "student discovers tool" to "professor puts it in the course." One professor adoption = every student in their course.

**The LTI technical requirements:**

- LTI 1.3 (JWT-based, replaced OAuth 1.0a)
- LTI Advantage: Deep Linking (embed content), Assignment and Grades Service (optional for grade reporting), Names and Role Provisioning Service (student roster)
- Canvas developer key registration
- SSL termination + JWKS endpoint for JWT verification
- Iframe sandboxing compliance

This is a 4-6 week engineering effort. It is not a Phase 1 move — it requires institutional relationships and procurement conversations that take months. It belongs in the roadmap as a Year 2 distribution channel.

---

## Privacy and Compliance Architecture

This section is not optional. Building without it creates liability that can kill the company.

### FERPA

FERPA (Family Educational Rights and Privacy Act) governs educational records. For college students (18+), FERPA gives *students* (not parents) rights over their own records.

Key requirements:
- If the product integrates with a school's LMS (Canvas LTI), it operates as a "school official" under FERPA. This means: data can only be used for the educational purpose for which it was shared; it cannot be sold; it cannot be used for marketing
- B2C (student signs up directly): FERPA does not apply. This is the clean path for Phase 1.
- The transition from B2C to B2I (institution) requires explicit data processing agreements with each institution

**Architecture decision:** Phase 1 is purely B2C. Students upload their own materials. No school data flows through the system. FERPA exposure is zero. This is the right call — it keeps the architecture simple and the legal exposure minimal until institutional revenue is worth the compliance investment.

### Data minimization by design

```python
# What we collect and why
STUDENT_DATA_POLICY = {
    "interaction_logs": {
        "stored": True,
        "purpose": "Model training and personalization",
        "retention": "2 years from last active session",
        "anonymized": True,           # student_id is UUID, not PII
        "pii_included": False,
    },
    "uploaded_documents": {
        "stored": True,
        "purpose": "Concept extraction; serving lesson content",
        "retention": "Delete on student request; 90 days after account closure",
        "pii_included": "Potentially (if student includes personal info in notes)",
        "encrypted_at_rest": True,
    },
    "exam_scores": {
        "stored": "Only if student voluntarily reports",
        "purpose": "Calibrating exam readiness model",
        "anonymized": True,
        "linked_to_pii": False,       # Linked to UUID, not name/email
    },
    "email_address": {
        "stored": True,
        "purpose": "Auth, product updates",
        "never_sold": True,
        "retention": "Until account deletion",
    }
}
```

**What we explicitly do not collect:**
- Name (email only for auth; display name optional)
- University affiliation (optional; used for anonymous course clustering)
- GPA or transcript data
- Behavioral data beyond product interactions

**Data processing agreement template:** Required before any institutional integration. Standard DPA covering: controller vs. processor roles, subprocessor list (Claude API, cloud infrastructure), breach notification procedures, student data deletion workflow.

### COPPA

COPPA applies to children under 13. College students are excluded by definition. If the product ever extends to K-12 (it shouldn't in Phase 1-3), this becomes a significant compliance burden. For now: Terms of Service requires users to be 13+, with 18+ recommended; age gate at signup.

### SOC 2 Type II

Not required at launch. Required before any enterprise/institutional contract above ~$50K. Timeline: roughly 6 months of documented security practices + audit. Start the documentation discipline now (access logs, change management, incident response playbook) so the audit isn't a scramble at Series A.

---

## The Answer Evaluation Problem

This is the most technically underspecified component across V1, V2, and V3: how do you evaluate a student's free-recall answer?

Multiple choice is trivial — answer matches one of four options. But the product promises retrieval practice including short answer and free recall. "Explain why glycolysis produces only 2 net ATP" requires evaluating a student's written response.

### The rubric evaluation pipeline

For every free-recall or short-answer checkpoint question, the system generates a rubric at question creation time:

```python
@dataclass
class AnswerRubric:
    concept_id: UUID
    question: str
    required_elements: List[str]     # Must appear for full credit
    partial_credit_elements: List[str]  # Mention earns partial
    misconception_markers: List[str] # If present → wrong despite correct-sounding answer
    acceptable_variations: List[str] # Alternative phrasings that are correct
```

At evaluation time:
```python
def evaluate_free_recall(student_answer: str, rubric: AnswerRubric) -> EvalResult:
    # Step 1: Semantic similarity check against required elements
    answer_embedding = embed(student_answer)
    element_scores = [
        cosine_similarity(answer_embedding, embed(element))
        for element in rubric.required_elements
    ]
    
    # Step 2: Misconception detection
    misconception_present = any(
        cosine_similarity(answer_embedding, embed(marker)) > 0.85
        for marker in rubric.misconception_markers
    )
    
    # Step 3: For borderline cases (score 0.55-0.80), LLM evaluation
    if 0.55 < mean(element_scores) < 0.80 or misconception_present:
        llm_verdict = evaluate_with_llm(student_answer, rubric)
        return llm_verdict
    
    # Step 4: Clear cases — no LLM needed
    if mean(element_scores) > 0.80 and not misconception_present:
        return EvalResult(correct=True, partial=False)
    if mean(element_scores) < 0.45:
        return EvalResult(correct=False, partial=False)
    
    return EvalResult(
        correct=mean(element_scores) > 0.65,
        partial=0.50 < mean(element_scores) <= 0.65,
        confidence=mean(element_scores)
    )
```

The LLM evaluation prompt for borderline cases:
```
You are evaluating a student's answer to this question:
"{question}"

The key concepts required for a correct answer are:
{required_elements}

The student's answer is: "{student_answer}"

Does the student's answer demonstrate understanding of the key concepts?
Consider: correct in substance even if imperfect in phrasing = correct.
Missing key mechanism or substituting a misconception = incorrect.

Return JSON: {"correct": bool, "partial": bool, "missing": [list], "feedback": "one sentence"}
```

LLM is only called for ~20-30% of free recall answers (borderline cases). Clear correct and clearly wrong are handled by embedding similarity alone. This keeps evaluation latency low (embedding similarity: <50ms; LLM call for borderlines: +800ms).

**The feedback loop:** When a student's answer is evaluated as incorrect, they receive specific feedback: not "wrong" but "Your answer mentioned glucose splitting, which is right. But it didn't address where glycolysis happens (cytoplasm, not mitochondria), which is what this question is testing." This is generated by the LLM verdict with the `feedback` field.

---

## Voice Mode: Study Walk — Full Architecture

The premise: retrieval practice for AirPods. Between classes, on a walk, in the gym. Hands-free, eyes-free study that enforces the same retrieval structure as the screen experience.

**Why this matters beyond convenience:**

The research on motor learning and interleaved activity suggests that low-effort physical activity (walking) paired with cognitive tasks can improve encoding — the dual-task is easier than demanding exercise and the mild arousal from movement aids consolidation. More practically: if a student can do 15 minutes of active retrieval practice while walking to class instead of passively scrolling, that is a genuine behavior change that requires no willpower.

### Technical architecture

```
Student activates Study Walk (tap on phone) →
  Scheduling engine: identify top 5 concepts due for review or in ZPD
  → Generate audio question sequence

Audio session flow:
  1. TTS reads question aloud (ElevenLabs, <250ms latency)
  2. Silence window: 8 seconds for student to formulate answer
  3. Student speaks answer
  4. Deepgram ASR transcribes in real-time (<300ms)
  5. Answer evaluation (embedding similarity + LLM for borderline)
  6. TTS reads verdict + correct answer
  7. Confidence rating via tap: two taps = confident, one tap = unsure
  8. Next question

Session length: 5-15 minutes (student-configurable)
```

**The spoken answer evaluation challenge:**

Spoken free-recall answers are harder to evaluate than written ones:
- Higher noise in transcription (ASR accuracy ~80-90% in real environments)
- Students speak in fragments and half-sentences more than they write
- Domain-specific terminology (NADH, chemiosmosis, phosphorylation) has high ASR error rates

Mitigations:
- Deepgram's domain-specific model fine-tuning: submit a vocabulary list of course-specific terms (extracted from the knowledge graph) to improve ASR accuracy on those terms
- Evaluation is more lenient on spoken answers: partial credit threshold is lower; the system accepts semantic equivalence rather than exact element matching
- Option: student can also tap "mark correct" or "mark incorrect" themselves as a metacognitive override — they know if they really knew the answer

**The question format for voice:**

Voice questions must be shorter and more binary-evaluable than written questions. Not "explain the mechanism of oxidative phosphorylation" — too long to evaluate from speech. Instead: "What's the net ATP output of glycolysis?" or "Name the location of glycolysis in the cell." These are fact-level questions, not mechanism questions. The mechanism questions stay on the screen experience.

Voice mode is retrieval practice for facts and definitions. Screen mode handles mechanisms, processes, and application questions. The two modes complement each other.

---

## The IRB Study: Design and Timeline

This is the most important long-term move in the GTM playbook. It gets its own section because it determines the Series A story.

### Study design

**Type:** Randomized controlled trial, crossover design (minimizes ethical concerns about withholding a potentially beneficial treatment)

**Population:** Students in an intro STEM course at UCLA (target: Bio 7A — 400+ students per quarter, multiple TAs, high exam frequency)

**Randomization:** Students assigned to Group A (treatment first) or Group B (control first) at course start. At midterm: Group A uses the product, Group B uses "whatever you normally do." After midterm scores collected, groups cross over: Group B gets access for the final.

**Crossover advantage:** Every student gets the treatment eventually. This makes faculty approval and IRB approval easier.

**Primary outcomes:**
1. Exam score (midterm/final grade, collected from registrar with student consent)
2. Predicted vs. actual score accuracy (internal product metric)

**Secondary outcomes:**
1. Day-3 session return rate (product metric)
2. Sessions completed per exam cycle (product metric)
3. Student satisfaction (single-item Likert after exam)

**Sample size calculation:**

Assuming conservative effect size d = 0.25 (below published ITS effects which inflate toward d = 0.35), power = 0.80, α = 0.05: need ~128 students per group. Bio 7A enrollment of 400 is sufficient.

**Timeline:**
- Month 6: Submit IRB application (expedited review, minimal risk)
- Month 7-8: IRB approval, faculty recruitment
- Month 9-12 (one quarter): Data collection
- Month 13-14: Analysis
- Month 15: Preprint submission (arXiv + EdArXiv)
- Month 16-18: Journal submission, peer review

**What a positive result looks like:**

If Group A scores 0.25 SD higher than Group B on the midterm, that's roughly half a letter grade. That is: the first published RCT evidence that a consumer AI study tool improves delayed exam performance. No competitor has this. It becomes the product's flagship claim and the moat that matters most.

**What a null result means:**

The product doesn't work as claimed, at least not at the measured effect size. This is actually useful information. It identifies what to fix before spending more on growth. A null result published honestly is still credibility — it demonstrates that the company runs real studies rather than just making claims.

---

## The Business Case in One Page

**The market:** 20 million college students in the US. 150 million globally. 59-86% already use AI to study. They're getting worse. There is no product that provably improves exam scores.

**The product:** An AI learning engine that works because it enforces the cognitive science that passive AI use bypasses. The student model is the teacher. The LLM is just its voice.

**The distribution:** Student-to-student word of mouth, anchored to a dramatic and verifiable outcome claim ("it predicted my grade"). University-by-university expansion. Canvas LTI at institutions when the outcome data exists.

**The moat:** Behavioral data that compounds. Every student makes the explanation library better. Every exam score report makes the prediction model more accurate. Neither can be replicated without the students.

**The business model:** Exam-cycle pricing ($4.99/exam month). Semester plan ($14.99). No annual subscription trap. Revenue is aligned with student use intensity — the product makes money when students use it intensively, which only happens when it's working.

**The raise:** Pre-seed or seed at university-scale traction (~500-1000 paying students, Day-3 return rate > 35%, at least one quote along the lines of "it told me I was going to get a C and I got a B+"). Series A on the IRB study result.

**The risk:** Students won't tolerate being forced to work harder than ChatGPT. The predicted exam score is the only behavioral answer to this. If the number doesn't move them, nothing else will.

---

*The architecture is complete. The research is done. The business model is clear. The only thing left is to build it, validate it, and let the outcomes speak.*
