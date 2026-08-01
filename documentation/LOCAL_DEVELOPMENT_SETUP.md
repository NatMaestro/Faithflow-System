# Local Development Setup with Caddy

This guide explains how to set up local HTTPS development with subdomain support for testing multi-tenant functionality.

## Why Caddy?

- Enables `*.localhost` subdomain testing (e.g., `apostolicchurchghana.localhost`)
- Automatic HTTPS with self-signed certificates
- Cookies work correctly (same-site on `*.localhost`)
- No Python 3.13 compatibility issues
- **Local only** - doesn't affect production

## Prerequisites

1. Install Caddy: https://caddyserver.com/docs/install
   - Windows: `choco install caddy` or download from website
   - Mac: `brew install caddy`
   - Linux: Follow official install instructions

2. Ensure your backend and frontend can run on:
   - Backend: `localhost:8000` (HTTP)
   - Frontend: `localhost:8080` (HTTP)

## Setup Steps

### 1. Create Logs Directory (Optional)

```bash
mkdir logs
```

### 2. Update Backend `.env` for Local HTTPS

Add to `faithflow-backend/.env`:

```env
# HTTPS + Cross-site cookies (for subdomain.localhost ↔ localhost)
SESSION_COOKIE_SECURE=True
CSRF_COOKIE_SECURE=True
SESSION_COOKIE_SAMESITE=None
CSRF_COOKIE_SAMESITE=None
COOKIE_DOMAIN=.localhost

# CORS/CSRF - Base origins (subdomains handled dynamically)
CORS_ALLOWED_ORIGINS=https://localhost:443
CSRF_TRUSTED_ORIGINS=https://localhost:443
```

### 3. Update Frontend Vite Config

Ensure `vite.config.ts` has the correct port:

```typescript
server: {
  host: "::",
  port: 8080,  // Must match Caddyfile
}
```

### 4. Start Services

**Terminal 1 - Backend (HTTP):**
```bash
cd faithflow-backend
python manage.py runserver 0.0.0.0:8000
```

**Terminal 2 - Frontend (HTTP):**
```bash
cd faithflow-studio
npm run dev
```

**Terminal 3 - Caddy (HTTPS Proxy):**
```bash
# From project root
caddy run
```

### 5. Access Your App

- Open: `https://apostolicchurchghana.localhost` (or any subdomain)
- Accept the self-signed certificate warning (once per browser)
- Cookies will work correctly!

## Testing Different Churches

You can test any subdomain:
- `https://apostolicchurchghana.localhost`
- `https://anychurch.localhost`
- `https://testchurch.localhost`

All will work because Caddy handles `*.localhost` automatically.

## Troubleshooting

### Certificate Warning
- First time: Browser will show "Not Secure" warning
- Click "Advanced" → "Proceed to site"
- Warning won't appear again for that browser

### Port Conflicts
- If port 443 is taken, Caddy will use 8443
- Access via `https://apostolicchurchghana.localhost:8443`
- Update CORS/CSRF origins accordingly

### Caddy Not Starting
- Check if port 443/8443 is available
- Check Caddyfile syntax: `caddy validate`
- Check logs: `logs/caddy.log`

## Stopping Caddy

Press `Ctrl+C` in the Caddy terminal. Your backend and frontend continue running normally.

## Production

**This setup is LOCAL ONLY:**
- Caddyfile is in `.gitignore` (not deployed)
- Production uses Render's HTTPS automatically
- No changes to production code or deployment

## Alternative: Plain Localhost (No Subdomains)

If you don't need subdomain testing:
- Just run backend on `localhost:8000` and frontend on `localhost:8080`
- Use `SESSION_COOKIE_SAMESITE=Lax` (no HTTPS needed)
- Cookies work because same-site

