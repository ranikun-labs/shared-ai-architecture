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

## 상태

```text
repository: bootstrap
runtime code: 없음 (문서 전용)
Shared AI physical runtime: not_started / deferred (harness-foundation ADR-0015, ADR-0013 기준)
```

RPL-97 Foundation 문서는 `docs/RPL-97-shared-ai-architecture-foundation` branch에서 작성되며,
독립 Architecture Review 전까지 merge하지 않는다.
