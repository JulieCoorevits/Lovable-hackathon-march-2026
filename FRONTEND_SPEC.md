# Frontend Specification: Interview Platform

> **Purpose**: This document describes the complete frontend product — what to build, how it looks, and how every screen works. Hand this to anyone who needs to build or understand the UI.

---

## 1. Product Concept

### What Is This?

A web platform where a **team lead** creates an interview session, gets **unique invite links** for each team member, and those team members click their link to have a **voice conversation with an AI agent** that learns about their daily work.

After interviews are done, the team lead sees a **dashboard** with extracted knowledge: process maps, automation opportunities, and knowledge risk areas.

### User Roles

| Role | What They Do |
|------|-------------|
| **Admin / Team Lead** | Creates a session, adds team members, sends invite links, views the dashboard after interviews |
| **Team Member / Interviewee** | Receives an invite link, clicks it, has a voice conversation with the AI agent |

---

## 2. User Flows

### Flow A: Admin Creates a Session

```
1. Admin lands on homepage → clicks "New Session"
2. Session setup form:
   - Session name (e.g., "Customer Success Team Q1")
   - Team members: Name + Email (add multiple)
   - Focus areas (optional): e.g., "Client onboarding", "Escalation handling"
   - Interview duration: 15 / 25 / 35 min (default: 25)
3. Admin clicks "Create Session"
4. Platform generates unique invite links per team member
5. Admin sees the Session Dashboard with:
   - List of team members + their unique links + copy buttons
   - Status per member: Pending / In Progress / Completed
   - "Send Invites" button (sends email with link) OR manual copy
6. As interviews complete, dashboard populates with insights
```

### Flow B: Team Member Does the Interview

```
1. Team member receives link: app.example.com/interview/abc123
2. Lands on a welcome page:
   - "Hi Sarah! Your team lead has invited you to share
     your expertise. This is a casual conversation about
     your daily work — there are no wrong answers."
   - Brief explanation (2-3 sentences)
   - Microphone permission prompt
   - "Start Conversation" button
3. Clicks "Start Conversation"
4. AI agent greets them by name and begins the interview
5. Full-screen voice call interface:
   - Animated waveform showing who's speaking
   - Elapsed time indicator (subtle, not stressful)
   - "End Call" button
   - NO transcript visible (feels like a natural conversation, not surveillance)
6. Interview ends (agent closes naturally, or user clicks End Call)
7. Thank-you screen:
   - "Thanks Sarah! Your insights are incredibly valuable."
   - Optional: "Anything else you'd like to add?" text box
   - Close tab
```

### Flow C: Admin Reviews Results

```
1. Admin returns to Session Dashboard
2. Sees completed interviews with summary cards
3. Clicks into the Analysis Dashboard:
   - Process maps (per person and combined)
   - Knowledge graph visualization
   - Automation opportunity scores
   - Knowledge risk / silo detection
4. Can export as PDF report
```

---

## 3. Screen-by-Screen Specification

### 3.1 Landing Page / Homepage

**URL**: `/`

**Purpose**: Explain the product and let admins start a session.

**Layout**:
```
┌──────────────────────────────────────────────────────┐
│  [Logo]  Fortano  ·  Process Intelligence        [?] │
├──────────────────────────────────────────────────────┤
│                                                      │
│         Understand your team's real processes         │
│         — not the documented ones.                    │
│                                                      │
│    Our AI interviews your team members one-on-one     │
│    to uncover the expertise, judgment calls, and      │
│    hidden knowledge that makes your team work.        │
│                                                      │
│              [ Create New Session ]                   │
│                                                      │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│   │ 1. Invite │  │ 2. Talk  │  │ 3. See   │          │
│   │ your team │  │ with AI  │  │ insights │          │
│   └──────────┘  └──────────┘  └──────────┘          │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**Key elements**:
- Hero section: headline, subheadline, CTA button
- Three-step explanation with icons
- No sign-up required for hackathon demo (session stored in URL/local state)
- Clean, modern, trust-building design (not corporate/cold)

**Design feel**: Warm, approachable. Think Notion or Linear meets a wellness app. Soft gradients, rounded corners, generous whitespace. The vibe should say "we care about people" not "we're optimizing headcount."

---

### 3.2 Session Setup Page

**URL**: `/session/new`

**Purpose**: Admin configures the interview session.

**Layout**:
```
┌──────────────────────────────────────────────────────┐
│  [← Back]          New Session                       │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Session Name                                        │
│  ┌──────────────────────────────────────────┐        │
│  │ Customer Success Team Q1                  │        │
│  └──────────────────────────────────────────┘        │
│                                                      │
│  Team Members                                        │
│  ┌──────────────────────────────────────────┐        │
│  │ Name              │ Email (optional)      │        │
│  ├───────────────────┼──────────────────────┤        │
│  │ Sarah Chen        │ sarah@company.com     │        │
│  │ Mike Rodriguez    │ mike@company.com      │        │
│  │ Priya Patel       │ priya@company.com     │        │
│  │ + Add team member                         │        │
│  └──────────────────────────────────────────┘        │
│                                                      │
│  Focus Areas (optional — helps guide the interview)  │
│  ┌──────────────────────────────────────────┐        │
│  │ Client onboarding, Escalation handling,   │        │
│  │ Quarterly reporting                       │        │
│  └──────────────────────────────────────────┘        │
│  (comma-separated topics the AI will prioritize)     │
│                                                      │
│  Interview Duration                                  │
│  ( ) 15 min — Quick pulse                            │
│  (●) 25 min — Standard (recommended)                 │
│  ( ) 35 min — Deep dive                              │
│                                                      │
│              [ Create Session ]                      │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**Behavior**:
- Generates a unique session ID (UUID) and unique interview token per team member
- Stores session in backend (or for hackathon: Supabase / local state)
- After creation, redirects to Session Dashboard

---

### 3.3 Session Dashboard (Admin View)

**URL**: `/session/:sessionId`

**Purpose**: Admin sees all invite links and interview statuses.

**Layout**:
```
┌──────────────────────────────────────────────────────┐
│  [Logo]     Customer Success Team Q1       [⚙ Edit]  │
├──────────────────────────────────────────────────────┤
│                                                      │
│  Team Members                        [Send All ✉]   │
│                                                      │
│  ┌──────────────────────────────────────────────┐    │
│  │ 👤 Sarah Chen                                │    │
│  │    Status: ✅ Completed (24 min)              │    │
│  │    Link: app.com/i/abc123  [📋 Copy]          │    │
│  │    [View Insights →]                          │    │
│  ├──────────────────────────────────────────────┤    │
│  │ 👤 Mike Rodriguez                             │    │
│  │    Status: 🟡 In Progress (12 min elapsed)    │    │
│  │    Link: app.com/i/def456  [📋 Copy]          │    │
│  ├──────────────────────────────────────────────┤    │
│  │ 👤 Priya Patel                                │    │
│  │    Status: ⏳ Pending                          │    │
│  │    Link: app.com/i/ghi789  [📋 Copy]          │    │
│  └──────────────────────────────────────────────┘    │
│                                                      │
│  ── Analysis (available after 2+ interviews) ──      │
│                                                      │
│  [Process Map]  [Knowledge Graph]  [Opportunities]   │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**Behavior**:
- Real-time status updates (WebSocket or polling)
- Copy button copies invite link to clipboard with toast confirmation
- "Send All" triggers emails via backend (or shows mailto: links for hackathon)
- Analysis tabs unlock after 2+ completed interviews
- Individual interview insights available immediately after each interview

---

### 3.4 Interview Welcome Page (Team Member View)

**URL**: `/i/:interviewToken`

**Purpose**: The interviewee lands here after clicking their invite link. This page builds trust and gets mic permission before the call starts.

**Layout**:
```
┌──────────────────────────────────────────────────────┐
│                                                      │
│                    [Warm illustration                 │
│                     of people talking]                │
│                                                      │
│             Hi Sarah! 👋                             │
│                                                      │
│    Your team lead invited you to share your           │
│    expertise in a quick voice conversation.           │
│                                                      │
│    This is a casual chat about your daily work —      │
│    the things you're great at, the judgment calls     │
│    you make, and the tricks you've figured out        │
│    that aren't written down anywhere.                 │
│                                                      │
│    There are no wrong answers. You're the expert.     │
│                                                      │
│    🎙️ We'll need microphone access for the call.     │
│                                                      │
│    ⏱️ About 25 minutes                               │
│                                                      │
│          [ Start Conversation 🎙️ ]                   │
│                                                      │
│    ─────────────────────────────────                  │
│    🔒 Your responses are used to help your team       │
│    work smarter. This is not a performance review.    │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**Critical design decisions**:
- **Warm, personal, non-corporate tone** — this page sets the emotional tone for the entire interview
- **Name personalization** — "Hi Sarah!" not "Welcome, participant"
- **Explicit reassurance** — "not a performance review", "you're the expert"
- **No mention of AI, automation, or efficiency** — frame as "sharing expertise"
- **Mic permission**: Request on button click, not on page load (less jarring)
- **Simple, calming design**: One primary action, no navigation, no distractions
- **Mobile-friendly**: Many people will open this link on their phone

**What happens on "Start Conversation" click**:
1. Request microphone permission
2. If granted → connect WebSocket to backend → transition to call screen
3. If denied → show friendly message explaining mic is needed with a retry button

---

### 3.5 Voice Call Screen (The Core Experience)

**URL**: Same `/i/:interviewToken` (SPA transition, no page reload)

**Purpose**: The actual voice conversation between the team member and the AI agent. This is the most important screen in the entire product.

**Layout**:
```
┌──────────────────────────────────────────────────────┐
│                                                      │
│                                                      │
│                                                      │
│                                                      │
│                   ╭─────────╮                        │
│                   │         │                        │
│                   │  ~~~~   │  ← Animated orb /      │
│                   │  ~~~~   │    waveform that        │
│                   │         │    pulses when AI       │
│                   ╰─────────╯    or user speaks       │
│                                                      │
│                  Listening...                         │
│                                                      │
│                                                      │
│                                                      │
│                                                      │
│                                                      │
│                                                      │
│     ⏱ 12:34                          [End Call 📞]   │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**Design principles**:
- **Minimalist, full-screen, distraction-free**
- **NO visible transcript** — this is critical. A visible transcript makes people self-conscious about their words, makes it feel like surveillance, and breaks the conversational flow. The transcript runs silently in the background.
- **Central animated element** that responds to audio:
  - When AI is speaking: element animates smoothly (e.g., pulsing orb, flowing waveform)
  - When user is speaking: element shows mic activity (different animation style)
  - During silence: element is calm/breathing
- **Subtle time indicator**: Small, bottom-left, low-contrast. NOT a countdown. Shows elapsed time only. The AI agent manages time internally.
- **End Call button**: Bottom-right, always visible but not prominent. Red when hovered.
- **No other UI elements**: No settings, no mute, no chat, no transcript toggle. Pure voice.
- **Dark background option**: Consider a dark/dim mode — feels more intimate, like a phone call in a quiet room

**Animation options for the central element** (pick one for hackathon):

1. **Breathing Orb**: A soft gradient sphere that expands/contracts with audio amplitude. Think Siri but warmer. Use CSS animations + Web Audio API for amplitude.

2. **Flowing Waveform**: A horizontal audio waveform with smooth, organic curves. Not a sharp EQ visualizer — more like gentle ocean waves.

3. **Concentric Rings**: Rings that ripple outward from center when audio is detected. Calming, abstract.

**Recommended for hackathon**: The breathing orb — it's the easiest to implement (CSS + a few lines of Web Audio API), looks polished, and feels warm/human.

**State indicators shown via the central element** (no text labels needed):

| State | Visual | Label (subtle, below orb) |
|-------|--------|---------------------------|
| AI speaking | Orb pulses smoothly in brand color | — (no label) |
| User speaking | Orb ripples with mic amplitude | — (no label) |
| AI thinking | Orb breathes slowly | "Thinking..." (only if >1.5s) |
| Silence / listening | Orb barely moves, soft glow | "Listening..." |
| Connecting | Orb loading animation | "Connecting..." |

**Technical notes**:
- WebRTC connection via LiveKit SDK (or raw WebSocket for hackathon)
- Web Audio API `AnalyserNode` for real-time amplitude → orb animation
- Audio stream sent to backend → Deepgram STT + Hume emotion + AI pipeline
- AI audio response streamed back and played via `<audio>` element or Web Audio API

---

### 3.6 Interview Complete Screen

**URL**: Same `/i/:interviewToken` (SPA transition)

**Purpose**: Thank the interviewee and close the experience warmly.

**Layout**:
```
┌──────────────────────────────────────────────────────┐
│                                                      │
│                                                      │
│                    ✨                                 │
│                                                      │
│           Thanks, Sarah!                             │
│                                                      │
│    Your insights are incredibly valuable.             │
│    The things you shared will help your               │
│    team work even better together.                    │
│                                                      │
│    ─────────────────────────────────                  │
│                                                      │
│    Anything else on your mind?                        │
│    ┌──────────────────────────────────────────┐      │
│    │ (optional — add any thoughts you didn't   │      │
│    │  get to mention during the conversation)  │      │
│    └──────────────────────────────────────────┘      │
│                                                      │
│              [ Submit & Close ]                      │
│                                                      │
│    You can close this tab at any time.                │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**Behavior**:
- Optional text box — not required, but valuable (people often think of things right after)
- On submit or close: mark interview as completed in backend
- Token becomes invalid (can't redo the interview)

---

### 3.7 Analysis Dashboard (Admin, Post-Interview)

**URL**: `/session/:sessionId/analysis`

**Purpose**: The payoff — visual insights from all completed interviews.

**Layout**:
```
┌──────────────────────────────────────────────────────┐
│  [← Session]    Analysis: Customer Success Team Q1   │
├──────────────────────────────────────────────────────┤
│                                                      │
│  [Overview] [Process Map] [Knowledge] [Opportunities]│
│                                                      │
│  ═══════════════════════════════════════════════════  │
│                                                      │
│  Overview Tab:                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │ 3 Interviews │  │ 47 Processes│  │ 12 Automation│  │
│  │ Completed    │  │ Discovered  │  │ Opportunities│  │
│  └─────────────┘  └─────────────┘  └─────────────┘  │
│                                                      │
│  Key Findings                                        │
│  ┌──────────────────────────────────────────────┐    │
│  │ • 3 processes have single-person knowledge    │    │
│  │ • Client onboarding has 4 undocumented steps  │    │
│  │ • Escalation criteria differ between members  │    │
│  │ • 2 manual data entry tasks are automatable   │    │
│  └──────────────────────────────────────────────┘    │
│                                                      │
│  Per-Person Summary Cards                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ Sarah    │  │ Mike     │  │ Priya    │          │
│  │ 24 min   │  │ 28 min   │  │ 22 min   │          │
│  │ 18 proc  │  │ 15 proc  │  │ 14 proc  │          │
│  │ 8 decis. │  │ 6 decis. │  │ 7 decis. │          │
│  └──────────┘  └──────────┘  └──────────┘          │
│                                                      │
│  ═══════════════════════════════════════════════════  │
│                                                      │
│  Process Map Tab:                                    │
│  Interactive BPMN-style flow chart (ReactFlow)       │
│  - Color-coded by: who does it / automation score    │
│  - Click node → see details, quotes, variants        │
│  - Toggle: per-person view / combined view           │
│                                                      │
│  ═══════════════════════════════════════════════════  │
│                                                      │
│  Knowledge Tab:                                      │
│  Force-directed graph (react-force-graph)            │
│  - Nodes = people, processes, tools, skills          │
│  - Edges = knows, performs, uses                     │
│  - Red highlight = single point of failure           │
│  - Cluster detection for knowledge silos             │
│                                                      │
│  ═══════════════════════════════════════════════════  │
│                                                      │
│  Opportunities Tab:                                  │
│  Bubble chart + ranked list                          │
│  - X-axis: ease of automation                        │
│  - Y-axis: business impact                           │
│  - Bubble size: time saved per week                  │
│  - Click bubble → detail card with reasoning         │
│                                                      │
│  [ Export Report (PDF) ]                             │
│                                                      │
└──────────────────────────────────────────────────────┘
```

---

## 4. Technical Frontend Stack

### Recommended for hackathon:

| Layer | Choice | Why |
|-------|--------|-----|
| **Framework** | React + Vite (or Next.js) | Fast to build, everyone knows it |
| **Styling** | Tailwind CSS + shadcn/ui | Beautiful components out of the box |
| **Voice connection** | LiveKit React SDK | Handles WebRTC, rooms, mic management |
| **Audio visualization** | Web Audio API `AnalyserNode` | Real-time amplitude for the orb animation |
| **Process maps** | ReactFlow | Interactive node-based diagrams |
| **Knowledge graph** | react-force-graph-2d | Force-directed graph visualization |
| **Charts** | Recharts | Simple, React-native charting |
| **State management** | React Context or Zustand | Lightweight, fast to implement |
| **Backend connection** | WebSocket (native or Socket.IO) | Real-time audio streaming + status updates |
| **Database** | Supabase (Postgres + Auth + Realtime) | Free tier, instant setup, real-time subscriptions |

### Key Frontend Libraries to Install

```bash
npm install @livekit/components-react livekit-client
npm install reactflow @reactflow/core @reactflow/controls
npm install react-force-graph-2d
npm install recharts
npm install zustand
npm install framer-motion        # for the orb animation
npm install lucide-react          # icons
```

---

## 5. Data Model (Frontend-Relevant)

### Session
```typescript
interface Session {
  id: string;                    // UUID
  name: string;                  // "Customer Success Team Q1"
  createdAt: Date;
  focusAreas: string[];          // ["Client onboarding", "Escalation"]
  interviewDurationMin: number;  // 15 | 25 | 35
  members: TeamMember[];
}
```

### TeamMember
```typescript
interface TeamMember {
  id: string;                    // UUID
  sessionId: string;
  name: string;
  email?: string;
  interviewToken: string;        // unique token for invite link
  status: 'pending' | 'in_progress' | 'completed';
  interviewStartedAt?: Date;
  interviewCompletedAt?: Date;
  interviewDurationSec?: number;
  additionalNotes?: string;      // from the post-interview text box
}
```

### InterviewInsight (returned by backend after processing)
```typescript
interface InterviewInsight {
  memberId: string;
  processes: ProcessNode[];
  decisions: DecisionNode[];
  painPoints: PainPoint[];
  knowledgeItems: KnowledgeItem[];
  automationOpportunities: AutomationOpp[];
}
```

### Invite Link Format
```
https://app.example.com/i/{interviewToken}

Example: https://app.example.com/i/a7f2c9e1-3b4d-4e5f-8a6b-9c0d1e2f3a4b
```

The token is a UUID that maps to a specific TeamMember record. It is single-use — once the interview is completed, the token is invalidated.

---

## 6. Voice Call Technical Flow

### Connection Sequence

```
1. User clicks "Start Conversation"
2. Frontend requests mic permission (getUserMedia)
3. Frontend calls backend: POST /api/interview/start { token }
4. Backend validates token, returns:
   - LiveKit room token (for WebRTC connection)
   - OR WebSocket URL (for direct audio streaming)
5. Frontend connects to LiveKit room / WebSocket
6. Audio flows:
   - User mic → WebRTC/WS → Backend → Deepgram STT + Hume AI
   - Backend LLM generates response
   - Response → TTS (ElevenLabs) → WebRTC/WS → User speakers
7. Frontend receives audio + plays it
8. Orb animation driven by:
   - Local mic amplitude (AnalyserNode on mic stream)
   - Remote audio amplitude (AnalyserNode on playback stream)
```

### Audio Visualization (Orb) Implementation Sketch

```javascript
// Simplified — connect to mic stream
const audioContext = new AudioContext();
const analyser = audioContext.createAnalyser();
analyser.fftSize = 256;

const micStream = await navigator.mediaDevices.getUserMedia({ audio: true });
const source = audioContext.createMediaStreamSource(micStream);
source.connect(analyser);

// Animation loop
const dataArray = new Uint8Array(analyser.frequencyBinCount);
function animate() {
  analyser.getByteFrequencyData(dataArray);
  const amplitude = dataArray.reduce((a, b) => a + b) / dataArray.length / 255;
  // amplitude is 0-1, use it to scale the orb
  orbElement.style.transform = `scale(${1 + amplitude * 0.4})`;
  requestAnimationFrame(animate);
}
animate();
```

---

## 7. Design Guidelines

### Color Palette

| Color | Usage | Hex (suggestion) |
|-------|-------|-------------------|
| **Primary** | Buttons, active states, orb glow | `#6366F1` (indigo-500) or warm purple |
| **Background** | Page backgrounds | `#FAFAFA` (light) / `#0F0F23` (dark call screen) |
| **Surface** | Cards, panels | `#FFFFFF` / `#1A1A2E` |
| **Text primary** | Headings, body | `#1A1A2E` / `#F0F0F0` |
| **Text secondary** | Descriptions, labels | `#6B7280` / `#9CA3AF` |
| **Success** | Completed status | `#10B981` (emerald-500) |
| **Warning** | In Progress status | `#F59E0B` (amber-500) |
| **Accent** | Highlights, the orb | Gradient from primary to a warm pink |

### Typography
- **Headings**: Inter or Plus Jakarta Sans, semibold
- **Body**: Inter, regular, 16px base
- **Monospace** (for tokens/links): JetBrains Mono

### Voice Call Screen — Special Considerations
- **Dark background** for the call screen (even if the rest of the app is light)
- Creates intimacy and focus, like a phone call
- The orb should be the only bright element — draws the eye, represents the AI
- Minimal UI — the less visual clutter, the more natural the conversation feels

### Mobile Responsiveness
- Welcome page and call screen MUST work on mobile (people will open invite links on phones)
- Call screen is actually ideal on mobile — it's just a phone call
- Dashboard/analysis can be desktop-only for hackathon

---

## 8. Hackathon Prioritization

### Must Have (Demo Day)
1. ✅ Welcome page with name personalization
2. ✅ Mic permission + WebSocket connection
3. ✅ Full-screen voice call with animated orb
4. ✅ AI agent speaks, listens, and conducts the interview
5. ✅ End call + thank you screen
6. ✅ Session dashboard showing invite links + status

### Nice to Have
7. 🟡 Email sending for invite links
8. 🟡 Real-time status updates on dashboard
9. 🟡 Post-interview analysis dashboard (even with mock data)
10. 🟡 Process map visualization
11. 🟡 Knowledge graph visualization

### Stretch Goals
12. ⭕ PDF export
13. ⭕ Cross-interview synthesis
14. ⭕ Automation opportunity scoring
15. ⭕ Dark mode toggle

### What to Mock for Demo if Time Runs Out
- Analysis dashboard can show **pre-computed results** from a test interview
- Process maps can use hardcoded ReactFlow nodes from a real interview transcript
- Knowledge graph can be populated from the Synthesizer agent's output saved as JSON

---

## 9. Key UX Principles

### For the Interviewee Experience

1. **Reduce anxiety at every step**: The welcome page, the orb, the closing screen — every touchpoint should feel warm and safe
2. **No surveillance vibes**: No transcript, no recording indicator, no "your manager will see this"
3. **Personal, not corporate**: Use their first name. Use casual language. The AI agent should feel like talking to a curious, friendly colleague — not a HR bot
4. **Effortless**: One click to start, one click to end. No signup, no settings, no configuration
5. **Respect their time**: Show duration upfront. The AI manages time. No "just one more question" after the stated time

### For the Admin Experience

1. **Quick setup**: Session creation should take under 60 seconds
2. **Copy-paste workflow**: Most admins will Slack or email the links manually — make copy easy
3. **Progress visibility**: Which interviews are done? Which are pending? At a glance.
4. **Actionable insights**: Don't just show data — show what to DO with it (automation opportunities, knowledge risks)

---

## 10. URL Structure

```
/                           → Landing page / homepage
/session/new                → Create new session (admin)
/session/:sessionId         → Session dashboard (admin)
/session/:sessionId/analysis → Analysis dashboard (admin)
/i/:interviewToken          → Interview flow (team member)
                              - Welcome → Call → Thank You (SPA transitions)
```

---

## 11. API Endpoints (Frontend Needs from Backend)

```
POST   /api/sessions                    → Create session + members
GET    /api/sessions/:id                → Get session details + member statuses
POST   /api/interview/start             → Validate token, return WebRTC/WS connection info
POST   /api/interview/end               → Mark interview as complete
POST   /api/interview/notes             → Submit optional post-interview text
GET    /api/sessions/:id/analysis       → Get processed insights for all interviews
WS     /api/interview/audio/:token      → WebSocket for real-time audio streaming
```

---

## 12. Example Invite Email (Content for "Send Invites" Feature)

```
Subject: Share your expertise — quick voice chat (25 min)

Hi Sarah,

[Team Lead Name] has invited you to share your expertise in
a quick voice conversation. An AI interviewer will ask you about
your daily work — the things you're great at, the judgment calls
you make, and the tricks you've figured out.

It takes about 25 minutes, there are no wrong answers, and
you're the expert.

Start your conversation: [link]

This is not a performance review. Your insights will help
the team work better together.

Thanks!
The [Product Name] Team
```

**Tone**: Warm, casual, respectful. No corporate jargon. No mention of "automation", "efficiency", "AI", or "optimization."
