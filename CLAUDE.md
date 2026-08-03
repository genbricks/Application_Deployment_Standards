# Application Deployment Standards

This repository contains GenBricks deployment standards and Claude Code skill files for cost-efficient AWS deployments.

## Available Skills

### `/aws-deploy`
The primary deployment skill. Guides you through deploying any application as a `<subdomain>.genbricks.io` subdomain using the most cost-efficient AWS pattern.

**Supports:**
- Static sites (S3 + CloudFront) — $0/month
- Python backends (Lambda container, arm64) — $0/month
- Node.js backends (Lambda zip) — $0/month
- Full-stack apps (SPA + API, two subdomains) — $0/month
- Next.js SSR (Vercel free tier) — $0/month

**Includes:**
- Neon PostgreSQL database setup, connection strategies, and branching
- JWT + bcrypt auth implementation (Node.js and Python)
- Data API CRUD patterns with pagination
- 10 hard-won deployment gotchas
- Teardown/decommission checklist
- All shared infrastructure IDs (Route53, ACM, CloudFront policies)

## Usage

1. Clone this repo
2. The `.claude/commands/aws-deploy.md` skill file is automatically available in Claude Code sessions run from this directory
3. Invoke with `/aws-deploy` in any Claude Code session
4. To make it available globally, copy the skill file to `~/.claude/commands/aws-deploy.md`

## Infrastructure

- **AWS Account:** 920653892745 (us-east-1)
- **Domain:** genbricks.io (Porkbun, expires Dec 2026)
- **Database:** Neon PostgreSQL (free tier)
- **Total monthly cost:** ~$0.50 (Route53 hosted zone fee only)
