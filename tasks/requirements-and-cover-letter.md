# AI Sales Closer Developer -- Requirements & Cover Letter

## Job Summary

**Client:** E-commerce education company selling a $950/month recurring program
**Budget:** $175 fixed (likely placeholder / negotiable based on scope)
**Level:** Expert
**Core Need:** Build an AI system that conducts and closes sales calls via voice, handles objections dynamically, and integrates with GoHighLevel CRM

**Client Already Has:**
- Hundreds of recorded sales calls
- A proven, battle-tested sales script
- Objection handling frameworks
- CRM (GoHighLevel)
- Automated AI-driven booking system

---

## Technical Requirements Analysis

### 1. Voice AI Pipeline (Core Engine)

**Architecture:** Cascading STT -> LLM -> TTS pipeline with full streaming

| Component | Recommended Stack | Rationale |
|-----------|------------------|-----------|
| **Orchestration** | Vapi ($0.05/min platform) | Developer-first, native GHL integration (May 2025), 7 specialized real-time models (endpointing, interruption detection, emotion detection, backchanneling, filler injection), 100+ languages, SOC2/HIPAA/PCI |
| **STT** | Deepgram Nova-3 | ~150ms latency, $0.0043/min, best for phone audio quality (8kHz PSTN) |
| **LLM** | Claude Sonnet 4.6 (primary), GPT-4o (fallback) | Claude for nuanced conversation + lower cost, GPT-4o for backup with provider hedging |
| **TTS** | ElevenLabs Flash v2.5 | 75ms to first audio, emotionally nuanced, most natural-sounding for sales conversations |
| **Telephony** | Twilio (SIP/PSTN) | Industry standard, reliable, Vapi-native integration |

**Target Latency:** <500ms end-to-end (achievable with streaming at every stage)
**Realistic All-In Cost:** ~$0.15-0.25/min per call

**Alternative Options:**
- **Retell AI** ($0.07+/min) -- Easier setup, 99.99% uptime, better for non-technical iteration. Users switching from Bland report 17% higher conversion.
- **GoHighLevel Native Voice AI** -- Built-in but less customizable. Could serve as Phase 1 MVP with upgrade path to Vapi.

### 2. Conversation Engine (Sales Logic)

**Objection Handling Architecture:**

```
Prospect Speech
    |
    v
[STT + Sentiment Detection]
    |
    v
[Intent Classifier] --> Identifies: objection type, buying signal, question, small talk, closing readiness
    |
    v
[Objection Router] --> Maps to pre-built response strategies
    |                   - Price/budget ("$950 is a lot")
    |                   - Timing ("not right now")
    |                   - Authority ("need to check with partner")
    |                   - Need/skepticism ("does this actually work?")
    |                   - Competitor ("already using X")
    |                   - Trust ("how do I know this is legit?")
    |
    v
[Context-Aware Response Generator]
    |   - Pulls prospect history from GHL
    |   - References specific pain points from earlier in call
    |   - Uses empathy-first framework: Acknowledge -> Clarify -> Reframe -> Advance
    |
    v
[Confidence Scorer] --> If confidence < threshold, escalate to human handoff
    |
    v
[TTS Output]
```

**Key Design Principles:**
- Pre-map the top 20-30 objections from existing recorded calls
- For each objection: empathy-first response (acknowledge, ask, reframe, advance)
- Dynamic context: pull CRM data mid-call to personalize rebuttals
- Sentiment-aware: calibrate tone/intensity based on detected emotion
- Never argue -- treat every objection as a buying signal

### 3. Call Flow Logic

```
Phase 1: Opening (30-60 sec)
    - Warm greeting, confirm identity
    - Set agenda/frame the call
    - Build initial rapport

Phase 2: Discovery/Qualification (3-5 min)
    - Ask qualifying questions (budget, timeline, motivation, current situation)
    - Active listening with backchanneling ("I see", "that makes sense")
    - Disqualify if not a fit (save time, maintain brand integrity)

Phase 3: Presentation (3-5 min)
    - Present offer tailored to their stated pain points
    - Use social proof / success stories
    - Tie features directly to their specific situation

Phase 4: Objection Handling (2-5 min, dynamic)
    - Handle objections as they arise
    - Multiple objection cycles supported
    - Escalation to human if stuck after 3+ attempts on same objection

Phase 5: Close (1-3 min)
    - Trial close -> gauge readiness
    - Assumptive close if qualified and ready
    - Send payment link via SMS (GHL workflow trigger)
    - Confirm next steps

Phase 6: Post-Call (automated)
    - Log call data to GHL (duration, outcome, objections raised, sentiment)
    - Update pipeline stage
    - Trigger follow-up workflow if not closed
    - Send call recording/summary to team
```

### 4. CRM Integration (GoHighLevel)

**API:** HighLevel API v2 (REST, base: `https://services.leadconnectorhq.com`)
**Auth:** Private Integration Token (Bearer) for single-account use

**Integration Points:**

| Timing | Action | API Endpoint |
|--------|--------|-------------|
| **Pre-call** | Pull contact record, past interactions, pipeline stage | `GET /contacts/{id}` |
| **Pre-call** | Check custom fields (qualification data) | Contact custom fields |
| **During call** | Book follow-up if needed | Calendar API |
| **During call** | Send payment link via SMS | Conversations API |
| **Post-call** | Update contact tags (e.g., "AI-called", "objection-price") | `POST /contacts/{id}/tags` |
| **Post-call** | Move opportunity stage (e.g., "Called" -> "Closed Won") | `PUT /opportunities/{id}` |
| **Post-call** | Log call notes/transcript | Conversations API (`type: "Call"`) |
| **Post-call** | Trigger follow-up workflow | Webhook trigger |

**Webhook Events to Subscribe:**
- `ContactCreated` -- trigger outbound call when new booking confirmed
- `AppointmentStatusChanged` -- handle no-shows, reschedules
- `OpportunityStatusChanged` -- track deal progression

### 5. Gong.io Integration (Call Analysis & Training)

**Purpose:** Mine existing sales call recordings to extract winning patterns for AI training

**Data Extraction Pipeline:**
1. Use Gong API (`/v2/calls/transcript`) to bulk-extract all historical call transcripts with speaker IDs
2. Cross-reference with deal outcomes (won/lost) via Gong's CRM sync
3. Use Smart Trackers to catalog every objection instance and the rep's response
4. Label data: call type, stage, outcome, deal value, objections raised
5. Build structured training corpus:
   - Winning opener sequences
   - Effective discovery question flows
   - Top objection-response pairs (filtered by win rate)
   - Closing language that converts

**Ongoing Integration:**
- MCP (Model Context Protocol) support enables real-time Gong queries from AI agent
- Pull account history and previous call insights before each call
- Track AI agent performance alongside human benchmarks in Gong's analytics

**API Limits:** 3 calls/sec, 10,000 calls/day (sufficient for batch extraction)

### 6. Conversational Memory & State Management

**Requirements:**
- Track conversation state across the full call flow (which phase, what's been discussed)
- Remember prospect responses from earlier in the call for contextual callbacks
- Maintain objection history (don't repeat the same rebuttal twice)
- Support mid-call tool calls without losing conversation thread

**Implementation:**
- Vapi handles conversation memory natively within the session
- Structured state object passed through the conversation engine:
  ```json
  {
    "phase": "objection_handling",
    "qualificationScore": 7,
    "objections": [
      {"type": "price", "response": "value_reframe", "resolved": true},
      {"type": "timing", "response": null, "resolved": false}
    ],
    "painPoints": ["struggling with current income", "wants location freedom"],
    "readinessSignals": ["asked about payment plans", "mentioned starting date"]
  }
  ```

---

## Estimated Timeline

| Phase | Duration | Deliverables |
|-------|----------|-------------|
| **Phase 0: Analysis** | Week 1 | Analyze recorded calls, extract objection database, document call flow |
| **Phase 1: MVP** | Weeks 2-4 | Working voice agent (Vapi + GHL), handles top 10 objections, basic call flow, CRM logging |
| **Phase 2: Optimization** | Weeks 5-8 | Full objection library, tone optimization, A/B testing, sentiment-aware responses |
| **Phase 3: Production** | Weeks 9-10 | Monitoring dashboard, human escalation flows, performance tracking, production hardening |

**MVP in 3-4 weeks. Production-ready in 8-10 weeks.**

---

## Estimated Costs (Ongoing Operations)

| Item | Monthly Cost (est. 500 calls/mo, avg 10 min) |
|------|----------------------------------------------|
| Vapi platform | ~$250 (5,000 min x $0.05) |
| Deepgram STT | ~$22 (5,000 min x $0.0043) |
| LLM (Claude/GPT) | ~$250-500 (depends on token usage) |
| ElevenLabs TTS | ~$200-400 |
| Twilio telephony | ~$50-150 |
| **Total** | **~$800-1,300/mo** for 500 calls |

At $950/mo recurring offer, **2 closed deals per month covers the entire AI infrastructure.**

---

## Success Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Call completion rate | >85% | % of calls where AI completes the full flow |
| Objection handling rate | >70% | % of objections resolved without human escalation |
| Close rate (AI) | >15-25% of human rate initially, scaling to 40-60% | Closed deals / qualified calls taken |
| Avg call duration | 8-15 min | Matches human call patterns |
| Response latency | <500ms | Measured end-to-end |
| CRM logging accuracy | 100% | All calls logged with correct data |
| Customer satisfaction | No complaints | Post-call survey or lack of negative feedback |

---

## Risk Mitigation

| Risk | Mitigation |
|------|-----------|
| AI sounds robotic | ElevenLabs emotional TTS + Vapi's filler injection + backchanneling models |
| Prospect asks off-script question | Graceful fallback: "That's a great question -- let me make sure I give you the best answer. [redirect to known territory]" |
| AI makes false promises | Strict guardrails in system prompt: never fabricate testimonials, pricing, or guarantees |
| Latency spikes | Provider hedging (dual STT/LLM providers), fallback routing |
| Regulatory compliance | Disclosure at call start ("This call is AI-assisted"), recording consent, TCPA compliance for outbound |
| Close rate underperforms | Human handoff escalation path; AI handles qualification, human closes |

---

---

# COVER LETTER

---

**Subject: AI Sales Closer -- Voice Agent with Objection Handling & GoHighLevel Integration**

Hi,

Your job post stands out because you've already done the hard part -- you have hundreds of recorded calls, a proven script, battle-tested objection frameworks, and a strong human close rate. Most AI voice projects fail because there's no foundation to build on. You have the foundation. My job is to engineer the AI layer on top of it.

**What I'd Build:**

A voice AI sales agent using **Vapi** for orchestration (native GoHighLevel integration as of 2025), **Deepgram** for real-time speech-to-text (<150ms), **Claude/GPT-4o** for the conversational brain, and **ElevenLabs** for natural-sounding voice synthesis (75ms to first audio). The entire pipeline streams end-to-end, targeting **sub-500ms response latency** -- fast enough to feel like a real conversation, not an IVR menu.

**How I'd Structure Objection Handling:**

This is the make-or-break piece. I'd start by mining your existing call recordings (and Gong.io data if available) to extract every objection-response pair, labeled by deal outcome. From there:

1. **Pre-map the top 20-30 objections** into structured categories (price, timing, authority, need, trust, competition)
2. **Build empathy-first response flows** for each: Acknowledge the concern -> Ask a clarifying question -> Reframe using their specific pain points -> Advance toward close
3. **Wire in real-time sentiment detection** (Vapi's built-in emotion model) so the agent adjusts tone -- more empathetic when frustration is detected, more confident when buying signals appear
4. **Never repeat the same rebuttal twice** -- conversational memory tracks what's been tried and escalates to alternative approaches
5. **Human handoff threshold** -- if the AI can't resolve an objection after 2-3 attempts, it gracefully transfers to a human closer

**Recommended Stack:**
- **Voice Orchestration:** Vapi (developer-first, 7 real-time conversation models, native GHL tools)
- **STT:** Deepgram Nova-3 (fastest, cheapest, best for phone audio)
- **LLM:** Claude Sonnet 4.6 primary, GPT-4o fallback (provider hedging for reliability)
- **TTS:** ElevenLabs Flash v2.5 (most natural emotional range)
- **Telephony:** Twilio SIP/PSTN
- **CRM:** GoHighLevel API v2 (contacts, pipelines, call logging, payment links, workflow triggers)
- **Call Intelligence:** Gong.io API for mining winning patterns from your existing call library

**Why AI Can Realistically Close at High Performance for This Offer:**

Your $950/mo education program is in the sweet spot for AI closing:
- **Defined, repeatable offer** -- not a custom enterprise negotiation
- **Consistent prospect profile** -- e-commerce aspirants with similar motivations and objections
- **Proven script** -- the AI doesn't need to improvise, it needs to execute a known-winning framework
- **Objections are finite and catalogued** -- price, timing, trust, and "is this legit?" cover 80%+ of resistance

Industry data shows AI agents achieve **3-4x conversion lifts** in transactional sales. For a structured offer like yours with pre-qualified booked calls, I'd target **15-25% of human close rate in the MVP** (week 3-4), scaling to **40-60%** after optimization (month 2-3). At $950/mo recurring, even a modest close rate generates significant ROI since the AI runs 24/7 at ~$0.20/min with no commissions, no sick days, and no ramp time.

**Estimated Timeline:**
- **Week 1:** Analyze your recorded calls, structure the objection database, design call flow logic
- **Weeks 2-4:** Build and test MVP voice agent connected to GHL calendar + CRM
- **Weeks 5-8:** Optimize objection handling, tone calibration, A/B test script variations
- **Weeks 9-10:** Production hardening, monitoring, human escalation flows

**MVP in 3-4 weeks. Production-ready in 8-10 weeks.**

I'm interested in a milestone-based structure and open to a performance bonus tied to close rate targets -- I have high confidence in the architecture and want skin in the game.

Happy to walk through the technical approach in more detail on a call.

Best,
[Your Name]

---

## Appendix: Key Sources & References

### Voice AI Platforms
- Vapi: https://vapi.ai / https://docs.vapi.ai
- Retell AI: https://www.retellai.com
- ElevenLabs: https://elevenlabs.io
- Deepgram: https://deepgram.com

### GoHighLevel
- API Docs: https://marketplace.gohighlevel.com/docs/
- Voice AI APIs: https://help.gohighlevel.com/support/solutions/articles/155000006379
- Private Integration Tokens: https://help.gohighlevel.com/support/solutions/articles/155000003054

### Gong.io
- API: https://help.gong.io/docs/what-the-gong-api-provides
- MCP Support: https://www.gong.io/press/gong-introduces-model-context-protocol-mcp-support
- Objection Handling Research: https://www.gong.io/blog/handling-sales-objections-with-ai
- Smart Trackers: https://help.gong.io/docs/smart-trackers

### Industry Research
- Gong Labs (300M+ cold call analysis): https://www.gong.io/blog/we-found-the-top-objections-across-300m-cold-calls
- AI Sales Statistics: https://www.envive.ai/post/ai-sales-agent-statistics
- Voice AI Latency Engineering (Sierra): https://sierra.ai/blog/voice-latency
- Voice AI Stack 2026 (AssemblyAI): https://www.assemblyai.com/blog/the-voice-ai-stack-for-building-agents
