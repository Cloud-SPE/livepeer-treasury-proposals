# NaaP MVP: GitHub Project Plan (FINAL)
## Timeline: January 6 - April 10, 2025 (7 Two-Week Sprints)
## Team: 2 Senior Full-Stack Engineers

---

## Capacity Planning

### Per Sprint (2 weeks):
- **2 engineers × 35 hours/week × 2 weeks = 140 total hours**
- **Sprint planning: 2 hours (1 hour/week)**
- **Net development capacity: 138 hours per sprint**

### Buffer Strategy:
- **P0 tasks:** 70% of sprint capacity (~97 hours)
- **P1 tasks:** 20% of sprint capacity (~28 hours)
- **P2 tasks:** 10% of sprint capacity (~13 hours)
- **Hard deadline April 10:** No Epic shifting allowed

---

## Components & Technology Stack

| Component | Description | Tech Stack |
|-----------|-------------|------------|
| **go-livepeer** | Gateway/Orchestrator modifications | Go, Trickle Protocol |
| **AI Job Tester** | Synthetic workload generator | Go (existing app, needs mods) |
| **Test Gateway** | Cloud SPE go-livepeer gateway for orchestrator tests | Go, go-livepeer |
| **Network Scraper** | Metadata collector from orchestrators | Python/Go, HTTP |
| **Kafka** | Event streaming (existing) | Apache Kafka, Kafka Connect |
| **ClickHouse** | Storage, transformation, serving | ClickHouse, SQL |
| **dbt** | Transformation logic | dbt Core, SQL |
| **API Service** | Public REST API | FastAPI or Go, OpenAPI |
| **Metabase** | Public dashboard | Metabase |
| **Infrastructure** | Hosting, monitoring, CI/CD | Docker, Terraform, Prometheus |

---

# Epic 1: Foundation & Infrastructure Bootstrap
**Dates:** January 6-17, 2025 (Sprint 1)  
**Goal:** Establish infrastructure, audit existing systems, define MVP scope

**Capacity:** 138 hours | **Allocated:** 130 hours

## Tasks

| ID | Task | Component | Priority | Size | Hours | Assignee | Deliverable |
|----|------|-----------|----------|------|-------|----------|-------------|
| 1.1 | Audit existing Kafka topics & schemas | Kafka | P0 | M | 12 | Eng 1 | Kafka inventory doc |
| 1.2 | Document current event flow (go-livepeer → Kafka) | go-livepeer | P0 | M | 10 | Eng 1 | Data flow diagram |
| 1.3 | Provision ClickHouse cluster (staging + prod) | ClickHouse | P0 | L | 20 | Eng 2 | ClickHouse URLs + access |
| 1.4 | Set up Kafka Connect with ClickHouse Sink | Kafka | P0 | L | 18 | Eng 2 | Working data pipeline |
| 1.5 | Create GitHub project structure | Infrastructure | P0 | S | 6 | Eng 1 | Repo with folders/README |
| 1.6 | Define final MVP metrics catalog | Documentation | P0 | M | 12 | Both | `METRICS.md` |
| 1.7 | Prioritize data gaps from analysis | Documentation | P0 | M | 10 | Eng 1 | `DATA_GAPS.md` |
| 1.8 | Set up CI/CD pipelines (GitHub Actions) | Infrastructure | P1 | M | 14 | Eng 2 | CI/CD workflows |
| 1.9 | Initialize dbt project skeleton | dbt | P1 | S | 8 | Eng 1 | dbt project structure |
| 1.10 | Set up monitoring stack (Prometheus/Grafana) | Infrastructure | P1 | M | 12 | Eng 2 | Basic monitoring |
| 1.11 | Document architecture (current + target) | Documentation | P0 | S | 8 | Eng 1 | `ARCHITECTURE.md` |

**Sprint Planning:** 2 hours (both engineers)

### Deliverables:
- ✅ ClickHouse staging + prod environments live
- ✅ Kafka → ClickHouse pipeline functional
- ✅ GitHub repo with project structure
- ✅ `METRICS.md` (frozen MVP metrics)
- ✅ `DATA_GAPS.md` (prioritized fixes)
- ✅ `ARCHITECTURE.md` (documented)
- ✅ dbt project initialized
- ✅ CI/CD pipelines operational
- ✅ Basic monitoring deployed

### Exit Criteria:
- [ ] Both engineers can deploy to staging
- [ ] Events flowing Kafka → ClickHouse verified
- [ ] MVP metrics list approved
- [ ] Sprint 2 tasks fully scoped

---

# Epic 2: Data Quality Fixes (Critical Path)
**Dates:** January 20 - January 31, 2025 (Sprint 2)  
**Goal:** Fix critical data attribution gaps in go-livepeer events

**Capacity:** 138 hours | **Allocated:** 136 hours

## Tasks

| ID | Task | Component | Priority | Size | Hours | Assignee | Deliverable |
|----|------|-----------|----------|------|-------|----------|-------------|
| 2.1 | **Add GPU_ID to ai_stream_status events** | go-livepeer | P0 | L | 24 | Eng 1 | PR + tests |
| 2.2 | **Populate gateway field in all events** | go-livepeer | P0 | M | 16 | Eng 1 | PR + tests |
| 2.3 | **Add region to orchestrator_info** | go-livepeer | P0 | M | 14 | Eng 1 | PR + tests |
| 2.4 | Deploy go-livepeer changes to test gateway | Test Gateway | P0 | S | 6 | Eng 1 | Deployed gateway |
| 2.5 | Design network scraper architecture | Network Scraper | P0 | S | 8 | Eng 2 | Design doc |
| 2.6 | Implement orchestrator metadata scraper | Network Scraper | P0 | L | 20 | Eng 2 | Python/Go service |
| 2.7 | Create ClickHouse raw events schema v1 | ClickHouse | P0 | M | 14 | Eng 2 | SQL DDL + migration |
| 2.8 | Configure Kafka topics for new event fields | Kafka | P0 | S | 6 | Eng 2 | Kafka config |
| 2.9 | Validate enriched events in ClickHouse | ClickHouse | P0 | M | 12 | Eng 2 | Data validation tests |
| 2.10 | Update event schema documentation | Documentation | P0 | S | 6 | Eng 1 | Updated schema docs |
| 2.11 | Deploy network scraper to staging | Network Scraper | P0 | S | 8 | Eng 2 | Running scraper |

**Sprint Planning:** 2 hours

### Deliverables:
- ✅ **CRITICAL:** GPU_ID in all new events (code deployed)
- ✅ **CRITICAL:** Gateway field populated (not empty)
- ✅ **CRITICAL:** Region data in orchestrator_info
- ✅ Network scraper deployed and running
- ✅ ClickHouse schema v1 with new fields
- ✅ Data validation tests passing
- ✅ Updated schema documentation

### Exit Criteria:
- [ ] **BLOCKER:** 100% of test gateway events have GPU_ID
- [ ] Gateway field non-empty in all events
- [ ] Region data present and accurate
- [ ] Network scraper collecting metadata every 5 min
- [ ] Validation tests green for 48+ hours

### Risk Mitigation:
- ⚠️ **Risk:** go-livepeer changes are complex
- **Mitigation:** Eng 1 focused 100% on go-livepeer (64 hours), Eng 2 handles infrastructure

---

# Epic 3: Core Analytics & Metrics
**Dates:** February 3-14, 2025 (Sprint 3)  
**Goal:** Implement dbt transformations for core MVP metrics

**Capacity:** 138 hours | **Allocated:** 134 hours

## Tasks

| ID | Task | Component | Priority | Size | Hours | Assignee | Deliverable |
|----|------|-----------|----------|------|-------|----------|-------------|
| 3.1 | Design dbt project structure (staging/mart) | dbt | P0 | S | 8 | Eng 1 | dbt structure doc |
| 3.2 | Implement staging models (raw → staging) | dbt | P0 | L | 18 | Eng 1 | dbt staging models |
| 3.3 | **Metric: Output FPS** | dbt | P0 | M | 14 | Eng 1 | dbt model + tests |
| 3.4 | **Metric: Prompt-to-First-Frame Latency** | dbt | P0 | L | 20 | Eng 1 | dbt model + tests |
| 3.5 | **Metric: Jitter Coefficient** | dbt | P0 | M | 16 | Eng 2 | dbt model + tests |
| 3.6 | **Metric: Inference Failure Rate** | dbt | P0 | M | 14 | Eng 2 | dbt model + tests |
| 3.7 | **Metric: Network Bandwidth (Up/Down)** | dbt | P0 | M | 12 | Eng 2 | dbt model + tests |
| 3.8 | Implement mart models (final metrics tables) | dbt | P0 | L | 16 | Eng 1 | dbt mart models |
| 3.9 | Create ClickHouse materialized views | ClickHouse | P1 | M | 10 | Eng 2 | SQL DDL |
| 3.10 | Set up dbt CI tests | Infrastructure | P1 | S | 6 | Eng 2 | dbt tests in CI |

**Sprint Planning:** 2 hours

### Deliverables:
- ✅ dbt project with staging → mart layers
- ✅ **5 core metrics calculated:**
  - Output FPS ✅
  - Prompt-to-First-Frame Latency ✅
  - Jitter Coefficient ✅
  - Inference Failure Rate ✅
  - Network Bandwidth ✅
- ✅ All metrics with dbt tests (>80% coverage)
- ✅ Materialized views for performance
- ✅ dbt CI pipeline operational

### Exit Criteria:
- [ ] All 5 P0 metrics compute correctly
- [ ] dbt tests pass in CI
- [ ] Sample queries validated against raw data
- [ ] Metric documentation complete

### Parallel Work:
- Eng 1: dbt models (Output FPS, Latency, Mart models)
- Eng 2: dbt models (Jitter, Failure Rate, Bandwidth) + ClickHouse optimization

---

# Epic 4: Load Test Infrastructure
**Dates:** February 17-28, 2025 (Sprint 4)  
**Goal:** Deploy AI Job Tester + test datasets for synthetic workloads

**Capacity:** 138 hours | **Allocated:** 136 hours

## Tasks

| ID | Task | Component | Priority | Size | Hours | Assignee | Deliverable |
|----|------|-----------|----------|------|-------|----------|-------------|
| 4.1 | Audit existing AI Job Tester capabilities | AI Job Tester | P0 | S | 6 | Eng 1 | Capabilities doc |
| 4.2 | Design load test architecture | AI Job Tester | P0 | M | 10 | Eng 1 | Architecture doc |
| 4.3 | Create test dataset repository (GitHub) | AI Job Tester | P0 | S | 6 | Eng 1 | GitHub repo |
| 4.4 | **Dataset: Good prompts v1** (streamdiffusion) | AI Job Tester | P0 | M | 12 | Eng 1 | JSON dataset |
| 4.5 | **Dataset: Random prompts v1** | AI Job Tester | P0 | M | 12 | Eng 1 | JSON dataset |
| 4.6 | Modify AI Job Tester for scheduled tests | AI Job Tester | P0 | L | 24 | Eng 2 | Enhanced Go app |
| 4.7 | Add `is_test_load` flag to events | AI Job Tester | P0 | S | 6 | Eng 2 | Event schema update |
| 4.8 | Implement test scheduler (cron/systemd) | AI Job Tester | P0 | M | 14 | Eng 2 | Scheduler config |
| 4.9 | Verify test gateway data capture | Test Gateway | P0 | M | 12 | Eng 1 | Validation tests |
| 4.10 | Deploy AI Job Tester to staging | AI Job Tester | P0 | M | 12 | Eng 2 | Running service |
| 4.11 | Validate test results in ClickHouse | ClickHouse | P0 | M | 14 | Eng 1 | Data validation |
| 4.12 | Create internal test dashboard | Metabase | P1 | S | 8 | Eng 2 | Metabase dashboard |

**Sprint Planning:** 2 hours

### Deliverables:
- ✅ AI Job Tester enhanced with scheduling
- ✅ Test dataset repository with 2 datasets
- ✅ Automated test execution (every 10 min)
- ✅ Test results flowing to ClickHouse
- ✅ `is_test_load` flag distinguishes synthetic tests
- ✅ Internal dashboard showing test coverage

### Exit Criteria:
- [ ] AI Job Tester running scheduled tests
- [ ] Test results identifiable in ClickHouse
- [ ] At least 2 workflows tested (streamdiffusion-sdxl + 1 other)
- [ ] Test execution reliable for 72+ hours
- [ ] Dataset contribution docs written

### Parallel Work:
- Eng 1: Test datasets + validation
- Eng 2: AI Job Tester modifications + deployment

---

# Epic 5: Public API Development
**Dates:** March 3-14, 2025 (Sprint 5)  
**Goal:** Build and deploy public REST API for metrics access

**Capacity:** 138 hours | **Allocated:** 134 hours

## Tasks

| ID | Task | Component | Priority | Size | Hours | Assignee | Deliverable |
|----|------|-----------|----------|------|-------|----------|-------------|
| 5.1 | Design API specification (OpenAPI 3.0) | API Service | P0 | M | 12 | Eng 1 | `openapi.yaml` |
| 5.2 | Set up API service framework (FastAPI/Go) | API Service | P0 | M | 14 | Eng 1 | API skeleton |
| 5.3 | **Endpoint: GET /gpu/metrics** | API Service | P0 | L | 20 | Eng 1 | Endpoint + tests |
| 5.4 | **Endpoint: GET /network/demand** | API Service | P0 | L | 18 | Eng 2 | Endpoint + tests |
| 5.5 | **Endpoint: GET /sla/compliance** | API Service | P0 | L | 18 | Eng 2 | Endpoint + tests |
| 5.6 | **Endpoint: GET /datasets** | API Service | P1 | M | 12 | Eng 1 | Endpoint + tests |
| 5.7 | Implement rate limiting (100 req/hour) | API Service | P0 | M | 12 | Eng 2 | Rate limiter |
| 5.8 | Implement response caching (Redis/in-mem) | API Service | P1 | M | 10 | Eng 2 | Cache layer |
| 5.9 | Deploy API service to staging | API Service | P0 | M | 10 | Eng 1 | Staging deployment |
| 5.10 | Load test API endpoints | Infrastructure | P1 | S | 6 | Eng 2 | Load test report |

**Sprint Planning:** 2 hours

### Deliverables:
- ✅ Public API deployed to staging
- ✅ **4 endpoints functional:**
  - `/gpu/metrics` ✅
  - `/network/demand` ✅
  - `/sla/compliance` ✅
  - `/datasets` ✅
- ✅ OpenAPI spec published
- ✅ Rate limiting active (100 req/hour)
- ✅ Response caching implemented
- ✅ Load test results <500ms (p95)

### Exit Criteria:
- [ ] All P0 endpoints passing integration tests
- [ ] API performance <500ms (p95)
- [ ] Rate limiting tested and working
- [ ] OpenAPI spec auto-generated from code

### Parallel Work:
- Eng 1: OpenAPI spec + /gpu/metrics + /datasets
- Eng 2: /network/demand + /sla/compliance + rate limiting

---

# Epic 6: Public Dashboard & Production Prep
**Dates:** March 17-28, 2025 (Sprint 6)  
**Goal:** Launch public Metabase dashboard and prepare production deployment

**Capacity:** 138 hours | **Allocated:** 136 hours

## Tasks

| ID | Task | Component | Priority | Size | Hours | Assignee | Deliverable |
|----|------|-----------|----------|------|-------|----------|-------------|
| 6.1 | Provision Metabase instance (prod) | Metabase | P0 | M | 12 | Eng 2 | Metabase URL |
| 6.2 | Configure Metabase ClickHouse connection | Metabase | P0 | S | 4 | Eng 2 | DB connection |
| 6.3 | **Dashboard: Network Overview** | Metabase | P0 | L | 18 | Eng 1 | Dashboard view |
| 6.4 | **Dashboard: Orchestrator Performance** | Metabase | P0 | L | 20 | Eng 1 | Dashboard view |
| 6.5 | **Dashboard: GPU Metrics** | Metabase | P0 | L | 18 | Eng 1 | Dashboard view |
| 6.6 | **Dashboard: Network Demand** | Metabase | P0 | M | 14 | Eng 2 | Dashboard view |
| 6.7 | **Dashboard: Test Load Results** | Metabase | P1 | M | 12 | Eng 2 | Dashboard view |
| 6.8 | Create dashboard user guide | Documentation | P1 | M | 10 | Eng 1 | User docs |
| 6.9 | Deploy all components to production | Infrastructure | P0 | L | 16 | Eng 2 | Prod deployment |
| 6.10 | Configure production monitoring alerts | Infrastructure | P0 | M | 10 | Eng 2 | Alert rules |

**Sprint Planning:** 2 hours

### Deliverables:
- ✅ Public Metabase dashboard live
- ✅ **5 dashboard views:**
  - Network Overview ✅
  - Orchestrator Performance ✅
  - GPU Metrics ✅
  - Network Demand ✅
  - Test Load Results ✅
- ✅ Dashboard user guide published
- ✅ All components in production
- ✅ Production monitoring & alerts active

### Exit Criteria:
- [ ] Dashboard publicly accessible (no auth)
- [ ] All 5 views loading data correctly
- [ ] User guide complete with screenshots
- [ ] Production deployment verified
- [ ] Alerts tested (trigger test alert)

### Parallel Work:
- Eng 1: Dashboard views (Network, Orchestrator, GPU)
- Eng 2: Dashboard views (Demand, Test) + production deployment

---

# Epic 7: Stabilization, Documentation & Launch
**Dates:** March 31 - April 10, 2025 (Sprint 7)  
**Goal:** Harden system, complete docs, community review, SHIP IT

**Capacity:** 138 hours | **Allocated:** 130 hours (8 hour buffer)

## Tasks

| ID | Task | Component | Priority | Size | Hours | Assignee | Deliverable |
|----|------|-----------|----------|------|-------|----------|-------------|
| 7.1 | Optimize ClickHouse queries (slow queries) | ClickHouse | P1 | L | 18 | Eng 2 | Query optimization |
| 7.2 | Reduce infrastructure costs | Infrastructure | P1 | M | 12 | Eng 2 | Cost reduction plan |
| 7.3 | Complete API documentation | Documentation | P0 | M | 14 | Eng 1 | Full API docs |
| 7.4 | Complete metrics catalog v1.0 | Documentation | P0 | M | 12 | Eng 1 | `METRICS_V1.md` |
| 7.5 | Write operational runbooks | Documentation | P0 | L | 18 | Eng 2 | Runbook docs |
| 7.6 | Create architecture decision records (ADRs) | Documentation | P1 | M | 10 | Eng 1 | ADR documents |
| 7.7 | Conduct production load testing | Infrastructure | P0 | M | 12 | Eng 2 | Load test report |
| 7.8 | Fix P0 bugs from testing | All | P0 | L | 16 | Both | Bug fixes |
| 7.9 | Document known limitations/gaps | Documentation | P0 | S | 6 | Eng 1 | `KNOWN_GAPS.md` |
| 7.10 | Prepare community update (forum/watercooler) | Documentation | P0 | S | 6 | Eng 1 | Forum post draft |
| 7.11 | Final security review | Infrastructure | P0 | S | 6 | Eng 2 | Security checklist |

**Sprint Planning:** 2 hours

### Deliverables:
- ✅ System hardened for production (99.5% uptime)
- ✅ **Complete documentation:**
  - API documentation ✅
  - Metrics catalog v1.0 ✅
  - Operational runbooks ✅
  - Architecture decision records ✅
  - Known limitations ✅
- ✅ Load test report (performance benchmarks)
- ✅ Community update prepared
- ✅ Security review completed
- ✅ **MVP SHIPPED** 🚀

### Exit Criteria:
- [ ] System uptime >99.5% for 7 days
- [ ] All P0 documentation complete
- [ ] Community update posted to forum
- [ ] Security checklist 100% complete
- [ ] Zero P0 bugs in production
- [ ] **April 10 deadline met**

### Parallel Work:
- Eng 1: Documentation (API, metrics, ADRs)
- Eng 2: Performance optimization + runbooks + load testing

---

# Risk Management & Contingency

## High-Risk Items (Mitigation Plans)

| Risk | Epic | Probability | Impact | Mitigation |
|------|------|-------------|--------|------------|
| go-livepeer changes break gateway | 2 | Medium | High | Thorough testing in staging, rollback plan |
| ClickHouse performance issues at scale | 3, 5 | Medium | High | Start optimization early (Epic 3), load test in Epic 7 |
| AI Job Tester modifications complex | 4 | Medium | Medium | Audit existing code in Epic 1, allocate 24 hours |
| API rate limiting under load | 5 | Low | Medium | Load test in Epic 5, can adjust limits post-launch |
| Dashboard queries slow | 6 | Medium | Medium | Optimize queries in Epic 7, use materialized views |
| P0 bugs discovered late | 7 | Medium | High | 16-hour bug fix buffer in Epic 7 |

## Contingency: What to Cut if Behind Schedule

### Priority Tiers (Cut in Order):

**Tier 1 - Cut First (P2 tasks):**
- Internal test dashboard (4.12)
- Dataset contribution docs (not tracked explicitly)
- Architecture decision records (7.6)

**Tier 2 - Cut Second (P1 non-critical):**
- API caching layer (5.8) - can add post-launch
- Test Load Results dashboard view (6.7) - focus on core 4 views
- Cost optimization (7.2) - can do post-launch

**Tier 3 - DO NOT CUT (P0 blockers):**
- Data quality fixes (Epic 2) - CRITICAL PATH
- Core 5 metrics (Epic 3) - MVP definition
- Load test infrastructure (Epic 4) - Foundation requirement
- Public API (Epic 5) - Deliverable #1
- Public dashboard (Epic 6) - Deliverable #2

## Buffer Allocation

| Epic | Allocated Hours | Buffer Hours | Buffer % |
|------|----------------|--------------|----------|
| 1 | 130 | 8 | 6% |
| 2 | 136 | 2 | 1.5% |
| 3 | 134 | 4 | 3% |
| 4 | 136 | 2 | 1.5% |
| 5 | 134 | 4 | 3% |
| 6 | 136 | 2 | 1.5% |
| 7 | 130 | 8 | 6% |
| **Total** | **936** | **30** | **3.2%** |

**Note:** Epic 2 and 4 have minimal buffer due to critical path dependencies. Epic 7 has largest buffer for unexpected issues.

---

# GitHub Project Configuration

## Board Columns

1. **Backlog** - All tasks not yet started
2. **Ready** - Tasks ready to start (dependencies met)
3. **In Progress** - Currently being worked on
4. **Review** - PR submitted, awaiting review
5. **Done** - Merged and deployed to staging/prod

## Labels

### Priority
- `P0-blocker` (red)
- `P1-critical` (orange)
- `P2-important` (yellow)

### Size
- `size/XS` (2-4h)
- `size/S` (4-8h)
- `size/M` (8-16h)
- `size/L` (16-32h)
- `size/XL` (32-64h)

### Component
- `component/go-livepeer`
- `component/ai-job-tester`
- `component/test-gateway`
- `component/network-scraper`
- `component/kafka`
- `component/clickhouse`
- `component/dbt`
- `component/api`
- `component/metabase`
- `component/infrastructure`
- `component/docs`

### Type
- `type/feature`
- `type/bug`
- `type/docs`
- `type/infrastructure`
- `type/optimization`

### Status
- `status/blocked`
- `status/waiting-review`

## Milestones

Create 7 GitHub Milestones:
1. **Epic 1: Foundation** (Jan 6-17)
2. **Epic 2: Data Quality** (Jan 20-31)
3. **Epic 3: Analytics** (Feb 3-14)
4. **Epic 4: Load Test** (Feb 17-28)
5. **Epic 5: API** (Mar 3-14)
6. **Epic 6: Dashboard** (Mar 17-28)
7. **Epic 7: Launch** (Mar 31-Apr 10)

---

# Success Metrics

## Technical KPIs (Measured at Launch)

| Metric | Target | How Measured |
|--------|--------|--------------|
| System Uptime | >99.5% | Prometheus uptime checks |
| API Response Time (p95) | <500ms | APM/load test tooling |
| Data Freshness | <5 min lag | ClickHouse ingestion monitoring |
| Test Coverage | >80% | CI test reports |
| Data Quality Score | >95% | Validation tests passing |
| ClickHouse Query Performance | <2s for dashboard | Query profiling |

## Delivery KPIs (April 10 Target)

| Metric | Target |
|--------|--------|
| On-Time Epic Completion | 7/7 Epics by Apr 10 |
| P0 Bugs in Production | <3 |
| Documentation Coverage | 100% of public APIs |
| Community Engagement | Forum post + watercooler |

---

# Weekly Sprint Rhythm

### Monday (Week Start)
- **Sprint Planning:** 1 hour (both engineers)
- Review previous sprint
- Pull tasks from backlog
- Assign work for week

### Wednesday (Mid-Sprint)
- **Async Check-in:** 15 min (Slack/async)
- Blockers?
- On track?

### Friday (Week End)
- **Sprint Planning (Week 2):** 1 hour (both engineers)
- Review sprint progress
- Plan week 2 tasks
- Update GitHub board

### Every 2 Weeks (Sprint End)
- **Demo:** Show completed work
- **Deploy:** Staging deployment
- **Docs:** Update documentation

---

# Communication Plan

### Internal (Cloud SPE Team)
- **Daily:** Async Slack check-ins
- **Weekly:** 1-hour sprint planning
- **Bi-weekly:** Sprint demos

### External (Daydream Coordination)
- **Epic 2 End:** Notify of go-livepeer changes
- **Epic 4 End:** Notify of load test gateway operational
- **Epic 6 End:** Notify of public dashboard launch
- **Epic 7 End:** Community forum post + watercooler

---

# Tools & Infrastructure

## Development Tools
- **GitHub:** Project management, code hosting
- **GitHub Actions:** CI/CD pipelines
- **Docker:** Containerization
- **Terraform:** Infrastructure as code

## Data Stack
- **Kafka:** Event streaming (existing)
- **ClickHouse:** Data warehouse (provision)
- **dbt:** Data transformations (setup)
- **Metabase:** Dashboards (provision)

## Observability
- **Prometheus:** Metrics collection
- **Grafana:** Metrics visualization
- **Alertmanager:** Alert routing

## Languages & Frameworks
- **Go:** go-livepeer, AI Job Tester, API (option)
- **Python:** Network scraper, API (FastAPI option)
- **SQL:** dbt, ClickHouse queries
- **YAML:** dbt, CI/CD configs

---

# Epic Summary Table

| Epic | Dates | Capacity | Allocated | P0 Tasks | P1 Tasks | Key Deliverable |
|------|-------|----------|-----------|----------|----------|-----------------|
| 1 | Jan 6-17 | 138h | 130h | 8 | 3 | Infrastructure + metrics catalog |
| 2 | Jan 20-31 | 138h | 136h | 10 | 1 | Data quality fixes (GPU_ID, gateway, region) |
| 3 | Feb 3-14 | 138h | 134h | 8 | 2 | 5 core metrics calculated |
| 4 | Feb 17-28 | 138h | 136h | 11 | 1 | Load test infrastructure operational |
| 5 | Mar 3-14 | 138h | 134h | 9 | 1 | Public API deployed |
| 6 | Mar 17-28 | 138h | 136h | 10 | 0 | Public dashboard live |
| 7 | Mar 31-Apr 10 | 138h | 130h | 7 | 4 | **MVP SHIPPED** 🚀 |
| **Total** | **14 weeks** | **966h** | **936h** | **63** | **12** | **NaaP MVP v1.0** |

---

# Final Notes

## What Makes This Plan Achievable

1. ✅ **Senior engineers:** High productivity, full-stack capability
2. ✅ **Realistic sizing:** Based on 70-hour weeks (35h × 2 engineers)
3. ✅ **Buffer built-in:** 30 hours total across 7 sprints
4. ✅ **Clear priorities:** P0 tasks are MVP blockers, P1 can slip
5. ✅ **Incremental delivery:** Ship to staging every 2 weeks
6. ✅ **Parallel work:** Engineers work independently where possible
7. ✅ **No external dependencies:** Cloud SPE controls everything

## Critical Success Factors

- ✅ Start Epic 2 (data quality) immediately after Epic 1
- ✅ Do NOT add scope mid-flight
- ✅ Cut P1/P2 tasks if falling behind
- ✅ Test continuously (don't save testing for Epic 7)
- ✅ Document as you go (don't defer to Epic 7)

## Hard Constraints

- ⚠️ **April 10 deadline is non-negotiable**
- ⚠️ **2 engineers, no more**
- ⚠️ **$90k budget** (already allocated)
