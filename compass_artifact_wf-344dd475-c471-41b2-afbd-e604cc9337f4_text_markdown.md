# Enterprise-Grade Multi-Vendor E-Commerce Marketplace: Complete Architecture Guide

Building an enterprise-grade multi-vendor marketplace with microservices architecture is achievable at **zero cost** using Oracle Cloud's Always Free tier (4 ARM OCPUs, 24GB RAM) combined with free database tiers from Neon, MongoDB Atlas, and Upstash Redis. This guide provides a complete technical blueprint for a 12-15 microservices system that demonstrates FAANG-level architecture understanding while remaining 100% free to deploy—ideal for a graduate student building an impressive portfolio project.

The architecture mirrors patterns from Netflix (Zuul, Eureka, Hystrix), Uber (DOMA domain organization), Shopify (modular approach), and Spotify (squad model), adapted for a multi-vendor e-commerce context. The recommended stack combines **Next.js 14/15** for the frontend with BFF pattern, **NestJS/FastAPI** for backend services, **PostgreSQL/MongoDB** for polyglot persistence, **Kafka/RabbitMQ** for async messaging, and **Kubernetes** for orchestration—all deployable within free tier limits.

---

## Domain-driven service decomposition creates clear boundaries

The marketplace decomposes into **six bounded contexts** containing **15 microservices**, following Domain-Driven Design principles from Eric Evans and Chris Richardson's microservices.io patterns. This decomposition mirrors how Amazon evolved from a monolith in 2001 to thousands of independently deployable services.

**Sales Context** encompasses the Product Catalog Service (products, categories, variants with MongoDB for flexible schemas), Pricing Service (dynamic pricing, promotions, tax calculation), Cart Service (shopping cart with Redis for low-latency access), and Order Service (the core domain handling order lifecycle as the aggregate root). The Order Service owns the most complex business logic and serves as the integration point for the Saga pattern coordinating payments, inventory, and shipping.

**Fulfillment Context** contains the Inventory Service (stock management, reservations, multi-warehouse support) and Shipping/Logistics Service (rate calculation, carrier integration, tracking). These services communicate via events—when an order is placed, `OrderPlaced` triggers `StockReserved` from Inventory and `DeliveryPrepared` from Shipping.

**Customer Context** includes User/Identity Service (authentication, profiles, OAuth 2.0/OIDC), Vendor Management Service (seller onboarding, verification, payouts), Notification Service (email, SMS, push via multi-channel adapters), and Review/Rating Service (product reviews, moderation, aggregate ratings).

**Discovery Context** houses the Search Service (Elasticsearch-powered full-text search with facets) and Recommendation Engine (collaborative filtering, "frequently bought together"). These read-heavy services benefit from CQRS with denormalized read models.

**Billing Context** isolates the Payment Service handling PCI-compliant payment processing, refunds, and wallet functionality. This critical service requires the highest security controls and idempotency guarantees.

**Infrastructure Context** provides shared capabilities: Media Service (image/video upload, CDN distribution via Cloudflare R2) and Analytics Service (event ingestion, real-time dashboards, business intelligence).

Each service owns its database exclusively—Product Catalog uses MongoDB for flexible product attributes, Order Service uses PostgreSQL for ACID transactions, Cart Service uses Redis for sub-millisecond access, and Search Service uses Elasticsearch for full-text queries. This **polyglot persistence** pattern optimizes each service for its specific data access patterns.

---

## Next.js App Router architecture enables optimal frontend performance

The frontend leverages Next.js 14/15's App Router with a **Server Components-first approach**, where approximately 80% of components render on the server to minimize JavaScript sent to browsers. Product listings, category pages, and order history render as Server Components for SEO and performance, while interactive elements like shopping cart, checkout forms, and search filters use Client Components with the `'use client'` directive.

**Streaming with Suspense** enables progressive page rendering when aggregating data from multiple microservices. A marketplace page can simultaneously fetch products from the Product Service, inventory from the Inventory Service, and reviews from the Review Service—each wrapped in separate Suspense boundaries with skeleton loaders. This pattern reduces time-to-first-byte and provides excellent perceived performance even with multiple backend calls.

**Parallel Routes** power complex dashboards like the vendor portal, where analytics, orders, and inventory panels load independently with their own loading and error states. The file structure `app/dashboard/@analytics`, `app/dashboard/@orders`, and `app/dashboard/@inventory` creates slots that render simultaneously without blocking each other.

**Intercepting Routes** enable product quick-view modals that maintain shareable URLs. Clicking a product from a listing opens a modal overlay (`@modal/(.)products/[id]`) while preserving the browser's back button behavior and allowing direct URL access to the full product page.

The **BFF (Backend-for-Frontend) pattern** implements through Next.js Route Handlers aggregating multiple microservices into frontend-optimized responses. A single `/api/marketplace/products` endpoint fetches from Product, Inventory, Vendor, and Review services in parallel using `Promise.all()`, aggregates the data with enriched vendor information and ratings, then caches responses using `unstable_cache` with tag-based invalidation.

**State management** combines TanStack Query (React Query) for server state with Zustand for minimal client state. TanStack Query handles all API data with features like optimistic updates (immediately reflecting cart additions before server confirmation), automatic cache invalidation on mutations, and background refetching. Zustand manages UI state like cart drawer visibility and recently viewed products, persisting to localStorage via the persist middleware.

For performance, **ISR (Incremental Static Regeneration)** pre-renders popular product pages at build time with `revalidate: 3600` (hourly refresh), while on-demand revalidation via webhook from the Product Service triggers immediate updates when products change. The combination of Server Components, aggressive caching, and optimized images typically achieves **95+ Lighthouse scores**.

---

## Backend microservices implement enterprise resilience patterns

Backend services implement **Hexagonal Architecture** (Ports and Adapters), creating clear separation between domain logic and infrastructure concerns. The domain layer contains business rules, entities, and value objects with no framework dependencies. Ports define interfaces for external interactions (repositories, message publishers), while adapters implement these interfaces with specific technologies (PostgreSQL repositories, Kafka publishers).

**Technology selection** follows the principle of choosing the right tool for each service's requirements:

| Service Category | Recommended Stack | Rationale |
|-----------------|-------------------|-----------|
| API Gateway | Go (Fiber) or Node.js (Fastify) | Maximum throughput for routing |
| Business Logic Services | NestJS (TypeScript) | Enterprise patterns, DI, modularity |
| Data Processing | Python (FastAPI) | ML integration, async support |
| Payment Service | Go (Gin) | Security, performance, reliability |
| Search/Recommendations | Python (FastAPI) | ML ecosystem, Elasticsearch clients |

**Resilience patterns** form the defensive layer preventing cascading failures. **Circuit Breaker** (via Resilience4j or opossum) monitors failure rates and opens the circuit when failures exceed 50% within a sliding window, immediately failing subsequent requests rather than waiting for timeouts. The circuit transitions through Closed → Open → Half-Open states, testing recovery with limited requests before fully reopening.

**Bulkhead isolation** limits concurrent calls to each downstream service, preventing a slow service from consuming all connection pool threads. **Retry with exponential backoff** handles transient failures with configurable delays (500ms → 1s → 2s → 4s) and jitter to prevent thundering herd problems. **Timeout patterns** ensure no request waits indefinitely—2 seconds for user-facing calls, 30 seconds for background processing.

**The recommended combination order** from outermost to innermost: Retry → Circuit Breaker → Rate Limiter → Timeout → Bulkhead → actual service call. This layering ensures retries happen before circuit evaluation, rate limits apply before expensive operations, and bulkheads isolate individual calls.

---

## Inter-service communication balances synchronous and asynchronous patterns

**Synchronous communication** uses gRPC with Protocol Buffers for internal service-to-service calls, providing **2x faster serialization** and **30-40% smaller payloads** compared to JSON over REST. gRPC's HTTP/2 foundation enables multiplexing multiple requests over a single connection and bidirectional streaming for real-time updates. For browser clients, gRPC-Web with an Envoy proxy translates HTTP/1.1 requests to gRPC, though unary and server-streaming RPCs are fully supported while client-streaming is not.

**GraphQL Federation** aggregates multiple service schemas into a unified supergraph accessible through Apollo Gateway. The Product subgraph defines `type Product @key(fields: "id")` while the Review subgraph extends it with `extend type Product @key(fields: "id") { reviews: [Review!]! }`. Clients query products with their reviews in a single request, with the gateway automatically routing to appropriate services. This pattern excels for complex frontend data requirements while maintaining service independence.

**Asynchronous communication** through Apache Kafka handles event-driven flows like order processing. When an order is placed, the Order Service publishes `OrderPlaced` to Kafka, consumed by Payment Service (triggering payment processing), Inventory Service (reserving stock), and Notification Service (sending confirmation emails). This decoupling means services operate independently—Inventory Service can process stock reservations even if Notification Service is temporarily down.

**The Saga pattern** coordinates distributed transactions across services. For order processing, **orchestration** (via Temporal or Camunda) provides clearer flow visibility: the Order Saga Orchestrator sequentially calls Payment Service, waits for success, calls Inventory Service, and upon any failure executes compensating transactions in reverse (refund payment → release inventory → cancel order). **Choreography** suits simpler flows where services react to events without central coordination, but becomes difficult to debug with many participants.

**The Outbox pattern** ensures atomic updates between database and message broker. Rather than directly publishing to Kafka (risking inconsistency if either operation fails), services write events to an outbox table within the same database transaction as business data. Debezium captures these inserts via Change Data Capture and publishes to Kafka, guaranteeing at-least-once delivery with database-event consistency.

---

## API Gateway and service mesh provide traffic management

**Kong Gateway** (open-source under Apache 2.0) serves as the API Gateway, installed via Helm in DB-less mode for simplicity. Kong handles authentication (JWT validation, OAuth 2.0 token introspection), rate limiting (100 requests/minute per IP with Redis-backed storage), request transformation (adding headers, modifying bodies), and caching (5-minute TTL for product listings). The declarative configuration stored in Git enables GitOps workflows—all routing rules version-controlled and applied automatically.

**Traefik** presents a lightweight alternative with first-class Kubernetes integration, serving as the default ingress in K3s. Its IngressRoute CRD enables path-based routing (`/api/products → product-service`, `/api/orders → order-service`) with middleware chains for authentication, rate limiting, and CORS. Traefik's automatic HTTPS via Let's Encrypt and native Gateway API support (v1.4.0) simplify TLS certificate management.

**Service mesh** adoption should be deliberate. With fewer than 10-15 services, Kubernetes-native service discovery and network policies provide sufficient functionality without the operational overhead of sidecar proxies. When mTLS between all services becomes a requirement (compliance, zero-trust architecture), **Linkerd** offers the lowest resource footprint—approximately **10-20MB memory per sidecar** versus Istio's 50-100MB, with sub-millisecond latency overhead versus Istio's 2-5ms.

If adopting a service mesh, Linkerd's default-on mTLS, lightweight Rust-based proxy, and simpler operational model make it preferable for resource-constrained environments. Istio's extensive feature set (traffic mirroring, fault injection, detailed telemetry) justifies its overhead only for complex multi-cluster deployments with sophisticated traffic management requirements.

---

## Security architecture implements defense in depth

**Authentication** implements OAuth 2.0 Authorization Code flow with PKCE for browser clients, eliminating the need to store client secrets in JavaScript. Access tokens (JWT, RS256-signed, 15-30 minute expiry) contain minimal claims: `sub` (user ID), `scope` (permissions), `roles`, and custom claims like `vendor_id`. Refresh tokens (7-14 day expiry) stored in HttpOnly cookies with SameSite=Strict enable silent token renewal without exposing credentials to JavaScript.

**Keycloak** (self-hosted, open-source) provides the identity provider with full OAuth 2.0/OIDC support, user federation, MFA, and fine-grained authorization. For managed simplicity, **Auth0's free tier** (7,000 monthly active users) offers equivalent functionality with social login integrations and breached password detection.

**Token refresh rotation** issues a new refresh token with each access token refresh, immediately invalidating the previous refresh token. This limits the damage if a refresh token is compromised—attackers can only use it once before it's invalidated. Implementing refresh token families allows detecting reuse attacks: if a previously-rotated token is used, the entire family is invalidated, forcing re-authentication.

**Authorization** implements RBAC with hierarchical roles: platform_admin (all permissions), vendor_admin (vendor-scoped operations), vendor_staff (limited to products/orders within vendor), customer (own orders, reviews). For complex policies like "vendors can only modify their own products" or "support staff can access orders only during business hours," **Open Policy Agent (OPA)** evaluates Rego policies as a sidecar in each service. OPA decisions execute in microseconds from locally-cached policies, avoiding network calls for authorization checks.

**Network policies** in Kubernetes implement default-deny ingress, allowing traffic only from explicitly permitted sources. The Order Service accepts connections only from the API Gateway and internal services like Inventory and Payment, blocking direct external access. Combined with mTLS from a service mesh, this creates a zero-trust network where every connection is authenticated and encrypted.

**Secrets management** through **External Secrets Operator** synchronizes secrets from external providers (AWS Secrets Manager, HashiCorp Vault, GCP Secret Manager) into Kubernetes Secrets. For GitOps workflows, **Sealed Secrets** encrypt secrets with a cluster-specific public key, allowing encrypted secrets to be safely committed to Git and decrypted only within the target cluster.

---

## Kubernetes orchestration enables production-grade deployment

The Kubernetes architecture deploys on **Oracle Cloud OKE** using the Always Free tier: 4 ARM Ampere A1 OCPUs and 24GB RAM configured as 2 worker nodes with 2 OCPUs and 12GB RAM each. This provides sufficient resources for 12-15 microservices with proper resource limits.

**Deployments** manage stateless services with rolling update strategy (`maxSurge: 1, maxUnavailable: 0`) ensuring zero-downtime deployments. Each pod specifies resource requests (guaranteed allocation) and limits (maximum allowed), with typical microservices requesting 200-300MB RAM and 100-150m CPU. **StatefulSets** manage databases and message brokers requiring stable network identities and persistent storage.

**Horizontal Pod Autoscaler (HPA)** scales deployments based on CPU utilization (target 70%) and custom metrics from Prometheus. The scaling behavior includes stabilization windows (300 seconds for scale-down) and gradual policies (10% reduction per minute) to prevent thrashing during traffic fluctuations.

**Pod Disruption Budgets** ensure minimum availability during voluntary disruptions like node maintenance or cluster upgrades. Setting `minAvailable: 2` for critical services guarantees at least 2 replicas remain running regardless of disruption causes.

**Ingress** with NGINX Ingress Controller routes external traffic to services based on hostname and path. TLS termination uses certificates from cert-manager with Let's Encrypt, automatically provisioning and renewing certificates. Annotations configure NGINX behavior: proxy body size limits for file uploads, SSL redirects, and connection timeouts.

**Configuration management** separates environment-specific values using Kustomize overlays. The base directory contains common manifests, while overlays for development, staging, and production apply patches for replicas, resource limits, and ConfigMap values. Helm charts package complex deployments with templated values, enabling `helm install ecommerce ./charts/ecommerce -f values-production.yaml`.

---

## The free deployment stack costs exactly zero dollars monthly

Oracle Cloud's **Always Free tier** provides the Kubernetes foundation: 4 ARM Ampere A1 OCPUs with 24GB RAM, 200GB block storage, 10TB monthly outbound transfer, and a flexible load balancer. OKE Basic cluster management is free (Enhanced clusters cost $0.10/hour). The critical consideration: instances with less than 20% CPU utilization for 7 days may be reclaimed, so include health check endpoints that maintain minimal activity.

**Database allocation** distributes across multiple free tiers:

| Database | Service | Free Tier | Storage |
|----------|---------|-----------|---------|
| Neon (PostgreSQL) | Users, Orders, Payments | 100 CU-hours/month | 0.5GB × 3 projects |
| MongoDB Atlas | Product Catalog, Reviews | Shared cluster | 512MB |
| Upstash Redis | Sessions, Cache, Rate Limiting | 500K commands/month | 256MB |

Neon's serverless PostgreSQL with database branching supports 3 separate projects for isolating user data, transactional data, and inventory data. The 100 compute-unit hours translates to approximately 400 hours at 0.25 CU (sufficient for always-on small workloads). MongoDB Atlas M0 provides 512MB for flexible document storage ideal for product catalogs with varied attributes.

**Frontend hosting** uses **Cloudflare Pages** with unlimited bandwidth (the only major platform offering this), 500 builds per month, and built-in DDoS protection. Cloudflare's global edge network ensures sub-50ms latency worldwide. For Next.js with server-side rendering, Vercel's free tier (100GB bandwidth, 100 deployments/day) or Cloudflare Workers integration handles SSR requirements.

**Object storage** through **Cloudflare R2** provides 10GB with zero egress fees—a significant advantage over AWS S3 or GCS where egress costs often exceed storage costs. Product images, vendor assets, and backup files store in R2 with S3-compatible API access.

**Container registry** via **GitHub Container Registry (ghcr.io)** currently provides free storage and bandwidth for both public and private images, with GitHub promising one month's notice before any billing changes.

**Message queue** self-hosts **RabbitMQ** within the Kubernetes cluster, consuming approximately 300MB RAM. For lower resource usage, **NATS** requires only 50MB RAM while providing pub/sub and queue semantics.

The complete resource allocation for 12 microservices fits comfortably within 8GB RAM (leaving 16GB headroom for scaling and system overhead), with total monthly cost of **$0**.

---

## Observability stack provides metrics, logs, and traces

The **Prometheus + Grafana + Loki + Tempo** stack delivers enterprise-grade observability within free tier constraints.

**Prometheus** scrapes metrics from all services via ServiceMonitor CRDs, storing time-series data with 7-day retention. Custom metrics instrument business KPIs: orders per minute, cart abandonment rate, payment success rate. PromQL queries power alerting rules: `sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) > 0.05` triggers when error rate exceeds 5%.

**Grafana** visualizes metrics with pre-built dashboards for Kubernetes cluster health, per-service RED metrics (Rate, Errors, Duration), and business intelligence. Alerts route to Slack via Alertmanager with configurable severity levels and escalation policies.

**Grafana Loki** aggregates logs without full-text indexing, storing only metadata labels. This approach dramatically reduces storage costs compared to Elasticsearch while enabling fast queries filtered by labels like `{namespace="ecommerce", app="order-service"}`. Fluent Bit daemonsets collect container logs, enriching with Kubernetes metadata (pod name, namespace, labels) before forwarding to Loki.

**Grafana Tempo** stores distributed traces with OpenTelemetry instrumentation. Application SDKs (@opentelemetry/sdk-node for Node.js, opentelemetry-instrumentation for Python) automatically instrument HTTP clients, database queries, and message queue operations. Traces correlate with logs and metrics via TraceID, enabling drill-down from a dashboard alert to the specific request's full trace and related log entries.

The complete observability stack requires approximately **2.6GB RAM**: Prometheus (1GB), Grafana (256MB), Loki (512MB), Tempo (512MB), plus OpenTelemetry Collector (256MB). Fluent Bit daemonsets add 128MB per node.

---

## CI/CD pipelines enable continuous deployment

**GitHub Actions** provides 2,000 free minutes monthly for private repositories (unlimited for public). The pipeline structure implements:

1. **Unit tests** run in parallel via matrix strategy across all services
2. **Contract tests** with Pact verify API compatibility between services
3. **Container builds** use multi-stage Dockerfiles reducing image sizes by 50-80%
4. **Security scanning** with Trivy blocks deployments containing CRITICAL/HIGH vulnerabilities
5. **GitOps deployment** updates image tags in a separate manifests repository

**ArgoCD** monitors the manifests repository and automatically syncs changes to the cluster. The declarative Application CRD specifies source repository, target namespace, and sync policy. Automated sync with `prune: true` and `selfHeal: true` ensures the cluster always matches Git state—manual changes are automatically reverted.

**Progressive delivery** with Flagger automates canary deployments. New versions receive 10% traffic initially, with automatic promotion to 50% then 100% based on success rate (>99%) and latency (p95 <500ms) metrics. Failed canaries automatically rollback without manual intervention.

**Testing strategy** follows the pyramid: 60-70% unit tests (Jest, pytest), 20-30% integration tests (Testcontainers for real databases), 5-10% E2E tests (Playwright). **Contract testing with Pact** prevents integration failures by verifying consumer expectations against provider implementations, with Pact Broker storing contracts and enabling "can-I-deploy" CI checks.

---

## MNC patterns inform architectural decisions

**Netflix's** contributions include Zuul (API Gateway), Eureka (service discovery), and Hystrix (circuit breaker, now in maintenance—use Resilience4j). Their Chaos Monkey philosophy of "failing constantly to avoid failure" justifies implementing chaos engineering practices: randomly terminating pods, injecting latency, and simulating zone failures during game days.

**Uber's DOMA** (Domain-Oriented Microservice Architecture) addresses the challenge of managing 2,200 microservices by organizing them into 70 domains with gateway entry points. For the marketplace, this translates to defining clear domain boundaries (Catalog, Orders, Payments) with gateway services that provide stable interfaces while allowing internal restructuring.

**Shopify's modular monolith** approach reminds that microservices add distributed system complexity—consider starting with a well-structured monolith using Packwerk for boundary enforcement, extracting to microservices only when team size or scaling demands require it. Their pods architecture for multi-tenant scaling (sharding by shop_id) applies directly to marketplace vendor isolation.

**Amazon's two-pizza teams** (5-10 people) own services end-to-end from development through operations. This organizational pattern creates clear ownership and accountability, with each team maintaining full autonomy over their service's technology choices and deployment cadence.

**DoorDash's** journey from Python 2 monolith to 1000+ microservices emphasizes standardization: unified caching libraries (Caffeine for local, Redis for distributed), consistent resilience patterns across services, and service mesh (Envoy-based) for observability and traffic management at scale.

---

## Implementation roadmap spans twelve weeks

**Weeks 1-2: Foundation**
- Set up Oracle Cloud account and OKE cluster
- Configure Neon, MongoDB Atlas, Upstash accounts
- Deploy Prometheus/Grafana stack
- Create GitHub repository with monorepo structure (Nx)

**Weeks 3-4: Core services**
- Implement User/Identity Service with Keycloak integration
- Build Product Catalog Service with MongoDB
- Create basic Next.js frontend with auth flow

**Weeks 5-6: Transaction services**
- Implement Cart Service with Redis
- Build Order Service with Saga orchestration
- Integrate Payment Service (Stripe test mode)

**Weeks 7-8: Supporting services**
- Deploy Inventory Service with reservation logic
- Implement Notification Service (email via SendGrid free tier)
- Build Search Service with Elasticsearch

**Weeks 9-10: Advanced features**
- Add Vendor Management with onboarding flow
- Implement Review/Rating Service
- Build vendor dashboard with analytics

**Weeks 11-12: Production readiness**
- Configure ArgoCD for GitOps deployment
- Set up Flagger for canary deployments
- Implement comprehensive monitoring and alerting
- Load testing with k6
- Documentation and ADRs

---

## Key interview talking points demonstrate depth

When discussing this project in FAANG interviews, emphasize:

**Architecture decisions**: "I chose event sourcing for the Order aggregate because e-commerce requires complete audit trails. Every state change—order created, payment processed, item shipped—is stored as an immutable event, enabling both compliance requirements and powerful analytics like 'show me all orders that were partially refunded after delivery.'"

**Trade-offs**: "I initially considered a service mesh but opted for Kubernetes-native networking with explicit network policies. With 12 services, Linkerd's 10-20MB per-pod overhead would consume 240MB+ that's better allocated to application logic. I documented this decision in an ADR with clear criteria for when to revisit—specifically, when we exceed 30 services or require cross-cluster service discovery."

**Failure handling**: "The Order Service implements the Saga pattern with orchestration via a state machine. If payment succeeds but inventory reservation fails, compensating transactions execute in reverse: release inventory, refund payment, cancel order. Each step is idempotent—processing the same compensation twice produces the same result, handling message redelivery scenarios."

**Observability**: "I implemented distributed tracing with OpenTelemetry, correlating traces with logs via TraceID. When an alert fires for high checkout latency, I can drill from the Grafana dashboard to the specific slow trace, identify the bottleneck span (database query, external API call), and view correlated logs—all within seconds."

**Scale considerations**: "The current architecture handles approximately 100 requests/second within free tier limits. Scaling to 10,000 RPS requires horizontal pod autoscaling, read replicas for databases, and potentially CQRS with separate read models. I've designed the event-driven architecture to support this evolution without significant refactoring."

This project demonstrates mastery of microservices patterns, Kubernetes orchestration, event-driven architecture, and modern frontend development—precisely the skills valued at top-tier technology companies. The free deployment approach shows resourcefulness and practical engineering judgment, while the comprehensive observability and security architecture demonstrate production-readiness thinking beyond typical portfolio projects.