# NexusOS — Insurance Agent OS

> Vertical-integrated insurance agent operating system. AI-coached dialer + CRM pipeline + carrier ranking + auto-quote. NOT a generic CRM — built for life insurance brokerages with independent agent networks.

## Stack

| Layer | Technology |
|-------|-----------|
| Framework | Next.js 15 (App Router, React 19, TypeScript strict) |
| UI | shadcn/ui + Tailwind CSS v4 + Framer Motion |
| Database | Supabase (Postgres + RLS + Realtime + Edge Functions + pgvector) |
| Auth | Supabase Auth → SSO (Microsoft Entra / Google Workspace) at scale |
| Voice | Twilio Voice SDK (browser softphone) |
| Transcription | Deepgram Streaming API |
| AI | Claude API (Sonnet, streaming tool-use) |
| Workflows | n8n (self-hosted) |
| Hosting | Vercel + VPS (Traefik + Docker) |

## File Structure

```
nexus-os/
├── app/                    # Next.js app router
│   ├── (auth)/             # Login / signup
│   ├── (dashboard)/        # Main app layout
│   │   ├── pipeline/       # Pipeline board (drag-drop stages)
│   │   ├── cockpit/        # AI call cockpit (dialer + transcript + coaching)
│   │   ├── contacts/       # Contact management
│   │   ├── dashboard/      # KPIs, commissions, leaderboard
│   │   ├── campaigns/      # Lead list + campaign management
│   │   ├── quotes/         # Manual quote tool
│   │   └── settings/       # Agent / agency settings
│   └── api/                # API routes
│       ├── twilio/         # Voice webhooks
│       ├── deepgram/       # STT proxy
│       ├── ai/             # Claude extraction endpoint
│       └── carriers/       # Ranking + quoting
├── components/             # React components (pipeline/, cockpit/, dashboard/, ui/)
├── lib/                    # Business logic (carriers/, twilio/, deepgram/, ai/, supabase/)
├── supabase/migrations/    # SQL migration files
├── data/carriers/          # JSON carrier knowledge base
└── package.json
```

## Quick Start

```bash
git clone [REPO_URL] && cd nexus-os
cp .env.example .env.local   # Fill in API keys (see nexus-library.yaml)
npm install
npx supabase start && npx supabase db push
npm run dev
```

## Active Toolset

**Skills:** tdd-workflow, api-design, backend-patterns, frontend-design, docker-patterns, deployment-patterns, security-review, council, verify, diagnose
**MCPs:** Claude Preview, Scheduled Tasks, Google Drive

## Status

Phase 0 — Scaffolding (April 2026). Engineering context doc complete. Awaiting carrier API credentials and LeadChampPro login from client.

> Relationships: `graph_manager.py query --depends-on insurance-crm`
