# Product ↔ Shared AI Responsibility

| 항목 | 값 |
| --- | --- |
| status | draft (RPL-97 Foundation) |
| owner | shared-ai-architecture |
| 상위 canonical | `ADR-0015` §7.4, boundaries §10~§11, `authenticated-carelog-context.md` |

---

## 1. Purpose

Product Backend와 Shared AI가 각각 무엇을 소유하고, 둘 사이가 어떤 경로로 연결되는지를
고정한다. 핵심 목표는 **Shared AI가 Product Backend를 대체하거나 Product DB를 직접 알지
않도록** 하는 것이다.

## 2. 기본 경계 원칙

```text
Client
  → Gateway
  → Product Backend        (business SSOT, authz, mutation, domain decision)
  → Shared AI API          (실행 요청: model/tool/retrieval/lens 조합)
      → Product Tool/API Adapter → Product Backend → Product DB   (사실값 조회)
      → Retriever                                                  (비정형 history)
      → Model Gateway → LLM
  → Evidence + Result
  → Product Backend        (검증·저장·업무 반영, 최종 책임)
```

- Shared AI는 Product DB table, persistence 구현, command transaction semantics를 **직접 알지 않는다.**
- Product의 정확한 사실값(미납 여부, 예약 날짜, 통증 score, journal metadata, market data)은
  **Shared AI가 Product Tool/API를 통해서만** 얻는다. Vector RAG가 사실값 SSOT가 되지 않는다.

## 3. Ownership split

### 3.1 Product Backend 소유

```text
Business Data SSOT (정확한 상태/금액/날짜/score)
Authorization / Product 권한 판단
Business Rules / Commands / Mutation
Product DB와 migration
제품별 System Prompt / Domain Context 선택
Product Workflow / Use Case(Skill) 정의
Domain Policy / Guardrail 규칙 원문
Corpus 선택 / Retrieval Policy
Domain 결과 검증 / 결과 저장 / 업무 반영 / 최종 책임
제품별 규제·승인·보존 정책
```

### 3.2 Shared AI 소유

```text
Model Invocation / Model Gateway (provider, key, alias, timeout, retry, fallback)
Rate limit / concurrency / usage / cost metering
Tool orchestration (Product Tool adapter 호출 규약)
Retrieval orchestration (tenant-scoped; corpus·policy는 Product 소유)
Skill / Lens 실행 계약 (candidate, NEEDS_ADR — 정의는 Product/Finance)
Evidence composition / provenance
Prompt/model version awareness (버전 인지; prompt 내용은 Product)
AI-specific trace / evaluation 실행 substrate
기술적 Safety / Streaming(SSE)
```

Shared AI의 "orchestration"은 Tool/Retriever 등을 호출·실행하는 mechanism(호출 순서·규약)만
의미한다. 어떤 Tool/Retriever/Lens를 선택할지에 대한 Product workflow semantics는 Shared AI가
소유하지 않으며, 계속 §3.1의 `Product Workflow / Use Case(Skill) 정의`에 귀속된다
(`ADR-0015` §7.4 정렬).

## 4. Multi-product / multi-tenant context

Shared AI의 모든 실행 경로(Tool·Retriever·Evidence)에서 아래 scope가 유실되지 않아야 한다.

```text
product      (carelog | finance | ...)
tenant / workspace   (Carelog Workspace, Finance user workspace 등)
principal / user     (identityUserId 등 stable id)
resource / customer  (Carelog Customer, Finance journal 대상 등)
```

원칙:

1. **Tenant isolation 없는 global RAG 금지.** Retriever는 항상 scope로 필터한다.
2. **Shared AI는 authz를 판단하지 않는다.** scope를 *전파*하고, 실제 접근 권한 판단은
   Product Backend가 한다(`authenticated-carelog-context.md`: identity는 authz의 입력일 뿐
   Carelog-local authorization을 대체하지 않는다).
3. **지금 물리 DB를 tenant별로 쪼개지 않는다.** Logical isolation(scope 필드·필터)과
   future physicalization seam을 구분한다. Platform의 "같은 물리 cluster ≠ 같은 owner" 원칙과 정렬.
4. `identityUserId`/customer id는 Shared AI 저장 계층에서도 **weak reference**로만 쓰고,
   Identity/Product DB에 물리 FK를 걸지 않는다.

## 5. 연결 방식 (통신)

- 동기 실행 요청: HTTP/JSON (ADR-0015 §7.1). 내부 호출은 외부 Gateway를 우회하지 않는다.
- Token streaming: SSE, flow `Shared AI → Product → Gateway → Client` (ADR-0015 §7.5).
- 장시간 AI Job / usage·cost 비동기 기록: NATS JetStream 후보 (ADR-0015 §7.6, deferred — 첫 use case 시).
- Shared AI → Product Tool 호출에도 timeout·제한 retry·circuit breaker를 적용한다(멱등/Idempotency-key 요청만 retry).

## 6. 예외 조건

Shared AI가 Product 사실값을 직접 알아야 하는 예외는 기본적으로 없다. 성능상 read model이
필요하면, Product가 소유하는 **projection/read model 또는 event**로 소비하며(cross-service
JOIN·직접 table read 금지), 그 projection의 SoT와 migration은 Product가 소유한다.

## 7. Explicit non-goals

```text
- Shared AI가 Product 상태를 직접 mutation
- Shared AI가 Product authz 판단을 대체
- Shared AI가 Product DB에 직접 접속/FK/JOIN
- LLM/Retriever 결과를 검증 없이 Product 사실/mutation으로 승격
- 물리 tenant DB 분리 설계(이번 Slice)
```

## 8. Open decisions

| # | Open decision | 분류 |
| --- | --- | --- |
| OD-5 | Shared AI ↔ Product Tool 호출의 인증(service-to-service credential vs trusted context) | deferred (ADR-0015 follow-up) |
| OD-6 | Retriever가 소비할 tenant-scoped corpus의 물리 저장 위치/소유 | NEEDS_ADR |
| OD-7 | Provider 장애·Identity 비가용 시 fail-closed/fail-open 정책 (Carelog는 아직 candidate follow-up) | NEEDS_ADR |

## 9. Related canonical sources

- `harness-foundation-docs/docs/adr/ADR-0015-...` §7.1, §7.4, §7.5, §7.6, §7.8
- `harness-foundation-docs/docs/architecture/repository-service-boundaries.md` §10, §11
- `carelog-architecture/contracts/authenticated-carelog-context.md`
- `carelog-architecture/architecture/ownership-map.md`
