# AGS One

Internal employee engagement and rewards platform for **Alliance Global Solutions (AGS)**. Employees earn points (attendance, tasks, KPIs, peer recognition, milestones) and spend them in a rewards marketplace. The same app hosts a social feed, multiplayer and solo minigames, a food-ordering board, a medicine request board, an HR policy assistant ("Ally"), an org chart, an Email HR request builder, and HR admin tooling (employees, rewards, redemptions, feedback, audit logs, documents).

- **Production:** https://agsone.space (also `www.agsone.space`)
- **Users:** AGS employees, managers, and HR admins. Sign-in is Google-only and restricted to `@allianceglobalsolutions.com` accounts.

This README has two parts:

1. **[Ownership handover](#1-ownership-handover)**: every account, credential, domain, and hardcoded value a new owner needs to take over.
2. **[Developer guide](#2-developer-guide)**: how to run, build, migrate, and deploy the app.

---

## 1. Ownership handover

### 1.1 Asset inventory: everything that must be transferred

The app depends on the services below. Each one is a separate account that needs a new owner (or a new admin added and the old one removed). Two services are not configured in production yet; you can skip them unless you want to enable them.

| # | Service | What it's used for | Current location / identifier | Required? | How to transfer |
|---|---------|--------------------|-------------------------------|-----------|-----------------|
| 1 | **GitHub repository** | Source code, PRs | `github.com/dreyes-netizen/agsone` (personal account, **public** repo) | Yes | Settings → Danger Zone → *Transfer ownership* to the company org or new owner. Consider making it **private** (it's an internal HR tool). |
| 2 | **Vercel project** | Hosting, builds, env vars, preview deployments | Team **AGS** (`agsone`) → project **`employegames`**, region `syd1` | Yes | Invite the new owner to the *AGS* team as **Owner**, then remove the old member. After the GitHub transfer, reconnect Git (Project → Settings → Git) so the Vercel GitHub app can see the repo under its new owner. |
| 3 | **Domain `agsone.space`** | Production URL | Registered **through Vercel** inside the AGS team; expires **22 Aug 2027** | Yes | Transfers with the Vercel team. Make sure auto-renew and the team's billing card belong to the new owner. |
| 4 | Other domains in the AGS Vercel team | Not attached to this project | `agsone.com`, `agsone.com.ph`, `agsattendancehub.ph`, all at **third-party registrars** | Check | Find out which registrar account holds each and transfer those registrar logins too. |
| 5 | **Supabase project** | Postgres database (all app data), file storage bucket `policy-docs` (HR policy PDFs), Realtime broadcast | Region `ap-southeast-2` (Sydney), pooled connection on port 6543 / direct on 5432 | Yes | Supabase org → *Members* → invite the new owner as **Owner**, or *Transfer project* to an org they own. Note: this is the **only** database; dev and production share it. |
| 6 | **Firebase project** (Google Cloud) | Authentication (Google sign-in), Admin SDK service account for verifying ID tokens | Project ID **`ags-one-82a7b`** | Yes | Firebase console → Project settings → *Users and permissions* → add the new owner as **Owner** (this is a Google Cloud IAM role). Also transfer the GCP billing account if one is linked. |
| 7 | **Cloudinary** | Image/video uploads (avatars, feed media, reward art, food photos, org-chart photos, document previews) | Cloud name **`dfrmh2iee`**, folder `ags-one/` | Yes | Settings → Users → invite the new owner as *Master admin*, or change the account's login email. |
| 8 | **Groq** | LLM for the "Ally" HR policy assistant (`app/api/assistant/chat`) | API key in `GROQ_API_KEY` | Yes (assistant breaks without it) | Create a key under the new owner's Groq account and swap the env var. |
| 9 | **GIPHY** | GIF search in feed comments (browser-side calls) | Public key in `NEXT_PUBLIC_GIPHY_API_KEY` | Yes (GIF picker breaks without it) | Create an app under the new owner's GIPHY developer account and swap the key. |
| 10 | **Gmail SMTP mailbox** | Outbound notification emails (`lib/email/mailer.ts`) | Sends via `smtp.gmail.com:587` as a **personal** `@allianceglobalsolutions.com` mailbox using an App Password | Yes (emails silently stop without it) | Move to a shared/service mailbox (e.g. `noreply@` or `agsone@`). Generate an App Password on it, then update `EMAIL_USER`, `EMAIL_PASS`, `EMAIL_FROM`. |
| 11 | **Web Push VAPID keypair** | Browser push notifications (`lib/push/send.ts`) | `VAPID_PUBLIC_KEY` / `VAPID_PRIVATE_KEY` / `VAPID_SUBJECT` | Yes | Hand over the existing keypair as-is. **Do not regenerate** it, because new keys invalidate every existing push subscription. Update `VAPID_SUBJECT` to the new owner's contact email. |
| 12 | **Google Workspace** (`allianceglobalsolutions.com`) | Sign-in is restricted to this domain (`hd` param in `lib/firebase/client.ts`); Email HR opens Gmail compose | Company IT | Yes | No transfer needed, but company IT should know the app depends on it. |
| 13 | OpenAI | Feed content moderation ("Alexa", `lib/guardrails/moderation.ts`) | `OPENAI_API_KEY`, **not set in production** | Optional | Fails open: without a key, posts are published unmoderated. Add a key from the new owner's account to enable moderation. |
| 14 | Upstash Redis | Distributed rate limiting (`lib/guardrails/rateLimiter.ts`) | `UPSTASH_REDIS_REST_URL/TOKEN`, **not set in production** | Optional | Falls back to per-instance in-memory limits. Create a database under the new owner's account to enable it. |
| 15 | Sentry | Error and trace reporting (`instrumentation*.ts`, `next.config.ts`) | `SENTRY_DSN` / `NEXT_PUBLIC_SENTRY_DSN`, **not set** | Optional | Inert until a DSN is set. Create a project under the new owner's account if wanted. |

> **Unrelated projects in the same Vercel team:** `agstardinesstracker` (agsattendancehub.vercel.app), `ags-performance-hub`, and `legacy-agent-activity` are separate apps that live in the same *AGS* Vercel team. They are **not** part of this repo, but transferring the team transfers them too.

### 1.2 Recommended transfer order (zero downtime)

1. **Add before you remove.** Add the new owner as an admin/owner on every service in 1.1 while the current owner still has access.
2. **Transfer the GitHub repo**, then reconnect Git in Vercel (Project `employegames` → Settings → Git). Push a trivial PR to confirm a preview deployment builds.
3. **Move the email sender** to a shared mailbox and update the `EMAIL_*` variables in Vercel.
4. **Rotate secrets** (see 1.3) and update them in Vercel for **both Production and Preview**, then redeploy.
5. **Update hardcoded personal values** (see 1.5) in a PR.
6. Verify production: sign in, award points, upload an image, post with a GIF, ask Ally a question, check that an email arrives.
7. **Remove the previous owner** from Vercel, Supabase, Firebase/GCP, Cloudinary, Groq, GIPHY, and GitHub.

### 1.3 Secrets to rotate after handover

The previous owner has had these values locally (in `.env.local`, and in local tooling config like `.mcp.json`), so rotate them once the new owner is in control:

| Secret | Where to rotate | Then update |
|--------|-----------------|-------------|
| Supabase **database password** | Supabase → Project Settings → Database → *Reset database password* | `DATABASE_URL`, `DIRECT_URL` |
| Supabase **service role key** (and anon key) | Supabase → Project Settings → API → *JWT keys / API keys*. Rotating the JWT secret regenerates both. | `SUPABASE_SERVICE_ROLE_KEY`, `NEXT_PUBLIC_SUPABASE_ANON_KEY` |
| Firebase **Admin service account key** | GCP → IAM → Service accounts → `firebase-adminsdk-…` → Keys → add new key, delete old | `FIREBASE_CLIENT_EMAIL`, `FIREBASE_PRIVATE_KEY` |
| Cloudinary API secret | Cloudinary → Settings → API Keys → generate new, revoke old | `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET` |
| Groq API key | Groq console → API Keys | `GROQ_API_KEY` |
| GIPHY key | GIPHY developers dashboard | `NEXT_PUBLIC_GIPHY_API_KEY` |
| Gmail App Password | Google Account → Security → App passwords (on the new mailbox) | `EMAIL_PASS` |
| `CRON_SECRET`, `BOOTSTRAP_SECRET` | Any random string (`openssl rand -hex 32`) | Vercel env vars |

Do **not** rotate the VAPID keys (see 1.1 #11). The Firebase web API key (`NEXT_PUBLIC_FIREBASE_*`) is public by design and doesn't need rotating.

### 1.4 Environment variables (complete list)

Production values live in **Vercel → `employegames` → Settings → Environment Variables** (Production + Preview). Locally they go in `.env.local`, which is gitignored. Prisma CLI commands also read `.env` (it only needs `DATABASE_URL` and `DIRECT_URL`). To pull the current values: `vercel link --project employegames --scope agsone && vercel env pull .env.local`.

| Variable | Status | Used by | Notes |
|----------|--------|---------|-------|
| `DATABASE_URL` | **Required** | `lib/prisma/client.ts`, scripts | Supabase **pooled** connection (pgbouncer, port 6543) |
| `DIRECT_URL` | **Required** | `prisma.config.ts` | Supabase **direct** connection (port 5432), used for migrations |
| `NEXT_PUBLIC_SUPABASE_URL` | **Required** | Realtime, storage | `https://<project-ref>.supabase.co` |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | **Required** | Browser Realtime subscriptions | Public |
| `SUPABASE_SERVICE_ROLE_KEY` | **Required** | `lib/supabase/storageClient.ts`, `lib/realtime/broadcast.ts` | **Server-only secret** |
| `NEXT_PUBLIC_FIREBASE_API_KEY` | **Required** | `lib/firebase/client.ts` | Public |
| `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN` | **Required** | 〃 | `ags-one-82a7b.firebaseapp.com` |
| `NEXT_PUBLIC_FIREBASE_PROJECT_ID` | **Required** | 〃 | `ags-one-82a7b` |
| `NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET` | **Required** | 〃 | |
| `NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID` | **Required** | 〃 | |
| `NEXT_PUBLIC_FIREBASE_APP_ID` | **Required** | 〃 | |
| `FIREBASE_PROJECT_ID` | **Required** | `lib/firebase/admin.ts` | |
| `FIREBASE_CLIENT_EMAIL` | **Required** | 〃 | Service-account email |
| `FIREBASE_PRIVATE_KEY` | **Required** | 〃 | Service-account private key; `\n` escapes are converted at runtime |
| `NEXT_PUBLIC_CLOUDINARY_CLOUD_NAME` | **Required** | `lib/cloudinary/*`, image URLs | `dfrmh2iee` |
| `CLOUDINARY_API_KEY` | **Required** | Server-side signed uploads/deletes | |
| `CLOUDINARY_API_SECRET` | **Required** | 〃 | **Secret** |
| `GROQ_API_KEY` | **Required** | `lib/groq/client.ts` (Ally assistant) | |
| `NEXT_PUBLIC_GIPHY_API_KEY` | **Required** | `lib/giphy/client.ts` | Public, browser-side by GIPHY's terms |
| `EMAIL_HOST` | **Required** | `lib/email/mailer.ts` | `smtp.gmail.com` |
| `EMAIL_PORT` | **Required** | 〃 | `587` |
| `EMAIL_USER` | **Required** | 〃 | Mailbox login. Email is skipped if this is unset. |
| `EMAIL_PASS` | **Required** | 〃 | Gmail App Password |
| `EMAIL_FROM` | **Required** | 〃 | e.g. `"AGS One <noreply@allianceglobalsolutions.com>"` |
| `NEXT_PUBLIC_APP_URL` | **Required** | Email links/logo, notification links | `https://agsone.space` in production |
| `VAPID_PUBLIC_KEY` | **Required** | `lib/push/send.ts`, push subscribe | Push is disabled (not broken) if missing |
| `VAPID_PRIVATE_KEY` | **Required** | 〃 | **Secret** |
| `VAPID_SUBJECT` | **Required** | 〃 | `mailto:<owner email>` |
| `BOOTSTRAP_SECRET` | Setup only | `app/api/admin/bootstrap` | Only needed to create the very first HR admin (see 2.6). Not set in production. |
| `OPENAI_API_KEY` | Optional | `lib/guardrails/moderation.ts` | Feed moderation, fails open. Not set in production. |
| `UPSTASH_REDIS_REST_URL` | Optional | `lib/guardrails/rateLimiter.ts` | Falls back to in-memory. Not set in production. |
| `UPSTASH_REDIS_REST_TOKEN` | Optional | 〃 | |
| `SENTRY_DSN` / `NEXT_PUBLIC_SENTRY_DSN` | Optional | `instrumentation.ts`, `instrumentation-client.ts` | Not set |
| `SENTRY_AUTH_TOKEN` / `SENTRY_ORG` / `SENTRY_PROJECT` | Optional | `next.config.ts` (source-map upload) | Not set |
| `CRON_SECRET` | Unused | none | Set in Vercel, but no cron routes or Vercel cron jobs exist. Safe to remove. |
| `GEMINI_API_KEY` | Unused | none | Set in Vercel, not read anywhere. Safe to remove. |
| `NEXT_PUBLIC_CLOUDINARY_UPLOAD_PRESET` | Unused | none | Set in Vercel, not read anywhere (uploads are signed server-side). |
| `NEXT_PUBLIC_FIREBASE_MEASUREMENT_ID` | Unused | none | Analytics isn't initialised |
| `VERCEL_OIDC_TOKEN` | Auto | none | Written by `vercel env pull`; ignore |

### 1.5 Hardcoded values tied to people or accounts

These are in the source code, not env vars, so changing them takes a code change (a PR):

| Value | File | Why it matters |
|-------|------|----------------|
| `HR_EMAIL = hr@allianceglobalsolutions.com` | `lib/constants/hr.ts`, `lib/constants/hrRequests.ts` | HR inbox for Email HR and the Points of Contact fallback |
| `HR_DRAFT_CC = r.febrero@allianceglobalsolutions.com` | `lib/constants/hr.ts` | A named person is CC'd on every Email HR draft. Update this if HR staff change. |
| `HR_NOTIFICATION_EMAILS` (`hr.ags@…`, `hr@…`) | `lib/constants/hr.ts` | Distribution list for HR notification emails |
| `hd: "allianceglobalsolutions.com"` | `lib/firebase/client.ts` | Restricts Google sign-in to the company domain |
| `https://ags-one-82a7b.firebaseapp.com` | `next.config.ts` (CSP `frame-src`) | Must match the Firebase auth domain if the Firebase project ever changes |
| CSP allow-list (Cloudinary, Supabase, Google, Groq, GIPHY, DiceBear) | `next.config.ts` | Any new external host must be added here |
| `allowedDevOrigins: jinx-delicious-jawline.ngrok-free.dev` | `next.config.ts` | The previous developer's personal ngrok tunnel (dev only). Safe to remove. |
| `"regions": ["syd1"]` | `vercel.json` | Functions run in Sydney, next to the Supabase DB (`ap-southeast-2`). Keep these two together. |

### 1.6 Things the new owner should know

- **One database for everything.** `npm run dev` on a laptop reads and writes **live production data**, and so do migrations and scripts. Test data created locally is immediately visible to real employees. There is no staging environment.
- **Migrations aren't run on deploy.** `npm run build` only runs `prisma generate`. Schema changes must be applied by hand (see 2.4).
- **No scheduled jobs.** No Vercel cron jobs are configured (`vercel crons ls` is empty). Some docs mention `/api/cron/*`, but those routes no longer exist.
- **Manual SQL lives outside Prisma.** The `document_chunks` table and its full-text-search function (used by Ally) come from `scripts/setup-rag.sql`, which was run once in the Supabase SQL editor. Prisma treats that table as external (`prisma.config.ts`) and must never drop it.
- **Pre-commit secret scanner.** `scripts/git-hooks/pre-commit` blocks commits that contain keys or PII spreadsheets. Each developer installs it with `npm run setup:hooks`.
- **The repo is public.** Never commit `.env*`, service-account JSON, or employee exports. `.gitignore` and the pre-commit hook guard against this, but making the repo private is the stronger fix.
- **Billing.** The Vercel team, the Vercel-registered domain, Supabase, Firebase/GCP, and Cloudinary each have their own billing. Move each one to a company card. The codebase was tuned for Vercel Hobby-tier limits; see `vercel-hobby-audit.md`.

---

## 2. Developer guide

### 2.1 Stack

| Layer | Technology |
|-------|-----------|
| Framework | **Next.js 16** (App Router), React 19, TypeScript (strict). This Next.js version has breaking changes: read `AGENTS.md`, and note that middleware lives in **`proxy.ts`**, not `middleware.ts`. |
| UI | Tailwind CSS v4, shadcn/ui (`base-nova` style), lucide icons, Recharts, React Flow (org chart) |
| Data | PostgreSQL on Supabase, **Prisma 7** with the `@prisma/adapter-pg` driver adapter. The client is generated to `lib/generated/prisma`, not `node_modules`. |
| Auth | Firebase Auth (Google sign-in) in the browser + `firebase-admin` token verification in every API route |
| Realtime | Supabase Realtime broadcast ("something changed" pings; clients re-fetch through the authed API) |
| Storage | Cloudinary (images/video), Supabase Storage bucket `policy-docs` (HR policy PDFs) |
| AI | Groq (Ally HR assistant over Postgres full-text search), OpenAI (feed moderation, optional) |
| Email / push | Nodemailer over Gmail SMTP; Web Push with VAPID (`public/sw.js`) |
| Tests | Vitest (Node environment, `**/*.test.ts`) |
| Hosting | Vercel, region `syd1` |

### 2.2 Local setup

Requirements: Node.js 24.x (matches Vercel) and npm.

```bash
git clone <repo-url> agsone && cd agsone
npm install                 # also runs `prisma generate`
# create .env.local with the variables in section 1.4 (or `vercel env pull .env.local`)
# create .env with DATABASE_URL and DIRECT_URL (read by Prisma CLI commands)
npm run setup:hooks         # install the secret-blocking pre-commit hook
npm run dev                 # http://localhost:3010
```

Port **3010** is deliberate (3000 was taken on the original dev machine). Remember that local dev uses the production database (see 1.6).

To sign in locally, `localhost` must be listed under Firebase console → Authentication → Settings → **Authorized domains** (as must `agsone.space` for production).

### 2.3 Commands

```bash
npm run dev           # dev server on :3010
npm run build         # prisma generate && next build
npm run start         # serve the production build
npm run lint          # ESLint (flat config)
npm test              # Vitest suite
npm run test:watch    # Vitest watch mode
npx tsc --noEmit      # type-check
npm run setup:hooks   # install scripts/git-hooks/pre-commit
npm run reset:data    # DANGEROUS: wipes transactional data (dry run unless --confirm)
npx tsx prisma/seed.ts                  # seed rewards/badges/etc.
npx tsx scripts/sync-arcade-badges.ts   # upsert Solo Arcade badge definitions
```

### 2.4 Database and migrations

- Schema: `prisma/schema.prisma` (37 models). Migrations: `prisma/migrations/`, re-baselined on 2026-08-19. Older history is kept in `prisma/migrations-archive-2026-08-19/` for reference only.
- **Create** a migration: `npx prisma migrate dev --name <change>` (uses `DIRECT_URL`).
- **Apply** to production: `npx prisma migrate deploy`. With one shared database, this applies it to production immediately. Run it **before** merging code that depends on the new schema.
- After any schema change, run `npx prisma generate` (`npm install` and `npm run build` do this automatically).
- One-off SQL in `scripts/*.sql` (`setup-rag.sql`, `add-indexes.sql`, …) was run manually in the Supabase SQL editor. Only re-run these when rebuilding a database from scratch.

**Rebuilding from an empty database:** `npx prisma migrate deploy` → run `scripts/setup-rag.sql` in Supabase → `npx tsx prisma/seed.ts` → `npx tsx scripts/sync-arcade-badges.ts` → create the `policy-docs` storage bucket in Supabase → bootstrap the first HR admin (2.6).

### 2.5 Deployment

- Vercel auto-deploys **`main` → Production** and every PR branch → a Preview deployment.
- **Never push directly to `main`.** Work on `feat/*` / `fix/*` / `docs/*` branches and merge via pull request.
- Preview deployments use the same production database and credentials (Preview env vars mirror Production).
- Before merging: `npm test`, `npm run lint`, `npx tsc --noEmit`, and apply any pending migration.

### 2.6 Roles and the first admin

Roles: `EMPLOYEE < MANAGER < HR_ADMIN < SUPER_ADMIN` (`lib/constants/roles.ts`). Existing HR admins promote users from **Admin → Employees**.

On a fresh database with no `HR_ADMIN` yet, set `BOOTSTRAP_SECRET`, sign in once, then call `POST /api/admin/bootstrap` with the `x-bootstrap-secret` header and your Firebase ID token as `Authorization: Bearer …`. That promotes the caller to `HR_ADMIN`. The route disables itself once any HR admin exists. Remove the secret afterwards.

### 2.7 Architecture at a glance

```
app/
  (auth)/         /login, /onboarding
  (dashboard)/    employee app: feed, marketplace, leaderboard, minigames, food, medicine,
                  documents, org-chart, contacts, email-hr, code-of-conduct, feedback, profile
  admin/          HR/manager tooling: employees, accounts, points, rewards, redemptions,
                  feedback, audit, departments, documents, hr-documents, hr-requests,
                  medicine, org-chart, points-of-contact, code-of-conduct
  api/            all backend route handlers
lib/
  auth/           verifyAuth (real auth boundary), requireRole, timingSafeCompare
  prisma/         Prisma client (generated code in lib/generated/prisma)
  minigames/      pure game logic (multiplayer + solo/)
  realtime/       Supabase broadcast + tab/visibility coordination
  guardrails/     moderation, jailbreak filter, rate limiter
  rag/            policy-document chunking + full-text search for Ally
  email/ push/ cloudinary/ supabase/ firebase/ groq/ openai/ giphy/
  constants/      domain vocabularies (statuses, categories, HR emails)
proxy.ts          page-level redirect to /login (UX only, not security)
next.config.ts    security headers + CSP, redirects, image config, Sentry wrapper
```

Every API route follows the same order: `verifyAuth` → `requireRole` → zod validation → business rules → `prisma.$transaction` → side effects run concurrently and never throw (notifications, email, push, broadcast) → audit log. Successful responses are `{ data }` and errors are `{ error }`. `app/api/points/award/route.ts` is the reference example.

### 2.8 Further documentation in this repo

| File | Contents |
|------|----------|
| `CLAUDE.md` / `AGENTS.md` | Detailed architecture notes and conventions (also used by AI coding assistants) |
| `PRODUCT.md` | Users, product purpose, brand personality |
| `DESIGN.md`, `.impeccable/design.json` | Design system (colors, type scale, spacing, radii) |
| `docs/superpowers/specs/`, `docs/superpowers/plans/` | Dated design docs and implementation plans for each major feature |
| `docs/solo-arcade-v1-engineering-notes.md` | Solo Arcade engineering notes |
| `vercel-hobby-audit.md`, `code-audit-report.md`, `qa-audit-report.md`, `audit-orchestrator-report.md` | Past cost/performance, code, and QA audits |
