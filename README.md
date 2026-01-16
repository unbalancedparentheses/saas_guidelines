# SaaS Application Checklist

A comprehensive checklist for building production-ready SaaS applications with Elixir/Phoenix and React/TypeScript.

---

## Quick Start: How We Work

Read this section to understand our stack, patterns, and philosophy in 5 minutes.

### Tech Stack at a Glance

```
Backend:   Elixir + Phoenix (API only, no HTML views)
Frontend:  React + TypeScript + Vite
Database:  PostgreSQL
Cache:     Redis
Search:    Meilisearch
Jobs:      Oban (PostgreSQL-backed)
Dev:       Nix flakes (no Docker)
Deploy:    Bare metal + NixOS + systemd + Caddy
```

### Architecture Overview

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   React SPA     │────▶│  Phoenix API    │────▶│   PostgreSQL    │
│   (Vite/pnpm)   │     │  (REST + JSON)  │     │   (via Ecto)    │
└─────────────────┘     └─────────────────┘     └─────────────────┘
                               │
                        ┌──────┴──────┐
                        ▼             ▼
                 ┌──────────┐  ┌──────────┐
                 │  Redis   │  │   Oban   │
                 │ (cache)  │  │  (jobs)  │
                 └──────────┘  └──────────┘
```

### Key Libraries

| Layer | Library | Purpose |
|-------|---------|---------|
| HTTP | Bandit | Modern Elixir HTTP server |
| Auth | Joken | JWT tokens (access + refresh) |
| Passwords | Argon2 | Password hashing |
| JSON | Jason | Fast serialization |
| Jobs | Oban | Background job processing |
| Frontend State | Zustand | Client state (UI only) |
| Server State | TanStack Query | API calls (no caching) |
| Forms | React Hook Form + Zod | Validation |
| UI | shadcn/ui + Radix | Accessible components |
| Toasts | Sonner | Notifications |

### Core Principles

**1. API-Only Phoenix**
- Phoenix serves REST/JSON only, no server-rendered HTML
- React handles all UI, routing, and user interaction
- Clear separation: backend = data + logic, frontend = presentation

**2. Nix Everything, No Docker**
- Reproducible dev environments with Nix flakes
- Local services (Postgres, Redis) via devenv or process-compose
- Production servers run NixOS for consistency
- `nix develop` gets you a working environment instantly

**3. Offline-First Frontend**
- Dexie.js (IndexedDB) for local data persistence
- API calls always fetch fresh data (no TanStack Query caching)
- Sync queue for mutations made offline
- Works without network, syncs when reconnected

**4. Single Shared Repository (Umbrella)**
- All Ecto schemas in one `my_app_repo` app
- All migrations in one place
- Domain apps (auth, billing, etc.) contain only business logic
- Single `Repo` module used by all apps

**5. UUIDs Everywhere**
- No auto-increment IDs (prevents enumeration attacks)
- Generate IDs client-side when needed
- Safe for distributed systems and data merging

**6. Deterministic Test Data**
- No Faker or random generators
- All test data uses explicit, predictable values
- Seeds produce identical data on every run
- Tests are reproducible and debuggable

### Authentication Flow

```
1. Login:     POST /api/v1/auth/login → {access_token, refresh_token}
2. API calls: Authorization: Bearer <access_token>
3. Refresh:   POST /api/v1/auth/refresh (when access_token expires)
4. Logout:    DELETE /api/v1/auth/logout (revokes refresh_token)
```

- Access tokens: 15 minutes, stateless JWT
- Refresh tokens: 7 days, stored in database, rotated on use

### Project Structure

```
backend/
  apps/
    my_app_repo/        # Ecto repo + ALL schemas + migrations
    my_app_auth/        # Authentication domain
    my_app_billing/     # Billing domain
    my_app_web/         # Phoenix router, controllers, plugs

frontend/
  apps/
    platform/           # Main SPA
  packages/
    ui/                 # Shared components
    api/                # API client
```

### Development Workflow

```bash
# Enter dev environment (installs all deps via Nix)
nix develop

# Start services (Postgres, Redis, etc.)
make services

# Run backend
make dev

# Run frontend (separate terminal)
cd frontend && pnpm dev

# Run tests
make test

# Format code
make format

# Lint
make lint
```

### Observability Stack

```
Logs:     Structured JSON → Promtail → Grafana Loki
Metrics:  Prometheus + Grafana dashboards
Errors:   Sentry (with request_id correlation)
Tracing:  OpenTelemetry (optional)
```

Every log includes: `timestamp`, `level`, `request_id`, `user_id`, `org_id`

### Security Defaults

- HTTPS everywhere (HSTS enabled)
- Secure cookies: `Secure`, `HttpOnly`, `SameSite=Lax`
- Database connections use SSL/TLS
- All input validated via Ecto changesets
- CSP headers prevent XSS
- Rate limiting on auth endpoints
- Audit logging for sensitive operations

### Deployment Model

```
                    ┌─────────────┐
   Internet ───────▶│    Caddy    │ (TLS termination, reverse proxy)
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         ┌────────┐   ┌────────┐   ┌────────┐
         │ App 1  │   │ App 2  │   │ App N  │  (Phoenix instances)
         └────────┘   └────────┘   └────────┘
              │            │            │
              └────────────┼────────────┘
                           ▼
                    ┌─────────────┐
                    │ PostgreSQL  │ (with streaming replica)
                    └─────────────┘
```

- Bare metal servers running NixOS
- systemd manages Phoenix processes
- Caddy handles TLS and load balancing
- No Kubernetes, no Docker in production

### What to Build First

1. **Foundation**: Nix env, Makefile, project structure, database
2. **Admin & Demo**: Admin dashboard, user impersonation, seed data
3. **Auth**: Login, signup, JWT tokens, password reset
4. **Core Features**: Your actual product
5. **Polish**: Error handling, loading states, accessibility

---

## Table of Contents

### PART 1: FOUNDATION (Build First)
1. [Technology Stack](#technology-stack)
2. [Nix Development Environment](#nix-development-environment)
3. [Makefile](#makefile)
4. [Project Structure](#project-structure)
5. [Database](#database) - includes seeding for demos
6. [Multi-tenancy](#multi-tenancy) - org_id on all tables from day 1
7. [Environment Configuration](#environment-configuration)
8. [Secrets Management](#secrets-management)

### PART 2: ADMIN & DEMO (Day 1 Priority)
9. [Admin Dashboard](#admin-dashboard)
10. [User Impersonation](#user-impersonation)

### PART 3: AUTHENTICATION
11. [Authentication & Authorization](#authentication--authorization)
12. [Email Verification](#email-verification)
13. [Password Policies](#password-policies)
14. [Session Management](#session-management)
15. [OAuth / Social Login](#oauth--social-login)
16. [API Keys](#api-keys)
17. [Magic Links / Passwordless Auth](#magic-links--passwordless-auth)
18. [Two-Factor Authentication (2FA)](#two-factor-authentication-2fa)

### PART 4: API & FRONTEND
19. [API Design](#api-design)
20. [Idempotency Keys](#idempotency-keys)
21. [API Documentation](#api-documentation)
22. [Frontend Architecture](#frontend-architecture)
23. [Accessibility (A11Y)](#accessibility-a11y)
24. [Error Pages](#error-pages)

### PART 5: TESTING & QUALITY
25. [Testing Infrastructure](#testing-infrastructure)
26. [Development Workflow](#development-workflow)

### PART 6: SECURITY
27. [CORS](#cors-cross-origin-resource-sharing)
28. [Security](#security)
29. [Rate Limiting](#rate-limiting)

### PART 7: OPERATIONS
30. [Health Checks](#health-checks)
31. [Graceful Shutdown](#graceful-shutdown)
32. [Observability](#observability)
33. [Request ID / Correlation ID](#request-id--correlation-id)
34. [Monitoring & Alerts](#monitoring--alerts)
35. [Circuit Breakers](#circuit-breakers)
36. [Retry Logic & Timeouts](#retry-logic--timeouts)
37. [Deployment](#deployment)
38. [Disaster Recovery](#disaster-recovery)

### PART 8: BACKEND & DATA
39. [Background Jobs](#background-jobs)
40. [Email](#email)
41. [Webhooks](#webhooks)
42. [File Uploads & Storage](#file-uploads--storage)
43. [Search](#search)

### PART 9: BUSINESS FEATURES
44. [Billing & Subscriptions](#billing--subscriptions)
45. [Team Invitations](#team-invitations)
46. [Onboarding Flow](#onboarding-flow)
47. [Feature Flags](#feature-flags)
48. [Audit Logging](#audit-logging)
49. [In-App Notifications](#in-app-notifications)
50. [Product Analytics](#product-analytics)

### PART 10: COMPLIANCE
51. [Internationalization](#internationalization)
52. [Data Export & GDPR](#data-export--gdpr)
53. [Terms of Service Tracking](#terms-of-service-tracking)

### PART 11: PERFORMANCE & POLISH (Post-MVP)
54. [Caching](#caching)
55. [CDN Configuration](#cdn-configuration)
56. [Nice to Have](#nice-to-have)

---

# TECHNOLOGY STACK

## Backend

| Component | Technology | Purpose |
|-----------|------------|---------|
| Language | Elixir | Functional, concurrent, fault-tolerant |
| Framework | Phoenix | Web framework, scalable real-time features |
| HTTP Adapter | Bandit | Modern, pure Elixir (replaces Cowboy) |
| Database | PostgreSQL | Primary data store |
| Cache | Redis | Caching, sessions, rate limiting |
| Search | Meilisearch | Full-text search |
| Background Jobs | Oban | Persistent job queue with PostgreSQL |
| JWT | Joken | Simple JWT encode/decode |
| Password Hashing | Argon2 | argon2_elixir, current best practice |
| JSON | Jason | Fast JSON serialization |
| CORS | cors_plug | CORS middleware |
| Email | Swoosh | Email composition and delivery |
| Static Analysis | Dialyxir | Type checking and specs |
| Code Coverage | ExCoveralls | Test coverage reporting |

## Frontend

| Component | Technology | Purpose |
|-----------|------------|---------|
| Language | TypeScript | Type-safe JavaScript |
| Framework | React 18+ | UI library |
| Build Tool | Vite | Fast bundler, HMR, ESM-native |
| Package Manager | pnpm | Fast, disk-efficient, workspace support |
| Routing | React Router | Client-side routing |
| Server State | TanStack Query | Data fetching (no caching - see below) |
| Client State | Zustand | Lightweight state management |
| Forms | React Hook Form + Zod | Form handling with validation |
| Styling | Tailwind CSS | Utility-first CSS |
| Components | shadcn/ui + Radix UI | Accessible components (MIT license) |
| Toasts | Sonner | Toast notifications |
| Offline Storage | Dexie.js | IndexedDB wrapper for offline-first |
| PWA | Workbox | Service worker tooling |

## Infrastructure

| Component | Technology | Purpose |
|-----------|------------|---------|
| Dev Environment | Nix Flakes | Reproducible, declarative setup |
| Services (dev) | devenv / process-compose | Run Postgres, Redis locally via Nix |
| Production | Bare metal + NixOS | Full control, reproducible servers |
| CI/CD | GitHub Actions | Automated testing and deployment |
| Monitoring | Prometheus + Grafana | Metrics and dashboards |
| Logging | structured JSON logs | Centralized logging |
| Error Tracking | Sentry | Error monitoring |

---

# NIX DEVELOPMENT ENVIRONMENT

Use Nix flakes for reproducible development environments. No Docker required—Nix provides better reproducibility without container overhead.

## Overview

Create a `flake.nix` that defines inputs (nixpkgs, flake-utils, devenv), pins nixpkgs to a specific commit for reproducibility, and creates a devShell with all dependencies. Include Elixir and Erlang (matching production versions), Node.js with pnpm, PostgreSQL (server + client), Redis, Meilisearch, Git, GNU Make, and direnv.

## Local Services

Use devenv or process-compose to run all services locally without Docker. Configure PostgreSQL, Redis, and Meilisearch with project-local data directories (under `.devenv/state/`). Add Mailpit for email testing. All services should start with a single command (`devenv up` or `make services`).

In devenv.nix, enable services like `services.postgres.enable = true` and `services.redis.enable = true`. Configure non-default ports to avoid conflicts with system services.

## Shell Hooks

Set MIX_HOME and HEX_HOME to project-local paths so Hex packages stay within the project. Configure ERL_AFLAGS for better shell experience. Set DATABASE_URL and REDIS_URL environment variables pointing to local services. Optionally print a welcome message showing available commands.

## .envrc

Keep it minimal: use the `use flake` directive and optionally load a `.env` file. nix-direnv handles caching automatically.

## Why Nix Over Docker

Nix provides faster startup (no container overhead), native performance, and easier debugging without container boundaries. The same Nix config works for both dev and CI. No Docker Desktop license concerns, and networking is simpler (localhost just works).

---

# MAKEFILE

Use GNU Make as a standard interface for common development tasks. Use `.PHONY` targets (task runner style, not file builder) with verb-based naming like `make test`, `make build`, `make deploy`.

## Essential Targets

Create these core targets: `setup` (deps, db create, seeds), `dev` (start development server), `test` (run all tests), `lint` (credo, eslint, format check), `format` (auto-format all code), `build` (production build), and `clean` (remove build artifacts).

## Database Targets

Use dotted naming for database tasks: `db.create`, `db.migrate`, `db.rollback`, `db.reset` (drop, create, migrate, seed), and `db.seed`.

## Services Targets

For Nix-based local services: `services` (start Postgres, Redis, etc.), `services.stop`, `services.logs`, and `services.status`.

## CI/Deployment Targets

Include `ci` (run full CI suite locally), `release` (build release), `deploy.staging`, and `deploy.prod` (with confirmation prompt).

## Help Target

Make `help` the default target. Use `##` comments after target names for auto-generated help text.

## Best Practices

Use variables for repeated values. Chain related commands with `&&`. Add confirmation prompts for destructive actions. Support environment overrides like `make target ENV=production`.

---

# PROJECT STRUCTURE

## Backend Structure

```
lib/
  my_app/                    # Business logic (contexts)
    accounts/                # User management
    billing/                 # Subscriptions, payments
    projects/                # Domain entities
    workers/                 # Oban workers
  my_app_web/               # Web layer
    controllers/            # API controllers
    plugs/                  # Middleware
    router.ex               # Route definitions
    endpoint.ex             # HTTP endpoint config
```

Keep controllers thin—they delegate to contexts for business logic.

## Frontend Structure

```
src/
  api/                      # API client and queries
  components/               # Reusable UI components
    ui/                     # Base components (Button, Input)
    features/               # Feature-specific components
  hooks/                    # Custom React hooks
  pages/                    # Route components
  stores/                   # Zustand stores
  utils/                    # Helper functions
  types/                    # TypeScript types
```

Use barrel exports (index.ts) for cleaner imports. Colocate components with their tests.

## Umbrella Projects

For larger applications, use an umbrella structure:

```
apps/
  my_app/                      # Core business logic
    lib/my_app/
      accounts/
      billing/
      projects/
  my_app_web/                  # Phoenix web layer
    lib/my_app_web/
      controllers/
      channels/
      router.ex
  my_app_worker/               # Background job processing
```

Use umbrella when you have a large team with clear domain boundaries, need independent compilation/testing per app, or want enforced dependency direction (web depends on core, not vice versa). Skip umbrella for small teams or early-stage projects.

## Single Shared Repository Pattern

**Critical**: All umbrella apps share ONE Ecto repository. Never create app-specific repos.

```
apps/
  my_app_repo/                   # Shared repository (ALL schemas here)
    lib/my_app_repo/
      repo.ex                    # Single Ecto.Repo
      schemas/                   # ALL schemas live here
        user.ex
        organization.ex
        product.ex
    priv/repo/migrations/        # ALL migrations here
  my_app_auth/                   # Auth domain (uses MyAppRepo.Repo)
  my_app_web/                    # Web layer (uses MyAppRepo.Repo)
  my_app_billing/                # Billing domain (uses MyAppRepo.Repo)
```

This provides simpler transaction management across domains, one migration path with no ordering conflicts, clear schema ownership, and avoids circular dependencies. Domain apps depend on the repo app and contain only business logic, not schemas.

## UUIDs for Primary Keys

Use UUIDs (UUID v4) instead of auto-incrementing integers. Benefits: prevents ID enumeration attacks (can't guess `/users/2` from `/users/1`), allows client-side ID generation before insert, enables merging data from multiple databases without conflicts, and hides business information (user count, order volume).

In Ecto, use `:binary_id` as the primary key type. Set `@primary_key {:id, :binary_id, autogenerate: true}` and `@foreign_key_type :binary_id` in schemas. Create a base schema module with these defaults.

Downsides: larger storage (16 bytes vs 4-8), potentially slower index performance due to random distribution, less readable in logs. Consider UUIDv7 if time-sortable IDs matter.

---

# AUTHENTICATION & AUTHORIZATION

Use JWT tokens with short-lived access tokens (15 minutes) and long-lived refresh tokens (30 days). Store tokens in httpOnly cookies for XSS protection. Use bcrypt for password hashing. Implement policy-based authorization for flexible, testable rules.

## Backend

Use Guardian (or Joken) for JWT generation and verification. Create an Auth plug to extract and validate tokens from requests. Store refresh tokens in the database with device fingerprint for "logout from all devices" support. Implement a token refresh endpoint. Create policy modules for authorization (e.g., `ProjectPolicy`) with role-based access (admin, member, viewer). Log all authentication events (login, logout, failed attempts) and implement account lockout after N failed attempts.

## Frontend

Tokens are set via httpOnly cookies by the backend—the frontend never sees the actual token. Implement a fetch interceptor to handle token refresh on 401 responses. Create an AuthContext for current user state and a ProtectedRoute component for guarded routes. Handle 401 responses globally by redirecting to login. Clear auth state on logout.

## Security

All cookies must be secure (HTTPS only), httpOnly (no JavaScript access), and SameSite=Lax. Implement CSRF protection if using cookie-based auth. Rate limit login attempts. Invalidate all sessions on password change.

---

# CORS (Cross-Origin Resource Sharing)

Use `cors_plug` to allow your frontend to communicate with the API. Never use `*` for origins in production—always use an explicit allowlist.

## Configuration

Allowlist specific origins (frontend domains) and allow credentials for cookie-based auth. Specify allowed methods (GET, POST, PUT, PATCH, DELETE, OPTIONS) and headers (Content-Type, Authorization, X-Requested-With). Set preflight cache max age to 86400 seconds.

Load origins from environment variables: development allows `http://localhost:3000` and `http://localhost:5173`, staging allows `https://staging.yourapp.com`, production allows `https://yourapp.com` and `https://www.yourapp.com`.

Review CORS config when adding subdomains. Test that preflight requests work correctly.

---

# SECURITY

Defense in depth with HTTP headers, secure cookies, input validation, and protection against common attacks.

## HTTP Security Headers

Configure these headers via a Phoenix Plug or at the reverse proxy level (Caddy). Essential headers include HSTS (`Strict-Transport-Security: max-age=31536000; includeSubDomains`), `X-Content-Type-Options: nosniff`, and `X-Frame-Options: DENY`.

For Content Security Policy (CSP), start strict with `default-src 'self'` and `script-src 'self'`. Tailwind requires `style-src 'self' 'unsafe-inline'`. Use `frame-ancestors 'none'` to prevent clickjacking. Disable unused browser features with Permissions-Policy: `camera=(), microphone=(), geolocation=()`.

Test your headers at securityheaders.com.

## Cookie Security

All session cookies must have `Secure` (HTTPS only), `HttpOnly` (no JavaScript access), and `SameSite=Lax` (CSRF protection). Phoenix configures this in `config/config.exs` under `session_options`. Always use signed and encrypted cookies (Phoenix default). Set appropriate expiration with `max_age`.

## Database Connection Security

Production database connections must use SSL/TLS with `verify-full` mode, which verifies both the certificate and hostname. Configure this in `runtime.exs` with `ssl: true` and appropriate `ssl_opts`. Store CA certificates securely and rotate before expiration.

## Input Validation

Validate at three layers: frontend (Zod schemas for user feedback), API (Ecto changesets for business rules), and database (constraints for integrity). Never trust client input.

Use Ecto changesets for all data mutations with `validate_required`, `validate_format`, `validate_length`, and `unique_constraint`. Sanitize HTML input with HtmlSanitizeEx to prevent stored XSS. Limit string lengths to prevent DoS.

## SQL Injection Prevention

Ecto prevents SQL injection by default through parameterized queries. The `^` pin operator safely escapes values. Never interpolate user input into SQL strings or fragments. If raw SQL is needed, use `Repo.query/3` with parameter placeholders (`$1`, `$2`).

## XSS Prevention

React escapes output by default when rendering with `{userInput}`. Never use `dangerouslySetInnerHTML` with user input. Validate URLs before rendering in `href` attributes to prevent `javascript:` protocol attacks. If rich text is required, sanitize with DOMPurify on the frontend. CSP headers provide defense in depth.

## CSRF Protection

For SPAs using JWT authentication, CSRF is less of a concern since tokens aren't sent automatically like cookies. If using cookie-based sessions, implement double-submit cookies: return a CSRF token in a response header, store it in JavaScript, and send it back on mutations. Verify the `Origin` header matches your domain.

---

# OAUTH / SOCIAL LOGIN

Use `ueberauth` with provider strategies (`ueberauth_google`, `ueberauth_github`) for OAuth authentication. Store OAuth credentials in environment variables and configure callback URLs in each provider's developer console.

Create a `user_identities` table to link OAuth accounts to users with fields: `user_id`, `provider`, `provider_uid`, `provider_email`, and `provider_data` (jsonb). Add a unique index on `(provider, provider_uid)`.

The flow: user clicks social login → redirect to provider → provider returns with auth code → exchange for tokens and fetch user info → find or create user by email → link identity → create session. If the email exists, link to existing account; if new, create account. Allow multiple providers per user and show connected accounts in settings.

Always validate the email is verified from the provider. Don't blindly trust provider emails. Log all OAuth events.

---

# API KEYS

API keys use a prefixed format (`sk_live_xxx`, `sk_test_xxx`) for easy identification. Generate 32 random bytes, base62 encode, add prefix. Store only the SHA-256 hash in the database—never the plaintext. Show the full key once on creation with a copy button and warning.

The `api_keys` table needs: `user_id`, `name`, `key_prefix` (first 8 chars for UI display), `key_hash`, `scopes` (array), `last_used_at`, `expires_at`, and `revoked_at`.

Authentication: accept key in `Authorization: Bearer` header, hash incoming key, lookup by hash, check not revoked/expired, validate scopes, update `last_used_at`. Define granular scopes like `read:users`, `write:users`. Rate limit key creation and log all usage.

---

# MAGIC LINKS

Magic links allow passwordless sign-in via email. Generate a secure random token (32 bytes), sign it with `Phoenix.Token` including user_id and timestamp, store the hash in a `magic_links` table, and email the plaintext link. Tokens expire in 15 minutes and are single-use.

The table needs: `user_id`, `token_hash`, `expires_at`, `used_at`, `ip_address`, and `user_agent`.

On verification: extract token from URL, verify signature and expiration, check not already used, mark as used, create session. Rate limit requests to 3 per hour per email. Log all magic link activity.

---

# TWO-FACTOR AUTHENTICATION (2FA)

Use TOTP (RFC 6238) with `nimble_totp` for code generation/verification and `eqrcode` for QR codes. Works with Google Authenticator, Authy, and 1Password.

Add to users table: `totp_secret` (encrypted), `totp_enabled`, `totp_enabled_at`, `backup_codes` (array of hashed codes), `backup_codes_generated_at`.

Setup flow: generate secret → show QR code → user scans with authenticator app → user enters code to verify → enable 2FA and generate 10 backup codes (8 chars each, format `xxxx-xxxx`). Backup codes are single-use and hashed.

Login flow: after password verification, if 2FA enabled, show challenge screen → user enters 6-digit code or backup code → verify and create session.

Encrypt TOTP secrets at rest, hash backup codes, rate limit verification attempts, require password to disable 2FA, allow 30-second time window tolerance.

---

# SESSION MANAGEMENT

Store sessions in a database table for persistence and queryability. The `sessions` table needs: `user_id`, `token_hash` (SHA-256 of JWT), `device_name`, `device_type`, `browser`, `os`, `ip_address`, `location` (optional geo-lookup), `last_active_at`, `expires_at`, and `revoked_at`.

Parse user agent strings with the `ua_parser` library to show friendly device names. Update `last_active_at` on each authenticated request. Sessions expire after 30 days.

Show users their active sessions in account settings with device icons, browser info, location, and last active time. Highlight the current session. Provide "log out" per session and "log out all other devices" buttons. Auto-revoke all sessions on password change.

---

# PASSWORD POLICIES

Minimum 12 characters (NIST recommendation), maximum 128 (prevent DoS). Check passwords against HaveIBeenPwned API using k-anonymity (send first 5 chars of SHA-1 hash, compare against returned list). Keep history of last 5 password hashes to prevent reuse.

Store `password_hash` (Argon2), `password_changed_at`, and `password_history` (array) on the user. On password change: require current password, validate against all rules, add old hash to history, invalidate other sessions, send email notification.

Show a password strength indicator during input. Rate limit reset requests, expire reset tokens after 1 hour, make them single-use. Never log passwords.

---

# EMAIL VERIFICATION

On signup, generate a secure token signed with `Phoenix.Token`, store its hash, and email a verification link. Tokens expire in 24 hours and are single-use.

Add to users: `email_verified_at`, `email_verification_token_hash`, `email_verification_sent_at`.

Allow unverified users limited access (view-only), but block sensitive actions like payments and invitations. Show a persistent banner prompting verification. Rate limit resend to 3 per hour with 1-minute cooldown between sends.

When a user changes their email, require re-verification. Optionally auto-delete unverified accounts after 7 days.

---

# API DESIGN

Use REST with URL-based versioning (`/api/v1/`). All responses use consistent JSON structure: `{ "data": ... }` for success with optional `"meta"` for pagination, and `{ "error": { "code": "...", "message": "...", "details": {...} } }` for errors following RFC 7807.

Use a FallbackController for consistent error handling. Map Ecto changeset errors to the API format with field-level details. Use consistent error codes like `validation_error`, `not_found`, `unauthorized`.

## Pagination

Use cursor-based pagination for list endpoints—it's stable even when data changes between requests, unlike offset pagination which can skip or duplicate items. Encode cursors as Base64 JSON containing `id` and `inserted_at`. Sort by a unique column (or composite) for determinism.

Response format: `{ "data": [...], "meta": { "has_more": true, "next_cursor": "xxx", "page_size": 20 } }`. Accept `?cursor=xxx&limit=20` params. Default page size 20, max 100. Return empty array (not error) for invalid cursors.

## Request Metadata

Create a Plug that captures `client_ip` (checking `X-Forwarded-For` for proxied requests), `user_agent`, and `request_id` (from header or generate UUID). Store these on sessions and include in all logs for debugging and audit trails.

---

# IDEMPOTENCY KEYS

Support Stripe-style idempotency via the `Idempotency-Key` header for safe retries on mutation endpoints. Clients generate a UUID per operation and reuse it for retries.

The `idempotency_keys` table stores: `key`, `user_id`, `request_path`, `request_hash`, `response_status`, `response_body` (jsonb), `locked_at`, `completed_at`, and `expires_at` (24-hour TTL).

Flow: extract header → check if key exists → if completed, return cached response with `Idempotent-Replayed: true` header → if locked, return 409 → if new, acquire lock, process, cache response. Return 422 if same key used with different request params. Schedule Oban job to clean up expired keys hourly.

Apply to POST, PUT, PATCH, DELETE endpoints. Skip for GET (naturally idempotent).

---

# WEBHOOKS

## Outgoing Webhooks

Let users register webhook endpoints to receive events from your system. The `webhook_endpoints` table stores: `user_id`, `url` (HTTPS required), `secret` (auto-generated for signing), `description`, `events` (array of subscribed types or `["*"]`), and `enabled`.

The `webhook_deliveries` table tracks each delivery: `endpoint_id`, `event_type`, `event_id` (idempotency), `payload`, `status`, `attempts`, timestamps, and `response_status`/`response_body` for debugging.

Sign payloads with HMAC-SHA256 in Stripe format: `X-Webhook-Signature: t=timestamp,v1=signature`. Include headers for event type and ID. Use exponential backoff for retries (1min, 5min, 30min, 2hr, 24hr), max 5 attempts. Process deliveries via Oban workers with 30-second timeout.

Provide documentation for signature verification with code samples and timestamp tolerance (5 minutes) to prevent replay attacks.

## Incoming Webhooks

Receive webhooks from external services (Stripe, GitHub, etc.). Store in `webhook_events` table: `source`, `event_type`, `event_id`, `payload`, `status`, `processed_at`, `error_message`. Add unique index on `(source, event_id)` for idempotency.

Always verify signatures before processing. For Stripe, use `Stripe.Webhook.construct_event/3` with raw body. For GitHub, verify `X-Hub-Signature-256` with HMAC-SHA256 constant-time comparison.

Return 200 immediately to acknowledge receipt, then enqueue Oban job for async processing. Check idempotency (event not already processed) before handling. Store full payloads for debugging and provide admin view of events with manual retry.

---

# FRONTEND ARCHITECTURE

Use TanStack Query for server state (API calls) with caching disabled (`staleTime: 0`, `cacheTime: 0`)—always fetch fresh data. Use Zustand only for UI state (modals, sidebar, theme). Use Dexie.js (IndexedDB wrapper) for offline data persistence.

## Offline-First / PWA

Define IndexedDB schema with Dexie for offline-capable entities. Implement a sync queue for pending mutations with conflict resolution (last-write-wins or merge). Use Workbox for service workers: cache static assets, use network-first for API calls, queue failed requests for retry when connection restores. Show offline indicator and sync status in UI.

## State & Routing

Create a typed API client with error handling and token refresh in interceptor. Define routes in central config with lazy loading for route components. Create layout components (AuthLayout, DashboardLayout) with route guards for protected pages.

## Forms & Components

Use React Hook Form + Zod for forms with type-safe validation matching API schemas. Show server errors inline with fields. Build design system with shadcn/ui + Radix + Tailwind. Ensure keyboard navigation and screen reader support.

## Error Boundaries

Use `react-error-boundary` to catch JavaScript errors. Create root-level boundary for entire app and feature-level boundaries for isolation. Display user-friendly error with "Try Again" button and "Go Home" link. Log errors to Sentry with component stack and request ID for support.

## Loading States

Create skeleton components matching content layout with CSS shimmer animation. Show skeletons immediately to avoid layout shift. Use TanStack Query's `onMutate` for optimistic updates—update UI immediately, rollback on error with toast. Disable buttons while submitting with spinner to prevent double-clicks.

## Performance Budgets

Target: main JS < 200 KB gzipped, main CSS < 50 KB, per-route chunk < 100 KB. Lazy load routes and heavy components. Run Lighthouse CI on every PR, block merge if scores drop. Target Core Web Vitals: LCP < 2.5s, FID < 100ms, CLS < 0.1.

---

# BACKGROUND JOBS

Use Oban for PostgreSQL-backed job processing. Configure separate queues by priority: `default`, `mailers`, `webhooks`, `maintenance`. Each queue has its own concurrency limit. Add Oban to the supervision tree and configure the pruner to clean completed jobs.

Create workers for async tasks (emails, webhooks, cleanup). Each worker implements `perform/1` with proper error handling. Use `unique` constraints to prevent duplicate jobs. Set reasonable `max_attempts` per worker with exponential backoff for retries.

Use `Oban.insert/1` for immediate jobs, `scheduled_at` for delayed execution, and the cron plugin for recurring jobs. Monitor job success/failure rates and alert on queue backlogs.

---

# EMAIL

Use Swoosh with a provider adapter (Postmark, SendGrid, or SES) for transactional emails. Create a base email module with default `from` and `reply-to`. Use sandbox adapter for tests and implement a preview route for development.

Build templates with MJML for responsive layouts that work across email clients. Create a base layout with header, footer, and styles. Use inline styles for compatibility. Keep emails under 102KB to avoid Gmail clipping. Always include a plain text version.

Send emails asynchronously via Oban EmailWorker to avoid blocking requests. Handle delivery failures with retries. Log email events and implement unsubscribe handling. Required email types: password reset, email verification, welcome, invoice/receipt, team invitation, activity notifications.

---

# RATE LIMITING

Use Hammer with Redis backend for distributed rate limiting with token bucket algorithm. Scope limits per-user for authenticated requests and per-IP for anonymous requests.

Limits by endpoint type: login/register (5/minute), password reset (3/hour), authenticated API (1000/minute), unauthenticated API (100/minute), file upload (10/minute).

Create a RateLimit plug that returns 429 with `Retry-After` header when exceeded. Include `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset` headers in responses. Log violations and allow bypass for internal services.

---

# FILE UPLOADS & STORAGE

Use S3-compatible storage with presigned URLs for direct client uploads (avoids server load). Flow: client requests presigned URL → uploads directly to S3 → notifies server of completion → server validates and enqueues processing job.

Validate files before accepting: check size limits, validate MIME type by checking magic bytes (not extension), sanitize filenames. For sensitive apps, scan with ClamAV.

Organize storage by tenant: `uploads/{tenant_id}/{uuid}.{ext}`. Store metadata (size, type, original name) in database. For images, generate thumbnails and responsive variants in background jobs. Strip EXIF data for privacy and convert to WebP.

---

# SEARCH

Use Meilisearch for fast, typo-tolerant full-text search. Index data asynchronously via Oban workers on create/update/delete events. Define searchable, filterable, and sortable attributes per index.

Create a Search context module with functions for indexing and searching. Add search endpoints to the API. On the frontend, implement search with debouncing (300ms) and show autocomplete suggestions. Support faceted search (filter by category, status), highlighted results, and pagination.

Handle bulk reindexing for initial data load or schema changes.

---

# BILLING & SUBSCRIPTIONS

Use Stripe for subscription billing. Create a Stripe Customer when users sign up. Implement checkout session creation for plan selection. Store subscription status locally (synced via webhooks).

Handle webhooks idempotently: `checkout.session.completed`, `customer.subscription.created`, `customer.subscription.updated`, `customer.subscription.deleted`, `invoice.paid`, `invoice.payment_failed`. Update local subscription state on each event.

Build subscription management UI: plan selection, upgrade/downgrade, cancellation with feedback. Handle failed payments with dunning (retry). Link to Stripe Customer Portal for payment method and invoice management.

---

# FEATURE FLAGS

Use FunWithFlags with Redis backend for feature flags. Support multiple targeting types: boolean (on/off globally), actor-based (specific users), group-based (by plan or role), and percentage rollout for gradual releases.

Check flags in controllers and contexts. Pass flag state to frontend for conditional UI. Use for gradual rollouts and A/B testing. Create an admin UI for flag management.

---

# AUDIT LOGGING

Log all create, update, delete operations plus auth events to a separate `audit_logs` table. Schema: `action`, `resource_type`, `resource_id`, `actor_id`, `actor_type` ("user", "system", "api_key"), `changes` (jsonb with before/after), `metadata` (IP, user agent, request_id), `inserted_at`.

Implement an audit logging helper called from context functions. Consider async logging via Oban for high-throughput. Retain logs for at least 90 days. Support filtering by resource, actor, and date range. Provide export functionality for compliance.

---

# INTERNATIONALIZATION

Use Gettext on the backend—extract strings to PO files and translate error messages. Implement a locale plug that reads from `Accept-Language` header or user preference stored in database.

Use react-i18next on the frontend with JSON translation files. Handle pluralization and format dates/numbers per locale. Implement a language switcher in the UI.

---

# DATA EXPORT & GDPR

For data export, collect all user data (including related records) and generate a ZIP with JSON and CSV files. Process in a background job (large exports take time), upload to S3 with an expiring download link, and email the user when ready.

For account deletion, soft delete or anonymize depending on requirements. Cancel active subscriptions, handle related data (cascade or nullify), send confirmation email. Consider a 30-day recovery window.

Document data retention policies, track privacy policy acceptance with timestamps, and log consent for marketing communications.

---

# API DOCUMENTATION

Use OpenAPI 3.0 generated from code annotations (single source of truth). Document all endpoints with request/response schemas, authentication requirements, example requests, and error codes. Serve interactive documentation via Redoc or Swagger UI. Version documentation with the API.

---

# DATABASE

## Connection Pooling

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Pooler | PgBouncer (production) | Connection multiplexing |
| Pool size | CPU cores * 2 | Balance connections vs. resources |
| Mode | Transaction | Best for web workloads |
| Checkout timeout | 5000ms | Fail fast, don't queue |

### Pool Sizing Guidelines

| Environment | Pool Size | Rationale |
|-------------|-----------|-----------|
| Dev | 10 | Minimal local resources |
| Test | 10 | Parallel tests use sandbox |
| Staging | 20 | Match expected load |
| Production | (CPU cores * 2) + 1 | PostgreSQL recommendation |

**Warning**: Total connections across all instances must not exceed `max_connections` in PostgreSQL (default: 100).

### Implementation

Configure Ecto pool size in `runtime.exs` based on environment. Set `queue_target: 50` and `queue_interval: 1000` for backpressure. Use PgBouncer for production connection multiplexing. Monitor `Ecto.Repo` telemetry for checkout times. Handle `DBConnection.ConnectionError` gracefully. Use separate pools for long-running queries (reports, exports). Alert when pool checkout times exceed threshold.

### Configuration Example

```elixir
# config/runtime.exs
config :my_app, MyApp.Repo,
  pool_size: String.to_integer(System.get_env("POOL_SIZE") || "10"),
  queue_target: 50,
  queue_interval: 1000
```

## N+1 Query Prevention

Avoid the N+1 query problem with proper preloading strategies.

### The Problem

```elixir
# BAD: N+1 queries (1 query for users, N queries for organizations)
users = Repo.all(User)
Enum.map(users, fn user -> user.organization.name end)

# GOOD: 2 queries total
users = Repo.all(User) |> Repo.preload(:organization)
Enum.map(users, fn user -> user.organization.name end)
```

### Preload Strategies

| Strategy | Use When |
|----------|----------|
| `Repo.preload/2` | After fetching, separate query |
| `from ... preload` | In query, JOIN-based |
| `Dataloader` | GraphQL, batched loading |

### Implementation

Always preload associations before accessing them. Use `Repo.preload/2` for simple cases and query-based preload with `join` for filtering. Never access associations in `Enum.map` without preload. Add EctoDevLogger in dev to spot N+1 queries. Consider Dataloader for complex GraphQL resolvers.

### Common Patterns

```elixir
# Preload in query (single query with JOIN)
from u in User,
  join: o in assoc(u, :organization),
  where: o.active == true,
  preload: [organization: o]

# Preload after query (separate queries, simpler)
User
|> Repo.all()
|> Repo.preload([:organization, :roles])

# Nested preloads
Repo.preload(users, [organization: :subscription, roles: :permissions])
```

### Detection

- [ ] Use `ecto_dev_logger` in development (logs all queries)
- [ ] Review query counts in test logs
- [ ] Monitor query count per request in production
- [ ] Set up alerts for endpoints exceeding query threshold

## Indexing

Proper indexes are critical for query performance. Understand when and how to use them.

### Index Types

| Type | Use Case | Example |
|------|----------|---------|
| B-tree (default) | Equality, range, sorting | `WHERE status = 'active'` |
| Hash | Equality only (rarely used) | Exact match lookups |
| GIN | Arrays, JSONB, full-text | `WHERE tags @> ARRAY['foo']` |
| GiST | Geometric, full-text, ranges | PostGIS, tsquery |
| Partial | Subset of rows | `WHERE deleted_at IS NULL` |
| Composite | Multi-column queries | `WHERE org_id = ? AND status = ?` |

### When to Add Indexes

- [ ] Foreign keys (always - prevents full table scans on JOINs)
- [ ] Columns in WHERE clauses
- [ ] Columns in ORDER BY clauses
- [ ] Columns in JOIN conditions
- [ ] Unique constraints (automatically creates index)

### When NOT to Index

- Small tables (< 1000 rows) - sequential scan is faster
- Columns with low cardinality (e.g., boolean with 50/50 split)
- Columns rarely used in queries
- Tables with heavy write load and few reads

### Composite Index Column Order

Column order matters. Put the most selective (highest cardinality) columns first.

```sql
-- Query: WHERE org_id = ? AND status = ? ORDER BY created_at DESC
-- Index: (org_id, status, created_at DESC)

CREATE INDEX idx_orders_org_status_created
ON orders (org_id, status, created_at DESC);
```

**Rule**: Index can be used for leftmost prefix. An index on `(a, b, c)` works for:
- `WHERE a = ?`
- `WHERE a = ? AND b = ?`
- `WHERE a = ? AND b = ? AND c = ?`

But NOT for `WHERE b = ?` alone.

### Partial Indexes

Index only rows that matter. Smaller index = faster queries + less storage.

```elixir
# Only index active users (most queries filter by active)
create index(:users, [:email], where: "deleted_at IS NULL")

# Only index pending jobs
create index(:jobs, [:scheduled_at], where: "status = 'pending'")
```

### JSONB Indexes (GIN)

```elixir
# Index entire JSONB column
create index(:events, [:metadata], using: :gin)

# Index specific JSONB path
execute """
CREATE INDEX idx_events_metadata_type
ON events ((metadata->>'type'))
"""
```

### Migration Examples

```elixir
defmodule MyApp.Repo.Migrations.AddIndexes do
  use Ecto.Migration

  # Regular index
  def change do
    create index(:orders, [:user_id])
    create index(:orders, [:org_id, :status])
  end
end

defmodule MyApp.Repo.Migrations.AddConcurrentIndex do
  use Ecto.Migration

  # Concurrent index (no table lock, use for large tables in production)
  @disable_ddl_transaction true
  @disable_migration_lock true

  def change do
    create index(:orders, [:created_at], concurrently: true)
  end
end
```

### Monitoring Index Usage

```sql
-- Find unused indexes
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexname NOT LIKE '%_pkey'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Find missing indexes (sequential scans on large tables)
SELECT relname, seq_scan, seq_tup_read,
       idx_scan, idx_tup_fetch
FROM pg_stat_user_tables
WHERE seq_scan > 0
ORDER BY seq_tup_read DESC;
```

### Implementation Checklist

- [ ] Add indexes for all foreign keys
- [ ] Add composite indexes for common query patterns
- [ ] Use partial indexes to reduce index size
- [ ] Use GIN indexes for JSONB columns queried with `@>` or `->>`
- [ ] Create indexes concurrently on production tables
- [ ] Monitor slow query logs weekly
- [ ] Review unused indexes quarterly (drop if not needed)
- [ ] Document index rationale in migration comments
- [ ] Test query plans with `EXPLAIN ANALYZE` before/after

## Zero-Downtime Migrations

### Key Principles

- [ ] Never lock tables for extended periods
- [ ] Add columns as nullable first, backfill, then add constraint
- [ ] Create indexes concurrently
- [ ] Avoid renaming columns (add new, migrate, remove old)
- [ ] Deploy code changes before and after schema changes

### Safe Migration Patterns

| Operation | Safe Approach |
|-----------|---------------|
| Add column | Add as nullable, backfill, add NOT NULL |
| Remove column | Stop reading, deploy, remove in next migration |
| Add index | CREATE INDEX CONCURRENTLY |
| Change column type | Add new column, migrate data, swap |
| Rename column | Add new, copy data, update code, remove old |

### Implementation Checklist

- [ ] Use `Ecto.Migration.execute/2` for concurrent index creation
- [ ] Split risky migrations into multiple deployments
- [ ] Test migrations against production-like data
- [ ] Have rollback plan for each migration

## Soft Deletes

Keep deleted records for recovery and audit trails instead of hard deleting.

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Column | `deleted_at` timestamp | NULL = active, timestamp = deleted |
| Queries | Default scope excludes deleted | Transparent to most code |
| Restoration | Set `deleted_at` to NULL | Simple recovery |

### Implementation Checklist

- [ ] Add `deleted_at` (datetime, nullable) to tables needing soft delete
- [ ] Create base query macro that filters `WHERE deleted_at IS NULL`
- [ ] Use `deleted_at = NOW()` instead of `DELETE`
- [ ] Create `with_deleted/1` query helper for admin views
- [ ] Create `only_deleted/1` query helper for trash views
- [ ] Add unique indexes with `WHERE deleted_at IS NULL` partial index
- [ ] Handle cascading soft deletes for related records

### When to Use

- [ ] User accounts (GDPR recovery period)
- [ ] Important business records (orders, invoices)
- [ ] Content that might be restored (posts, comments)
- [ ] Audit-sensitive data

### When NOT to Use

- [ ] High-volume ephemeral data (logs, sessions)
- [ ] Data that must be truly deleted (PII after retention period)
- [ ] Join tables / associations (usually hard delete)

## Data Seeding

Populate database with initial and test data.

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Location | `priv/repo/seeds.exs` | Mix convention |
| Test data | Factories (ex_machina) | Flexible, composable |
| Idempotent | Check before insert | Safe to run multiple times |

### Implementation Checklist

### Production Seeds (`seeds.exs`)

- [ ] Create default admin user (if applicable)
- [ ] Insert static reference data (countries, categories, plans)
- [ ] Make idempotent (use `insert_or_update` or check existence)
- [ ] Run via `mix ecto.setup` or release migration

### Development Seeds

- [ ] Create realistic sample data for development
- [ ] Include various user roles and states
- [ ] Generate enough data to test pagination
- [ ] Include edge cases (long names, special characters)

### Factories (ex_machina)

- [ ] Define factory for each schema
- [ ] Use sequences for unique fields (`sequence(:email, &"user#{&1}@test.com")`)
- [ ] Create traits for common variations (`:admin`, `:with_subscription`)
- [ ] Build associations lazily

### Best Practices

- [ ] Never seed production with test data
- [ ] Use environment checks (`if Mix.env() == :dev`)
- [ ] Document seed data in README
- [ ] Keep seeds fast (batch inserts)

## Pagination

Paginate list endpoints for performance and usability.

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Default style | Cursor-based | Stable for real-time data |
| Fallback | Offset for simple cases | Easier for static data |
| Default limit | 20 | Balance between UX and performance |
| Max limit | 100 | Prevent abuse |

### Cursor-Based Pagination

Best for: feeds, real-time data, infinite scroll

- [ ] Encode cursor as opaque string (base64 of ID + timestamp)
- [ ] Use indexed columns for cursor (id, inserted_at)
- [ ] Return `next_cursor` in response meta
- [ ] Accept `cursor` and `limit` params
- [ ] Handle cursor decoding errors gracefully

### Offset-Based Pagination

Best for: admin tables, search results, static lists

- [ ] Accept `page` and `per_page` (or `limit`) params
- [ ] Return total count in response meta
- [ ] Calculate `total_pages`
- [ ] Cap offset to prevent deep pagination attacks

### Response Format

```json
{
  "data": [...],
  "meta": {
    "next_cursor": "eyJpZCI6MTIzfQ==",
    "has_more": true,
    "total": 1234,
    "page": 1,
    "per_page": 20
  }
}
```

### Implementation Checklist

- [ ] Create Pagination module with helpers
- [ ] Support both cursor and offset in same codebase
- [ ] Add `limit` validation (1-100)
- [ ] Index columns used for sorting/cursors
- [ ] Return consistent meta object structure
- [ ] Handle empty results gracefully

### Frontend

- [ ] Infinite scroll for cursor pagination
- [ ] Page numbers/prev/next for offset pagination
- [ ] Loading states during pagination
- [ ] "Load more" button alternative to infinite scroll

## Database Transactions (Ecto.Multi)

Compose multiple database operations into atomic transactions.

### Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Tool | Ecto.Multi | Built-in, composable, named steps |
| Rollback | Automatic on failure | All-or-nothing semantics |
| Error handling | Pattern match on result | Clear success/failure paths |

### When to Use Ecto.Multi

- [ ] Creating related records together (user + profile + settings)
- [ ] Operations that must all succeed or all fail
- [ ] When you need to use results from earlier steps
- [ ] Complex workflows with multiple database writes

### Implementation Patterns

#### Basic Multi

- [ ] Create Multi with `Ecto.Multi.new()`
- [ ] Add operations with `insert`, `update`, `delete`
- [ ] Name each step for error identification
- [ ] Run with `Repo.transaction(multi)`

#### Using Previous Results

- [ ] Use `Ecto.Multi.run/3` for dynamic operations
- [ ] Access previous results via `changes` map
- [ ] Return `{:ok, result}` or `{:error, reason}`

#### Conditional Operations

- [ ] Use `Multi.run/3` for conditional logic
- [ ] Check conditions and return early if needed
- [ ] Can skip operations by returning existing data

### Error Handling

- [ ] Pattern match: `{:ok, results}` or `{:error, step, reason, changes}`
- [ ] `step` identifies which operation failed
- [ ] `changes` contains successful operations before failure
- [ ] All changes rolled back automatically

### Best Practices

- [ ] Keep transactions short (avoid external calls inside)
- [ ] Name steps descriptively (`:create_user`, `:send_welcome_email`)
- [ ] Don't put side effects inside transactions (emails, webhooks)
- [ ] Use `Multi.run/3` for complex validations
- [ ] Consider optimistic locking for concurrent updates

### Common Patterns

- [ ] User registration: create user, profile, send verification email (email outside transaction)
- [ ] Order placement: create order, create line items, update inventory
- [ ] Team creation: create team, add creator as admin member

---

# MULTI-TENANCY

Isolate customer data from day 1. Add `org_id` to all transactional tables.

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Strategy | Row-level with org_id | Simpler, sufficient for most SaaS |
| Column name | `org_id` or `organization_id` | Consistent across all tables |
| Identification | JWT claim + header | Extracted from auth token |
| Enforcement | Query scoping | All queries filter by org_id |

## Why Row-Level (Not Schema-Per-Tenant)

| Aspect | Row-Level | Schema-Per-Tenant |
|--------|-----------|-------------------|
| Complexity | Low | High |
| Migrations | Single migration | Per-tenant migrations |
| Connection pooling | Standard | Complex (schema switching) |
| Cross-tenant queries | Easy (admin) | Requires schema hopping |
| Data isolation | Application-enforced | Database-enforced |
| Best for | Most SaaS apps | Strict compliance needs |

## Database Schema

Every transactional table includes:

```sql
CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id UUID NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  name VARCHAR(255) NOT NULL,
  -- ... other fields
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_products_org_id ON products(org_id);
```

## Implementation Checklist

### Organization Schema

- [ ] Create `organizations` table (id, name, slug, settings, created_at)
- [ ] Add unique index on `slug`
- [ ] Create Organization Ecto schema
- [ ] Add `org_id` foreign key to ALL transactional tables
- [ ] Index `org_id` on all tables for query performance

### Query Scoping

```elixir
# Always filter by org_id
def list_products(org_id) do
  from(p in Product, where: p.org_id == ^org_id)
  |> Repo.all()
end

# Never allow queries without org_id on tenant data
def get_product!(org_id, id) do
  Repo.get_by!(Product, id: id, org_id: org_id)
end
```

- [ ] All context functions take `org_id` as first parameter
- [ ] Never query tenant data without org_id filter
- [ ] Create base query helpers that enforce scoping
- [ ] Audit existing queries for missing org_id filters

### Request Context

```elixir
# Plug to set org_id from authenticated user
defmodule MyAppWeb.Plugs.SetOrgContext do
  def call(conn, _opts) do
    case conn.assigns[:current_user] do
      %{organization_id: org_id} ->
        conn
        |> assign(:current_org_id, org_id)
        |> put_private(:org_id, org_id)
      _ ->
        conn
    end
  end
end
```

- [ ] Extract org_id from JWT claims
- [ ] Store in `conn.assigns.current_org_id`
- [ ] Pass to all context functions
- [ ] Log org_id in structured logs

### Data Isolation Verification

- [ ] Write tests that verify cross-tenant data cannot be accessed
- [ ] Test that user from org A cannot see org B's data
- [ ] Add CI check for queries missing org_id filter
- [ ] Regular audit of data access patterns

### Admin/Super-Admin Access

- [ ] Admin endpoints can query across orgs (with audit logging)
- [ ] Use separate admin authentication
- [ ] Log all cross-tenant access
- [ ] Implement org impersonation for support

---

# ENVIRONMENT CONFIGURATION

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Config method | Environment variables | 12-factor app |
| Secrets | External secret manager | Never in code/repo |
| Validation | Runtime config | Fail fast on missing config |

## Implementation Checklist

- [ ] Use `config/runtime.exs` for environment-specific config
- [ ] Validate required environment variables on startup
- [ ] Provide sensible defaults for development
- [ ] Document all environment variables
- [ ] Use `.env.example` as template

---

# SECRETS MANAGEMENT

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Storage | AWS Secrets Manager / Vault / Doppler | Centralized, audited |
| Rotation | Automated where possible | Reduce exposure |
| Access | Service-specific credentials | Least privilege |

## Implementation Checklist

- [ ] Never commit secrets to repository
- [ ] Use secret manager for production
- [ ] Rotate secrets regularly
- [ ] Audit secret access
- [ ] Use different secrets per environment

---

# OBSERVABILITY

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Logging | Structured JSON | Parseable, searchable |
| Log aggregation | Grafana Loki | Efficient, label-based, Grafana native |
| Metrics | Prometheus | Standard, extensive ecosystem |
| Tracing | OpenTelemetry | Vendor-neutral, comprehensive |
| Errors | Sentry | Real-time alerts, context |

## Structured Logging

Output logs as JSON objects for searchability and analysis.

### Why Structured Logging

| Benefit | Description |
|---------|-------------|
| Searchable | Query by field: `user_id="123"` |
| Filterable | Show only errors: `level="error"` |
| Aggregatable | Count errors per endpoint |
| Correlatable | Trace via `request_id` across services |
| PII-safe | Redact specific fields automatically |

### Log Levels

| Level | Use For | Example |
|-------|---------|---------|
| `debug` | Development details | SQL queries, internal state |
| `info` | Normal operations | User login, order created |
| `warn` | Recoverable issues | Rate limit approached, retry succeeded |
| `error` | Failures requiring attention | Payment failed, external API down |

### Elixir Logger Configuration

```elixir
# config/config.exs
config :logger, :console,
  format: {MyApp.LogFormatter, :format},
  metadata: [:request_id, :user_id, :org_id, :module, :function]

# In production, use JSON formatter
config :logger, :console,
  format: "$message\n",
  metadata: :all

# lib/my_app/log_formatter.ex
defmodule MyApp.LogFormatter do
  def format(level, message, timestamp, metadata) do
    %{
      timestamp: format_timestamp(timestamp),
      level: level,
      message: IO.iodata_to_binary(message),
      request_id: metadata[:request_id],
      user_id: metadata[:user_id],
      org_id: metadata[:org_id],
      module: metadata[:module],
      function: metadata[:function]
    }
    |> Jason.encode!()
    |> Kernel.<>("\n")
  end

  defp format_timestamp({date, {h, m, s, ms}}) do
    NaiveDateTime.from_erl!({date, {h, m, s}}, {ms * 1000, 3})
    |> NaiveDateTime.to_iso8601()
  end
end
```

### Grafana Loki Integration

Loki uses labels for indexing (like Prometheus) rather than full-text indexing.

#### Log Shipping Options

| Option | Description |
|--------|-------------|
| Promtail | Loki's native agent, reads log files |
| Vector | High-performance, flexible routing |
| Direct HTTP | Push logs via Loki API |

#### Promtail Configuration

```yaml
# promtail-config.yml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: myapp
    static_configs:
      - targets:
          - localhost
        labels:
          job: myapp
          env: production
          __path__: /var/log/myapp/*.log
    pipeline_stages:
      - json:
          expressions:
            level: level
            request_id: request_id
            user_id: user_id
      - labels:
          level:
      - timestamp:
          source: timestamp
          format: RFC3339
```

#### Useful Loki Queries (LogQL)

```logql
# All errors in last hour
{job="myapp", level="error"}

# Errors for specific user
{job="myapp"} |= `user_id":"123"` | json | level="error"

# Request latency issues
{job="myapp"} | json | duration > 1000

# Count errors by endpoint
sum by (endpoint) (count_over_time({job="myapp", level="error"}[1h]))

# Trace specific request
{job="myapp"} |= `request_id":"abc-123"`
```

### PII Redaction

Never log sensitive data. Redact before logging.

```elixir
defmodule MyApp.LogSanitizer do
  @sensitive_fields ~w(password password_hash token api_key credit_card ssn)

  def sanitize(data) when is_map(data) do
    Map.new(data, fn {k, v} ->
      if to_string(k) in @sensitive_fields do
        {k, "[REDACTED]"}
      else
        {k, sanitize(v)}
      end
    end)
  end

  def sanitize(data), do: data
end

# Usage
Logger.info("User updated", user: LogSanitizer.sanitize(user_params))
```

### Implementation Checklist

- [ ] Configure JSON log formatter
- [ ] Include `request_id` in all logs
- [ ] Include `user_id` and `org_id` when available
- [ ] Set appropriate log levels per environment
- [ ] Set up Promtail or Vector for log shipping
- [ ] Configure Grafana Loki datasource
- [ ] Create Grafana dashboards for log analysis
- [ ] Implement PII redaction for sensitive fields
- [ ] Set up alerts for error rate spikes
- [ ] Configure log retention policy in Loki

### Log Levels by Environment

| Environment | Default Level | Notes |
|-------------|---------------|-------|
| Dev | `debug` | All details for development |
| Test | `warn` | Reduce noise in test output |
| Staging | `info` | Match production behavior |
| Production | `info` | No debug logs (performance) |

### Metrics

- [ ] Instrument request latency and throughput
- [ ] Track error rates
- [ ] Monitor background job queues
- [ ] Create dashboards for key metrics

### Tracing

- [ ] Add OpenTelemetry instrumentation
- [ ] Trace requests across services
- [ ] Include span attributes for debugging

### Error Tracking

- [ ] Configure Sentry with source maps
- [ ] Add user context to errors
- [ ] Set up alerts for new errors

---

# REQUEST ID / CORRELATION ID

Trace requests across your entire system for debugging and observability.

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Header | `X-Request-ID` | Industry standard |
| Format | UUID v4 | Unique, no collisions |
| Generation | Accept or generate | Honor client IDs, generate if missing |
| Propagation | All services | End-to-end tracing |

## Implementation Checklist

### Backend (Phoenix)

- [ ] Create RequestId plug
- [ ] Check for existing `X-Request-ID` header
- [ ] Generate UUID if not present
- [ ] Store in `conn.assigns` and Logger metadata
- [ ] Return in response headers
- [ ] Include in all log entries

### Logger Integration

- [ ] Configure Logger with `$metadata` including request_id
- [ ] Use `Logger.metadata(request_id: id)` in plug
- [ ] All subsequent logs automatically include ID
- [ ] Include in error reports (Sentry)

### External Service Calls

- [ ] Pass `X-Request-ID` to downstream services
- [ ] Include in HTTP client headers
- [ ] Pass to background jobs (Oban job args)
- [ ] Include in webhook payloads

### Background Jobs

- [ ] Accept request_id in job args
- [ ] Set Logger metadata at job start
- [ ] Trace job back to originating request

### Frontend

- [ ] Generate request ID for user-initiated actions
- [ ] Include in API request headers
- [ ] Display in error messages ("Error ID: xxx")
- [ ] Help support locate issues quickly

### Response Headers

- [ ] Always return `X-Request-ID` in responses
- [ ] Client can reference ID when reporting issues
- [ ] Useful for debugging API integrations

---

# CIRCUIT BREAKERS

Protect your application from cascading failures when external services are down.

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Library | `fuse` | Battle-tested Erlang library |
| Strategy | Per-service breaker | Isolate failures |
| States | Closed, Open, Half-Open | Standard circuit breaker pattern |

## Circuit Breaker States

- **Closed**: Normal operation, requests pass through
- **Open**: Service is down, fail fast without calling
- **Half-Open**: Test if service recovered with limited requests

## Implementation Checklist

### Fuse Configuration

- [ ] Add `fuse` to dependencies
- [ ] Define fuse per external service
- [ ] Configure failure threshold (e.g., 5 failures)
- [ ] Configure reset timeout (e.g., 30 seconds)
- [ ] Start fuses in application supervision tree

### Wrapping External Calls

- [ ] Check fuse state before calling
- [ ] If open, return error immediately (fail fast)
- [ ] If closed, make the call
- [ ] On success, reset failure count
- [ ] On failure, record failure (may trip breaker)

### HTTP Client Integration

- [ ] Create wrapper module for external APIs
- [ ] Include circuit breaker check
- [ ] Handle timeouts as failures
- [ ] Handle connection errors as failures

### Services to Protect

- [ ] Payment providers (Stripe)
- [ ] Email services (SendGrid, Postmark)
- [ ] SMS providers (Twilio)
- [ ] Third-party APIs
- [ ] Internal microservices

### Fallback Strategies

- [ ] Return cached data if available
- [ ] Return degraded response
- [ ] Queue for retry (if idempotent)
- [ ] Show user-friendly error message

### Monitoring

- [ ] Log circuit breaker state changes
- [ ] Alert when circuit opens
- [ ] Track time in open state
- [ ] Dashboard showing breaker status

### Configuration per Service

| Service | Threshold | Timeout | Notes |
|---------|-----------|---------|-------|
| Stripe | 3 failures | 60s | Critical, conservative |
| Email | 5 failures | 30s | Can queue and retry |
| Analytics | 10 failures | 15s | Non-critical |

---

# RETRY LOGIC & TIMEOUTS

Handle transient failures gracefully with retries and prevent hanging requests with timeouts.

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| HTTP client | Req / Finch | Modern, configurable |
| Retry strategy | Exponential backoff | Avoid thundering herd |
| Max retries | 3 | Balance reliability vs. latency |
| Jitter | Random | Spread retry storms |

## Timeout Configuration

### HTTP Client Timeouts

| Timeout | Value | Purpose |
|---------|-------|---------|
| Connect | 5s | Time to establish connection |
| Receive | 30s | Time to receive response |
| Pool | 5s | Time to get connection from pool |

### Database Timeouts

- [ ] Query timeout: 15 seconds (Ecto `:timeout` option)
- [ ] Queue timeout: 5 seconds (waiting for connection)
- [ ] Use shorter timeouts for user-facing requests
- [ ] Longer timeouts OK for background jobs

### Background Job Timeouts

- [ ] Set Oban job timeout per worker
- [ ] Default: 60 seconds
- [ ] Long-running jobs: up to 30 minutes
- [ ] Always have a timeout (never infinite)

## Retry Implementation

### What to Retry

- [ ] Network errors (connection refused, timeout)
- [ ] 5xx server errors (503, 502, 500)
- [ ] 429 Too Many Requests (with Retry-After header)
- [ ] DO NOT retry: 4xx client errors, business logic failures

### Exponential Backoff

- [ ] Base delay: 100ms
- [ ] Multiplier: 2x each attempt
- [ ] Max delay cap: 30 seconds
- [ ] Add random jitter (0-25% of delay)
- [ ] Sequence: 100ms, 200ms, 400ms, 800ms...

### Implementation Checklist

- [ ] Create HTTP client wrapper with retry logic
- [ ] Configure timeouts on all HTTP clients
- [ ] Respect `Retry-After` header from APIs
- [ ] Log retry attempts with context
- [ ] Track retry metrics (success after N retries)
- [ ] Set circuit breaker integration (don't retry if open)

### Per-Service Configuration

| Service | Retries | Base Delay | Notes |
|---------|---------|------------|-------|
| Stripe | 3 | 500ms | Idempotent with Idempotency-Key |
| Email | 2 | 1s | Can queue if all fail |
| Webhooks | 0 | - | Handled by delivery system |
| Analytics | 1 | 100ms | Non-critical, fail fast |

### Idempotency

- [ ] Only retry idempotent operations
- [ ] Use idempotency keys for non-idempotent APIs
- [ ] POST requests: check if safe to retry
- [ ] GET/HEAD/OPTIONS: always safe to retry

### Monitoring

- [ ] Track timeout occurrences
- [ ] Track retry success rates
- [ ] Alert on elevated timeout/retry rates
- [ ] Log external service latency

---

# HEALTH CHECKS

Expose endpoints for load balancers, orchestrators, and monitoring systems to verify application health.

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Endpoint | `/health` and `/ready` | Industry convention |
| Liveness | Simple ping | "Is the process running?" |
| Readiness | Dependency checks | "Can it handle traffic?" |

## Implementation Checklist

### Liveness Check (`/health` or `/healthz`)

- [ ] Return 200 if process is running
- [ ] No dependency checks (avoid false negatives)
- [ ] Fast response (under 100ms)
- [ ] Used by: process supervisors, basic monitoring

### Readiness Check (`/ready` or `/readyz`)

- [ ] Check database connectivity
- [ ] Check Redis connectivity
- [ ] Check critical external services
- [ ] Return 200 only if all checks pass
- [ ] Return 503 with details if any check fails
- [ ] Used by: load balancers, monitoring systems

### Deep Health Check (`/health/detailed`) - Optional

- [ ] Return status of all dependencies
- [ ] Include version information
- [ ] Include uptime
- [ ] Protect with authentication (sensitive info)

### Response Format

```json
{
  "status": "healthy",
  "checks": {
    "database": { "status": "healthy", "latency_ms": 2 },
    "redis": { "status": "healthy", "latency_ms": 1 },
    "search": { "status": "degraded", "message": "slow response" }
  }
}
```

### Implementation

- [ ] Create HealthController with check endpoints
- [ ] Add routes outside authentication middleware
- [ ] Set appropriate timeouts for dependency checks
- [ ] Cache check results briefly (avoid hammering dependencies)

---

# GRACEFUL SHUTDOWN

Handle termination signals properly to avoid dropped requests and data loss.

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Signal | SIGTERM | Standard termination signal |
| Drain period | 30 seconds | Allow in-flight requests to complete |
| Order | Stop accepting, drain, terminate | Clean shutdown |

## Implementation Checklist

### Phoenix Endpoint

- [ ] Configure `draining_timeout` in endpoint config
- [ ] Stop accepting new connections on SIGTERM
- [ ] Wait for in-flight requests to complete
- [ ] Return 503 for new requests during drain

### Background Jobs (Oban)

- [ ] Configure graceful shutdown in Oban config
- [ ] Allow running jobs to complete
- [ ] Don't start new jobs during shutdown
- [ ] Set reasonable job timeout (shorter than drain period)

### WebSocket Connections

- [ ] Notify connected clients of impending shutdown
- [ ] Allow clients to reconnect to another instance
- [ ] Close connections after notification

### Database Connections

- [ ] Return connections to pool
- [ ] Wait for transactions to complete
- [ ] Close pool connections cleanly

### systemd Integration

- [ ] Configure `TimeoutStopSec` >= drain period
- [ ] Use `ExecStop` for graceful shutdown script if needed
- [ ] Remove from load balancer before stopping service

---

# ERROR PAGES

Custom error pages for better user experience when things go wrong.

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Style | Match app design | Consistent experience |
| Information | Helpful, not technical | User-friendly |
| Actions | Clear next steps | Reduce frustration |

## Implementation Checklist

### Error Pages to Create

- [ ] 404 Not Found - "Page doesn't exist"
- [ ] 403 Forbidden - "You don't have access"
- [ ] 500 Internal Error - "Something went wrong"
- [ ] 503 Service Unavailable - "Maintenance mode"
- [ ] Offline page (for PWA)

### Content Guidelines

- [ ] Friendly, non-technical language
- [ ] Suggest actions (go home, search, contact support)
- [ ] Include branding (logo, colors)
- [ ] Don't expose stack traces to users
- [ ] Log errors server-side for debugging

### Implementation

- [ ] Phoenix API returns structured JSON errors
- [ ] React ErrorBoundary for component errors
- [ ] React error page components (404, 500, etc.)
- [ ] API client error interceptor
- [ ] Test error handling end-to-end

---

# TERMS OF SERVICE TRACKING

Track user acceptance of terms of service and privacy policy for legal compliance.

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Storage | Separate table | Audit trail, multiple versions |
| Versioning | Date-based or semantic | Track policy changes |
| Re-acceptance | Required on major changes | Legal compliance |

## Database Schema (terms_acceptances table)

- `user_id` (foreign key) - User who accepted
- `document_type` (string) - "terms_of_service", "privacy_policy"
- `version` (string) - Version accepted
- `accepted_at` (datetime) - When accepted
- `ip_address` (string) - For audit
- `user_agent` (string) - For audit

## Implementation Checklist

### Backend

- [ ] Create TermsAcceptance schema
- [ ] Store current version in config
- [ ] Check acceptance on login/signup
- [ ] Require re-acceptance on version change
- [ ] Log all acceptances with context

### API Endpoints

- [ ] `GET /api/terms/status` - Check if user needs to accept
- [ ] `POST /api/terms/accept` - Record acceptance
- [ ] `GET /api/terms/current` - Get current version info

### Frontend

- [ ] Show terms acceptance checkbox on signup
- [ ] Display acceptance modal when new version required
- [ ] Block access until accepted (for major changes)
- [ ] Link to full terms document

### Compliance

- [ ] Keep history of all policy versions
- [ ] Never delete acceptance records
- [ ] Export acceptance history on data request

---

# MONITORING & ALERTS

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Metrics | Prometheus | Industry standard |
| Dashboards | Grafana | Powerful, flexible |
| Alerts | PagerDuty / Opsgenie | Incident management |

## Implementation Checklist

### Metrics to Track

- [ ] Request latency (p50, p95, p99)
- [ ] Error rates (4xx, 5xx)
- [ ] Database query times
- [ ] Background job queue depth
- [ ] Cache hit rates
- [ ] External API latencies

### Dashboards

- [ ] Overview dashboard (key health metrics)
- [ ] API performance dashboard
- [ ] Database dashboard
- [ ] Background jobs dashboard
- [ ] Business metrics dashboard

### Alerts to Configure

| Alert | Threshold | Severity |
|-------|-----------|----------|
| Error rate spike | >5% 5xx | High |
| High latency | p95 > 1s | Medium |
| Database connections | >80% pool | Medium |
| Job queue backup | >1000 pending | Medium |
| Disk space | >80% used | High |
| Certificate expiry | <14 days | High |

---

# DEPLOYMENT

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Platform | Bare metal with NixOS | Full control, cost-effective, reproducible |
| Strategy | Rolling deployment | Zero downtime |
| Build | Nix flake | Reproducible, cacheable builds |
| Secrets | agenix or sops-nix | Encrypted secrets in git |
| Reverse proxy | Caddy or nginx | TLS termination, load balancing |

## Implementation Checklist

### Server Setup (NixOS)

- [ ] Install NixOS on bare metal servers
- [ ] Configure flake-based system configuration
- [ ] Set up SSH access with keys only
- [ ] Configure firewall (only 22, 80, 443)
- [ ] Set up automatic security updates
- [ ] Configure swap and kernel parameters for BEAM

### Elixir Releases

- [ ] Configure `mix release` for production
- [ ] Use runtime configuration (`config/runtime.exs`)
- [ ] Include migrations in release
- [ ] Set up release health check script
- [ ] Configure for clustering if needed (libcluster)
- [ ] Build release via Nix for reproducibility

### NixOS Application Module

- [ ] Create NixOS module for the Phoenix app
- [ ] Define systemd service with proper user
- [ ] Configure environment variables
- [ ] Set up automatic restarts on failure
- [ ] Configure resource limits (memory, file descriptors)
- [ ] Enable systemd watchdog integration

### Secrets Management

- [ ] Use agenix or sops-nix for secrets
- [ ] Encrypt secrets in git repository
- [ ] Configure age keys per server
- [ ] Include DATABASE_URL, SECRET_KEY_BASE, API keys
- [ ] Rotate secrets periodically

### Reverse Proxy (Caddy)

- [ ] Configure Caddy via NixOS module
- [ ] Automatic HTTPS with Let's Encrypt
- [ ] WebSocket proxy for Phoenix Channels
- [ ] Configure rate limiting at proxy level
- [ ] Set security headers

### Database (PostgreSQL)

- [ ] Run PostgreSQL on dedicated server or same host
- [ ] Configure via NixOS module
- [ ] Set up automated backups (pgBackRest or pg_dump)
- [ ] Configure connection limits and memory
- [ ] Enable SSL for connections

### Deployment Process

- [ ] Use deploy-rs or nixos-rebuild switch
- [ ] Deploy from CI or local machine
- [ ] Run migrations before switching
- [ ] Health check before marking deploy complete
- [ ] Automatic rollback on failure

### CI/CD Pipeline

- [ ] Run tests on every push
- [ ] Build Nix derivation on merge to main
- [ ] Push to Nix binary cache (Cachix)
- [ ] Deploy to staging automatically
- [ ] Manual promotion to production
- [ ] Tag releases in git

### High Availability (Optional)

- [ ] Multiple app servers behind load balancer
- [ ] PostgreSQL replication (primary + replica)
- [ ] Redis replication or Redis Cluster
- [ ] Distributed Erlang clustering
- [ ] Health checks remove unhealthy nodes

---

# DISASTER RECOVERY

## RTO and RPO Definitions

| Metric | Definition | Example |
|--------|------------|---------|
| **RTO** (Recovery Time Objective) | Max acceptable downtime | "Back online within 4 hours" |
| **RPO** (Recovery Point Objective) | Max acceptable data loss | "Lose at most 1 hour of data" |

### Setting RTO/RPO by Tier

| Data Tier | RPO | RTO | Backup Strategy |
|-----------|-----|-----|-----------------|
| Critical (payments, orders) | 0-5 min | 15 min | Synchronous replication |
| Important (user data, content) | 1 hour | 4 hours | Hourly snapshots |
| Standard (logs, analytics) | 24 hours | 24 hours | Daily backups |
| Disposable (cache, sessions) | N/A | N/A | No backup needed |

### Implementation Checklist

- [ ] Classify all data by criticality tier
- [ ] Document RTO/RPO for each tier
- [ ] Communicate targets to stakeholders
- [ ] Test that backups meet RPO targets
- [ ] Test that recovery meets RTO targets
- [ ] Review and update quarterly

## Backup Strategy

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Full backup | Daily at 03:00 UTC | Low traffic window |
| Incremental | Hourly | Balance storage vs. RPO |
| WAL archiving | Continuous | Point-in-time recovery |
| Retention | 30 days | Compliance, recovery options |
| Storage | Separate region | Survive regional outages |

## Backup Restoration Testing

**CRITICAL**: Untested backups are not backups. Schedule regular restoration drills.

### Monthly Restoration Drill

- [ ] Restore latest backup to isolated environment
- [ ] Verify data integrity (row counts, checksums)
- [ ] Test application functionality against restored data
- [ ] Measure actual restoration time (compare to RTO)
- [ ] Document any issues found
- [ ] Update runbooks based on learnings

### Restoration Test Checklist

```bash
# 1. Restore to test environment
pg_restore -h test-db -d myapp_restore latest_backup.dump

# 2. Verify row counts match production
psql -c "SELECT 'users', COUNT(*) FROM users UNION ALL
         SELECT 'orders', COUNT(*) FROM orders;"

# 3. Run application health checks
curl https://test-restored.myapp.com/health

# 4. Verify critical queries work
psql -c "SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '1 day';"
```

### Quarterly Full DR Drill

- [ ] Simulate complete infrastructure failure
- [ ] Restore from backups to new infrastructure
- [ ] Measure total recovery time
- [ ] Test failover procedures
- [ ] Update documentation based on findings
- [ ] Report results to stakeholders

### Backup Verification

- [ ] Automated daily backup integrity checks
- [ ] Alert if backup size deviates >20% from expected
- [ ] Alert if backup age exceeds threshold
- [ ] Verify backups are encrypted
- [ ] Verify backups are in separate region

## High Availability

- [ ] Run multiple application instances (min 2)
- [ ] Use database replication (streaming replica)
- [ ] Configure health checks and auto-restart
- [ ] Plan for region failover
- [ ] Test failover procedures quarterly

## Incident Response

- [ ] Document incident response procedure
- [ ] Define on-call rotation
- [ ] Create runbooks for common issues
- [ ] Conduct blameless post-mortems
- [ ] Track incidents and resolution times

---

# DEVELOPMENT WORKFLOW

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Branching | GitHub Flow | Simple, effective |
| Commits | Conventional Commits | Consistent, automation-friendly |
| Reviews | Required PR reviews | Code quality |

## Implementation Checklist

### Git Workflow

- [ ] Main branch is always deployable
- [ ] Feature branches for development
- [ ] Require PR reviews before merge
- [ ] Squash commits on merge

### Code Quality

- [ ] Run linter on commit (pre-commit hook)
- [ ] Run tests on push
- [ ] Require passing CI for merge
- [ ] Use code formatting tools (mix format, prettier)

### Dependency Security

Automate vulnerability detection in dependencies.

#### Backend (Elixir)

- [ ] Run `mix deps.audit` in CI (via `mix_audit` package)
- [ ] Run `mix hex.audit` for Hex package advisories
- [ ] Pin major versions in `mix.exs`
- [ ] Review dependency updates weekly
- [ ] Subscribe to security advisories for critical deps

```elixir
# mix.exs - Add mix_audit
{:mix_audit, "~> 2.0", only: [:dev, :test], runtime: false}
```

```yaml
# CI step
- name: Security Audit
  run: |
    mix deps.audit
    mix hex.audit
```

#### Frontend (npm/pnpm)

- [ ] Run `pnpm audit` in CI
- [ ] Enable Dependabot or Renovate for automated PRs
- [ ] Set up GitHub security alerts
- [ ] Review and merge security PRs promptly

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "mix"
    directory: "/backend"
    schedule:
      interval: "weekly"
  - package-ecosystem: "npm"
    directory: "/frontend"
    schedule:
      interval: "weekly"
```

#### Security Practices

- [ ] Block merges if audit finds high/critical vulnerabilities
- [ ] Document exceptions for false positives
- [ ] Update dependencies at least monthly
- [ ] Monitor CVE databases for zero-days

### Documentation

- [ ] Keep README up to date
- [ ] Document setup process
- [ ] Maintain CHANGELOG
- [ ] Document API changes

---

# TESTING INFRASTRUCTURE

## Test Types and Tools

| Test Type | Backend Tool | Frontend Tool |
|-----------|--------------|---------------|
| Unit | ExUnit | Vitest |
| Integration | ExUnit + Ecto Sandbox | React Testing Library |
| E2E | - | Playwright |
| Load | k6 | k6 |

## Deterministic Test Data Rule

**IMPORTANT**: Never use random data in tests or seeds. All test data must be deterministic.

- [ ] No `Faker` or random generators in factories
- [ ] Use explicit, predictable values ("test@example.com", not random email)
- [ ] Seeds produce identical data on every run
- [ ] Tests are reproducible and debuggable
- [ ] Timestamps use fixed values or `~U[2024-01-01 00:00:00Z]`

Rationale: Random data causes flaky tests and makes debugging difficult. Deterministic data ensures consistent behavior across runs.

## Factory Pattern

Use simple factory functions instead of complex libraries. Keep test data explicit.

```elixir
# test/support/factory.ex
defmodule MyApp.Factory do
  alias MyApp.Repo

  def build(:user, attrs \\ %{}) do
    %MyApp.User{
      email: "user@example.com",
      name: "Test User",
      password_hash: Argon2.hash_pwd_salt("password123")
    }
    |> Map.merge(attrs)
  end

  def build(:organization, attrs \\ %{}) do
    %MyApp.Organization{
      name: "Test Org",
      slug: "test-org"
    }
    |> Map.merge(attrs)
  end

  def insert(factory, attrs \\ %{}) do
    factory
    |> build(attrs)
    |> Repo.insert!()
  end
end
```

### Factory Rules

- [ ] `build/2` returns struct without inserting
- [ ] `insert/2` builds and inserts to database
- [ ] Default values are deterministic (no random)
- [ ] Override via attrs map for test-specific values
- [ ] Keep factories in `test/support/factory.ex`

## Implementation Checklist

### Backend Testing

- [ ] Unit test contexts and schemas
- [ ] Test controllers with integration tests
- [ ] Use simple factory pattern for test data
- [ ] Test with Ecto sandbox for isolation
- [ ] Mock external services
- [ ] No random data in tests

### Frontend Testing

- [ ] Unit test utilities and hooks
- [ ] Component tests with React Testing Library
- [ ] Mock API responses with MSW
- [ ] Test user interactions

### E2E Testing (Playwright)

End-to-end tests verify critical user journeys work correctly across the full stack.

**Key Decisions**

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Framework | Playwright | Cross-browser, fast, great DX |
| Run against | Staging or local | Real backend, seeded data |
| Pattern | Page Object Model | Maintainable, reusable |
| Parallelism | Per-file | Isolated test data per file |

**Project Structure**

```
frontend/
  e2e/
    fixtures/           # Test setup, auth helpers
      auth.ts
    pages/              # Page objects
      login.page.ts
      dashboard.page.ts
    tests/
      auth.spec.ts
      onboarding.spec.ts
    playwright.config.ts
```

**Page Object Example**

```typescript
// e2e/pages/login.page.ts
import { Page, Locator } from '@playwright/test';

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorMessage: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel('Email');
    this.passwordInput = page.getByLabel('Password');
    this.submitButton = page.getByRole('button', { name: 'Sign in' });
    this.errorMessage = page.getByRole('alert');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }
}
```

**Test Example**

```typescript
// e2e/tests/auth.spec.ts
import { test, expect } from '@playwright/test';
import { LoginPage } from '../pages/login.page';

test.describe('Authentication', () => {
  test('successful login redirects to dashboard', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login('test@example.com', 'password123');

    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByText('Welcome back')).toBeVisible();
  });

  test('invalid credentials show error', async ({ page }) => {
    const loginPage = new LoginPage(page);
    await loginPage.goto();
    await loginPage.login('test@example.com', 'wrongpassword');

    await expect(loginPage.errorMessage).toContainText('Invalid credentials');
    await expect(page).toHaveURL('/login');
  });
});
```

**Auth Fixture (Reusable Login State)**

```typescript
// e2e/fixtures/auth.ts
import { test as base } from '@playwright/test';

export const test = base.extend<{ authenticatedPage: Page }>({
  authenticatedPage: async ({ page }, use) => {
    // Login once, reuse session
    await page.goto('/login');
    await page.getByLabel('Email').fill('test@example.com');
    await page.getByLabel('Password').fill('password123');
    await page.getByRole('button', { name: 'Sign in' }).click();
    await page.waitForURL('/dashboard');

    await use(page);
  },
});
```

**Critical User Journeys to Test**

- [ ] Sign up flow (email verification if applicable)
- [ ] Login / logout
- [ ] Password reset
- [ ] Onboarding wizard
- [ ] Core CRUD operations (create, read, update, delete)
- [ ] Billing flow (upgrade, downgrade, cancel)
- [ ] Team invite and accept
- [ ] Settings changes persist
- [ ] Error states display correctly

**Implementation Checklist**

- [ ] Install Playwright: `pnpm create playwright`
- [ ] Configure `playwright.config.ts` with base URL
- [ ] Create page objects for main pages
- [ ] Write tests for critical user journeys
- [ ] Set up test database seeding for E2E
- [ ] Add to CI pipeline (run on PR, before deploy)
- [ ] Configure retries for flaky network tests
- [ ] Use `test.describe.serial` for order-dependent tests
- [ ] Screenshot on failure for debugging
- [ ] Run in headless mode in CI

**CI Configuration**

```yaml
# .github/workflows/e2e.yml
e2e:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - uses: actions/setup-node@v4
    - run: pnpm install
    - run: pnpm exec playwright install --with-deps
    - run: pnpm exec playwright test
    - uses: actions/upload-artifact@v4
      if: failure()
      with:
        name: playwright-report
        path: playwright-report/
```

### Performance Testing

- [ ] Define load test scenarios
- [ ] Test with realistic data volumes
- [ ] Establish baseline metrics
- [ ] Run before major releases

---

# USER IMPERSONATION

Allow admins to view the app as another user for support and debugging.

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Access | Super admin only | Sensitive capability |
| Audit | Full logging | Accountability |
| Indicator | Visible banner | Prevent confusion |
| Actions | Read-only or full | Depends on use case |

## Implementation Checklist

### Backend

- [ ] Create impersonation endpoint (admin only)
- [ ] Store original admin ID in session alongside impersonated user
- [ ] Check impersonation permission (super admin role)
- [ ] Log impersonation start/end with admin ID, target user, timestamp
- [ ] Optionally restrict write operations while impersonating

### Session Management

- [ ] Add `impersonator_id` to session data
- [ ] Create `impersonating?/1` helper function
- [ ] Create `stop_impersonating/1` to restore admin session
- [ ] Auto-expire impersonation after timeout (30 minutes)

### API Endpoints

- [ ] `POST /api/admin/impersonate/:user_id` - Start impersonation
- [ ] `POST /api/admin/stop-impersonation` - Return to admin
- [ ] `GET /api/account/me` - Include `impersonated_by` field if active

### Frontend

- [ ] Show prominent banner when impersonating ("Viewing as user@example.com")
- [ ] Include "Stop impersonation" button in banner
- [ ] Use different header/theme color during impersonation
- [ ] Disable or warn on destructive actions

### Security

- [ ] Require 2FA before impersonating
- [ ] Limit impersonation to lower-privilege users (no admin-to-admin)
- [ ] Log all actions taken while impersonating
- [ ] Alert user that their account was accessed by support (optional)
- [ ] IP/location restrictions for impersonation

---

# ADMIN DASHBOARD

Internal tools for managing users, content, and system health.

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Access | Role-based | admin, super_admin roles |
| Routes | Separate namespace | `/admin/*` paths |
| Auth | Same system + role check | Unified authentication |

## Implementation Checklist

### Authentication & Authorization

- [ ] Create admin role(s) in user schema
- [ ] Create AdminAuth plug checking admin role
- [ ] Separate admin routes in router (`/admin` scope)
- [ ] Log all admin actions

### Dashboard Sections

#### User Management

- [ ] User list with search, filter, pagination
- [ ] User detail view (profile, activity, sessions)
- [ ] Edit user (role, status, email verification)
- [ ] Impersonate user
- [ ] Suspend/ban user
- [ ] Delete/anonymize user (GDPR)

#### Content Moderation (if applicable)

- [ ] Flagged content queue
- [ ] Approve/reject/delete actions
- [ ] Ban content creators

#### System Health

- [ ] Background job queue status
- [ ] Error rates and recent errors
- [ ] Database/cache connection status
- [ ] Feature flag status

#### Billing (if applicable)

- [ ] Subscription overview
- [ ] Revenue metrics
- [ ] Failed payment alerts
- [ ] Manual subscription adjustments

### API Endpoints

- [ ] `GET /api/admin/users` - List users (with filters)
- [ ] `GET /api/admin/users/:id` - User details
- [ ] `PATCH /api/admin/users/:id` - Update user
- [ ] `POST /api/admin/users/:id/suspend` - Suspend user
- [ ] `GET /api/admin/stats` - Dashboard metrics

### Security

- [ ] All admin routes require admin role
- [ ] Audit log all admin actions
- [ ] Rate limit admin endpoints
- [ ] Two-factor required for admin access
- [ ] IP allowlist for admin routes (optional)

---

# IN-APP NOTIFICATIONS

Real-time notifications within the application UI.

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Delivery | Phoenix Channels | Real-time push |
| Storage | Database | Persistence, history |
| Types | Multiple channels | In-app, email, push |

## Database Schema (notifications table)

- `id` (uuid) - Primary key
- `user_id` (foreign key) - Recipient
- `type` (string) - Notification type ("mention", "comment", "system")
- `title` (string) - Short title
- `body` (text) - Notification content
- `data` (jsonb) - Structured payload (links, references)
- `read_at` (datetime, nullable) - When read
- `seen_at` (datetime, nullable) - When seen (in list)
- `inserted_at` - Timestamp

## Implementation Checklist

### Backend

- [ ] Create Notifications context
- [ ] `create_notification/2` - Create and broadcast
- [ ] `list_notifications/2` - Paginated, unread first
- [ ] `mark_as_read/2` - Mark single notification
- [ ] `mark_all_as_read/1` - Mark all for user
- [ ] `unread_count/1` - Get count for badge
- [ ] Broadcast new notifications via Phoenix Channel

### Real-time Delivery

- [ ] Create user notifications channel (`user:notifications:{user_id}`)
- [ ] Broadcast on notification create
- [ ] Include notification payload in broadcast
- [ ] Handle reconnection (fetch missed notifications)

### API Endpoints

- [ ] `GET /api/notifications` - List notifications
- [ ] `POST /api/notifications/:id/read` - Mark as read
- [ ] `POST /api/notifications/read-all` - Mark all as read
- [ ] `GET /api/notifications/unread-count` - Badge count

### Frontend

- [ ] Notification bell icon with unread count badge
- [ ] Notification dropdown/panel
- [ ] Mark as read on view
- [ ] Click to navigate to related resource
- [ ] Real-time updates via WebSocket
- [ ] Toast/popup for new notifications

### Notification Types

- [ ] System announcements
- [ ] User mentions
- [ ] Comment replies
- [ ] Assignment notifications
- [ ] Billing alerts
- [ ] Security alerts (new login, password change)

---

# ACCESSIBILITY (A11Y)

Build an accessible application that works for everyone.

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Standard | WCAG 2.1 AA | Industry standard, legal compliance |
| Components | Radix UI | Accessible primitives |
| Testing | axe-core | Automated accessibility testing |

## Implementation Checklist

### Semantic HTML

- [ ] Use proper heading hierarchy (h1 > h2 > h3)
- [ ] Use semantic elements (nav, main, article, aside)
- [ ] Use button for actions, a for navigation
- [ ] Use lists for list content (ul, ol)
- [ ] Use tables for tabular data (with headers)

### Keyboard Navigation

- [ ] All interactive elements focusable
- [ ] Visible focus indicators (not just outline: none)
- [ ] Logical tab order
- [ ] Skip to main content link
- [ ] Escape closes modals/dropdowns
- [ ] Arrow keys for menus and lists

### Screen Readers

- [ ] Alt text for all images
- [ ] ARIA labels for icon buttons
- [ ] ARIA live regions for dynamic content
- [ ] Proper form labels (associated with inputs)
- [ ] Error messages announced

### Visual

- [ ] Color contrast ratio 4.5:1 minimum (text)
- [ ] Don't rely on color alone for meaning
- [ ] Resizable text (up to 200%)
- [ ] Responsive at all zoom levels
- [ ] Reduced motion support (`prefers-reduced-motion`)

### Forms

- [ ] Labels associated with inputs
- [ ] Error messages linked to fields (aria-describedby)
- [ ] Required fields indicated
- [ ] Autocomplete attributes
- [ ] Fieldsets for grouped inputs

### Automated Testing with axe-core

Catch accessibility issues automatically in development and CI.

#### Development Setup

```typescript
// Install: pnpm add -D @axe-core/react
// main.tsx (dev only)
import React from 'react';
import ReactDOM from 'react-dom';

if (process.env.NODE_ENV === 'development') {
  import('@axe-core/react').then(axe => {
    axe.default(React, ReactDOM, 1000);
  });
}
```

#### Component Testing

```typescript
// Install: pnpm add -D jest-axe
import { axe, toHaveNoViolations } from 'jest-axe';

expect.extend(toHaveNoViolations);

test('Button is accessible', async () => {
  const { container } = render(<Button>Click me</Button>);
  const results = await axe(container);
  expect(results).toHaveNoViolations();
});
```

#### E2E Testing (Playwright)

```typescript
// Install: pnpm add -D @axe-core/playwright
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('homepage is accessible', async ({ page }) => {
  await page.goto('/');
  const results = await new AxeBuilder({ page }).analyze();
  expect(results.violations).toEqual([]);
});

test('dashboard is accessible', async ({ page }) => {
  await page.goto('/dashboard');
  const results = await new AxeBuilder({ page })
    .withTags(['wcag2a', 'wcag2aa'])
    .analyze();
  expect(results.violations).toEqual([]);
});
```

#### CI Integration

```yaml
# Run accessibility tests in CI
- name: Accessibility Tests
  run: pnpm test:a11y

- name: Playwright A11y
  run: pnpm playwright test --grep @a11y
```

#### Implementation Checklist

- [ ] Add @axe-core/react for dev-time feedback
- [ ] Add jest-axe for component tests
- [ ] Add @axe-core/playwright for E2E tests
- [ ] Run a11y tests in CI pipeline
- [ ] Block PRs with accessibility violations
- [ ] Test critical user flows with screen reader
- [ ] Test with keyboard-only navigation
- [ ] Verify at 200% browser zoom

### Radix UI Integration

- [ ] Use Radix primitives (Dialog, Dropdown, Tabs, etc.)
- [ ] Follow Radix accessibility patterns
- [ ] Don't override accessibility features
- [ ] Test compound components

---

# ONBOARDING FLOW

Guide new users through setup to increase activation and reduce churn.

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Style | Checklist + contextual | Clear progress, in-context help |
| Persistence | Database | Track completion across sessions |
| Skippable | Yes | Don't block power users |

## Database Schema (user_onboarding table)

- `user_id` (foreign key) - User being onboarded
- `completed_steps` (array of strings) - Completed step IDs
- `current_step` (string, nullable) - Current step ID
- `skipped_at` (datetime, nullable) - If user skipped onboarding
- `completed_at` (datetime, nullable) - When fully completed
- `inserted_at`, `updated_at`

## Implementation Checklist

### Onboarding Steps (Example)

- [ ] Welcome / account setup (name, avatar)
- [ ] Connect integrations (if applicable)
- [ ] Create first resource (project, workspace)
- [ ] Invite team members
- [ ] Enable notifications
- [ ] Complete profile

### Backend

- [ ] Create Onboarding context
- [ ] `get_onboarding_status/1` - Get user's current state
- [ ] `complete_step/2` - Mark step as complete
- [ ] `skip_onboarding/1` - Allow skipping
- [ ] `reset_onboarding/1` - For testing/support
- [ ] Track completion metrics

### API Endpoints

- [ ] `GET /api/onboarding` - Get current status and next steps
- [ ] `POST /api/onboarding/steps/:step/complete` - Mark complete
- [ ] `POST /api/onboarding/skip` - Skip remaining steps

### Frontend

- [ ] Onboarding checklist component (sidebar or modal)
- [ ] Progress indicator (3 of 5 steps complete)
- [ ] Contextual tooltips and highlights
- [ ] "Skip for now" option
- [ ] Celebration on completion (confetti!)
- [ ] Re-access onboarding from help menu

### Contextual Guidance

- [ ] Highlight UI elements with tooltips
- [ ] Show empty states with CTAs
- [ ] Provide sample data option
- [ ] Link to documentation

### Analytics

- [ ] Track step completion rates
- [ ] Identify drop-off points
- [ ] Measure time to complete
- [ ] Correlate with retention

---

# PRODUCT ANALYTICS

Track user behavior to understand usage patterns and improve the product.

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Provider | PostHog / Mixpanel / Amplitude | Feature-rich, self-host option |
| Self-hosted | PostHog | Privacy, cost control |
| Consent | Required for GDPR | Legal compliance |

## Implementation Checklist

### Setup

- [ ] Choose analytics provider
- [ ] Install SDK (frontend and/or backend)
- [ ] Configure user identification
- [ ] Set up development vs production environments
- [ ] Respect Do Not Track / consent preferences

### User Identification

- [ ] Identify users on login with user ID
- [ ] Set user properties (plan, role, signup date)
- [ ] Handle anonymous → identified transition
- [ ] Reset on logout

### Events to Track

#### Core Product Events

- [ ] Feature usage (which features, how often)
- [ ] Key actions (create, edit, delete resources)
- [ ] Search queries
- [ ] Error occurrences

#### Engagement Events

- [ ] Session start/end
- [ ] Page/screen views
- [ ] Time on page
- [ ] Scroll depth (for content)

#### Conversion Events

- [ ] Signup completed
- [ ] Onboarding step completed
- [ ] Subscription started
- [ ] Feature adoption (first use of key features)

### Frontend Integration

- [ ] Create analytics wrapper/hook
- [ ] Track page views on navigation
- [ ] Track clicks on key buttons
- [ ] Track form submissions
- [ ] Include relevant properties with events

### Backend Integration (Optional)

- [ ] Track server-side events for accuracy
- [ ] Subscription events from Stripe webhooks
- [ ] Background job completions
- [ ] API usage metrics

### Privacy & Compliance

- [ ] Honor cookie consent choices
- [ ] Provide opt-out mechanism
- [ ] Don't track PII unnecessarily
- [ ] Document what's tracked in privacy policy
- [ ] Support data deletion requests

### Dashboards

- [ ] Daily/weekly/monthly active users
- [ ] Feature adoption rates
- [ ] Conversion funnels
- [ ] Retention cohorts
- [ ] Power user identification

---

# CDN CONFIGURATION

Use a CDN (Cloudflare, BunnyCDN, Fastly) to serve static assets (JS, CSS, images, fonts) from edge locations. Include content hash in filenames (`app.abc123.js`) for cache busting. Set long Cache-Control headers (`public, max-age=31536000, immutable`) for static assets.

Configure the CDN with your origin (server or S3), custom domain (`cdn.yourapp.com`), and SSL. Keep API on main domain for cookies/CORS. Enable Brotli/Gzip compression and optimize images (WebP, AVIF).

For cache headers: static assets get long TTL, HTML pages get `no-cache`, API responses get `no-store`. Monitor cache hit ratio and bandwidth usage.

---

# CACHING

Use Redis with Redix client for distributed caching. Implement a cache module with `get/1`, `put/3`, `delete/1`, and `fetch/3` (get or compute). Use cache-aside pattern: check cache, on miss compute and store.

TTL by data type: user sessions (30 days), feature flags (1 minute), computed data (1 hour). Invalidate on data mutation. Use cache tags for bulk invalidation.

For HTTP caching, set appropriate `Cache-Control` headers, use `ETag` for conditional requests, and implement `304 Not Modified` responses. Monitor cache hit rates.

---

# TEAM INVITATIONS

Allow users to invite others to join their team or organization.

## Key Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Delivery | Email | Universal, no account needed |
| Token TTL | 7 days | Balance security and convenience |
| Flow | Accept → create account or login | Handle both new and existing users |

## Database Schema (invitations table)

- `id` (uuid) - Primary key
- `team_id` (foreign key) - Target team
- `email` (string) - Invitee email (normalized)
- `role` (string) - Role to assign on accept
- `invited_by_id` (foreign key) - User who sent invite
- `token_hash` (string) - SHA-256 hash of token
- `expires_at` (datetime) - Token expiration
- `accepted_at` (datetime, nullable) - When accepted
- `declined_at` (datetime, nullable) - When declined
- `revoked_at` (datetime, nullable) - When cancelled by inviter
- `inserted_at`

Index: unique on `(team_id, email)` where not accepted/declined/revoked

## Implementation Checklist

### Backend

- [ ] Create Invitations context
- [ ] `create_invitation/3` - Create and send invitation
- [ ] `accept_invitation/2` - Accept and add to team
- [ ] `decline_invitation/1` - Decline invitation
- [ ] `revoke_invitation/1` - Cancel pending invitation
- [ ] `list_pending_invitations/1` - For team settings
- [ ] Check for existing user with email

### Invitation Flow

1. User enters email and selects role
2. Check if email already on team (error)
3. Check for existing pending invitation (resend or error)
4. Create invitation with secure token
5. Send invitation email
6. Recipient clicks link
7. If logged in as different user: prompt to switch or decline
8. If not logged in: login or create account
9. After auth: auto-accept invitation
10. Add user to team with role

### API Endpoints

- [ ] `POST /api/teams/:team_id/invitations` - Send invitation
- [ ] `GET /api/teams/:team_id/invitations` - List pending
- [ ] `DELETE /api/teams/:team_id/invitations/:id` - Revoke
- [ ] `POST /api/invitations/:token/accept` - Accept
- [ ] `POST /api/invitations/:token/decline` - Decline
- [ ] `GET /api/invitations/:token` - Get invitation details (for preview)

### Email

- [ ] Invitation email with team name, inviter name
- [ ] Clear call-to-action button
- [ ] Expiration notice
- [ ] Link to decline (optional)

### Frontend

- [ ] Invite form (email input, role selector)
- [ ] Pending invitations list with resend/revoke
- [ ] Invitation accept page
- [ ] Handle logged-in-as-wrong-user case
- [ ] Success message after accepting

### Edge Cases

- [ ] User already on team: show error
- [ ] Invitation expired: show error with option to request new
- [ ] Duplicate invitation: option to resend or update role
- [ ] Invitee deletes account: clean up invitations
- [ ] Team deleted: invalidate all invitations

### Notifications

- [ ] Notify team admins of accepted invitations
- [ ] Reminder email before expiration (optional)

---

# NICE TO HAVE

## A/B Testing Framework

- [ ] Implement experiment assignment (user bucketing)
- [ ] Track experiment participation
- [ ] Collect metrics per variant
- [ ] Build analysis dashboard
- [ ] Consider: Growthbook, Optimizely, or custom

## Referral System

- [ ] Generate unique referral codes per user
- [ ] Track referral signups
- [ ] Implement reward logic (credits, discounts)
- [ ] Prevent fraud (same IP, fake accounts)
- [ ] Build referral dashboard

## Notification Preferences

- [ ] Define notification types (email, push, in-app)
- [ ] Create preferences schema per user
- [ ] Build preferences UI (matrix of type x channel)
- [ ] Respect preferences in notification sending
- [ ] Support notification digests

## Bulk Import

- [ ] Support CSV upload for batch data
- [ ] Validate data before processing
- [ ] Process in background job
- [ ] Provide progress feedback
- [ ] Generate import report with errors

## Status Page

- [ ] Display service health publicly
- [ ] Track incident history
- [ ] Allow subscription to updates
- [ ] Consider: Statuspage.io, Instatus, or custom

## OpenTelemetry

- [ ] Add OpenTelemetry SDK
- [ ] Instrument Phoenix and Ecto
- [ ] Configure exporter (Jaeger, Honeycomb, etc.)
- [ ] Add custom spans for business operations
- [ ] Correlate logs with traces

---

# SUMMARY

This checklist covers the essential components for building a production-ready SaaS application. Prioritize based on your specific needs:

**Start with (MVP):**
- Authentication & Authorization
- API Design
- Database setup
- Basic frontend

**Add for launch:**
- Email sending
- Background jobs
- Error tracking
- Health checks

**Scale with:**
- Caching
- Rate limiting
- Search
- Multi-tenancy

**Enterprise features:**
- Audit logging
- GDPR compliance
- SSO/SAML
- Advanced analytics
