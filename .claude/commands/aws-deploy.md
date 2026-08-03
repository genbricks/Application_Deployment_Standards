---
description: Deploy any application (static, Python, Node.js, full-stack) as a subdomain of genbricks.io using the most cost-efficient AWS pattern
---

# GenBricks AWS Deployment Playbook

Deploy applications as `<subdomain>.genbricks.io` on AWS account **920653892745** (us-east-1).
Total infrastructure cost target: **< $1/month** (Route53 only; S3+CloudFront+Lambda = free tier).

---

## STEP 0 — Choose Your Deployment Pattern

Ask the user what they are deploying. Pick the matching pattern:

| What you're deploying | Pattern | Monthly cost |
|---|---|---|
| Static HTML/CSS/JS site | [Pattern A](#pattern-a--static-site-s3--cloudfront) | $0 |
| React / Vue / Next.js (static export) / Vite SPA | [Pattern A](#pattern-a--static-site-s3--cloudfront) | $0 |
| Python backend (FastAPI / Flask / Django) | [Pattern B](#pattern-b--python-backend-lambda-container) | $0 |
| Node.js backend (Express / Hono) | [Pattern C](#pattern-c--nodejs-backend-lambda-zip) | $0 |
| Full-stack (SPA frontend + API backend) | [Pattern D](#pattern-d--full-stack-spa--api) | $0 |
| Next.js with SSR / ISR | [Pattern E](#pattern-e--nextjs-ssr-via-vercel) | $0 (Vercel free) |

**RULE: Never suggest EC2.** EC2 t3.micro = $8.50/month. Lambda = $0/month at GenBricks scale. All legacy EC2 backends have been terminated.

---

## Shared Infrastructure Reference

These resources already exist. **Do not recreate them.**

```
AWS Account:        920653892745
Region:             us-east-1
Route53 Zone:       Z05133172TAH6LQX4R71F (genbricks.io)
ACM Wildcard Cert:  *.genbricks.io, genbricks.io, www.genbricks.io
                    (DNS-validated, us-east-1 — required for CloudFront)
CF Hosted Zone ID:  Z2FDTNDATAQYW2 (AWS global constant for alias targets)

Shared CloudFront Policy IDs:
  Cache (static):   658327ea-f89d-4fab-a63d-7e88639e58f6  (CachingOptimized)
  Cache (API):      4135ea2d-6df8-44a3-9df3-4b5a84be39ad  (CachingDisabled)
  Origin Req (API): 216adef6-5c7f-47e4-b989-5492eafa07d3  (AllViewer)
  Response Headers: 67f7725c-6f97-4210-82d7-5512b31e9d03  (SecurityHeadersPolicy)

Database:           Neon PostgreSQL (external, not AWS — free tier)
Domain registrar:   Porkbun (genbricks.io, expires Dec 25, 2026)
```

---

## Pattern A — Static Site (S3 + CloudFront)

**Use for:** HTML/CSS/JS, React/Vue/Vite SPAs (after `npm run build`), Next.js static export, Hugo, Jekyll, any pre-built frontend.

**Cost: $0/month** (S3 + CloudFront free tier covers millions of requests).

### A1. Create the S3 bucket

```bash
SUBDOMAIN="myapp"
BUCKET="${SUBDOMAIN}-frontend"

aws s3api create-bucket \
  --bucket "$BUCKET" \
  --region us-east-1

aws s3api put-public-access-block \
  --bucket "$BUCKET" \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

### A2. Create CloudFront Origin Access Control (OAC)

Reuse an existing OAC if one is available for the project, or create a new one:

```bash
OAC_ID=$(aws cloudfront create-origin-access-control \
  --origin-access-control-config \
    Name="${SUBDOMAIN}-oac",\
Description="OAC for ${SUBDOMAIN}.genbricks.io",\
SigningProtocol=sigv4,\
SigningBehavior=always,\
OriginAccessControlOriginType=s3 \
  --query 'OriginAccessControl.Id' --output text)

echo "OAC ID: $OAC_ID"
```

### A3. Create CloudFront distribution

Look up the ACM certificate ARN first:

```bash
CERT_ARN=$(aws acm list-certificates --query \
  "CertificateSummaryList[?DomainName=='genbricks.io'].CertificateArn" \
  --output text)
```

Create a distribution config JSON and deploy:

```bash
cat > /tmp/cf-dist-config.json << 'DISTEOF'
{
  "CallerReference": "SUBDOMAIN-TIMESTAMP",
  "Aliases": { "Quantity": 1, "Items": ["SUBDOMAIN.genbricks.io"] },
  "DefaultRootObject": "index.html",
  "Origins": {
    "Quantity": 1,
    "Items": [{
      "Id": "S3-BUCKET",
      "DomainName": "BUCKET.s3.us-east-1.amazonaws.com",
      "OriginAccessControlId": "OAC_ID_HERE",
      "S3OriginConfig": { "OriginAccessIdentity": "" }
    }]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "S3-BUCKET",
    "ViewerProtocolPolicy": "redirect-to-https",
    "AllowedMethods": { "Quantity": 2, "Items": ["GET", "HEAD"] },
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "ResponseHeadersPolicyId": "67f7725c-6f97-4210-82d7-5512b31e9d03",
    "Compress": true
  },
  "CustomErrorResponses": {
    "Quantity": 2,
    "Items": [
      { "ErrorCode": 403, "ResponsePagePath": "/index.html", "ResponseCode": "200", "ErrorCachingMinTTL": 10 },
      { "ErrorCode": 404, "ResponsePagePath": "/index.html", "ResponseCode": "200", "ErrorCachingMinTTL": 10 }
    ]
  },
  "ViewerCertificate": {
    "ACMCertificateArn": "CERT_ARN_HERE",
    "SSLSupportMethod": "sni-only",
    "MinimumProtocolVersion": "TLSv1.2_2021"
  },
  "HttpVersion": "http2and3",
  "PriceClass": "PriceClass_100",
  "Enabled": true,
  "Comment": "SUBDOMAIN.genbricks.io static site"
}
DISTEOF

# Replace placeholders
sed -i '' \
  -e "s/SUBDOMAIN/${SUBDOMAIN}/g" \
  -e "s/BUCKET/${BUCKET}/g" \
  -e "s/OAC_ID_HERE/${OAC_ID}/g" \
  -e "s|CERT_ARN_HERE|${CERT_ARN}|g" \
  -e "s/TIMESTAMP/$(date +%s)/g" \
  /tmp/cf-dist-config.json

CF_DIST_ID=$(aws cloudfront create-distribution \
  --distribution-config file:///tmp/cf-dist-config.json \
  --query 'Distribution.Id' --output text)

CF_DOMAIN=$(aws cloudfront get-distribution --id "$CF_DIST_ID" \
  --query 'Distribution.DomainName' --output text)

echo "CloudFront Distribution: $CF_DIST_ID"
echo "CloudFront Domain: $CF_DOMAIN"
```

### A4. Set S3 bucket policy (allow CloudFront OAC)

```bash
cat > /tmp/bucket-policy.json << BPEOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "AllowCloudFrontServicePrincipalReadOnly",
    "Effect": "Allow",
    "Principal": { "Service": "cloudfront.amazonaws.com" },
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::${BUCKET}/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::920653892745:distribution/${CF_DIST_ID}"
      }
    }
  }]
}
BPEOF

aws s3api put-bucket-policy --bucket "$BUCKET" --policy file:///tmp/bucket-policy.json
```

### A5. Create Route53 DNS record

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z05133172TAH6LQX4R71F \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "'"${SUBDOMAIN}"'.genbricks.io",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z2FDTNDATAQYW2",
          "DNSName": "'"${CF_DOMAIN}"'",
          "EvaluateTargetHealth": false
        }
      }
    }]
  }'
```

### A6. Build and deploy your site

**For plain HTML/CSS/JS:**
```bash
aws s3 sync ./public/ s3://${BUCKET}/ --delete
aws cloudfront create-invalidation --distribution-id ${CF_DIST_ID} --paths "/*"
```

**For React / Vite / Vue (npm-based):**
```bash
cd frontend
npm install && npm run build
aws s3 sync dist/ s3://${BUCKET}/ --delete
aws cloudfront create-invalidation --distribution-id ${CF_DIST_ID} --paths "/*"
```

**For Next.js static export:**
Add `output: "export"` to `next.config.js`, then:
```bash
npm run build
aws s3 sync out/ s3://${BUCKET}/ --delete
aws cloudfront create-invalidation --distribution-id ${CF_DIST_ID} --paths "/*"
```

**Cache control best practice (split HTML vs assets):**
```bash
# HTML — short cache (5 min) so updates propagate fast
aws s3 sync dist/ s3://${BUCKET}/ --exclude "*" --include "*.html" \
  --cache-control "max-age=300" --content-type "text/html"

# Everything else — long cache (1 day), hashed filenames handle busting
aws s3 sync dist/ s3://${BUCKET}/ --delete \
  --cache-control "max-age=86400" --exclude "*.html"

aws cloudfront create-invalidation --distribution-id ${CF_DIST_ID} --paths "/*"
```

---

## Pattern B — Python Backend (Lambda Container)

**Use for:** FastAPI, Flask, Django backends that need a database, file processing, AI/ML endpoints.

**Cost: $0/month** (Lambda free tier = 1M requests + 400,000 GB-seconds).

### B1. Project structure

```
myapp/
├── backend/
│   ├── app.py              # FastAPI/Flask app
│   ├── handler.py           # Lambda entry point (Mangum wrapper)
│   ├── requirements.txt
│   └── Dockerfile.lambda
├── frontend/                # (optional — deploy via Pattern A)
│   ├── src/
│   └── package.json
└── docker-compose.yml       # local dev only
```

### B2. Create the Lambda handler

**FastAPI + Mangum** (`handler.py`):
```python
from mangum import Mangum
from app import app

handler = Mangum(app, lifespan="off")
```

**Flask** (`handler.py`):
```python
import serverless_wsgi
from app import app

def handler(event, context):
    return serverless_wsgi.handle_request(app, event, context)
```

### B3. Write the Dockerfile

```dockerfile
FROM public.ecr.aws/lambda/python:3.12

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . ${LAMBDA_TASK_ROOT}/

CMD ["handler.handler"]
```

### B4. Create ECR repository, build, and push

**CRITICAL: Build for arm64 (Graviton2) — 20% cheaper than x86.**

```bash
SUBDOMAIN="myapp"
ECR_REPO="${SUBDOMAIN}-backend"
ACCOUNT_ID="920653892745"
REGION="us-east-1"
ECR_URI="${ACCOUNT_ID}.dkr.ecr.${REGION}.amazonaws.com/${ECR_REPO}"

# Create ECR repo
aws ecr create-repository --repository-name "${ECR_REPO}" --region ${REGION}

# Login to ECR
aws ecr get-login-password --region ${REGION} | \
  docker login --username AWS --password-stdin ${ECR_URI}

# Build for arm64 — MUST use these exact flags
docker build --no-cache \
  --platform linux/arm64 \
  --provenance=false \
  --sbom=false \
  -t ${ECR_URI}:latest \
  -f Dockerfile.lambda .

# Push
docker push ${ECR_URI}:latest
```

### B5. Create Lambda function

```bash
# Create IAM role first
aws iam create-role \
  --role-name "${SUBDOMAIN}-lambda-role" \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": { "Service": "lambda.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name "${SUBDOMAIN}-lambda-role" \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Wait for role propagation
sleep 10

# Create Lambda (arm64 for cost savings)
aws lambda create-function \
  --function-name "${SUBDOMAIN}-backend" \
  --package-type Image \
  --code ImageUri=${ECR_URI}:latest \
  --role "arn:aws:iam::${ACCOUNT_ID}:role/${SUBDOMAIN}-lambda-role" \
  --architectures arm64 \
  --memory-size 512 \
  --timeout 30 \
  --environment "Variables={DATABASE_URL=<your-neon-connection-string>}"
```

### B6. Create API Gateway (HTTP API — cheaper than REST API)

```bash
API_ID=$(aws apigatewayv2 create-api \
  --name "${SUBDOMAIN}-api" \
  --protocol-type HTTP \
  --target "arn:aws:lambda:${REGION}:${ACCOUNT_ID}:function:${SUBDOMAIN}-backend" \
  --query 'ApiId' --output text)

# Add Lambda permission
aws lambda add-permission \
  --function-name "${SUBDOMAIN}-backend" \
  --statement-id "apigateway-invoke" \
  --action "lambda:InvokeFunction" \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:${REGION}:${ACCOUNT_ID}:${API_ID}/*"

echo "API Endpoint: https://${API_ID}.execute-api.${REGION}.amazonaws.com"
```

### B7. Create custom domain for API

```bash
CERT_ARN=$(aws acm list-certificates --query \
  "CertificateSummaryList[?DomainName=='genbricks.io'].CertificateArn" --output text)

# Create custom domain on API Gateway
aws apigatewayv2 create-domain-name \
  --domain-name "${SUBDOMAIN}-api.genbricks.io" \
  --domain-name-configurations \
    CertificateArn=${CERT_ARN},EndpointType=REGIONAL,SecurityPolicy=TLS_1_2

# Map API to domain
aws apigatewayv2 create-api-mapping \
  --domain-name "${SUBDOMAIN}-api.genbricks.io" \
  --api-id ${API_ID} \
  --stage '$default'

# Get the API GW target domain for DNS
TARGET=$(aws apigatewayv2 get-domain-name \
  --domain-name "${SUBDOMAIN}-api.genbricks.io" \
  --query 'DomainNameConfigurations[0].ApiGatewayDomainName' --output text)

# Create Route53 record
aws route53 change-resource-record-sets \
  --hosted-zone-id Z05133172TAH6LQX4R71F \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "'"${SUBDOMAIN}"'-api.genbricks.io",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "'"$(aws apigatewayv2 get-domain-name \
            --domain-name "${SUBDOMAIN}-api.genbricks.io" \
            --query 'DomainNameConfigurations[0].HostedZoneId' --output text)"'",
          "DNSName": "'"${TARGET}"'",
          "EvaluateTargetHealth": false
        }
      }
    }]
  }'
```

### B8. Redeploy (updates only)

```bash
# Rebuild and push new image
docker build --no-cache --platform linux/arm64 --provenance=false --sbom=false \
  -t ${ECR_URI}:latest -f Dockerfile.lambda .
docker push ${ECR_URI}:latest

# Update Lambda to use new image
aws lambda update-function-code \
  --function-name "${SUBDOMAIN}-backend" \
  --image-uri ${ECR_URI}:latest

aws lambda wait function-updated --function-name "${SUBDOMAIN}-backend"

# Clean up old ECR images (save storage costs)
UNTAGGED=$(aws ecr list-images --repository-name ${ECR_REPO} \
  --filter tagStatus=UNTAGGED --query 'imageIds[*]' --output json)
if [ "$UNTAGGED" != "[]" ]; then
  aws ecr batch-delete-image --repository-name ${ECR_REPO} --image-ids "$UNTAGGED"
fi
```

---

## Pattern C — Node.js Backend (Lambda Zip)

**Use for:** Express.js, Hono, or lightweight Node.js APIs.

**Cost: $0/month** (same Lambda free tier).

### C1. Project structure

```
myapp/
├── app.js           # Express app
├── lambda.js         # Lambda wrapper
├── package.json
└── node_modules/
```

### C2. Lambda wrapper (`lambda.js`)

```javascript
const serverlessExpress = require('@vendia/serverless-express');
const app = require('./app');

exports.handler = serverlessExpress({ app });
```

Add to `package.json` dependencies: `@vendia/serverless-express`.

### C3. Create deployment zip

```bash
SUBDOMAIN="myapp"

# Install production dependencies only
npm ci --production

# Create zip (exclude dev files)
zip -rq /tmp/${SUBDOMAIN}-lambda.zip . \
  -x "*.git*" "*.env*" "tests/*" "*.test.*" "docker*" "Docker*"
```

### C4. Create Lambda function

```bash
# Create IAM role (same as Pattern B)
aws iam create-role \
  --role-name "${SUBDOMAIN}-lambda-role" \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": { "Service": "lambda.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }]
  }'
aws iam attach-role-policy \
  --role-name "${SUBDOMAIN}-lambda-role" \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

sleep 10

aws lambda create-function \
  --function-name "${SUBDOMAIN}-backend" \
  --runtime nodejs20.x \
  --handler lambda.handler \
  --zip-file fileb:///tmp/${SUBDOMAIN}-lambda.zip \
  --role "arn:aws:iam::920653892745:role/${SUBDOMAIN}-lambda-role" \
  --memory-size 256 \
  --timeout 30 \
  --environment "Variables={DATABASE_URL=<your-neon-connection-string>,NODE_ENV=production}"
```

Then follow **Pattern B steps B6–B7** for API Gateway + custom domain setup.

### C5. Redeploy (updates only)

```bash
npm ci --production
zip -rq /tmp/${SUBDOMAIN}-lambda.zip . \
  -x "*.git*" "*.env*" "tests/*" "*.test.*" "docker*" "Docker*"

aws lambda update-function-code \
  --function-name "${SUBDOMAIN}-backend" \
  --zip-file fileb:///tmp/${SUBDOMAIN}-lambda.zip

aws lambda wait function-updated --function-name "${SUBDOMAIN}-backend"
```

---

## Pattern D — Full-Stack (SPA + API)

**Use for:** React/Vue frontend + Python/Node.js API backend.

This combines Pattern A (frontend) + Pattern B or C (backend). You get two subdomains:
- `myapp.genbricks.io` — frontend (S3 + CloudFront)
- `myapp-api.genbricks.io` — backend (Lambda + API Gateway)

### D1. Deploy frontend

Follow **Pattern A** with `SUBDOMAIN="myapp"`.

### D2. Deploy backend

Follow **Pattern B** (Python) or **Pattern C** (Node.js) with `SUBDOMAIN="myapp"`.
The API Gateway custom domain will be `myapp-api.genbricks.io`.

### D3. Configure CORS on the backend

The frontend at `myapp.genbricks.io` calls the API at `myapp-api.genbricks.io`, so CORS is required.

**FastAPI:**
```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://myapp.genbricks.io"],
    allow_methods=["*"],
    allow_headers=["*"],
    allow_credentials=True,
)
```

**Express:**
```javascript
const cors = require('cors');
app.use(cors({
  origin: 'https://myapp.genbricks.io',
  credentials: true
}));
```

### D4. Frontend API base URL

Set the production API URL in the frontend build:

```bash
# .env.production
VITE_API_URL=https://myapp-api.genbricks.io
```

---

## Pattern E — Next.js SSR via Vercel

**Use for:** Next.js apps that need server-side rendering, ISR, or API routes.

**Cost: $0/month** (Vercel hobby plan).

### E1. Deploy to Vercel

```bash
npx vercel --prod
```

### E2. Add custom subdomain

1. In Vercel dashboard: Settings → Domains → add `myapp.genbricks.io`
2. Create CNAME in Route53:

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z05133172TAH6LQX4R71F \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "myapp.genbricks.io",
        "Type": "CNAME",
        "TTL": 300,
        "ResourceRecords": [{ "Value": "cname.vercel-dns.com" }]
      }
    }]
  }'
```

---

## Database — Neon PostgreSQL (Free Tier)

All GenBricks projects use **Neon** serverless Postgres, not AWS RDS.

**Why:** RDS db.t3.micro = $15/month. Neon free tier = $0/month (0.5 GB storage, autoscaling compute, connection pooling built-in). Neon provides branching (instant database copies for dev/staging), autosuspend (compute stops when idle), and a built-in PgBouncer connection pooler — all free.

### Existing Neon Projects

| Project | Neon Endpoint | Region | Database | Pooler? |
|---|---|---|---|---|
| VoyceSoles | `ep-lingering-cell-afwpzm3e` | us-west-2 | `voycesoles` | No (direct) |
| Voyce | `ep-lucky-wave-aff6ajql` | us-west-2 | `neondb` | No (direct) |
| ClearPoint + Demos Auth | `ep-jolly-wildflower-an6vurri` | us-east-1 | `neondb` | Yes (`-pooler`) |
| VidyaBricks | `ep-falling-waterfall-am850k7s` | us-east-1 | `neondb` | Yes (`-pooler`) |

### N1. Create a new Neon project

1. Go to https://neon.tech → Create project
2. **Region:** Pick `us-east-1` (same as Lambda) to minimize latency. Use `us-west-2` only if the app serves West Coast users primarily
3. **Database name:** Use the app name (e.g., `myapp`) instead of default `neondb`
4. Copy the connection string — you get two variants:

```
# Direct connection (for migrations, admin scripts, long-running queries)
postgresql://neondb_owner:<pass>@ep-xxx.us-east-1.aws.neon.tech/myapp?sslmode=require

# Pooled connection (for Lambda/serverless — add -pooler to hostname)
postgresql://neondb_owner:<pass>@ep-xxx-pooler.us-east-1.aws.neon.tech/myapp?sslmode=require
```

5. Set as `DATABASE_URL` in Lambda environment variables

### N2. Connection strategy — pick the right driver

The connection approach depends on your runtime. Lambda functions are short-lived, so persistent TCP connection pools are wasteful. Use the right driver for your stack:

**Node.js on Lambda (RECOMMENDED: Neon serverless driver)**
```javascript
import { neon } from '@neondatabase/serverless';

const sql = neon(process.env.DATABASE_URL);

// Each call is a stateless HTTP request — no TCP pool, no connection leak
const users = await sql`SELECT * FROM users WHERE email = ${email}`;
```

Install: `npm install @neondatabase/serverless`

This is the most cost-efficient approach for Lambda. The `@neondatabase/serverless` driver uses HTTP/WebSocket instead of TCP, so there's zero connection overhead per Lambda invocation. No pool configuration needed. Use the **pooler endpoint** (`-pooler` in hostname) for best performance.

**Node.js on Lambda (alternative: pg with single Client)**
```javascript
import pg from 'pg';
const { Client } = pg;

export async function handler(event) {
  const client = new Client({
    connectionString: process.env.DATABASE_URL,
    ssl: { rejectUnauthorized: false }
  });
  await client.connect();
  try {
    const result = await client.query('SELECT * FROM users WHERE email = $1', [email]);
    return result.rows;
  } finally {
    await client.end();
  }
}
```

Use this when you can't use the Neon serverless driver (e.g., existing Express apps). Connect and disconnect per invocation. Do NOT use `Pool` on Lambda — pools leak across frozen containers.

**Python on Lambda (SQLAlchemy + psycopg2)**
```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker
from sqlalchemy.pool import NullPool

# Use NullPool on Lambda — no persistent connections
engine = create_engine(
    os.environ["DATABASE_URL"],
    pool_pre_ping=True,
    poolclass=NullPool  # critical for Lambda
)
SessionLocal = sessionmaker(bind=engine)
```

For FastAPI dependency injection:
```python
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

Use the **pooler endpoint** (`-pooler`) so Neon's built-in PgBouncer handles connection reuse across Lambda invocations.

**Python on Lambda (async — SQLAlchemy + asyncpg)**
```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

# Swap postgresql:// for postgresql+asyncpg://
DATABASE_URL = os.environ["DATABASE_URL"].replace("postgresql://", "postgresql+asyncpg://")

engine = create_async_engine(
    DATABASE_URL,
    pool_pre_ping=True,
    pool_size=1,
    max_overflow=2,
    pool_recycle=300,
    connect_args={"ssl": "require"} if "neon.tech" in DATABASE_URL else {}
)
AsyncSessionLocal = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)
```

### N3. Auth schema — JWT with bcrypt (standard pattern)

All new GenBricks projects should use this auth pattern. It's proven across 4 production apps.

**Users table (SQL):**
```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;

CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) NOT NULL UNIQUE,
  name VARCHAR(255) NOT NULL,
  password_hash VARCHAR(255),  -- nullable for future OAuth users
  role VARCHAR(50) NOT NULL DEFAULT 'user',
  auth_provider VARCHAR(50) NOT NULL DEFAULT 'local',  -- local, google, github
  is_active BOOLEAN NOT NULL DEFAULT true,
  email_verified BOOLEAN NOT NULL DEFAULT false,
  last_login_at TIMESTAMPTZ,
  metadata JSONB,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_role ON users (role);
```

**Why `password_hash` is nullable:** OAuth users (Google, GitHub) authenticate via external provider and never set a local password. The `auth_provider` column tracks which flow created the account.

### N4. Auth implementation — Node.js (Express/Lambda)

```javascript
import bcrypt from 'bcryptjs';
import jwt from 'jsonwebtoken';

const JWT_SECRET = process.env.JWT_SECRET;
const TOKEN_EXPIRY = '24h';

// Register
async function register(email, password, name) {
  const hash = await bcrypt.hash(password, 12);
  const result = await sql`
    INSERT INTO users (email, password_hash, name)
    VALUES (${email}, ${hash}, ${name})
    RETURNING id, email, name, role
  `;
  return createToken(result[0]);
}

// Login
async function login(email, password) {
  const [user] = await sql`SELECT * FROM users WHERE email = ${email} AND is_active = true`;
  if (!user || !await bcrypt.compare(password, user.password_hash)) {
    throw new Error('Invalid credentials');
  }
  await sql`UPDATE users SET last_login_at = now() WHERE id = ${user.id}`;
  return createToken(user);
}

// Create JWT
function createToken(user) {
  const token = jwt.sign(
    { id: user.id, email: user.email, role: user.role, name: user.name },
    JWT_SECRET,
    { expiresIn: TOKEN_EXPIRY }
  );
  return { access_token: token, user: { id: user.id, email: user.email, name: user.name, role: user.role } };
}

// Auth middleware
function requireAuth(req, res, next) {
  // Check cookie first (browser), then Authorization header (API clients)
  const token = req.cookies?.auth_token ||
    req.headers.authorization?.replace('Bearer ', '');
  if (!token) return res.status(401).json({ error: 'Not authenticated' });
  try {
    req.user = jwt.verify(token, JWT_SECRET);
    next();
  } catch {
    return res.status(401).json({ error: 'Token expired or invalid' });
  }
}

// Role check
function requireRole(...roles) {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) return res.status(403).json({ error: 'Forbidden' });
    next();
  };
}
```

**Token delivery — use HttpOnly cookies for browser apps:**
```javascript
res.cookie('auth_token', token, {
  httpOnly: true,
  secure: true,
  sameSite: 'None',     // required for cross-subdomain (myapp.genbricks.io → myapp-api.genbricks.io)
  domain: '.genbricks.io', // share cookie across all subdomains
  maxAge: 24 * 60 * 60 * 1000  // 24 hours
});
```

Setting `domain: '.genbricks.io'` lets the auth cookie work across subdomains — the frontend at `myapp.genbricks.io` sends it automatically to `myapp-api.genbricks.io`. This is the pattern used by ClearPoint, VidyaBricks, and Demos Auth.

### N5. Auth implementation — Python (FastAPI/Lambda)

```python
from passlib.context import CryptContext
from jose import jwt, JWTError
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from datetime import datetime, timedelta, timezone

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
security = HTTPBearer()

JWT_SECRET = os.environ["JWT_SECRET"]
JWT_ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE = timedelta(hours=24)

def hash_password(password: str) -> str:
    return pwd_context.hash(password)

def verify_password(plain: str, hashed: str) -> bool:
    return pwd_context.verify(plain, hashed)

def create_access_token(user_id: str, email: str, role: str) -> str:
    expire = datetime.now(timezone.utc) + ACCESS_TOKEN_EXPIRE
    return jwt.encode(
        {"sub": user_id, "email": email, "role": role, "exp": expire},
        JWT_SECRET, algorithm=JWT_ALGORITHM
    )

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: Session = Depends(get_db)
):
    try:
        payload = jwt.decode(credentials.credentials, JWT_SECRET, algorithms=[JWT_ALGORITHM])
        user = db.query(User).filter(User.id == payload["sub"]).first()
        if not user or not user.is_active:
            raise HTTPException(status_code=401, detail="User not found or inactive")
        return user
    except JWTError:
        raise HTTPException(status_code=401, detail="Invalid token")

# Routes
@router.post("/auth/register")
async def register(body: RegisterRequest, db: Session = Depends(get_db)):
    user = User(email=body.email, name=body.name, password_hash=hash_password(body.password))
    db.add(user)
    db.commit()
    token = create_access_token(str(user.id), user.email, user.role)
    return {"access_token": token}

@router.post("/auth/login")
async def login(body: LoginRequest, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.email == body.email).first()
    if not user or not verify_password(body.password, user.password_hash):
        raise HTTPException(status_code=401, detail="Invalid credentials")
    user.last_login_at = datetime.now(timezone.utc)
    db.commit()
    token = create_access_token(str(user.id), user.email, user.role)
    return {"access_token": token}
```

Dependencies: `pip install python-jose[cryptography] passlib[bcrypt] bcrypt`

### N6. Data API pattern — CRUD endpoints

Standard REST pattern used across all GenBricks apps:

```
GET    /api/{resource}          — list (with pagination)
GET    /api/{resource}/{id}     — get by ID
POST   /api/{resource}          — create
PUT    /api/{resource}/{id}     — update
DELETE /api/{resource}/{id}     — delete
```

**FastAPI example with pagination:**
```python
@router.get("/api/items")
async def list_items(
    page: int = 1,
    limit: int = 20,
    db: Session = Depends(get_db),
    user = Depends(get_current_user)
):
    offset = (page - 1) * limit
    items = db.query(Item).filter(Item.owner_id == user.id) \
        .order_by(Item.created_at.desc()) \
        .offset(offset).limit(limit).all()
    total = db.query(Item).filter(Item.owner_id == user.id).count()
    return {"items": items, "total": total, "page": page, "pages": (total + limit - 1) // limit}
```

**Express example with Neon serverless driver:**
```javascript
app.get('/api/items', requireAuth, async (req, res) => {
  const page = parseInt(req.query.page) || 1;
  const limit = parseInt(req.query.limit) || 20;
  const offset = (page - 1) * limit;

  const items = await sql`
    SELECT * FROM items WHERE owner_id = ${req.user.id}
    ORDER BY created_at DESC LIMIT ${limit} OFFSET ${offset}
  `;
  const [{ total }] = await sql`
    SELECT count(*)::int as total FROM items WHERE owner_id = ${req.user.id}
  `;
  res.json({ items, total, page, pages: Math.ceil(total / limit) });
});
```

### N7. Database migrations

**Python projects — use Alembic:**
```bash
pip install alembic
alembic init alembic

# Edit alembic.ini: sqlalchemy.url = <your-DIRECT-neon-url>  (NOT pooler)
# Edit alembic/env.py: import your models' Base

alembic revision --autogenerate -m "initial schema"
alembic upgrade head
```

Run migrations against the **direct endpoint** (not `-pooler`), since DDL statements don't work well through PgBouncer in transaction mode.

**Node.js projects — use raw SQL scripts:**
```javascript
// scripts/setup-db.js
import { neon } from '@neondatabase/serverless';

const sql = neon(process.env.DATABASE_URL);

await sql`CREATE TABLE IF NOT EXISTS users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email VARCHAR(255) NOT NULL UNIQUE,
  ...
)`;

console.log('Schema created');
```

Run: `node scripts/setup-db.js`

### N8. Neon branching (dev/staging/production)

Neon branches are instant, zero-cost copies of your database. Use them for safe development.

```
main branch          → development (seed data, experiments)
  └── production     → live data (Lambda connects here)
```

**Create a production branch:**
Via Neon dashboard: Project → Branches → Create Branch from `main` → name it `production`.

Each branch gets its own connection endpoint. Set the `production` branch endpoint in Lambda's `DATABASE_URL`.

**Why branch:** You can test migrations on `main` without risking production data. If a migration works on `main`, apply it to `production`. If it breaks, the `production` branch is untouched.

### N9. Cost comparison — Neon vs alternatives

| Service | Free tier | Cost at GenBricks scale | Notes |
|---|---|---|---|
| **Neon** (what we use) | 0.5 GB storage, 190h compute/mo | **$0/month** | Autosuspend after 5 min idle |
| AWS RDS db.t3.micro | None | $15/month | Always running, even idle |
| AWS Aurora Serverless v2 | None | ~$10/month minimum | 0.5 ACU minimum |
| PlanetScale | 1 GB, 1B reads/mo | $0/month | MySQL only (no Postgres) |
| Supabase | 500 MB, 2 projects | $0/month | Good alternative, more opinionated |

Neon wins because: (1) it's Postgres (not MySQL), (2) autosuspend means zero compute cost when idle, (3) built-in connection pooler (PgBouncer) is included free, (4) branching is unique and free.

### N10. Neon gotchas

1. **Use the pooler endpoint for Lambda.** Direct connections from Lambda can exhaust Neon's connection limit (25 on free tier). The `-pooler` endpoint runs PgBouncer and handles connection reuse. Format: `ep-xxx-pooler.us-east-1.aws.neon.tech`

2. **DDL via direct endpoint only.** `CREATE TABLE`, `ALTER TABLE`, and Alembic migrations must use the direct (non-pooler) endpoint. PgBouncer in transaction mode doesn't support prepared statements or DDL reliably.

3. **Autosuspend cold start.** Neon suspends compute after 5 minutes of inactivity. First query after suspension takes ~500ms-2s to wake the database. This adds to Lambda cold start. For apps where this matters, Neon paid plans offer configurable autosuspend (or always-on).

4. **`sslmode=require` is mandatory.** All Neon connections require SSL. Add `?sslmode=require` to the connection string. In Node.js, set `ssl: { rejectUnauthorized: false }`.

5. **SQLAlchemy driver prefix matters.**
   - Sync: `postgresql+psycopg2://...` (requires `psycopg2-binary`)
   - Async: `postgresql+asyncpg://...` (requires `asyncpg`)
   - Raw psycopg2/pg: `postgresql://...` (no prefix)

6. **Free tier limits.** 0.5 GB storage, 190 compute-hours/month, 1 project (with 10 branches). If you need more projects, create a second Neon account or upgrade ($19/mo for 10 GB + unlimited compute).

---

## Teardown / Decommission Checklist

When removing a subdomain:

```bash
SUBDOMAIN="myapp"
CF_DIST_ID="EXXXXXXXXX"
BUCKET="${SUBDOMAIN}-frontend"

# 1. Disable CloudFront (must disable before delete)
aws cloudfront get-distribution-config --id ${CF_DIST_ID} > /tmp/cf-config.json
# Edit: set Enabled=false, extract ETag
aws cloudfront update-distribution --id ${CF_DIST_ID} \
  --if-match <ETAG> --distribution-config file:///tmp/cf-disabled.json

# 2. Wait for distribution to deploy (can take 10-15 min)
aws cloudfront wait distribution-deployed --id ${CF_DIST_ID}

# 3. Delete CloudFront distribution
aws cloudfront delete-distribution --id ${CF_DIST_ID} --if-match <NEW_ETAG>

# 4. Delete Route53 records
aws route53 change-resource-record-sets \
  --hosted-zone-id Z05133172TAH6LQX4R71F \
  --change-batch '{
    "Changes": [{
      "Action": "DELETE",
      "ResourceRecordSet": {
        "Name": "'"${SUBDOMAIN}"'.genbricks.io",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z2FDTNDATAQYW2",
          "DNSName": "dxxxxxxxxx.cloudfront.net",
          "EvaluateTargetHealth": false
        }
      }
    }]
  }'

# 5. Empty and delete S3 bucket
aws s3 rm s3://${BUCKET} --recursive
aws s3api delete-bucket --bucket ${BUCKET}

# 6. (If Lambda) Delete function, API Gateway, ECR repo, IAM role
aws lambda delete-function --function-name "${SUBDOMAIN}-backend"
aws apigatewayv2 delete-api --api-id <API_ID>
aws ecr delete-repository --repository-name "${SUBDOMAIN}-backend" --force
aws iam detach-role-policy --role-name "${SUBDOMAIN}-lambda-role" \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name "${SUBDOMAIN}-lambda-role"
```

---

## HARD-WON GOTCHAS

These cost hours to debug. Read before deploying.

### 1. Lambda arm64 — ALWAYS build for Graviton

Building amd64 images for an arm64 Lambda causes `Runtime.InvalidEntrypoint` / `ProcessSpawnFailed` with ~5ms init. The process literally cannot spawn.

```bash
# CORRECT — always use these three flags together
docker build --platform linux/arm64 --provenance=false --sbom=false ...
```

`--provenance=false` is also required: Docker Desktop adds OCI attestation manifests that Lambda rejects with "image manifest media type not supported".

### 2. S3 bucket policy — distribution ID must match exactly

The bucket policy's `AWS:SourceArn` must reference the exact CloudFront distribution ID. If you recreate a distribution (new ID), you MUST update the bucket policy. Otherwise: 403 Access Denied.

### 3. CloudFront invalidation after every deploy

CloudFront caches aggressively. After `s3 sync`, ALWAYS run:
```bash
aws cloudfront create-invalidation --distribution-id ${CF_DIST_ID} --paths "/*"
```

### 4. SPA routing — custom error responses required

Single-page apps need 403 and 404 errors redirected to `/index.html` with HTTP 200. Without this, direct URL access (e.g., `myapp.genbricks.io/dashboard`) returns 403.

### 5. PriceClass_100 — use it, save money

`PriceClass_100` = North America + Europe edge locations only. `PriceClass_All` costs more and adds no value for US-focused apps. All GenBricks sites use `PriceClass_100`.

### 6. ECR image cleanup

Old untagged images in ECR accumulate storage costs. Clean up after every deploy:
```bash
aws ecr batch-delete-image --repository-name ${ECR_REPO} \
  --image-ids "$(aws ecr list-images --repository-name ${ECR_REPO} \
    --filter tagStatus=UNTAGGED --query 'imageIds[*]' --output json)"
```

### 7. API Gateway — use HTTP API, not REST API

HTTP API is up to 71% cheaper than REST API and supports Lambda proxy integration natively. Only use REST API if you need API keys, usage plans, or request validation.

### 8. Lambda cold starts

- Python (FastAPI via Mangum): ~1-3s cold start. Use `memory-size 512` minimum.
- Node.js (Express via serverless-express): ~500ms cold start. 256MB is fine.
- To reduce: enable Provisioned Concurrency ($$$) — NOT recommended for GenBricks scale.

### 9. Git push to genbricks org repos

The default SSH key (`~/.ssh/id_rsa`) authenticates as `suryasai87`, not `genbricks-ops`. For pushing to genbricks org repos, use HTTPS via `gh` CLI:
```bash
git remote set-url origin https://github.com/genbricks/<repo>.git
git push
git remote set-url origin git@github.com:genbricks/<repo>.git
```

### 10. SMTP — use Zoho app-specific password

Regular Zoho login password fails with `SMTPAuthenticationError 535`. Generate an app-specific password: Zoho Accounts → Security → Application-Specific Passwords. SMTP server: `smtppro.zoho.com:587`, sender: `genbricks@genbricks.io`.

---

## Quick Reference — Existing Deployments

| Subdomain | Pattern | CloudFront ID | S3 / Lambda |
|---|---|---|---|
| `genbricks.io` | A (static) | `E18X8NUBNXCVGJ` | `genbricks-ai-website` |
| `serenade.genbricks.io` | A (static) | `E3KQ0L17LCWCHM` | `serenade-genbricks` |
| `drlalita.genbricks.io` | A (static) | — | `drlalita-website` |
| `demos.genbricks.io` | A (static) | `E2I9UCRD38TCGQ` | `demos-genbricks-portal` |
| `voyce.genbricks.io` | D (full-stack) | `E3G29HR8UR7Q9Q` | `voycesoles-frontend` + Lambda |
| `assessment.genbricks.io` | C (Node+Lambda) | `E9WYFOFNB89TP` | `clearpoint-insights` Lambda |
| `arogya.genbricks.io` | A (auth-gated) | `E3KVL5EAC3CHUQ` | `arogya-frontend-920653892745` |
| `lims.genbricks.io` | A (auth-gated) | `E1U1974MDGSX6Q` | `genlims-frontend-1766516950` |
| `inventory.genbricks.io` | A (auth-gated) | `E1E7WK61YNXTE6` | `inventory-frontend-920653892745` |
| `fieldarchitects.genbricks.io` | A (auth-gated) | `EXDCBIEZ5UUFH` | `gen-fieldarchitects-app` |

---

## Cost Optimization Summary

| Resource | Our approach | Alternative we avoid | Monthly savings |
|---|---|---|---|
| Frontend hosting | S3 + CloudFront | EC2 + nginx | $8.50 |
| Backend compute | Lambda (arm64) | EC2 t3.micro | $8.50 |
| API Gateway | HTTP API | REST API | ~70% cheaper |
| Database | Neon free tier | RDS db.t3.micro | $15 |
| CDN edge locations | PriceClass_100 | PriceClass_All | ~30% cheaper |
| Docker images | arm64 Graviton | x86_64 | 20% cheaper |
| **Total** | **~$0.50/month** | **~$32.50/month** | **$32/month** |

The $0.50 is Route53 hosted zone fee. Everything else runs within free tier at GenBricks traffic levels.
