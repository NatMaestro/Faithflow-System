# Active API Endpoints

## Public Schema Endpoints (Accessible from `localhost:8000` or base domain)

### Authentication (Public - for registration)

- `POST /api/v1/auth/register/` - **Church registration** (use this for new church signup)
- `POST /api/v1/auth/login/` - Super admin login
- `POST /api/v1/auth/refresh/` - Token refresh

### Setup

- `POST /api/v1/setup/initial/` - Initial setup (requires setup token)

### Documentation

- `GET /api/docs/` - Swagger UI
- `GET /api/redoc/` - ReDoc
- `GET /api/schema/` - OpenAPI schema

---

## Tenant Schema Endpoints (Accessible from `{subdomain}.localhost:8000`)

### Authentication (Tenant-specific)

- `POST /api/v1/auth/login/` - User login
- `POST /api/v1/auth/register/` - User registration (within tenant)
- `POST /api/v1/auth/logout/` - Logout
- `POST /api/v1/auth/refresh/` - Token refresh
- `GET /api/v1/auth/me/` - Get current user
- `POST /api/v1/auth/change-password/` - Change password
- `POST /api/v1/auth/forgot-password/` - Forgot password
- `POST /api/v1/auth/reset-password/` - Reset password
- `GET /api/v1/auth/users/` - List users
- `GET /api/v1/auth/users/{id}/` - Get user details

### Churches

- `GET /api/v1/churches/` - List churches (tenant info)
- `GET /api/v1/churches/{id}/` - Get church details

### Members

- `GET /api/v1/members/` - List members
- `POST /api/v1/members/` - Create member
- `GET /api/v1/members/{id}/` - Get member
- `PUT /api/v1/members/{id}/` - Update member
- `DELETE /api/v1/members/{id}/` - Delete member

### Events

- `GET /api/v1/events/` - List events
- `POST /api/v1/events/` - Create event
- `GET /api/v1/events/{id}/` - Get event
- `PUT /api/v1/events/{id}/` - Update event
- `DELETE /api/v1/events/{id}/` - Delete event

### Payments & Giving

- `GET /api/v1/payments/` - List payments
- `POST /api/v1/payments/` - Create payment
- `GET /api/v1/giving/` - Giving history

### Ministries

- `GET /api/v1/ministries/` - List ministries
- `POST /api/v1/ministries/` - Create ministry

### Volunteers

- `GET /api/v1/volunteer-opportunities/` - List opportunities
- `GET /api/v1/volunteer-signups/` - List signups
- `GET /api/v1/volunteer-hours/` - List hours

### Requests

- `GET /api/v1/service-requests/` - List service requests
- `GET /api/v1/requests/` - Alias for service requests

### Prayer Requests

- `GET /api/v1/prayer-requests/` - List prayer requests
- `POST /api/v1/prayer-requests/` - Create prayer request

### Altar Calls

- `GET /api/v1/altar-calls/` - List altar calls
- `POST /api/v1/altar-calls/` - Create altar call

### Announcements

- `GET /api/v1/announcements/` - List announcements
- `POST /api/v1/announcements/` - Create announcement

### Notifications

- `GET /api/v1/notifications/` - List notifications

### Roles & Permissions

- `GET /api/v1/roles/` - List roles
- `GET /api/v1/permissions/` - List permissions
- `GET /api/v1/user-roles/` - List user roles

### Themes

- `GET /api/v1/themes/` - Get theme
- `PUT /api/v1/themes/` - Update theme

### Documents

- `GET /api/v1/documents/` - List documents
- `POST /api/v1/documents/` - Upload document

### Dashboard & Analytics

- `GET /api/v1/dashboard/` - Dashboard data
- `GET /api/v1/analytics/` - Analytics data
- `GET /api/v1/reports/` - Reports

---

## Important Notes:

1. **For Church Registration**: Use `http://localhost:8000/api/v1/auth/register/` (public schema)

   - ✅ Correct: `http://localhost:8000/api/v1/auth/register/`
   - ❌ Wrong: `https://localhost:8000/api/v1/auth/register/` (HTTPS not supported in dev)
   - ❌ Wrong: `http://olamchurch.localhost:8000/api/v1/auth/register/` (tenant doesn't exist yet)

2. **For User Login**: Use `http://{subdomain}.localhost:8000/api/v1/auth/login/` (tenant schema)

   - Example: `http://olamchurch.localhost:8000/api/v1/auth/login/`

3. **301 Redirects - Common Causes**:

   - ❌ Using `https://` instead of `http://` in development
   - ❌ Missing trailing slash (Django redirects `/register` → `/register/`)
   - ❌ Accessing tenant endpoint before tenant exists
   - ✅ **Fix**: Always use `http://localhost:8000` for registration (public schema)

4. **CORS Preflight (OPTIONS requests)**:
   - Make sure your frontend is using `http://` not `https://`
   - The 301 redirect on OPTIONS requests breaks CORS
