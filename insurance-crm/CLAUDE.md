# NexusOS — Insurance Agent OS

## Stack
Next.js 15 (app router) · TypeScript strict · React 19 · Supabase · Twilio Voice SDK · Deepgram · Claude API

## Code Style
- Immutable — never mutate, return new objects
- Max 400 lines per file, max 50 lines per function
- No `any` type — all exports fully typed
- Feature-based file organization (not by type)
- API response envelope: `{ success, data, error, metadata }`

## Build
```bash
npm run build && npm run lint && npm run test
```

## Anti-Patterns
- NO RAG for underwriting decisions — deterministic rules engine ONLY
- NO hardcoded carrier rules in React components — all rules in `data/carriers/*.json`
- NO `shell=True` in any subprocess call
- NO secrets in source code — `process.env.KEY` only
- NO direct Twilio API calls from client — all voice/SMS through API routes

## Compliance (Non-Negotiable)
- Encrypt PII columns (DOB, SSN, medical data) with pgcrypto / Supabase Vault
- Audit log ALL data access (pgAudit extension)
- TCPA consent check BEFORE every outbound dial or SMS
- 2-party consent recording disclosure for CA, FL, IL, MD, MA, MT, NH, PA, WA
- Carrier ranking must be deterministic + auditable (no LLM hallucination in underwriting)
