# AI Sales Closer

An autonomous AI voice agent that calls leads, handles objections in real-time, and closes deals — 24/7.

## Mockup

Open `mockup.html` in any browser to see the interactive prototype. No server or dependencies required.

### Three Views

**Dashboard** — Real-time KPIs (calls, close rate, revenue, active calls), call history table with outcomes and sentiment scores, sales pipeline visualization, and objection frequency analytics.

**Live Call Monitor** — Split-panel view with an auto-scrolling conversation transcript, live sentiment gauge, objection tracker with resolved/active status, BANT qualification checklist, and call control buttons (mute, transfer, end, send payment link).

**Marketing/Overview** — Product overview with hero section, 4-step how-it-works flow, tech stack grid, key performance metrics, and AI vs Human closer comparison table.

### Features

- Dark theme UI with responsive layout (desktop + mobile)
- Animated live call transcript with messages appearing in real-time
- CSS-only charts and visualizations
- Fully self-contained — single HTML file, no external JS dependencies
- Inter font via Google Fonts

## Tech Stack (Planned)

| Component | Technology |
|-----------|-----------|
| Voice API | Vapi |
| Speech-to-Text | Deepgram |
| Text-to-Speech | ElevenLabs |
| LLM / Reasoning | Claude |
| Telephony | Twilio |
| CRM Integration | GoHighLevel |
| Call Analytics | Gong.io |

## Architecture

```
Lead books call → GHL webhook triggers AI agent
    → Vapi initiates outbound call via Twilio
    → Deepgram transcribes in real-time
    → Claude processes and generates responses
    → ElevenLabs synthesizes speech
    → Objections detected and handled autonomously
    → Payment link sent on close → CRM updated
```

## Getting Started

```bash
# Just open the mockup
open mockup.html
# or
xdg-open mockup.html
```

## License

Proprietary — All rights reserved.
