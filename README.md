# RSV Lab — Frontend (Cloudflare Pages)

This is the **frontend half** of a two-part deployment:

```
┌─────────────────────────────┐        ┌──────────────────────────────┐
│  Cloudflare Pages (this repo) │        │  A Python host, deployed      │
│  - static HTML/CSS/JS only    │──────▶ │  from GitHub (Render/Railway/ │
│  - password-gated             │  API   │  Fly/etc.)                    │
│  - /            marketing site│ calls  │  - Flask API (rsv-lab-app)    │
│  - /community/  the posts/    │        │  - talks to a database        │
│    marketplace app            │        │  - issues JWTs, no HTML       │
└─────────────────────────────┘        └──────────────────────────────┘
```

Cloudflare Pages cannot run Flask/Python servers — it only serves static
files (plus small JS "Functions" at the edge, used here just for the
password gate). The Flask **backend** from earlier in this conversation
(`rsv-lab-app.zip`) is a separate repo/deploy that has to live on an
actual Python host. This folder is what goes on Cloudflare; that other
one is what goes on the Python host. They talk to each other over
plain HTTPS API calls once both are live.

## What's in here

```
rsv-lab-frontend/
├── index.html              # RSV Lab marketing/portfolio site (the home page)
├── robots.txt
├── community/
│   ├── index.html           # the posts / marketplace / trending app
│   ├── dashboard.html        # the stats dashboard
│   └── css/styles.css
└── functions/
    └── _middleware.js         # password-gates the whole site (Cloudflare Pages Function)
```

`community/*.html` are the same files from `rsv-lab-app`'s
`app/templates/`, adjusted to work as plain static files instead of
Flask/Jinja templates (the `{{ url_for(...) }}` CSS tag became a plain
relative path, `/dashboard` became `dashboard.html`, etc.).

## Step 1 — deploy the backend somewhere (generic guide)

You said "not sure yet" on the host, so here's the common path that
works, with minor naming differences, on Render, Railway, Fly.io,
DigitalOcean App Platform, and most similar GitHub-connected PaaS hosts
(`rsv-lab-app.zip` from earlier already has everything these need):

1. Push the `rsv-lab-app` folder to its own GitHub repo.
2. On the host, create a new app/service and connect that GitHub repo.
   Most of them auto-detect Python and the included `Procfile`
   (`web: gunicorn run:app`); if a host asks you to fill these in
   manually instead:
   - **Build command:** `pip install -r requirements.txt`
   - **Start command:** `gunicorn run:app`
3. Set environment variables (see `rsv-lab-app/.env.example`):
   - `SECRET_KEY`, `JWT_SECRET_KEY` — long random strings
   - `DATABASE_URL` — if the host gives you a managed Postgres
     add-on, use its connection string here. **Don't rely on SQLite
     in production** — most of these hosts wipe the local disk
     (and your database with it) on every redeploy.
   - `CORS_ORIGINS` — the Cloudflare Pages URL(s) that will call this
     API, comma-separated, e.g.
     `https://rsv-lab.pages.dev,https://your-custom-domain.com`
   - `SESSION_COOKIE_SECURE=True`
4. Deploy. Copy the live URL the host gives you
   (e.g. `https://rsv-lab-app.onrender.com`).

   One exception: **PythonAnywhere** doesn't use a `Procfile`/`gunicorn`
   — it wants a WSGI config file instead. If you land there, point its
   WSGI file's `application` import at `run:app` per their docs rather
   than following the build/start-command steps above.

## Step 2 — point this frontend at that backend

Edit **two lines**, one in each file:

- `community/index.html`
- `community/dashboard.html`

Both have this near the top of the `<script>` block:

```js
const API_BASE = 'https://YOUR-BACKEND-URL.example.com/api';
```

Replace `https://YOUR-BACKEND-URL.example.com` with the real URL from
Step 1 (keep the trailing `/api`).

## Step 3 — deploy this folder to Cloudflare Pages

**Option A — connect a GitHub repo (auto-deploys on every push)**

1. Push this folder to its own GitHub repo (separate from the backend's).
2. Cloudflare dashboard → **Workers & Pages** → **Create** → **Pages**
   → **Connect to Git** → pick the repo.
3. Framework preset: **None**. Build command: empty. Output directory: `/`.
4. Deploy — you'll get a `https://<project-name>.pages.dev` URL.

**Option B — direct upload, no GitHub**

```bash
npm install -g wrangler
wrangler login
wrangler pages deploy . --project-name=rsv-lab
```

## Step 4 — password-protect it

`functions/_middleware.js` gates every page on this site behind one
shared username/password (browser's native login prompt — no extra
UI to build). It's already in the repo; you just need to set the
password as a Cloudflare **environment variable** (never commit a
real password to git):

1. Cloudflare dashboard → your Pages project → **Settings** →
   **Environment variables** → add:
   - `SITE_PASSWORD` = your shared password (tick **Encrypt**)
   - `SITE_USER` = a username (optional — defaults to `rsvlab`)
2. Redeploy (env var changes need a fresh deploy to take effect —
   trigger one from the dashboard, or push an empty commit if using
   Git integration).
3. Visiting the site now prompts for that username/password. Share
   those credentials with whoever should have access.

This locks the **frontend pages**. It does *not* lock the backend API
itself — that's a separate, directly-reachable URL. The API already
requires a valid JWT for anything that creates/edits data (posts,
items, comments); its read endpoints (listing posts/items, stats) are
public by design, same as before. If you also want the *entire* API
private, say so and I'll add a shared-secret check there too.

## What's new: Search

`community/index.html` now has a **Search** tab. It calls
`GET {API_BASE}/search?q=...` on the backend and renders three grouped
result sections — People, Posts, Marketplace items — using the same
card-rendering functions the rest of the app already uses, so results
look identical to browsing those sections directly. No auth required
to search; it's a public GET like the posts/items listings.

While wiring this up, a real privacy gap got fixed on the backend
side: items marked `visibility: "private"` were previously still
visible to everyone in `GET /api/items` and on their own detail page
(only the seller's contact link was hidden). Both endpoints now
actually exclude private items from anyone but their owner. If you
had any private items created before this update, they're now
properly private — no action needed, this is a correctness fix, not
a breaking change to how you use the API.

## Step 5 — sanity check

Once both are deployed and Step 2's URLs are filled in:

1. Open your `*.pages.dev` URL → enter the password.
2. Go to **Community** → try creating an account → create a post.
   If it fails with a CORS error in the browser console, double-check
   `CORS_ORIGINS` on the backend includes your exact Pages URL
   (protocol + domain, no trailing slash).

## Updating the marketing site's URLs later

Once you have a permanent domain (custom domain on Cloudflare Pages,
or you keep the `*.pages.dev` one), update these in `index.html` to
match it (currently placeholders left over from the original draft):

- `<link rel="canonical" href="...">`
- `<meta property="og:url" content="...">` / `og:image`
- `CONFIG.site.url` in the inline `<script>`
- The JSON-LD `"url"` field near the top of `<head>`
