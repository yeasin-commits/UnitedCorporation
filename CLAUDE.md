# United Corporation — project context for Claude Code

> Bangladesh's largest equipment rental network. React (Vite + Tailwind) SPA + PHP/MySQL backend.
> Designed to be hosted on a BDIX shared-hosting cPanel/Apache/PHP/MySQL stack.
> The folder is named `ironyard` for legacy reasons — the original demo brand was IRONYARD; the live brand is **United Corporation**.

---

## Stack

- **Frontend:** React 19, Vite 8, Tailwind v4, lucide-react. SPA rendered into `<div id="root">`.
- **Backend:** PHP 7.4+ (PDO), MySQL on production / SQLite for local dev. Hand-rolled HS256 JWT (no Composer, no deps).
- **Same-origin in production:** the SPA is served from `public_html/` and the API from `public_html/api/` — no CORS issues.
- **Dev proxy:** `vite.config.js` proxies `/api/*` from Vite (`:5173`) to PHP (`:8000`).

---

## Folder layout

```
ironyard/
├── api/                          ← PHP backend (deploys to public_html/api/)
│   ├── index.php                 ← front controller / router
│   ├── routes.php                ← all endpoint handlers (one function each)
│   ├── install.php               ← one-time setup: tables + seed + admin user. DELETE after install.
│   ├── config.example.php        ← template — copy to config.php
│   ├── config.php                ← real credentials (gitignored). Defaults to SQLite for dev.
│   ├── core/
│   │   ├── bootstrap.php         ← loads config, connects DB, sets CORS + JSON error handler
│   │   ├── db.php                ← PDO connect (mysql/sqlite) + db_schema()
│   │   ├── auth.php              ← jwt_issue / jwt_verify / require_admin / get_bearer_token
│   │   └── response.php          ← json_response / json_error / read_json_body / send_cors
│   ├── data/                     ← SQLite file lives here (gitignored, .htaccess denies access)
│   ├── uploads/                  ← user-uploaded images (.htaccess blocks PHP execution)
│   └── .htaccess                 ← rewrites /api/* → index.php; blocks config.php
│
├── src/                          ← React frontend
│   ├── main.jsx                  ← React entry
│   ├── App.jsx                   ← public site (Header, Hero, Home, Fleet, Services, About, Contact, BookingModal, Footer)
│   ├── Admin.jsx                 ← admin panel: Login + tabbed UI (Content/Vehicles/Services/Bookings/Messages/Media/Account)
│   ├── content.jsx               ← ContentProvider context, useContent(), Icon name→component map, DEFAULTS
│   ├── api.js                    ← fetch wrapper + bearer-token storage + every API call
│   ├── index.css, App.css
│   └── assets/
│
├── DEPLOYMENT.md                 ← BDIX shared-hosting walkthrough (cPanel, MySQL, .htaccess)
├── README.md                     ← short top-level intro
├── vite.config.js                ← React + Tailwind plugins + /api proxy
├── package.json                  ← name still "ironyard", scripts: dev / build / preview / lint
└── .gitignore                    ← excludes api/config.php, api/data/, api/uploads/* (keeps .htaccess)
```

---

## Database schema (see `api/core/db.php → db_schema()`)

| Table       | Purpose                                                                         |
|-------------|---------------------------------------------------------------------------------|
| `users`     | Admin accounts (currently 1). `password_hash` via `password_hash()` (bcrypt).   |
| `settings`  | KV store for ALL editable site copy. `skey` (PK) → `svalue` (string OR JSON).   |
| `vehicles`  | Equipment catalog. `specs` is a JSON column (object of key→value strings).      |
| `services`  | The 6 service tiles (`icon`, `title`, `description`, `sort_order`).             |
| `bookings`  | Quote requests submitted from the public site. `reference` is `BK<base36>`.    |
| `messages`  | Contact-form submissions. `is_read` toggle.                                     |
| `media`     | Uploaded image registry — populated by `/api/upload`.                           |

Schema is driver-agnostic — works on both MySQL and SQLite via PDO. JSON is stored as TEXT and en/decoded in PHP.

---

## API endpoints (see `api/routes.php`)

Routing is a flat array in `api/index.php`. Auth is `Authorization: Bearer <jwt>`. JWT is HS256, 8-hour TTL by default.

**Public:**
- `GET  /api/site` — bootstrap: returns `{ settings, vehicles (active only), services }`
- `POST /api/bookings` — create booking from the BookingModal
- `POST /api/messages` — create contact message

**Auth:**
- `POST /api/auth/login` — `{ username, password }` → `{ token, username, expires_in }`
- `POST /api/auth/logout` — no-op (JWT is stateless)
- `GET  /api/auth/me` — verify token
- `POST /api/auth/change-password` — `{ old_password, new_password }`

**Admin (all require Bearer token):**
- `GET/PUT /api/settings` — bulk read or upsert (PUT body is `{ "key": value, ... }`)
- `GET/POST /api/vehicles`, `PUT/DELETE /api/vehicles/{id}`
- `GET/POST /api/services`, `PUT/DELETE /api/services/{id}`
- `GET /api/bookings`, `PUT/DELETE /api/bookings/{id}` (PUT changes status/fields)
- `GET /api/messages`, `PUT/DELETE /api/messages/{id}`
- `POST /api/upload` — multipart, field name `file`. Returns `{ id, url, filename, size, mime }`
- `GET /api/media`, `DELETE /api/media/{id}`

---

## Frontend architecture

- **`ContentProvider`** in `src/content.jsx` calls `GET /api/site` on mount and exposes `{ settings, vehicles, services, loading, error, reload }` via `useContent()`.
- **Settings keys** are dotted (`hero.title_line1`, `contact.phone`, etc.). The full list with friendly labels lives in `CONTENT_GROUPS` inside `src/Admin.jsx`. To add an editable field, add the key to `install.php`'s seed AND to `CONTENT_GROUPS`.
- **DEFAULTS** in `content.jsx` is the offline fallback shown if `/api/site` fails. Keep it in sync with the install-seeded keys so the page never looks broken.
- **Icons** are stored as strings in the DB (`"Wrench"`, `"Truck"`, etc.) and resolved via `<Icon name={s.icon} />` from `content.jsx`. Available names are in `ICON_NAMES`.
- **Token storage**: `localStorage.ironyard_token` (legacy key — leaving it for backwards compat with anyone who already logged in). Fetched/cleared by `getToken/setToken/clearToken` in `api.js`.
- **Routing**: hash-based (`#admin`) — no React Router. State lives in `App.jsx`'s `page` state.

---

## What's editable from the admin

The Content tab has 10 collapsible groups (see `CONTENT_GROUPS` in `Admin.jsx`):
brand, contact, hero, home_sections, cta, fleet, services_page, about_page, contact_page, booking. Each field declares a kind: `text` | `textarea` | `image` | `jsonlist`. The `jsonlist` editor renders a friendly per-row UI for arrays of strings or arrays of objects (sniffs the first row's shape).

Image uploads accept JPEG/PNG/WebP/GIF/SVG up to 8 MB (configurable in `api/config.php`). Files land in `api/uploads/` and are served at `/api/uploads/<filename>`.

---

## Local dev

```bash
# terminal 1 — PHP backend (uses SQLite, no MySQL needed)
php -S localhost:8000 -t api

# terminal 2 — one-time DB seed
php api/install.php

# terminal 3 — Vite dev server (proxies /api → :8000)
npm install
npm run dev
```

Open <http://localhost:5173>. Footer → **Admin Portal** → log in with `admin` / `ironyard2025`. Change password on the **Account** tab.

To switch to MySQL locally, edit `api/config.php` (`'driver' => 'mysql'`, set host/user/pass), then re-run `php api/install.php`.

---

## Production deploy (BDIX shared hosting)

Detailed walkthrough in **`DEPLOYMENT.md`**. Summary:

1. `npm run build` → upload `dist/*` to `public_html/`, upload `api/` to `public_html/api/`
2. Create MySQL DB in cPanel; copy `api/config.example.php` → `api/config.php`; fill credentials and a strong `jwt_secret`
3. Visit `https://yoursite.com/api/install.php` once — then **delete the file**
4. Add SPA fallback `.htaccess` in `public_html/` (sample in `DEPLOYMENT.md` step 6)
5. Log in via the Admin Portal link in the footer; change password immediately

---

## Common task recipes

**Add a new editable text field to the public site**
1. Add the key to `$default_settings` in `api/install.php`
2. Add a matching default in `DEFAULTS.settings` in `src/content.jsx`
3. Add the key to the appropriate group in `CONTENT_GROUPS` in `src/Admin.jsx`
4. Render it in the relevant component in `src/App.jsx` via `useContent()` or `useSetting('your.key')`
5. Existing users need to re-run `install.php` (it skips existing keys, so safe) — or set the value in admin

**Add a new API endpoint**
1. Add a route line in `api/index.php` (`['METHOD', '#regex#', 'route_handler_name']`)
2. Implement `route_handler_name(PDO $pdo, array $cfg, array $args)` in `api/routes.php`
3. Use `require_admin($cfg['jwt_secret'])` for protected endpoints
4. Return via `json_response($data, $status)` or `json_error($msg, $status)`
5. Add a thin wrapper in `src/api.js`

**Add a column to the `vehicles` table**
1. Edit `db_schema()` in `api/core/db.php` (only used for fresh installs)
2. For existing DBs: write a one-off ALTER TABLE migration (no migration framework — keep it manual)
3. Allow it through the field whitelist in `route_vehicles_create` and `route_vehicles_update`
4. Render/edit it in `VehicleForm` in `src/Admin.jsx`
5. Use it on the public site in `VehicleCard` in `src/App.jsx`

**Reset all content to fresh seed values**
- Drop the `settings` table (or delete `api/data/ironyard.sqlite` for SQLite) and re-run `php api/install.php`. The script is idempotent on tables and only fills missing keys, so it's safe.

---

## Gotchas

- **Don't commit `api/config.php`** — already in `.gitignore`. Contains DB password and `jwt_secret`.
- **Delete `api/install.php` after first deploy** — it can re-create an admin user if the table is empty.
- **`Authorization` header on shared hosts** — Apache sometimes strips it. The `api/.htaccess` handles this with the `RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]` block. Don't remove it.
- **`uploads/.htaccess`** — explicitly blocks `.php`, `.phtml`, `.phar` execution inside the uploads dir. Required if you allow user uploads.
- **Vite 8 build can be slow on first run** because of the rolldown native binary download. On Linux ARM/musl you may need `npm rebuild`.
- **`is_active = 0`** vehicles are hidden from `/api/site` (public) but visible in the admin vehicles list — by design.
- **Booking reference** format is `BK<base36(time*1000)>`. Unique-ish; if you ever hit a collision, retry. Not crypto-secure — fine for human-readable refs only.
- **Folder rename** — the project folder is still called `ironyard` and `package.json` name is still `ironyard`. Renaming is cosmetic but would touch the workspace mount path.

---

## Open / future work

- No automated tests yet. Smoke-test endpoints with `curl` after backend changes.
- No email notification on new booking/message — easy add: hook `mail()` into `route_booking_create` / `route_message_create`.
- No rate limiting on `/api/bookings` or `/api/messages` — add IP throttling if spam becomes a problem.
- Multi-admin support: schema already has `users` table with unique username — just needs UI in the Account tab to add users.
- Folder rename `ironyard` → `united-corporation` is purely cosmetic; update `package.json` `"name"` and the workspace mount if you do it.
