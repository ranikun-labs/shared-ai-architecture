# ADR-0005 — Phase 1 Runtime Placement와 Platform Core Modular Monolith Boundary

| 항목 | 값 |
| --- | --- |
| status | Accepted |
| jira | RPL-103 Stage A |
| predecessor | RPL-102 / [ADR-0004](ADR-0004-execution-context-api-tenant-isolation.md) |
| ADR namespace | shared-ai-architecture 로컬 (`ADR-000N`) |
| coordinated owner decision | `harness-foundation-docs` ADR-0018 / PR #23 |
| independent review | `RPL103_PLACEMENT_REVIEW_PASS` (BLOCKER 0 / MAJOR 0 / MINOR 1 non-blocking) |
| approved by | 박성환 (owner decision) |
| reviewed / approved / effective | 2026-08-26 |
| scope | Phase 1 physical placement와 extraction boundary만 결정 |

---

## 1. Status

**Accepted. Effective from 2026-08-26.**

Independent Architecture Review 결과는 `RPL103_PLACEMENT_REVIEW_PASS`다. 이 ADR과
`harness-foundation-docs` ADR-0018은 2026-08-26 coordinated owner acceptance를 받았다.
어느 한 ADR도 단독으로 Architecture를 변경하지 않았으며, 양쪽 decision이 함께 accepted
됨으로써 Phase 1 placement가 effective가 됐다. Canonical `main` projection은 coordinated
PR #6 / Foundation PR #23 merge 전까지 완료로 주장하지 않는다.

```text
Phase 1 placement = accepted / effective
Canonical main projection = pending coordinated merge
!= RPL-103 complete
!= runtime technology selected
!= implementation started
```

## 2. Context

기존 Foundation 문서는 Shared AI를 `platform-services` 밖의 future independent Python
Runtime 후보로 기록했다. 이후 RPL-97, RPL-99~102는 Product와 Shared AI 사이의 logical
ownership, Evidence, Retrieval, Execution Context와 tenant isolation을 확정했지만 물리
runtime 위치와 technology는 RPL-103으로 넘겼다.

현재 `platform-services` canonical target은 다음 두 process다.

```text
gateway-app    = independent Spring Cloud Gateway / WebFlux process
platform-core  = one-process Spring MVC modular monolith
```

`platform-core/identity`의 current main은 business extraction 전 inert runtime scaffold에
가깝다. 소규모 운영에서 capability마다 별도 JVM을 두면 ApplicationContext, embedded
server, actuator, thread pool, connection pool, container, health check와 service-to-service
network/auth 비용이 반복된다. 반면 Shared AI의 workload는 memory, streaming, provider retry와
latency로 같은 JVM의 blast radius를 키울 수 있다.

## 3. Decision Scope

### Scope In

- Shared AI Phase 1 physical process placement
- `platform-core` host와 feature module 실행 경계
- same-JVM ownership 및 dependency rule
- blast-radius acceptance와 최소 evaluation criteria
- future process extraction trigger와 extraction seam
- ADR-0017 및 source-of-truth map의 placement assumption partial supersession

### Scope Out

- Kotlin 대 Java
- Spring Boot 및 Spring AI exact version
- Spring AI 대 LangChain/LangGraph
- pgvector, NATS와 streaming implementation
- Identity 또는 Shared AI 구현
- RPL-54 behavior extraction과 RPL-55 cutover
- RPL-103 completion

## 4. Preserved Logical Invariants

이 placement decision은 ADR-0001~0004와 RPL-97 Foundation의 logical architecture를
변경하지 않는다.

```text
Product authorization semantics = Product-owned
Identifier != Authorization
Runtime Projection != Semantic SoT
exact structured fact = Product Tool / SSOT
narrative/document = Retriever / RAG
query-time current authorization > cached historical scope
child capability scope cannot implicitly widen parent scope
Evidence != authorization proof
Shared AI cannot directly access Product DB
provider/framework type != canonical contract
```

## 5. Considered Options

### Option A — Independent Runtime / Process

```text
Shared AI = independent deployable process
```

장점:

- independent scaling, deployment와 rollback
- AI memory/thread/streaming/provider 장애의 process isolation
- Python-only/GPU/local-model dependency 수용이 쉬움

비용:

- 별도 JVM 또는 Python runtime, container, health/observability와 connection pool
- 내부 network contract, service credential, timeout/retry와 serialization 필요
- Phase 1의 낮은 workload에서도 고정 운영 비용 발생

Option A는 유효한 future extraction target이지만 현재 증거만으로 Phase 1 기본값으로
선택하지 않는다.

### Option B — Platform Core Logical Module / Same JVM

```text
Shared AI = platform-core logical module
          + same Spring MVC process
```

장점:

- Platform Core의 고정 runtime·container·observability 비용 공유
- network hop 없이 typed in-process application API 사용
- 현재 small-scale workload와 one-process modular-monolith target에 정렬
- Identity business extraction 전에 host topology를 한 번만 확정 가능

비용:

- OOM, GC, process crash, startup/readiness와 deploy/rollback 단위 공유
- AI executor/provider/streaming saturation이 Identity latency에 영향을 줄 수 있음
- dependency/classpath compatibility를 process 전체에서 관리해야 함

## 6. Decision — Option B

**Phase 1 placement로 Option B를 선택한다.** 이 선택은 coordinated Foundation
ADR-0018과 함께 2026-08-26 accepted / effective가 됐다.

```text
platform-services
├── gateway-app                  independent WebFlux process
└── platform-core
    ├── app                      single executable Spring MVC host
    ├── identity                 library / logical module
    ├── shared-ai                future library / logical module
    ├── commerce                 future, deferred
    └── audit                    future, deferred
```

Gradle target notation:

```text
:gateway-app
:platform-core:app
:platform-core:identity
:platform-core:shared-ai         future; implementation not started
:platform-core:commerce          future; deferred
:platform-core:audit             future; deferred
```

`platform-core:app`만 executable Spring Boot process와 composition root를 소유한다.
Feature module은 `main()`, embedded server ownership과 독립 `bootJar` deploy unit을 갖지
않는다.

## 7. Gateway Boundary

Gateway는 별도 process를 유지한다.

- WebFlux/Netty edge runtime
- routing, security와 rate limit
- independent scaling과 rollback
- long-lived connection isolation
- Platform Core MVC lifecycle과 분리

Gateway는 Platform Core feature module을 compile-time dependency로 참조하지 않는다.
Gateway와 Platform Core 사이에는 network boundary가 유지된다.

## 8. Module Ownership and Dependency Rules

```text
platform-core:app
 ├─> identity
 ├─> shared-ai
 ├─> commerce
 └─> audit

feature module -> another feature module internal implementation = prohibited by default
```

금지:

- `shared-ai -> identity repository/internal service`
- `commerce -> identity repository/internal service`
- cross-module DB table direct read/write
- cross-module FK/JOIN convenience coupling
- module repository/entity exposure
- shared mutable domain entity

허용되는 경계:

- explicit public module API
- consumer-owned Port
- provider/framework-neutral typed contract
- 필요한 경우 owner가 명시한 event; event architecture는 기본값이 아님

동일 PostgreSQL physical instance는 가능하지만 schema, table, migration과 data ownership은
모듈별로 분리한다. Host가 module 내부 transaction을 하나의 cross-module transaction으로
합치지 않는다.

## 9. Same-JVM Failure Acceptance

다음 failure boundary를 의도적으로 공유한다.

```text
OOM / metaspace exhaustion / GC pause / process crash
startup and readiness failure
thread and executor starvation
dependency and classpath conflict
deployment and rollback unit
```

특히 long model call, provider retry, streaming, large payload와 executor saturation이
Identity latency와 process health에 영향을 줄 수 있다.

Phase 1 implementation/evaluation은 최소 다음 seam과 관측 가능성을 제공해야 한다.

- AI bounded executor와 concurrency limit
- provider timeout
- bulkhead/circuit-breaker seam
- payload/result limit
- module-specific latency/error/queue metrics
- thread, heap/RSS와 GC saturation metrics
- AI dependency failure가 Identity readiness failure로 자동 전파되지 않는 health model

이 목록은 library/framework 선택이나 수치 threshold를 확정하지 않는다.

## 10. Extraction Seam

Shared AI public application API와 internal implementation을 분리한다.

Canonical contract는 provider/framework-neutral하게 유지한다.

```text
ExecutionContext
Execute Request / Result
Evidence / Provenance reference
```

유지할 Port:

```text
Product Tool
Current Authority
Retriever
Model Gateway
```

- Product/Identity DB 직접 접근을 금지한다.
- Shared AI-owned storage가 필요하면 schema/migration owner를 분리한다.
- transport는 adapter다.
- Phase 1은 in-process adapter를 사용한다.
- future extraction은 HTTP/SSE 또는 concrete async use case가 승인된 경우 NATS adapter로
  교체할 수 있다.
- 내부 호출에 network DTO, service discovery와 distributed retry를 미리 흉내 내지 않는다.

## 11. Process Extraction Triggers

다음 중 하나 이상이 실제 evidence로 확인되면 독립 process 재검토를 시작한다.

- independent scaling requirement
- sustained streaming concurrency
- AI workload가 Identity latency/SLO에 유의미한 영향
- memory/RSS/GC pressure
- independent deployment cadence 필요
- AI failure blast-radius isolation 필요
- Python-only ML dependency가 load-bearing
- local embedding/reranker/model 또는 GPU 필요

RPL-103 Technology Stage 또는 운영 review에서 latency, concurrency, executor queue,
stream count, heap/RSS, GC, restart/failure correlation과 deployment frequency를 측정한다.
근거 없는 threshold는 이 ADR에서 만들지 않는다.

## 12. Identity / G3 Ordering

Placement가 accepted 되면 권장 순서는 다음이다.

```text
1. RPL-103 Stage A topology canonical acceptance
2. platform-core host structural refactor
3. identity library module 전환
4. RPL-54 G3 behavior-preserving extraction
5. RPL-55 Carelog cutover
```

이는 G3를 독립 Identity JVM에 완료한 뒤 다시 host consolidation하는 이중 변경을 피한다.
이 ADR은 해당 refactor 또는 RPL-54 구현 authority가 아니다.

## 13. Verification Tool Boundary

Gradle dependency와 package boundary가 1차 방어선이다. Spring Modulith 또는 ArchUnit은
다음 verification-only 후보로 사용할 수 있다.

- dependency cycle
- internal package access
- allowed dependency
- public interface

모든 communication event화, event repository 선도입, Modulith annotation의 canonical
architecture 승격과 도구에 맞춘 domain package 왜곡은 금지한다.

## 14. Partial Supersession

이 ADR과 coordinated Foundation ADR-0018의 2026-08-26 acceptance는 다음 physical
placement assumption만 부분 대체한다.

| Existing source | Partially superseded placement assumption | Replacement |
| --- | --- | --- |
| Foundation ADR-0017 §7 | Shared AI를 `platform-services/platform-core`에 넣지 않음 | Shared AI Phase 1 selected placement는 `platform-core` logical module + same Spring MVC JVM |
| Foundation ADR-0017 §7 | future independent Python Runtime을 기본 placement로 표현 | independent runtime은 evidence-triggered extraction option; language는 deferred |
| `source-of-truth-map.md` §3/§4 | `platform-services` 밖 independent Python Runtime 및 위치/시점 전체 deferred | Phase 1 placement는 Platform Core same JVM; technology와 activation은 deferred |

다음은 supersede하지 않는다.

- ADR-0017의 independent Gateway WebFlux process
- ADR-0017의 one-process Spring MVC Platform Core
- Identity/Commerce/Audit logical module target과 Commerce/Audit deferral
- ADR-0001~0004 전체 logical ownership/invariant
- Product data/authorization/contract ownership
- implementation/runtime support/release 상태 분리
- independent extraction 가능성과 trigger-based physicalization 원칙

이 ADR 하나가 독립적으로 ADR-0017을 대체했다고 주장하지 않는다. Partial supersession은
Foundation ADR-0018과의 coordinated acceptance로만 성립한다.

## 15. Technology Decisions Deferred

```text
Kotlin vs Java
Spring Boot exact version
Spring AI exact version
Spring AI vs LangChain/LangGraph
pgvector
NATS
streaming implementation
provider/model
```

RPL-104 Python/FastAPI PoC는 comparison evidence로 유지한다. RPL-103 Technology Stage는
fresh official compatibility source와 추가 JVM comparison evidence를 검토한다.

## 16. Canonical Source Reality

| Repository | Ref | SHA / state |
| --- | --- | --- |
| `shared-ai-architecture` | `origin/main` | `95c9da5da8cb3764ae3d54d4175b2c9cd17fb94f` |
| `harness-foundation-docs` | `origin/main` | `69fceb267094756913f400f8b4045590bd0e8ac5` |
| `platform-services` | `origin/main` | `31636aa43937a0119318de0fe7bc2e51ebfda3d0` |
| `shared-ai-runtime` | comparison evidence branch HEAD | `e23cb4f1c4f02114bda08d1d2b99ba56911f0476`; no `main` ref observed |

Feature/review checkout은 canonical source로 사용하지 않았다. SHA는 이번 acceptance decision의
재현 anchor이며 영구 invariant가 아니다.

## 17. Consequences and Follow-up

긍정:

- Phase 1 운영 고정 비용을 낮추면서 logical ownership을 유지한다.
- G3 전에 Platform Core target host를 한 번만 확정할 수 있다.
- extraction trigger와 seam을 canonical review 대상으로 만든다.

비용:

- Identity와 Shared AI가 process failure/deploy unit을 공유한다.
- resource isolation과 module verification 규율이 필요하다.
- Foundation과 Shared AI 양쪽 owner approval이 필요하다.

후속:

- coordinated PR #6 / Foundation PR #23의 canonical main projection 검토
- RPL-104 및 JVM comparison evidence 검토
- RPL-103 Technology Stage 계속; RPL-103은 아직 complete가 아님
- accepted 이후에만 Platform Core structural implementation 별도 수행
