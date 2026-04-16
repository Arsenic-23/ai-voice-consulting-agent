-- ============================================
-- AI Calling Agent — Supabase Schema
-- Run this in the Supabase SQL Editor
-- ============================================

-- Enable UUID generation
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- ============================================
-- LEADS TABLE
-- ============================================
CREATE TABLE IF NOT EXISTS leads (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name          TEXT,
  phone         TEXT NOT NULL UNIQUE,
  email         TEXT,
  company       TEXT,
  source        TEXT DEFAULT 'manual',      -- website | manual | import | inbound-call
  status        TEXT DEFAULT 'new',         -- new | contacted | hot | warm | cold | booked | lost
  score         INT DEFAULT 0,              -- 0–100
  budget        TEXT,
  business_size TEXT,
  problem       TEXT,
  urgency       TEXT DEFAULT 'unknown',     -- low | medium | high | unknown
  retry_count   INT DEFAULT 0,
  last_called   TIMESTAMPTZ,
  notes         TEXT,
  created_at    TIMESTAMPTZ DEFAULT NOW(),
  updated_at    TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- CALL LOGS TABLE
-- ============================================
CREATE TABLE IF NOT EXISTS call_logs (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  lead_id       UUID REFERENCES leads(id) ON DELETE SET NULL,
  call_sid      TEXT UNIQUE,
  direction     TEXT DEFAULT 'outbound',    -- inbound | outbound
  started_at    TIMESTAMPTZ DEFAULT NOW(),
  ended_at      TIMESTAMPTZ,
  duration_sec  INT DEFAULT 0,
  transcript    TEXT,                       -- JSON array of {role, content} turns
  sentiment     TEXT DEFAULT 'neutral',     -- positive | neutral | negative
  outcome       TEXT DEFAULT 'initiated',   -- initiated | answered | no-answer | busy | failed | booked | follow-up | not-interested | escalated
  recording_url TEXT,
  notes         TEXT,
  created_at    TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- APPOINTMENTS TABLE
-- ============================================
CREATE TABLE IF NOT EXISTS appointments (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  lead_id         UUID REFERENCES leads(id) ON DELETE CASCADE,
  call_log_id     UUID REFERENCES call_logs(id) ON DELETE SET NULL,
  scheduled_at    TIMESTAMPTZ NOT NULL,
  duration_min    INT DEFAULT 30,
  status          TEXT DEFAULT 'pending',   -- pending | confirmed | cancelled | completed | no-show
  meeting_link    TEXT,
  google_event_id TEXT,
  reminder_sent   BOOLEAN DEFAULT FALSE,
  notes           TEXT,
  created_at      TIMESTAMPTZ DEFAULT NOW(),
  updated_at      TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- WORKFLOW LOGS TABLE (debug / audit trail)
-- ============================================
CREATE TABLE IF NOT EXISTS workflow_logs (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  workflow_name  TEXT NOT NULL,
  lead_id        UUID,
  call_log_id    UUID,
  step           TEXT,
  status         TEXT DEFAULT 'success',    -- success | error | skipped
  input          JSONB,
  output         JSONB,
  error_message  TEXT,
  created_at     TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- COACHING REPORTS TABLE
-- ============================================
CREATE TABLE IF NOT EXISTS coaching_reports (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  week_start      DATE NOT NULL,
  week_end        DATE NOT NULL,
  total_calls     INT DEFAULT 0,
  conversion_rate NUMERIC(5,2),
  top_objections  JSONB,
  missed_opps     JSONB,
  best_phrases    JSONB,
  worst_phrases   JSONB,
  recommendations TEXT,
  raw_analysis    TEXT,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- ============================================
-- INDEXES
-- ============================================
CREATE INDEX IF NOT EXISTS idx_leads_phone      ON leads(phone);
CREATE INDEX IF NOT EXISTS idx_leads_status     ON leads(status);
CREATE INDEX IF NOT EXISTS idx_leads_score      ON leads(score DESC);
CREATE INDEX IF NOT EXISTS idx_call_logs_lead   ON call_logs(lead_id);
CREATE INDEX IF NOT EXISTS idx_call_logs_sid    ON call_logs(call_sid);
CREATE INDEX IF NOT EXISTS idx_call_logs_outcome ON call_logs(outcome);
CREATE INDEX IF NOT EXISTS idx_appointments_lead ON appointments(lead_id);
CREATE INDEX IF NOT EXISTS idx_workflow_logs_lead ON workflow_logs(lead_id);

-- ============================================
-- ANALYTICS VIEWS
-- ============================================

-- Calls per hour
CREATE OR REPLACE VIEW v_calls_per_hour AS
SELECT
  DATE_TRUNC('hour', started_at) AS hour,
  COUNT(*) AS total_calls,
  COUNT(*) FILTER (WHERE outcome NOT IN ('no-answer','failed','initiated')) AS connected,
  COUNT(*) FILTER (WHERE outcome = 'booked') AS booked
FROM call_logs
GROUP BY hour
ORDER BY hour DESC;

-- Lead pipeline summary
CREATE OR REPLACE VIEW v_lead_pipeline AS
SELECT
  status,
  COUNT(*) AS count,
  AVG(score) AS avg_score,
  COUNT(*) FILTER (WHERE last_called IS NOT NULL) AS ever_called
FROM leads
GROUP BY status;

-- Conversion funnel
CREATE OR REPLACE VIEW v_conversion_funnel AS
SELECT
  COUNT(*) AS total_leads,
  COUNT(*) FILTER (WHERE status != 'new') AS contacted,
  COUNT(*) FILTER (WHERE status IN ('hot','warm')) AS qualified,
  COUNT(*) FILTER (WHERE status = 'booked') AS booked,
  COUNT(*) FILTER (WHERE status = 'lost') AS lost,
  ROUND(
    COUNT(*) FILTER (WHERE status = 'booked') * 100.0 /
    NULLIF(COUNT(*) FILTER (WHERE status != 'new'), 0), 2
  ) AS conversion_rate_pct
FROM leads;

-- ============================================
-- ROW LEVEL SECURITY (enable after setup)
-- ============================================
-- ALTER TABLE leads         ENABLE ROW LEVEL SECURITY;
-- ALTER TABLE call_logs     ENABLE ROW LEVEL SECURITY;
-- ALTER TABLE appointments  ENABLE ROW LEVEL SECURITY;
-- ALTER TABLE workflow_logs ENABLE ROW LEVEL SECURITY;
