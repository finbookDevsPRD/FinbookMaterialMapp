# `ec2prod` Branch — Production Deployment Plan

**Goal:** Move MaterialMApp from "runs on one local PC" (`main`) to a public production setup:

| Piece | `main` (today) | `ec2prod` (target) |
|---|---|---|
| Frontend | `npx expo start --web` on PC, port 8081 | Static Expo web build on **Vercel** at `https://materialapp.arthvik.com` |
| Backend | `uvicorn --reload` on PC, port 8000 | FastAPI on **Oracle Cloud Free VPS** (or AWS EC2) at `https://api.materialapp.arthvik.com` |
| Database | MongoDB on `localhost:27017` | **MongoDB Atlas M0 (free)** or MongoDB on the same VPS |
| Phone access | `bridge_proxy.js` on port 8005 over Wi‑Fi | Not needed: phones open the public HTTPS URL |
| Start scripts | `run_*.bat`, `allow_firewall.bat` | systemd service + Caddy/Nginx on Linux |

> **Domain note:** a hostname can't contain a space, so `material app.arthvik.com` has to be
> `materialapp.arthvik.com` (or `material-app.arthvik.com`). This plan uses `materialapp.arthvik.com`
> for the frontend and `api.materialapp.arthvik.com` for the backend.

---

## 1. Why the backend needs its own HTTPS domain

Vercel serves the site over **HTTPS**. Browsers block calls from an HTTPS page to a plain
`http://<ip>:8000` API ("mixed content"). So the backend **must** sit behind HTTPS with a real
hostname, i.e. `api.materialapp.arthvik.com` → VPS public IP, with a free Let's Encrypt
certificate (Caddy does this automatically).

```
Browser / Phone
   │  https://materialapp.arthvik.com        (Vercel, static files)
   │  https://api.materialapp.arthvik.com    (API calls)
   ▼
Oracle/EC2 VPS ── Caddy :443 (TLS) ──► uvicorn/gunicorn 127.0.0.1:8000 (FastAPI)
                                              │
                                              ▼
                                   MongoDB Atlas  (or mongod on 127.0.0.1:27017)
```

---

## 2. Security fixes that are REQUIRED before going public

On `main` these were fine because only the local network could reach the API. On the
internet they are serious:

| # | Problem found in code | File | Fix in `ec2prod` |
|---|---|---|---|
| 2.1 | **Masters and Entries endpoints have no auth.** Only `/api/auth/me` and `/logout` use `get_current_user`. Anyone with the URL can read, add, delete and bulk‑delete all data. | `backend/routes/master_routes.py`, `backend/routes/entry_routes.py` | Add `dependencies=[Depends(get_current_user)]` to both `APIRouter(...)` definitions (keep `/template` public or protect it too). |
| 2.2 | **`/api/auth/register` is open and creates `admin` users by default.** Anyone can make themselves an admin. | `backend/routes/auth_routes.py` | Disable public register (env flag `ALLOW_REGISTRATION=false`), or require an existing admin token; default role → `user`. |
| 2.3 | **Hardcoded JWT secret default** `materialmapp_super_secure_secret_key_2026_xyz`. | `backend/config.py` | No default in prod: fail startup if `JWT_SECRET` is missing/weak. Generate with `openssl rand -hex 32`. |
| 2.4 | **Default admin `admin@materialmapp.com / admin123`** created by `seed.py` and printed in README. | `backend/seed.py`, `README.md` | Read admin email/password from env (`ADMIN_EMAIL`, `ADMIN_PASSWORD`); change password right after first login. |
| 2.5 | **CORS is `*` with `allow_credentials=True`.** | `backend/main.py`, `.env` | `CORS_ORIGINS=https://materialapp.arthvik.com` (add Vercel preview URL if needed). |
| 2.6 | **Swagger `/docs` is public.** | `backend/main.py` | Turn off in prod: `docs_url=None, redoc_url=None` when `ENV=production`. |
| 2.7 | **No rate limit on `/login`** (password guessing). | `backend/main.py` | Add `slowapi` limit (e.g. 5/min per IP) or Caddy rate limit. |
| 2.8 | **Excel upload size unbounded.** | `entry_routes.py` / Caddy | Limit request body (e.g. 10 MB) in Caddy/Nginx. |

---

## 3. Backend changes (`ManageAPP/backend`)

1. **`config.py`**
   - Add `ENV: str = "development"` (`production` on the server).
   - Add `ALLOW_REGISTRATION: bool = False`, `ADMIN_EMAIL`, `ADMIN_PASSWORD`.
   - Validate `JWT_SECRET` when `ENV == "production"`.
2. **`main.py`**
   - Disable `/docs`/`/redoc` in production.
   - Strict CORS list from env (already supported via `CORS_ORIGINS`, just set it).
   - Remove `reload=True` from the `__main__` block (use the prod start command instead).
   - Add `ProxyHeadersMiddleware` / run uvicorn with `--proxy-headers --forwarded-allow-ips=127.0.0.1` since it sits behind Caddy.
3. **Routes** — auth fixes from section 2.
4. **`requirements.txt`**
   - Split into `requirements.txt` (runtime) and `requirements-dev.txt` (`pytest`, `pytest-asyncio`, `httpx`).
   - Add `gunicorn` (process manager) and optionally `slowapi`.
   - Pin versions (`==`) so the server installs the same thing every time.
   - Python version: README says 3.14 — use whatever the VPS image ships (3.11/3.12 on Ubuntu 22.04/24.04) and confirm tests pass on it.
5. **`.env.example`** — production template:
   ```env
   ENV=production
   MONGODB_URL=mongodb+srv://<user>:<pass>@<cluster>.mongodb.net/?retryWrites=true&w=majority
   DB_NAME=materialmapp
   JWT_SECRET=<openssl rand -hex 32>
   JWT_ALGORITHM=HS256
   JWT_EXPIRATION_HOURS=24
   HOST=127.0.0.1
   PORT=8000
   CORS_ORIGINS=https://materialapp.arthvik.com
   ALLOW_REGISTRATION=false
   ADMIN_EMAIL=you@arthvik.com
   ADMIN_PASSWORD=<strong password>
   ```
6. **Scripts that hardcode `mongodb://localhost:27017`** — `seed_type_modes.py`, `clear_data.py`; and `verify_system.py` hardcodes `http://localhost:8000`. Make them read `MONGODB_URL` / `API_URL` from env. Consider **not** shipping `clear_data.py` to the server (it wipes data).
7. **New deploy files** (new folder `ManageAPP/deploy/`):
   - `materialmapp.service` — systemd unit:
     ```ini
     [Unit]
     Description=MaterialMApp FastAPI
     After=network.target

     [Service]
     User=materialmapp
     WorkingDirectory=/opt/materialmapp/ManageAPP/backend
     EnvironmentFile=/opt/materialmapp/ManageAPP/backend/.env
     ExecStart=/opt/materialmapp/venv/bin/gunicorn main:app -k uvicorn.workers.UvicornWorker -w 2 -b 127.0.0.1:8000 --forwarded-allow-ips=127.0.0.1
     Restart=always

     [Install]
     WantedBy=multi-user.target
     ```
   - `Caddyfile`:
     ```
     api.materialapp.arthvik.com {
         request_body { max_size 10MB }
         reverse_proxy 127.0.0.1:8000
     }
     ```
   - `setup_server.sh` — one-time install script (packages, user, venv, service, Caddy).
   - *(Optional alternative)* `Dockerfile` + `docker-compose.yml` (api + caddy [+ mongo]) if you prefer Docker.

---

## 4. Frontend changes (`ManageAPP/frontend`)

The good news: `src/api/client.js` already gives priority to `EXPO_PUBLIC_API_URL`, so the core
change is small.

1. **`src/api/client.js`**
   - Production: `EXPO_PUBLIC_API_URL=https://api.materialapp.arthvik.com` (set in Vercel, baked in at build time).
   - Remove/guard the LAN logic for web: today, any non‑localhost hostname falls back to
     `http://<host>:8005` (the bridge proxy). On Vercel that would become
     `http://materialapp.arthvik.com:8005` → broken. If `EXPO_PUBLIC_API_URL` is not set in a
     production build, fail loudly instead of guessing.
   - Remove the hardcoded `192.168.1.15` default.
2. **`bridge_proxy.js`** — local‑only helper; not used in prod. Keep for dev or delete in this branch.
3. **`package.json`** — add a build script:
   ```json
   "build:web": "expo export --platform web"
   ```
   Output goes to `dist/` (already in `.gitignore`).
4. **`app.json`**
   - Real app name/slug (`"name": "MaterialMApp"`, not `"frontend"`).
   - `"web": { "favicon": "./assets/favicon.png", "bundler": "metro", "output": "single", "name": "MaterialMApp", "themeColor": "#134e4a" }`.
5. **`vercel.json`** (new, in `ManageAPP/frontend`):
   ```json
   {
     "buildCommand": "npx expo export --platform web",
     "outputDirectory": "dist",
     "installCommand": "npm ci",
     "rewrites": [{ "source": "/(.*)", "destination": "/index.html" }],
     "headers": [
       {
         "source": "/(.*)",
         "headers": [
           { "key": "X-Content-Type-Options", "value": "nosniff" },
           { "key": "X-Frame-Options", "value": "DENY" },
           { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" }
         ]
       }
     ]
   }
   ```
6. **`.env.example`** — replace LAN host/port/proxy vars with just `EXPO_PUBLIC_API_URL`.
7. **Excel template download** (`ExcelImportModal.js` → `getTemplateDownloadUrl()`): opens the
   URL directly with no `Authorization` header. If `/template` gets protected (section 2.1),
   switch it to `fetch` with the token and download the blob.
8. **Verify the build locally first:** `npx expo export -p web && npx serve dist`, then click
   through login, dashboard, entries, masters and Excel import. (There's no `babel.config.js`
   for NativeWind; confirm styles still render the same in the exported build.)

---

## 5. Database

**Recommended: MongoDB Atlas M0 (free, 512 MB, managed backups/HA).**
- Create a cluster in an Indian region (Mumbai, `ap-south-1`) close to the VPS.
- Network Access: allow only the VPS public IP.
- Create a DB user with a strong password and put the SRV string into `MONGODB_URL`.
- Migrate existing local data: `mongodump --uri mongodb://localhost:27017/materialmapp` on the PC →
  `mongorestore --uri "<atlas uri>"`.
- Note: the challan counter (`counters.challan_seq`) and unique indexes come along with the dump;
  `connect_to_mongo()` will also recreate indexes on startup.

**Alternative: MongoDB on the same VPS** (Oracle ARM has plenty of RAM). Then bind to
`127.0.0.1` only, enable auth (`security.authorization: enabled`), never open port 27017 in the
firewall, and set up a nightly `mongodump` cron to Object Storage/S3.

---

## 6. Server: Oracle Cloud Free vs AWS EC2

| | Oracle Cloud Always Free | AWS EC2 |
|---|---|---|
| Free compute | Ampere A1 ARM, up to 4 OCPU / 24 GB RAM, **no expiry** | Free tier is time‑limited (credits/12 months depending on account age), small instance (≈1 GB RAM) |
| Gotchas | ARM instances are sometimes "out of capacity" in a region; must open ports in **both** the VCN Security List **and** the instance's iptables | Simple networking; pay after free tier ends |
| Recommendation | ✅ **Use this** (pick Mumbai or Hyderabad region) | Fine if you already use AWS |

**Server setup steps (Ubuntu 22.04/24.04):**
1. Create instance, attach a **reserved/Elastic public IP** (so DNS doesn't break on restart).
2. Firewall: open only **22, 80, 443**. (Oracle: VCN Security List + `iptables`/`netfilter-persistent`; EC2: Security Group.) Do **not** open 8000 or 27017.
3. `sudo apt install python3-venv python3-pip git caddy` (Caddy from its official apt repo).
4. Create user `materialmapp`, clone repo to `/opt/materialmapp`, `git checkout ec2prod`.
5. `python3 -m venv /opt/materialmapp/venv && pip install -r ManageAPP/backend/requirements.txt`.
6. Create `ManageAPP/backend/.env` from the template (chmod 600).
7. Run seed once with `ADMIN_EMAIL`/`ADMIN_PASSWORD` env vars.
8. Install `deploy/materialmapp.service` → `systemctl enable --now materialmapp`.
9. Install `deploy/Caddyfile` → `systemctl reload caddy` (gets the TLS cert automatically once DNS points here).
10. Check: `curl https://api.materialapp.arthvik.com/health` → `{"status":"healthy"}`.
11. Basic hardening: SSH keys only, disable password login, `unattended-upgrades`, `fail2ban`.

---

## 7. DNS (at the `arthvik.com` registrar / DNS provider)

| Record | Name | Value |
|---|---|---|
| CNAME | `materialapp` | `cname.vercel-dns.com` (Vercel shows the exact value when you add the domain) |
| A | `api.materialapp` | VPS public IP |

If DNS is on Cloudflare: keep the `api.materialapp` record **DNS‑only (grey cloud)** at first so
Caddy can get its certificate; you can turn on the proxy later with SSL mode "Full (strict)".

---

## 8. Vercel setup

1. Import the GitHub repo `finbookdevsprd/finbookmaterialmapp` into Vercel.
2. **Root Directory:** `ManageAPP/frontend`. **Production Branch:** `ec2prod`.
3. Framework preset: "Other" (settings come from `vercel.json`).
4. Environment variable: `EXPO_PUBLIC_API_URL = https://api.materialapp.arthvik.com` (Production).
5. Settings → Domains → add `materialapp.arthvik.com`.
6. Add the Vercel URL(s) you'll actually use to backend `CORS_ORIGINS`.

---

## 9. CI/CD (optional but recommended)

- **Frontend:** automatic — Vercel redeploys on every push to `ec2prod`.
- **Backend:** GitHub Action `.github/workflows/deploy-backend.yml` on push to `ec2prod`
  (paths `ManageAPP/backend/**`): run `pytest`, then SSH to the VPS →
  `git pull && pip install -r requirements.txt && systemctl restart materialmapp`.
  Secrets: `SSH_HOST`, `SSH_USER`, `SSH_KEY`.

---

## 10. Files touched in `ec2prod` — checklist

**Modify**
- [ ] `backend/config.py` — ENV, ALLOW_REGISTRATION, admin env, JWT secret validation
- [ ] `backend/main.py` — docs off in prod, no reload, proxy headers, rate limit
- [ ] `backend/routes/master_routes.py` — require auth
- [ ] `backend/routes/entry_routes.py` — require auth
- [ ] `backend/routes/auth_routes.py` — lock down register, default role
- [ ] `backend/seed.py` — admin from env
- [ ] `backend/requirements.txt` (+ new `requirements-dev.txt`) — pin, add gunicorn
- [ ] `backend/.env.example` — production template
- [ ] `backend/tests/*` — send auth token in tests now that routes are protected
- [ ] `seed_type_modes.py`, `clear_data.py`, `verify_system.py` — read URLs from env
- [ ] `frontend/src/api/client.js` — prod URL only, remove LAN fallback/hardcoded IP
- [ ] `frontend/src/components/ExcelImportModal.js` — authenticated template download (if protected)
- [ ] `frontend/app.json` — name/slug/web config
- [ ] `frontend/package.json` — `build:web` script
- [ ] `frontend/.env.example` — only `EXPO_PUBLIC_API_URL`
- [ ] `README.md` — production deploy instructions, remove default credentials

**Add**
- [ ] `frontend/vercel.json`
- [ ] `deploy/materialmapp.service`
- [ ] `deploy/Caddyfile`
- [ ] `deploy/setup_server.sh`
- [ ] `.github/workflows/deploy-backend.yml` (optional)
- [ ] `Dockerfile` / `docker-compose.yml` (optional alternative)

**Remove / not used in prod**
- [ ] `run_all.bat`, `run_backend.bat`, `run_frontend.bat`, `allow_firewall.bat` (Windows local only)
- [ ] `frontend/bridge_proxy.js` (LAN only)

---

## 11. Rollout order

1. Make the code changes above on `ec2prod`; run backend tests and a local `expo export` build.
2. Create Atlas cluster (or local Mongo on VPS) and migrate data.
3. Provision VPS, deploy backend, add `api.materialapp` DNS record, confirm `/health` over HTTPS.
4. Connect Vercel to `ec2prod`, set `EXPO_PUBLIC_API_URL`, add `materialapp` domain.
5. Smoke test on desktop and phone: login, create entry (challan number increments), masters CRUD, Excel import, delete.
6. Change the admin password, confirm `/docs` and `/api/auth/register` are not reachable publicly.
7. Set up backups (Atlas snapshots or nightly `mongodump`).
