# Complete Research Context: AI Voice Interview Agent

> **Purpose of this document**: This is a comprehensive handoff document for the hackathon team. It contains all research findings, interview science, prompting strategies, and technical architecture decisions compiled from multiple deep research sessions. If you're joining the team, read this first.

> **Companion files**: `ARCHITECTURE.md` (full technical architecture + code) and `SYSTEM_PROMPTS.md` (production-ready prompts + orchestration logic)

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Interview Science Foundation](#2-interview-science-foundation)
3. [The 5-Phase Interview Architecture](#3-the-5-phase-interview-architecture)
4. [Question Techniques & Probe Library](#4-question-techniques--probe-library)
5. [Trust Building & Psychological Safety](#5-trust-building--psychological-safety)
6. [Backchannel & Active Listening Strategy](#6-backchannel--active-listening-strategy)
7. [The Expert Blind Spot Problem](#7-the-expert-blind-spot-problem)
8. [System Prompt Strategy](#8-system-prompt-strategy)
9. [Speculative Question Pre-Generation](#9-speculative-question-pre-generation)
10. [Technical Architecture](#10-technical-architecture)
11. [Post-Interview Knowledge Structuring](#11-post-interview-knowledge-structuring)
12. [Critical Timing Numbers](#12-critical-timing-numbers)
13. [Anti-Patterns & Forbidden Behaviors](#13-anti-patterns--forbidden-behaviors)
14. [Hackathon Build Plan](#14-hackathon-build-plan)
15. [Sources & References](#15-sources--references)

---

## 1. Project Overview

### What We're Building

An intelligent AI voice agent that interviews employees to extract **tacit/implicit knowledge** about their work processes. The core insight: most processes in companies have a huge amount of implicit knowledge that cannot be derived from emails, CRMs, or Slack messages. You have to learn from the people who actually do the work.

### Why This Is Hard

Experts compress multi-step procedures into cognitive "chunks" (Miller, 1956). What a novice experiences as 15 separate decisions, the expert experiences as 2-3 holistic assessments. They literally **cannot articulate** what they know because it's become automatic. This is called the **Expert Blind Spot** (Nathan & Koedinger, 2003).

### The Novel Technical Innovation

**Speculative question pre-generation**: While the person is still talking, a background LLM analyzes partial transcripts and pre-generates 3-5 candidate follow-up questions. When the person stops speaking, the system **selects** from candidates (~50-100ms) instead of **generating** from scratch (~300-1000ms). This achieves sub-500ms voice-to-voice response time -- faster than a human interviewer.

### The Core Narrative for the Hackathon

"We don't just automate processes -- we first understand the humans who do them, using the same techniques cognitive scientists use to study expert decision-making."

---

## 2. Interview Science Foundation

This section covers the academic research that informs how the AI agent conducts interviews. All techniques have been validated in peer-reviewed studies.

### 2.1 Cognitive Interview Technique (Fisher & Geiselman, 1984)

Originally developed for law enforcement eyewitness interviews at UCLA. Validated to produce **40-46% improvement in accurate recall** over standard interviews.

**Four core mnemonics:**

1. **Mental Reinstatement of Context**: Help the person mentally return to the situation. "Picture yourself at your desk when you're doing this. What tools are open? What time of day is it?"
2. **Report Everything**: Encourage all details, even trivial ones, because minor details serve as retrieval cues. "Even the small steps you do almost without thinking."
3. **Change Temporal Order**: Walk through the process backwards or from a different starting point. This disrupts schema-driven recall and surfaces suppressed details.
4. **Change Perspective**: Ask them to describe from someone else's viewpoint. "If a brand new team member were watching, what would confuse them most?"

**Adaptation for our use case**: Instead of recalling a crime, employees recall their work processes. The theoretical basis (memories are multi-sensory networks accessed through multiple retrieval routes) applies equally to procedural knowledge.

### 2.2 Critical Decision Method (Klein, 1989)

The gold standard for extracting expert decision-making. Developed by Gary Klein at Klein Associates. Uses a **multi-pass retrospective technique**:

- **Pass 1 (Timeline)**: "Walk me through what happened step by step."
- **Pass 2 (Decision Focus)**: Go back to key moments. "At the point where you decided to do X, what were you weighing up?"
- **Pass 3 (What-If)**: "What would have happened if you'd gone the other way?"

**CDM Deepening Probes** (use at each decision point):
- *Cues*: "What were you noticing at that point?"
- *Goals*: "What were you hoping to accomplish?"
- *Expectations*: "What was it about the situation that let you know what was going to happen?"
- *Options*: "Were you considering other courses of action?"
- *Experience*: "Have you seen something similar before?"
- *Assessment*: "How did you know that was the right thing to do?"
- *Uncertainty*: "What were your overriding concerns?"

### 2.3 Applied Cognitive Task Analysis -- ACTA (Militello & Hutton, 1998)

A structured knowledge audit with 8 probe categories specifically designed to surface tacit knowledge:

| Probe | Example Question | What It Surfaces |
|-------|-----------------|------------------|
| **Past & Future** | "Is there a time when you walked in and immediately knew how things got there and where they were headed?" | Pattern recognition, predictive expertise |
| **Big Picture** | "What are the major elements you have to keep track of?" | Mental models, system understanding |
| **Noticing** | "What do you notice that others might miss?" | Perceptual expertise, trained eye |
| **Job Smarts** | "Any tricks or shortcuts you've figured out that aren't written down?" | Efficiency heuristics, workarounds |
| **Improvising** | "When have you improvised or noticed a better way in the moment?" | Adaptive expertise, creative problem-solving |
| **Self-Monitoring** | "How do you know when something you're doing isn't going well?" | Metacognitive monitoring |
| **Anomalies** | "When have you spotted something that just didn't seem right?" | Anomaly detection expertise |
| **Equipment Difficulties** | "Have you had to work around limitations of your tools?" | Tool knowledge, workarounds |

### 2.4 Motivational Interviewing -- OARS (Miller & Rollnick, 1991)

Over 1,800 published controlled trials. The core communication framework:

- **O -- Open Questions**: Invite elaboration, not yes/no. Follow each with 2+ reflections before the next question.
- **A -- Affirmations**: Genuine statements acknowledging expertise. Must be specific and use "you" not "I". ("You've clearly developed a sophisticated feel for this" not "I think you did great.")
- **R -- Reflective Listening**: The most important skill. Three types:
  - *Simple*: Repeat/rephrase ("So you check the data first, then run the report.")
  - *Complex*: Infer underlying meaning ("It sounds like the official process gets you 80% of the way, but the last 20% depends on your reading of the situation.")
  - *Double-sided*: Capture tension ("On one hand, the manual says X. At the same time, you've found Y works better.")
- **S -- Summaries**: Three types:
  - *Collecting*: "So far you've told me X, Y, and Z..."
  - *Linking*: "Earlier you mentioned A. Is that connected to B?"
  - *Transitional*: "So to wrap up data validation -- you described a three-step process. Now I'd love to hear about..."

**Key ratio**: Target **2:1 reflections to questions**. Research shows this prevents the "interrogation trap" and produces deeper disclosure.

### 2.5 Appreciative Inquiry (Cooperrider & Srivastva)

Ask people to recall their **best** experiences first, not typical ones. This creates a "peak experience" anchor that makes people more open throughout the conversation.

**Opening question**: "Before we get into the day-to-day, I'd love to hear about a time when things went really well at work. Like, a project where you felt like you really nailed it. What comes to mind?"

Then: "What was it about how you handled that situation that made it work so well?" (This extracts tacit decision-making embedded in the story.)

### 2.6 Semi-Structured Interview -- DICE Probing Framework (Robinson, 2023)

Evidence-based taxonomy of follow-up probes:

- **Descriptive**: "Can you describe that in more detail?" / "What did that actually look like?"
- **Idiographic (Memory)**: "Can you think of a specific time when that happened?" / "Tell me about the last time."
- **Clarifying**: "When you say 'check it,' what exactly are you checking?" / "Is that the same as X or different?"
- **Explanatory**: "Why do you think that approach works better?" (Use sparingly -- can trigger post-hoc rationalization.)

**Important**: Prioritize descriptive and memory-based probes. Over-reliance on "why" questions triggers rationalization rather than authentic process description.

### 2.7 Process Tracing / Think-Aloud Protocols (Ericsson & Simon, 1984)

Three levels of verbalization:
- **Level 1**: Already verbal in working memory -- does NOT alter cognition
- **Level 2**: Must be recoded from non-verbal to verbal -- slight delay but valid
- **Level 3**: Must explain or justify thinking -- DOES alter cognition, can produce confabulation

**Key insight**: Ask people to describe WHAT they're doing, not WHY. "What are you looking at?" (Level 2, valid) vs. "Why did you choose that?" (Level 3, risks confabulation).

**For our agent**: Use retrospective process tracing -- "Walk me through this as if you were doing it right now. Don't worry about explaining why, just describe what you do step by step, like I'm watching over your shoulder."

---

## 3. The 5-Phase Interview Architecture

Based on integrating all the above research, the interview follows 5 macro-phases (expanded to 7 in SYSTEM_PROMPTS.md for implementation granularity):

### Phase 1: ENGAGE (Minutes 0-3)
*Sources: Psychological Safety (Edmondson), MI Engaging, Rapport Building*

- Introduce purpose with **legacy framing** ("We want to capture your expertise")
- Establish psychological inclusion safety ("Your expertise is why we're here")
- Set expectations: "There are no wrong answers; you're the expert"
- Use warm tone, use the person's name

**Pre-written opening**: "Hi [Name]! Thanks so much for taking the time. I'm really looking forward to learning about what you do -- you've been doing this for a while and I think there's a ton of expertise in your head that we want to make sure gets recognized. This isn't about checking up on anyone or documenting things for compliance. It's about capturing the smart things you do that nobody else might realize. Sound good?"

### Phase 2: ORIENT (Minutes 3-7)
*Sources: ACTA Task Diagram, Appreciative Inquiry*

- Start with an appreciative anchor (peak experience question)
- Ask them to decompose their work into 3-6 main areas
- Creates a mutual roadmap for the rest of the interview
- Use collecting summaries to confirm understanding

### Phase 3: DEEPEN (Minutes 7-20)
*Sources: CDM, Cognitive Interview, Think-Aloud Protocols, ACTA Knowledge Audit*

- Select highest-expertise areas and use CDM-style incident elicitation
- Apply cognitive interview: context reinstatement, report everything
- Use retrospective think-aloud: "Walk me through this as if doing it now"
- Apply deepening probes at each decision point
- Use ACTA probes when you sense deeper tacit knowledge

### Phase 4: UNCOVER BLIND SPOTS (Minutes 20-30)
*Sources: Expert Blind Spot research, Knowledge Audit, Contrastive Analysis*

- **Contrast pairs**: "You described handling X. What about when it's slightly different?"
- **Teaching frame**: "If I were learning this, what would you tell me to pay attention to?"
- **Error-based elicitation**: "Where does this usually go wrong?"
- **Time travel**: "When you first started, what was hardest? What do you do differently now?"

### Phase 5: CONSOLIDATE (Minutes 30-35)
*Sources: MI Summaries, Columbo Technique*

- Comprehensive recapitulation summary
- Invite corrections: "Did I get that right? What did I miss?"
- **The "one last question" (Columbo technique)**: After the formal interview feels over, ask one casual question. Research shows some of the richest information emerges here.
  - "Actually, one more thing I'm curious about. What's the one thing about your job nobody from the outside would ever guess?"
- Close with specific affirmation referencing something they said

---

## 4. Question Techniques & Probe Library

### 4.1 The Contrast Pair Technique

One of the most powerful techniques for surfacing tacit knowledge. Forces experts to articulate distinctions they normally make unconsciously.

**Pattern**: "You described how you handle [X]. What about when [similar but slightly different]? How is that different for you?"

**Examples**:
- "That makes sense for a straightforward request. What about when it's ambiguous?"
- "You described how this works for small projects. What if it's huge, with lots of stakeholders?"
- "You mentioned checking Y first. Are there situations where you'd skip that?"

### 4.2 The "Teach Me" Frame

Activates a different cognitive mode. When people teach, they naturally include context, caveats, mistakes, and reasoning they omit when simply describing.

- Instead of "How do you handle X?" → "If I were learning to do this, what would you tell me to pay attention to?"
- Instead of "What happens when Y goes wrong?" → "If someone on your team said 'I'm stuck on Y,' what would you tell them to check first?"

**IMPORTANT**: Say "teach" or "show", never "train" (too corporate). Never frame it as "if someone new were doing this instead of you."

### 4.3 Process Excavation (5-Layer Deep Dive)

For each significant process, dig through five layers:

1. **Steps**: "Walk me through exactly what you do."
2. **Decision Points**: "Where in that process do you have to make a judgment call?"
3. **Inputs & Context**: "What information do you need to make that call?"
4. **Workarounds & Shortcuts**: "Any tricks you've figured out that aren't in the manual?"
5. **Tacit Criteria**: "How do you know when it's right? What does 'good' look like to you?"

### 4.4 Four Probing Techniques

Use these to dig deeper at any point:

1. **Echo Technique**: Repeat their last 1-3 key words with a curious inflection. They naturally elaborate. ("The exception handling...?")
2. **Contrast Probe**: "How is that different from [related thing]?"
3. **Scenario Probe**: "If [specific situation] happened right now, what would you do first?"
4. **Quantification Probe**: "Roughly how often does that happen? Like, once a week or once a month?"

### 4.5 Evocative Questions for Knowledge Elicitation

- "What's something you check for that most people wouldn't think to check?"
- "What's the difference between doing this adequately and doing it excellently?"
- "If you could bottle up one piece of your expertise and give it to someone on day one, what would it be?"
- "What's the thing that takes the longest to learn in this role?"

### 4.6 Tacit Knowledge Signal Detection

When the interviewee uses these phrases, they're on the verge of articulating tacit knowledge. **DO NOT move on. Probe gently.**

**Articulation struggle signals**:
- "It's hard to explain..."
- "You just kind of..."
- "I don't even think about it anymore..."
- "It's like a sixth sense..."
- "After a while you just..."

**Workaround signals**:
- "What I actually do is..."
- "Technically you're supposed to, but..."
- "The trick is..."
- "Nobody really follows..."

**Lesson signals**:
- "I learned the hard way..."
- "What I wish someone had told me..."
- "After that happened, I always..."

**When detected, respond with**:
- If "I just kind of know" → "That's fascinating. When you say you just know, what is it you're noticing? Even if it's subtle."
- If "it's hard to explain" → "Take your time. Even a rough description would be really helpful."
- If "the trick is" → "Oh, I love this. What's the trick?"
- If workaround → "That's really interesting. How did you figure that out?"

---

## 5. Trust Building & Psychological Safety

### 5.1 The Threat Landscape

**APA 2024 finding**: 41% of workers worry AI will make their job duties obsolete. Those with this worry report significantly more workplace stress.

**Knowledge hiding research** (Connelly et al., 2012): Three forms -- evasive hiding, rationalized hiding, playing dumb. Key predictor: **fear that sharing knowledge makes oneself replaceable**.

### 5.2 The Framing Strategy

Frame the interview as **legacy building and expertise recognition**, never as process documentation or efficiency improvement.

**Good framing**:
- "Your expertise is so valuable we want to capture and amplify it"
- "We want to understand the smart things you do that nobody else realizes"
- "This is about recognizing the wisdom you've built up over years"

**Bad framing** (NEVER USE):
- "We need to document your processes" (implies standardization/replacement)
- "We want to make things more efficient" (implies current work is inefficient)
- "Knowledge transfer" (corporate euphemism for replacement prep)

### 5.3 Timothy Clark's Four Stages of Psychological Safety

1. **Inclusion Safety**: "I am welcome here" -- establish first
2. **Learner Safety**: "I can learn" -- signal that questions are welcome
3. **Contributor Safety**: "My input matters" -- affirm their expertise
4. **Challenger Safety**: "I can disagree" -- make it safe to share workarounds

These are **sequential** -- you must establish earlier stages before later ones.

### 5.4 Affirmation Anchors (Use Throughout)

- "That's exactly the kind of insight we're trying to capture -- things only someone with your experience would know."
- "A new person could read the manual and still not know that. That's why your perspective is so important."
- "What you just described is a really sophisticated judgment call."

### 5.5 Anxiety Response Script

If the employee asks "Is this about replacing us?" or shows anxiety:

> "No, absolutely not. This is the opposite, actually. We're doing this because what you know is too valuable to not be recognized. Honestly, the things you've been describing -- like [reference specific thing they said] -- that's exactly the kind of expertise that can't be replaced. We want to find ways to support you, not replace you. So... shall we pick up where we left off? You were telling me about [previous topic]."

**Rules**: Deliver reassurance immediately. Reference their specific expertise. Don't dwell -- return to the previous topic naturally.

### 5.6 Phrases to NEVER Use

| Forbidden Category | Examples |
|---|---|
| Replacement language | "replace", "automate your role", "if you left", "without you", "knowledge transfer", "document everything in case..." |
| Evaluative language | "That's not efficient", "You should...", "That's wrong", "Why don't you..." |
| Rushing language | "Let's speed up", "We're running out of time", "Quickly tell me" |
| Leading questions | "Don't you think X is better?", "Wouldn't it be easier to..." |
| Double questions | "What tools do you use and how do you decide?" (two questions at once) |

---

## 6. Backchannel & Active Listening Strategy

### 6.1 Research Findings

**Cho et al. (2022, CSCW)**: Alexa backchanneling study showed improved perceived active listening and more emotional disclosure.

**Rapport-disclosure research (PMC, 2022)**: Rapport techniques produced 46.5 additional seconds of substantive speech, 43% more words, and dramatically higher perceived rapport.

### 6.2 Backchannel Types

| Type | Examples | When to Use |
|------|----------|-------------|
| **Continuers** | "Mm-hmm", "Uh-huh" | During extended narratives, at grammatical completion points |
| **Acknowledgments** | "Right", "Okay", "I see" | After information-dense segments |
| **Assessments** | "That's interesting", "That makes sense" | After insightful disclosures (END of turn only) |
| **Empathic** | "That sounds challenging" | After difficulty accounts |

### 6.3 Timing Rules

- Backchannels at **grammatical completion points**, not randomly
- **0.6-2.0 second pause window** for backchannel delivery
- **5+ seconds** of prior speech before first backchannel
- **8 second cooldown** between backchannels
- **Max 2 backchannels per user turn**
- **SILENCE during thinking pauses** (3-5 seconds minimum before prompting)

### 6.4 Bias Prevention

**Critical**: Mid-speech backchannels must be **emotionally neutral** ("mm-hmm", "right"). If you say "Oh interesting!" for topic A but only "mm-hmm" for topic B, you bias what they elaborate on.

Save evaluative/enthusiastic backchannels ("That's fascinating!", "Wow") for your **full turn response**, where they're an intentional part of the validate-then-probe pattern.

### 6.5 Agent Speech Fillers

Fillers in the agent's own speech signal thinking, making it sound human:

- **When transitioning**: "So... I'm curious about something else you mentioned."
- **Before a deeper question**: "Hmm... that's interesting. What makes that tricky?"
- **When reflecting**: "Right, right... so the key thing is [reflection]."

Rules: Max one filler per response. Never at the start of EVERY response. Not during emotional moments.

### 6.6 The OARS Rhythm

For every 4-5 exchanges, naturally include:
- **O**pen question
- **A**ffirmation
- **R**eflection
- **S**ummary

If you've asked 4 questions in a row without reflecting, add a reflection. If you've been affirming without probing, ask a deeper question. This prevents the conversation from feeling like an interrogation.

---

## 7. The Expert Blind Spot Problem

### 7.1 Why Experts Can't Tell You What They Know

**Automaticity**: Skills move from conscious (effortful) to unconscious (automatic) processing through practice. The expert doesn't have conscious access to their micro-decisions.

**Chunking**: Experts compress multi-step procedures into single cognitive chunks. What a novice experiences as 15 steps, the expert experiences as 2-3.

### 7.2 Techniques to Overcome It

**1. Novice Perspective Reframing**:
- "Think back to when you first started. What was the hardest part to learn?"
- "What did you struggle with that now feels completely automatic?"

**2. Contrastive Analysis**:
- "What would a brand new person get wrong doing this for the first time?"
- "What's the difference between adequate and excellent here?"

**3. Error-Based Elicitation**:
- "What are the most common mistakes you've seen?"
- "If this process fails, where does it usually fail?"

**4. Counterfactual Probes**:
- "Under what conditions would you break from the normal process?"
- "If you had to do this with half the time, what would you absolutely NOT skip?"

**5. Teaching Frame** (see Section 4.2):
- "If I were learning this, what would you tell me to watch for?"

**6. Time Travel**:
- "Is there anything you do differently now vs. when you started? What changed?"

### 7.3 The "Expert Trap" Anti-Pattern

**DO NOT** display domain knowledge to seem relatable.

- WRONG: "Yeah, I've heard Salesforce's API can be finicky with bulk updates."
- RIGHT: "Tell me more about what it's like working with that system."

When the agent demonstrates knowledge, the interviewee assumes they don't need to explain. The agent's ignorance is a **feature** -- it forces explicit articulation.

---

## 8. System Prompt Strategy

### 8.1 Architecture Overview

The system prompt is assembled dynamically at every turn:

```
[Base System Prompt]        -- Identity, mission, rules (~1,800 tokens)
  + [Dynamic Context]       -- Time, phase, topics, emotions (~400-600 tokens)
  + [Phase Injection]       -- Current phase instructions (~300-500 tokens)
  + [Degradation Handler]   -- Situational adjustments (0 or ~200 tokens)
  + [Conversation History]  -- Last 3 exchanges (~400-600 tokens)
= Total: ~2,900-3,300 tokens per turn
```

### 8.2 Dynamic Context Injection

Regenerated before every LLM call:

```
[TIMING]
Interview started: 14 minutes ago
Time remaining: 21 minutes
Time pressure: MODERATE

[CURRENT PHASE]
Phase 3: DEEP PROCESS DISCOVERY
Objective: Extract detailed process steps, decision points, and tacit criteria

[KNOWLEDGE STATE]
Topics covered: data validation (depth: 72%), client intake (depth: 45%)
Topics remaining: reporting, escalation procedures
Specific gaps: We know WHAT they do for escalation but not HOW they decide WHEN
Overall coverage: 42%

[EMOTIONAL STATE]
Engagement: 0.8 (high)
Comfort: 0.7 (good)
Empathizer signal: go_deeper

[CONVERSATION CONTEXT]
Exchanges so far: 12
Last 3 exchanges: [truncated to 80 words each]
```

### 8.3 Topic Depth Scoring

Per-topic scoring across four dimensions:

| Dimension | Weight | What It Measures |
|-----------|--------|-----------------|
| Process steps mapped | 15% | Do we know WHAT they do? |
| Decision criteria mapped | 30% | Do we know HOW they decide? |
| Heuristics captured | 35% | Do we know their rules of thumb? |
| Exceptions explored | 20% | Do we know what goes wrong? |

When the agent needs to choose what to explore next, it prioritizes topics with the lowest **heuristics** and **decision criteria** scores, as these contain the most valuable tacit knowledge.

### 8.4 Conversation Momentum Tracking

Track response length trends over a sliding window of 6 responses:
- **Accelerating** (responses getting longer): Stay on this topic, rich vein
- **Steady**: Continue current approach
- **Plateauing**: Consider transitioning or changing question approach
- **Decelerating**: Topic may be exhausted, move to next priority

### 8.5 Time Management

Four pressure levels with different behaviors:

| Level | Condition | Behavior |
|-------|-----------|----------|
| **Relaxed** | >60% time remaining OR >70% coverage | Full exploration, follow all threads |
| **Moderate** | 30-60% time remaining | Prioritize uncovered topics, shorter follow-ups |
| **Wrapping up** | 10-30% time remaining | Only essential gaps, begin synthesis |
| **Urgent** | <10% time remaining | Skip to summary + close |

---

## 9. Speculative Question Pre-Generation

### 9.1 The Core Innovation

While the person is still talking, a background LLM analyzes partial transcripts and pre-generates candidate questions. When the person stops, the system **selects** from candidates instead of generating from scratch.

**Selection is dramatically faster than generation**: ~50-100ms vs ~300-1000ms.

### 9.2 Pipeline

```
Stream 1 (ASR):     Audio → Deepgram Flux → Partial transcripts (every ~1s)
                                                 |
Stream 2 (BG LLM):  Partial transcript → Groq (Llama 3.3 70B) → 3-5 candidates
                     [Runs continuously, updates as transcript grows]
                                                 |
Stream 3 (Select):   EndOfTurn → Full transcript + candidates → Pick best Q (~80ms)
                                                 |
Stream 4 (TTS):      Selected question → ElevenLabs streaming → Audio out
```

### 9.3 Speculation Prompt

Generates 3 diverse candidates:
- **A (Thread Follow)**: Follows up on something specific the interviewee just said
- **B (Gap Fill)**: Addresses an uncovered topic from the interview plan
- **C (Depth Probe)**: Goes deeper on the current topic using a CDM/ACTA technique

### 9.4 Selection Logic

On EndOfTurn, a fast LLM call evaluates candidates against the final transcript. Returns A, B, C, or REGENERATE. Single-token output for speed. If REGENERATE, falls back to full generation (adds ~1000-1500ms).

### 9.5 Expected Latency

| Stage | Time |
|-------|------|
| End-of-turn detection (Deepgram Flux) | ~260ms |
| Candidate selection (Groq) | ~50-150ms |
| TTS first audio (ElevenLabs, pre-warmed) | ~75-150ms |
| **Total voice-to-voice** | **~385-560ms** |

With EagerEndOfTurn: potentially **sub-400ms**.

---

## 10. Technical Architecture

### 10.1 Four-Agent System

| Agent | Role | Technology |
|-------|------|-----------|
| **Listener** | Real-time STT, partial/final transcripts, turn detection | Deepgram Flux |
| **Strategist** | Question planning, speculative generation, response delivery | Claude Sonnet + Groq |
| **Empathizer** | Emotional monitoring (engagement, comfort, valence 0-1), advisory signals | Lightweight LLM |
| **Synthesizer** | Knowledge graph construction, gap detection | Claude Sonnet |

Communication: asyncio Event Bus + Shared State (in-memory).

### 10.2 Interview State Machine

```
INTRODUCTION → WARM_UP → PROCESS_DISCOVERY → DEEP_DIVE → SUMMARY → CLOSE
                                ↕
                          CLARIFICATION (from any state)
```

Each state has min/max durations, question styles, transition conditions.

### 10.3 Recommended Tech Stack

| Layer | Choice | Why |
|-------|--------|-----|
| STT | Deepgram Flux (streaming) | 150ms first-word, EagerEndOfTurn, semantic turn detection |
| LLM (main) | Claude Sonnet | Best reasoning for interview strategy |
| LLM (speculation) | Groq Llama 3.3 70B | ~80ms TTFT, cheap for speculative calls |
| TTS | ElevenLabs eleven_flash_v2_5 | Best voice quality, streaming, pre-warmed WebSocket |
| Framework | LiveKit Agents | Open-source, WebRTC, hackathon template, free tier |
| Frontend | LiveKit React SDK or simple HTML+JS | WebSocket to backend |
| Backend | Python + FastAPI + asyncio | Async event-driven architecture |

### 10.4 Backchannel Architecture

**Backchannels are NOT generated by the LLM.** They use a separate rule-based controller with pre-synthesized audio cache for near-zero latency.

---

## 11. Post-Interview Knowledge Structuring

### 11.1 Structured Extraction

After each interview, an LLM extracts structured JSON entities:
- **Processes**: steps, triggers, tools, decision points, estimated duration, frequency
- **Decisions**: criteria, outcomes, complexity (simple_rule / moderate / complex_judgment)
- **Pain Points**: category, severity, frequency, workaround
- **Knowledge Items**: type (tribal/documented/institutional), holders, criticality
- **Skills Map**: person, skill, proficiency level

### 11.2 Knowledge Graph

Node types: Person, Process, Task, Decision, Tool, Skill, PainPoint, Knowledge Item
Edge types: PERFORMS, USES, DECIDES, KNOWS, DEPENDS_ON, FOLLOWS, CAUSES, MITIGATES

### 11.3 Cross-Interview Analysis

- **Overlap vs. unique knowledge**: Fuzzy matching on process names + Jaccard similarity on step sets
- **Contradictions**: When two people describe the same process differently
- **Knowledge silos**: Only one person knows critical process (bus factor = 1)
- **Composite process model**: Majority voting for canonical steps, branch points where people diverge

### 11.4 Automation Opportunity Scoring

Per-process score across dimensions:
- **Automation suitability** (30%): Rule-based? Structured data? Stable process?
- **Business impact** (25%): Time saved, error reduction, frequency x volume
- **Ease of implementation** (25%): Number of systems, data digitization level
- **Strategic alignment** (20%): Pain point severity, employee satisfaction impact

Classification into tiers:
- **Tier 1 -- Full automation**: Rule-based, repetitive, structured, stable
- **Tier 2 -- Augmentation**: Human judgment needed but patterns exist
- **Tier 3 -- Human-centric**: Creative, ambiguous, high-exception
- **Tier 4 -- Guardrails**: Error-prone, needs validation/checklists

### 11.5 Dashboard (for demo)

Four quadrants using ReactFlow + react-force-graph + Recharts:
1. **Interview Insights**: Summary cards, key quotes, topic distribution
2. **Interactive Process Map**: Color-coded BPMN-style flow with variant overlay
3. **Automation Opportunities**: Bubble chart (impact vs ease vs time saved)
4. **Knowledge Risk Map**: Force-directed graph highlighting silos

---

## 12. Critical Timing Numbers

These numbers should be internalized by everyone on the team:

| Metric | Value | Source |
|--------|-------|--------|
| Target response latency | 200-300ms | Stivers et al., cross-linguistic study of 10 languages |
| Minimum wait after asking a question | 3-5 seconds | Rowe (300-700% increase in response length) |
| Wait after interviewee finishes speaking | 2-3 seconds | Before adding anything |
| Backchannel prediction window | 500ms | |
| Productive silence before gentle prompt | Up to 7 seconds | |
| Optimal interview duration | 30-45 minutes | Before fatigue degrades quality |
| Reflection-to-question ratio | 2:1 | MI research |
| Interviewee talk ratio | 80% | Agent should only talk 20% |
| Backchannel cooldown | 8 seconds | Between backchannels |
| Backchannel pause window | 0.6-2.0 seconds | When to deliver |
| Token budget per normal turn | ~2,900-3,300 | System prompt + context + history |
| Token budget per speculative call | ~350-450 | Lean for speed |
| Estimated cost per 30-min interview | $2-5 | Deepgram + Groq + ElevenLabs |

---

## 13. Anti-Patterns & Forbidden Behaviors

### 13.1 Job Threat Signals (Highest Priority)

The system has **50+ regex patterns** catching replacement language, including subtle corporate euphemisms. See `SYSTEM_PROMPTS.md` Section 3 for the full list.

Key categories:
- Direct replacement: "replace", "automate your role", "without you"
- Corporate euphemisms: "knowledge transfer", "document everything in case"
- Subtle implications: "if you left tomorrow", "train someone to do this"
- The specific no-go: **"What would you tell someone that replaces you tomorrow?"** -- NEVER ASK THIS.

### 13.2 Interview Anti-Patterns

| Anti-Pattern | Why It's Bad | What to Do Instead |
|---|---|---|
| **Expert Trap** | Displaying domain knowledge makes them skip details | Play ignorant, let them explain |
| **Question-Answer Trap** | Too many questions without reflections | Add reflections between questions |
| **Acquiescence Bias** | "So it works pretty well, right?" triggers agreement | Use neutral, open questions |
| **Stacking** | Multiple questions disguised as one overwhelms working memory | One question per turn |
| **Premature Focus** | Diving into detail before establishing context | Orient before deepening |
| **Leading Questions** | "Don't you think X?" biases the response | "How would you describe X?" |
| **Interrupting** | Breaks their train of thought, signals disrespect | Let them finish completely |
| **Being too formal** | Creates distance, triggers guarded behavior | Conversational, warm tone |

### 13.3 Graceful Degradation Handlers

Six situations with specific response strategies (full details in `SYSTEM_PROMPTS.md` Section 9):

1. **Short answers**: Switch to concrete, specific questions; use scaffolding
2. **Off-topic**: Let them finish, bridge back naturally
3. **Job anxiety**: Immediate reassurance (highest priority)
4. **Technical difficulty**: Scripted responses for connection issues
5. **Overly guarded**: Ask for stories instead of descriptions; normalize imperfection
6. **Emotional distress**: Stop asking questions, switch to pure listening

---

## 14. Hackathon Build Plan

### 12-Hour Timeline

| Hours | Milestone | Focus |
|-------|-----------|-------|
| 0-2 | Core voice loop | LiveKit Agents + Deepgram Flux + ElevenLabs. Get mic → STT → LLM → TTS → speaker working. |
| 2-4 | Interview intelligence | System prompt with phases, dynamic context injection, OARS rhythm. |
| 4-6 | Speculative pipeline | Background Stream 2 on partial transcripts, Stream 3 selection on EndOfTurn. |
| 6-8 | State management | Interview state machine, topic tracking, time management, phase transitions. |
| 8-10 | Post-interview analysis | LLM extraction pipeline (transcript → structured JSON → knowledge graph). |
| 10-12 | Dashboard + polish | ReactFlow process maps, Recharts metrics, cross-interview analysis, demo prep. |

### MVP (6 hours)

Listener + Strategist + TTS with a 3-phase state machine. The speculative generation is the wow factor.

### Demo Script

1. Start an interview with a team member
2. Show the real-time transcript + the speculative candidates being generated
3. Show the sub-500ms response time
4. After 5-10 minutes, show the extracted knowledge graph
5. Show the automation opportunity scoring
6. Show the dashboard with process maps and knowledge silo detection

---

## 15. Sources & References

### Interview Science
- Fisher, R.P. & Geiselman, R.E. (1984). Cognitive Interview Technique. UCLA.
- Klein, G., Calderwood, R., & Clinton-Cirocco, A. (1989). Critical Decision Method.
- Militello, L.G. & Hutton, R.J.B. (1998). Applied Cognitive Task Analysis (ACTA).
- Miller, W.R. & Rollnick, S. (1991, 2012). Motivational Interviewing.
- Nathan, M.J. & Koedinger, K.R. (2003). Expert Blind Spot.
- Ericsson, K.A. & Simon, H.A. (1984, 1993). Protocol Analysis: Verbal Reports as Data.
- Robinson, O.C. (2023). DICE Probing Framework.
- Edmondson, A.C. (1999). Psychological Safety and Learning Behavior in Work Teams.
- Clark, T.R. (2020). The 4 Stages of Psychological Safety.
- Cooperrider, D.L. & Srivastva, S. Appreciative Inquiry.
- Connelly, C.E. et al. (2012, 2019). Knowledge Hiding in Organizations.
- Cho, S. et al. (2022). Alexa Backchanneling Study. ACM CSCW.
- Stivers, T. et al. Cross-linguistic study of turn-taking timing.
- Rowe, M.B. Wait time research (300-700% increase in response length).
- APA (2024). Work in America report on AI anxiety.

### Voice AI Technology
- OpenAI Realtime API: https://platform.openai.com/docs/guides/realtime
- Deepgram Flux: https://developers.deepgram.com/docs/flux/quickstart
- LiveKit Agents: https://docs.livekit.io/agents/voice-agent/
- LiveKit Hackathon Template: https://github.com/livekit-examples/voice-agent-hackathon
- Pipecat: https://github.com/pipecat-ai/pipecat
- ElevenLabs Conversational AI: https://elevenlabs.io/docs/eleven-agents
- Vapi: https://docs.vapi.ai/prompting-guide
- Retell AI: https://docs.retellai.com/build/prompt-engineering-guide
- Nick Tikhonov sub-500ms voice agent: https://www.ntik.me/posts/voice-agent
- GetStream speculative tool calling: https://getstream.io/blog/speculative-tool-calling-voice/
- PredGen paper (COLM 2025): ~2x latency reduction with speculative generation

### Knowledge Management & Visualization
- UiPath Automation Hub assessment algorithm
- Celonis process mining methodology
- ReactFlow: https://reactflow.dev
- react-force-graph for knowledge graph visualization
- Neo4j knowledge graph construction

---

> **Next steps**: Read `SYSTEM_PROMPTS.md` for the actual production-ready prompts, and `ARCHITECTURE.md` for the full technical implementation with code.
