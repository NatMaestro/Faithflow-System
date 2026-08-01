# Subdomain Creation & Routing Architecture

## Table of Contents

1. [Overview](#overview)
2. [Backend: Subdomain Creation](#backend-subdomain-creation)
3. [Backend: Request Routing](#backend-request-routing)
4. [Frontend: Subdomain Detection](#frontend-subdomain-detection)
5. [Frontend: Routing Logic](#frontend-routing-logic)
6. [Complete Flow Examples](#complete-flow-examples)
7. [Key Components Reference](#key-components-reference)

---

## Overview

This application uses **multi-tenant architecture** with **subdomain-based routing**:

- **Backend**: Django with `django-tenants` - each church gets its own PostgreSQL schema
- **Frontend**: React with subdomain-aware routing and API client configuration
- **Isolation**: Each church's data is completely isolated in separate database schemas

### Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    User's Browser                            │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  URL: apostolicchurch.localhost:5173/admin            │  │
│  └──────────────────────────────────────────────────────┘  │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    Frontend (React)                          │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  1. SubdomainRouter detects subdomain                │  │
│  │  2. getCurrentSubdomain() → "apostolicchurch"         │  │
│  │  3. axiosClient baseURL → http://apostolicchurch...    │  │
│  │  4. Routes to /admin based on user role               │  │
│  └──────────────────────────────────────────────────────┘  │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       │ HTTP Request
                       │ GET /api/v1/members/
                       │ Host: apostolicchurch.localhost:8000
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    Backend (Django)                          │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  1. TenantMainMiddleware extracts subdomain           │  │
│  │  2. Looks up Church by subdomain in public schema     │  │
│  │  3. connection.set_tenant(church)                      │  │
│  │  4. All queries now use tenant schema                  │  │
│  │  5. Returns data from apostolicchurch schema only     │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## Backend: Subdomain Creation

### 1. Registration Flow

When a new church registers, the backend creates:

1. **Church record** (in public schema)
2. **Domain records** (maps subdomain → church)
3. **Tenant schema** (isolated database schema)
4. **Admin user** (in tenant schema)

### 2. Code Flow

**File**: `faithflow-backend/apps/authentication/views.py`

```python
class RegisterView(APIView):
    def post(self, request):
        # Step 1: Validate registration data
        serializer = ChurchRegistrationSerializer(data=request.data)
        if serializer.is_valid():
            validated_data = serializer.validated_data

            # Step 2: Create Church (in PUBLIC schema)
            church = Church.objects.create(
                schema_name=validated_data['subdomain'],  # e.g., "apostolicchurch"
                name=validated_data['church_name'],
                subdomain=validated_data['subdomain'],
                email=validated_data['email'],
                plan='trial',
                is_active=True
            )

            # Step 3: Create Domain records (maps subdomain → church)
            Domain.objects.create(
                domain=f"{validated_data['subdomain']}.localhost",  # Development
                tenant=church,
                is_primary=True
            )

            # Production domain (if not DEBUG)
            if not settings.DEBUG:
                Domain.objects.create(
                    domain=f"{validated_data['subdomain']}.faithflows.com",
                    tenant=church,
                    is_primary=False
                )

            # Step 4: Switch to tenant schema
            connection.set_tenant(church)

            # Step 5: Create admin user (in TENANT schema)
            admin_user = User.objects.create_user(
                email=validated_data['email'],
                password=validated_data['password'],
                name=validated_data['name'],
                church=church,
                role='admin',
                is_active=True
            )

            # Step 6: Generate JWT tokens
            refresh = RefreshToken.for_user(admin_user)

            return Response({
                'user': {...},
                'church': {...},
                'access': str(refresh.access_token),
                'refresh': str(refresh)
            })
```

### 3. Database Structure

```
PostgreSQL Database
├── public schema (shared)
│   ├── churches (Church model)
│   │   ├── id: 1, name: "Apostolic Church", subdomain: "apostolicchurch"
│   │   └── id: 2, name: "Grace Community", subdomain: "gracecommunity"
│   └── domains (Domain model)
│       ├── domain: "apostolicchurch.localhost" → tenant_id: 1
│       └── domain: "gracecommunity.localhost" → tenant_id: 2
│
├── apostolicchurch schema (tenant-specific)
│   ├── users (User model)
│   ├── members (Member model)
│   ├── events (Event model)
│   ├── payments (Payment model)
│   └── ... (all tenant apps)
│
└── gracecommunity schema (tenant-specific)
    ├── users (User model)
    ├── members (Member model)
    └── ... (isolated data)
```

### 4. Key Models

**File**: `faithflow-backend/apps/churches/models.py`

```python
class Church(TenantMixin):
    """
    Each church is a tenant with isolated data.
    """
    name = models.CharField(max_length=255)
    subdomain = models.CharField(max_length=63, unique=True)
    schema_name = models.CharField(max_length=63)  # Database schema name
    is_active = models.BooleanField(default=True)
    # ... other fields

    auto_create_schema = True  # Automatically create schema on save
    auto_drop_schema = False   # Don't drop schema on delete (safety)

class Domain(DomainMixin):
    """
    Maps subdomain/domain to a church tenant.
    """
    domain = models.CharField(max_length=253)  # e.g., "apostolicchurch.localhost"
    tenant = models.ForeignKey(Church, on_delete=models.CASCADE)
    is_primary = models.BooleanField(default=True)
```

---

## Backend: Request Routing

### 1. Middleware Stack

**File**: `faithflow-backend/config/settings.py`

```python
MIDDLEWARE = [
    'django_tenants.middleware.main.TenantMainMiddleware',  # MUST be first!
    'core.middleware_dev.TenantQueryMiddleware',  # Fallback for ?tenant= param
    # ... other middleware
]
```

### 2. TenantMainMiddleware Flow

**What it does**:

1. Extracts subdomain from `Host` header
2. Looks up `Domain` record in public schema
3. Finds associated `Church` tenant
4. Calls `connection.set_tenant(church)`
5. All subsequent queries use tenant schema

**Example Request**:

```
GET /api/v1/members/
Host: apostolicchurch.localhost:8000
```

**Middleware Processing**:

```python
# 1. Extract subdomain from Host header
host = request.get_host()  # "apostolicchurch.localhost:8000"
subdomain = extract_subdomain(host)  # "apostolicchurch"

# 2. Look up Domain in public schema
domain = Domain.objects.get(domain__contains=subdomain)
# Returns: Domain(domain="apostolicchurch.localhost", tenant_id=1)

# 3. Get Church tenant
church = domain.tenant
# Returns: Church(id=1, schema_name="apostolicchurch")

# 4. Set tenant for this request
connection.set_tenant(church)
# Now all queries use "apostolicchurch" schema

# 5. Process request (views, serializers, etc.)
# User.objects.all() → queries apostolicchurch.users table
# Member.objects.all() → queries apostolicchurch.members table
```

### 3. TenantQueryMiddleware (Fallback)

**File**: `faithflow-backend/core/middleware_dev.py`

For environments without wildcard DNS (e.g., Render free tier):

```python
class TenantQueryMiddleware:
    """
    Allows ?tenant=subdomain when subdomain routing isn't available.
    Usage: http://localhost:8000/api/v1/members/?tenant=apostolicchurch
    """
    def __call__(self, request):
        tenant_subdomain = request.GET.get('tenant')

        if tenant_subdomain:
            church = Church.objects.get(subdomain=tenant_subdomain)
            connection.set_tenant(church)

        return self.get_response(request)
```

### 4. URL Configuration

**File**: `faithflow-backend/config/settings.py`

```python
ROOT_URLCONF = 'config.urls_tenants'  # Tenant-aware URLs
PUBLIC_SCHEMA_URLCONF = 'config.urls_public'  # Public schema URLs (registration, etc.)
```

- **Tenant URLs**: Used when a tenant is detected (most routes)
- **Public URLs**: Used when no tenant (registration, health checks)

---

## Frontend: Subdomain Detection

### 1. Subdomain Extraction

**File**: `faithflow-studio/src/utils/subdomainUtils.ts`

```typescript
export const getCurrentSubdomain = (): string | null => {
  const hostname = window.location.hostname;
  const parts = hostname.split(".");

  // Development: subdomain.localhost
  if (hostname.includes("localhost")) {
    if (parts.length >= 2 && parts[1] === "localhost") {
      return parts[0]; // "apostolicchurch" from "apostolicchurch.localhost"
    }
    return null; // No subdomain on localhost:5173
  }

  // Production: subdomain.faithflows.com
  if (parts.length >= 3 && parts[1] === "faithflows" && parts[2] === "com") {
    return parts[0]; // "apostolicchurch" from "apostolicchurch.faithflows.com"
  }

  return null;
};
```

**Examples**:

- `apostolicchurch.localhost:5173` → `"apostolicchurch"`
- `localhost:5173` → `null`
- `apostolicchurch.faithflows.com` → `"apostolicchurch"`
- `faithflows.com` → `null`

### 2. API Client Configuration

**File**: `faithflow-studio/src/api/axiosClient.ts`

```typescript
const getBaseURL = (): string => {
  const subdomain = getCurrentSubdomain();
  const tenant = subdomain || "olamchurch"; // Default fallback

  // Development
  if (import.meta.env.DEV) {
    return `http://${tenant}.localhost:8000/api/v1`;
  }

  // Production
  return (
    import.meta.env.VITE_API_URL || `http://${tenant}.localhost:8000/api/v1`
  );
};

export const axiosClient = axios.create({
  baseURL: getBaseURL(),
  timeout: 30000,
  headers: {
    "Content-Type": "application/json",
  },
});
```

**How it works**:

- User visits `apostolicchurch.localhost:5173`
- `getCurrentSubdomain()` returns `"apostolicchurch"`
- `axiosClient.baseURL` = `"http://apostolicchurch.localhost:8000/api/v1"`
- All API calls go to the correct subdomain backend

---

## Frontend: Routing Logic

### 1. SubdomainRouter Component

**File**: `faithflow-studio/src/components/SubdomainRouter.tsx`

```typescript
const SubdomainRouter = ({ children }: SubdomainRouterProps) => {
  const navigate = useNavigate();
  const location = useLocation();
  const { isAuthenticated, user } = useAppSelector((state) => state.auth);
  const currentSubdomain = getCurrentSubdomain();

  useEffect(() => {
    // Only run on root path "/"
    if (location.pathname !== "/") return;

    // Rule 1: Subdomain + Not authenticated → /login
    if (currentSubdomain && !isAuthenticated) {
      navigate("/login", { replace: true });
      return;
    }

    // Rule 2: Subdomain + Authenticated → Dashboard based on role
    if (currentSubdomain && isAuthenticated && user) {
      if (user.role === "admin") {
        navigate("/admin", { replace: true });
      } else if (user.role === "member") {
        navigate("/member/dashboard", { replace: true });
      }
      return;
    }

    // Rule 3: No subdomain → Show landing page
    // (default behavior, no redirect)
  }, [currentSubdomain, isAuthenticated, user, location.pathname, navigate]);

  return <>{children}</>;
};
```

**Routing Rules**:

| Subdomain | Authenticated | User Role | Redirect To                |
| --------- | ------------- | --------- | -------------------------- |
| ✅ Yes    | ❌ No         | -         | `/login`                   |
| ✅ Yes    | ✅ Yes        | `admin`   | `/admin`                   |
| ✅ Yes    | ✅ Yes        | `member`  | `/member/dashboard`        |
| ❌ No     | -             | -         | Landing page (no redirect) |

### 2. ChurchLoginGuard

**File**: `faithflow-studio/src/features/auth/components/ChurchLoginGuard.tsx`

Enforces subdomain requirement for church login:

```typescript
const ChurchLoginGuard = ({ children }: ChurchLoginGuardProps) => {
  const [isValid, setIsValid] = useState(false);
  const currentSubdomain = getCurrentSubdomain();

  useEffect(() => {
    // Block login if no subdomain
    if (!currentSubdomain) {
      setIsValid(false);
      return;
    }

    // Verify subdomain exists and is active
    const church = await getChurchBySubdomain(currentSubdomain);
    if (!church || !church.isActive) {
      setIsValid(false);
      return;
    }

    setIsValid(true);
  }, []);

  if (!isValid) {
    return <ErrorComponent message="Login requires a valid church subdomain" />;
  }

  return <>{children}</>;
};
```

**Usage in routes**:

```typescript
{
  path: "/login",
  element: (
    <ChurchLoginGuard>
      <Login />
    </ChurchLoginGuard>
  ),
}
```

### 3. SubdomainEnforcement

**File**: `faithflow-studio/src/components/SubdomainEnforcement.tsx`

Enforces subdomain for admin routes:

```typescript
const SubdomainEnforcement = ({ children, requiredSubdomain }) => {
  const currentSubdomain = getCurrentSubdomain();

  // Require subdomain (except superadmin)
  if (!currentSubdomain && !isSuperAdminRoute) {
    return <ErrorComponent message="Admin panel requires a church subdomain" />;
  }

  // Verify subdomain exists and is active
  const church = await getChurchBySubdomain(currentSubdomain);
  if (!church || !church.isActive) {
    return <ErrorComponent message="Church not found or inactive" />;
  }

  return <>{children}</>;
};
```

### 4. Route Configuration

**File**: `faithflow-studio/src/app/routes.tsx`

```typescript
export const router = createBrowserRouter([
  // Root route with SubdomainRouter
  {
    path: "/",
    element: (
      <SubdomainRouter>
        <LandingPage />
      </SubdomainRouter>
    ),
  },

  // Login with ChurchLoginGuard
  {
    path: "/login",
    element: (
      <ChurchLoginGuard>
        <Login />
      </ChurchLoginGuard>
    ),
  },

  // Admin routes with AuthGuard + subdomain enforcement
  {
    path: "/admin",
    element: (
      <AuthGuard requiredRole="admin" enforceSubdomain={true}>
        <AdminDashboard />
      </AuthGuard>
    ),
  },

  // Member routes
  {
    path: "/member",
    element: (
      <AuthGuard requiredRole="member">
        <MemberDashboard />
      </AuthGuard>
    ),
  },
]);
```

---

## Complete Flow Examples

### Example 1: New Church Registration

```
1. User visits: localhost:5173/onboarding
   └─> No subdomain detected
   └─> Shows landing/onboarding page

2. User completes registration form:
   - Church Name: "Apostolic Church"
   - Subdomain: "apostolicchurch"
   - Admin Email: "admin@apostolicchurch.com"

3. Frontend calls: POST http://localhost:8000/api/v1/auth/register/
   └─> Public schema (no tenant)
   └─> Backend creates:
       ├─> Church record (public.churches)
       ├─> Domain records (public.domains)
       ├─> Tenant schema (apostolicchurch.*)
       └─> Admin user (apostolicchurch.users)

4. Frontend receives response with tokens
   └─> Stores tokens in localStorage
   └─> Redirects to: apostolicchurch.localhost:5173/admin
```

### Example 2: Admin Login

```
1. User visits: apostolicchurch.localhost:5173/login
   └─> SubdomainRouter detects subdomain: "apostolicchurch"
   └─> ChurchLoginGuard verifies subdomain exists

2. User enters credentials and submits

3. Frontend calls: POST http://apostolicchurch.localhost:8000/api/v1/auth/login/
   └─> Backend TenantMainMiddleware:
       ├─> Extracts subdomain: "apostolicchurch"
       ├─> Looks up Domain → Church
       ├─> Sets tenant: connection.set_tenant(church)
       └─> Queries User in apostolicchurch.users

4. Backend returns JWT tokens + user data

5. Frontend:
   └─> Stores tokens
   └─> Updates Redux state (auth.user, auth.isAuthenticated)
   └─> SubdomainRouter redirects to /admin (user.role === "admin")
```

### Example 3: Fetching Members (Authenticated)

```
1. User is on: apostolicchurch.localhost:5173/admin/members
   └─> Authenticated as admin
   └─> Subdomain: "apostolicchurch"

2. Component calls: dispatch(fetchMembers(churchId))

3. Frontend API call:
   GET http://apostolicchurch.localhost:8000/api/v1/members/
   Headers: { Authorization: "Bearer <token>" }

4. Backend processing:
   └─> TenantMainMiddleware:
       ├─> Extracts subdomain from Host header
       ├─> Sets tenant: connection.set_tenant(church)
       └─> All queries use apostolicchurch schema
   └─> View: MemberViewSet.list()
       └─> Queries: Member.objects.filter(church=church)
       └─> Returns: Members from apostolicchurch.members only

5. Frontend receives data
   └─> Updates Redux state
   └─> Component re-renders with members list
```

### Example 4: Wrong Subdomain Access

```
1. User visits: localhost:5173/admin
   └─> No subdomain detected
   └─> SubdomainRouter: Shows landing page (no redirect)

2. User manually navigates to: localhost:5173/admin/members
   └─> AuthGuard checks: enforceSubdomain={true}
   └─> SubdomainEnforcement:
       ├─> getCurrentSubdomain() → null
       └─> Returns error: "Admin panel requires a church subdomain"

3. User must visit: apostolicchurch.localhost:5173/admin/members
   └─> Subdomain detected: "apostolicchurch"
   └─> Access granted
```

---

## Key Components Reference

### Backend

| Component                 | File                           | Purpose                          |
| ------------------------- | ------------------------------ | -------------------------------- |
| `Church` model            | `apps/churches/models.py`      | Tenant model (one per church)    |
| `Domain` model            | `apps/churches/models.py`      | Maps subdomain → church          |
| `TenantMainMiddleware`    | `django_tenants.middleware`    | Routes requests to tenant schema |
| `TenantQueryMiddleware`   | `core/middleware_dev.py`       | Fallback for ?tenant= param      |
| `RegisterView`            | `apps/authentication/views.py` | Creates church + tenant schema   |
| `connection.set_tenant()` | Django-tenants                 | Switches to tenant schema        |

### Frontend

| Component               | File                                            | Purpose                               |
| ----------------------- | ----------------------------------------------- | ------------------------------------- |
| `getCurrentSubdomain()` | `utils/subdomainUtils.ts`                       | Extracts subdomain from URL           |
| `axiosClient`           | `api/axiosClient.ts`                            | Configures API baseURL with subdomain |
| `SubdomainRouter`       | `components/SubdomainRouter.tsx`                | Routes based on subdomain + auth      |
| `ChurchLoginGuard`      | `features/auth/components/ChurchLoginGuard.tsx` | Enforces subdomain for login          |
| `SubdomainEnforcement`  | `components/SubdomainEnforcement.tsx`           | Enforces subdomain for admin routes   |
| `AuthGuard`             | `features/auth/components/AuthGuard.tsx`        | Checks auth + permissions             |

---

## Important Notes

### 1. Schema Isolation

- Each church's data is **completely isolated** in separate schemas
- No cross-tenant data leakage possible
- Users from one church cannot access another church's data

### 2. Subdomain Requirements

- **Admin routes**: Require subdomain (enforced by `SubdomainEnforcement`)
- **Member routes**: Can work without subdomain (but subdomain is recommended)
- **Superadmin routes**: No subdomain required
- **Public routes**: No subdomain required

### 3. Development vs Production

**Development**:

- Subdomain format: `subdomain.localhost:5173` (frontend)
- Backend format: `subdomain.localhost:8000` (backend)
- Requires `/etc/hosts` entries or local DNS

**Production**:

- Subdomain format: `subdomain.faithflows.com`
- Requires wildcard DNS: `*.faithflows.com` → backend server
- Backend must handle all subdomains

### 4. Fallback Mechanisms

- **TenantQueryMiddleware**: Allows `?tenant=subdomain` when DNS isn't available
- **Default tenant**: `axiosClient` falls back to `'olamchurch'` if no subdomain
- **Public schema**: Registration and health checks work without tenant

---

## Troubleshooting

### Issue: "Access denied. Admin panel requires a church subdomain"

**Cause**: User is accessing admin route without subdomain  
**Solution**: Visit `subdomain.localhost:5173/admin` instead of `localhost:5173/admin`

### Issue: "Church not found"

**Cause**: Subdomain doesn't exist in database  
**Solution**: Verify church was created and Domain record exists

### Issue: API calls going to wrong subdomain

**Cause**: `getCurrentSubdomain()` returning wrong value  
**Solution**: Check browser hostname and `subdomainUtils.ts` logic

### Issue: Backend returning data from wrong church

**Cause**: `TenantMainMiddleware` not setting tenant correctly  
**Solution**: Verify Domain record exists and middleware is first in stack

---

## Summary

1. **Subdomain Creation**: Backend creates Church + Domain + Tenant schema during registration
2. **Backend Routing**: `TenantMainMiddleware` extracts subdomain and sets tenant schema
3. **Frontend Detection**: `getCurrentSubdomain()` extracts subdomain from URL
4. **API Configuration**: `axiosClient` uses subdomain in baseURL
5. **Frontend Routing**: `SubdomainRouter` routes based on subdomain + auth state
6. **Enforcement**: Guards ensure subdomain is present for admin routes

This architecture provides **complete data isolation** while maintaining a **single codebase** for all churches.
