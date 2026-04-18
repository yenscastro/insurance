# NexusOS — Adversarial Edge Case Review

**Scope:** Pre-build architecture review for a 2,500-agent insurance brokerage OS
**Date:** 2026-04-18
**Severity Scale:** 1 = minor UX friction | 5 = data loss / legal exposure / production down
**Total edge cases:** 52 | Severity 5: 10 | Severity 4: 16 | Severity 3: 21 | Severity 2: 5

---

## Critical Pre-Launch Checklist (Top 10 — Must-Fix Before Day 1)

| # | Issue | Section | Severity | Why |
|---|-------|---------|----------|-----|
| 1 | Two-party consent recording disclosure | A5 | 5 | Legal exposure, CA Penal Code 632 fines $5K/violation |
| 2 | TCPA opt-out real-time sync | E4 | 5 | One violation triggers class action, $500-1500/violation |
| 3 | PII/PHI transcript encryption | E2 | 5 | Regulatory requirement, state insurance privacy laws |
| 4 | RLS test suite | E1 | 5 | Automated, CI-gated, 100% pass — PII breach prevention |
| 5 | Medical data audit log | E6 | 5 | Append-only, on all sensitive tables — regulatory audit |
| 6 | Claude hallucination guard on diagnoses | C5 | 5 | E&O liability — fabricated diagnosis = insurance fraud |
| 7 | AI extraction confidence gating | C1 | 5 | No `possible` facts reach the quote engine |
| 8 | Supabase connection pooler via PgBouncer | F4 | 4 | Day-1 production readiness at 2,500 agents |
| 9 | Deepgram enterprise capacity agreement | F2 | 4 | Standard plan won't support 2,500 concurrent streams |
| 10 | Lead lock mechanism | G4 | 3 | Prevents ownership disputes that destroy agent morale |

---

## A. Voice / Dialer Edge Cases

### A1. Browser microphone permission denied
**Scenario:** Agent opens dialer, browser blocks mic. SDK fails silently.
**Severity:** 3
**Mitigation:** Pre-call `navigator.mediaDevices.getUserMedia()` check. Blocking modal with step-by-step unlock instructions. Never show dial button without green mic status. Log `mic_permission_denied` events.

### A2. Twilio Voice SDK WebSocket disconnect mid-call
**Scenario:** Network hiccup drops WebSocket. Agent hears nothing; client is still connected.
**Severity:** 4
**Mitigation:** Subscribe to `device.on('error')` + `connection.on('disconnect')`. Persistent "Reconnecting..." banner. `closeProtection: true`. Use Twilio Conferences (not direct dial) so PSTN leg holds while browser reconnects. Store transcript in IndexedDB. 90s server-side timeout plays "We'll call you back" and ends call if no reconnect.

### A3. Poor audio quality / heavy accent / background noise
**Scenario:** Deepgram returns gibberish. AI extracts wrong medical facts. Wrong carrier quoted. Policy rescinded at claim time.
**Severity:** 5
**Mitigation:** Deepgram `noise_reduction: true` + `smart_format: true`. Per-token confidence threshold (0.75) — show `?` underline on low-confidence words. Every extracted fact gets `confidence` + `source_quote` fields. If underwriting-critical fact has confidence < 0.85, prompt agent with specific clarifying question. Never auto-submit quote on low-confidence extractions.

### A4. Landline voicemail vs human pickup
**Scenario:** No AMD — agent wastes 60s pitching to voicemail. Or AMD misclassifies human as machine and drops the call.
**Severity:** 3 (productivity) / 4 (compliance if drops human)
**Mitigation:** Twilio AMD `MachineDetection: 'DetectMessageEnd'`, timeout 3000ms. Machine → auto voicemail drop (agent pre-records). Human → connect immediately. Never auto-drop `unknown` calls — route to agent as live.

### A5. Call recording in 2-party consent states
**Scenario:** Agent records in California without disclosure. CCPA + Penal Code 632. Fines up to $5,000 per violation.
**Severity:** 5
**Mitigation:** State-consent matrix in lead data model. 2-party states (CA, CT, FL, IL, MD, MA, MI, MO, NH, OR, PA, WA) auto-play legally reviewed disclosure before bridging agent audio. Agent cannot speak until disclosure completes (Twilio `<Pause>` + `<Play>` before `<Dial>`). Log disclosure event with timestamp + call SID. Never let agents manually toggle recording.

### A6. Agent browser tab crashes mid-call
**Scenario:** Chrome OOM-kills tab. PSTN leg stays connected. Client on hold indefinitely.
**Severity:** 3
**Mitigation:** Maintain `active_call_sid` per agent in Supabase. On tab reload, check for active call → "Reconnect?" modal within 3s. Use Twilio Conferences so PSTN leg is held. Pre-crash transcript stored in Supabase Realtime.

### A7. WebRTC firewall on corporate networks
**Scenario:** Enterprise firewall blocks UDP ports. One-way audio or silent failure.
**Severity:** 3
**Mitigation:** Enable Twilio Edge Locations. Force TCP fallback via `forceTURN: true`. "Network Diagnostics" button runs `Twilio.Device.runPreflight()`. Firewall whitelist doc in agent onboarding.

### A8. Safari / Firefox compatibility
**Scenario:** Safari WebRTC issues. Firefox ICE negotiation differences. Echo or no audio.
**Severity:** 2
**Mitigation:** Lock dialer to Chrome/Edge only. Hard browser detection gate at dialer route — non-Chromium = "Download Chrome" CTA. No cross-browser WebRTC polyfills.

---

## B. Transcription Edge Cases

### B1. Deepgram latency spikes
**Scenario:** Streaming latency spikes to 2-3s. Transcript lags 15s. Coaching cues stale.
**Severity:** 3
**Mitigation:** `interim_results: true` for real-time partial display. Extraction on 30s chunks with 5s overlap. Latency monitor — yellow "Transcript delayed" banner if > 5s drift. Secondary pre-warmed WebSocket on standby for failover.

### B2. Medical terminology accuracy
**Scenario:** "metformin" → "met for men". "A1C" → "a one see". Carrier ranking misses diabetes indicator.
**Severity:** 5
**Mitigation:** Deepgram `keywords` feature — 200+ medical terms boosted 3-5x. Versioned keyword list, quarterly clinical review. Extraction prompt includes phonetic variants. Post-extraction medication validation against RxNorm API.

### B3. Multi-speaker cross-talk
**Scenario:** Agent talks over client. Deepgram merges utterances. AI extracts agent's question as client's answer.
**Severity:** 4
**Mitigation:** Twilio dual-channel recording (separate agent/client tracks). Deepgram `multichannel: true`. Extraction prompt: "Extract facts ONLY from Speaker 1 (client). Ignore Speaker 0 (agent)." Tag every fact with `speaker_id`.

### B4. Background noise on client's end
**Scenario:** Client driving, kids screaming. Accuracy drops to 60%.
**Severity:** 3
**Mitigation:** `noise_reduction: true`. Noise-level indicator in UI (RMS amplitude from incoming stream). "Ask client to move to a quieter location" banner if threshold exceeded for 10+ seconds.

### B5. Deepgram rate limits or outage
**Scenario:** Partial Deepgram outage. 500 agents mid-call, all STT fails.
**Severity:** 4
**Mitigation:** Circuit breaker on Deepgram connection (3 retries + exponential backoff). Degraded mode: "Transcription offline — manual mode" banner, manual intake form surfaces, call recording queued for batch transcription. Global `transcription_status` flag via Supabase Realtime prevents 2,500 individual reconnect storms.

### B6. Non-English / code-switching
**Scenario:** Client speaks Spanish mid-sentence. Medical facts in Spanish are lost.
**Severity:** 3
**Mitigation:** Deepgram `detect_language: true`. If Spanish detected, reconnect with `language: 'multi'`. Extraction prompt includes Spanish medical term mappings. Language dropdown on campaign config.

### B7. Phone number / dollar amount transcription
**Scenario:** "twelve/fifteen/nineteen sixty-two" vs "12/15/1962". Application prefill errors.
**Severity:** 2
**Mitigation:** Deepgram `smart_format: true` for number conversion. For PII fields (SSN, DOB), use structured agent input — never rely on transcription.

---

## C. AI Extraction Edge Cases

### C1. Ambiguous condition statements
**Scenario:** "I'm on some pills for my sugar" → AI asserts `Type 2 Diabetes`. Could be supplement or pre-diabetic.
**Severity:** 5
**Mitigation:** Three confidence levels: `confirmed` (client named diagnosis), `indicated` (strong inference, requires verification), `possible` (weak signal). Only `confirmed` facts reach the quote engine. `indicated` and `possible` generate specific agent prompts. Never let `possible` reach ranking.

### C2. Client self-contradiction
**Scenario:** "I quit smoking 3 years ago" at minute 2. "I vape sometimes" at minute 11.
**Severity:** 4
**Mitigation:** Rolling 30s extraction windows + final full-transcript reconciliation. Reconciliation identifies contradictions with both quotes + timestamps. "Conflicts to verify" card surfaces before quote generation. Agent must resolve each conflict with attestation timestamp.

### C3. Claude returns malformed JSON
**Scenario:** Partial JSON, wrong types (`age: "forty-two"` instead of `age: 42`).
**Severity:** 3
**Mitigation:** Zod schema validation on all Claude output. `ExtractionSchema.safeParse()` — failure logs raw output + Zod error, falls back to manual intake form. Use Claude tool-use (function calling) API — enforces schema at API level.

### C4. Claude streaming drops mid-extraction
**Scenario:** Connection drops at 60% through response. Partial object passed to ranking engine.
**Severity:** 3
**Mitigation:** Do NOT use streaming for extraction. Standard (non-streaming) Claude API with 30s timeout. Check `stop_reason === 'end_turn'`. Retry on `max_tokens` or connection drop.

### C5. Extraction hallucination
**Scenario:** Client says "chest pain a couple years back" → AI returns `coronary artery disease, confidence: 0.9`. Fabricated diagnosis.
**Severity:** 5
**Mitigation:** Extraction prompt: "Only extract conditions the client EXPLICITLY NAMED as a medical diagnosis. Do not infer diagnoses from symptom descriptions." Negative examples in prompt. `symptoms_unresolved` flag for symptom-only mentions. Test with 50 adversarial transcript examples before launch.

### C6. Claude API rate limiting at peak
**Scenario:** 9 AM Monday. 800 agents → 2,400-4,000 Claude requests/hour. Tier 2 limit: 2,000 RPM.
**Severity:** 3
**Mitigation:** Server-side BullMQ + Redis request queue. Priority lanes (live calls: high, post-call: low). Cache extraction results (hash-based dedup). Adaptive frequency: extend rolling window from 30s to 60s during high queue. Apply for Anthropic enterprise tier.

---

## D. Carrier Ranking Edge Cases

### D1. Zero carriers match
**Scenario:** 68yo with COPD + CHF + insulin-dependent diabetes. All carriers decline. Empty UI.
**Severity:** 3
**Mitigation:** Explicit `no_match` path: structured response with decline reasons + alternatives (guaranteed-issue, graded-benefit). Auto-route to "declined — alternative products" pipeline stage.

### D2. All carriers tie
**Scenario:** Healthy 35yo non-smoker. All 8 carriers accept at Preferred Plus. No differentiation.
**Severity:** 2
**Mitigation:** Secondary sort: (1) brokerage commission rate, (2) AM Best financial strength, (3) claim payout speed, (4) rider availability. Tie-breaking order configurable per brokerage.

### D3. Carrier rules become stale
**Scenario:** Carrier tightens BMI requirements in Q2. JSON still has Q1 rules. Quoting clients who'll be declined.
**Severity:** 4
**Mitigation:** `rules_effective_date` + `rules_expiry_date` per carrier file. Admin UI with version history. 30-day expiry alert cron. "Quote based on guidelines effective [date]" disclaimer on every quote.

### D4. Boundary conditions
**Scenario:** BMI exactly 40.0. `bmi <= 40` vs floating-point `40.00000001`.
**Severity:** 3
**Mitigation:** Epsilon tolerance on boundary comparisons. `boundary_zone` concept: if value within 5% of threshold, flag as `boundary: true` with "verify with underwriter" note. Integer comparison for age.

### D5. Carrier temporarily not accepting policies
**Scenario:** Carrier hits monthly volume cap. System continues ranking them.
**Severity:** 4
**Mitigation:** `carrier_status: active | paused | sunset` toggle. Instant removal from ranking without deleting rules. Realtime notification to agents with open quotes including paused carrier.

### D6. State-specific availability
**Scenario:** Carrier licensed in 32 states, not Texas. Agent quotes Texas lead with that carrier.
**Severity:** 4
**Mitigation:** `licensed_states: string[]` in carrier schema. First filter step: `carrier.licensed_states.includes(lead.state)`. Full carrier × state test matrix. State field immutable after lead assignment.

### D7. Rider availability differences
**Scenario:** Client wants LTC rider. #1 carrier has no LTC rider.
**Severity:** 2
**Mitigation:** `available_riders` in carrier schema. "Riders needed" input on quote config. Hard filter on required riders.

---

## E. Data & Compliance Edge Cases

### E1. RLS misconfiguration — cross-agent data exposure
**Scenario:** Bug: `auth.uid() = agency_id` instead of `agent_id`. Agent A sees Agent B's contacts.
**Severity:** 5
**Mitigation:** Automated RLS tests using `set_config('request.jwt.claims', ...)` impersonation. CI-gated — 100% pass required. Explicit test cases: "Agent A cannot see Agent B's contacts." Dedicated test database.

### E2. PII in transcripts stored unencrypted
**Scenario:** Supabase breach exposes names, DOBs, SSNs, medical histories in plaintext.
**Severity:** 5
**Mitigation:** Application-layer AES-256-GCM encryption. Keys in KMS (AWS KMS or Vault — NOT .env). Separate PII redaction pass (regex + Claude) replaces SSNs/DOBs with `[REDACTED]` in stored transcript. Original in separately encrypted vault with stricter RLS.

### E3. Agent leaves the brokerage
**Scenario:** Agent deactivated. Their leads/calls/commissions orphaned.
**Severity:** 3
**Mitigation:** Data owned by `agency_id` + `team_id`, not `agent_id`. `created_by_agent_id` is metadata only. Deactivation triggers reassignment workflow. Agency owners always have read access. `reassigned_to_agent_id` field on leads.

### E4. TCPA consent revocation mid-campaign
**Scenario:** Lead texts "STOP". 3 minutes later, agent dials them from the queue. TCPA violation.
**Severity:** 5
**Mitigation:** Opt-out handler updates `sms_consent: false` + `do_not_call: true` + removes from all campaign queues in single atomic transaction. Dialer checks `do_not_call` AT MOMENT OF DIAL — not at campaign load. Nightly DNC list scrub (Gryphon or similar). Build opt-out system BEFORE building dialer.

### E5. Client data deletion request (CCPA)
**Scenario:** California client requests deletion. Data in 3 leads, 12 call records, 2 SMS threads.
**Severity:** 4
**Mitigation:** DSR workflow from day one. Identify all records by phone + email. Redact PII in place (don't hard-delete — referential integrity). Delete transcript content. Log deletion with timestamp. Confirm within 45 days (CCPA requirement). Retain commission records 7 years with PII stripped.

### E6. Medical data access audit trail
**Scenario:** Regulatory audit: "Who accessed Client X's medical transcript on March 15?" No audit log.
**Severity:** 5
**Mitigation:** Postgres trigger on all sensitive tables → append-only `audit_log`: `{table, record_id, action, user_id, timestamp, ip, session_id}`. No UPDATE/DELETE on audit_log even for admins. Separate schema with stricter access. Admin UI with filtering.

### E7. Supabase Realtime drops — coaching panel freezes
**Scenario:** Realtime disconnects. Coaching panel shows stale 3-minute-old data. Agent acts on it.
**Severity:** 3
**Mitigation:** Subscribe to `channel.on('system', ...)`. "Coaching disconnected — reconnecting..." banner immediately. Exponential backoff reconnect. Full state refresh on reconnect (HTTP GET, not event stream catchup). Staleness indicator: dim panel + timestamp if last update > 60s ago.

---

## F. Scale Edge Cases (2,500 Agents)

### F1. Twilio concurrent call limits
**Scenario:** 700 agents dial at 9:05 AM. Default limit: 2,000 concurrent.
**Severity:** 4
**Mitigation:** Request Twilio limit increase (enterprise sales). Server-side concurrency enforcement. Slow auto-dialer at 80% capacity. Monitor live concurrent calls via Twilio `InProgress` API.

### F2. Deepgram concurrent stream limits
**Scenario:** 700 concurrent calls = 700 simultaneous Deepgram streams. Standard plan limits: 100-250.
**Severity:** 4
**Mitigation:** Deepgram enterprise agreement. Route all streams through dedicated Node.js media server (not direct from browser). Stream multiplexing + connection reuse + backpressure.

### F3. Claude API rate limits at scale
**Scenario:** 500 concurrent calls × 3-5 extractions each = 1,000+ RPM during peak.
**Severity:** 3
**Mitigation:** BullMQ queue with priority lanes. Hash-based dedup caching. Adaptive extraction frequency. Anthropic enterprise tier.

### F4. Supabase connection pool exhaustion
**Scenario:** 2,500 sessions × pooled connections → exceed Postgres 500-connection default.
**Severity:** 4
**Mitigation:** PgBouncer in transaction mode (port 6543). Read replica for analytics/dashboard queries. Never run reporting on primary.

### F5. n8n workflow queue backup
**Scenario:** 300 webhook triggers queue up. Default concurrency: 10. SMS delayed 30 minutes.
**Severity:** 3
**Mitigation:** `N8N_CONCURRENCY_PRODUCTION_LIMIT=50`. Queue mode (Redis + BullMQ). Separate priority queues for time-sensitive (SMS) vs batch (commissions).

### F6. Database millions of transcript rows
**Scenario:** 6 months = 15M transcript rows. "Show diabetes calls last 30 days" → full scan → 45s timeout.
**Severity:** 3
**Mitigation:** Index on `(agent_id, created_at)`, `(campaign_id, created_at)`, `(contact_id)`. GIN index on extracted facts JSONB. Partition `transcripts` by month. Archive > 13 months to S3 + Parquet.

### F7. Real-time dashboard for 2,500 sessions
**Scenario:** Live leaderboard every 30s × 2,500 clients = O(n^2) fan-out.
**Severity:** 3
**Mitigation:** Server-side aggregation cron → single JSON snapshot in `dashboard_snapshots` table → one Realtime broadcast per 30s. Individual agent drill-down subscribes separately.

---

## G. Business Logic Edge Cases

### G1. Commission chargeback (free-look period)
**Scenario:** $800 commission earned. Client cancels within 10-day free-look. Dashboard still shows $800. Agent's check is short.
**Severity:** 4
**Mitigation:** Append-only commission ledger: `{policy_id, agent_id, event_type: earned|chargeback|adjusted, amount, date, reason}`. Balance always computed from ledger sum. Agent sees full ledger history. No disputes.

### G2. Brokerage-of-record change mid-policy
**Scenario:** Client moves policy to another brokerage. Commission stream stops. Policy shows "active" incorrectly.
**Severity:** 3
**Mitigation:** `policy_status: active | lapsed | bor_change | surrendered`. BOR notification triggers manual review workflow. Commission ledger stops accruing renewals for `bor_change` policies.

### G3. Agent hierarchy permissions
**Scenario:** Team Lead A sees their 12 agents but not Team Lead B's. Agent promoted to team lead — permissions must change immediately.
**Severity:** 3
**Mitigation:** RBAC via `agent_roles` table + Postgres function `get_accessible_agent_ids(viewer_id)`. All RLS policies call this function. Role change = immediate effect, no re-login.

### G4. Lead ownership dispute
**Scenario:** Import bug: same lead in Agent A and Agent B's queues. Both dial. Both quote. Commission dispute.
**Severity:** 3
**Mitigation:** `lead_lock` mechanism: `UPDATE contacts SET locked_by = ?, locked_at = NOW() WHERE id = ? AND locked_by IS NULL`. 0 rows = locked by another agent → "Being worked by [Agent Name]" message. Auto-expire after 24h. Phone number dedup on import.

### G5. Campaign overlap
**Scenario:** Same lead in 2 campaigns. 6 calls in 3 days. Client complains to state insurance commissioner.
**Severity:** 4
**Mitigation:** Cross-campaign dedup at creation time. Minimum contact frequency rule: max 3 dial attempts per 7 days across ALL campaigns. Enforced at dial-time.

### G6. Quote expiration
**Scenario:** Quote generated Monday. Carrier raises rates Wednesday. Agent submits Monday rate Thursday. Declined.
**Severity:** 3
**Mitigation:** `expires_at` on every quote (default 48h). Expired quote → "Quote expired — regenerate?" CTA. Block application submission on expired quotes. Log rate version on every submission.
