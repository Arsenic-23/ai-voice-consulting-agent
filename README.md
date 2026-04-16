# 🤖 AI Calling Agent — Consulting Firm

An AI-powered voice sales automation system combining Twilio, OpenAI GPT-4o, ElevenLabs, n8n, and Supabase.

---

## 📁 Project Structure

```
ai-calling-agent/
├── README.md
├── .env.example
├── database/
│   └── schema.sql                        # Supabase table definitions
├── prompts/
│   └── sales-agent-system-prompt.md      # LLM system prompt
├── workflows/
│   ├── 01-outbound-calling.json          # Trigger → Twilio call → route outcome
│   ├── 02-inbound-call-handling.json     # Receive call → CRM lookup → TwiML stream
│   ├── 03-ai-conversation.json           # STT → GPT-4o → TTS → intent routing
│   ├── 04-followup-automation.json       # SMS / Email / WhatsApp by outcome
│   ├── 05-booking-calendar.json          # Google Calendar + confirmation
│   └── utils/
│       ├── retry-logic.json              # Exponential backoff retry calls
│       ├── lead-scoring.json             # Score leads 0–100 after each call
│       └── escalation.json              # Slack + email + live call transfer
└── docs/
    └── architecture.md                   # System design + flow diagram
```

---

## ⚡ Quick Start

### 1. Clone & Configure
```bash
cp .env.example .env
# Fill in all values in .env
```

### 2. Set Up Database
```bash
# Run in Supabase SQL Editor
psql -f database/schema.sql
```

### 3. Import Workflows into n8n
- Open n8n → **Workflows** → **Import from File**
- Import all `.json` files from `workflows/` and `workflows/utils/`
- Activate each workflow

### 4. Deploy AI Realtime Server *(required for voice streaming)*
```bash
# Separate Node.js server — not included here
# Must expose: POST /audio-response, WSS /stream
```

### 5. Configure Twilio Webhooks
| Event | URL |
|-------|-----|
| Incoming Call | `https://your-n8n.com/webhook/incoming-call` |
| Call Status | `https://your-n8n.com/webhook/call-status` |
| Recording Ready | `https://your-n8n.com/webhook/recording-ready` |

---

## 🔑 Required Services

| Service | Purpose | Free Tier |
|---------|---------|-----------|
| [Twilio](https://twilio.com) | VoIP calls + SMS | Trial credits |
| [OpenAI](https://platform.openai.com) | GPT-4o LLM | Pay-per-use |
| [ElevenLabs](https://elevenlabs.io) | Text-to-speech | 10K chars/mo |
| [Supabase](https://supabase.com) | Database + realtime | Free |
| [n8n](https://n8n.io) | Workflow automation | Self-host free |
| [Google Calendar](https://calendar.google.com) | Booking | Free |

---

## 🔁 Workflow Overview

| # | Workflow | Trigger | Output |
|---|----------|---------|--------|
| 1 | Outbound Calling | New lead in DB | Call initiated → routed |
| 2 | Inbound Call Handling | Twilio webhook | TwiML → AI stream |
| 3 | AI Conversation | Speech turn (per message) | Voice response + intent |
| 4 | Follow-Up Automation | Call outcome | SMS/Email/WhatsApp |
| 5 | Booking & Calendar | `BOOK_MEETING` intent | Calendar event + confirm |
| U1 | Retry Logic | No-answer / busy | Retry with backoff |
| U2 | Lead Scoring | Post-call | Score 0–100, hot/warm/cold |
| U3 | Escalation | High-value / complex | Slack + transfer to human |

---

## ⚠️ Critical Architecture Note

> n8n handles **orchestration only**. Real-time audio streaming requires a separate **Node.js / FastAPI WebSocket server**.

```
Lead → n8n → Twilio → [WebSocket Server] ↔ OpenAI + ElevenLabs
                              ↓
                         n8n workflows
                              ↓
                    DB + Follow-ups + Booking
```

---

## 🔐 Security Checklist
- [ ] Rotate all API keys after setup
- [ ] Enable Supabase RLS on all tables  
- [ ] Add call recording consent message (legal requirement)
- [ ] Set GDPR data retention policy on transcripts
- [ ] Restrict n8n webhook endpoints to Twilio IP ranges

---

## 📊 Key Metrics to Track
- **Connection Rate** — calls answered / total calls
- **Conversion Rate** — bookings / answered calls  
- **Avg Call Duration** — target > 2 min for qualified leads
- **Lead Score Distribution** — % hot / warm / cold
- **Retry Success Rate** — % no-answer leads converted on retry
