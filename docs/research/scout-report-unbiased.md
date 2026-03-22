# What Does the Best Research Say About How People Learn, and What Role Can AI Play?
## A Balanced Evidence Review — March 2026

---

## PART I: The Learning Science Foundations

### 1.1 What the Evidence Robustly Supports

**Retrieval Practice (Testing Effect)**
The single best-supported finding in modern cognitive psychology. Two major meta-analyses:
- Rowland (2014), Psychological Bulletin: meta-analytic g = 0.50 across studies comparing retrieval practice to restudy.
- Adesope et al. (2017): 217 studies, laboratory and classroom, consistent positive results.

**Caveats:** A 2025 ScienceDirect paper found retrieval practice is effective for memory but "its effects on transfer are less clear." A 2024 STEM review found spaced retrieval in some intro courses showed "no clear-cut benefit" — implementation quality matters enormously.

**Spaced Practice (Distributed Learning)**
Well-replicated. A 2024 meta-analysis confirms the effect translates to real classrooms — but with smaller effect sizes than lab studies suggest.

**Critical caveat:** Spacing effects are strongest for declarative knowledge (facts, vocabulary). Application to complex problem-solving, creative thinking, or conceptual understanding is weaker and less consistent.

**Cognitive Load Theory (Sweller, 1988)**
Working memory is limited (~4 chunks at a time). Instructional designs that overload working memory impede learning. Has NOT been substantially challenged by the replication crisis.

**Desirable Difficulties (Bjork)**
Robustly supported, but carries a practical problem: students and teachers interpret worse short-term performance as evidence the approach isn't working, leading to abandonment.

### 1.2 What Has NOT Held Up

**Learning Styles (VARK):** No rigorous empirical support. Pashler et al. (2008) set the bar. No properly controlled study has demonstrated the meshing hypothesis. A 2020 Frontiers in Education review confirmed this.

**Growth Mindset Interventions:** Sisk et al. (2019) found weak and inconsistent effects on academic achievement. Effect sizes average d ≈ 0.10–0.20 — small. Real but easily swamped by other factors.

---

## PART II: The Replication Crisis in Educational Psychology

### The Core Problem: Publication Bias Has Inflated Everything

Ropovik, Adamkovic & Greger (2021, PLOS ONE):
> "In education, field-wide evidence synthesis employing 800 meta-analyses leads to the conclusion that almost everything works, with an average effect size of d = .40. At the same time, pre-registered large-scale RCTs yield an average effect size of just d = .06."

**That is a 6-7x inflation due to publication bias.**

Johns Hopkins found insider (developer-funded) education research shows 70% more benefits than independent research.

### What Survived Scrutiny
- Retrieval practice: survives, effect sizes shrink under rigorous conditions
- Spacing effects: survives at smaller effect sizes in classrooms than labs
- Worked example / explicit instruction for novices: survives
- Formative corrective feedback: survives robustly
- Cognitive load theory: broadly survives

### What Has Weakened or Failed
- Discovery learning superiority: failed vs. explicit instruction for novices
- Many motivational interventions: weak and inconsistent
- Growth mindset at scale: much smaller than claimed

---

## PART III: Intelligent Tutoring Systems — The Evidence Record

### Ma et al. (2014, Journal of Educational Psychology) — The Foundational Meta-Analysis
107 effect sizes, 14,321 participants:
- ITS vs. non-adaptive instruction: g = +0.35 (moderate positive)
- ITS vs. individualized human tutoring: g = -0.11 (NO significant difference)
- ITS vs. small-group instruction: g = +0.05 (trivially small)

**ITS looks good against traditional large-class instruction. It does NOT outperform individual human tutoring.**

### Carnegie Learning Cognitive Tutor — RAND Study (2014)
Most rigorous large-scale RCT of an ITS:
- Year 1: No effect on algebra proficiency
- Year 2: Modest positive effects, only for some students
- What works in controlled conditions does not automatically work at population scale.

### ALEKS Meta-Analysis (2021)
56 effect sizes, 9,238 students:
- ALEKS vs. traditional instruction: g = 0.05 (essentially zero)
- ALEKS as supplement to traditional: g = 0.43 (meaningful)

**Adaptive software is not better than good teaching. It adds value alongside good teaching.**

### Knewton — What Happened
Raised $157M. Promised "Netflix for education." Failed because:
- Adaptive algorithms only as good as underlying content
- Overpromised, outcome data never matched marketing
- Algorithm assumed more data than real students generate in a semester
- Competitors replicated the framework without the costs

### Harvard/MIT AI Tutoring RCT (Kestin et al., 2025, Scientific Reports)
194 undergraduate physics students, crossover design:
- AI tutor students scored median 4.5 vs. 3.5 for control
- Effect size: 0.73–1.3 SD (large)
- Students "learned more than twice as much in less time"

**Caveats:** AI tutor was custom-designed around evidence-based principles (not ChatGPT out of the box). One-session study, two topics. Elite institution students. 1:1 vs. group dynamic may account for much of the effect.

### GPT Without Guardrails HARMS Learning (Bastani et al., PNAS, 2025)
**THE MOST IMPORTANT FINDING FOR THIS STARTUP.**

Large-scale RCT at a Turkish high school:
- GPT Base (unrestricted GPT-4): students performed WORSE on unassisted exams than control
- Students used AI as a crutch — got answers without learning
- GPT frequently hallucinated incorrect answers
- GPT Tutor (with guardrails, scaffolded help) mitigated negative effects but did not produce clear positive effects either

**Access to AI that gives answers harms learning. The design of the interaction determines everything.**

---

## PART IV: The Engagement vs. Learning Problem

### The Seductive Details Effect
Mayer (2005), meta-analysis in Educational Research Review (2012): Adding interesting but educationally irrelevant information to lessons can IMPAIR learning by drawing attention away from core content. Students report finding material more interesting while actually retaining less.

**This directly challenges "make school interesting" as a strategy. If "interesting" means adding engaging but tangential content, it actively hurts learning.**

---

## REASONS THIS MIGHT NOT WORK

1. **The engagement trap.** "Making learning interesting" can mean adding seductive details that impair actual learning. If the product optimises for engagement metrics (time spent, sessions opened, satisfaction ratings), it may actively harm learning outcomes. Duolingo is the cautionary tale: high engagement, low completion, questionable fluency outcomes.

2. **The crutch effect (Bastani et al., 2025).** If the AI makes learning feel easier, students may learn less. The entire desirable difficulties literature says the opposite of "interesting and easy" is what produces learning. The product must make students uncomfortable — and students may not pay for discomfort.

3. **Effect size inflation.** Most learning science findings have been inflated 6-7x by publication bias. Building a product on d = 0.40 effects that are actually d = 0.06 in the real world is dangerous.

4. **ITS don't outperform human tutors.** The ceiling for even the best adaptive systems is roughly equivalent to individual human tutoring (Ma et al., 2014). That's good — but it means the product is competing with office hours, TAs, and study groups, not just bad lectures.

5. **Knewton's lesson.** The most well-funded adaptive learning startup in history failed not because the science was wrong, but because: (a) content quality matters more than algorithms, (b) students don't generate enough data in a semester to feed adaptive models, and (c) the promises outran the evidence.

6. **Personalisation has limits.** Learning styles are debunked. The specific form of personalisation that DOES work (adaptive difficulty, spaced scheduling) is narrow. "Personalised to your life" is a different claim with no direct evidence base — the self-reference effect exists in labs but has not been tested in semester-long educational interventions.

7. **College students won't pay.** The market for student-facing edtech is brutal. Quizlet is free. ChatGPT is free. Students pirate textbooks. A premium subscription tool needs to be 10x better than free alternatives, and the bar keeps rising as base models improve.

8. **The "just upload slides" problem.** Slide quality varies enormously. Many professors' slides are incomplete, disorganised, or actively misleading. A system that builds a knowledge graph from bad slides will teach bad content confidently. Garbage in, garbage out — and the student won't know the difference.

9. **Measurement is hard.** How do you prove the product works? GPA improvement? Exam scores? Self-reported learning? Each has massive confounds. Without credible outcome data, the product is marketing claims on top of learning science citations — exactly what Knewton did.

10. **The competition is improving fast.** Google, OpenAI, and Anthropic are all investing in educational applications. NotebookLM will likely add retrieval practice features. Khanmigo will likely open to custom content. The window for a startup may be narrow before incumbents add these features to existing platforms with massive distribution.
