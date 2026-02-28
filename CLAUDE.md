# CLAUDE.md — MonitoringCRM Codebase Guide

## Project Overview

**Подсистема контроля и надзора** (Monitoring Control and Supervision Subsystem), short name **MonitoringCRM**, is a PHP + vanilla JavaScript web application delivered as a Progressive Web App (PWA). It is hosted at `monitoring.msr.mosreg.ru`.

The system supports regulatory monitoring workflows: task assignment, registries, API management, and real-time notifications via WebSocket and Web Push.

---

## Technology Stack

| Layer | Technology |
|---|---|
| Backend | PHP (namespaced OOP, `Core\*`) |
| Frontend | JavaScript + jQuery (no SPA framework) |
| UI Libraries | TinyMCE (rich text), Flatpickr (dates), Bootstrap (modals), jQuery Toast (notifications) |
| Icons | Material Design Icons (CDN) |
| PWA | Service Worker v1.6.1 + Web App Manifest |
| WebSockets | Ratchet v0.4.4 PHP library (ports 3010 / 3011 TLS) |
| Web Push | VAPID-based push notifications |
| Database | PostgreSQL (via custom `Core\Db` abstraction) |
| Dependency mgmt | Composer (PHP) |
| Config/secrets | `vlucas/phpdotenv` — `.env` file at document root |

---

## Directory Structure

```
/
├── index.php              # Main entry point — handles ALL requests (pages + AJAX)
├── manifest.json          # PWA manifest
├── service-worker.js      # PWA service worker
├── checkappmodile.svg     # App logo/icon asset
├── modules.zip            # Archive of modules (not active code)
│
├── core/                  # PHP backend core
│   └── composer.lock      # Composer lock file (vendor/ lives at /var/www/html/core/vendor/)
│
├── js/
│   ├── core/              # Core JavaScript
│   │   ├── app.js         # Global `el_app` object — all frontend methods
│   │   ├── tools.js       # Utility/helper functions
│   │   ├── autocomplete.js
│   │   └── suggest.js
│   └── assets/            # Standalone asset scripts
│       ├── agreement_list.js
│       ├── cades_sign.js          # CryptoPro CAdES signing
│       ├── cadesplugin_api.js     # CryptoPro browser plugin API
│       ├── certificate.js
│       ├── nmcades_plugin_api.js
│       └── quarter_select.js
│
├── modules/               # Feature modules (see Module Structure below)
│   ├── api/               # API management module (id=20, parent=settings)
│   │   └── props/         # Registry/field management sub-module (id=5)
│   └── assigned/          # Task assignment module (id=15)
│
├── web-push/              # Web Push notification support
│   ├── config.php         # Loads VAPID keys from .env
│   ├── public-key.php     # Returns VAPID public key to browser
│   └── push.php           # Sends push notifications
│
├── websocket/
│   └── index.php          # WebSocket client-side connection endpoint
│
└── ws/                    # WebSocket server
    ├── server.php         # Ratchet BroadcastingServer (port 3010 / TLS 3011)
    ├── index.php          # WS client entry
    └── send_message.php   # Broadcast message sender
```

---

## Module Structure

Every feature module lives under `modules/{module-name}/` and follows this **mandatory** convention:

```
modules/{name}/
├── module.json          # Module metadata (required)
├── pages/
│   ├── index.php        # Full page render (server-side, non-AJAX)
│   ├── index_ajax.php   # AJAX page content (mode=mainpage)
│   └── index_id_ajax.php
├── dialogs/             # Popup/dialog HTML fragments (mode=popup)
│   └── *.php
├── ajaxHandlers/        # AJAX action handlers (default mode)
│   └── *.php
└── js/                  # Module-specific JavaScript (optional)
```

### `module.json` Schema

```json
{
  "id": 15,
  "name": "Задания",
  "description": "Human-readable description",
  "minimumCoreVersion": "0.1",
  "title": "Display title",
  "version": "0.1",
  "author": "Author name",
  "path": "assigned",          // must match directory name
  "icon": "list_alt",          // Material Design icon name
  "number": 3,                 // navigation order
  "parentItem": "settings"     // optional: parent nav item
}
```

---

## Request Routing (`index.php`)

All HTTP traffic goes through `index.php`. The routing logic:

1. **Logout**: `GET ?logout` — destroys session and redirects to `/`
2. **AJAX requests** (`POST ajax=1`):
   - Requires authentication for all actions except `login` / `mobileLogin`
   - Dispatched by `mode` parameter:
     - `mode=popup` → `modules/{module}/dialogs/{url}.php` (or `core/ajaxHandlers/{url}.php`)
     - `mode=mainpage` → `modules/{url}/pages/{page}_ajax.php`
     - *(default)* → `modules/{path}/ajaxHandlers/{action}.php` (or `core/ajaxHandlers/{action}.php`)
3. **Page requests** (non-AJAX):
   - Generates CSRF token (cookie `CSRF-TOKEN` + session `csrf-token`)
   - Unauthenticated: renders `tmpl/page/login.php`, saves `last_path` cookie
   - Authenticated: renders `modules/{default_page}/pages/index.php`

---

## Authentication & Security

- **Session-based auth** via `Core\Auth` class
- **CSRF protection**: token generated per page load, stored in both `$_SESSION['csrf-token']` and cookie `CSRF-TOKEN`
- **Auth entry points**: `login` and `mobileLogin` AJAX actions bypass auth check
- **Session expiry**: stale sessions return an alert redirect script to `/`
- Cache-control headers set to `no-store, no-cache` on all responses

---

## Core PHP Classes

PHP classes use the `Core\` namespace. The autoloader/bootstrap is at:

```
/core/connect.php   (required by index.php via require_once)
```

Vendor dependencies are at `/var/www/html/core/vendor/autoload.php` (Composer autoload).

Key classes:
- `Core\Auth` — login state, CSRF token generation, default page per role
- `Core\Db` — database abstraction layer (`select`, `selectOne`, `insert`, `getRegistry`, etc.)
- `Core\Gui` — UI building utilities (forms, tables, filters)
- `Core\Registry` — data registry management
- `Core\Date` — date/time utilities
- `Core\WebSocketServer` — real-time communication
- `R` — shorthand database class used in web-push handlers

---

## Frontend JavaScript

The frontend uses three global namespace objects — avoid polluting the global scope with anything else.

### `el_app` (js/core/app.js — 142 KB, ~2,233 lines)

The primary application object. All frontend functionality is attached as methods to `el_app`:

- `loadContent()` — AJAX content loader
- `setMainContent()` — sets main page content area
- `dialog_open()` / `dialog_close()` — modal management
- `loader_show()` / `loader_hide()` — loading indicators
- `reloadMainContent()` — refreshes current page content
- Push notification registration and WebSocket initialization

Extend this object for new frontend features.

### `el_tools` (js/core/tools.js — 47 KB, ~1,274 lines)

Utility/helper object:

- `notify()` — custom alert/confirm dialogs (overrides native `alert()`/`confirm()`)
- `notify_close()` — close notification dialogs
- Toast notification helpers

### `el_registry` (modules/*/js/registry.js)

Per-module registry object for CRUD operations on dynamic data tables.

### `autocomplete.js` / `suggest.js`

Reusable input autocomplete and suggestion components.

### JavaScript Patterns

```javascript
// Async/await with el_app
async function handler() {
    await el_app.loadContent(url, params);
}

// Event-based communication
PromiseAlert.on('alert_close', resolve);
$(element).trigger('alert_confirmed');

// jQuery chaining convention
$('.selector').off('click').on('click', handler);

// WebSocket check before send
if (ws.readyState === WebSocket.OPEN) {
    ws.send(JSON.stringify({ type: 'event', data: payload }));
}
```

### Asset Scripts (`js/assets/`)

Standalone scripts for specific features:
- `cades_sign.js` / `cadesplugin_api.js` / `nmcades_plugin_api.js` — CryptoPro digital signature (Russian PKI standard)
- `agreement_list.js` — Agreement list UI
- `certificate.js` — Certificate handling
- `quarter_select.js` — Quarter/period date selector

---

## Database Patterns

**Database:** PostgreSQL, accessed via `Core\Db`.

### Query Methods

```php
$db->select('table_name', 'WHERE id = ?', [$id])
$db->selectOne('table_name', 'WHERE id = ?', [$id])
$db->insert('table_name', $dataArray)
$db->getRegistry($registryId)
$db->db::getAll('SELECT ... FROM ... WHERE ...', [$params])
```

### Table Naming

- Use `TBL_PREFIX` constant for table name prefixes
- Registry tables: `registry`, `regfields`, `regprops`
- Domain tables: `documents`, `checksplans`, `institutions`, `units`, `users`, `persons`, `inspections`, `tasks`
- Push subscriptions: `cam_subscriptions`

### Special Patterns

- JSON columns for complex nested data (e.g., `pl->addinstitution`)
- PostgreSQL upsert: `ON CONFLICT` clauses in push subscription handling
- Parameterized queries using `?` placeholders throughout

---

## WebSocket Server

The Ratchet-based `BroadcastingServer` broadcasts messages to **all connected clients**.

- **Plain WS**: port `3010`
- **TLS WSS**: port `3011`
- Start server: `php /path/to/ws/server.php`
- Send message: POST to `ws/send_message.php`

---

## Web Push Notifications

VAPID keys are loaded from environment variables (`.env`):

```
VAPID_PUBLIC_KEY=...
VAPID_PRIVATE_KEY=...
VAPID_SUBJECT=...
```

Flow: browser subscribes → subscription stored → `web-push/push.php` sends notification via Composer `web-push` library.

---

## Environment Configuration

Secrets and environment-specific config are stored in a `.env` file at the document root, loaded via `vlucas/phpdotenv`.

**Never commit `.env` to version control.**

Required env vars:
- `VAPID_PUBLIC_KEY`
- `VAPID_PRIVATE_KEY`
- `VAPID_SUBJECT`

---

## Development Conventions

### Adding a New Module

1. Create directory: `modules/{name}/`
2. Add `module.json` with correct `id`, `path`, `name`, `icon`, `number`
3. Create `pages/index.php` (full render) and `pages/index_ajax.php` (AJAX content)
4. Add `dialogs/` and `ajaxHandlers/` as needed
5. Register module in the system database/config (via `Core\Auth::getDefaultPage()` role mapping)

### AJAX Handler Pattern

- Handler file: `modules/{module}/ajaxHandlers/{action}.php`
- Called via POST: `ajax=1`, `action={action}`, `path={module}`
- Response: HTML fragment or JSON (no wrapping shell)

### Dialog/Popup Pattern

- Dialog file: `modules/{module}/dialogs/{name}.php`
- Called via POST: `ajax=1`, `mode=popup`, `module={module}`, `url={name}`
- Response: HTML fragment injected into modal container

### Security Rules

- Always validate `$_POST['action']` (strip `.` and `/` — already done in `index.php` for default mode)
- Check `$auth->isLogin()` inside handlers for sensitive operations
- Use CSRF token from session/cookie for state-changing requests
- Never trust `$_SERVER['HTTP_REFERER']` for path resolution alone

### PHP Conventions

- Use `Core\` namespace for shared classes
- Use `require_once $_SERVER['DOCUMENT_ROOT'] . '/core/connect.php'` to bootstrap
- No framework — write plain PHP with HTML output in page/dialog files

### JavaScript Conventions

- Attach new functionality to `el_app` object (defined in `js/core/app.js`)
- Use `el_tools` utility functions before reinventing notification/dialog logic
- No build step — files are served directly (no bundler/transpiler)
- Use JSDoc comments with `@function`, `@memberof`, `@param` tags
- Use jQuery for DOM — no raw `document.querySelector` chains
- Comments and variable names may be in Russian (Cyrillic) — this is expected

---

## Git Workflow

- Main branch: `master`
- Development branches: `claude/{description}` pattern (as used by AI assistants)
- No CI/CD configuration present — manual deployment assumed
- Commit history uses simple messages like "Add files via upload"

---

## Known TODOs (from `index.php` header)

- Directory/registry table creation UI
- Logo preloader (spinner)
- Field drag & drop in form builder
- Inspection template builder with code-based lookup
- Logging system with event type classification

---

## Codebase Scale

- ~83 source files (PHP, JS, JSON, CSS)
- ~27,650 lines of PHP and JavaScript
- `js/core/app.js` — 142 KB (~2,233 lines), core frontend logic
- `js/core/tools.js` — 47 KB (~1,274 lines)
- `core/composer.lock` — 191 KB
- `checkappmodile.svg` (~1.8 MB) / `modules.zip` (~1.7 MB) — large binary assets, do not edit
