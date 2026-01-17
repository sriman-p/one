# Enterprise Web Application Architecture Plan

## TypeScript/Node.js Full-Stack Web Application

**Target:** Enterprise-Grade | **Stack:** TypeScript/Node.js | **Compliance:** SOC 2, GDPR/CCPA Ready

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Technology Stack Overview](#2-technology-stack-overview)
3. [Backend Architecture](#3-backend-architecture)
4. [Frontend Architecture](#4-frontend-architecture)
5. [Data Architecture](#5-data-architecture)
6. [Security & Compliance](#6-security--compliance)
7. [Infrastructure & DevOps](#7-infrastructure--devops)
8. [Implementation Roadmap](#8-implementation-roadmap)
9. [Project Structure](#9-project-structure)
10. [Appendix: Decision Records](#10-appendix-decision-records)

---

## 1. Executive Summary

### Vision

Build a scalable, secure, and maintainable enterprise web application using modern TypeScript/Node.js technologies with full compliance readiness for SOC 2, GDPR, and CCPA requirements.

### Key Architectural Decisions

| Area | Decision | Rationale |
|------|----------|-----------|
| **Architecture Style** | Modular Monolith (evolve to microservices) | Reduces complexity while maintaining boundaries |
| **Backend Framework** | NestJS | Enterprise features, DI, strong TypeScript support |
| **Frontend Framework** | Next.js 14+ (App Router) | SSR/SSG flexibility, enterprise adoption |
| **Database** | PostgreSQL (Aurora) | ACID compliance, JSON support, mature ecosystem |
| **Caching** | Redis (ElastiCache) | Data structures, pub/sub, persistence |
| **Cloud Provider** | AWS | Most mature, broadest services |
| **CI/CD** | GitHub Actions | Native integration, OIDC support |
| **Observability** | Datadog | Unified platform, excellent Node.js APM |

### Success Metrics

- **Availability:** 99.95% uptime SLA
- **Performance:** P95 API latency < 200ms
- **Security:** Zero critical vulnerabilities in production
- **Deployment:** Multiple deployments per day capability
- **Recovery:** RTO < 1 hour, RPO < 5 minutes

---

## 2. Technology Stack Overview

### Core Stack

```
┌─────────────────────────────────────────────────────────────────────┐
│                        PRESENTATION LAYER                            │
├─────────────────────────────────────────────────────────────────────┤
│  Next.js 14+ │ React 18 │ TanStack Query │ Zustand │ shadcn/ui     │
│  Tailwind CSS │ TypeScript │ Playwright (E2E) │ Vitest (Unit)      │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                          API LAYER                                   │
├─────────────────────────────────────────────────────────────────────┤
│  NestJS │ REST/GraphQL │ OpenAPI │ Zod Validation │ Passport.js     │
│  Bull MQ (Jobs) │ Redis (Cache/Sessions) │ Rate Limiting            │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         DATA LAYER                                   │
├─────────────────────────────────────────────────────────────────────┤
│  PostgreSQL (Aurora) │ Prisma ORM │ Redis │ OpenSearch │ Kafka      │
│  S3 (Files) │ Snowflake (Analytics)                                  │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      INFRASTRUCTURE                                  │
├─────────────────────────────────────────────────────────────────────┤
│  AWS (EKS/Aurora/ElastiCache) │ Terraform │ GitHub Actions          │
│  Datadog │ PagerDuty │ HashiCorp Vault │ CloudFlare (CDN/WAF)       │
└─────────────────────────────────────────────────────────────────────┘
```

### Technology Matrix

| Layer | Technology | Version | Purpose |
|-------|------------|---------|---------|
| **Runtime** | Node.js | 20 LTS | Server runtime |
| **Language** | TypeScript | 5.x | Type safety |
| **Backend Framework** | NestJS | 10.x | API framework |
| **Frontend Framework** | Next.js | 14.x | React framework |
| **UI Components** | shadcn/ui + Radix | Latest | Component library |
| **Styling** | Tailwind CSS | 3.x | Utility CSS |
| **ORM** | Prisma | 5.x | Database access |
| **Validation** | Zod | 3.x | Schema validation |
| **State (Server)** | TanStack Query | 5.x | Server state |
| **State (Client)** | Zustand | 4.x | Client state |
| **Testing** | Vitest + Playwright | Latest | Unit/E2E testing |
| **Job Queue** | BullMQ | 5.x | Background jobs |
| **Authentication** | Auth0 | - | Identity provider |
| **Secrets** | HashiCorp Vault | - | Secrets management |

---

## 3. Backend Architecture

### 3.1 Architectural Pattern: Clean Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         PRESENTATION                                 │
│  Controllers │ Guards │ Interceptors │ Pipes │ Exception Filters    │
├─────────────────────────────────────────────────────────────────────┤
│                         APPLICATION                                  │
│  Use Cases │ DTOs │ Application Services │ Event Handlers            │
├─────────────────────────────────────────────────────────────────────┤
│                           DOMAIN                                     │
│  Entities │ Value Objects │ Domain Events │ Repository Interfaces    │
├─────────────────────────────────────────────────────────────────────┤
│                        INFRASTRUCTURE                                │
│  Repository Implementations │ External APIs │ Message Queues         │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 Module Structure (Domain-Driven)

```
src/
├── modules/
│   ├── users/
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   │   └── user.entity.ts
│   │   │   ├── value-objects/
│   │   │   │   └── email.vo.ts
│   │   │   ├── events/
│   │   │   │   └── user-created.event.ts
│   │   │   └── repositories/
│   │   │       └── user.repository.interface.ts
│   │   ├── application/
│   │   │   ├── use-cases/
│   │   │   │   ├── create-user/
│   │   │   │   │   ├── create-user.use-case.ts
│   │   │   │   │   ├── create-user.dto.ts
│   │   │   │   │   └── create-user.spec.ts
│   │   │   │   └── get-user/
│   │   │   └── services/
│   │   ├── infrastructure/
│   │   │   ├── repositories/
│   │   │   │   └── prisma-user.repository.ts
│   │   │   └── adapters/
│   │   └── presentation/
│   │       ├── controllers/
│   │       │   └── users.controller.ts
│   │       └── dtos/
│   │           └── user-response.dto.ts
│   ├── orders/
│   │   └── ... (same structure)
│   └── auth/
│       └── ... (same structure)
├── shared/
│   ├── kernel/          # Shared domain concepts
│   ├── infrastructure/  # Shared infra (logging, etc.)
│   └── utils/
└── main.ts
```

### 3.3 API Design Standards

#### REST Endpoints

```
Base URL: /api/v1

Resource Patterns:
  GET    /users              # List users (paginated)
  POST   /users              # Create user
  GET    /users/:id          # Get user by ID
  PATCH  /users/:id          # Partial update
  DELETE /users/:id          # Delete user
  GET    /users/:id/orders   # Nested resources

Query Parameters:
  ?page=1&limit=20           # Pagination
  ?sort=-createdAt,name      # Sorting (- for desc)
  ?filter[status]=active     # Filtering
  ?include=orders,profile    # Relations
```

#### Response Format

```typescript
// Success Response
{
  "data": { ... } | [ ... ],
  "meta": {
    "pagination": {
      "page": 1,
      "limit": 20,
      "total": 100,
      "totalPages": 5
    }
  }
}

// Error Response
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Validation failed",
    "details": [
      { "field": "email", "message": "Invalid email format" }
    ],
    "requestId": "req_abc123"
  }
}
```

#### Rate Limiting

| Tier | Limit | Window |
|------|-------|--------|
| Anonymous | 100 requests | 15 minutes |
| Authenticated | 1000 requests | 15 minutes |
| Premium | 5000 requests | 15 minutes |
| Internal Services | Unlimited | - |

### 3.4 Background Jobs Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                       Job Queue Architecture                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐        │
│  │   Producer   │────▶│  Redis/Bull  │────▶│   Workers    │        │
│  │  (API/Cron)  │     │   Queues     │     │  (Consumers) │        │
│  └──────────────┘     └──────────────┘     └──────────────┘        │
│                              │                    │                  │
│                              ▼                    ▼                  │
│                       ┌──────────────┐    ┌──────────────┐          │
│                       │   Bull Board │    │  Dead Letter │          │
│                       │  (Dashboard) │    │    Queue     │          │
│                       └──────────────┘    └──────────────┘          │
│                                                                      │
│  Queues:                                                            │
│    - email        (critical, priority: high)                        │
│    - notifications (critical, priority: medium)                     │
│    - reports      (non-critical, priority: low)                     │
│    - cleanup      (scheduled, daily)                                │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.5 Event-Driven Communication

```typescript
// Domain Events (Internal)
UserCreatedEvent
UserUpdatedEvent
OrderPlacedEvent
PaymentReceivedEvent

// Integration Events (External/Kafka)
user.created.v1
order.placed.v1
payment.completed.v1

// Event Schema (Avro)
{
  "type": "record",
  "name": "UserCreated",
  "namespace": "com.company.events",
  "fields": [
    { "name": "eventId", "type": "string" },
    { "name": "timestamp", "type": "long" },
    { "name": "version", "type": "int" },
    { "name": "userId", "type": "string" },
    { "name": "email", "type": "string" },
    { "name": "tenantId", "type": "string" }
  ]
}
```

---

## 4. Frontend Architecture

### 4.1 Application Structure

```
apps/web/
├── src/
│   ├── app/                    # Next.js App Router
│   │   ├── (auth)/
│   │   │   ├── login/
│   │   │   └── register/
│   │   ├── (dashboard)/
│   │   │   ├── layout.tsx
│   │   │   ├── page.tsx
│   │   │   └── settings/
│   │   ├── api/               # API routes (BFF)
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── components/
│   │   ├── ui/                # shadcn/ui components
│   │   ├── forms/
│   │   ├── layouts/
│   │   └── features/
│   │       ├── users/
│   │       └── orders/
│   ├── hooks/
│   │   ├── use-auth.ts
│   │   └── use-debounce.ts
│   ├── lib/
│   │   ├── api-client.ts      # API client instance
│   │   ├── utils.ts
│   │   └── validators.ts
│   ├── stores/
│   │   ├── auth.store.ts
│   │   └── ui.store.ts
│   └── styles/
│       └── globals.css
├── public/
├── tests/
│   ├── e2e/
│   └── unit/
├── next.config.js
├── tailwind.config.js
└── tsconfig.json
```

### 4.2 State Management Strategy

```
┌─────────────────────────────────────────────────────────────────────┐
│                    State Management Architecture                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                     SERVER STATE                               │  │
│  │              (TanStack Query / React Query)                    │  │
│  │                                                                 │  │
│  │  - API responses          - Pagination state                   │  │
│  │  - Cached data            - Background refetching              │  │
│  │  - Loading/error states   - Optimistic updates                 │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                     CLIENT STATE                               │  │
│  │                       (Zustand)                                │  │
│  │                                                                 │  │
│  │  - UI state (modals, sidebars)   - User preferences           │  │
│  │  - Form state                     - Theme settings             │  │
│  │  - Wizard/multi-step flows                                     │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                     URL STATE                                  │  │
│  │              (Next.js Router / nuqs)                           │  │
│  │                                                                 │  │
│  │  - Filters         - Sort order        - Active tabs           │  │
│  │  - Search queries  - Pagination                                │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.3 Component Library Strategy

**shadcn/ui + Radix Primitives (Copy-paste approach)**

Advantages:
- Full ownership of component code
- Built on accessible Radix primitives
- Tailwind CSS integration
- No version lock-in

Component Categories:
```
ui/
├── primitives/          # From shadcn/ui
│   ├── button.tsx
│   ├── input.tsx
│   ├── dialog.tsx
│   └── ...
├── composed/            # Custom compositions
│   ├── data-table.tsx
│   ├── command-menu.tsx
│   └── file-uploader.tsx
└── patterns/            # Layout patterns
    ├── page-header.tsx
    ├── empty-state.tsx
    └── error-boundary.tsx
```

### 4.4 Testing Strategy

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Testing Pyramid                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│                         /\                                           │
│                        /  \        E2E Tests (Playwright)            │
│                       /    \       - Critical user flows             │
│                      /      \      - 5-10 tests                      │
│                     /────────\                                       │
│                    /          \    Integration Tests                 │
│                   /            \   - API + Component                 │
│                  /              \  - 50-100 tests                    │
│                 /────────────────\                                   │
│                /                  \ Unit Tests (Vitest)              │
│               /                    \- Utils, hooks, stores           │
│              /                      \- 200+ tests                    │
│             /________________________\                               │
│                                                                      │
│  Coverage Targets:                                                   │
│    - Statements: 80%                                                 │
│    - Branches: 80%                                                   │
│    - Functions: 80%                                                  │
│    - Lines: 80%                                                      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 4.5 Performance Budgets

| Metric | Budget | Measurement |
|--------|--------|-------------|
| **First Contentful Paint** | < 1.8s | Lab (Lighthouse) |
| **Largest Contentful Paint** | < 2.5s | Lab + Field |
| **Interaction to Next Paint** | < 200ms | Field (CrUX) |
| **Cumulative Layout Shift** | < 0.1 | Lab + Field |
| **Initial JS Bundle** | < 200KB gzip | Build analysis |
| **Total Page Weight** | < 1MB | Network waterfall |

---

## 5. Data Architecture

### 5.1 Database Schema Design

```sql
-- Core Schema Structure

-- Tenants (Multi-tenancy support)
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    plan VARCHAR(50) NOT NULL DEFAULT 'free',
    settings JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- Users
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    email VARCHAR(255) NOT NULL,
    email_verified BOOLEAN DEFAULT FALSE,
    name VARCHAR(255),
    avatar_url TEXT,
    role VARCHAR(50) NOT NULL DEFAULT 'member',
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    deleted_at TIMESTAMPTZ,

    UNIQUE(tenant_id, email)
);

-- Enable Row-Level Security
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.tenant_id')::uuid);

-- Indexes
CREATE INDEX idx_users_tenant_id ON users(tenant_id);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_created_at ON users(created_at DESC);

-- Audit Trail
CREATE TABLE audit_logs (
    id BIGSERIAL PRIMARY KEY,
    tenant_id UUID NOT NULL,
    user_id UUID,
    action VARCHAR(100) NOT NULL,
    resource_type VARCHAR(100) NOT NULL,
    resource_id UUID,
    old_values JSONB,
    new_values JSONB,
    ip_address INET,
    user_agent TEXT,
    request_id VARCHAR(100),
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_audit_logs_tenant_resource ON audit_logs(tenant_id, resource_type, resource_id);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at DESC);
```

### 5.2 Caching Strategy

```
┌─────────────────────────────────────────────────────────────────────┐
│                      Caching Layers                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Layer 1: CDN (CloudFlare)                                          │
│    - Static assets: immutable, 1 year                               │
│    - HTML pages: stale-while-revalidate                             │
│    - API: selective caching for public endpoints                    │
│                                                                      │
│  Layer 2: Application (LRU Cache)                                   │
│    - Feature flags: 5 minutes                                       │
│    - Configuration: 1 minute                                        │
│    - Per-instance, not shared                                       │
│                                                                      │
│  Layer 3: Distributed (Redis)                                       │
│    - Session data: 24h sliding                                      │
│    - User profiles: 1h + event invalidation                         │
│    - API responses: 5-15 minutes                                    │
│    - Rate limit counters: window-based                              │
│                                                                      │
│  Layer 4: Database (Query Cache)                                    │
│    - PostgreSQL shared_buffers                                      │
│    - Connection pooling (PgBouncer)                                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Cache Key Patterns:**
```
user:{userId}                    # User profile
user:{userId}:permissions        # User permissions
session:{sessionId}              # Session data
tenant:{tenantId}:settings       # Tenant settings
api:v1:products:list:{hash}      # API response cache
ratelimit:{ip}:{endpoint}        # Rate limiting
```

### 5.3 Search Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Search Infrastructure                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  PostgreSQL ──────┐                                                  │
│       │           │                                                  │
│       │      ┌────┴─────┐      ┌──────────────┐                     │
│       │      │ Debezium │─────▶│    Kafka     │                     │
│       │      │  (CDC)   │      │   Topics     │                     │
│       │      └──────────┘      └──────┬───────┘                     │
│       │                               │                              │
│       │                               ▼                              │
│       │                        ┌──────────────┐                     │
│       │                        │  OpenSearch  │                     │
│       │                        │   Cluster    │                     │
│       │                        └──────────────┘                     │
│       │                               │                              │
│       ▼                               ▼                              │
│  ┌──────────────────────────────────────────────────────────┐       │
│  │                    Application                            │       │
│  │  - CRUD operations → PostgreSQL                          │       │
│  │  - Search queries → OpenSearch                           │       │
│  │  - Autocomplete → OpenSearch (suggest)                   │       │
│  └──────────────────────────────────────────────────────────┘       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.4 Data Pipeline Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Analytics Pipeline                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  Operational DB ─────┐                                               │
│                      │                                               │
│  Event Streams ──────┼───▶ Fivetran ───▶ Snowflake (Raw)            │
│                      │       (EL)           │                        │
│  Third-party APIs ───┘                      │                        │
│                                             ▼                        │
│                                        dbt (Transform)               │
│                                             │                        │
│                                ┌────────────┼────────────┐           │
│                                ▼            ▼            ▼           │
│                           Staging       Marts        Metrics         │
│                                                          │           │
│                                                          ▼           │
│                                                    ┌──────────┐      │
│                                                    │  Looker  │      │
│                                                    │  Metabase│      │
│                                                    └──────────┘      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 6. Security & Compliance

### 6.1 Authentication Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                   Authentication Flow                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  User ──▶ Application ──▶ Auth0 (IdP)                               │
│              │                 │                                     │
│              │                 │ OIDC Authorization Code + PKCE      │
│              │                 ▼                                     │
│              │            ┌─────────┐                                │
│              │            │ Tokens  │                                │
│              │            │- Access │ (15 min, JWT)                  │
│              │            │- Refresh│ (7 days, rotated)              │
│              │            │- ID     │ (user claims)                  │
│              │            └─────────┘                                │
│              │                 │                                     │
│              ◀─────────────────┘                                     │
│              │                                                       │
│              ▼                                                       │
│         API Gateway                                                  │
│              │                                                       │
│              │ Validate JWT                                          │
│              │ Extract claims                                        │
│              │ Set request context                                   │
│              ▼                                                       │
│         Application                                                  │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**MFA Requirements:**
- Required for all users
- Hardware keys (FIDO2) for admin accounts
- Authenticator apps (TOTP) for standard users
- Risk-based step-up authentication

### 6.2 Authorization Model (RBAC + ABAC)

```typescript
// Role Hierarchy
const roles = {
  super_admin: ['*'],                           // All permissions
  admin: ['users:*', 'settings:*', 'reports:read'],
  manager: ['users:read', 'users:update', 'reports:*'],
  member: ['users:read:own', 'reports:read:own'],
  viewer: ['reports:read:own']
};

// Resource-based permissions
interface Permission {
  resource: string;      // e.g., 'users', 'orders'
  action: string;        // e.g., 'create', 'read', 'update', 'delete'
  scope?: 'own' | 'team' | 'all';
  conditions?: {
    tenantId?: string;
    department?: string;
    // ... other attributes
  };
}

// Policy evaluation
function canAccess(user: User, permission: Permission, resource: Resource): boolean {
  // 1. Check role-based permissions
  // 2. Apply attribute-based conditions
  // 3. Check resource ownership
  // 4. Evaluate custom policies (OPA)
}
```

### 6.3 Security Headers Configuration

```typescript
// helmet configuration
const securityHeaders = {
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'nonce-{random}'"],
      styleSrc: ["'self'", "'nonce-{random}'"],
      imgSrc: ["'self'", "data:", "https:"],
      fontSrc: ["'self'"],
      connectSrc: ["'self'", "https://api.example.com"],
      frameAncestors: ["'none'"],
      baseUri: ["'self'"],
      formAction: ["'self'"],
      upgradeInsecureRequests: []
    }
  },
  strictTransportSecurity: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true
  },
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
  frameguard: { action: 'deny' },
  noSniff: true
};
```

### 6.4 Data Protection

| Data Type | Classification | Protection |
|-----------|---------------|------------|
| Passwords | Critical | bcrypt/Argon2, never stored plain |
| API Keys | Critical | Hashed (SHA-256), partial display |
| PII (email, name) | Sensitive | AES-256-GCM at rest |
| Financial Data | Sensitive | Tokenization (Stripe) |
| Session Data | Internal | Encrypted, Redis |
| Audit Logs | Compliance | Immutable, encrypted |

### 6.5 Compliance Checklist

**SOC 2 Type II**
- [ ] Access control policies documented
- [ ] MFA enforced for all users
- [ ] Audit logging implemented
- [ ] Incident response plan established
- [ ] Vendor management process
- [ ] Annual penetration testing
- [ ] Background checks for employees

**GDPR/CCPA**
- [ ] Privacy policy published
- [ ] Consent management implemented
- [ ] Data subject request workflow
- [ ] Data processing agreements
- [ ] Data retention policies
- [ ] Right to deletion (erasure)
- [ ] Data portability export
- [ ] Breach notification process

### 6.6 Audit Logging

```typescript
interface AuditEvent {
  id: string;
  timestamp: string;

  // Actor
  actor: {
    id: string;
    type: 'user' | 'service' | 'system';
    ip: string;
    userAgent?: string;
  };

  // Action
  action: string;          // e.g., 'user.create', 'order.delete'
  resource: {
    type: string;
    id: string;
  };

  // Result
  result: 'success' | 'failure';

  // Changes
  changes?: {
    before: Record<string, unknown>;
    after: Record<string, unknown>;
  };

  // Correlation
  requestId: string;
  traceId: string;
}

// Events to capture
const auditableActions = [
  'auth.login',
  'auth.logout',
  'auth.mfa_enable',
  'user.create',
  'user.update',
  'user.delete',
  'role.assign',
  'permission.grant',
  'data.export',
  'settings.change',
  'api_key.create',
  'api_key.revoke'
];
```

---

## 7. Infrastructure & DevOps

### 7.1 Cloud Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AWS Architecture                              │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                      CloudFlare                              │    │
│  │  - CDN      - WAF       - DDoS Protection    - DNS           │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│                              ▼                                       │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │                    AWS Global Accelerator                    │    │
│  └─────────────────────────────────────────────────────────────┘    │
│                              │                                       │
│         ┌────────────────────┴────────────────────┐                 │
│         ▼                                         ▼                 │
│  ┌──────────────────┐                    ┌──────────────────┐      │
│  │   Region: US-East │                    │  Region: EU-West │      │
│  │                    │                    │    (DR/GDPR)     │      │
│  │  ┌────────────┐   │                    │  ┌────────────┐  │      │
│  │  │    ALB     │   │                    │  │    ALB     │  │      │
│  │  └─────┬──────┘   │                    │  └─────┬──────┘  │      │
│  │        │          │                    │        │         │      │
│  │  ┌─────┴──────┐   │                    │  ┌─────┴──────┐  │      │
│  │  │    EKS     │   │                    │  │    EKS     │  │      │
│  │  │  Cluster   │   │                    │  │  Cluster   │  │      │
│  │  └────────────┘   │                    │  └────────────┘  │      │
│  │        │          │                    │        │         │      │
│  │  ┌─────┴──────┐   │                    │  ┌─────┴──────┐  │      │
│  │  │   Aurora   │───┼────────────────────┼──│   Aurora   │  │      │
│  │  │  Primary   │   │   Global Database  │  │  Replica   │  │      │
│  │  └────────────┘   │                    │  └────────────┘  │      │
│  │                    │                    │                  │      │
│  │  ┌────────────┐   │                    │  ┌────────────┐  │      │
│  │  │ElastiCache │───┼────────────────────┼──│ElastiCache │  │      │
│  │  │   Redis    │   │  Global Datastore  │  │   Redis    │  │      │
│  │  └────────────┘   │                    │  └────────────┘  │      │
│  └──────────────────┘                    └──────────────────┘      │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.2 Kubernetes Architecture

```yaml
# Namespace structure
namespaces:
  - production          # Production workloads
  - staging             # Staging environment
  - monitoring          # Prometheus, Grafana, etc.
  - istio-system        # Service mesh (if adopted)
  - cert-manager        # TLS certificates
  - external-secrets    # Secrets sync from Vault

# Deployment strategy
deployment:
  strategy: RollingUpdate
  maxSurge: 25%
  maxUnavailable: 0

# Resource limits
resources:
  api:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      cpu: 2000m
      memory: 2Gi
  worker:
    requests:
      cpu: 250m
      memory: 256Mi
    limits:
      cpu: 1000m
      memory: 1Gi

# Autoscaling
hpa:
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### 7.3 CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v2
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm type-check
      - run: pnpm test:unit
      - run: pnpm test:e2e

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      - uses: github/codeql-action/analyze@v2

  build:
    needs: [test, security]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: |
            ${{ env.ECR_REGISTRY }}/app:${{ github.sha }}
            ${{ env.ECR_REGISTRY }}/app:latest

  deploy-staging:
    needs: build
    if: github.ref == 'refs/heads/main'
    environment: staging
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::xxx:role/deploy
          aws-region: us-east-1
      - run: |
          kubectl set image deployment/api \
            api=${{ env.ECR_REGISTRY }}/app:${{ github.sha }}

  deploy-production:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    environment: production
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
      - run: |
          # Blue-green deployment via Argo Rollouts
          kubectl argo rollouts set image api \
            api=${{ env.ECR_REGISTRY }}/app:${{ github.sha }}
```

### 7.4 Observability Stack

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Observability Architecture                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐               │
│  │   Metrics   │   │   Traces    │   │    Logs     │               │
│  └──────┬──────┘   └──────┬──────┘   └──────┬──────┘               │
│         │                 │                 │                        │
│         │      ┌──────────┴──────────┐      │                        │
│         │      │    OpenTelemetry    │      │                        │
│         │      │     Collector       │      │                        │
│         │      └──────────┬──────────┘      │                        │
│         │                 │                 │                        │
│         └────────────────┬┴─────────────────┘                        │
│                          │                                           │
│                          ▼                                           │
│                   ┌─────────────┐                                    │
│                   │   Datadog   │                                    │
│                   │             │                                    │
│                   │  - APM      │                                    │
│                   │  - Logs     │                                    │
│                   │  - Metrics  │                                    │
│                   │  - RUM      │                                    │
│                   │  - Synthetics│                                   │
│                   └─────────────┘                                    │
│                          │                                           │
│                          ▼                                           │
│                   ┌─────────────┐                                    │
│                   │  PagerDuty  │                                    │
│                   │  (Alerting) │                                    │
│                   └─────────────┘                                    │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

**Key Metrics:**

| Category | Metric | Alert Threshold |
|----------|--------|-----------------|
| **Availability** | Error rate | > 1% for 5 min |
| **Latency** | P95 response time | > 500ms for 5 min |
| **Saturation** | CPU utilization | > 80% for 10 min |
| **Saturation** | Memory utilization | > 85% for 5 min |
| **Traffic** | Request rate | > 2x baseline |
| **Database** | Connection pool usage | > 80% |
| **Database** | Replication lag | > 30 seconds |
| **Cache** | Hit rate | < 80% |

### 7.5 Disaster Recovery

| Metric | Target | Implementation |
|--------|--------|----------------|
| **RTO** | < 1 hour | Multi-region failover |
| **RPO** | < 5 minutes | Aurora continuous backup |

**DR Runbook:**
1. Detect failure (automated monitoring)
2. Assess impact (automated health checks)
3. Initiate failover (Route 53 health check)
4. Verify secondary region
5. Notify stakeholders
6. Post-incident review

---

## 8. Implementation Roadmap

### Phase 1: Foundation (Weeks 1-4)

**Week 1-2: Project Setup**
- [ ] Initialize monorepo with Turborepo
- [ ] Configure TypeScript, ESLint, Prettier
- [ ] Set up NestJS backend structure
- [ ] Set up Next.js frontend structure
- [ ] Configure Prisma with PostgreSQL
- [ ] Implement CI/CD pipeline (GitHub Actions)

**Week 3-4: Core Infrastructure**
- [ ] Terraform modules for AWS resources
- [ ] EKS cluster setup
- [ ] Aurora PostgreSQL deployment
- [ ] ElastiCache Redis deployment
- [ ] Basic observability (Datadog)
- [ ] Secrets management (Vault)

### Phase 2: Authentication & Authorization (Weeks 5-8)

**Week 5-6: Authentication**
- [ ] Auth0 integration
- [ ] JWT handling
- [ ] Session management
- [ ] MFA implementation
- [ ] Password policies

**Week 7-8: Authorization**
- [ ] RBAC implementation
- [ ] Permission system
- [ ] Multi-tenancy foundation
- [ ] Row-level security
- [ ] API key management

### Phase 3: Core Features (Weeks 9-16)

**Week 9-12: Backend Core**
- [ ] User management module
- [ ] Audit logging
- [ ] Background jobs (BullMQ)
- [ ] Email service
- [ ] File upload (S3)
- [ ] API documentation (OpenAPI)

**Week 13-16: Frontend Core**
- [ ] Design system (shadcn/ui)
- [ ] Authentication flows
- [ ] Dashboard layout
- [ ] User management UI
- [ ] Settings pages
- [ ] Error handling & boundaries

### Phase 4: Advanced Features (Weeks 17-24)

**Week 17-20: Search & Analytics**
- [ ] OpenSearch integration
- [ ] Real-time indexing (CDC)
- [ ] Search UI components
- [ ] Analytics dashboard
- [ ] Reporting system

**Week 21-24: Operations**
- [ ] Advanced monitoring
- [ ] Alerting rules
- [ ] Runbooks
- [ ] DR testing
- [ ] Load testing
- [ ] Security audit

### Phase 5: Production Readiness (Weeks 25-28)

- [ ] Performance optimization
- [ ] Security hardening
- [ ] Compliance documentation
- [ ] Penetration testing
- [ ] User acceptance testing
- [ ] Production deployment
- [ ] Go-live checklist

---

## 9. Project Structure

### Monorepo Structure

```
enterprise-app/
├── apps/
│   ├── api/                    # NestJS backend
│   │   ├── src/
│   │   │   ├── modules/
│   │   │   ├── shared/
│   │   │   └── main.ts
│   │   ├── test/
│   │   ├── prisma/
│   │   │   ├── schema.prisma
│   │   │   └── migrations/
│   │   ├── nest-cli.json
│   │   ├── tsconfig.json
│   │   └── package.json
│   │
│   ├── web/                    # Next.js frontend
│   │   ├── src/
│   │   │   ├── app/
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   ├── lib/
│   │   │   └── stores/
│   │   ├── public/
│   │   ├── tests/
│   │   ├── next.config.js
│   │   ├── tailwind.config.js
│   │   ├── tsconfig.json
│   │   └── package.json
│   │
│   └── worker/                 # Background job worker
│       ├── src/
│       ├── tsconfig.json
│       └── package.json
│
├── packages/
│   ├── ui/                     # Shared UI components
│   │   ├── src/
│   │   ├── tsconfig.json
│   │   └── package.json
│   │
│   ├── config/                 # Shared configs
│   │   ├── eslint/
│   │   ├── typescript/
│   │   └── tailwind/
│   │
│   ├── types/                  # Shared TypeScript types
│   │   ├── src/
│   │   └── package.json
│   │
│   └── utils/                  # Shared utilities
│       ├── src/
│       └── package.json
│
├── infrastructure/
│   ├── terraform/
│   │   ├── modules/
│   │   │   ├── networking/
│   │   │   ├── eks/
│   │   │   ├── rds/
│   │   │   └── redis/
│   │   ├── environments/
│   │   │   ├── dev/
│   │   │   ├── staging/
│   │   │   └── production/
│   │   └── backend.tf
│   │
│   └── kubernetes/
│       ├── base/
│       ├── overlays/
│       └── kustomization.yaml
│
├── docs/
│   ├── adr/                    # Architecture Decision Records
│   ├── api/                    # API documentation
│   ├── runbooks/               # Operational runbooks
│   └── onboarding/             # Developer onboarding
│
├── scripts/
│   ├── setup.sh
│   ├── deploy.sh
│   └── seed.ts
│
├── .github/
│   ├── workflows/
│   │   ├── ci.yml
│   │   ├── deploy.yml
│   │   └── security.yml
│   └── CODEOWNERS
│
├── turbo.json
├── pnpm-workspace.yaml
├── package.json
├── tsconfig.json
└── README.md
```

---

## 10. Appendix: Decision Records

### ADR-001: Framework Selection

**Status:** Accepted

**Context:** Need to select backend and frontend frameworks for enterprise web application.

**Decision:**
- Backend: NestJS
- Frontend: Next.js 14 with App Router

**Rationale:**
- NestJS provides enterprise features (DI, modules, guards) out of the box
- Strong TypeScript support in both frameworks
- Large ecosystem and community support
- Excellent documentation

**Consequences:**
- Team needs NestJS training
- Commitment to React/Next.js ecosystem

---

### ADR-002: Database Selection

**Status:** Accepted

**Context:** Need relational database with strong consistency guarantees.

**Decision:** PostgreSQL via Amazon Aurora

**Rationale:**
- Superior JSON support for flexible schemas
- Strong consistency and ACID compliance
- Excellent TypeScript ORM support (Prisma)
- Aurora provides automatic scaling and HA

**Consequences:**
- Higher cost than standard RDS
- Aurora-specific features may create lock-in

---

### ADR-003: Authentication Strategy

**Status:** Accepted

**Context:** Need secure, scalable authentication for enterprise users.

**Decision:** Auth0 as identity provider with OIDC

**Rationale:**
- Enterprise-grade security features
- Built-in MFA, SSO, compliance
- Reduced development and maintenance burden
- SOC 2 and HIPAA compliant

**Consequences:**
- Recurring SaaS cost
- Dependency on third-party service
- Need internet connectivity for auth

---

### ADR-004: Monorepo vs Polyrepo

**Status:** Accepted

**Context:** Need to decide on repository structure.

**Decision:** Monorepo with Turborepo

**Rationale:**
- Shared code without publishing packages
- Atomic commits across frontend/backend
- Simplified CI/CD
- Better code reuse and refactoring

**Consequences:**
- Larger repository size
- Need for monorepo tooling expertise
- Potential for slower CI without caching

---

### ADR-005: State Management

**Status:** Accepted

**Context:** Need frontend state management solution.

**Decision:** TanStack Query for server state, Zustand for client state

**Rationale:**
- Clear separation of concerns
- TanStack Query handles caching, deduplication automatically
- Zustand is lightweight with minimal boilerplate
- Both have excellent TypeScript support

**Consequences:**
- Team needs to understand state type distinction
- Two libraries to maintain

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2024-01-16 | Architecture Team | Initial version |

---

## Next Steps

1. **Review and approve** this architecture plan with stakeholders
2. **Set up development environment** using the project structure
3. **Begin Phase 1** implementation following the roadmap
4. **Schedule architecture review** at end of each phase
