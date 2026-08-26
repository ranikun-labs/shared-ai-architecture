# Shared AI Charter

| 항목 | 값 |
| --- | --- |
| status | draft (RPL-97 Foundation) |
| owner | shared-ai-architecture |
| jira | RPL-97 |
| decision posture | Architecture foundation. 구현·배포·framework 확정 아님 |
| 상위 canonical | `harness-foundation-docs` `ADR-0015` §7.4, `ADR-0013`, `repository-service-boundaries.md` §9 |

---

## 1. Purpose

이 문서는 Shared AI가 **무엇을 책임지는 서비스인가**를 한 문장 경계로 고정한다.
Carelog, Finance Harness, 향후 Product가 각자 AI runtime을 중복 구축하지 않고 하나의
공통 실행 substrate를 사용할 수 있게 하되, 각 Product의 domain ownership을 흡수하지
않도록 한다.

RPL-97의 성공 기준은 문서 수가 아니라, **두 실존 Product(Carelog·Finance)와 미래 Product가
같은 Shared AI Execution Core를 쓸 수 있는 최소한의 안정적 책임 경계를 확정**하는 것이다.

## 2. Charter statement

```text
Shared AI는 Product가 AI 작업을 요청하면,
Model 실행 / Tool 호출 / Retrieval / (실행 계약 수준의) Skill·Lens 조합 / 기술적 Safety를
tenant scope와 함께 수행하고, 결과를 Evidence와 함께 반환하는 공통 AI Execution Runtime이다.

Shared AI = "AI를 어떻게 실행하는가"
Product   = "AI가 이 제품에서 무엇을 해야 하는가"
```

이 문장은 새로운 발명이 아니라 `harness-foundation-docs/ADR-0015` §7.4에서 이미 승인된
경계(`Shared AI = AI를 어떻게 실행하는가 / Product = AI가 이 제품에서 무엇을 해야 하는가`)를
RPL-97 use case로 구체화한 것이다.

## 3. Ownership

### 3.1 Shared AI가 소유하는 것 (confirmed — ADR-0015 §7.4 정렬)

```text
Provider Adapter / 다중 Provider 실행
AI Provider API Key·Secret 소유 책임
Model Alias / Model Gateway
Timeout / 제한적 Retry / Fallback
Rate Limit / Concurrency
Usage / Cost Metering
공통 Observability / Runtime Trace
Provider 장애 처리
기술적 Safety와 Streaming(SSE) 기술
Tenant scope 전파 (product / workspace / principal / resource)
Evidence envelope 조립(어떤 Model/Tool/Retriever/Document decision에 근거했는지)
```

### 3.2 Shared AI가 소유하는 제품 중립 capability (follow-up ADR에서 확정)

`ADR-0015` §7.4와 §10은 아래를 처음에는 "제품 중립 기술 Capability 후보"로만 인정하고, 실제
use case 발생 시 후속 Decision으로 소유권을 확정하라고 명시했다. RPL-97 use case가 그 trigger다.

```text
Provider-neutral Tool orchestration 계약
Provider-neutral Retriever / RAG Adapter, Embedding Runtime, Vector Infrastructure
공통 Skill 실행 계약 (Skill 정의가 아니라 실행 interface)
공통 Lens 실행 계약 (Lens semantics가 아니라 실행 interface)
공통 AI Orchestration / AI Job Runtime
```

> Skill/Lens **실행 계약**과 공통 orchestration의 소유는 [ADR-0001](adr/ADR-0001-skill-lens-workflow-execution-ownership.md)에서
> confirmed(실행 mechanism = Shared AI, 정의·선택·검증 = Product/Domain)로 확정되었다. Retriever/RAG·
> Embedding·Index projection은 [ADR-0003](adr/ADR-0003-retrieval-knowledge-runtime-ownership-lifecycle.md)에서
> confirmed(semantic SoT·admission·selection = Product/Domain, projection·lifecycle·retrieval execution =
> Shared AI)로 확정되었다. 물리 technology는 RPL-103으로 deferred한다. Execution Context 계약·tenant/resource
> isolation·identity≠authz·query-time authority는 [ADR-0004](adr/ADR-0004-execution-context-api-tenant-isolation.md)가
> confirmed로 확정한다(OD-5 부분, OD-7 security-boundary).

→ 상세 경계와 충돌은 [source-of-truth-map](source-of-truth-map.md),
[skill-lens-tool-retriever-model](skill-lens-tool-retriever-model.md) 참조.

### 3.3 Shared AI가 소유하지 않는 것 (Product / Product-docs 소유)

```text
제품별 System Prompt              → Product
Product Workflow / Use Case 정의   → Product
Domain Policy / Guardrail 규칙 원문 → Product / Product-docs
Domain Context·입력 데이터 선택     → Product
Corpus 선택 / Retrieval Policy      → Product
Domain 결과 검증 / 최종 책임        → Product
결과 저장과 업무 반영               → Product
Finance Lens semantics / PolicyGuard / Output Validator 규칙 → finance-harness-docs
Carelog business rule / Customer·Workspace ownership          → carelog-architecture
Platform 통신·배포·데이터 소유권    → harness-foundation-docs
```

## 4. Guiding principles

1. **Framework는 implementation detail이다.** LangGraph/RAGFlow/LlamaIndex를 먼저 고르고
   구조를 맞추지 않는다. Use case → capability → 책임 경계 → 실행 모델 → 계약 → framework 순서.
2. **모든 요청이 RAG/Agent/embedding이 아니다.** 정확한 사실은 Tool(Product SSOT),
   비정형 history는 Retriever, 관점은 Lens로 분리한다. ([execution-model](execution-model.md))
3. **Tenant scope 없는 retrieval 금지.** ([product-ai-responsibility](product-ai-responsibility.md) §4)
4. **LLM 결과는 검증 없이 Product mutation으로 승격하지 않는다.** Evidence·Human Review 유지.
   `harness-foundation` Candidate/Promotion 원칙과 정렬.
5. **Product별 if/else를 Shared AI Core에 쌓지 않는다.** Provider-neutral, product-neutral.

## 5. Explicit non-goals (이번 Slice)

```text
- Python/FastAPI runtime 구현
- 실제 API schema 확정
- pgvector schema / embedding model / chunk size / reranker 확정
- LangGraph workflow / RAGFlow deployment / GraphRAG 확정
- Skill Marketplace / Dynamic Plugin Loader / Workflow DSL
- dedicated Vector DB / full Prompt Management Platform
- Finance Lens 판단 내용 또는 Carelog business rule의 Shared AI 이전
- Platform 통신/배포/데이터 소유권 재정의
```

## 6. Open decisions

| # | Open decision | 분류 |
| --- | --- | --- |
| OD-1 | Shared AI가 "Skill/Lens 실행 계약"을 소유하는가, 아니면 Model Gateway까지만 소유하고 Skill/Lens 실행은 각 Product runtime에 두는가 | **RESOLVED — [ADR-0001](adr/ADR-0001-skill-lens-workflow-execution-ownership.md)** (실행 mechanism 소유, 정의·선택·검증은 Product/Domain) |
| OD-2 | Retriever/RAG·Embedding·Index projection의 Shared AI 소유 여부와 범위 | **RESOLVED — [ADR-0003](adr/ADR-0003-retrieval-knowledge-runtime-ownership-lifecycle.md)** (semantic SoT·admission·selection=Product/Domain, projection·lifecycle·execution=Shared AI) |
| OD-3 | Shared AI physical runtime 위치와 도입 시점 | **IN REVIEW — [ADR-0005](adr/ADR-0005-runtime-placement-platform-core-modular-monolith-boundary.md)**는 Phase 1 `platform-core` same-JVM placement를 선택하며 Foundation ADR-0018 coordinated acceptance가 필요. Language/framework/version과 구현은 deferred. |
| OD-4 | Finance의 기존 자체 runtime contract와 Shared AI 실행 계약의 관계(채택/병존) | **RESOLVED — [ADR-0001](adr/ADR-0001-skill-lens-workflow-execution-ownership.md)** (병존 + opt-in 채택) |

## 7. Related canonical sources

- `harness-foundation-docs/docs/adr/ADR-0015-platform-communication-messaging-scaling.md` §7.4, §7.5, §7.6, §10
- `harness-foundation-docs/docs/adr/ADR-0013-target-deployment-and-data-boundaries.md`
- `harness-foundation-docs/docs/architecture/repository-service-boundaries.md` §9, §13.3
- `harness-foundation-docs/docs/master/product-architecture-master.md` §22
- `carelog-architecture/contracts/authenticated-carelog-context.md`
- `finance-harness-docs/00-core/0-c-runtime-contract/finance-routing-contract.md`
- `finance-harness-docs/00-core/0-d-knowledge-architecture/finance-knowledge-storage-decision.md`
