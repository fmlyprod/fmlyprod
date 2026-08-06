# Scout Lane — SaaS Architecture

**Version:** 1.0  
**Status:** Pre-build specification  
**Source:** Derived from fmlyprod.com prototype (fully working proof of concept)

---

## 1. What the Prototype Proved

fmlyprod.com is a fully working production gallery tool built as a single-file app on GitHub Pages. It proves the product concept and UX. Every screen, flow, and feature has been designed, tested, and used in real productions. The prototype limitations are infrastructure-only — not product or UX:

- GitHub API used as a write database → rate limits, no multi-tenancy
- Credentials hardcoded in HTML → can't onboard multiple companies
- Single GitHub PAT per admin → doesn't scale
- No billing, no self-service onboarding, no tenant isolation

The rebuild uses the same UX and feature set on the correct infrastructure.

---

## 2. Core Features (from prototype, carry forward 1:1)

### Admin (production company)
- Create and manage projects
- Upload photos/videos organized as: **Category → Folder → Media**
- Picdrop-style breadcrumb navigation (Category ▾ / Folder ▾)
- Assign client users to projects with username/password
- View analytics: who viewed what, how long, what they liked
- Download liked photos per client as ZIP
- Casting director upload portal (separate URL, restricted to specific folders)

### Client (casting director / external user)
- Login with username + password (no email required)
- Browse gallery: Category → Folder → Photos/Videos
- Like individual photos and entire folders
- View their own "Liked" collection
- Download their liked photos as ZIP
- Mobile-first, no account creation needed

---

## 3. Tech Stack

| Layer | Tool | Reason |
|---|---|---|
| **Frontend** | Vanilla HTML/CSS/JS (same as prototype) | Already built, no framework needed |
| **Hosting** | Cloudflare Pages | Instant global CDN, git-based deploys, free |
| **Backend API** | Cloudflare Workers | Already have one running; edge, global, auto-scales |
| **Database** | Cloudflare D1 (SQLite at edge) | Zero-latency reads, same Worker, generous free tier |
| **Media storage** | Cloudflare R2 | Already running; no egress fees, S3-compatible |
| **Auth** | JWT via Worker (custom, lightweight) | No external dep; admin JWT + client session tokens |
| **Billing** | Stripe | Industry standard; webhooks → Worker → D1 |
| **Email** | Resend or Cloudflare Email Routing | Transactional emails (invite, billing) |

**What does NOT change from the prototype:**
- Cloudflare R2 for all media (identical bucket structure, just re-keyed per tenant)
- Cloudflare Worker for upload auth and presigned URLs (extend existing worker)
- The full client-facing gallery UX
- The admin photo manager UX

---

## 4. Multi-Tenancy Model

Each **Organization** = one production company (one paying customer).

```
Organization (e.g. "FMLY Production")
  └── Projects (e.g. "DUNKIN BCN 2025", "APPLE Q4 2025")
        └── Categories (e.g. "DAY 1", "SCOUT", "REFS")
              └── Folders (e.g. "LOCATION A", "WARDROBE")
                    └── Media (photos, videos)
  └── Admin Users (company staff with login)
  └── Client Users (casting directors, external — per project)
  └── Billing (Stripe subscription)
```

**Tenant isolation:**
- Every D1 row has an `org_id` foreign key
- Every R2 key is prefixed: `{orgId}/{projectId}/...`
- JWT tokens carry `org_id` and are validated on every Worker request
- No cross-tenant data access possible at the API level

---

## 5. Database Schema (Cloudflare D1 / SQLite)

```sql
-- ─────────────────────────────────────────
-- ORGANIZATIONS (one per paying customer)
-- ─────────────────────────────────────────
CREATE TABLE organizations (
  id                    TEXT PRIMARY KEY,          -- nanoid, e.g. "org_k3j9..."
  name                  TEXT NOT NULL,             -- "FMLY Production"
  slug                  TEXT NOT NULL UNIQUE,      -- "fmly" → app.scoutlane.com/fmly
  plan                  TEXT NOT NULL DEFAULT 'trial',  -- trial | starter | pro | agency
  stripe_customer_id    TEXT,
  stripe_subscription_id TEXT,
  storage_used_bytes    INTEGER NOT NULL DEFAULT 0,
  logo_url              TEXT,
  created_at            TEXT NOT NULL DEFAULT (datetime('now')),
  trial_ends_at         TEXT
);

-- ─────────────────────────────────────────
-- ADMIN USERS (production company staff)
-- ─────────────────────────────────────────
CREATE TABLE admin_users (
  id            TEXT PRIMARY KEY,
  org_id        TEXT NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  email         TEXT NOT NULL,
  password_hash TEXT NOT NULL,              -- bcrypt
  role          TEXT NOT NULL DEFAULT 'admin',  -- owner | admin
  name          TEXT,
  created_at    TEXT NOT NULL DEFAULT (datetime('now')),
  last_login_at TEXT,
  UNIQUE(org_id, email)
);

-- ─────────────────────────────────────────
-- PROJECTS
-- ─────────────────────────────────────────
CREATE TABLE projects (
  id          TEXT PRIMARY KEY,
  org_id      TEXT NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name        TEXT NOT NULL,               -- "DUNKIN BCN 2025"
  slug        TEXT NOT NULL,               -- "dunkin-bcn-2025"
  status      TEXT NOT NULL DEFAULT 'active',  -- active | archived
  logo_url    TEXT,                        -- optional client-facing logo
  created_at  TEXT NOT NULL DEFAULT (datetime('now')),
  UNIQUE(org_id, slug)
);

-- ─────────────────────────────────────────
-- CATEGORIES (top-level folder groupings)
-- ─────────────────────────────────────────
CREATE TABLE categories (
  id          TEXT PRIMARY KEY,
  project_id  TEXT NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  name        TEXT NOT NULL,               -- "DAY 1 CASTING"
  position    INTEGER NOT NULL DEFAULT 0,
  created_at  TEXT NOT NULL DEFAULT (datetime('now'))
);

-- ─────────────────────────────────────────
-- FOLDERS (inside categories)
-- ─────────────────────────────────────────
CREATE TABLE folders (
  id          TEXT PRIMARY KEY,
  category_id TEXT NOT NULL REFERENCES categories(id) ON DELETE CASCADE,
  name        TEXT NOT NULL,               -- "LOCATION A"
  position    INTEGER NOT NULL DEFAULT 0,
  created_at  TEXT NOT NULL DEFAULT (datetime('now'))
);

-- ─────────────────────────────────────────
-- MEDIA (photos and videos)
-- ─────────────────────────────────────────
CREATE TABLE media (
  id            TEXT PRIMARY KEY,
  folder_id     TEXT NOT NULL REFERENCES folders(id) ON DELETE CASCADE,
  org_id        TEXT NOT NULL,             -- denormalized for fast queries
  r2_key        TEXT NOT NULL UNIQUE,      -- "org_abc/proj_xyz/cat_1/fold_2/photo.jpg"
  cdn_url       TEXT NOT NULL,             -- "https://media.scoutlane.com/org_abc/..."
  thumbnail_url TEXT,                      -- Cloudflare image resize URL
  filename      TEXT NOT NULL,
  mime_type     TEXT NOT NULL,
  size_bytes    INTEGER NOT NULL DEFAULT 0,
  width         INTEGER,
  height        INTEGER,
  duration_s    REAL,                      -- for videos
  position      INTEGER NOT NULL DEFAULT 0,
  uploaded_at   TEXT NOT NULL DEFAULT (datetime('now'))
);

-- ─────────────────────────────────────────
-- CLIENT USERS (external; per project)
-- ─────────────────────────────────────────
CREATE TABLE client_users (
  id            TEXT PRIMARY KEY,
  project_id    TEXT NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  org_id        TEXT NOT NULL,             -- denormalized
  username      TEXT NOT NULL,             -- "casting_director_1"
  password_hash TEXT NOT NULL,
  display_name  TEXT,
  active        INTEGER NOT NULL DEFAULT 1,
  created_at    TEXT NOT NULL DEFAULT (datetime('now')),
  last_login_at TEXT,
  UNIQUE(project_id, username)
);

-- Per-client category visibility (if empty = all categories visible)
CREATE TABLE client_category_access (
  client_id   TEXT NOT NULL REFERENCES client_users(id) ON DELETE CASCADE,
  category_id TEXT NOT NULL REFERENCES categories(id) ON DELETE CASCADE,
  PRIMARY KEY (client_id, category_id)
);

-- ─────────────────────────────────────────
-- LIKES
-- ─────────────────────────────────────────
CREATE TABLE likes (
  id          TEXT PRIMARY KEY,
  client_id   TEXT NOT NULL REFERENCES client_users(id) ON DELETE CASCADE,
  media_id    TEXT REFERENCES media(id) ON DELETE CASCADE,    -- null if folder like
  folder_id   TEXT REFERENCES folders(id) ON DELETE CASCADE,  -- null if media like
  liked_at    TEXT NOT NULL DEFAULT (datetime('now')),
  UNIQUE(client_id, media_id),
  UNIQUE(client_id, folder_id)
);

-- ─────────────────────────────────────────
-- SESSIONS (client analytics)
-- ─────────────────────────────────────────
CREATE TABLE client_sessions (
  id              TEXT PRIMARY KEY,
  client_id       TEXT NOT NULL REFERENCES client_users(id) ON DELETE CASCADE,
  org_id          TEXT NOT NULL,
  project_id      TEXT NOT NULL,
  started_at      TEXT NOT NULL DEFAULT (datetime('now')),
  last_active_at  TEXT NOT NULL DEFAULT (datetime('now')),
  duration_s      INTEGER NOT NULL DEFAULT 0,
  ip              TEXT,
  country         TEXT,
  city            TEXT,
  device          TEXT                                -- mobile | desktop
);

-- ─────────────────────────────────────────
-- UPLOAD SESSIONS (casting director uploads)
-- ─────────────────────────────────────────
CREATE TABLE upload_tokens (
  id          TEXT PRIMARY KEY,             -- the bearer token
  org_id      TEXT NOT NULL,
  project_id  TEXT NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
  folder_id   TEXT REFERENCES folders(id), -- locked to specific folder, or null = admin
  label       TEXT,                         -- "Casting Director Day 1"
  expires_at  TEXT NOT NULL,
  created_by  TEXT NOT NULL REFERENCES admin_users(id),
  created_at  TEXT NOT NULL DEFAULT (datetime('now'))
);

-- ─────────────────────────────────────────
-- INDEXES
-- ─────────────────────────────────────────
CREATE INDEX idx_projects_org ON projects(org_id);
CREATE INDEX idx_categories_project ON categories(project_id);
CREATE INDEX idx_folders_category ON folders(category_id);
CREATE INDEX idx_media_folder ON media(folder_id);
CREATE INDEX idx_media_org ON media(org_id);
CREATE INDEX idx_client_users_project ON client_users(project_id);
CREATE INDEX idx_likes_client ON likes(client_id);
CREATE INDEX idx_sessions_client ON client_sessions(client_id);
CREATE INDEX idx_sessions_org ON client_sessions(org_id);
```

---

## 6. R2 Storage Structure

```
{bucket}/
  {org_id}/
    {project_id}/
      {category_id}/
        {folder_id}/
          original/
            {media_id}.jpg          ← original upload, never deleted
          thumb/
            {media_id}.jpg          ← generated on upload (Cloudflare Images)
```

**CDN URL pattern:**  
`https://media.scoutlane.com/{org_id}/{project_id}/{category_id}/{folder_id}/original/{media_id}.jpg`

**Thumbnail via Cloudflare Images transform:**  
`https://media.scoutlane.com/cdn-cgi/image/width=400,fit=cover/{org_id}/...`

---

## 7. Cloudflare Worker API Routes

All routes return JSON. Auth via `Authorization: Bearer {jwt}` header.

### Auth
```
POST /auth/admin/login        { email, password, org_slug } → { token, admin }
POST /auth/client/login       { username, password, project_id } → { token, client }
POST /auth/refresh            { refreshToken } → { token }
POST /auth/logout
```

### Organizations
```
GET  /org/:slug               → org info (public: name, logo)
PUT  /org/:slug               [admin] → update settings
GET  /org/:slug/stats         [admin] → storage used, project count, client count
```

### Projects
```
GET  /org/:slug/projects      [admin] → list projects
POST /org/:slug/projects      [admin] → create project
GET  /org/:slug/projects/:id  [admin] → project + categories + client count
PUT  /org/:slug/projects/:id  [admin] → update name/logo/status
DEL  /org/:slug/projects/:id  [admin] → delete project + all media (async R2 purge)
```

### Gallery structure
```
GET  /projects/:id/manifest   [admin|client] → { categories: [{ id, name, folders: [{ id, name, mediaCount, thumbUrl }] }] }
GET  /folders/:id/media       [admin|client] → paginated media list
POST /categories              [admin] → create category
PUT  /categories/:id          [admin] → rename / reorder
DEL  /categories/:id          [admin] → delete + purge media
POST /folders                 [admin] → create folder
PUT  /folders/:id             [admin] → rename / reorder
DEL  /folders/:id             [admin] → delete + purge media
DEL  /media/:id               [admin] → delete single item
```

### Uploads
```
POST /upload/presign          [admin|upload_token] → { uploadUrl, mediaId, r2Key }
POST /upload/confirm          [admin|upload_token] → { mediaId } → finalises DB record
POST /upload/token            [admin] → generate time-limited upload token for casting director
```

### Clients
```
GET  /projects/:id/clients    [admin] → list client users
POST /projects/:id/clients    [admin] → { username, password, display_name, categoryIds? }
PUT  /clients/:id             [admin] → update password / access / active status
DEL  /clients/:id             [admin] → deactivate
```

### Likes
```
GET  /clients/:id/likes       [admin|self] → { media: [...], folders: [...] }
POST /likes                   [client] → { mediaId? | folderId? }
DEL  /likes/:id               [client]
GET  /projects/:id/likes      [admin] → all likes grouped by client (for review)
```

### Downloads
```
POST /projects/:id/download   [client] → { jobId } → async ZIP of liked photos
GET  /download/:jobId         → { status, url? }
```

### Analytics
```
POST /analytics/heartbeat     [client] → { sessionId, durationDelta }
GET  /projects/:id/analytics  [admin] → { sessions, topViewers, topPhotos, likesByClient }
```

### Billing (Stripe)
```
POST /billing/checkout        [admin] → { checkoutUrl } (Stripe hosted page)
GET  /billing/portal          [admin] → { portalUrl } (manage subscription)
POST /billing/webhook         [stripe] → handle subscription events → update D1
```

---

## 8. Auth Flow

### Admin JWT
```
1. Admin POSTs /auth/admin/login
2. Worker verifies password (bcrypt), checks org plan is active
3. Worker returns:
   - accessToken (JWT, 1hr, contains: org_id, admin_id, role)
   - refreshToken (opaque, 30 days, stored in D1)
4. Frontend stores tokens in memory (access) + httpOnly cookie (refresh)
5. On 401: frontend calls /auth/refresh automatically
```

### Client Session Token
```
1. Client POSTs /auth/client/login (username + password + project_id)
2. Worker verifies, creates client_sessions row
3. Returns sessionToken (JWT, 7 days, contains: client_id, project_id, org_id)
4. Client stores in localStorage (same as prototype)
5. All client API calls include token; Worker validates and extracts context
```

### Upload Token (casting director)
```
1. Admin generates token via POST /upload/token (sets expiry + folder lock)
2. Token is a short random string stored in upload_tokens table
3. Casting director visits: app.scoutlane.com/{orgSlug}/upload?token={token}
4. Worker validates token on every upload presign request
5. Token can be revoked by admin at any time
```

---

## 9. Pricing Tiers

| Plan | Price | Projects | Clients | Storage | Support |
|---|---|---|---|---|---|
| **Trial** | Free / 14 days | 1 | 5 | 5 GB | – |
| **Starter** | €49/mo | 3 | 20 | 50 GB | Email |
| **Pro** | €149/mo | 15 | 100 | 250 GB | Priority |
| **Agency** | €399/mo | Unlimited | Unlimited | 1 TB | Dedicated |
| **Enterprise** | Custom | Unlimited | Unlimited | Custom | SLA |

Storage overage: €0.02/GB above plan limit (Cloudflare R2 cost + margin).

Stripe products: one product per tier, monthly and annual billing (annual = 2 months free).

---

## 10. URL Structure

```
app.scoutlane.com/                        → marketing / login
app.scoutlane.com/signup                  → org registration + Stripe trial
app.scoutlane.com/{orgSlug}/              → admin dashboard
app.scoutlane.com/{orgSlug}/projects      → project list
app.scoutlane.com/{orgSlug}/projects/{id} → admin photo manager
app.scoutlane.com/{orgSlug}/analytics     → analytics dashboard
app.scoutlane.com/{orgSlug}/settings      → org settings, billing
app.scoutlane.com/{orgSlug}/upload        → casting director upload portal (token-gated)

gallery.scoutlane.com/{orgSlug}/{projectId}/         → client gallery login
gallery.scoutlane.com/{orgSlug}/{projectId}/gallery  → client gallery (post-login)
```

Or: each org gets a custom domain (`gallery.{clientdomain}.com` → CNAME to Cloudflare Pages).

---

## 11. Development Roadmap

### Phase 1 — Core (6 weeks)
**Goal:** One production company can sign up, create a project, upload photos, add clients, and clients can browse and like.

- [ ] D1 schema migration scripts
- [ ] Worker: auth routes (admin + client JWT)
- [ ] Worker: project CRUD
- [ ] Worker: category + folder CRUD
- [ ] Worker: upload presign + confirm (extend existing R2 worker)
- [ ] Worker: manifest endpoint
- [ ] Worker: likes CRUD
- [ ] Frontend: admin dashboard (port from fmlyprod.com prototype)
- [ ] Frontend: client gallery (port from fmlyprod.com prototype)
- [ ] Cloudflare Pages deploy pipeline

### Phase 2 — Multi-tenancy + Billing (3 weeks)
**Goal:** Multiple companies can sign up independently. Billing gates features.

- [ ] Stripe integration (checkout + webhook handler + portal)
- [ ] Plan enforcement (project/client/storage limits)
- [ ] Self-service signup flow
- [ ] Org settings page (logo, name, custom domain)
- [ ] Email (invite admin users, billing receipts)

### Phase 3 — Analytics + Downloads (2 weeks)
- [ ] Client session heartbeat + duration tracking
- [ ] Analytics dashboard (port from fmlyprod.com prototype)
- [ ] Async ZIP download (Cloudflare Queue or Durable Object)
- [ ] Geo + device tracking

### Phase 4 — Casting Upload Portal (1 week)
- [ ] Upload token generation + management
- [ ] Casting director upload UI (port from upload.html prototype)
- [ ] Folder-locked uploads

### Phase 5 — Polish + Scale (ongoing)
- [ ] Custom domain per org (Cloudflare for SaaS)
- [ ] Bulk media operations (delete all, reorder)
- [ ] Client access control (per-category)
- [ ] White-label option (logo + colours per org)
- [ ] API for integrations

---

## 12. Infrastructure Setup Checklist

```
Cloudflare
  ✓ R2 bucket: scoutlane-media (already: fmly-media, rename/new)
  ✓ Worker: scoutlane-api (extend existing fmly-upload worker)
  □ D1 database: scoutlane-db
  □ Custom domain: media.scoutlane.com → R2 public bucket
  □ Custom domain: app.scoutlane.com → Cloudflare Pages
  □ Cloudflare Images (for thumbnail transforms) OR R2 + resizing worker
  □ Cloudflare for SaaS (for custom org domains, Phase 5)

Stripe
  □ Products + prices for each plan
  □ Webhook endpoint → Worker /billing/webhook
  □ Customer portal enabled

Resend / Email
  □ Domain verified: mail.scoutlane.com
  □ Templates: welcome, invite, billing receipt, trial expiry
```

---

## 13. Key Differences from Prototype

| Prototype (fmlyprod.com) | Scout Lane |
|---|---|
| GitHub API as write database | Cloudflare D1 |
| GitHub Pages hosting | Cloudflare Pages |
| Single hardcoded admin | Multi-tenant admin users |
| Admin PAT in localStorage | JWT auth via Worker |
| One manifest.json per project | Media rows in D1 + R2 keys |
| No billing | Stripe subscriptions |
| Cloudflare R2 for media ✓ | Cloudflare R2 for media ✓ |
| Cloudflare Worker for uploads ✓ | Cloudflare Worker (extended) ✓ |
| Picdrop-style UX ✓ | Same UX, carried forward ✓ |

---

## 14. Estimated Monthly Infrastructure Cost (at 100 paying orgs)

| Service | Usage | Cost |
|---|---|---|
| Cloudflare Workers | ~50M requests/mo | $5 |
| Cloudflare D1 | ~500M row reads/mo | $0 (free tier) |
| Cloudflare R2 | 10 TB stored, 1B reads | ~$150 |
| Cloudflare Pages | Unlimited | $0 |
| Stripe | 2.9% + 30¢ per transaction | Variable |
| Resend | ~10K emails/mo | $0 (free tier) |
| **Total infra** | | **~$160/mo** |

At €149/mo average plan × 100 orgs = **€14,900/mo revenue**.  
Infrastructure cost = **~1% of revenue**. Margins are extremely healthy.

---

*Document prepared August 2026. Based on fmlyprod.com prototype — all features validated in production use.*
