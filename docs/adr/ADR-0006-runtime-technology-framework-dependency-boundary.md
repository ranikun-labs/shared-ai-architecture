# ADR-0006 — Phase 1 Runtime Technology, Framework와 Dependency Boundary

| 항목 | 값 |
| --- | --- |
| status | Proposed — owner decision recorded, independent review pending |
| jira | RPL-103 Stage B |
| predecessor | [ADR-0005](ADR-0005-runtime-placement-platform-core-modular-monolith-boundary.md) / Foundation ADR-0018 |
| ADR namespace | shared-ai-architecture 로컬 (`ADR-000N`) |
| owner decision | JVM Core + optional future Python ML Worker |
| approved by | 박성환 (owner decision) |
| decision date | 2026-08-26 |
| independent review | pending |
| scope | Phase 1 runtime language, application framework, AI integration과 dependency boundary |

---

## 1. Status

**Proposed. Owner decision recorded; independent Architecture Review와 canonical merge는 pending이다.**

이 ADR은 review candidate다. 문서가 존재하거나 branch에 commit됐다는 사실은 canonical
acceptance, 구현 또는 배포를 의미하지 않는다.

```text
RPL-103 Stage A = CANONICAL COMPLETE
RPL-103 Stage B owner decision = recorded
RPL-103 Stage B independent review = pending
RPL-103 overall = IN PROGRESS
Shared AI implementation = not_started
```

## 2. Context와 Decision Question

Stage A는 다음 placement를 이미 canonical로 확정했다.

```text
JVM #1: gateway-app
        Spring Cloud Gateway / WebFlux / independent process

JVM #2: platform-core-app
        Spring MVC / single process
        logical modules: identity, shared-ai,
                         commerce(future), audit(future)
```

Stage B는 이 placement를 다시 평가하지 않는다. 질문은 Platform Core의 Shared AI logical
module이 Product별 AI application flow를 어떤 language와 framework boundary로 구현할지다.
Shared AI는 generic RAG server가 아니라 다음 조합을 Product use case별로 실행하는
multi-product AI Application Runtime이다.

```text
ExecutionContext
→ Product-specific SkillHandler
→ Product Tool / Retriever / Policy / Validator
→ Model
→ Evidence + Result
```

## 3. Decision Scope

이 ADR은 다음을 결정한다.

- Phase 1 Shared AI implementation language와 Platform Core host model
- Kotlin과 Java 사이의 module contract rule
- Spring AI의 architecture classification과 dependency direction
- Product-specific application/Skill composition shape
- RAG, Tool, streaming과 observability의 technology boundary
- Python worker와 LangGraph를 다시 검토할 evidence trigger
- Gateway와 Platform Core 사이에 필요한 build/version isolation 원칙

다음은 결정하지 않는다.

- Stage A placement 또는 Gateway architecture
- production package, API/DTO schema, database schema와 migration
- provider, model, embedding model, chunk size와 reranker
- exact Product workflow, Prompt, Lens, Policy 또는 Validator semantics
- service-to-service credential mechanism의 concrete HOW
- implementation schedule 또는 deployment activation

## 4. Preserved Canonical Invariants

Technology 선택은 ADR-0001~0005의 다음 invariant를 변경하지 않는다.

- Product authorization semantics는 Product가 소유한다.
- identifier는 authorization이 아니다.
- Runtime Projection은 Semantic Source of Truth가 아니다.
- exact structured fact는 Product Tool / structured SSOT가 제공한다.
- narrative/document knowledge는 Retriever/RAG가 제공한다.
- query-time current Product authorization이 cached/indexed historical scope보다 우선한다.
- child capability scope는 parent scope를 암묵적으로 넓힐 수 없다.
- Evidence는 authorization proof가 아니다.
- Shared AI는 Product DB에 직접 접근하지 않는다.
- framework/provider type은 canonical contract가 아니다.
- same JVM은 module ownership bypass를 허용하지 않는다.

## 5. Decision

Phase 1은 **JVM Core + optional future Python ML Worker**를 선택한다.

| 영역 | Phase 1 결정 |
| --- | --- |
| Platform Core host | Spring Boot / Spring MVC, imperative application core |
| Identity language | Java 유지 |
| Shared AI language | Kotlin |
| AI integration | Spring AI를 adapter/integration library로 사용 |
| Retrieval persistence | PostgreSQL + pgvector + 필요 시 PostgreSQL FTS |
| Streaming boundary | MVC SSE / async servlet; adapter-local Reactor 허용 |
| Observability | Micrometer Observation·metrics + OpenTelemetry/OTLP |
| Python | Phase 1 runtime/worker 없음; evidence-triggered future option |

Kotlin 선택은 Java보다 architecture적으로 필수여서가 아니라 typed request/result,
nullable provider output, Evidence variant와 mapping이 많은 Shared AI codebase에 대한 가까운
owner choice다. Java는 유효한 대안이며 Identity의 구현 언어로 유지한다.

## 6. Kotlin / Java Boundary

Shared AI Kotlin code는 다음 규칙을 따른다.

1. Spring proxy compatibility를 위해 `kotlin-spring` plugin을 사용한다.
2. Java와 Kotlin은 Platform Core의 같은 Java toolchain target을 사용한다.
3. application core는 imperative다. Coroutine/Flow는 기본 programming model이 아니다.
4. Coroutine/Flow는 이후 실제 필요가 증명된 adapter boundary에서만 검토한다.
5. cross-module / Java-facing public API는 Java-friendly JVM contract여야 한다.
6. cross-module / Java-facing public API는 `suspend` API, `kotlin.Result`,
   `kotlinx.coroutines.flow.Flow`, Kotlin function type, `kotlin.Pair`, `kotlin.Triple`, Java 사용이
   Kotlin default parameter에 의존하는 API 또는 adapter 밖의 provider/framework-specific
   Kotlin type에 의존해서는 안 된다. 이 금지는 Shared AI 내부의 Kotlin 사용을 제한하지 않는다.
7. Kotlin 내부에서는 `data class`, `sealed interface`, exhaustive `when`을 Evidence/Result
   variant에 권장한다.
8. extension function은 명시적인 adapter/mapping transformation으로 제한한다.
9. 과도한 DSL 또는 scope-function nesting을 피한다.

이 경계 아래에서 Java Identity와 Kotlin Shared AI는 같은 build/JVM에서 일반 JVM bytecode와
Java-friendly contract로 상호 운용한다. Kotlin compiler/plugin과 interop test 비용은 남지만,
별도 runtime, serialization protocol 또는 network hop은 생기지 않는다.

## 7. Spring AI Dependency Boundary

Spring AI의 classification은 **ADAPTER / INTEGRATION LIBRARY**다. Spring AI는 canonical
architecture도 application abstraction도 아니다.

```text
Shared AI Application
→ application-owned Port
→ Spring AI Adapter
→ Provider / VectorStore
```

adapter 아래에서 사용할 수 있는 Spring AI type은 다음과 같다.

- `ChatClient`, `ChatModel`
- `ToolCallback`
- `Document`, `VectorStore`
- `Advisor`
- provider SDK type

다음 contract는 Shared AI application이 소유하고 framework-neutral하게 유지한다.

- `ExecutionContext`, `SkillId`, `SkillHandler`
- `ProductToolPort`, `RetrieverPort`, `ModelGateway`
- `Evidence`, `AuthorizationDecision`, `RetrievedItem`
- `ModelRequest`, `ModelResult`, neutral `StreamEvent`
- `Policy`, `Validator`

Spring AI type이 web API, Product/module contract, Evidence schema 또는 application service
signature로 역류하면 이 ADR을 위반한다. Advisor는 provider concern, prompt decoration,
observation 같은 adapter-local composition에는 사용할 수 있지만 Product-specific orchestration을
숨기는 workflow owner가 될 수 없다.

## 8. Product-specific Application / Skill Architecture

Target logical shape는 conceptual architecture이며 이 ADR task에서 package를 생성하지 않는다.

```text
shared-ai/
├─ core/
│  ├─ execution
│  ├─ evidence
│  ├─ retrieval
│  └─ ports
├─ carelog/
│  ├─ customer-summary
│  ├─ treatment-preparation
│  └─ conversation-review
├─ finance/
│  ├─ investment-review
│  ├─ journal-review
│  └─ risk-analysis
└─ adapters/
   ├─ model
   ├─ retrieval
   ├─ product-tool
   └─ web
```

각 Product use case는 typed `SkillHandler`로 application flow를 소유하고 explicit registry로
`SkillId`에 연결한다. Tool, Retriever, Model과 Policy의 선택·순서는 handler/application
service에서 명시적으로 보인다. Product별 orchestration을 framework annotation, dynamic
plugin loader 또는 Advisor chain 안에 숨기지 않는다.

Dependency direction은 Product-specific handler → core port이며, core/application layer는
Spring AI, provider SDK, pgvector client 또는 web transport에 의존하지 않는다.

## 9. RAG / Persistence Boundary

Phase 1 retrieval baseline은 **PostgreSQL + pgvector + 필요한 경우 PostgreSQL FTS**다.
별도 Vector DB는 요구하지 않는다.

Application이 소유하는 것은 다음과 같다.

- projection lifecycle, eligibility, deletion과 invalidation
- stale-scope 처리와 current authorization revalidation
- tenant/product metadata와 filter semantics
- retrieval result와 Evidence의 correlation
- index lifecycle과 semantic SoT 사이의 비동일성

Spring AI `VectorStore`는 `RetrieverPort` 뒤의 adapter다. VectorStore filter 결과는
authorization decision이 아니다. Product Tool의 exact structured fact와 Retriever의
narrative knowledge를 혼합하지 않는다.

query rewriting, hybrid retrieval, reranking, retrieval validation, agentic RAG, GraphRAG와
separate vector infrastructure는 실제 quality/scale/operational evidence가 있을 때만 도입한다.

## 10. Tool / Authorization Boundary

Product Tool은 authoritative Product capability다. Spring AI Tool Calling은 실행 machinery다.

- model tool choice는 authorization이 아니다.
- model-generated argument는 untrusted input으로 validation한다.
- authoritative call은 가능한 경우 Product-specific handler가 결정적으로 선택한다.
- model-owned selection은 request-specific capability allowlist 안에서만 허용한다.
- 매 실행 전 current Product authority/scope를 재검증한다.
- side-effecting tool은 blind model retry를 받지 않는다.
- Tool result는 Evidence input이지만 authorization proof가 아니다.

Retry, timeout, error mapping과 observation은 adapter/application policy가 명시적으로 통제한다.
Provider의 tool abstraction이 `ProductToolPort`의 ownership이나 semantics를 대체하지 않는다.

## 11. Streaming Boundary

Shared AI를 WebFlux-first로 만들지 않는다. application core는 imperative를 유지한다.

Phase 1 streaming adapter는 MVC SSE와 async servlet을 기본으로 한다. Spring AI streaming
API가 Reactor를 제공할 경우 `Flux`는 adapter-local로 한정하고 neutral `StreamEvent`로
mapping한다. Kotlin을 선택했다는 이유만으로 Coroutine/Flow를 도입하지 않는다.

구현 acceptance에는 cancellation propagation, timeout, client disconnect cleanup, bounded
stream duration, bounded concurrency와 active stream resource metric이 포함된다. 같은 JVM의
Identity를 보호하기 위해 provider executor, connection pool과 concurrency bulkhead를
공유 기본값에 무제한으로 맡기지 않는다.

## 12. Observability Boundary

Phase 1 baseline은 다음이다.

- Micrometer Observation과 Micrometer metrics
- OpenTelemetry / OTLP export
- Spring AI observations at adapter boundary

Canonical-neutral correlation은 최소 다음 field/event를 다룬다.

- `execution_id`, Skill/use-case, request correlation
- Tool invocation과 retrieval/evidence reference
- provider/model operational metadata
- usage/token, latency, retry, timeout, cancellation과 rejection
- active stream/concurrency metric

Prompt, Tool argument/result, retrieved content와 model output은 기본 export 대상이 아니다.
민감 payload 관측은 별도 policy, redaction과 명시적 enablement 없이는 허용하지 않는다.
Spring AI observation type이나 vendor trace ID는 canonical Evidence contract가 아니다.

## 13. Python Worker Extraction Trigger

Phase 1에는 Python process 또는 worker를 만들지 않는다. 다음 load-bearing need가 evidence로
확인될 때 future worker/extraction을 검토한다.

- PyTorch/Transformers 또는 local model
- GPU execution
- JVM/provider path가 만족하지 못하는 local embedding/reranking
- Python-only document processing
- materially justified Python-only durable/agent capability
- Stage A의 independent scale, failure 또는 deploy trigger

현재 ports/adapters를 extraction-friendly하게 유지하되 아직 존재하지 않는 distributed seam,
protocol 또는 worker lifecycle을 미리 구현하지 않는다. 추출 시 Product authorization,
Evidence와 neutral contract는 그대로 보존하며 새로운 owner decision을 거친다.

## 14. LangChain / LangGraph Reconsideration Trigger

LangChain/LangGraph는 Phase 1 runtime으로 선택하지 않는다. Python AI/ML ecosystem과 durable
graph execution의 강점을 부정하는 결정이 아니다. 현재 예상되는 bounded request workflow는
typed application handler와 일반 imperative code로 합리적으로 표현할 수 있고, 이들을 지금
채택하면 canonical same-JVM topology에서 별도 runtime cost가 생긴다.

LangGraph는 다음 요구가 실제로 load-bearing해질 때 재검토한다.

- durable checkpoint와 process restart/resume
- long-running stateful workflow
- human-in-the-loop pause/resume
- persistent branching, replay와 audit recovery

Agent라는 명칭 또는 branch가 있다는 사실만으로 graph runtime을 도입하지 않는다.

## 15. Compatibility와 Build Isolation

### 15.1 2026-08-26 official compatibility anchor

아래 버전은 authoring 시점의 official compatibility를 재현하는 anchor다. patch version을
영구 고정하는 invariant가 아니며 구현 시점에 같은 supported line의 최신 patch를 다시 확인한다.

| 항목 | 공식 확인 결과 |
| --- | --- |
| Spring AI | stable `2.0.1`; `2.0.x`는 Spring Boot `4.0.x`와 `4.1.x` 지원 |
| Spring Boot | stable `4.1.1`; Java 17 이상, Spring Framework 7.0.9 이상 |
| Gradle | Boot 4.1.1 공식 지원은 8.14 이상 8.x 또는 9.x |
| Kotlin | Boot 4.1.1은 Kotlin 2.2.x 이상 요구; managed dependency `2.3.21` |
| Kotlin/Spring | `kotlin-stdlib`, `kotlin-reflect`, `kotlin-spring`, Jackson Kotlin module 필요 |
| Spring Cloud | `2025.1.x`는 2025.1.2부터 Boot 4.0/4.1 지원; `2024.0.x`는 Boot 3.4.x line이며 EOL 표시 |

Official sources:

- [Spring AI Getting Started](https://docs.spring.io/spring-ai/reference/getting-started.html)
- [Spring AI 2.0.1 release](https://github.com/spring-projects/spring-ai/releases/tag/v2.0.1)
- [Spring Boot system requirements](https://docs.spring.io/spring-boot/system-requirements.html)
- [Spring Boot Kotlin support](https://docs.spring.io/spring-boot/reference/features/kotlin.html)
- [Spring Boot managed dependencies](https://docs.spring.io/spring-boot/appendix/dependency-versions/coordinates.html)
- [Spring Cloud release-train mapping](https://spring.io/projects/spring-cloud/)

### 15.2 Current source reality와 architectural requirement

Freshly verified `platform-services/main` at
`74fd5a3001c1bf5e200de6a73afc3765685809c0` uses Boot 3.4.5, Cloud 2024.0.1,
Java toolchain 17과 Gradle 8.13. 이전 authoring reference
`31636aa43937a0119318de0fe7bc2e51ebfda3d0`에서 fresh main까지의 변경은 root `AGENTS.md`
추가뿐이며 application code, Gradle, dependency 또는 runtime topology 변경은 없다. 따라서
Stage B runtime analysis는 materially valid하다. `AGENTS.md`는 repository policy이며 runtime
implementation evidence가 아니다. Spring AI 2.0.x를 사용하는 Platform Core는 현재 공통
baseline을 그대로 사용할 수 없다.

Platform Core runtime/build baseline은 Gateway와 독립적으로 진화할 수 있어야 한다.
구현 track은 적절한 build/version isolation을 먼저 확립하고, Platform Core를 supported
Boot 4.1.x / Spring AI 2.0.x compatible line으로 올린다. 현재 Java 17 toolchain은 Boot 4.1.1의
minimum을 만족하지만, Gradle은 최소 8.14 이상으로 조정해야 한다.

이 결정은 Gateway를 Shared AI 때문에 upgrade하라고 요구하거나 Spring Cloud release train을
선택하지 않는다. Gateway는 Stage A의 independent WebFlux process와 현재 baseline을 유지할 수
있다. build isolation 구현이 Foundation의 platform-wide build governance를 변경해야 한다면
그 변경은 별도 Foundation owner decision 없이는 진행할 수 없다.

## 16. RPL-104 Evidence Use

RPL-104 Python/FastAPI PoC evidence head
`e23cb4f1c4f02114bda08d1d2b99ba56911f0476`는 다음을 증명한다.

- Python/FastAPI vertical slice가 feasible하다.
- `ExecutionContext → Skill → Tool + Retriever → Model → Evidence + Result`가 동작한다.
- domain/application boundary를 framework-neutral하게 유지할 수 있다.

이 evidence는 production provider, production authorization integration 또는 real retrieval
substrate를 검증하지 않았고 Python-only requirement도 증명하지 않는다. 비교 evidence로
보존하며 폐기하거나 archive하지 않는다. 현재 source reality와 official compatibility가
결정을 구분하기에 충분하므로 symmetric JVM PoC는 이 ADR 이전에 요구하지 않는다.

## 17. Considered Alternatives

### A. Python + FastAPI Core

Python AI/ML ecosystem, concise modeling과 LangChain/LangGraph 접근성이 강하다. RPL-104는
feasibility를 입증했다. 그러나 Phase 1에는 Python-only ML requirement가 없고, 이 선택은
canonical same-JVM placement 아래에서 별도 process/runtime을 요구한다. 선택하지 않는다.

### B. Java + Spring Shared AI

Identity와 같은 언어, 단순한 compiler/build와 가장 직접적인 Java API interop가 장점이다.
Architecture적으로 완전히 유효하며 점수 차이도 작았다. 다만 owner는 typed request/result,
sealed Evidence variant, null-safety와 mapping ergonomics가 많은 Shared AI에 Kotlin의 장기
유지보수 이점이 더 크다고 판단했다. Java는 Identity에 유지한다.

### C. TypeScript + NestJS

강한 module/DI model과 productive typing을 제공한다. 하지만 Platform Core JVM 안의 logical
module이 될 수 없어 Node runtime과 cross-process contract를 추가한다. Phase 1 topology fit이
낮아 선택하지 않는다.

### D. Kotlin + Spring Shared AI

선택한 Shared AI language다. Java interoperability, Spring ecosystem, data/sealed/null-safety
ergonomics가 장점이다. Kotlin compiler/plugin, proxy/open configuration과 mixed-language
contract discipline 비용을 §6 규칙으로 명시적으로 수용한다. Coroutine은 선택 이유가 아니다.

### E. JVM Core + Optional Python Worker

선택한 전체 candidate다. Phase 1 application/execution core는 Kotlin/JVM이고 Python worker는
만들지 않는다. §13 trigger가 evidence로 확인될 때만 좁은 ML/agent workload를 추출한다.

## 18. Consequences

### Positive

- Stage A same-JVM placement와 일치하고 별도 Phase 1 runtime을 추가하지 않는다.
- Product별 typed application flow를 Spring의 DI, configuration, testing과 observability 안에서
  명시적으로 구성할 수 있다.
- Kotlin의 null-safety와 sealed/data modeling을 내부 domain/application code에 활용한다.
- Spring AI/provider 교체가 application-owned ports 뒤로 격리된다.
- PostgreSQL 운영 footprint를 재사용하면서 future extraction seam은 보존한다.

### Costs / Risks

- Platform Core build/version isolation과 Boot/Gradle baseline 변경이 구현 선행 조건이다.
- Java/Kotlin mixed build, compiler/plugin과 interop testing 비용이 생긴다.
- Kotlin feature와 Spring AI type leakage를 architecture test/review로 통제해야 한다.
- same JVM의 provider latency, streaming과 resource pressure가 Identity를 방해하지 않도록
  bounded concurrency, bulkhead, timeout과 readiness policy가 필요하다.

## 19. Implementation Acceptance Concerns

이 목록은 구현 명령이 아니라 후속 implementation review의 acceptance concern이다.

- module dependency와 forbidden framework type architecture test
- Java-friendly public contract compile/interop test
- fake Product Tool, Retriever, ModelGateway를 사용한 pure application test
- provider/tool/retrieval adapter contract와 integration test
- current authorization revalidation과 scope non-widening test
- projection deletion/invalidation/stale-scope test
- streaming cancellation/disconnect/timeout와 bounded concurrency test
- provider executor, connection pool, retry, circuit breaker와 bulkhead isolation
- sensitive payload 기본 비관측과 correlation continuity 검증

## 20. Related Records와 Review Gate

- [ADR-0001](ADR-0001-skill-lens-workflow-execution-ownership.md)
- [ADR-0002](ADR-0002-evidence-provenance-core-domain-boundary.md)
- [ADR-0003](ADR-0003-retrieval-knowledge-runtime-ownership-lifecycle.md)
- [ADR-0004](ADR-0004-execution-context-api-tenant-isolation.md)
- [ADR-0005](ADR-0005-runtime-placement-platform-core-modular-monolith-boundary.md)
- `harness-foundation-docs` ADR-0018 (Stage A placement owner record)

Independent review는 최소 다음을 확인한다.

- Stage A placement와 canonical ownership invariant가 보존되는가
- Kotlin/Java public contract와 Spring AI adapter boundary가 충분히 명시적인가
- Product-specific orchestration이 application handler에 남는가
- Tool/Retriever/Evidence가 authorization 또는 Semantic SoT로 오해되지 않는가
- compatibility anchor와 Gateway/Platform Core build isolation이 정확한가
- Python/LangGraph trigger가 future option을 보존하되 Phase 1 runtime을 만들지 않는가

이 ADR이 review를 통과하고 canonical `main`에 merge되기 전까지 Stage B를 canonical complete로
표현하지 않는다. Merge 후에도 production implementation은 별도 task다.
