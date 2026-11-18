# HANDOFF — helm-postiz access-gated steps (2026-11-18)

You've been handed this because the current operator's accounts don't cover three
gated areas: **GitHub repo admin**, **DO Managed Database (doadmin)**, **Meta for
Developers / Gandi DNS / DO API token**. Everything else is already done.

This document is self-contained. The companion README.md covers the full picture
(architecture, gotchas, deploy, integration).

## Status when this was written

DONE (no access needed — already complete):

- Repo scaffold at `GliaX/helm-postiz` — postiz / temporal / n8n values + CD workflow.
  All three charts validated by `helm template` against the real upstream charts.
- Local tooling on the workstation (`~/.local/bin`): kubectl v1.37.2, helm v4.3.2,
  doctl v2.6.0.
- Deployment model: push to `main` (or Actions → deploy → Run workflow) → GitHub
  Actions authenticates to DOKS and runs `helm upgrade --install` with secrets
  pulled from the GitHub Environment `production`.

NOT DONE (needs your access — the three steps below):

1. GitHub Environment `production` + 5 secrets      (GliaX/helm-postiz admin)
2. DO Managed PostgreSQL bootstrap                   (doadmin / DO console)
3. Meta app + DNS + hand the DO token back           (developers.facebook.com,
                                                      Gandi, DO console)

Recommended order: 2 → 3 → 1 → deploy. (Step 1's secrets need the PG host/password
from step 2; Facebook keys from step 3 are optional at first deploy.)

---

## Step 1 — GitHub Environment `production` + secrets

Repo: https://github.com/GliaX/helm-postiz → Settings → Environments → New
environment → name it exactly `production`. (Optionally add a required reviewer
here — that gives you a manual approval gate on prod deploys.)

Add these **Environment secrets** (repo Settings → Environments → production →
Add environment secret):

| Secret name | Contents |
|---|---|
| `DIGITALOCEAN_ACCESS_TOKEN` | DO API token — see "DO token" under Step 3 |
| `CLUSTER_NAME` | DOKS cluster name — `doctl kubernetes cluster list`, or the name shown in the DO console |
| `POSTIZ_SECRET_VALUES` | Full file contents of `postiz/values.secret.yaml` (below) |
| `TEMPORAL_SECRET_VALUES` | Full file contents of `temporal/values.secret.yaml` (below) |
| `N8N_SECRET_VALUES` | Full file contents of `n8n/values.secret.yaml` (below) |

The three `*_SECRET_VALUES` secrets are the gitignored overlays as literal text.
Build them like this (never commit them):

```bash
git clone git@github.com:GliaX/helm-postiz.git && cd helm-postiz
cp postiz/values.secret.example.yaml   postiz/values.secret.yaml
cp temporal/values.secret.example.yaml temporal/values.secret.yaml
cp n8n/values.secret.example.yaml      n8n/values.secret.yaml
openssl rand -base64 48   # JWT_SECRET
openssl rand -base64 32   # ENCRYPTION_KEY, and again for n8n encryption_key
openssl rand -base64 32   # redis password (generate once)
```

Then fill in each overlay and paste its **entire content** into the matching
environment secret. Field-by-field:

**postiz/values.secret.yaml**
- `secrets.DATABASE_URL` — `postgresql://postiz:<PG postiz password>@<PG host>:25061/postiz?sslmode=require` (host + password from Step 2)
- `secrets.REDIS_URL` — `redis://:<redis password>@postiz-redis-master.postiz.svc.cluster.local:6379/0` (password must equal `redis.auth.password` in the same file)
- `secrets.JWT_SECRET`, `secrets.ENCRYPTION_KEY` — generated above
- `secrets.FACEBOOK_APP_ID` / `FACEBOOK_APP_SECRET` — from Step 3's Meta app (may be empty at first deploy; add later and re-run the deploy)
- `secrets.EMAIL_PASS` — optional (SMTP `erp@glia.org` on `mail.emlondon.ca`, same credential the ERPNext Email Account uses). If set, also uncomment the `env:` email block in that file.
- `redis.auth.password` — the redis password (keep in sync with REDIS_URL)

**temporal/values.secret.yaml**
- `server.config.persistence.datastores.default.sql.password` and
  `.../visibility/sql/password` — the `temporal` role password from Step 2 (same
  password for both stores)

**n8n/values.secret.yaml**
- `main.secret.n8n.encryption_key` — generated above. **Back this up** (password
  manager/vault): losing it makes every n8n credential unreadable.

Also note: `temporal/values.yaml` (committed) has a placeholder PG host —
`connectAddr: "REPLACE-ME.db.ondigitalocean.com:25061"` (two occurrences). Replace
with the real host from Step 2 and commit normally (hostnames aren't secret).
Secrets overlays go only into the Environment secrets.

---

## Step 2 — DO Managed PostgreSQL bootstrap

The managed DB cluster already exists. Two things:

**a. Trusted sources:** DO console → Databases → (the cluster) → Settings →
Trusted Sources → add the DOKS cluster (so pods can reach :25061). Equivalent:
`doctl db firewalls append <db-id> --type k8s <cluster-uuid>`.

**b. Roles + databases** — connect as `doadmin` (DO console web SQL editor, or
`doctl db connect <db-id>` with psql installed) and run:

```sql
CREATE ROLE postiz   LOGIN PASSWORD '<REPLACE>';
CREATE ROLE temporal LOGIN PASSWORD '<REPLACE>';
CREATE DATABASE postiz              OWNER postiz;
CREATE DATABASE temporal            OWNER temporal;
CREATE DATABASE temporal_visibility OWNER temporal;
```

(No further grants needed — owners manage their own schemas; Temporal's schema job
runs as `temporal`. ERPNext still runs on its in-cluster MariaDB; migrating it to
this managed DB is out of scope here.)

Record: **PG host** (`<name>.db.ondigitalocean.com`, port **25061**) and the two
passwords → used in Step 1's overlays and in `temporal/values.yaml`.

---

## Step 3 — Meta app, DNS, DO token

### Meta app (Facebook + Instagram — one app covers both)

Follow docs.postiz.com → Providers → Facebook (full walkthrough). Summary:

1. developers.facebook.com → Create app → type "Other"/Business, business portfolio selected.
2. Add **Login for Business** (Facebook) and, if using standalone Instagram, **Instagram Business Login**.
3. Redirect URIs — register all that apply:
   - `https://postiz.glia.org/integrations/social/facebook`
   - `https://postiz.glia.org/integrations/social/instagram`
   - `https://postiz.glia.org/integrations/social/instagram-standalone` (only for the standalone flow)
4. Advanced permissions: `pages_show_list`, `business_management`, `pages_manage_posts`,
   `pages_manage_engagement`, `pages_read_engagement`, `read_insights`
   (Instagram standalone adds `instagram_basic`, `instagram_content_publish`,
   `instagram_manage_comments`, `instagram_manage_insights`).
5. App Mode → **Live** (in Development mode, posts/images are visible only to app roles).
6. Basic settings → copy **App ID + App Secret** → these become
   `secrets.FACEBOOK_APP_ID` / `FACEBOOK_APP_SECRET` in the Step 1 overlay (add now or later).

Business verification is only needed once the app serves people outside its roles.
Bluesky needs **no app registration** — channels connect with handle + App Password
inside the Postiz UI later.

### DNS (Gandi — glia.org zone)

Get the ingress LB IP:

```bash
kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

Create records in the glia.org zone: `postiz` → A (or CNAME to the LB hostname) and
`n8n` → A. Same pattern as `asset.glia.org` on the ERPNext side. cert-manager will
issue Let's Encrypt certs as soon as the names resolve.

### DO token (for the CD workflow)

DO console → API → Generate New Token → **Custom** scope: enable **Kubernetes
manage** (read + write) and **Database read** if you want `doctl db` convenience;
set an expiry. Don't hand out the account-wide token. This token's value goes into
the `DIGITALOCEAN_ACCESS_TOKEN` environment secret (Step 1).

---

## After the three steps: deploy + verify

Nothing else needs your access. Trigger: repo → Actions → **deploy** → Run workflow
→ app: `all`. The workflow creates namespaces, applies the uploads PVC, deploys
temporal → postiz → n8n (`--atomic`, waits for rollout). Pushes to `main` touching
the app folders do the same going forward.

Verification (anyone with the kubeconfig):

```bash
kubectl -n postiz get pods,certificate                    # postiz + temporal Running, postiz-tls True
kubectl -n n8n    get pods,certificate                    # n8n Running, n8n-tls True
```

Then the first-run checklist from README.md (first Postiz signup becomes the owner
and registration self-disables — do it promptly after deploy).

## Questions / files that explain more

- `README.md` — architecture, gotchas (read §"CRITICAL gotchas" before manual helm
  commands: the postiz scale-to-0 rule matters), deploy commands, integration recipes.
- helm-erpnext repo — same conventions; ERPNext is untouched by this project.
