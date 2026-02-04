# CMS Admin Auth Integration Status

This document tracks the integration status of CMS admin auth across the platform.

---

## Overview

- **CMS** issues JWT tokens for Platform Console
- **Platform Console** uses JWT tokens for CMS and CRM APIs
- **CRM** verifies CMS-issued JWT tokens with shared secret

---

## Implementation Status

### CMS Service (Content-Management-System)

| Component | Status | Notes |
|-----------|--------|-------|
| AdminUser Model | ✅ Complete | `src/models/admin_user.go` with UUID public_id, role, permissions, password_hash |
| JWT Utils | ✅ Complete | `src/utils/auth.go` with HS256, bcrypt, 24h expiration |
| Auth Middleware | ✅ Complete | `src/utils/admin_auth_middleware.go` validates JWT for /admin/* routes |
| Login Controller | ✅ Complete | `src/controllers/adminAuthController.go` POST /admin/login |
| Me Controller | ✅ Complete | `src/controllers/adminAuthController.go` GET /admin/me |
| Rate Limiter | ✅ Complete | `src/utils/rate_limiter.go` in-memory, 5 attempts/15min |
| Admin Routes | ✅ Complete | `src/routes/adminAuthRoutes.go` POST /admin/login, GET /admin/me |
| Admin Seeding | ✅ Complete | `src/utils/database.go` SeedAdminUser for dev only |
| Migrations | ✅ Complete | `supabase/migrations/20260113000000_init.sql` admin_users table |
| Tests | ✅ Complete | `src/tests/integration/admin_auth_test.go` login/me tests |

### CRM Service

| Component | Status | Notes |
|-----------|--------|-------|
| JWT Auth Middleware | ✅ Complete | `src/middleware/auth.go` updated to verify CMS-issued JWTs |
| Database Context | ✅ Complete | Database set in Gin context via middleware |
| JWT Claims | ✅ Complete | Supports both CMS admin (sub + permissions) and CRM user (user_id) tokens |
| Me Endpoint | ✅ Complete | `src/handlers/auth.go` GET /admin/me |

### Platform Console

| Component | Status | Notes |
|-----------|--------|-------|
| Auth Store | ✅ Complete | `src/lib/stores/auth.ts` Zustand with localStorage |
| Auth API Client | ✅ Complete | `src/lib/api/cms/auth.ts` login/getMe functions |
| Login Form | ✅ Complete | `src/components/auth/login-form.tsx` React Hook Form |
| Auth Provider | ✅ Complete | `src/lib/stores/auth.ts` checkAuth on mount |
| Auth Guard | ✅ Complete | `src/components/auth/auth-guard.tsx` Redirects on 401 |
| API Client | ✅ Complete | `src/lib/api/client.ts` Authorization header + 401/403 handling |

---

## Environment Variables Required

### CMS (Content-Management-System)
```
JWT_SECRET=your_secure_secret_here        # Required - HS256 signing key
JWT_EXPIRATION_HOURS=24                   # Optional - Default 24 hours
ADMIN_EMAIL=admin@example.com              # Dev only - Default admin user
ADMIN_PASSWORD=ChangeMe123!              # Dev only - Default admin password
ADMIN_ROLE=admin                          # Dev only - Default admin role
```

### Platform Console
```
NEXT_PUBLIC_CMS_BASE_URL=http://localhost:8080
NEXT_PUBLIC_CRM_BASE_URL=http://localhost:8081
```

### CRM Service
```
JWT_SECRET=<same as CMS>                # Required - Must match CMS JWT_SECRET
```

---

## API Endpoints

### CMS Admin Endpoints

#### POST /admin/login
Authenticates user and issues JWT token.

**Request:**
```json
{
  "email": "admin@example.com",
  "password": "ChangeMe123!"
}
```

**Success (200):**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIs...",
  "user": {
    "id": "uuid-string",
    "email": "admin@example.com",
    "role": "admin"
  }
}
```

**Errors:**
- 400: Invalid/missing fields
- 401: Invalid credentials or disabled account
- 429: Rate limited (5 attempts per 15 minutes)
- 500: Internal server error

#### GET /admin/me
Validates token and returns current user info.

**Headers:**
```
Authorization: Bearer <jwt_token>
```

**Success (200):**
```json
{
  "id": "uuid-string",
  "email": "admin@example.com",
  "role": "admin",
  "permissions": ["read:sources", "write:sources", ...]
}
```

**Errors:**
- 401: Missing/invalid/expired token
- 403: Forbidden (disabled account)

### CRM Admin Endpoints

#### GET /admin/me
Returns current user from JWT claims (CMS-issued or CRM-issued).

**Headers:**
```
Authorization: Bearer <jwt_token>
```

**Success (200):**
```json
{
  "user": {
    "id": "uuid-string",  // For CMS admin tokens
    "email": "admin@example.com",
    "role": "admin",
    "permissions": [...]
  },
  "permissions": [...]
}
```

---

## Authentication Flow

1. **Login:** Platform Console → POST /admin/login (CMS)
2. **JWT Issuance:** CMS validates credentials, issues HS256 JWT
3. **Token Storage:** Platform Console stores JWT in localStorage
4. **Subsequent Requests:** Platform Console sends `Authorization: Bearer <token>` header
5. **CMS Validation:** `/admin/*` routes verify JWT with AdminAuthMiddleware
6. **CRM Validation:** `/admin/*` routes verify JWT with JWTAuth middleware (same secret)

---

## Testing

### CMS Integration Tests
```bash
# In CMS directory
go test ./src/tests/integration -run Admin
```

### Platform Console E2E Tests
```bash
# In Platform Console directory
npm run test:e2e
```

### Manual Testing with cURL

#### Login
```bash
curl -X POST http://localhost:8080/admin/login \
  -H "Content-Type: application/json" \
  -d '{"email":"admin@example.com","password":"ChangeMe123!"}'
```

#### Get Me (CMS)
```bash
curl http://localhost:8080/admin/me \
  -H "Authorization: Bearer <token_from_login>"
```

#### Get Me (CRM)
```bash
curl http://localhost:8081/admin/me \
  -H "Authorization: Bearer <token_from_login>"
```

---

## Security Features

| Feature | Implementation | Notes |
|---------|----------------|--------|
| Password Hashing | ✅ bcrypt (cost factor 10) | Stored securely in database |
| JWT Signing | ✅ HS256 | Shared secret between CMS and CRM |
| Token Expiration | ✅ 24 hours (configurable) | Console handles redirect on expiration |
| Rate Limiting | ✅ 5 attempts per 15 minutes per IP | In-memory, per CMS instance |
| CORS | ✅ All origins (dev) | Tighten for production to Console domains |
| Account Status | ✅ is_active flag | Disabled accounts return 401 |

---

## Notes

- The same JWT token is used for both CMS and CRM operations
- CMS admin users are stored in `admin_users` table in CMS database
- CRM users (if any) are stored in `users` table in CRM database
- Platform Console stores token in localStorage and refreshes on page load
- On 401/403 errors, Platform Console clears auth and redirects to /login

---

## Production Checklist

- [ ] Set strong `JWT_SECRET` in production (min 256 bits)
- [ ] Configure CORS to allow only Platform Console domains
- [ ] Remove/rotate default admin credentials
- [ ] Set `ENV=production` to disable dev seeding
- [ ] Configure HTTPS only (no HTTP in production)
- [ ] Test token verification across CMS and CRM
- [ ] Set up monitoring for auth failures
- [ ] Implement Redis-based rate limiting for production multi-instance
