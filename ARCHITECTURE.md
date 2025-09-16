# Architecture Principles

**Purpose**  
A short, actionable set of principles that guide system design independent of technology choices. Use them to structure ADRs, design reviews, and runbooks. Make better decisions faster. These principles explain **what to do** and **why**, with concrete fitness checks.

**Scope**  
Applies to product features, platforms, data pipelines, and internal tools. Exceptions require an ADR with rollback plans.

## Precedence (when principles conflict)
**Order:** Correctness → Security/Privacy → Reliability → Observability → Operability → Cost → Speed of change → Reuse

**Why**  
This order minimizes user harm and long‑term risk. Shipping fast but wrong or insecure creates permanent costs. We first protect truth and trust, then ensure the system stays up and diagnosable, then optimize for ease of operations and money, and only then chase speed and reuse.

**What each means**
- **Correctness**: Data integrity and invariant‑preserving behavior. Never ship known wrong results; prefer a slower right answer over a fast wrong one.  
- **Security/Privacy**: Least privilege, safe handling of secrets and PII, compliance, and auditable access. If user data is at risk, stop and fix.  
- **Reliability**: Meeting SLOs for availability and latency; graceful degradation and safe rollback paths.  
- **Observability**: The ability to detect, trace, and diagnose issues quickly (metrics, logs, traces, IDs). If we can’t see it, we can’t keep SLOs.  
- **Operability**: Ease of running the system—simple deploys, backpressure, quotas, runbooks, on‑call health.  
- **Cost**: Total cost of ownership ($/req, $/user, infra + people time). Optimize without hurting the higher priorities.  
- **Speed of change**: Lead time, batch size, flags/canaries, and reversibility that keep delivery fast and safe.  
- **Reuse**: Shared libraries/frameworks after proven need (≥3 call sites). Useful, but last.

**How to use this order**
1) Identify the impacted dimensions and **user impact/SLO risk**. Anything affecting a higher item wins.  
2) Prefer **reversible** options (flags/canaries) when trade‑offs are close.  
3) Time‑box exceptions with an **ADR** (owner, exit criteria, review date).  
4) Record the trade‑off in the ADR and link the **fitness metrics** you’ll watch.

**Examples**
- A fix increases infra cost by 15%: **Correctness > Cost** → ship fix now; open a cost‑reduction ticket.  
- Proposing a shared DB to ship faster: **Security/Privacy + Boundaries > Speed** → expose an API; no cross‑service table reads.  
- Dropping logs to save CPU: **Observability > Cost** → keep essential telemetry; optimize sampling/retention instead.  
- Deadline pressures to ship without rollback: **Reliability/Operability > Speed** → slice scope and ship behind a flag.

**Fitness**  
ADRs that cite Precedence must list: impacted dimensions, user/SLO impact, the chosen trade‑off, and the rollback plan. 

**Rule of thumb**  
*Never ship fast‑wrong or fast‑insecure. Make it right, safe, and observable; then make it cheap and fast; reuse last.*

**How to apply**  
For any design/change: pick the **3–5 most relevant** principles, state how you’ll meet the **tenets**, and attach evidence to the **fitness** metrics. Capture decisions in an ADR.

---

## 0) SLOs & users first (make reliability explicit)
*Because users feel outcomes, not components, therefore define SLOs and design to them.*

**Description**  
Define availability, latency, and durability SLOs per capability. Tie rollouts, capacity, and incident response to these targets. Document RTO/RPO and test them.

**Tenets**
- SLOs defined per endpoint/flow (**availability, p95/p99 latency, durability**).  
- **RTO/RPO** agreed, tested via failure drills.  
- **Error budgets** gate risky changes.

**Fitness**
- SLO docs exist for **100%** public endpoints; monthly SLO report; **GameDay** run ≥ **quarterly**.

---

## 1) Optimize for change over reuse
*Because requirements often change, therefore prefer designs that are easy to modify even if they duplicate code.*

**Description**  
Prefer designs that are easy to modify tomorrow over abstractions that look elegant today. Duplication is cheaper than the wrong abstraction; let patterns emerge from real usage (3+ call sites) before you framework‑ize. In reviews, ask: *Will this choice make the next change safer/faster?* Ship behind feature flags to learn in prod and refactor from evidence.

**Tenets**
- Don’t build frameworks until **3+** call sites exist.  
- Prefer **feature flags** to long‑lived branches; keep branches short and integrate behind flags.

**Fitness**
- Lead time (commit → prod) **≤ 1 day** for standard changes.

---

## 2) Evolve in production (humility over hubris)
*Because complex systems kick back, therefore ship thin, observable changes and design for reversibility.*

**Description**  
A complex system that works evolves from a simple one that works. Treat production as the primary learning environment: ship thin vertical slices, observe with telemetry, and keep changes reversible (flags, canaries, rollbacks). Favor parallel runs and shadow traffic over big‑bang cutovers.

**Tenets**
- Ship **thin vertical slices**; run new paths **in parallel** before cutover.  
- Stage rollouts (canaries) with **easy rollback**; design for **graceful retreat**.

**Fitness**
- ≥ **90%** of releases behind flags/canaries; rollback **< 10 min** median.

---

## 3) Boundaries over layers
*Because clear contracts reduce coupling, therefore model cohesive domains with explicit APIs—not shared DBs.*

**Description**  
Organize around cohesive domains with explicit APIs, not generic multi‑layer stacks. Stable contracts reduce coupling and make evolution safe; leaking internals or sharing databases creates hidden dependencies. Version APIs sparingly with deprecation plans. Count hops and N+1 calls in hot paths.

**Tenets**
- **No service** reads another service’s tables.  
- Breaking API changes require **versioning + deprecation plan**.  
- Keep hot‑path hops **≤ 2**; budget N+1.

**Fitness**
- **≥ 95%** cross‑domain calls via **HTTP/queue** (vs. DB links).

---

## 4) Evidence over fashion
*Because workloads, SLOs, and cost are the real constraints, therefore decide with measurements, not trends.*

**Description**  
Pick designs from measured workloads, SLOs, and cost—not trends. Before choosing a tool, write the workload model (RPS, payloads, p95/p99), spike with realistic data, and record results in an ADR with review dates and rollback criteria. Make performance budgets first‑class in CI.

**Tenets**
- Define workload model (**RPS, data shapes, p95/p99**) before choosing tech.  
- Run realistic spikes; record in **ADRs** with review dates + rollback criteria.  
- **Performance gates** in CI/CD: fail merges on **>10%** p95/CPU/bytes regressions.

**Fitness**
- % changes with ADR **≥ 95%**; CI perf‑gate pass rate trending **up**.

---

## 5) Data shapes drive design (compute near data)
*Because access patterns dominate cost/latency, therefore shape boundaries and algorithms around them and push compute to where bytes live.*

**Description**  
Access patterns dictate structure, boundaries, and algorithms. Model ownership by read/write shapes and change cadence; push filters/joins/aggregations to where the bytes live. Avoid broad fetch + in‑app filtering; control payloads and serialization overhead.

**Tenets**
- Express rules as **predicates/joins/group‑bys**—no post‑query scans.  
- Each hot query has a **matching index/sort**; prune unused indexes.  
- Control **payloads/serialization**; return only needed fields.

**Fitness**
- **≥ 90%** hot queries covered by **indexes**; payload p95 **≤ 64 KB**.

---

## 6) Flow control by design
*Because load is bursty and skewed, therefore make backpressure and fairness explicit.*

**Description**  
Throughput and latency collapse without explicit limits. Design bounded queues, timeouts, retries with budgets, and graceful shedding. Tune concurrency to cores/I/O; protect fairness with per‑tenant quotas and hot‑key detection. Measure queue **age** (not just length) and tail latency.

**Tenets**
- **Bounded queues**, timeouts/retries with budgets, **graceful shedding**.  
- Concurrency tuned to cores/I/O; **per‑tenant quotas** and hot‑key detection.

**Fitness**
- Queue age **p95 SLO** met; per‑tenant **P99/P999** within caps; retry budget respected.

---

## 7) Correctness before cleverness (with cache discipline)
*Because integrity beats speed‑that‑lies, therefore invariants and atomicity come first.*

**Description**  
Speed that returns the wrong answer is a bug. Make transaction boundaries and invariants explicit; use idempotency and test crash/retry paths. Treat caches as additional copies of truth with defined invalidation strategies (dogpile protection, negative caching). Track read/write amplification so derived views don’t spiral. Split a fast‑path for the common case with a verified slow‑path for correctness.

**Tenets**
- State **transaction boundaries & invariants**; idempotent commands; crash‑path tests.  
- Caching has **taxonomy + invalidation tests** (dogpile protection; negative caching).  
- Track **read/write amplification**; enforce fast‑ vs. slow‑path split.

**Fitness**
- Consistency checkers pass; amplification within budget (e.g., writes **≤ 3×** base).

---

## 8) Observability is a feature
*Because MTTR impacts customer trust, therefore instrumentation is part of “done.”*

**Description**  
If you can’t see it, you can’t operate it. Instrument every endpoint with traces, metrics, and structured logs carrying correlation IDs. Keep “golden traces” for hot flows and wire SLOs to alerting and error budgets. Dashboards and runbooks are acceptance criteria.

**Tenets**
- Every endpoint emits **traces, metrics, structured logs** with **correlation IDs**.  
- **Golden traces** maintained per hot path; **error budgets** gate feature work.

**Fitness**
- Incident **MTTR p50 < 30 min**; **100%** endpoints traced.

---

## 9) Secure & private by default
*Because trust is table stakes, therefore security and privacy are built‑in, not bolted‑on.*

**Description**  
Enforce least‑privilege IAM, short‑lived creds, encryption in transit/at rest, and audited access paths. Keep PII in designated services with reviewed data contracts, minimization, and retention policies. Treat boundary changes as security events (threat model, review, tests). Automation (linting, CI checks, secrets scanning) makes the secure path the easy path.

**Tenets**
- **Least‑privilege IAM**; no **long‑lived human creds**.  
- **PII** stays in designated services; data contracts reviewed; encryption **in transit/at rest**; **data minimization**.  
- Security reviews for **boundary changes**; automated linting/secret scanning.

**Fitness**
- **0** criticals from periodic audits; secret **age & access** baselines met.

---

## 10) Cost is a first‑class constraint
*Because efficiency compounds, therefore design and operate to TCO budgets.*

**Description**  
Make $/req and $/user visible next to latency and error rates; attribute spend to services and features. Prefer managed services when they improve SLOs or reduce TCO; otherwise justify the build/operate cost. Reduce waste by batching, compressing payloads, and eliminating unnecessary hops.

**Tenets**
- **Managed services** unless they blow SLOs or cost.  
- **Batch/async** where user latency isn’t visible.  
- Dashboard **$/req** and **$/user** next to latency.

**Fitness**
- Cost per active user trends **down** monthly; cost SLOs **met**.

---

## 11) Reliability & disaster readiness
*Because failures are normal, therefore design for graceful degradation and recovery.*

**Description**  
Architect for failure domains (AZ/region/service). Provide graceful degradation for partial outages. Back up state and practice restore. Document and test DR plans.

**Tenets**
- Multi‑AZ for stateful systems; region failover strategy documented.  
- **Graceful degradation** paths for core user journeys.  
- **Backups** tested; **restore time** measured.

**Fitness**
- **RTO/RPO** met in drills; successful **region failover** test **≥ annually**.

---

## 12) Ownership & paved road
*Because consistency lowers cognitive load, therefore default to golden paths with accountable ownership.*

**Description**  
Adopt “you build it, you run it.” Prefer a curated stack (“paved road”) with templates, libraries, and docs. Deviations require an ADR and a support plan. Every service has an owner, on‑call rotation, and clear runbooks.

**Tenets**
- Services use **templates** (logging, metrics, CI gates) by default.  
- Each service has **owner + pager**; on‑call health reviewed.  
- **ADR** required for deviating from paved road.

**Fitness**
- **≥ 90%** services on paved road; on‑call SLOs (page fatigue, time‑to‑ack) met.

---

## 13) Documentation‑as‑code
*Because shared context scales teams, therefore keep docs close to code and verify them.*

**Description**  
Version architecture diagrams (light C4), API contracts, SLOs, and runbooks with the code. Validate diagrams and OpenAPI/IDL in CI to prevent drift.

**Tenets**
- **C4‑lite** diagrams + data flows checked into repo.  
- **Contracts** (OpenAPI/IDL/schemas) validated in CI; breaking changes flagged.  
- Runbooks + dashboards are part of **“done.”**

**Fitness**
- Docs updated in the same PR for **≥ 95%** changes; schema drift alerts **= 0**.

---

# Operationalization

### ADR template
**Context → Options → Decision → Consequences → Metrics/SLOs → Rollback criteria → Review date → Owner.**

### Review checklist (1‑page)
- [ ] Pick **3–5** relevant principles for this change.  
- [ ] Link workload model (RPS, payloads, p95/p99).  
- [ ] Show evidence: traces/benchmarks/cost estimate.  
- [ ] State rollback and flag plan.  
- [ ] Update contracts/diagrams/runbooks.  
- [ ] SLO/error‑budget impact acknowledged.  
- [ ] Security/PII review if boundaries/data change.  
- [ ] Fitness metrics set & owner assigned.

### Scorecard (map metrics to dashboards)
| Principle | Primary metric(s) |
|---|---|
| 0 SLOs & users first | Availability %, p95/p99 latency, RTO/RPO drill results |
| 1 Change over reuse | Lead time, change failure rate |
| 2 Evolve in prod | % flagged/canary releases, rollback time |
| 3 Boundaries over layers | % cross‑domain via API/queue, hot‑path hop count |
| 4 Evidence over fashion | ADR coverage %, CI perf‑gate trend |
| 5 Data shapes | Index coverage %, payload p95 |
| 6 Flow control | Queue age p95, retry budget, per‑tenant P99/P999 |
| 7 Correctness | Consistency checker pass rate, amplification budget |
| 8 Observability | Endpoint trace coverage, MTTR p50 |
| 9 Security & privacy | Criticals from audits, secret age/access baseline |
|10 Cost | $/req, $/user, cost SLO adherence |
|11 Reliability/DR | RTO/RPO met, failover drill pass |
|12 Ownership/paved road | % services on paved road, on‑call health |
|13 Docs‑as‑code | % PRs updating docs/contracts, schema drift alerts |

### Exception process
- File an ADR tagged **Exception** with explicit time limit and exit criteria.  
- Create a **Principle debt** ticket linked to a measurable risk.  
- Review exceptions **monthly**.

---

**Glossary**  
- **SLO/SLA/Error budget**: Target reliability; permissible risk for change.  
- **RTO/RPO**: Recovery Time/Point Objectives.  
- **GameDay**: Planned failure drill to validate recovery.  
- **Paved road**: Curated defaults for fast, safe delivery.  
- **Golden trace**: Canonical distributed trace for a hot user flow.


