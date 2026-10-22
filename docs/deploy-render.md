# Render Blueprint: OpenReply

The root `render.yaml` defines four paid resources in Frankfurt: a native Node
web service, a native Node background worker, PostgreSQL 16, and a Redis-compatible
Render Key Value instance. No Docker build is used. Render uses its own native
Node build system, not Railway's Railpack; the standard npm commands and bounded
Node engine also work with Railpack. `.node-version` and `NODE_VERSION` select
Node 22, and `package.json` requires >=22.12 and <23 (Prisma 7 compatible).

## Before creating resources

Review Render's current [pricing](https://render.com/pricing). The Blueprint uses
paid compute IDs: `0.5c-512mb` for both processes, `0.1c-256mb` for Postgres plus
5 GB storage, and `256mb` for Key Value. Workers and pre-deploy commands require
paid compute, and Key Value persistence requires a paid instance. These are
small starting sizes, not a capacity guarantee; increase memory if your workload
needs it. Creating a Blueprint incurs charges. Merely pushing these files does
not create resources. Auto-deploys and preview environments are disabled.

## Manual variables (enter only in Render)

During initial Blueprint creation, Render prompts for `sync: false` variables on
**openreply-web**. No actual credentials belong in Git or a chat message.

| Variable | Value |
| --- | --- |
| `ENCRYPTION_KEY` | Exactly 64 hexadecimal characters (32 random bytes). Generate securely, for example `openssl rand -hex 32`, outside Git. Preserve an existing key when migrating encrypted accounts. Render's `generateValue` is base64 and is unsuitable here. |
| `EMAIL_FROM` | Sender address verified with your email provider, e.g. `OpenReply <login@your-domain>`. |
| `RESEND_API_KEY` | Your Resend API key. For SMTP instead, leave this blank and add `EMAIL_SERVER` manually to Web before testing login. |
| `ALLOWED_EMAILS` | Comma-separated administrator/team email addresses. Blank permits public signup into separate workspaces. |
| `INSTAGRAM_APP_ID` | Instagram product app ID, not the Facebook app ID. |
| `INSTAGRAM_APP_SECRET` | Instagram product app secret. |
| `FACEBOOK_APP_SECRET` | Meta app's Basic settings app secret; used for webhook signature verification. |

The Meta variables can stay blank while inspecting the app. For the direct Meta
connection of Instagram 1Fit, fill them before starting OAuth. If you choose
Zernio, configure its credentials in Settings instead (see `docs/zernio.md`).

Render generates `NEXTAUTH_SECRET`, `CRON_SECRET`, and `WEBHOOK_VERIFY_TOKEN`.
It supplies `DATABASE_URL` and `REDIS_URL` from **internal** datastore connection
strings. Both datastores reject public connections (`ipAllowList: []`). BullMQ
uses `noeviction`; Key Value has disk persistence enabled.

Web's `NEXTAUTH_URL` references its `RENDER_EXTERNAL_URL`, so it uses the actual
assigned HTTPS address, not a guessed `onrender.com` hostname. Worker references
Web's `NEXTAUTH_URL`, `ENCRYPTION_KEY`, and Instagram app credentials. Enter the
encryption key only once, on Web. After changing shared values, sync the
Blueprint and redeploy both services: cross-service references update on sync,
not immediately. For a custom domain, override Web's NEXTAUTH_URL reference in
the Blueprint with the intended public HTTPS URL and resync both services.

## Build, migration and start lifecycle

| Service | Build | Pre-deploy | Start |
| --- | --- | --- | --- |
| Web | `npm ci --include=dev && npm run build` | `npm run db:migrate` | `npm run start -- --hostname 0.0.0.0 --port "$PORT"` |
| Worker | `npm ci --include=dev && npm run db:generate` | `npm run db:migrate` | `npm run worker` |

`build` already runs `prisma generate` before `next build`. Worker runs TypeScript
through `tsx` and resolves `@/*` using `tsconfig.json`; it needs the generated
Prisma client and source tree, not `.next`. Do not prune dev dependencies:
`tsx` and the Prisma CLI are declared there. `--include=dev` preserves them even
with NODE_ENV=production. Worker also loads `.env` for local runs; Render injects
the same variables directly without an `.env` file.

Migrations use `prisma migrate deploy`, never `migrate dev`, schema resets, or
`db push`. Both services run the same idempotent migration command before start,
so a separately deployed worker cannot start against an unmigrated database.
Prisma uses a PostgreSQL advisory lock; if simultaneous deploys time out waiting
for that lock, retry after the other migration finishes. Build commands never
require database network access, which belongs in pre-deploy/runtime. Always
review future migrations for compatibility with the previously running version.

Render's Web health check is `/login`, independent of worker startup. For full
operational monitoring, check `/api/health` after **both** services start: it
must return HTTP 200 with database, Redis, queue, and worker healthy. A working
login page alone does not prove delivery or email configuration is correct.

## Later deployment and Instagram setup

When you are ready to deploy (not performed as part of this change):

1. In Render choose New > Blueprint, connect your fork and select the branch
   containing `render.yaml`. Review the four resources, region, billing, and
   prompted variables before creating them. Initial Blueprint creation starts
   deployment even though future automatic deploys are disabled.
2. After both processes start, check `/login` and `/api/health` on Web's assigned
   HTTPS address. Verify email magic-link login with an allowed address.
3. Configure Meta OAuth at `https://YOUR_DOMAIN/api/instagram/callback` and the
   webhook at `https://YOUR_DOMAIN/api/webhook`. Copy the generated
   WEBHOOK_VERIFY_TOKEN from Render into Meta. Subscribe to `comments` and
   `messages`; add/accept the professional 1Fit account's tester role as needed.
4. Review privacy/terms/data-deletion pages and Meta's Live/App Review requirements,
   then connect the account in Settings. See `docs/setup.md` for full details.
5. Schedule daily authenticated GET requests to `/api/cron/refresh-tokens` and
   `/api/cron/snapshot-followers`, using `Authorization: Bearer <CRON_SECRET>`.
   This four-resource Blueprint does not add a billable fifth cron service.
   Render does not execute `vercel.json`. Use a Render Cron Job or an existing
   external scheduler with its own protected secret configuration. The existing
   `scripts/cron.sh` is a long-running scheduler, not a one-shot Render Cron Job.
   `attach-next-reel` is already polled by the worker. Daily token refresh must
   be scheduled before relying on long-term Instagram operation.

## Validation

Check YAML and cross-service references, then validate with Render's published
[JSON Schema](https://render.com/schema/render.yaml.json), using a validator
supporting JSON Schema 2020-12. Authenticated Render CLI validation is also
available (`render blueprints validate render.yaml`); it does not deploy.
Static validation and a local production build do not test provisioning,
Render account permissions, provider credentials, or real Instagram delivery.

References: [Blueprint specification](https://render.com/docs/blueprint-spec),
[Node version selection](https://render.com/docs/node-version),
[pre-deploy commands](https://render.com/docs/deploys#pre-deploy-command),
[Key Value](https://render.com/docs/key-value).
