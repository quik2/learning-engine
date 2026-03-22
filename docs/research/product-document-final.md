# AI Learning Product — Research & Product Context

## What This Is

An AI learning system that takes your course material and teaches it to you using the methods cognitive science has proven build durable memory — methods students never use on their own because they feel harder than the ones that don't work.

You upload your slides. You tell it what you're studying for — MC midterm in 6 days, short answer final in 3 weeks, or "I just need to understand this." The product builds a curriculum: a visual, paced experience that teaches the material one concept at a time, forces you to think before it gives you anything, brings things back before you forget them, and adapts based on your goal, test format, and how you personally retain information. Not a chatbot. Not flashcards. A designed lesson you scroll through. A chatbot exists in the corner if you need it. The product is the curriculum.

This is a data system, not an AI. The LLM is commodity infrastructure — swap GPT for Claude for Gemini and nothing changes. The value is in the curriculum generation logic, the per-student learning model, the cross-student data network that learns which explanations actually produce retention, and the designed experience layer. OpenAI adding a "study mode" to ChatGPT is not the threat, same way ChatGPT isn't a threat to Narrow. They're a model company, not a vertical product company.

---

## The Core Problem

Students paste slides into ChatGPT, read a clean explanation, feel like they learned it, and remember almost none of it two days later. The research calls this the "fluency illusion" (Koriat & Bjork, 2005) — reading a clear answer feels like knowing it, but at test time you only get the question.

The key studies:
- **Barcaui (2025)**, N=120, RCT: ChatGPT group scored 57.5% vs. 68.5% traditional on a 45-day retention test. **11 points worse.**
- **Bastani et al. (2025)**, PNAS, N~1,000: Unrestricted GPT-4 users solved 48% more practice problems but scored **17% lower** on the real exam. Guardrailed Socratic GPT-4 showed null gains on unassisted exams.
- **MIT Media Lab (Kosmyna et al., 2025)**: LLM users showed 55% reduced brain connectivity. 83% couldn't recall content from essays they'd just written with AI.

59-86% of college students use AI for studying. 66% use ChatGPT specifically. They're not choosing it because it's good for learning — they're choosing it because nothing better has matched its accessibility.

---

## The Learning Science That Drives the Product

These are the mechanisms that directly shape product design decisions. Each one maps to a specific feature.

### Retrieval Practice (g = 0.50-0.61)
Being tested on material produces dramatically better retention than re-reading. Four independent meta-analyses converge (Rowland 2014, Adesope et al. 2017, Yang et al. 2021, Pan & Rickard 2018). Effect is larger with feedback (g = 0.73), grows over time (g = 0.82 at 1-6 days), and transfers to new questions (d = 0.40). **Product implication: the system asks questions, not answers them. Checkpoints are embedded in every lesson and can't be skipped.**

### Successive Relearning (68% retention at 1 month vs. 11% control)
Rawson & Dunlosky (2013, 2022): Retrieve until 3 correct in a row, come back days later, repeat. Retrieval practice + spaced repetition fused. Effect is larger than either alone. **This is the core loop. No existing product implements it.** Anki does half (spacing without retrieval-to-criterion). Quizlet does the other half (quizzing without spacing).

### Spaced Repetition (d = 0.54-0.85)
Cepeda et al. (2006): One of the most robust effects in experimental psychology. Personalized algorithms outperform generic spacing by 10 percentage points (Lindsey et al. 2014). Works for conceptual understanding, not just facts. **The motivation problem:** 72% of students judge massing as more effective than spacing (Kornell & Bjork, 2008). The system must make spacing invisible.

### Generation Effect (d = 0.40)
Self-generated information is remembered better than passively read information (Slamecka & Graf, 1978; meta-analysis of 86 studies). **Product implication: force the student to produce an answer — even a wrong one — before revealing anything.**

### Desirable Difficulties (Bjork, 1994)
Conditions that feel harder produce more durable memory: spacing > cramming, testing > re-reading, interleaving > blocking, generating > reading. Students consistently choose the easy methods because they feel more effective. **The product must override student preferences by building difficulty into the structure so they don't have to choose it.**

### Hypercorrection Effect
When a student answers with high confidence and is wrong, they correct that error MORE reliably than low-confidence errors (Metcalfe & Butterfield, 2001). But high-confidence errors return after a week unless retested (Butterfield & Metcalfe, 2006). **Product implication: confidence rating (sure/not sure) before every checkpoint. High-confidence errors get scheduled for review within 2-3 days.**

### LECTOR: Interference-Aware Scheduling
Traditional SRS treats every item as independent. LECTOR (arxiv 2508.03275, 2025) uses semantic similarity to detect interference — reviewing "mitosis" makes you more likely to confuse "meiosis." It factors this into scheduling. **Implementable from day one because the knowledge graph already has concept relationships.**

### Context-Varied Retrieval
Same concept, different question format each review (MC on day 3, fill-in-blank on day 8, application on day 15). Produces better recall than consistent presentation (Shin et al., 2025).

### Expertise Reversal (Kalyuga, 2003)
Techniques optimal for novices become harmful for experts. A static approach is wrong by definition — the system must progressively withdraw scaffolding as competence grows.

### Multimedia (Mayer's Principles)
Words + pictures > words alone (d = 1.67). Segment into learner-paced chunks (d = 0.70). Remove interesting-but-irrelevant content (d = -0.33 for seductive details). Personalize framing with "you/your" (d = 1.11). **Never add fun tangents. Personalize how relevant content is framed.**

### The Critical Evidence Gap
**No published RCT of autonomous LLM tutoring shows positive results on delayed tests.** The two studies measuring delayed outcomes (Bastani, Barcaui) both found harm from unrestricted AI. The Bastani finding is existential: AI must enforce retrieval effort or it substitutes for the cognitive work that produces learning.

---

## The Product

### Setup: Three Questions
1. What are you preparing for? (Midterm, Final, Assignment, Long-term)
2. What kind of test? (MC, Short answer, Essays, Open note, Mixed)
3. How deep? (Get the gist, Know it well, Master it)

These completely change the curriculum. MC trains recognition/discrimination. Short answer trains free recall. Open note trains application. "Get the gist" covers core concepts in ~4 minutes. "Master it" goes 20+ minutes with edge cases.

### The Core Loop
1. **Upload** — slides parsed into knowledge graph (concepts, prerequisites, Bloom's levels)
2. **Teach** — visual cards, one concept at a time, paced, conversational
3. **Checkpoint** — answer before seeing explanation; can't skip; confidence rating (sure/not sure)
4. **Retrieve to criterion** — not done until 3 correct recalls
5. **Space** — graduated concepts enter FSRS queue with interference awareness
6. **Relearn** — concepts return at optimal intervals with varied question formats
7. **Predicted exam score** — home screen number, updates in real-time

### The Predicted Exam Score
The single most important feature. It's the home screen. A number: "Predicted: 74% (C)". It drops when you don't study (forgetting curves erode mastery on un-reviewed concepts). It climbs when you do. This is the day-2 return hook, the viral mechanism ("It told me I was gonna get a 68, I studied and got an 84"), and the answer to every behavioral objection about students not wanting to do hard things.

### The Data Flywheel
**Per-student:** What they know, what they're forgetting, what explanation styles work, misconception patterns.

**Population-level:** Which explanations produce highest checkpoint accuracy (A/B tested over time), universal misconceptions, optimal concept sequencing. After 100K students study the same bio chapter, the system knows exactly which explanation of the Krebs cycle produces retention. This is the moat. It compounds with every user.

---

## Technical Architecture

### Three Hard Problems
1. **Slides to knowledge graph** — PDF/PPTX parsed (MinerU, python-pptx), LLM extracts concepts with definitions/prerequisites/Bloom's levels as structured JSON, stored in Postgres + vector DB (Chroma/Qdrant). **Riskiest piece** — LLM gets ~80% right on concept extraction, ~60% on prerequisite ordering.
2. **Knowledge tracking** — BKT (Bayesian Knowledge Tracing) for per-concept mastery probability, FSRS for spaced repetition scheduling (open-source, better than SM-2 by ~30%), LECTOR-style interference via concept embeddings.
3. **Content generation** — Scheduling engine (deterministic, no LLM) decides what to teach based on mastery map + exam date + knowledge graph. LLM generates lesson cards with structured prompts including student context, depth, test format, prior knowledge, and what explanation styles have worked.

### Stack
Frontend: Next.js or React Native (mobile-first). Backend: Node/Python. DB: Postgres + vector DB. LLM: Claude API. Algorithms: BKT (pyBKT) + FSRS.

### Build Order
1. Weeks 1-2: Parser (make-or-break). 2. Weeks 3-4: Lesson + checkpoint generation. 3. Weeks 5-6: Knowledge tracker + scheduling. 4. Weeks 7-8: UI. 5. Weeks 9+: Data flywheel.

---

## Competitive Landscape

### The Pattern
Every competitor follows: **Upload -> Generate Tools -> Self-Serve Study.** They're all tool factories — flashcards, quizzes, notes, chatbot. The student decides what to do. Nobody builds a curriculum. Nobody controls the sequence. Nobody forces retrieval before explanation.

### Key Competitors

| Company | Scale | What They Do | Why They're Not This |
|---|---|---|---|
| **HyperKnow** | ~10K users, $1M seed | Closest competitor. Upload materials, proactive study calendar, "Deep Learn Sessions." Canvas integration. | Still chat-based. No forced retrieval. No real SRS. "Learner's Persona" is a dropdown form, not behavioral tracking. Tech is thin (numpy .npy files, Render free tier, 30 incidents/90 days). 6 people, Chinese international student niche via Xiaohongshu. AI-first, not learning-science-first. |
| **Vaia** | 40M users | Upload PDF, auto-generates courses with chapters + study plans. Spaced repetition included. | Generic teaching quality. No forced generation. No test-format adaptation. Scale is real but experience is undifferentiated. |
| **Oboe** | $16M Series A (a16z) | Designed multi-format courses (articles, podcasts, quizzes, games). Beautiful experience. | Generates from topics, NOT uploaded course materials. Can't do "teach me my psych professor's Lecture 12." |
| **StudyFetch** | $11.5M, 6M users | Upload + flashcards/quizzes/chatbot. TikTok growth machine (1B+ views). | Tool generator. Not adaptive. Billing complaints. |
| **Knowt** | 5M+ students | Has real SRS + test date spacing. | Flashcard tool only. No curriculum. No teaching. |
| **Gizmo** | 3M+, $3.5M seed | Best gamification (Duolingo for studying). Real SRS. | Review/retention only. Can't teach new concepts. |

### Big Tech Threats
- **ChatGPT Study Mode**: Socratic tutoring, free. Threat if they add upload + curriculum + progress tracking. But they're a model company, not a vertical product company.
- **Google NotebookLM**: Upload docs, get multi-modal outputs. Free, massive distribution. Zero structure.
- **Microsoft Copilot Study Agent**: Learning science, LMS integration. $18/mo. Unclear on material upload.

### What's Actually Defensible
1. Upload YOUR materials + structured teaching (HyperKnow tries but is early/thin, Vaia does it generically)
2. Forced generation before explanation (nobody does this)
3. Cross-student data flywheel (nobody does this)
4. Goal-aware adaptation — test type + date + depth changes the entire curriculum (nobody does this)
5. The designed experience — scrolling through visual cards, not chatting with a bot (only Oboe is close, but they don't take uploads)

---

## Risks and Rebuttals

The biggest risk is that students won't use a product that makes them work harder. Deslauriers (2019, Harvard): active learning produced 0.46 SD higher test scores but students rated it as feeling LESS effective. 84% of students use rereading as their main strategy; 1% use self-testing.

**The answer to every behavioral objection is the predicted exam score.** Deslauriers' students felt confused and assumed confusion = bad teaching because they had no performance signal. This product gives a performance signal after every session:

- "It feels harder than ChatGPT" -> but the score went up 5 points
- "Education apps have 2-3% day-30 retention" -> the product isn't designed for daily year-round use; it's designed for intense 2-3 week exam cycles; the academic calendar provides the retention structure
- "Students cram" -> the spacing schedule compresses near deadlines; cram mode still uses retrieval practice; still better than ChatGPT
- "Free AI will kill you" -> ChatGPT has no memory, no curriculum, no scheduling, no predicted score; it's a different category
- "The adaptive learning graveyard" -> Knewton had no pedagogy (hype), Byju's was fraud, AltSchool tried to replace schools; this is a study supplement, not a school replacement

**Real risks that don't have clean answers:**
- Slide parser quality at launch — if the first generated lesson feels generic, students bounce to simpler tools
- Content accuracy trust — one wrong explanation before a final and the student never opens the app again
- Effect sizes shrink at scale — Carnegie Learning showed d=0.20 at scale vs. d=0.76 in lab; budget for ~50% of cited effects

---

## The Sarah Walkthrough (The Product in Action)

Sarah: Sophomore at UNC. Pre-med. Bio 202 midterm in 8 days. MC. Uploaded 4 lecture PDFs on cellular respiration. Picked "Know it well." System knows from prior uploads she understands ATP and mitochondria but has zero knowledge of glycolysis, Krebs, ETC.

Knowledge graph: ~18 concepts with dependencies (glycolysis -> pyruvate oxidation -> Krebs -> ETC -> chemiosmosis). Scheduling engine finds the frontier: glycolysis (prerequisites mastered, concept at zero).

**Card 1** — Activates prior knowledge: "Your cells need ATP. Glucose has way more energy than one ATP can carry. How does the cell extract it?" (forward testing effect)

**Card 2** — Pretest before teaching: "Take a guess. Glucose has 6 carbons. What does the first step produce?" MC with confidence rating. Even a wrong guess primes encoding (pretesting effect, d = 0.82-1.24).

**Cards 3-4** — Teach one concept per card with clean diagram. Glycolysis = glucose splits into 2 pyruvate in the cytoplasm. Net +2 ATP. (segmenting d = 0.79, coherence d = 0.86)

**Card 5** — Retrieval checkpoint: "Without scrolling back: glycolysis splits glucose into ____. Happens in the ____. Net ATP: ____." MC format matching her exam. Distractors are real misconceptions. Confidence rating. (testing effect g = 0.50-0.73)

**Card 6** — MC trap coaching: "Answer choices love to say '4 ATP' for glycolysis. That's gross, not net." (test-format-aware)

**Card 7** — "Why does glycolysis happen in the cytoplasm?" Open text. She generates an explanation. (elaborative interrogation)

**Card 8** — Pretest for next concept (conditional on performance). Interleaving preview.

**End** — "Coming back in 2 days to make sure this sticks. Midterm readiness: 12% -> 18%." Total time: ~8 minutes.

**Day 3 relearning** (~2 minutes): "Where does glycolysis happen, what's the net ATP?" Context-varied retrieval — same concept, different angle, different question format. If she nails it: "Still sharp. Next review in 5 days." FSRS stability increases. 90 seconds of her time.

**Why this works:** Every card maps to a specific research finding. The lesson forces her brain to work at every step. If she'd pasted the same slides into ChatGPT, she'd read a 500-word explanation, feel like she understood it, close the tab, and remember almost none of it two days later.

---

## V2+ Ideas (Not for Launch)
- Voice mode ("Study Walk") — retrieval practice for AirPods
- Canvas LTI integration — zero-setup distribution through LMS
- RL-trained teaching model — 7B model aligned to teach, not answer (EMNLP 2025 proves ceiling exists; Series A move)
- Cross-course knowledge transfer — bio student who learned protein structure in biochem doesn't relearn it in molecular bio
- Multi-armed bandit over teaching strategies — real but only works at scale

## Validation Test
50 students, one course, one university, 2-3 weeks before a midterm. Measure: (1) day-2 return rate, (2) predicted score accuracy, (3) actual exam performance vs. ChatGPT control. Even half a letter grade improvement = the only published evidence that an LLM learning tool improves delayed test performance. That's the fundraising slide and the moat.
