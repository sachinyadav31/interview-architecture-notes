# 99: General questions interviewers ask around this project, and system design

## A) Project-level questions

**Q: What was your role?**
A: Full-stack engineer. Backend: Spring Boot microservices (PO/POR, invoice, supplier procurement, eSourcing clarifications, the new inventory/manufacturing/logistics/operations services, tenant onboarding and Superset provisioning, ACL seeding). Frontend: Angular portals (marketplace checkout, coupons, tenant portal upgrade to 18). Plus CI/CD templates and deployments. At Inclufy also Django (SSO/tenants), FastAPI (IQ Helix) and the AI copilot.

**Q: How big was the system / team?**
A: 30+ backend services, 4 Angular apps, about 15–20 developers over time. Several tenants in production. (Give your honest numbers.)

**Q: What was the hardest problem?**
A: Pick one: (1) IQ Helix: two DB engines silently writing to SQLite, 104 Sentry events (see 16); (2) SSO: associated tenants hidden by a phantom profile (see 14); (3) tenant onboarding automation with Superset (see 11).

**Q: What would you redesign?**
A: (1) Transactional outbox for every DB-write + Kafka publish; (2) a real IdP with token exchange instead of password propagation; (3) runtime tenant registration instead of container restarts; (4) RS256 JWTs + strip internal headers at the gateway; (5) DLTs + idempotent consumers everywhere; (6) ShedLock for scheduled jobs.

**Q: How did you test?**
A: JUnit 5 + Mockito unit tests (mock repositories and Feign clients), `@WebMvcTest` for controllers, `@DataJpaTest` for repositories, Jasmine/Karma for Angular, SonarQube gates in CI. What I'd add: Testcontainers (MySQL, Kafka) integration tests and contract tests (Spring Cloud Contract/Pact) for events.

**Q: How did you monitor production?**
A: Spring Boot Actuator health endpoints, centralised logs, Sentry for the Python/React apps. I'd add Micrometer + Prometheus/Grafana, OpenTelemetry tracing with a trace ID across gateway → services → Kafka headers, and alerts on error rate, consumer lag and DLT size.

## B) Spring Boot / Java deep-dive questions tied to the code

**Q: How does `@Transactional` work? Common pitfalls?**
A: Spring creates a proxy around the bean; the proxy opens/commits/rolls back. Pitfalls: **self-invocation** (calling a `@Transactional` method from the same class skips the proxy); by default rollback only happens for **unchecked** exceptions; `private` methods aren't proxied; long transactions holding connections. Propagation: `REQUIRED` (default, join or create), `REQUIRES_NEW` (suspend and create, e.g. audit logs that must persist even on failure).

**Q: How do you solve N+1 in JPA?**
A: `JOIN FETCH` in JPQL, `@EntityGraph(attributePaths=...)`, batch fetching (`hibernate.default_batch_fetch_size`), or DTO projections. Our workflow `Process ↔ Task` used `FetchType.EAGER` both ways, which is convenient but loads more than needed. LAZY + fetch joins where needed is better.

**Q: What's `open-in-view`?**
A: It keeps the Hibernate session open during view rendering so lazy loads work in controllers and serialisation. It hides N+1 problems and holds DB connections longer. I'd set `spring.jpa.open-in-view=false` and fetch what's needed in the service layer.

**Q: How does Feign work? How do you handle failures?**
A: An interface + `@FeignClient`; Spring generates the HTTP client at runtime. We add a `RequestInterceptor` that copies tenant/user headers. For failures: timeouts (connect/read), retries with care (only idempotent calls), and a circuit breaker (Resilience4j) with a fallback, so a slow service doesn't exhaust our threads.

**Q: Circuit breaker states?**
A: CLOSED (calls flow, failures counted) → OPEN (fail fast for a wait period) → HALF_OPEN (let a few test calls through; success → CLOSED, failure → OPEN).

**Q: HashMap vs ConcurrentHashMap: where did you use the latter?**
A: Tenant datasource map, tenant cache, SSE emitter registry: all read and written by many request threads. ConcurrentHashMap gives thread-safe access without locking the whole map; `computeIfAbsent` is atomic per key.

**Q: ThreadLocal pitfalls?**
A: Thread pools reuse threads, so you must `remove()` in `finally`; values don't cross async boundaries unless you propagate them; `InheritableThreadLocal` copies only at thread creation. All of this is visible in our tenant context (see 01).

**Q: Explain the design patterns in this codebase.**
A: Factory (`IndexJobProcessorFactory`, AI provider factory), Template Method (`DefaultIndexJobProcessor`, `WorkflowUtil`), Strategy (serial vs parallel workflow, DocuSign hooks), Interceptor/Chain of Responsibility (gateway filters, HandlerInterceptors, Kafka interceptors), Decorator (`TenantAwareTaskDecorator`), Observer/pub-sub (Kafka), Adapter (LLM providers), Builder (Lombok `@Builder`, Stripe params).

## C) System design: use your project as the answer
Framework (say it out loud): **requirements → APIs → data model → high-level diagram → deep dive on 1–2 parts → scaling → failure handling → trade-offs.**

### "Design a multi-tenant SaaS" → files 00, 01, 02, 11
Talk about: tenancy model choice, tenant resolution (subdomain/header/JWT claim), isolation, per-tenant config cache, onboarding automation, noisy neighbours (rate limits per tenant), migrations across tenants.

### "Design an approval workflow system" → file 05
Entities (Process, Task, ApprovalRule), serial/parallel/quorum, an event on completion, escalation timers, optimistic locking, audit.

### "Design a notification system" → file 10
Async through a queue, templates + i18n, channels (email/SMS/push/SSE), user preferences, retries + DLT, idempotency, rate limiting, delivery tracking.

### "Design product search" → file 07
DB as source of truth, CDC/events to ES, mapping, relevance, facets, alias reindexing, pagination with `search_after`, tenant isolation.

### "Design a payment flow" → file 08
Checkout session, webhook as the source of truth, idempotency keys, a reconciliation job, a ledger table, PCI scope.

### "Design a job scheduler" → file 12
Store jobs, claim with `SKIP LOCKED` / ShedLock, at-least-once execution + idempotent handlers, retry with backoff, monitoring.

## D) Rapid-fire facts to memorise
- Spring Boot 2.4 · Java 8/11 · MySQL · Kafka on AWS MSK · Elasticsearch Java API client · Angular 18 · PrimeNG 18 · Gradle · GitLab CI · Docker · ECR
- Gateway port 8200 · tenant service 8307 · header `X-TARGET-TENANT-QUALIFIER` · topic pattern `ep.<tenant>.<topic>` · index pattern `<tenant>_<index>`
- API ACL modes: PERMISSIVE / RESTRICTIVE · super-admin bypass · URI template matching
- Workflow: SERIAL / PARALLEL · mandatory flag · the entity task holds the final outcome · `task-approved-rejected-topic`
