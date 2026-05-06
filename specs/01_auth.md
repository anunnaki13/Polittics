# Spec 01 — Authentication

## Purpose

User can register an account, login, and remain authenticated across sessions. JWT-based.

## User Stories

### As a new visitor
- I can click "Daftar" on the home page
- I can fill in email, username, password, optional display name
- I receive immediate feedback on validation errors (e.g., username taken)
- After successful registration, I am logged in automatically

### As a returning user
- I can click "Masuk" on the home page
- I can enter my email and password
- I see clear error if credentials are wrong
- I see lockout message if I try too many times
- After successful login, I land on the home dashboard

### As a logged-in user
- I see my username/avatar in the header
- I can click "Keluar" to logout
- After logout, I am redirected to home page (logged out state)

## Backend Implementation

### Files
```
backend/app/auth/
├── __init__.py
├── router.py         # FastAPI endpoints
├── service.py        # Business logic
├── schemas.py        # Pydantic models
├── models.py         # SQLAlchemy User model
├── jwt.py            # Token utilities
└── dependencies.py   # get_current_user dep
```

### Endpoints
See `docs/05_api_design.md` for full specs:
- `POST /api/auth/register`
- `POST /api/auth/login`
- `POST /api/auth/refresh`
- `POST /api/auth/logout`
- `GET /api/auth/me`

### Key Logic

#### Registration
```python
async def register_user(db, data: RegisterRequest) -> RegisterResponse:
    # Validate
    await check_email_unique(db, data.email)
    await check_username_unique(db, data.username)
    
    # Hash password
    password_hash = ph.hash(data.password)
    
    # Create user
    user = User(
        email=data.email.lower(),
        username=data.username,
        password_hash=password_hash,
        display_name=data.display_name or data.username,
    )
    db.add(user)
    await db.commit()
    
    # Issue tokens
    access = create_access_token(user.id, user.username)
    refresh = create_refresh_token(user.id)
    
    return RegisterResponse(
        user=UserResponse.from_orm(user),
        access_token=access,
        refresh_token=refresh,
    )
```

#### Login
```python
async def login_user(db, data: LoginRequest, ip: str) -> LoginResponse:
    # Rate limit check (Redis)
    await check_login_rate_limit(ip, data.email)
    
    # Find user
    user = await get_user_by_email(db, data.email.lower())
    if not user:
        # Constant-time response (don't reveal user doesn't exist)
        await asyncio.sleep(0.5)
        raise InvalidCredentialsError()
    
    # Verify password
    try:
        ph.verify(user.password_hash, data.password)
    except VerifyMismatchError:
        raise InvalidCredentialsError()
    
    # Rehash if needed
    if ph.check_needs_rehash(user.password_hash):
        user.password_hash = ph.hash(data.password)
        await db.commit()
    
    # Update last_login_at
    user.last_login_at = datetime.utcnow()
    await db.commit()
    
    # Issue tokens
    return LoginResponse(...)
```

### Security Notes
- Passwords hashed with Argon2id
- JWT secret must be strong (64+ char hex)
- Refresh tokens have unique `jti` for revocation
- Logged-out tokens added to Redis blacklist with TTL
- Constant-time comparison for credentials

## Frontend Implementation

### Files
```
frontend/src/
├── pages/
│   ├── LoginPage.tsx
│   ├── RegisterPage.tsx
│   └── HomePage.tsx
├── components/auth/
│   ├── LoginForm.tsx
│   └── RegisterForm.tsx
├── stores/authStore.ts
├── hooks/useAuth.ts
└── lib/api.ts
```

### Auth Store (Zustand)
```typescript
interface AuthState {
  user: User | null;
  accessToken: string | null;
  isAuthenticated: boolean;
  login: (credentials: LoginRequest) => Promise<void>;
  register: (data: RegisterRequest) => Promise<void>;
  logout: () => Promise<void>;
  refreshToken: () => Promise<void>;
}
```

Access token kept in memory (more secure than localStorage). Refresh token in HTTPOnly cookie (set by backend, JS can't read).

### Protected Routes
```typescript
function ProtectedRoute({ children }: { children: ReactNode }) {
  const { isAuthenticated } = useAuth();
  if (!isAuthenticated) return <Navigate to="/login" />;
  return <>{children}</>;
}
```

### Token Refresh Flow
```typescript
// API client interceptor
async function apiCall(url, options) {
  let response = await fetch(url, {
    ...options,
    credentials: 'include',
    headers: { ...options.headers, Authorization: `Bearer ${getAccessToken()}` },
  });
  
  if (response.status === 401) {
    // Try to refresh
    await refreshToken();
    response = await fetch(url, /* retry */);
  }
  
  return response;
}
```

## Validation

### Email
- Valid format (use Zod's email validator)
- Lowercased before storage and comparison
- Max length 255

### Username
- 3-32 characters
- Alphanumeric + underscore only (no dots, dashes, etc.)
- Lowercased? No — preserve case but compare case-insensitively
- Cannot start with underscore or number
- Cannot be reserved words (admin, system, root, arena)

### Password
- 8-128 characters
- At least 1 letter and 1 number
- No other complexity requirements (long is enough)

### Display Name (optional)
- 1-64 characters
- Any printable characters
- Trimmed of whitespace
- Defaults to username if not provided

## Tests

### Backend
- `test_register_success` — Happy path
- `test_register_duplicate_email` — Returns 409
- `test_register_duplicate_username` — Returns 409
- `test_register_invalid_email` — Returns 422
- `test_register_weak_password` — Returns 422
- `test_login_success` — Happy path
- `test_login_wrong_password` — Returns 401
- `test_login_nonexistent_email` — Returns 401 (constant time)
- `test_login_rate_limit` — Returns 429 after 5 attempts
- `test_refresh_token_success` — Returns new tokens
- `test_refresh_token_blacklisted` — Returns 401
- `test_logout_blacklists_token` — Refresh after logout fails
- `test_get_me_authenticated` — Returns user
- `test_get_me_unauthenticated` — Returns 401

### Frontend
- `LoginForm.test.tsx` — Renders, submits, shows errors
- `RegisterForm.test.tsx` — Same
- `useAuth.test.ts` — Login flow, logout flow, persisted session
- E2E: `auth.e2e.ts` — Full register-login-logout flow

## Acceptance Criteria

- [ ] User can register with valid data and is auto-logged-in
- [ ] User cannot register with duplicate email or username
- [ ] User can login with correct credentials
- [ ] User cannot login with wrong credentials
- [ ] Login is rate-limited after 5 failed attempts
- [ ] Logged-in user can call protected endpoints
- [ ] Token refreshes automatically before expiry
- [ ] User can logout and is redirected to login page
- [ ] Logged-out user cannot access protected routes
- [ ] All UI text in Bahasa Indonesia
- [ ] Forms have proper error display in Bahasa Indonesia
- [ ] Tests pass (backend + frontend)
