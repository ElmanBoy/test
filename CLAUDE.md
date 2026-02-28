# CLAUDE.md — MonitoringCRM Codebase Guide

## Project Overview

**Подсистема контроля и надзора** (Monitoring Control and Supervision Subsystem), short name **MonitoringCRM**, is a PHP + vanilla JavaScript web application delivered as a Progressive Web App (PWA). It is hosted at `monitoring.msr.mosreg.ru`.

The system supports regulatory monitoring workflows: task assignment, registries, API management, and real-time notifications via WebSocket and Web Push.

---

## Technology Stack

| Layer | Technology |
|---|---|
| Backend | PHP (namespaced OOP, `Core\Auth`, `Core\Db`) |
| Frontend | Vanilla JavaScript (no framework) |
| PWA | Service Worker + Web App Manifest |
| WebSockets | Ratchet PHP library (ports 3010 / 3011 TLS) |
| Web Push | VAPID-based push notifications |
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
│   ├── api/               # API management module
│   └── assigned/          # Task assignment module
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
- `Core\Db` — database access layer

---

## Frontend JavaScript

### `el_app` (js/core/app.js)

The global application object. All frontend functionality is attached as methods to `el_app`. This is the primary frontend API — prefer extending this object for new frontend features.

### `tools.js`

Utility/helper functions. Use these before writing new utility code.

### `autocomplete.js` / `suggest.js`

Reusable input autocomplete and suggestion components.

### Asset Scripts (`js/assets/`)

Standalone scripts for specific features:
- `cades_sign.js` / `cadesplugin_api.js` / `nmcades_plugin_api.js` — CryptoPro digital signature (Russian PKI standard)
- `agreement_list.js` — Agreement list UI
- `certificate.js` — Certificate handling
- `quarter_select.js` — Quarter/period date selector

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
- Use utility functions from `tools.js` before reinventing
- No build step — files are served directly (no bundler/transpiler)

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

## File Size Notes

- `js/core/app.js` — 142 KB (large, core frontend logic)
- `js/core/tools.js` — 47 KB
- `core/composer.lock` — 191 KB
- `checkappmodile.svg` / `modules.zip` — large binary assets, do not edit in place
