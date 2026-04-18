# Migrating NexusOS to OpenClaw

## What is OpenClaw?

OpenClaw is an open-source AI coding agent (similar to Claude Code / Cursor / Codex). It provides:
- Terminal-based AI coding assistant
- Project context awareness via CLAUDE.md / context files
- Tool use (file editing, bash, web search)
- Agent spawning for parallel work

## Step-by-Step Migration

### 1. Install OpenClaw

```bash
# Follow official installation at: https://github.com/openclawai/openclaw
# macOS:
brew install openclaw
# or
npm install -g @openclaw/cli
```

### 2. Clone the NexusOS Repository

```bash
git clone https://github.com/yenscastro/insurance.git
cd nexus-os
```

### 3. Set Up Your Environment

Each teammate creates their OWN API keys. Never share key values.

```bash
cp .env.example .env.local
```

Open `.env.local` and fill in each key. See `nexus-library.yaml` for:
- What each key does
- Where to create it (direct links to each console)

Required keys:
- `NEXT_PUBLIC_SUPABASE_URL` + `NEXT_PUBLIC_SUPABASE_ANON_KEY` + `SUPABASE_SERVICE_ROLE_KEY`
- `TWILIO_ACCOUNT_SID` + `TWILIO_AUTH_TOKEN` + `TWILIO_PHONE_NUMBER` + `TWILIO_TWIML_APP_SID`
- `DEEPGRAM_API_KEY`
- `ANTHROPIC_API_KEY`

### 4. Point OpenClaw at Project Context

OpenClaw reads project rules from context files:

```bash
# These files already exist in the repo:
# _CONTEXT.md  — what this project is, stack, structure
# CLAUDE.md    — code style rules, anti-patterns, compliance requirements
# nexus-library.yaml — full tooling manifest

# OpenClaw should auto-detect CLAUDE.md. If not:
openclaw config set context_file ./CLAUDE.md
```

### 5. Understand the Skill Mapping

Claude Code skills → OpenClaw equivalents:

| Claude Code Skill | OpenClaw Equivalent | Notes |
|-------------------|---------------------|-------|
| tdd-workflow | Built-in test support | Write tests first, implement to pass |
| api-design | N/A (use CLAUDE.md rules) | REST patterns enforced via code review |
| frontend-design | N/A (use CLAUDE.md rules) | UI patterns in context file |
| council | Manual — use OpenRouter API | Multi-model opinions via API calls |
| verify | Manual — run verification scripts | `npm run test && npm run lint` |
| diagnose | Built-in linting | `npm run lint` covers most |

### 6. Shared Database Access

The team shares ONE Supabase project. The project owner:
1. Creates the Supabase project at https://supabase.com/dashboard
2. Invites teammates via Organization → Members → Invite
3. Each teammate gets their own `SUPABASE_SERVICE_ROLE_KEY` from the dashboard
4. The `NEXT_PUBLIC_SUPABASE_URL` and `NEXT_PUBLIC_SUPABASE_ANON_KEY` are the SAME for everyone

### 7. First-Run Checklist

After setup, verify everything works:

```bash
# 1. Start Supabase locally
npx supabase start
npx supabase db push

# 2. Start the dev server
npm run dev

# 3. Verify Supabase connection
# Open http://localhost:3000 — should load without DB errors

# 4. Test Twilio (optional — needs phone number)
# Navigate to /cockpit — softphone should initialize

# 5. Test Deepgram (optional — needs active call)
# Start a call → verify transcription appears

# 6. Test Claude extraction (optional)
# During a call, mention "I have diabetes" → verify fact extraction panel updates
```

### 8. Git Workflow

```bash
# Branch naming
git checkout -b feature/carrier-ranking-engine
git checkout -b fix/twilio-sdk-initialization

# Commit format
git commit -m "feat(cockpit): add live carrier ranking panel"
git commit -m "fix(twilio): handle browser permissions for microphone"

# Types: feat, fix, refactor, docs, test, chore
# Scopes: cockpit, pipeline, carriers, dashboard, api, auth
```

### 9. Key Files to Read First

In order of importance:
1. `CLAUDE.md` — code rules and compliance requirements
2. `_CONTEXT.md` — project overview and structure
3. `nexus-library.yaml` — full tooling and API key manifest
4. `ENGINEERING_CONTEXT.html` — comprehensive engineering brief (open in browser)
5. `data/carriers/` — carrier knowledge base JSON files
6. `lib/carriers/ranking.ts` — the core ranking engine (the moat)

### 10. Communication

- All code goes through PR review before merge
- Use the repo's issue tracker for task assignment
- Tag issues with: `carrier`, `dialer`, `coaching`, `pipeline`, `compliance`, `infra`
