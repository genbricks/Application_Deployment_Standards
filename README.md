# Application Deployment Standards

> GenBricks deployment standards for cost-efficient AWS hosting. Five patterns, $0.50/month total infrastructure.

## What This Is

A Claude Code skill file (`/aws-deploy`) that guides team members through deploying any application — static sites, Python backends, Node.js APIs, or full-stack apps — as a subdomain of `genbricks.io` using the most cost-efficient AWS pattern possible.

## Quick Start

### Option 1: Use from this repo
```bash
git clone https://github.com/genbricks/Application_Deployment_Standards.git
cd Application_Deployment_Standards
# Open Claude Code — /aws-deploy is now available
```

### Option 2: Install globally
```bash
cp .claude/commands/aws-deploy.md ~/.claude/commands/aws-deploy.md
# /aws-deploy is now available in all Claude Code sessions
```

### Option 3: Reference directly
Read `.claude/commands/aws-deploy.md` as a standalone deployment guide — all CLI commands, patterns, and gotchas are documented inline.

## Deployment Patterns

| Pattern | What | Stack | Cost |
|---------|------|-------|------|
| **A** | Static Site | S3 + CloudFront | $0/mo |
| **B** | Python Backend | Lambda Container (arm64) + API Gateway | $0/mo |
| **C** | Node.js Backend | Lambda Zip + API Gateway | $0/mo |
| **D** | Full-Stack | Pattern A + B/C (two subdomains) | $0/mo |
| **E** | Next.js SSR | Vercel (free tier) | $0/mo |

## Database

All projects use **Neon PostgreSQL** (free tier) instead of AWS RDS.

| Resource | GenBricks | Conventional | Savings |
|----------|-----------|-------------|---------|
| Frontend | S3 + CloudFront | EC2 + nginx | $8.50/mo |
| Compute | Lambda (arm64) | EC2 t3.micro | $8.50/mo |
| API Gateway | HTTP API | REST API | ~70% |
| Database | Neon free | RDS db.t3.micro | $15/mo |
| **Total** | **$0.50/mo** | **$32.50/mo** | **$32/mo** |

## What's Covered

- Step-by-step deployment commands for each pattern
- Neon PostgreSQL connection strategies per runtime (serverless driver, pg, SQLAlchemy)
- JWT + bcrypt auth implementation (Node.js and Python)
- Cross-subdomain cookie auth on `.genbricks.io`
- Data API CRUD patterns with pagination
- Database migration tooling (Alembic / raw SQL)
- Neon branching strategy (main → production)
- 10 hard-won deployment gotchas
- Teardown/decommission checklist
- All shared infrastructure reference IDs

## Contributing

Update the skill file at `.claude/commands/aws-deploy.md` and submit a PR. Keep the format compatible with Claude Code's skill system (YAML frontmatter + markdown body).

---

*GenBricks LLC — "Automation with AI Systems"*
