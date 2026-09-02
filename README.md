# Shared AI Architecture

이 Repository는 Ranikun Labs의 여러 Product(Carelog, Finance Harness 및 향후 Product)가
공통으로 사용할 **Shared AI Runtime의 canonical architecture와 responsibility boundary
문서**를 소유한다. 실제 runtime code(Python/FastAPI 등)를 소유하지 않으며, 문서 수준의
책임 경계·실행 모델·계약 초안만 관리한다.

## 이 Repository가 하는 것 / 하지 않는 것

| 하는 것 | 하지 않는 것 |
| --- | --- |
| Shared AI의 책임 경계와 non-goal 정의 | Product Prompt·Workflow·Domain Policy 소유 |
| Product ↔ Shared AI 경계 계약(문서) | Finance Lens semantics, Carelog business rule 소유 |
| 실행 모델과 capability 분리(Skill/Lens/Tool/Retriever/Model) | Platform 통신·배포·데이터 소유권 재정의 |
| Consumer use case로 구조 검증 | 실제 API schema·pgvector schema·framework 확정 |

## Canonical authority

이 Repository는 상위 canonical source를 **재정의하지 않고 참조**한다.

| Repository / system | Canonical responsibility |
| --- | --- |
| `harness-foundation-docs` | Platform-wide Architecture Constitution, ADR, Shared AI ↔ Product 상위 경계(`ADR-0015`) |
| `carelog-architecture` | Carelog Product Architecture, Customer/Workspace/Event ownership |
| `finance-harness-docs` | Finance Lens / PolicyGuard / Routing / Knowledge / Professional Standards |
| `shared-ai-architecture` (이 repo) | Shared AI runtime responsibility boundary와 실행 모델 문서 |

## Document map (RPL-97 Foundation)

| 영역 | 문서 |
| --- | --- |
| 헌장·경계 | [shared-ai-charter](docs/shared-ai-charter.md) |
| Canonical owner·충돌 | [source-of-truth-map](docs/source-of-truth-map.md) |
| Product ↔ Shared AI 책임 | [product-ai-responsibility](docs/product-ai-responsibility.md) |
| 실행 모델 | [execution-model](docs/execution-model.md) |
| Capability 분리 | [skill-lens-tool-retriever-model](docs/skill-lens-tool-retriever-model.md) |
| Consumer 검증 | [consumer-use-case-matrix](docs/consumer-use-case-matrix.md) |
| 실행 ownership ADR | [ADR-0001 Skill/Lens/Workflow Execution Ownership](docs/adr/ADR-0001-skill-lens-workflow-execution-ownership.md) (OD-1/OD-4 확정) |
| Evidence/Provenance ADR | [ADR-0002 Evidence/Provenance Core와 Domain Extension 경계](docs/adr/ADR-0002-evidence-provenance-core-domain-boundary.md) (OD-8 확정) |
| Retrieval/Knowledge ADR | [ADR-0003 Retrieval/Knowledge Runtime Ownership과 Projection Lifecycle](docs/adr/ADR-0003-retrieval-knowledge-runtime-ownership-lifecycle.md) (OD-2/OD-6 확정) |
| Execution Context/Isolation ADR | [ADR-0004 Execution Context·API Boundary·Tenant Isolation](docs/adr/ADR-0004-execution-context-api-tenant-isolation.md) (identity≠authz·query-time authority·fail-closed isolation; OD-5 부분·OD-7 security-boundary 확정) |
| Phase 1 Runtime Placement ADR | [ADR-0005 Platform Core Modular Monolith Boundary](docs/adr/ADR-0005-runtime-placement-platform-core-modular-monolith-boundary.md) (RPL-103 Stage A accepted/effective 2026-08-26; Foundation ADR-0018 coordinated acceptance) |
| Phase 1 Runtime Technology ADR | [ADR-0006 Runtime Technology, Framework와 Dependency Boundary](docs/adr/ADR-0006-runtime-technology-framework-dependency-boundary.md) (RPL-103 Stage B accepted/effective 2026-08-26; PR #7 canonical projection complete) |
| Current OpenAI/GPT Provider Target ADR | [ADR-0007 Current OpenAI/GPT Implementation Target and Provider-Neutral Boundary](docs/adr/ADR-0007-current-openai-gpt-provider-target.md) (current model-inference target; exact GPT model, embeddings, and reranker deferred) |

Decision status와 implementation status는 분리한다. 문서 존재가 구현·배포·runtime 활성화를
의미하지 않는다.

## 상태

```text
repository: bootstrap
runtime code: 없음 (문서 전용)
Shared AI Phase 1 placement: platform-core same-JVM logical module (accepted/effective 2026-08-26)
Shared AI neutral foundation: canonical in platform-services (RPL-106)
Current implementation projection: RPL-107 synchronous OpenAI Slice A = IMPLEMENTED;
                              see docs/source-of-truth-map.md §2.1
Current model-inference provider target: OpenAI API / GPT family (ADR-0007; exact GPT model deferred)
Runtime technology: Kotlin + Spring MVC, Spring AI=adapter/integration library,
                              PostgreSQL+pgvector baseline, no Phase 1 Python worker
Stage B decision status: accepted/effective 2026-08-26; canonical main projection complete via PR #7
Product-facing AI feature, runtime support, deployment and release: not claimed
```

The mutable current implementation/status package is maintained in
[source-of-truth-map §2.1](docs/source-of-truth-map.md#21-current-implementation-projection).
This repository remains documentation-only; the implementation evidence belongs to the
external `platform-services` repository.

RPL-97 Foundation 문서는 `docs/RPL-97-shared-ai-architecture-foundation` branch에서 작성되며,
독립 Architecture Review 전까지 merge하지 않는다.
