# Non‑Functional Requirements — Demo .NET Modular Monolith

**Context**  
Modular monolith using **Vertical Slice Architecture**, **DDD**, **CQRS (no Event Sourcing)**, **Outbox** for async integration, and **domain events** to trigger background work. Goals: **easy to change**, **easy to reason about in production**, **fast**, **reliable**, **secure**.

> Numbers below are starter targets for a demo scale. Tune in your ADR to match real workloads while keeping the **precedence** order.

---

## Performance (PERF)
**PERF‑1 – Request latency (reads)**  
**Target**: p95 ≤ **100 ms**, p99 ≤ **250 ms** for hot read endpoints at 1× baseline load.  
**Measure**: HTTP server timings; OpenTelemetry histogram per endpoint.  
**Notes**: Read model uses pre‑joined shapes; responses return only needed fields; every hot query indexed.

**PERF‑2 – Command latency (writes)**  
**Target**: p95 ≤ **150 ms**, p99 ≤ **400 ms** for standard commands (payload ≤ 4 KB).  
**Measure**: Command pipeline timer around handler; include DB round‑trip and Outbox insert.  
**Notes**: Single transaction per command; avoid chatty ORM patterns; batch derived updates async.

**PERF‑3 – Throughput**  
**Target**: Sustain **300 RPS** average and **1000 RPS** 1‑minute bursts without violating PERF‑1/2.  
**Measure**: k6/Locust load test; record SLI time series.  
**Notes**: Enable **Server GC**, minimize allocations, use response compression for text/json.

**PERF‑4 – Payload & serialization**  
**Target**: Response size p95 ≤ **64 KB**; request body p95 ≤ **32 KB**.  
**Measure**: Middleware to log sizes; histogram.  
**Notes**: System.Text.Json source‑gen where applicable.

**PERF‑5 – DB query budgets**  
**Target**: Hot reads ≤ **2 queries**; hot writes ≤ **1 query + outbox insert**; query p95 ≤ **15 ms**.  
**Measure**: EF Core interceptor to count/trace queries; SQL timing.

---

## Scalability (SCAL)
**SCAL‑1 – Horizontal scale‑out**  
**Target**: API is stateless; scales linearly to **4 instances** with ≤ **20%** efficiency loss.  
**Measure**: Load test at 1,2,4 instances; compare RPS/latency.  
**Notes**: Sessionless auth; cache is per‑request or distributed.

**SCAL‑2 – Worker throughput**  
**Target**: Background processors scale by **partition key** (e.g., tenant/aggregate); no hot‑key exceeds **2×** median processing time.  
**Measure**: Per‑partition queue metrics; p95/p99 per key.  
**Notes**: Use **IHostedService**/**BackgroundService** with bounded channels; parallelism capped by CPU/I/O.

**SCAL‑3 – Startup & readiness**  
**Target**: Cold start ≤ **5 s**; readiness becomes **true** only after DB, Outbox, and message transport are reachable.  
**Measure**: Startup probe timing; health checks.

---

## Availability (AVAIL)
**AVAIL‑1 – Service availability**  
**Target**: Monthly API availability **≥ 99.9%** for public endpoints.  
**Measure**: Synthetic checks + server SLI (success ratio).  
**Notes**: Rolling deploys; health‑based load balancer; graceful shutdown (drain ≤ **30 s**).

**AVAIL‑2 – Zero‑downtime deploys**  
**Target**: No 5xx spikes during deploys; error budget untouched.  
**Measure**: Compare error rate/latency before vs during rollout.  
**Notes**: Migrate DB with **expand/contract**; gate risky paths behind flags.

---

## Reliability (RELY)
**RELY‑1 – Outbox delivery**  
**Target**: Commit‑to‑publish delay p95 **≤ 30 s**, p99 **≤ 120 s**; delivery success **≥ 99.99%**.  
**Measure**: Timestamps on outbox row insert and publish; success ratio.  
**Notes**: Exactly‑once effects via **idempotent handlers** and **de‑dupe** by message ID.

**RELY‑2 – Retry budgets & DLQ**  
**Target**: Max **3** exponential retries per message, then DLQ; DLQ rate **≤ 0.01%** of processed.  
**Measure**: Per‑reason counters; alert on budget breach.  
**Notes**: Include jitter; poison‑message quarantine with replay tool.

**RELY‑3 – Queue age & backpressure**  
**Target**: Queue age p95 **≤ 30 s** under peak; shed/slow producers when age > **60 s**.  
**Measure**: Age histogram; producer 429/503 counters.  
**Notes**: Bounded channels; per‑tenant quotas to protect fairness.

**RELY‑4 – Crash/restart safety**  
**Target**: No message loss or duplicate side‑effects after **kill‑9** of workers.  
**Measure**: Chaos test in CI; invariants checker passes.  
**Notes**: Handlers idempotent; transactional outbox + **at‑least‑once** semantics.

---

## Security (SEC)
**SEC‑1 – Transport & secrets**  
**Target**: TLS **1.2+** end‑to‑end; secrets stored in a secret manager; **no long‑lived human creds**.  
**Measure**: TLS scanner; CI secret‑scan; infra policy checks.

**SEC‑2 – AuthN/Z**  
**Target**: JWT/OIDC with scopes/roles; **deny‑by‑default**.  
**Measure**: AuthZ tests for each endpoint; 100% of endpoints require policy.  
**Notes**: Use **[Authorize]** with policy names; require anti‑forgery for cookie flows.

**SEC‑3 – Data protection**  
**Target**: PII confined to designated slices; encryption at rest; data minimization enforced.  
**Measure**: Schema tags for PII; weekly scan for drift.  
**Notes**: Map PII to DTOs; log redaction; audit access.

**SEC‑4 – AppSec hygiene**  
**Target**: OWASP ASVS **L2** baseline; SAST/SCA gate builds; high‑severity vulns **= 0**.  
**Measure**: Pipeline reports; dependency‑update cadence.

---

## Maintainability (MAINT)
**MAINT‑1 – Change lead time**  
**Target**: Median commit→prod **≤ 1 day**.  
**Measure**: DORA metrics from CI/CD.  
**Notes**: Trunk‑based dev; **feature flags** over long‑lived branches.

**MAINT‑2 – Vertical slice boundaries**  
**Target**: Handlers own their data access; cross‑slice calls go through interfaces/domain events—**no direct table reads** across slices.  
**Measure**: Architecture tests (e.g., **NetArchTest**) enforce allowed references.

**MAINT‑3 – Complexity & size**  
**Target**: Handler cyclomatic complexity **< 15**; file length **< 300** LOC.  
**Measure**: Analyzer in CI; block on regressions > **10%**.

**MAINT‑4 – Docs as code**  
**Target**: Each slice has a **README** (purpose, endpoints, data shapes) and updated **C4‑lite** diagram in the repo.  
**Measure**: PRs touching code also touch docs **≥ 95%**.

**MAINT‑5 – Tests**  
**Target**: Unit tests for handlers and domain; integration tests for endpoints; mutation score **≥ 60%** on critical domain libs.  
**Measure**: CI test reports; mutation testing tool.

---

## Observability & Operability (OBS)
**OBS‑1 – Tracing & logging**  
**Target**: **100%** endpoints and message handlers emit **OpenTelemetry** traces with correlation IDs; structured JSON logs.  
**Measure**: Trace coverage; log schema check.  
**Notes**: Use `ActivitySource`/`ILogger` scopes; propagate `traceparent` via events.

**OBS‑2 – Golden traces**  
**Target**: Maintain golden traces for top **3** user flows and one async pipeline.  
**Measure**: Periodic re‑capture; compare spans and budgets.

**OBS‑3 – SLO dashboards & alerts**  
**Target**: Dashboards display availability, latency (p95/p99), error rate, queue age, outbox delay, $/req. Alerts tied to error budgets.  
**Measure**: Dashboard checklists; alert tests.

---

## Verification plan
- **Load & soak**: k6/Locust scenarios covering hot reads/writes and 1‑minute burst; assert PERF & AVAIL SLOs.  
- **DB profiling**: EF Core interceptor + SQL statistics to enforce PERF‑5.  
- **Chaos**: kill/restart workers during message processing; verify RELY‑4.  
- **Security**: SAST/SCA/secret‑scan in CI; authz e2e tests per endpoint.  
- **Architecture tests**: enforce MAINT‑2 references; fail build on violations.  
- **Observability checks**: unit test for `Activity` creation; smoke test for trace propagation.

## Operational notes (.NET specifics)
- Use **Minimal APIs** or thin controllers → MediatR‑style handlers per slice.  
- **Outbox**: EF Core transaction boundary; background publisher with exactly‑once via de‑dupe table/hash.  
- **Background concurrency**: `Channel<T>` or queue client with bounded capacity; exponential backoff with jitter.  
- **Health checks**: `IHealthChecksBuilder` for DB, outbox publisher, and message transport; expose `/health/ready` & `/health/live`.  
- **Config**: strongly‑typed `IOptions<T>`; validate on startup; fail fast.  
- **Analyzers**: enable nullable reference types, code‑style analyzers, and architecture tests in CI.

