# AGS One

Internal employee engagement and rewards platform for **Alliance Global Solutions (AGS)**. Employees earn points (attendance, tasks, KPIs, peer recognition, milestones) and spend them in a rewards marketplace. The same app hosts a social feed, multiplayer and solo minigames, a food-ordering board, a medicine request board, an HR policy assistant ("Ally"), an org chart, an Email HR request builder, and HR admin tooling (employees, rewards, redemptions, feedback, audit logs, documents).

- **Production:** https://agsone.space (also `www.agsone.space`)
- **Users:** AGS employees, managers, and HR admins. Sign-in is Google-only and restricted to `@allianceglobalsolutions.com` accounts.

This README has two parts:

1. **[Ownership handover](#1-ownership-handover)**: the owner account, every service the app depends on, credentials, and hardcoded values.
2. **[Developer guide](#2-developer-guide)**: how to run, build, migrate, and deploy the app.

---

## 1. Ownership handover

### 1.0 Owner account

All accounts are owned by the shared company account **`agsdev@allianceglobalsolutions.com`**. The original developer (Dither Reyes, `d.reyes@`) handed it over in October 2026.

To sign in to **GitHub, Vercel, Supabase, and Firebase**, use `agsdev@allianceglobalsolutions.com`:

1. Try **Continue with GitHub** first, using the GitHub account registered to `agsdev@`.
2. If that doesn't work, use **Continue with Google** with the same `agsdev@` Google Workspace account.
3. Firebase is part of Google Cloud, so it always uses **Google** sign-in with `agsdev@`.

| Service | Where | Owner |
|---------|-------|-------|
| GitHub | `github.com/agsdev-code/agsone` (moved from `github.com/dreyes-netizen/agsone`, which no longer resolves) | agsdev |
| Vercel | Team **AGS** (`agsone`) → project **`agsone`** (renamed from `employegames`) | agsdev |
| Supabase | Org **AGS Dev** → project **AGS One**, ref `qtugwwotqnznpnttzvhl`, region `ap-southeast-2` | agsdev |
| Firebase / GCP | Project **`ags-one-82a7b`** | agsdev |
| Cloudinary | Cloud **`cdnl9dwe`** (agsdev's own Cloudinary account; AGS One assets were moved in from the old cloud `dfrmh2iee`) | agsdev |

### 1.1 Handover status (2026-10-14)

**Done.** Production runs entirely on agsdev-owned accounts. It was verified after the final redeploy: sign-in, image upload, GIF search, Ally, and email all work.

- [x] GitHub repository transferred to agsdev (`agsdev-code/agsone`)
- [x] Vercel team **AGS** and project `agsone` transferred to agsdev
- [x] Firebase project `ags-one-82a7b` transferred to agsdev
- [x] Supabase **copied** into a new project owned by agsdev, `qtugwwotqnznpnttzvhl` (see 2.4, "Copying to a new Supabase project"). The copy was a fresh project rather than a transfer, so its usage and billing history start clean. All 36 data tables and 9 `policy-docs` files were copied and verified.
- [x] Cloudinary **moved** into agsdev's existing cloud `cdnl9dwe` (see 2.4, "Moving to a different Cloudinary cloud"). All 349 assets were copied byte-for-byte with the same `public_id`s, and the 218 database rows holding URLs were rewritten. PDF/ZIP delivery is enabled on the new cloud.
- [x] GIPHY key moved to agsdev's GIPHY app (API type)
- [x] Groq key moved to agsdev's Groq account. Ally switched to `openai/gpt-oss-20b` after Groq retired `llama-3.1-8b-instant` (PR #18).
- [x] Email sender moved from the previous developer's personal mailbox (`d.reyes@`) to `agsdev@` (Gmail App Password `AGS One SMTP`)
- [x] All of the above set in Vercel **Production, Preview, and Development**. The two `NEXT_PUBLIC_SUPABASE_*` values, `NEXT_PUBLIC_CLOUDINARY_CLOUD_NAME`, and `NEXT_PUBLIC_GIPHY_API_KEY` are stored as *Config*, because Vercel won't store browser-exposed values as *Secret*.
- [x] Production redeployed from `main` (`4cefeba`) and verified on `agsone.space`

**Still to do** (agsdev unless noted):

- [ ] **Make the GitHub repo private** (Settings → General → Danger Zone; only `agsdev-code` has admin)
- [ ] **Rotate the credentials shared during the handover.** The Cloudinary API key/secret, Groq key, and Gmail App Password were pasted in a chat session. Regenerate each one and set it in Vercel directly (Production, Preview, Development), then redeploy. See 1.3 for the rest.
- [ ] **Pause, then delete, the old Supabase project** `fqmrnvjjkotusiymrqum` (old org **AGS**). Until it's deleted it still holds a full copy of employee data. *(previous developer)*
- [ ] **Delete the old Cloudinary account** (cloud `dfrmh2iee`) after about a week. Every asset is already in `cdnl9dwe`. *(previous developer)*
- [ ] **Revoke the previous developer's old credentials:** the Gmail App Password on `d.reyes@`, the GIPHY key `PXLe…PbVy`, and the Groq key `gsk_Wnro…` *(previous developer)*
- [x] Previous developer removed from GitHub (`Ditherys`, `dreyes-netizen`) and the Vercel team (`reyesditherb-9328`)
- [ ] **Remove `d.reyes@` from Firebase/GCP IAM** on `ags-one-82a7b`. First confirm `agsdev@` is **Owner**.
- [ ] **Review other access:** GitHub collaborator `vtion001` (write) and Vercel viewer `patrick@allianceglobalsolutions.com`
- [ ] Apply for a GIPHY **production** key. The current key is a rate-limited beta key.
- [ ] Update the hardcoded values in 1.5 if HR contacts change
- [ ] Each developer points their clone at the new remote: `git remote set-url origin https://github.com/agsdev-code/agsone.git`

### 1.2 Services the app depends on

Each service is a separate account. Three (rows 13–15) are not configured in production yet; skip them unless you want to enable them.

| # | Service | What it's used for | Current location / identifier | Required? | Notes |
|---|---------|--------------------|-------------------------------|-----------|-------|
| 1 | **GitHub repository** | Source code, PRs | `github.com/agsdev-code/agsone`, owned by agsdev (GitHub user `agsdev-code`). Still **public**. | Yes | Make it **private** (Settings → General → Danger Zone); it's an internal HR tool. If Vercel stops deploying, reconnect Git under Vercel Project → Settings → Git. |
| 2 | **Vercel project** | Hosting, builds, env vars, preview deployments | Team **AGS** (`agsone`) → project **`agsone`**, region `syd1` | Yes | Billing for the team belongs to the agsdev account. |
| 3 | **Domain `agsone.space`** | Production URL | Registered **through Vercel** inside the AGS team; expires **22 Aug 2027** | Yes | Moves with the Vercel team. Keep auto-renew on with a company card. |
| 4 | Other domains in the AGS Vercel team | Not attached to this project | `agsone.com`, `agsone.com.ph`, `agsattendancehub.ph`, all at **third-party registrars** | Check | Find out which registrar account holds each one. |
| 5 | **Supabase project** | Postgres database (all app data), file storage bucket `policy-docs` (HR policy PDFs), Realtime broadcast | Org **AGS Dev** → **AGS One** (`qtugwwotqnznpnttzvhl`), `ap-southeast-2` (Sydney). Pooler host `aws-0-ap-southeast-2.pooler.supabase.com`: port 6543 for the app, 5432 for migrations. | Yes | This is the **only** database; dev and production share it. Old project `fqmrnvjjkotusiymrqum` (org **AGS**) is to be deleted after cutover. The same org also contains an unrelated project, *AGS Break Tracker*. |
| 6 | **Firebase project** (Google Cloud) | Authentication (Google sign-in), Admin SDK service account for verifying ID tokens | Project ID **`ags-one-82a7b`** | Yes | Owned by agsdev. Also check that the GCP billing account, if one is linked, belongs to the company. |
| 7 | **Cloudinary** | Image/video uploads (avatars, feed media, reward art, food photos, org-chart photos, document previews) | Cloud name **`cdnl9dwe`** in agsdev's account (shared with Cloudinary's sample assets) | Yes | The database stores **full delivery URLs** that include the cloud name, so changing clouds means copying assets and rewriting URLs (see 2.4). *Settings → Security → Allow delivery of PDF and ZIP files* must stay enabled. |
| 8 | **Groq** | LLM for the "Ally" HR policy assistant (`app/api/assistant/chat`) | API key in `GROQ_API_KEY`, from agsdev's Groq account (set 2026-10-14). Model: `openai/gpt-oss-20b`, hardcoded in `lib/groq/client.ts`. | Yes (assistant breaks without it) | Groq retires models periodically: `llama-3.1-8b-instant` was retired and broke Ally until PR #18. If Ally fails with `model_not_found`, pick a current **production** model from the Groq console and change the one line in `lib/groq/client.ts`. |
| 9 | **GIPHY** | GIF search in feed comments (browser-side calls) | Public key in `NEXT_PUBLIC_GIPHY_API_KEY`, from an **API**-type app (not SDK) in agsdev's GIPHY developer account (key `sJNo…Qaa7`, set 2026-10-14) | Yes (GIF picker breaks without it) | New keys start as rate-limited **beta** keys. Use *Upgrade to Production* in the GIPHY dashboard for higher limits. Comments store only `gifId`, so swapping keys needs no database change. |
| 10 | **Gmail SMTP mailbox** | Outbound notification emails (`lib/email/mailer.ts`) | Sends via `smtp.gmail.com:587` as **`agsdev@allianceglobalsolutions.com`** using a Gmail App Password (requires 2-Step Verification on agsdev@) | Yes (emails silently stop without it) | Sends are fire-and-forget, so a broken login fails **silently**. If HR stops getting notification emails, check the App Password first (myaccount.google.com/apppasswords on agsdev@). Changing agsdev@'s Google password revokes every App Password. |
| 11 | **Web Push VAPID keypair** | Browser push notifications (`lib/push/send.ts`) | `VAPID_PUBLIC_KEY` / `VAPID_PRIVATE_KEY` / `VAPID_SUBJECT` | Yes | Hand over the existing keypair as-is. **Do not regenerate** it, because new keys invalidate every existing push subscription. Update `VAPID_SUBJECT` to the new owner's contact email. |
| 12 | **Google Workspace** (`allianceglobalsolutions.com`) | Sign-in is restricted to this domain (`hd` param in `lib/firebase/client.ts`); Email HR opens Gmail compose | Company IT | Yes | No transfer needed, but company IT should know the app depends on it. |
| 13 | OpenAI | Feed content moderation ("Alexa", `lib/guardrails/moderation.ts`) | `OPENAI_API_KEY`, **not set in production** | Optional | Fails open: without a key, posts are published unmoderated. Add a key from the new owner's account to enable moderation. |
| 14 | Upstash Redis | Distributed rate limiting (`lib/guardrails/rateLimiter.ts`) | `UPSTASH_REDIS_REST_URL/TOKEN`, **not set in production** | Optional | Falls back to per-instance in-memory limits. Create a database under the new owner's account to enable it. |
| 15 | Sentry | Error and trace reporting (`instrumentation*.ts`, `next.config.ts`) | `SENTRY_DSN` / `NEXT_PUBLIC_SENTRY_DSN`, **not set** | Optional | Inert until a DSN is set. Create a project under the new owner's account if wanted. |

> **Unrelated projects in the same Vercel team:** `agstardinesstracker` (agsattendancehub.vercel.app) and `ags-performance-hub` are separate apps in the same *AGS* Vercel team. They are **not** part of this repo.

**Post-cutover checks.** After changing env vars and redeploying, verify production:

- sign in
- award points
- upload an image
- post with a GIF
- ask Ally a question (this tests the database and the `policy-docs` storage)
- open a document under Documents
- check that an email arrives
- check that the notification bell updates without a page reload (Realtime)

### 1.3 Secrets to rotate after handover

The previous developer had these values locally (in `.env.local`, and in local tooling config like `.mcp.json`), so rotate them now that agsdev is in control. The **Supabase** credentials don't need rotating: the new project `qtugwwotqnznpnttzvhl` was created with a fresh database password and fresh API keys, and the old project's credentials stop mattering once it's deleted (1.1).

| Secret | Where to rotate | Then update |
|--------|-----------------|-------------|
| Firebase **Admin service account key** | GCP → IAM → Service accounts → `firebase-adminsdk-…` → Keys → add new key, delete old | `FIREBASE_CLIENT_EMAIL`, `FIREBASE_PRIVATE_KEY` |
| Cloudinary API secret | Cloudinary → Settings → API Keys → generate new, revoke old | `CLOUDINARY_API_KEY`, `CLOUDINARY_API_SECRET` |
| Groq API key | Groq console → API Keys | `GROQ_API_KEY` |
| GIPHY key | GIPHY developers dashboard | `NEXT_PUBLIC_GIPHY_API_KEY` |
| Gmail App Password | Google Account → Security → App passwords (on the new mailbox) | `EMAIL_PASS` |
| `CRON_SECRET`, `BOOTSTRAP_SECRET` | Any random string (`openssl rand -hex 32`) | Vercel env vars |

Do **not** rotate the VAPID keys (see 1.2 #11). The Firebase web API key (`NEXT_PUBLIC_FIREBASE_*`) is public by design and doesn't need rotating.

### 1.4 Environment variables (complete list)

Values live in **Vercel → `agsone` → Settings → Environment Variables**.
- **Production and Preview** hold the deployed values. Secrets are stored as the *Secret* type, which can't be read back.
- **Development** holds a complete working local set: the same services, with `NEXT_PUBLIC_APP_URL=http://localhost:3010`.

To set up locally, run `vercel link --project agsone --scope agsone && vercel env pull .env.local`, then copy `DATABASE_URL` and `DIRECT_URL` into `.env`, which Prisma CLI commands read. Both files are gitignored. Development leaves out the unused variables listed below, and `UPSTASH_*`, which is optional and falls back to in-memory limits.

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
| `NEXT_PUBLIC_CLOUDINARY_CLOUD_NAME` | **Required** | `lib/cloudinary/*`, image URLs | `cdnl9dwe`. Inlined at build time, so redeploy after changing it. |
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

**Copying to a new Supabase project** (how the 2026-10-14 move to agsdev was done; reuse it for any future move). Requires the Supabase CLI and Docker. Take the app offline first so no writes are lost.

```bash
# 1. Dump from the old project (CLI logged in as the old owner, linked to the old ref)
supabase db dump --linked -f roles.sql --role-only
supabase db dump --linked -f schema.sql
supabase db dump --linked -f data.sql --use-copy --data-only
supabase storage cp -r ss:///policy-docs ./storage --experimental --linked

# 2. Create the new project (CLI logged in as the new owner), same region as Vercel
supabase projects create "AGS One" --org-id <org-id> --region ap-southeast-2 --db-password <pw>

# 3. Remove COPY blocks for empty tables from data.sql. Internal Supabase tables such as
#    storage.buckets_vectors reject inserts and abort the restore.
# 4. Restore in one transaction, with triggers off so circular foreign keys load cleanly
psql --single-transaction -v ON_ERROR_STOP=1 -f roles.sql -f schema.sql \
  -c 'SET session_replication_role = replica' -f data.sql \
  --dbname "postgresql://postgres.<new-ref>:<pw>@<pooler-host>:5432/postgres"
```

5. **Upload the storage files.** The restore already created the `storage.objects` metadata rows, so a plain upload fails with `409 Duplicate`. Upload each file to `policy-docs/<file name>` through the Storage API with the header `x-upsert: true`. Don't use `storage cp -r` into the bucket root: it nests the files under an extra `policy-docs/` folder.
6. **Verify.** Compare per-table row counts with the dump, and call `match_document_chunks('leave policy', 8)` to confirm Ally's search works. Then swap the five Supabase env vars in Vercel (1.1).

Nothing in code needs to change when the Supabase project changes: the CSP and image config in `next.config.ts` allow any `*.supabase.co` host.

**Moving to a different Cloudinary cloud** (done 2026-10-14, `dfrmh2iee` → `cdnl9dwe`). Uploads store the full `secure_url`, cloud name included, in the database. Org-chart overrides store only `User.orgChartPhotoPublicId`, and their URL is built from the env var at render time.

1. List every asset on the old cloud with the Admin API (`resources/image|video|raw`).
2. Upload each one to the new cloud by URL, with the same `public_id`, `overwrite=false`, and the same `resource_type`. Copy every asset, not just referenced ones.
3. In one transaction, `replace()` `res.cloudinary.com/<old>/` with `res.cloudinary.com/<new>/` in these columns:
   - `FoodListing.imageUrls`
   - `SocialPost.imageUrls`
   - `SocialPost.videoUrl`
   - `Reward.imageUrls`
   - `MedicineItem.imageUrl`
   - `HrDocument.fileUrl`
   - `AuditLog.beforeState`
4. Swap `NEXT_PUBLIC_CLOUDINARY_CLOUD_NAME`, `CLOUDINARY_API_KEY`, and `CLOUDINARY_API_SECRET` in Vercel, then redeploy.

Notes:
- The `/v<version>/` segment and the transforms baked into stored URLs keep working on the new cloud.
- New Cloudinary accounts block PDF delivery (`401`) until *Allow delivery of PDF and ZIP files* is enabled.
- Avatars are Google profile photos, not Cloudinary, so they're unaffected.

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
