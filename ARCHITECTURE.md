# AI Voice Interview Agent -- Orchestration Architecture

## Executive Summary

This document defines the complete architecture for a multi-agent AI system that
conducts voice interviews with employees to extract implicit process knowledge.
The system uses four specialized agents (Listener, Strategist, Empathizer,
Synthesizer), a speculative question preparation pipeline, a finite state machine
for interview flow, and a knowledge graph that accumulates across interviews.

The design targets a ~12-hour hackathon build window.

---

## 1. Multi-Agent Orchestration Pattern

### 1.1 Agent Roles

```
+------------------------------------------------------------------+
|                        ORCHESTRATOR                               |
|  (Event loop + shared state + routing)                            |
+------------------------------------------------------------------+
       |              |               |               |
  +----------+  +-----------+  +------------+  +-------------+
  | LISTENER |  | STRATEGIST|  | EMPATHIZER |  | SYNTHESIZER |
  | (real-   |  | (question |  | (emotional |  | (knowledge  |
  |  time    |  |  planning |  |  monitor)  |  |  graph      |
  |  STT)    |  |  + spec.) |  |            |  |  builder)   |
  +----------+  +-----------+  +------------+  +-------------+
```

**Agent 1 -- Listener**
- Consumes the raw audio stream from the browser via WebSocket
- Runs streaming STT (Deepgram Nova-3 or AssemblyAI Universal-Streaming)
- Emits two event types:
  - `partial_transcript` -- interim/unstable words (every ~100-300ms)
  - `final_transcript` -- stable, punctuated segment (on endpointing)
- Also emits `speech_started` and `speech_ended` signals for turn detection
- Implementation: thin wrapper around the STT WebSocket SDK

**Agent 2 -- Strategist**
- Receives `partial_transcript` and `final_transcript` events
- On partials: speculatively generates 2-3 candidate follow-up questions
- On finals: selects the best candidate and emits `next_question`
- Consults the current interview state and knowledge graph gaps
- Uses Claude API with streaming for fast response generation
- This is the "brain" -- it decides WHAT to ask

**Agent 3 -- Empathizer**
- Receives `final_transcript` events (and optionally audio-level features)
- Analyzes emotional valence, engagement level, and comfort
- Emits advisory signals:
  - `back_off` -- person seems uncomfortable, switch to lighter topic
  - `go_deeper` -- person is engaged and knowledgeable, probe further
  - `validate` -- person seems uncertain, offer encouragement
  - `pace_change` -- adjust speaking speed or question complexity
- These signals influence the Strategist but do not directly control output
- Implementation: Claude API call with a specialized system prompt analyzing
  linguistic cues (hedging language, enthusiasm markers, response length, etc.)

**Agent 4 -- Synthesizer**
- Receives `final_transcript` events
- Incrementally builds a structured knowledge graph (see Section 4)
- After each segment, updates the graph and emits:
  - `graph_update` -- new nodes/edges added
  - `coverage_report` -- what areas are well-covered vs. unexplored
- The coverage report feeds back to the Strategist so it knows where to probe
- Implementation: Claude API call with structured output (JSON schema)

### 1.2 Communication Pattern: Shared State + Event Bus

The agents communicate through a hybrid pattern:

```
+--------------------------------------------+
|           SHARED STATE (In-Memory)          |
|                                             |
|  interview_state: InterviewState            |
|  knowledge_graph: KnowledgeGraph            |
|  emotion_state: EmotionState                |
|  transcript_buffer: List[Segment]           |
|  candidate_questions: List[Question]        |
|  current_question: Question                 |
|  timing: TimingState                        |
+--------------------------------------------+
         ^                    |
         |   reads/writes     |
         |                    v
+--------------------------------------------+
|              EVENT BUS (asyncio)            |
|                                             |
|  Events:                                    |
|    partial_transcript(text, timestamp)       |
|    final_transcript(text, timestamp)         |
|    speech_started()                          |
|    speech_ended()                            |
|    next_question(question, confidence)       |
|    graph_update(nodes, edges)                |
|    coverage_report(gaps)                     |
|    emotion_signal(type, intensity)           |
|    state_transition(from, to)                |
+--------------------------------------------+
```

**Why this hybrid?**

- **Event bus** for temporal coordination: agents react to events asynchronously
  and in parallel. This is critical for the speculative execution pattern where
  the Strategist must begin generating questions WHILE the person is still
  talking.
- **Shared state** for context: every agent needs to read the current interview
  state, knowledge graph, and emotional context. A pure message-passing system
  would require excessive data copying.

**Implementation: Python `asyncio` with a simple pub/sub**

```python
import asyncio
from dataclasses import dataclass, field
from typing import Any, Callable, Dict, List
from enum import Enum

class EventType(Enum):
    PARTIAL_TRANSCRIPT = "partial_transcript"
    FINAL_TRANSCRIPT = "final_transcript"
    SPEECH_STARTED = "speech_started"
    SPEECH_ENDED = "speech_ended"
    NEXT_QUESTION = "next_question"
    GRAPH_UPDATE = "graph_update"
    COVERAGE_REPORT = "coverage_report"
    EMOTION_SIGNAL = "emotion_signal"
    STATE_TRANSITION = "state_transition"

@dataclass
class Event:
    type: EventType
    data: Dict[str, Any]
    timestamp: float

class EventBus:
    def __init__(self):
        self._subscribers: Dict[EventType, List[Callable]] = {}

    def subscribe(self, event_type: EventType, handler: Callable):
        if event_type not in self._subscribers:
            self._subscribers[event_type] = []
        self._subscribers[event_type].append(handler)

    async def publish(self, event: Event):
        handlers = self._subscribers.get(event.type, [])
        # Fire all handlers concurrently
        await asyncio.gather(
            *[handler(event) for handler in handlers],
            return_exceptions=True
        )

@dataclass
class SharedState:
    interview_state: str = "introduction"
    knowledge_graph: dict = field(default_factory=dict)
    emotion_state: dict = field(default_factory=lambda: {
        "valence": 0.5,
        "engagement": 0.5,
        "comfort": 0.5
    })
    transcript_buffer: list = field(default_factory=list)
    candidate_questions: list = field(default_factory=list)
    current_question: str = ""
    time_remaining_seconds: int = 1800  # 30 min default
    topics_covered: list = field(default_factory=list)
    topics_remaining: list = field(default_factory=list)
```

### 1.3 Agent Lifecycle

```
                     Browser (mic audio)
                           |
                      [WebSocket]
                           |
                           v
                    +-------------+
                    |  LISTENER   |  <-- always running, processes audio stream
                    +------+------+
                           |
              +------------+-------------+
              |            |             |
              v            v             v
         STRATEGIST   EMPATHIZER   SYNTHESIZER
         (async)      (async)      (async)
              |            |             |
              v            v             v
         candidate     emotion       graph
         questions     signals       updates
              |            |             |
              +------+-----+-------------+
                     |
                     v
              ORCHESTRATOR
              (picks best question,
               applies emotion modifiers,
               triggers state transitions)
                     |
                     v
              TTS --> WebSocket --> Browser (speaker)
```

---

## 2. Speculative / Predictive Question Preparation

This is the most latency-critical part of the system. The goal: when the person
stops talking, deliver the next question within 300-500ms, not 2-3 seconds.

### 2.1 The Problem

Standard sequential flow:

```
Person stops talking
  --> STT finalizes (200ms)
  --> LLM generates question (1000-2000ms)
  --> TTS synthesizes (200-400ms)
  --> Audio plays
Total: 1400-2600ms of silence  <-- unacceptable for natural conversation
```

### 2.2 The Solution: Speculative Execution

Inspired by CPU branch prediction and the speculative tool calling pattern
described by GetStream, we run question generation IN PARALLEL with the
person's speech:

```
Timeline:
=========

Person speaking: |████████████████████████████████████|
                 t0                                   t1 (speech ends)

Partial transcripts:  |p1  |p2  |p3  |p4  |p5  |p6  |
                      |    |    |    |    |    |    |
Speculation rounds:   |    |  [R1]  |  [R2]  | [R3] |
                      |    |   |    |   |    |  |   |
Candidate Qs:         |    | Q1a   | Q2a   | Q3a  |
                      |    | Q1b   | Q2b   | Q3b  |
                      |    | Q1c   |       |      |
                                                    |
                                              t1: speech ends
                                                    |
                                              SELECT best Q from
                                              latest speculation
                                              round (Q3a or Q3b)
                                                    |
                                              TTS begins immediately
                                                    |
                                              Latency: ~300-500ms
                                              (just selection + TTS start)
```

### 2.3 Speculation Architecture

```python
class SpeculativeStrategist:
    """
    Generates candidate questions while the person is still speaking.
    Uses partial transcripts to predict what the person is saying and
    pre-computes follow-up questions.
    """

    def __init__(self, event_bus: EventBus, state: SharedState):
        self.event_bus = event_bus
        self.state = state
        self.current_speculation: List[Question] = []
        self.speculation_generation: int = 0  # increments each round
        self._speculation_task: Optional[asyncio.Task] = None

        # Subscribe to events
        event_bus.subscribe(EventType.PARTIAL_TRANSCRIPT,
                            self.on_partial_transcript)
        event_bus.subscribe(EventType.FINAL_TRANSCRIPT,
                            self.on_final_transcript)
        event_bus.subscribe(EventType.SPEECH_ENDED,
                            self.on_speech_ended)

    async def on_partial_transcript(self, event: Event):
        """
        Called every ~300ms with interim transcript text.
        Triggers a new speculation round, cancelling the previous one.
        """
        partial_text = event.data["text"]
        # Only speculate if we have enough text to work with
        # (at least ~10 words to have meaningful context)
        word_count = len(partial_text.split())
        if word_count < 10:
            return

        # Cancel previous speculation if still running
        if self._speculation_task and not self._speculation_task.done():
            self._speculation_task.cancel()

        # Start new speculation round
        self.speculation_generation += 1
        gen = self.speculation_generation
        self._speculation_task = asyncio.create_task(
            self._speculate(partial_text, gen)
        )

    async def _speculate(self, partial_text: str, generation: int):
        """
        Generate 2-3 candidate follow-up questions based on partial
        transcript. Uses a fast Claude call with constrained output.
        """
        prompt = self._build_speculation_prompt(partial_text)

        try:
            response = await claude_client.messages.create(
                model="claude-sonnet-4-20250514",
                max_tokens=300,  # keep it short and fast
                messages=[{"role": "user", "content": prompt}],
                # Use streaming for even faster first-token
            )

            # Parse the candidate questions from response
            candidates = self._parse_candidates(response.content[0].text)

            # Only update if this is still the latest generation
            # (a newer partial may have already started a new round)
            if generation == self.speculation_generation:
                self.current_speculation = candidates
                self.state.candidate_questions = candidates

        except asyncio.CancelledError:
            pass  # Expected when a new partial supersedes this one

    async def on_speech_ended(self, event: Event):
        """
        Person stopped talking. Pick the best candidate question
        from the latest speculation round.
        """
        # Wait briefly for any in-flight speculation to complete
        if self._speculation_task and not self._speculation_task.done():
            try:
                await asyncio.wait_for(self._speculation_task, timeout=0.3)
            except asyncio.TimeoutError:
                pass  # Use whatever candidates we have

        if self.current_speculation:
            best = await self._select_best_candidate(
                self.current_speculation,
                self.state.transcript_buffer[-1] if self.state.transcript_buffer else ""
            )
            await self.event_bus.publish(Event(
                type=EventType.NEXT_QUESTION,
                data={"question": best.text, "confidence": best.confidence},
                timestamp=time.time()
            ))
        else:
            # Fallback: generate question from scratch (slower path)
            await self._generate_fresh_question()

    async def on_final_transcript(self, event: Event):
        """
        Stable transcript arrived. Use it to refine selection
        if speech hasn't ended yet, or as the basis for fresh
        generation if speculation produced poor candidates.
        """
        final_text = event.data["text"]
        self.state.transcript_buffer.append({
            "text": final_text,
            "timestamp": event.timestamp,
            "speaker": "interviewee"
        })

    def _build_speculation_prompt(self, partial_text: str) -> str:
        return f"""You are an expert interviewer extracting process knowledge.

Current interview state: {self.state.interview_state}
Knowledge gaps still to explore: {self.state.topics_remaining[:5]}
Emotional state: {self.state.emotion_state}

The person is currently saying (partial, may be incomplete):
"{partial_text}"

Based on what they seem to be describing, generate exactly 3 candidate
follow-up questions. Each question should:
1. Probe deeper into the process they're describing
2. Try to uncover implicit knowledge, decision criteria, or edge cases
3. Be conversational and natural

Format:
Q1: [question]
Q2: [question]
Q3: [question]"""
```

### 2.4 Speculation Efficiency Rules

1. **Minimum text threshold**: Do not speculate on fewer than ~10 words of
   partial transcript. The signal is too weak.
2. **Cancel-and-replace**: Each new partial cancels the previous speculation.
   Only the latest round matters.
3. **Use Sonnet for speculation, Opus for synthesis**: Speculation needs speed;
   use a faster model. Knowledge graph construction can use a more capable model.
4. **Cap speculation rounds**: At most 1 active speculation task at any time.
   Do not queue them.
5. **Fallback path**: If no speculation is ready when speech ends, fall back to
   a direct generation from the final transcript. This is slower (~1-2s) but
   ensures we always have a question.

### 2.5 TTS Pre-warming

To further reduce latency, pre-warm the TTS connection:

```python
class TTSManager:
    """
    Maintains a persistent WebSocket to the TTS service.
    Can begin synthesizing as soon as text is available.
    """
    def __init__(self):
        self.ws = None  # Persistent WebSocket connection

    async def connect(self):
        # Deepgram TTS WebSocket -- keep-alive connection
        self.ws = await websockets.connect(
            "wss://api.deepgram.com/v1/speak?model=aura-2&encoding=linear16",
            extra_headers={"Authorization": f"Token {DEEPGRAM_API_KEY}"}
        )

    async def speak(self, text: str) -> AsyncIterator[bytes]:
        """
        Stream text to TTS and yield audio chunks as they arrive.
        First audio byte typically arrives within 200ms.
        """
        await self.ws.send(json.dumps({"type": "Speak", "text": text}))
        async for message in self.ws:
            if isinstance(message, bytes):
                yield message  # PCM audio chunk
```

---

## 3. Interview State Machine

### 3.1 States and Transitions

```
                    +----------------+
                    | INTRODUCTION   |  (1-2 min)
                    | - Rapport      |
                    | - Set context  |
                    +-------+--------+
                            |
                     [comfort > 0.4]
                            |
                    +-------v--------+
                    |   WARM_UP      |  (2-3 min)
                    | - Easy Qs      |
                    | - Role overview |
                    +-------+--------+
                            |
                     [engagement > 0.5
                      AND 2+ responses]
                            |
                    +-------v--------+
                    | PROCESS        |  (8-12 min)
                    | DISCOVERY      |
                    | - Map workflow  |
                    | - Identify steps|
                    +-------+--------+
                            |
               +------------+------------+
               |                         |
        [found interesting          [coverage > 70%
         decision point]             OR time < 10min]
               |                         |
        +------v-------+         +-------v--------+
        |  DEEP DIVE   |         |   SUMMARY      |
        | - Edge cases  |         | - Confirm model|
        | - Why not how |         | - Fill gaps    |
        | - Exceptions  |         +-------+--------+
        +------+-------+                 |
               |                   [confirmed OR
        [exhausted OR               time < 3min]
         discomfort OR                   |
         time pressure]           +------v---------+
               |                  |    CLOSE       |
               +---->             | - Thank        |
                                  | - Next steps   |
                                  +----------------+
                                         |
                                   [CLARIFICATION]
                                   can be entered
                                   from any state
                                   when confusion
                                   is detected
```

### 3.2 State Definitions

```python
from enum import Enum
from dataclasses import dataclass
from typing import Optional, List

class InterviewPhase(Enum):
    INTRODUCTION = "introduction"
    WARM_UP = "warm_up"
    PROCESS_DISCOVERY = "process_discovery"
    DEEP_DIVE = "deep_dive"
    CLARIFICATION = "clarification"
    SUMMARY = "summary"
    CLOSE = "close"

@dataclass
class PhaseConfig:
    """Configuration for each interview phase."""
    phase: InterviewPhase
    min_duration_seconds: int
    max_duration_seconds: int
    question_style: str  # system prompt modifier for this phase
    goals: List[str]
    transition_conditions: dict  # conditions that trigger transitions

PHASE_CONFIGS = {
    InterviewPhase.INTRODUCTION: PhaseConfig(
        phase=InterviewPhase.INTRODUCTION,
        min_duration_seconds=60,
        max_duration_seconds=180,
        question_style="""You are warmly introducing yourself and the purpose
        of this interview. Be friendly, explain that you want to understand
        their work processes, and assure them there are no wrong answers.
        Ask their name, role, and how long they've been in the position.""",
        goals=["establish_rapport", "set_expectations", "get_role_context"],
        transition_conditions={
            "next": InterviewPhase.WARM_UP,
            "triggers": {
                "comfort_threshold": 0.4,
                "min_exchanges": 2
            }
        }
    ),
    InterviewPhase.WARM_UP: PhaseConfig(
        phase=InterviewPhase.WARM_UP,
        min_duration_seconds=120,
        max_duration_seconds=240,
        question_style="""Ask easy, open-ended questions about their typical
        day. What does a normal Monday look like? What tools do they use?
        Who do they interact with most? Keep it light and conversational.""",
        goals=["understand_daily_routine", "identify_key_processes",
               "map_tools_and_people"],
        transition_conditions={
            "next": InterviewPhase.PROCESS_DISCOVERY,
            "triggers": {
                "engagement_threshold": 0.5,
                "min_exchanges": 3,
                "identified_processes": 1
            }
        }
    ),
    InterviewPhase.PROCESS_DISCOVERY: PhaseConfig(
        phase=InterviewPhase.PROCESS_DISCOVERY,
        min_duration_seconds=300,
        max_duration_seconds=720,
        question_style="""Now systematically explore the processes they
        mentioned. For each process: What triggers it? What are the steps?
        What decisions do they make? What information do they need?
        What happens if something goes wrong? Who else is involved?""",
        goals=["map_process_steps", "identify_decision_points",
               "find_dependencies", "discover_exceptions"],
        transition_conditions={
            "deep_dive": InterviewPhase.DEEP_DIVE,
            "summary": InterviewPhase.SUMMARY,
            "triggers": {
                "deep_dive_on": "interesting_decision_point_found",
                "summary_on": "coverage_above_70_percent_or_time_low"
            }
        }
    ),
    InterviewPhase.DEEP_DIVE: PhaseConfig(
        phase=InterviewPhase.DEEP_DIVE,
        min_duration_seconds=180,
        max_duration_seconds=600,
        question_style="""Focus intensely on a specific decision point or
        process. Ask WHY, not just HOW. Probe for:
        - What makes this situation different from the normal case?
        - How do you know when X vs Y?
        - What would a new person get wrong here?
        - What's the thing that only experience teaches you?
        - Can you give me a specific example of when this was tricky?""",
        goals=["extract_heuristics", "find_edge_cases",
               "capture_tacit_knowledge", "understand_judgment_calls"],
        transition_conditions={
            "next": InterviewPhase.PROCESS_DISCOVERY,
            "summary": InterviewPhase.SUMMARY,
            "triggers": {
                "return_on": "topic_exhausted_or_discomfort",
                "summary_on": "time_below_10_minutes"
            }
        }
    ),
    InterviewPhase.CLARIFICATION: PhaseConfig(
        phase=InterviewPhase.CLARIFICATION,
        min_duration_seconds=30,
        max_duration_seconds=120,
        question_style="""Something was unclear or contradictory.
        Gently ask for clarification without making them feel corrected.
        Use phrases like 'I want to make sure I understand...' or
        'Could you help me connect...'""",
        goals=["resolve_ambiguity", "confirm_understanding"],
        transition_conditions={
            "next": "return_to_previous_state",
            "triggers": {
                "return_on": "clarification_received"
            }
        }
    ),
    InterviewPhase.SUMMARY: PhaseConfig(
        phase=InterviewPhase.SUMMARY,
        min_duration_seconds=120,
        max_duration_seconds=300,
        question_style="""Summarize what you've learned and play it back
        to the person. Ask them to correct anything wrong. Identify the
        2-3 biggest gaps in your understanding and ask about those.
        'So if I understand correctly, when X happens, you do Y because Z.
        Is that right? Is there anything I'm missing?'""",
        goals=["validate_model", "fill_critical_gaps",
               "get_corrections"],
        transition_conditions={
            "next": InterviewPhase.CLOSE,
            "triggers": {
                "close_on": "confirmed_or_time_below_3_minutes"
            }
        }
    ),
    InterviewPhase.CLOSE: PhaseConfig(
        phase=InterviewPhase.CLOSE,
        min_duration_seconds=30,
        max_duration_seconds=120,
        question_style="""Thank them sincerely. Summarize the key insights.
        Ask if there's anything important they want to add that you didn't
        ask about. Explain what happens next with the information.""",
        goals=["express_gratitude", "capture_final_thoughts",
               "set_expectations_for_followup"],
        transition_conditions={}
    ),
}
```

### 3.3 State Machine Implementation

```python
class InterviewStateMachine:
    def __init__(self, event_bus: EventBus, state: SharedState):
        self.event_bus = event_bus
        self.state = state
        self.current_phase = InterviewPhase.INTRODUCTION
        self.phase_start_time = time.time()
        self.exchange_count_in_phase = 0
        self.previous_phase = None  # for returning from CLARIFICATION

        event_bus.subscribe(EventType.FINAL_TRANSCRIPT,
                            self.on_exchange)
        event_bus.subscribe(EventType.EMOTION_SIGNAL,
                            self.on_emotion_signal)
        event_bus.subscribe(EventType.COVERAGE_REPORT,
                            self.on_coverage_report)

    async def on_exchange(self, event: Event):
        self.exchange_count_in_phase += 1
        await self._evaluate_transitions()

    async def on_emotion_signal(self, event: Event):
        signal_type = event.data["type"]
        if signal_type == "back_off" and self.current_phase == InterviewPhase.DEEP_DIVE:
            await self._transition_to(InterviewPhase.PROCESS_DISCOVERY)

    async def on_coverage_report(self, event: Event):
        await self._evaluate_transitions()

    async def _evaluate_transitions(self):
        config = PHASE_CONFIGS[self.current_phase]
        elapsed = time.time() - self.phase_start_time

        # Check minimum duration
        if elapsed < config.min_duration_seconds:
            return

        # Check maximum duration (force transition)
        if elapsed > config.max_duration_seconds:
            next_phase = config.transition_conditions.get("next")
            if next_phase:
                await self._transition_to(next_phase)
            return

        # Evaluate specific trigger conditions
        triggers = config.transition_conditions.get("triggers", {})
        # ... evaluate each trigger against current state ...

    async def _transition_to(self, new_phase: InterviewPhase):
        old_phase = self.current_phase
        if new_phase == InterviewPhase.CLARIFICATION:
            self.previous_phase = old_phase

        self.current_phase = new_phase
        self.phase_start_time = time.time()
        self.exchange_count_in_phase = 0
        self.state.interview_state = new_phase.value

        await self.event_bus.publish(Event(
            type=EventType.STATE_TRANSITION,
            data={"from": old_phase.value, "to": new_phase.value},
            timestamp=time.time()
        ))
```

---

## 4. Knowledge Graph Construction

### 4.1 Schema

The knowledge graph captures six categories of information:

```
+------------------+       performs        +------------------+
|      ROLE        |--------------------->|      TASK         |
| - title          |                      | - name            |
| - department     |                      | - description     |
| - tenure         |                      | - frequency       |
+------------------+                      | - trigger         |
                                          +--------+---------+
                                                   |
                                          has_subtask|
                                                   v
                                          +--------+---------+
                                          |    SUBTASK       |
                                          | - name           |
                                          | - sequence_order |
                                          | - duration_est   |
                                          +--------+---------+
                                                   |
                            +-----------+----------+---------+----------+
                            |           |                    |          |
                            v           v                    v          v
                    +-------+--+ +------+---+        +------+--+ +----+------+
                    | DECISION | |   TOOL   |        | HANDOFF | | EXCEPTION |
                    | POINT    | |          |        |         | |           |
                    | -criteria| | - name   |        | - to    | | - type    |
                    | -options | | - purpose|        | - info  | | - freq    |
                    | -default | | - pain   |        | - format| | - handling|
                    | -heurist.| |  points  |        |         | |           |
                    +----------+ +----------+        +---------+ +-----------+
                         |
                         v
                   +-----+------+
                   | HEURISTIC  |
                   | - rule     |
                   | - context  |
                   | - learned  |
                   |   _from    |
                   | - confidence|
                   +------------+
```

### 4.2 Graph Data Structure

```python
from dataclasses import dataclass, field
from typing import Optional, List, Dict
from enum import Enum
import uuid

class NodeType(Enum):
    ROLE = "role"
    TASK = "task"
    SUBTASK = "subtask"
    DECISION_POINT = "decision_point"
    TOOL = "tool"
    HANDOFF = "handoff"
    EXCEPTION = "exception"
    HEURISTIC = "heuristic"

class EdgeType(Enum):
    PERFORMS = "performs"
    HAS_SUBTASK = "has_subtask"
    REQUIRES_DECISION = "requires_decision"
    USES_TOOL = "uses_tool"
    HANDS_OFF_TO = "hands_off_to"
    HAS_EXCEPTION = "has_exception"
    APPLIES_HEURISTIC = "applies_heuristic"
    PRECEDES = "precedes"  # ordering between subtasks
    DEPENDS_ON = "depends_on"

@dataclass
class KnowledgeNode:
    id: str = field(default_factory=lambda: str(uuid.uuid4())[:8])
    type: NodeType = NodeType.TASK
    label: str = ""
    properties: Dict = field(default_factory=dict)
    confidence: float = 0.5  # how certain we are about this node
    source_quotes: List[str] = field(default_factory=list)  # verbatim quotes
    interview_id: str = ""  # which interview this came from
    timestamp: float = 0.0

@dataclass
class KnowledgeEdge:
    source_id: str = ""
    target_id: str = ""
    type: EdgeType = EdgeType.PERFORMS
    properties: Dict = field(default_factory=dict)
    confidence: float = 0.5

@dataclass
class KnowledgeGraph:
    nodes: Dict[str, KnowledgeNode] = field(default_factory=dict)
    edges: List[KnowledgeEdge] = field(default_factory=list)

    def add_node(self, node: KnowledgeNode):
        self.nodes[node.id] = node

    def add_edge(self, edge: KnowledgeEdge):
        self.edges.append(edge)

    def get_nodes_by_type(self, node_type: NodeType) -> List[KnowledgeNode]:
        return [n for n in self.nodes.values() if n.type == node_type]

    def get_coverage_score(self) -> Dict[str, float]:
        """
        Calculate how well each area has been explored.
        Returns coverage by node type as a percentage.
        """
        coverage = {}
        for node_type in NodeType:
            nodes = self.get_nodes_by_type(node_type)
            if not nodes:
                coverage[node_type.value] = 0.0
            else:
                avg_confidence = sum(n.confidence for n in nodes) / len(nodes)
                coverage[node_type.value] = avg_confidence
        return coverage

    def find_gaps(self) -> List[str]:
        """Identify areas that need more exploration."""
        gaps = []
        tasks = self.get_nodes_by_type(NodeType.TASK)
        for task in tasks:
            task_edges = [e for e in self.edges if e.source_id == task.id]
            edge_types = {e.type for e in task_edges}

            if EdgeType.REQUIRES_DECISION not in edge_types:
                gaps.append(
                    f"Task '{task.label}': no decision points identified"
                )
            if EdgeType.HAS_EXCEPTION not in edge_types:
                gaps.append(
                    f"Task '{task.label}': no exceptions/edge cases captured"
                )
            if EdgeType.APPLIES_HEURISTIC not in edge_types:
                gaps.append(
                    f"Task '{task.label}': no heuristics/rules of thumb found"
                )
        return gaps
```

### 4.3 Incremental Graph Building via Claude

```python
class SynthesizerAgent:
    """
    Listens to final transcripts and incrementally builds the
    knowledge graph using Claude's structured output.
    """

    EXTRACTION_PROMPT = """You are analyzing an interview transcript segment
to extract structured process knowledge.

Current knowledge graph state:
{graph_summary}

New transcript segment:
"{segment}"

Extract any NEW information from this segment. Return a JSON object:
{{
  "new_nodes": [
    {{
      "type": "task|subtask|decision_point|tool|handoff|exception|heuristic",
      "label": "short name",
      "properties": {{"key": "value"}},
      "confidence": 0.0-1.0,
      "source_quote": "verbatim quote from transcript"
    }}
  ],
  "new_edges": [
    {{
      "source_label": "existing or new node label",
      "target_label": "existing or new node label",
      "type": "performs|has_subtask|requires_decision|uses_tool|
              hands_off_to|has_exception|applies_heuristic|precedes|
              depends_on"
    }}
  ],
  "updated_nodes": [
    {{
      "label": "existing node label",
      "updates": {{"key": "new_value"}},
      "new_confidence": 0.0-1.0
    }}
  ]
}}

Only include genuinely new information. Do not repeat what is already
in the graph. If the segment contains no extractable process knowledge,
return empty arrays."""

    async def process_segment(self, segment: str):
        graph_summary = self._summarize_graph()
        prompt = self.EXTRACTION_PROMPT.format(
            graph_summary=graph_summary,
            segment=segment
        )

        response = await claude_client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=1000,
            messages=[{"role": "user", "content": prompt}],
        )

        extraction = json.loads(response.content[0].text)
        self._apply_extraction(extraction)

        # Publish updates
        gaps = self.state.knowledge_graph.find_gaps()
        await self.event_bus.publish(Event(
            type=EventType.COVERAGE_REPORT,
            data={"gaps": gaps,
                  "coverage": self.state.knowledge_graph.get_coverage_score()},
            timestamp=time.time()
        ))
```

---

## 5. Cross-Interview Synthesis

### 5.1 Architecture

After multiple interviews, a separate pipeline compares and merges graphs:

```
Interview 1 Graph  ──┐
Interview 2 Graph  ──┼──> MERGER ──> UNIFIED GRAPH ──> ANALYSIS
Interview 3 Graph  ──┘                                    |
                                                          v
                                                 +--------+--------+
                                                 | - Contradictions |
                                                 | - Gaps           |
                                                 | - Consensus      |
                                                 | - Process Map    |
                                                 +------------------+
```

### 5.2 Merge Strategy

```python
class CrossInterviewSynthesizer:
    """
    Merges knowledge graphs from multiple interviews about
    the same process/team.
    """

    async def merge_graphs(
        self,
        graphs: List[KnowledgeGraph],
        interview_metadata: List[dict]
    ) -> dict:
        """
        Use Claude to intelligently merge multiple knowledge graphs.
        Returns a unified graph plus analysis.
        """
        graph_summaries = []
        for i, (graph, meta) in enumerate(zip(graphs, interview_metadata)):
            summary = {
                "interview_id": i,
                "interviewee_role": meta.get("role", "unknown"),
                "nodes": [
                    {
                        "type": n.type.value,
                        "label": n.label,
                        "properties": n.properties,
                        "confidence": n.confidence
                    }
                    for n in graph.nodes.values()
                ],
                "edges": [
                    {
                        "source": graph.nodes.get(e.source_id, KnowledgeNode()).label,
                        "target": graph.nodes.get(e.target_id, KnowledgeNode()).label,
                        "type": e.type.value
                    }
                    for e in graph.edges
                ]
            }
            graph_summaries.append(summary)

        prompt = f"""Analyze these knowledge graphs from {len(graphs)}
interviews about the same team/process.

Graphs:
{json.dumps(graph_summaries, indent=2)}

Produce:
1. UNIFIED_GRAPH: Merge matching nodes, keep all unique information
2. CONTRADICTIONS: Where do interviewees disagree on process/decisions?
3. GAPS: What areas were not covered by ANY interview?
4. SHARED_IMPLICIT: What rules/heuristics do multiple people mention
   (suggesting these are real institutional knowledge)?
5. UNIQUE_KNOWLEDGE: What does only one person know (bus factor risk)?
6. PROCESS_MAP: A sequential process flow combining all perspectives

Return as structured JSON."""

        response = await claude_client.messages.create(
            model="claude-sonnet-4-20250514",
            max_tokens=4000,
            messages=[{"role": "user", "content": prompt}],
        )

        return json.loads(response.content[0].text)
```

### 5.3 Outputs

The cross-interview synthesis produces:

1. **Unified Process Map** -- a merged, high-confidence view of how work
   actually gets done (not how the documentation says it gets done)
2. **Contradiction Report** -- where people disagree, flagged for
   investigation
3. **Bus Factor Analysis** -- knowledge held by only one person
4. **Implicit Rule Catalog** -- heuristics and rules of thumb that
   multiple people independently confirmed
5. **Gap Report** -- areas no one could explain (possible automation
   opportunities or documentation needs)

---

## 6. Technology Stack (Hackathon MVP)

### 6.1 Component Selection

```
LAYER           TECHNOLOGY                  WHY
------          ----------                  ---
Frontend        Next.js or plain HTML+JS    Simple mic capture, WebSocket
Transport       WebSocket (native)          Bidirectional streaming
Backend         Python + FastAPI            Async-native, Claude SDK
STT             Deepgram Nova-3 (stream)    Best latency, WebSocket API
LLM             Claude Sonnet 4             Fast enough for speculation
                                            Smart enough for extraction
TTS             Deepgram Aura-2             Low latency, WebSocket API
State           In-memory (Python dict)     No database needed for MVP
Knowledge Graph In-memory Python objects    Export to JSON at end
Orchestration   asyncio + custom EventBus   Simple, no framework overhead
```

**Why NOT use LangGraph/CrewAI/AutoGen for the hackathon?**
- These frameworks add abstraction overhead that slows iteration
- Our agent communication pattern (event bus + shared state) is simpler
  than what these frameworks optimize for
- For 4 agents with clear roles, custom orchestration in ~100 lines of
  Python is faster to build and debug than learning a framework
- We can always migrate to a framework post-hackathon

**Why NOT use Pipecat or LiveKit?**
- These are excellent production frameworks, but for the hackathon:
  - Pipecat: great for 1-agent voice pipelines; less natural for our
    4-agent parallel architecture. Could be used for the audio I/O
    layer only.
  - LiveKit: more infrastructure than needed for a single-user demo.
    Designed for multi-participant rooms.
- Recommendation: use raw WebSocket + Deepgram SDK directly. Fewer
  moving parts. Consider Pipecat for production.

### 6.2 System Architecture Diagram

```
+--------------------------------------------------+
|  BROWSER (Frontend)                               |
|                                                   |
|  [Mic] -> MediaRecorder -> WebSocket --------+    |
|                                              |    |
|  [Speaker] <- AudioContext <- WebSocket <-+  |    |
|                                           |  |    |
|  [UI] - transcript display               |  |    |
|        - knowledge graph viz (optional)   |  |    |
|        - interview state indicator        |  |    |
+------------------------------------------|--|----+
                                           |  |
                                    audio  |  | audio
                                    out    |  | in
                                           |  |
+------------------------------------------|--|----+
|  PYTHON BACKEND (FastAPI + asyncio)       |  |    |
|                                           |  |    |
|  WebSocket Handler <---------------------+  |    |
|       |                                      |    |
|       v                                      |    |
|  +----------+     +---------+                |    |
|  | LISTENER |---->| EVENT   |                |    |
|  | (Deepgram|     | BUS     |                |    |
|  |  STT WS) |     |         |                |    |
|  +----------+     +----+----+                |    |
|                        |                     |    |
|            +-----------+-----------+         |    |
|            |           |           |         |    |
|            v           v           v         |    |
|     +----------+ +----------+ +--------+     |    |
|     |STRATEGIST| |EMPATHIZER| |SYNTHES.|     |    |
|     |(Claude   | |(Claude   | |(Claude |     |    |
|     | Sonnet)  | | Sonnet)  | | Sonnet)|     |    |
|     +----+-----+ +----+-----+ +---+----+     |    |
|          |             |           |          |    |
|          v             v           v          |    |
|     +---------+  +----------+ +--------+     |    |
|     |candidate|  | emotion  | |knowledge|    |    |
|     |questions|  | signals  | |graph    |    |    |
|     +---------+  +----------+ +--------+     |    |
|          |             |           |          |    |
|          +------+------+-----------+          |    |
|                 |                              |    |
|                 v                              |    |
|          +------+------+                       |    |
|          |ORCHESTRATOR |                       |    |
|          |select quest.|                       |    |
|          |apply emotion|                       |    |
|          |check state  |                       |    |
|          +------+------+                       |    |
|                 |                              |    |
|                 v                              |    |
|          +------+------+                       |    |
|          | TTS MANAGER |                       |    |
|          | (Deepgram   +-----------------------+    |
|          |  Aura WS)   |     audio chunks           |
|          +-------------+                            |
+-----------------------------------------------------+
```

### 6.3 File Structure (Hackathon MVP)

```
voice-interview-agent/
|
+-- frontend/
|   +-- index.html              # Single-page app
|   +-- app.js                  # Mic capture + WebSocket + audio playback
|   +-- style.css               # Minimal styling
|
+-- backend/
|   +-- main.py                 # FastAPI app + WebSocket endpoint
|   +-- event_bus.py            # EventBus + SharedState
|   +-- agents/
|   |   +-- listener.py         # Deepgram STT integration
|   |   +-- strategist.py       # Speculative question generation
|   |   +-- empathizer.py       # Emotional analysis
|   |   +-- synthesizer.py      # Knowledge graph extraction
|   +-- state_machine.py        # Interview phase management
|   +-- knowledge_graph.py      # Graph data structures
|   +-- tts_manager.py          # Deepgram TTS integration
|   +-- cross_synthesis.py      # Post-interview graph merging
|   +-- config.py               # API keys, model selection
|
+-- requirements.txt
+-- .env                        # API keys (gitignored)
+-- README.md
```

### 6.4 Key Dependencies

```
# requirements.txt
fastapi==0.115.*
uvicorn[standard]==0.34.*
websockets==14.*
anthropic==0.49.*           # Claude SDK
deepgram-sdk==3.*           # STT + TTS
python-dotenv==1.*
pydantic==2.*
```

### 6.5 Build Timeline (12 hours)

```
Hour  0-1:   Project setup, API keys, WebSocket skeleton
Hour  1-3:   Listener agent (Deepgram STT streaming)
              + basic frontend with mic capture
              + audio playback from TTS
Hour  3-5:   Strategist agent (Claude integration)
              + basic question generation (non-speculative first)
              + end-to-end: speak -> transcribe -> generate Q -> speak Q
Hour  5-7:   Interview state machine
              + phase transitions
              + phase-specific system prompts
Hour  7-9:   Speculative execution
              + partial transcript handling
              + cancel-and-replace speculation
              + candidate selection on speech end
Hour  9-10:  Empathizer agent
              + emotional signal detection
              + integration with Strategist
Hour 10-11:  Synthesizer agent
              + knowledge graph extraction
              + gap detection feeding back to Strategist
Hour 11-12:  Polish + demo prep
              + cross-interview synthesis (basic version)
              + UI improvements
              + bug fixes
              + demo script
```

### 6.6 Minimum Viable Path (If Time-Constrained)

If the 12-hour window shrinks, here is what to cut and in what order:

**Must have (6 hours):**
- Listener + Strategist + TTS (basic voice loop)
- Simple state machine (3 states: intro, discovery, close)
- Basic question generation (non-speculative)

**Should have (add 3 hours):**
- Speculative question preparation
- Full state machine with all phases
- Knowledge graph extraction (Synthesizer)

**Nice to have (add 3 hours):**
- Empathizer agent
- Cross-interview synthesis
- UI knowledge graph visualization

---

## 7. Critical Design Decisions and Tradeoffs

### 7.1 Latency Budget

```
Component               Target      Notes
---------               ------      -----
STT final transcript    200ms       Deepgram endpointing
Speculation (pre-done)    0ms       Already computed
Question selection       50ms       Simple scoring
TTS first byte          200ms       Deepgram Aura streaming
Network (2x)            100ms       Depends on deployment
---------               ------
TOTAL                   ~550ms      Acceptable for conversation
```

### 7.2 Claude API Call Strategy

The system makes multiple concurrent Claude API calls. Budget for cost:

```
Agent          Frequency              Model         Est. tokens/call
-----          ---------              -----         ----------------
Strategist     Every partial (~3/min) Sonnet        ~500 in, ~200 out
               Cancelled frequently
Empathizer     Every final (~1/min)   Sonnet        ~800 in, ~100 out
Synthesizer    Every final (~1/min)   Sonnet        ~1500 in, ~500 out

For a 30-minute interview:
- Strategist: ~90 calls, ~50% cancelled = ~45 completed = ~31K tokens
- Empathizer: ~30 calls = ~27K tokens
- Synthesizer: ~30 calls = ~60K tokens
- Total: ~118K tokens per interview

At Claude Sonnet pricing this is well within hackathon budget.
```

### 7.3 Handling Interruptions

If the person starts speaking while the agent is asking a question:

1. Immediately stop TTS playback (send silence / stop command)
2. Switch Listener back to transcription mode
3. Mark the interrupted question as "partially delivered"
4. Strategist notes what part of the question was heard

### 7.4 Context Window Management

For a 30-minute interview, the full transcript could be 5000-8000 words.
Strategy:

1. **Strategist** receives: last 5 exchanges (full text) + knowledge graph
   summary + current phase config. Not the full transcript.
2. **Empathizer** receives: last 3 exchanges + previous emotion state.
   Minimal context needed.
3. **Synthesizer** receives: current segment + graph summary.
   Incremental by design.
4. **Full transcript** is stored in SharedState but only sent to Claude
   during the Summary phase for validation.

---

## 8. Frontend Implementation Notes

### 8.1 Browser Audio Capture

```javascript
// Minimal browser-side implementation
class InterviewClient {
    constructor() {
        this.ws = null;
        this.mediaRecorder = null;
        this.audioContext = new AudioContext({ sampleRate: 16000 });
    }

    async start() {
        // Connect WebSocket to backend
        this.ws = new WebSocket('ws://localhost:8000/ws/interview');

        this.ws.onmessage = async (event) => {
            if (event.data instanceof Blob) {
                // Audio data from TTS -- play it
                await this.playAudio(event.data);
            } else {
                // JSON message (transcript, state update, etc.)
                const msg = JSON.parse(event.data);
                this.handleMessage(msg);
            }
        };

        // Capture microphone
        const stream = await navigator.mediaDevices.getUserMedia({
            audio: {
                sampleRate: 16000,
                channelCount: 1,
                echoCancellation: true,
                noiseSuppression: true
            }
        });

        // Use AudioWorklet for raw PCM streaming
        await this.audioContext.audioWorklet.addModule('pcm-processor.js');
        const source = this.audioContext.createMediaStreamSource(stream);
        const processor = new AudioWorkletNode(
            this.audioContext, 'pcm-processor'
        );

        processor.port.onmessage = (event) => {
            // Send raw PCM chunks to backend
            if (this.ws.readyState === WebSocket.OPEN) {
                this.ws.send(event.data.buffer);
            }
        };

        source.connect(processor);
        processor.connect(this.audioContext.destination);
    }

    handleMessage(msg) {
        switch (msg.type) {
            case 'partial_transcript':
                document.getElementById('live-transcript')
                    .textContent = msg.text;
                break;
            case 'final_transcript':
                this.appendTranscript(msg.text, 'interviewee');
                break;
            case 'agent_question':
                this.appendTranscript(msg.text, 'agent');
                break;
            case 'state_change':
                document.getElementById('phase-indicator')
                    .textContent = msg.phase;
                break;
            case 'knowledge_update':
                this.updateKnowledgeDisplay(msg.graph);
                break;
        }
    }

    async playAudio(blob) {
        const arrayBuffer = await blob.arrayBuffer();
        const audioBuffer = await this.audioContext
            .decodeAudioData(arrayBuffer);
        const source = this.audioContext.createBufferSource();
        source.buffer = audioBuffer;
        source.connect(this.audioContext.destination);
        source.start();
    }
}
```

### 8.2 Backend WebSocket Endpoint

```python
from fastapi import FastAPI, WebSocket
import asyncio

app = FastAPI()

@app.websocket("/ws/interview")
async def interview_websocket(websocket: WebSocket):
    await websocket.accept()

    # Initialize all components
    event_bus = EventBus()
    state = SharedState()
    tts = TTSManager()
    await tts.connect()

    listener = ListenerAgent(event_bus, state, websocket)
    strategist = SpeculativeStrategist(event_bus, state)
    empathizer = EmpathizerAgent(event_bus, state)
    synthesizer = SynthesizerAgent(event_bus, state)
    state_machine = InterviewStateMachine(event_bus, state)

    # When a question is ready, speak it
    async def on_next_question(event):
        question = event.data["question"]
        # Send question text to frontend for display
        await websocket.send_json({
            "type": "agent_question",
            "text": question
        })
        # Stream TTS audio to frontend
        async for audio_chunk in tts.speak(question):
            await websocket.send_bytes(audio_chunk)

    event_bus.subscribe(EventType.NEXT_QUESTION, on_next_question)

    # Deliver the introduction
    intro = "Hi! Thanks for taking the time to chat with me today. ..."
    await websocket.send_json({"type": "agent_question", "text": intro})
    async for chunk in tts.speak(intro):
        await websocket.send_bytes(chunk)

    # Main loop: receive audio from browser, forward to Listener
    try:
        while True:
            data = await websocket.receive_bytes()
            await listener.process_audio_chunk(data)
    except Exception:
        pass  # WebSocket closed
    finally:
        # Export knowledge graph
        graph_export = state.knowledge_graph
        await websocket.send_json({
            "type": "interview_complete",
            "knowledge_graph": graph_export
        })
```

---

## 9. Sequence Diagram: One Full Turn

```
Browser          Listener       Strategist     Empathizer    Synthesizer    TTS
  |                |               |               |              |          |
  |--audio chunk-->|               |               |              |          |
  |                |--STT WS------>|               |              |          |
  |                |               |               |              |          |
  |                |  partial_t    |               |              |          |
  |                |  "I usually"  |               |              |          |
  |                |-------------->|               |              |          |
  |                |               | (too short,   |              |          |
  |                |               |  skip)        |              |          |
  |                |               |               |              |          |
  |--audio chunk-->|               |               |              |          |
  |                |  partial_t    |               |              |          |
  |                |  "I usually   |               |              |          |
  |                |   check the   |               |              |          |
  |                |   dashboard   |               |              |          |
  |                |   first and   |               |              |          |
  |                |   then look   |               |              |          |
  |                |   at the..."  |               |              |          |
  |                |-------------->|               |              |          |
  |                |               |               |              |          |
  |                |               |--Claude call-->|              |          |
  |                |               |  (speculate)  |              |          |
  |                |               |               |              |          |
  |                |               |  candidates:  |              |          |
  |                |               |  Q1: "What do |              |          |
  |                |               |   you look for|              |          |
  |                |               |   on the      |              |          |
  |                |               |   dashboard?" |              |          |
  |                |               |  Q2: "What    |              |          |
  |                |               |   happens if  |              |          |
  |                |               |   something   |              |          |
  |                |               |   looks off?" |              |          |
  |                |               |  Q3: ...      |              |          |
  |                |               |               |              |          |
  |  (person stops |               |              |              |          |
  |   talking)     |               |               |              |          |
  |                |  speech_ended |               |              |          |
  |                |-------------->|               |              |          |
  |                |               |               |              |          |
  |                |  final_t      |               |              |          |
  |                |  "I usually   |               |              |          |
  |                |   check the   |               |              |          |
  |                |   dashboard   |               |              |          |
  |                |   first and   |               |              |          |
  |                |   then look   |               |              |          |
  |                |   at the open |               |              |          |
  |                |   tickets"    |               |              |          |
  |                |--+----------->|               |              |          |
  |                |  |            | SELECT Q1     |              |          |
  |                |  |            | (best match)  |              |          |
  |                |  |            |---------------------------->TTS        |
  |                |  |            |               |              |  "What do|
  |                |  |            |               |              |  you look|
  |<-------audio---|--|------------|---------------|--------------|--for on  |
  |                |  |            |               |              |  the..." |
  |                |  |            |               |              |          |
  |                |  +------------|-------------->|              |          |
  |                |               |  (emotion     |              |          |
  |                |               |   analysis)   |              |          |
  |                |               |               |              |          |
  |                |  +------------|---------------|------------->|          |
  |                |  |            |               |  (extract    |          |
  |                |  |            |               |   knowledge) |          |
  |                |               |               |              |          |
  |                |               |<--go_deeper---|              |          |
  |                |               |               |              |          |
  |                |               |<--graph_update+coverage------|          |
  |                |               |  gaps: "no exceptions for    |          |
  |                |               |   dashboard task"            |          |
  |                |               |               |              |          |
```

---

## 10. Production Considerations (Post-Hackathon)

These are out of scope for the hackathon but noted for future development:

1. **Persistent storage**: Replace in-memory state with Redis for state
   and a graph database (Neo4j) for the knowledge graph
2. **Multi-user**: Add authentication, session management, interview
   scheduling
3. **Pipecat migration**: Move audio I/O to Pipecat for production-grade
   audio handling, echo cancellation, VAD, and interruption handling
4. **Observability**: Add Langfuse or similar for tracing all Claude calls,
   latency metrics, and cost tracking
5. **Audio analysis**: Add prosody analysis (pitch, pace, volume) to the
   Empathizer for better emotional intelligence
6. **RAG for context**: Feed company documentation into the Strategist so
   it can ask informed questions about specific tools and processes
7. **Export formats**: Generate process documentation, BPMN diagrams,
   training materials from the knowledge graph
8. **A2A / MCP protocols**: For production multi-agent, consider adopting
   Google's Agent-to-Agent protocol or Anthropic's MCP for standardized
   agent communication
