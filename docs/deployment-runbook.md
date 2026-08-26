# Deployment Runbook — Garcar Enterprise

## Pipeline Architecture

```
push to main
    │
    ▼
┌─────────────────────────────────────────────┐
│  GATE 1: Preflight Validation               │
│  • Lint + static analysis                   │
│  • Unit tests                               │
│  • Anchor last known-good SHA               │
│  • Notify Linear: preflight started         │
└────────────────┬────────────────────────────┘
                 │ pass
                 ▼
┌─────────────────────────────────────────────┐
│  GATE 2: Staging Deploy + Smoke Test        │
│  • Deploy to staging environment            │
│  • Supabase health check                    │
│  • Stripe connectivity check                │
│  • HTTP smoke test on staging URL           │
└────────┬────────────────────┬───────────────┘
         │ pass               │ fail
         ▼                   ▼
┌──────────────────┐  ┌──────────────────────┐
│  GATE 3:         │  │  AUTO-ROLLBACK       │
│  Production      │  │  • Revert to last    │
│  Promote         │  │    known-good SHA    │
│  • Tag SHA       │  │  • Linear: urgent    │
│  • Deploy prod   │  │    issue created     │
│  • Notion log    │  │  • Notion log entry  │
│  • Linear issue  │  │  • GitHub issue      │
└──────────────────┘  └──────────────────────┘
```

## Required Secrets

| Secret | Purpose | Required |
|---|---|---|
| `GITHUB_TOKEN` | Workflow permissions (auto-provided) | ✅ Always |
| `LINEAR_API_KEY` | Issue creation + deployment tracking | Recommended |
| `SUPABASE_URL` | Database health checks | If using Supabase |
| `SUPABASE_SERVICE_ROLE_KEY` | Supabase auth | If using Supabase |
| `STRIPE_SECRET_KEY` | Billing connectivity check | If using Stripe |
| `NOTION_TOKEN` | Deployment log entries | If using Notion |
| `VERCEL_TOKEN` | Vercel deploy (if applicable) | If using Vercel |

## Authorization Boundaries

| Action | Requires |
|---|---|  
| Push to main → staging | Automatic |
| Staging passes → production | Automatic (gated on smoke test) |
| Staging fails → rollback | Automatic |
| Force rollback | Manual `workflow_dispatch` with `force_rollback: true` |
| Production deploy | Staging MUST pass first (no bypass) |
| Stripe key rotation | Separate manual approval |
| Database migrations | Separate manual approval |
| Merge to main | PR review or `auto-approve-merge.yml` |

## Rollback Procedure

### Automatic (triggered by smoke test failure)
The pipeline rolls back automatically. A GitHub issue, Linear issue (Urgent priority), and Notion entry are all created.

### Manual force rollback
```bash
gh workflow run staging-gate.yml \
  --repo Garrettc123/autonomous-zero-touch-deploy \
  --field force_rollback=true
```

## Health Monitor
Runs every 4 hours via `health-monitor.yml`.
Checks: Supabase · Stripe · Linear · Notion
Degraded systems → GitHub issue with label `health-check`.

## Deploy Targets Detected
The pipeline auto-detects deploy target from config files:
- `vercel.json` → Vercel (requires `VERCEL_TOKEN`)
- `fly.toml` → Fly.io
- `railway.toml` → Railway
- No config → simulation mode (smoke tests still run)
