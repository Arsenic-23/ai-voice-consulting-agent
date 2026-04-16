# System Architecture

## Full Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                     ENTRY POINTS                            │
│                                                             │
│   New Lead (DB/Webhook)          Incoming Call (Twilio)     │
└──────────────┬──────────────────────────┬───────────────────┘
               │                          │
               ▼                          ▼
┌─────────────────────┐      ┌─────────────────────────┐
│ Workflow 01          │      │ Workflow 02              │
│ Outbound Calling     │      │ Inbound Call Handling    │
│                      │      │                          │
│ Validate → Twilio    │      │ Lookup Lead → TwiML      │
│ Call API → Wait      │      │ Stream to AI Server      │
└──────────┬──────────┘      └──────────┬───────────────┘
           │                            │
           └────────────┬───────────────┘
                        │
                        ▼
           ┌────────────────────────┐
           │   TWILIO MEDIA STREAM  │
           │  (Real-Time WebSocket) │
           │                        │
           │  Audio In → STT        │
           │  (Whisper / Deepgram)  │
           └──────────┬─────────────┘
                      │ transcript chunk
                      ▼
        ┌─────────────────────────────┐
        │ Workflow 03                  │
        │ AI Conversation Processing   │
        │                              │
        │ Load History → GPT-4o →     │
        │ Parse Intent → ElevenLabs → │
        │ Send Audio Back              │
        └──────────────┬───────────────┘
                       │ intent detected
                       ▼
        ┌──────────────────────────────┐
        │ Intent Router (Switch Node)  │
        │                              │
        │ BOOK_MEETING ──► Workflow 05 │
        │ FOLLOW_UP    ──► Workflow 04 │
        │ ESCALATE     ──► Util: Escal.│
        │ NOT_INTERESTED ► Archive DB  │
        │ CONTINUE     ──► Next turn   │
        └──────────────────────────────┘
               │              │
               ▼              ▼
  ┌────────────────┐  ┌────────────────────┐
  │ Workflow 04    │  │ Workflow 05         │
  │ Follow-Up      │  │ Booking & Calendar  │
  │                │  │                     │
  │ SMS/WhatsApp/  │  │ Google Calendar →  │
  │ Email by       │  │ SMS + Email confirm │
  │ outcome        │  └──────────┬──────────┘
  └──────┬─────────┘             │
         │                       │
         └───────────┬───────────┘
                     ▼
          ┌────────────────────┐
          │   SUPABASE DB      │
          │                    │
          │ leads              │
          │ call_logs          │
          │ appointments       │
          │ workflow_logs      │
          └────────────────────┘
```

---

## Component Responsibilities

| Component | Role | Technology |
|-----------|------|-----------|
| n8n | Workflow orchestration, routing, DB ops | n8n (self-hosted) |
| AI Realtime Server | WebSocket audio streaming bridge | Node.js / FastAPI |
| Twilio | Voice calls, SMS, WhatsApp | Twilio API |
| OpenAI GPT-4o | Conversation intelligence | OpenAI API |
| ElevenLabs | Voice synthesis (TTS) | ElevenLabs API |
| Supabase | Database, RLS, realtime | Supabase (Postgres) |
| Google Calendar | Appointment slots + events | Google Calendar API |
| Slack | Internal alerts on escalations | Slack Webhooks |

---

## Latency Budget (per conversation turn)

```
Lead speaks       :   0ms  (starts)
STT transcript    : ~400ms  (Deepgram streaming)
n8n webhook recv  :  ~50ms
DB load context   : ~100ms
GPT-4o response   : ~800ms (streaming first token)
ElevenLabs TTS    : ~400ms (first audio chunk)
Audio delivery    :  ~50ms
─────────────────────────────
Total perceived   : ~1.8 sec  ✅ under 2s target
```

---

## Scaling Considerations

- **Queue outbound calls** — use n8n's built-in queue or a Redis queue to avoid Twilio rate limits (1 call/sec default)
- **Session state** — store conversation history in Supabase, NOT in-memory (n8n is stateless between webhook calls)
- **Parallel calls** — each active call has its own `call_sid` as the unique session key
- **AI Server scaling** — deploy on Railway / Fly.io with auto-scaling; each call = 1 persistent WebSocket connection

---

## Failure Recovery

| Failure | Recovery |
|---------|---------|
| Twilio call fails | Log → retry workflow with backoff |
| OpenAI timeout | Return fallback phrase: "Could you repeat that?" |
| ElevenLabs down | Fallback to Twilio's built-in `<Say>` TwiML |
| DB write fails | Log to workflow_logs, retry once |
| Calendar full | Offer 3 alternative slots |
| n8n crashes mid-flow | Error trigger node → Slack alert |
