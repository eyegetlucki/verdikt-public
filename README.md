# Verdikt

AI-powered ATS resume scorer. Paste your resume and a job description — get an instant match score, keyword gap analysis, ranked suggestions, and rewritten bullets powered by Claude.

**Live at [verdikt.cc](https://verdikt.cc)**

---

## What it does

- **ATS Score (0–100)** — weighted across keyword match, impact quality, relevance, and formatting
- **Keyword analysis** — matched vs. missing keywords with priority (critical / preferred)
- **Bullet rewrites** — Claude rewrites weak bullets with stronger verbs and measurable outcomes
- **Seniority analysis** — detects language mismatch between your resume and the target role (Pro)
- **Red flag detection** — employment gaps, weak verbs, missing sections, job hopping (Pro)
- **Projected score** — shows how much your score improves after applying all suggestions (Pro)
- **Generate improved resume** — apply selected changes and download a clean PDF (Pro)

---

## Tiers

| Feature | Free | Pro ($9/month) |
|---|---|---|
| ATS score | ✓ | ✓ |
| Matched / missing keywords | ✓ | ✓ |
| Sub-scores | — | ✓ |
| Seniority analysis | — | ✓ |
| Red flag breakdown | — | ✓ |
| Keyword priority + frequency | — | ✓ |
| Projected score simulation | — | ✓ |
| Ranked suggestions | — | ✓ |
| Bullet rewrites | — | ✓ |
| Generate improved resume PDF | — | ✓ |
| Scans per month | 5 | 200 |

---

## Tech stack

**Backend**
- FastAPI (Python, async)
- Claude API (`claude-sonnet-4-6`) — scoring engine + resume generation
- Supabase — Postgres DB + row-level security
- Stripe — subscription billing
- Docker → AWS ECS Fargate + ECR
- AWS ALB + ACM + Route 53

**Web**
- Next.js 15 (App Router)
- Clerk — auth (email + Google OAuth)
- Tailwind CSS

**iOS**
- Expo (React Native) + Expo Router
- Clerk auth via dev build (not Expo Go compatible)

---

## Monorepo structure

```
verdikt/
├── apps/
│   ├── web/          — Next.js 15 frontend
│   └── mobile/       — Expo iOS app
├── backend/          — FastAPI backend
│   ├── routers/      — score, history, billing, parse, resume
│   ├── services/     — claude_service, usage_service, stripe_service, supabase_client
│   └── models/       — Pydantic request/response models
└── packages/
    └── tokens/       — shared design tokens (web + iOS)
```

---

## Running locally

### Backend

```bash
cd backend
cp .env.example .env   # fill in your keys
pip install -r requirements.txt
uvicorn main:app --reload --port 8000
```

Required env vars:
```
ANTHROPIC_API_KEY=
SUPABASE_URL=
SUPABASE_SERVICE_KEY=
CLERK_JWKS_URL=https://<your-clerk-domain>/.well-known/jwks.json
CLERK_SECRET_KEY=
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
STRIPE_PRICE_ID=
ALLOWED_ORIGINS=http://localhost:3000
```

### Web

```bash
cd apps/web
cp .env.local.example .env.local   # fill in your keys
npm install
npm run dev
```

Required env vars:
```
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=
CLERK_SECRET_KEY=
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
NEXT_PUBLIC_API_URL=http://localhost:8000
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=
```

### iOS

```bash
cd apps/mobile
npm install --legacy-peer-deps
# create apps/mobile/.env with EXPO_PUBLIC_CLERK_PUBLISHABLE_KEY and EXPO_PUBLIC_API_URL
npx expo run:ios --device   # requires Mac + Xcode
```

> Expo Go is not supported — Clerk requires a dev build with native modules.

---

## Supabase schema

```sql
create table public.profiles (
  id text primary key,   -- Clerk user ID
  email text not null,
  plan text not null default 'free',
  stripe_customer_id text,
  stripe_subscription_id text,
  created_at timestamptz default now()
);

create table public.scans (
  id uuid primary key default gen_random_uuid(),
  user_id text references public.profiles(id) on delete cascade,
  score integer not null,
  verdict text,
  result_json jsonb not null,
  resume_text text,
  job_title text,
  created_at timestamptz default now()
);

create table public.usage (
  id uuid primary key default gen_random_uuid(),
  user_id text references public.profiles(id) on delete cascade,
  month text not null,
  scan_count integer default 0,
  unique(user_id, month)
);
```

RLS: users can SELECT their own profile (no UPDATE — plan changes only via service role/Stripe webhook).

---

## API endpoints

```
POST  /score              — run a scan (auth required, usage checked)
GET   /history            — paginated scan history
GET   /history/{id}       — single scan result
GET   /usage              — current month scan count + plan limit
POST  /parse-resume       — extract text from PDF or DOCX
POST  /generate-resume    — generate improved resume HTML via Claude Vision (Pro only)
POST  /billing/checkout   — create Stripe checkout session
POST  /billing/portal     — Stripe billing portal
POST  /billing/webhook    — Stripe webhook handler
GET   /billing/status     — current plan + subscription status
GET   /health             — health check
```

---

## Deployment

Backend is containerized and runs on AWS ECS Fargate behind an ALB. Frontend is deployed on Vercel.

```bash
# Build and push backend image
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account>.dkr.ecr.us-east-1.amazonaws.com
docker build -t verdikt-backend ./backend
docker tag verdikt-backend:latest <account>.dkr.ecr.us-east-1.amazonaws.com/verdikt-backend:latest
docker push <account>.dkr.ecr.us-east-1.amazonaws.com/verdikt-backend:latest

# Force ECS redeploy
aws ecs update-service --cluster verdikt --service verdikt-backend --force-new-deployment --region us-east-1
```

Frontend deploys automatically via Vercel on push to `main`. Root directory is set to `apps/web` in Vercel project settings.

---

## License

MIT
