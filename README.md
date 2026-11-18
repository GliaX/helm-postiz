# helm-postiz

Infrastructure-as-code for Glia's **Postiz** social-media scheduler (`postiz.glia.org`)
and the **n8n** integration layer (`n8n.glia.org`) that bridges Postiz ↔ ERPNext
(`asset.glia.org`, repo [`GliaX/helm-erpnext`](https://github.com/GliaX/helm-erpnext)).

Same philosophy as helm-erpnext: **Helm-as-Code**, sanitized values in git, secrets in
gitignored `values.secret.yaml` overlays (deployed from GitHub Environment secrets).
`kubectl apply` / manual dashboard edits should be avoided going forward.

---

## What's deployed

| App | Release | Namespace | Chart (pinned) | Image (pinned) | Public URL |
|---|---|---|---|---|---|
| Postiz | `postiz` | `postiz` | `postiz-app` **1.0.5** (OCI `ghcr.io/gitroomhq/postiz-helmchart`) | `ghcr.io/gitroomhq/postiz-app:v2.26.0` | https://postiz.glia.org |
| Temporal | `temporal` | `postiz` | `temporal` **1.8.0** (`https://go.temporal.io/helm-charts`) | `temporalio/*` 1.32.x | internal only |
| n8n | `n8n` | `n8n` | `n8n` **2.1.1** (OCI `8gears.container-registry.com/library/n8n`) | chart appVersion | https://n8n.glia.org |

Shared infrastructure (already in place):

- **DOKS cluster** with nginx ingress + cert-manager (`letsencrypt-prod` ClusterIssuer) —
  same ingress/tls conventions as helm-erpnext (`values.yaml` of that repo).
- **DO Managed PostgreSQL**: hosts `postiz`, `temporal`, `temporal_visibility` (+ ERPNext
  DBs when migrated). The Postiz chart's in-cluster PostgreSQL subchart is **disabled**.
- **Redis**: in-cluster bitnami subchart (with the `bitnamilegacy` image override, same
  trick as helm-erpnext). Moving to DO managed Valkey is a deliberate future decision.

### Architecture

```
                     GitHub (GliaX/helm-postiz)
                        │ push to main → Actions (environment: production)
                        ▼
┌──────────────────────── DOKS ────────────────────────────────┐
│  ns: postiz                              ns: n8n             │
│  ┌───────────────┐   ┌───────────────────┐   ┌───────────┐   │
│  │ Postiz :5000  │   │ Temporal (pgsql)  │   │ n8n :5678 │   │
│  └──────┬────────┘   └─────────┬─────────┘   └─────┬─────┘   │
│         │ REDIS_URL            │ TEMPORAL_ADDRESS   │         │
│  ┌──────▼────────┐             │             ┌──────▼──────┐  │
│  │ redis (chart) │             │             │ SQLite+PVC  │  │
│  └───────────────┘             │             └──────┬──────┘  │
│        ingress postiz.glia.org │                    │         │
└────────────────────────────────┼────────────────────┼─────────┘
                                 ▼                    ▼
                  DO Managed PostgreSQL (postiz,    REST: ERPNext asset.glia.org
                  temporal, temporal_visibility)    REST: Postiz /public/v1
```

### Integration (n8n ↔ ERPNext ↔ Postiz)

Postiz exposes a public API at `https://postiz.glia.org/public/v1` with `Authorization:
<API key>` (create the key in Postiz → Settings → API Keys). Official n8n node:
[`n8n-nodes-postiz`](https://www.npmjs.com/package/n8n-nodes-postiz) (install via
n8n UI → Settings → Community nodes). Reference flows:

1. **ERPNext → scheduled social posts**: ERPNext document events (webhooks on a custom
   DocType, e.g. "Social Post" with approval status) → n8n webhook →
   `POST /public/v1/upload` (media, if any) → `POST /public/v1/posts` (`type: schedule`)
   → write the returned Postiz post id back into the ERPNext doc.
2. **Direct events**: Blog Post / Item publish → n8n → draft/schedule in Postiz.
3. **Analytics back**: n8n cron → `GET /public/v1/posts` (analytics fields) → upsert
   custom ERPNext DocType → Glia reports/number cards (same manifest pattern as the
   donor dashboard).

Notes: the create-post endpoint is rate-limited to `API_LIMIT`/hour (we set 90 —
schedule multiple posts per request); generate exact payloads with Postiz's payload
wizard (`https://postiz.glia.org/modal/dark/all`). ERPNext API auth is the standard
`Authorization: token KEY:SECRET` pair (pattern from `glia-sync-secrets` on the
erpnext side).

---

## CRITICAL gotchas (read before deploying)

1. **RWO uploads PVC → scale to 0 before every postiz helm upgrade.** The upstream chart
   has no `deployment.strategy` knob, so rollouts default to RollingUpdate(maxSurge=1,
   maxUnavailable=0); with the RWO `postiz-uploads` PVC the new pod deadlocks on
   Multi-Attach exactly like helm-erpnext's sites PVC (see that repo's values.pin.yaml).
   ```bash
   kubectl -n postiz scale deployment/postiz --replicas=0
   helm -n postiz upgrade --install postiz ...   # brings it back to 1
   ```
   The CD workflow does this automatically. (n8n's chart defaults to `Recreate` — no
   issue there. Postiz storage on Cloudflare R2 would also remove the PVC entirely.)
2. **Temporal is REQUIRED since Postiz v2.12** for scheduled posts; the upstream postiz
   chart does NOT ship it, which is why this repo deploys the temporal chart separately.
   Deploy the `temporal` release (once) before scheduling posts. Postiz must be able to
   resolve `TEMPORAL_ADDRESS=temporal-frontend.postiz.svc.cluster.local:7233`.
3. **`temporal` chart ≥1.0 removed all datastore subcharts** — `cassandra`/`mysql`/
   `postgresql`/`elasticsearch` top-level keys hard-fail rendering. SQL config goes under
   `server.config.persistence.datastores`, and the chart requires **`connectAddr`
   (`host:port`)** — separate `host:`/`port:` fields are ignored by the schema job.
4. **Bitnami images moved to `docker.io/bitnamilegacy`** (frozen) — the redis subchart
   image is overridden here, same as helm-erpnext. Plan the DO managed Valkey migration
   before the legacy tag disappears.
5. **n8n chart env flattening**: nested keys join with `_` but camelCase is NOT split —
   use snake_case keys in `main.config` (`community_packages`, `editor_base_url`,
   `encryption_key`) or you'll silently produce wrong env var names.
6. **n8n `encryption_key` loss = all n8n credentials unreadable.** Back it up.
7. **Cloudflare R2 is Postiz's upstream default for channel avatars** — with local
   storage, avatar saving may fail (cosmetic; posting works). Not configured for now.
8. **Postiz secret rotation**: `JWT_SECRET` rotation kills sessions; `ENCRYPTION_KEY`
   rotation forces reconnecting every social channel (tokens are encrypted at rest since
   we set `ENCRYPTION_KEY`).

---

## Layout

```
helm-postiz/
├── postiz/
│   ├── values.yaml                   # sanitized Glia config
│   ├── values.secret.example.yaml    # template — copy to values.secret.yaml (gitignored)
│   └── manifests/postiz-uploads-pvc.yaml   # applied pre-helm (chart has no PVC template)
├── temporal/
│   ├── values.yaml
│   └── values.secret.example.yaml
├── n8n/
│   ├── values.yaml
│   └── values.secret.example.yaml
└── .github/workflows/deploy.yml      # verify (helm template) + deploy per app
```

## One-time bootstrap

### 1. Local tooling (user-local, no sudo needed)

```bash
mkdir -p ~/.local/bin
# kubectl
curl -sL -o ~/.local/bin/kubectl "$(curl -s https://dl.k8s.io/release/stable.txt | sed 's|^|https://dl.k8s.io/release/|')/bin/linux/amd64/kubectl"
# helm
# doctl: https://github.com/digitalocean/doctl/releases → linux-amd64.tar.gz
export PATH="$HOME/.local/bin:$PATH"
doctl auth init                       # DO API token (create: DO console → API)
doctl kubernetes cluster list
doctl kubernetes cluster kubeconfig save <cluster-name>
kubectl get nodes
```

### 2. Managed PostgreSQL bootstrap

Ensure the DOKS cluster is a **trusted source** on the managed DB (DO console →
Databases → Settings → Trusted Sources, or `doctl db firewalls append <db-id> --type
droplet|k8s ...`). Then create the DBs (once, as `doadmin`):

```sql
CREATE ROLE postiz   LOGIN PASSWORD '<REPLACE>';
CREATE ROLE temporal LOGIN PASSWORD '<REPLACE>';
CREATE DATABASE postiz              OWNER postiz;
CREATE DATABASE temporal            OWNER temporal;
CREATE DATABASE temporal_visibility OWNER temporal;
```

(`doctl db connect <db-id>` needs psql; the DO console web SQL editor works too.)

### 3. Meta app (Facebook + Instagram) — done in developers.facebook.com

One app covers both. Steps: docs.postiz.com → Providers → Facebook / Instagram.
Essentials:

- Redirect URIs (Login for Business / Instagram Business Login):
  - `https://postiz.glia.org/integrations/social/facebook`
  - `https://postiz.glia.org/integrations/social/instagram`
  - (`https://postiz.glia.org/integrations/social/instagram-standalone` only if using
    the standalone flow instead — needs `INSTAGRAM_APP_ID/SECRET` instead)
- Advanced permissions (FB): `pages_show_list`, `business_management`,
  `pages_manage_posts`, `pages_manage_engagement`, `pages_read_engagement`,
  `read_insights`. (Instagram adds: `instagram_basic`, `instagram_content_publish`,
  `instagram_manage_comments`, `instagram_manage_insights`.)
- App Mode → **Live** (else posts/images visible only to app roles).
- Personal-use installs can skip app review; public/business use needs Meta business
  verification for the advanced scopes.
- Copy App ID/Secret → `postiz/values.secret.yaml` (`FACEBOOK_APP_ID/SECRET`).

**Bluesky needs no app registration** — connect via handle + App Password inside the
Postiz UI (create the App Password in Bluesky → Settings → Privacy and data).

### 4. DNS (Gandi)

Point `postiz.glia.org` and `n8n.glia.org` (A/ALIAS/CNAME) at the cluster ingress LB:

```bash
kubectl -n ingress-nginx get svc ingress-nginx-controller -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

cert-manager issues certificates after DNS resolves (watch
`kubectl -n postiz get certificate postiz-tls`).

### 5. GitHub

1. Create repo `GliaX/helm-postiz`, push this repo (README §Deploy, below).
2. Repo → Settings → Environments → create **production** (optionally require a
   reviewer for prod deploys).
3. Add Environment secrets:
   - `DIGITALOCEAN_ACCESS_TOKEN` — DO API token
   - `CLUSTER_NAME` — DOKS cluster name
   - `POSTIZ_SECRET_VALUES` — full contents of `postiz/values.secret.yaml`
   - `TEMPORAL_SECRET_VALUES` — full contents of `temporal/values.secret.yaml`
   - `N8N_SECRET_VALUES` — full contents of `n8n/values.secret.yaml`

## Deploy

**Via CD (default):** push to `main` touching `postiz/**`, `temporal/**`, or `n8n/**`;
or Actions → deploy → Run workflow (choose the app). The workflow renders all apps
against the example overlays first (catches YAML errors), then helm-upgrades only what
changed, with `--atomic`.

**Manual:**

```bash
export PATH="$HOME/.local/bin:$PATH"
kubectl apply -f postiz/manifests/postiz-uploads-pvc.yaml

# temporal (once, before scheduling posts)
helm repo add temporal https://go.temporal.io/helm-charts && helm repo update
helm -n postiz upgrade --install temporal temporal/temporal --version 1.8.0 \
  -f temporal/values.yaml -f temporal/values.secret.yaml --atomic --timeout 10m

# postiz (scale-to-0 gotcha!)
kubectl -n postiz scale deployment/postiz --replicas=0
helm -n postiz upgrade --install postiz \
  oci://ghcr.io/gitroomhq/postiz-helmchart/charts/postiz-app --version 1.0.5 \
  -f postiz/values.yaml -f postiz/values.secret.yaml --atomic --timeout 10m

# n8n
helm -n n8n upgrade --install n8n \
  oci://8gears.container-registry.com/library/n8n --version 2.1.1 \
  -f n8n/values.yaml -f n8n/values.secret.yaml --atomic --timeout 10m
```

## First-run checklist

1. `https://postiz.glia.org` → sign up **once** (registration self-disables after the
   first user). Invite staff from Settings (email via nodemailer SMTP if configured).
2. Connect channels (Facebook Page, Instagram, Bluesky app-password, others).
3. Postiz → Settings → API Keys → create the key n8n will use.
4. `https://n8n.glia.org` → create owner account → Settings → Community nodes →
   install `n8n-nodes-postiz` → Credentials: Postiz API key + ERPNext token pair.
5. Build the first flow (ERPNext webhook → Postiz schedule) and a test post.

## Backups

- **Managed PostgreSQL**: automatic DO backups (retention per DB settings).
- **postiz-uploads PVC / n8n PVC**: DO volume snapshots before changes:
  `doctl compute volume snapshot <volume-id> --name pre-upgrade` (list volumes via
  `doctl compute volume list`).
- **Secrets overlays** (`values.secret.yaml` × 3): store copies in the team vault.
  Losing them means recreating DB users/keys and reconnecting every channel.

## Upgrading versions

- Postiz image: bump `postiz/values.yaml` `image.tag` (check release notes for new
  required env vars; pin deliberately, never `latest`).
- Charts: bump the `--version` pins in the workflow + manual commands together
  (upstream postiz chart 1.0.5 / temporal 1.8.0 / n8n 2.1.1 today).
- After a temporal chart bump: read its UPGRADING.md (deprecation validation fails
  rendering loudly — good).

## Secret hygiene

`values.yaml` files are secret-free (verified at commit time). The three
`values.secret.yaml` overlays and the GitHub Environment secrets are the only secret
locations. Same rule as helm-erpnext: never paste secrets into issues, chat, or
non-overlay files.
