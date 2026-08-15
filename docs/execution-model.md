# Execution Model

| 항목 | 값 |
| --- | --- |
| status | draft (RPL-97 Foundation) |
| owner | shared-ai-architecture |
| 상위 canonical | `ADR-0015` §7.4~7.6, `finance-routing-contract.md`(finance domain 예시) |

---

## 1. Purpose

Shared AI가 하나의 요청을 어떻게 실행하는지의 **공통 실행 골격**을 정의한다. 목표는
API endpoint마다 별도 AI pipeline을 하드코딩하지 않고, Skill이 Tool/Retriever/Lens/Policy를
조합하는 단일 실행 모델을 갖는 것이다.

## 2. Core execution shape

```text
API Request (product + tenant scope + skill ref + input)
  → Execution Context 구성 (scope 전파, provider 무관)
  → Skill 실행 계약   (무엇을 수행할지 = Product/Domain 소유 정의)
      ├─ Tool         (결정론적 사실값 / Product API·계산 / market data)
      ├─ Retriever    (비정형 history/knowledge, tenant-scoped)
      ├─ Lens         (관점; 정의·semantics는 Domain 소유)
      └─ Policy       (허용/금지; 규칙 원문은 Product/Domain 소유)
  → Model Gateway → LLM
  → Output (기술적 safety 통과)
  → Evidence + Result
```

핵심:

```text
Skill ≈ Application Service / Use Case
Shared AI Core는 "실행 골격"을 제공하고,
Skill/Lens/Policy의 내용은 Product/Domain이 채운다.
```

## 3. 실행 형태 결정 (Workflow vs Agent)

| 형태 | 언제 | 원칙 |
| --- | --- | --- |
| Workflow | 실행 조합이 사전에 정해진 경우 (대부분의 초기 use case) | 기본값. 결정론적·검증 가능·저비용 |
| Agent | 실행 중 다음 action/tool을 동적으로 판단해야 하는 경우에만 | 예외. 모든 요청을 Agent로 만들지 않는다 |

- 초기 RPL-97 consumer use case 3종은 전부 Workflow로 표현 가능하다
  ([consumer-use-case-matrix](consumer-use-case-matrix.md)).

## 4. Anti-pattern 금지 (실행 모델 차원)

```text
- 모든 질문 → RAG
- 모든 요청 → Agent
- 모든 데이터 → embedding
- endpoint마다 별도 pipeline 하드코딩
- Product별 if/else를 Core에 누적
- Vector RAG를 정확한 사실값(금액/날짜/score)의 SSOT로 사용
- LLM 결과 → 검증 없이 Product mutation
- Tenant scope 없는 retrieval
- Prompt = Skill, Tool = Skill (개념 혼합; skill-lens-tool-retriever-model 참조)
```

## 5. Safety / Policy 위치

- **기술적 Safety**(provider-level content safety, streaming 안정성)는 Shared AI 소유.
- **Domain Policy / Guardrail 규칙**(예: Finance PolicyGuard의 intent 분류·redirect·output
  validation 규칙 원문)은 **Domain 소유**. Shared AI는 이 Policy를 실행 흐름의 gate로
  *호출*할 뿐 규칙을 정의하지 않는다.
- Finance의 경우 Input Guard → intent → guard action → lens/redirect → template → Output Validator
  순서 자체가 finance-harness-docs의 canonical runtime contract다. Shared AI는 이 순서를
  재정의하지 않으며, 그 아래의 Model Gateway·trace·streaming만 제공할 수 있다(OD-4, NEEDS_ADR).

## 6. Streaming / async

- 사용자 대면 token streaming: SSE (`Shared AI → Product → Gateway → Client`).
- 장시간 job / usage·cost 기록 / batch ingestion: NATS JetStream 후보 (deferred, 첫 use case 시).
  도입 시 at-least-once + idempotent consumer + DB commit 후 ACK (ADR-0015 §7.6).

## 7. Evidence

모든 결과는 어떤 실행 근거에서 나왔는지를 Evidence로 동반한다.

```text
Evidence 후보 필드 (core envelope):
  used_tools[]        (tool id, input scope, 결과 요약 ref)
  retrieved_docs[]    (doc ref, tenant scope, timestamp)
  applied_lenses[]    (lens key/version; semantics는 domain)
  policy_decisions[]  (guard action 결과 ref)
  model_calls[]       (provider, model alias, version)
  timestamps / trace id
```

- Evidence **core envelope**는 Shared AI 후보, domain별 확장(예: Finance context snapshot,
  lens version, market data timestamp)은 Domain 소유 → core/domain 경계는 NEEDS_ADR(OD-8).

## 8. Explicit non-goals

```text
- LangGraph/RAGFlow workflow 구현
- 실제 pipeline orchestration engine 선택
- Evidence의 wire schema 확정
- Finance runtime contract를 Shared AI로 대체 (병존/채택은 NEEDS_ADR)
```

## 9. Open decisions

| # | Open decision | 분류 |
| --- | --- | --- |
| OD-4 | Finance 기존 runtime pipeline과 Shared AI 실행 골격의 관계 | **RESOLVED — [ADR-0001](adr/ADR-0001-skill-lens-workflow-execution-ownership.md)** (병존 + opt-in; Shared AI는 선언된 순서·gate를 재정의 없이 집행) |
| OD-8 | Evidence core envelope vs domain extension 경계와 schema | NEEDS_ADR |
| OD-9 | Workflow orchestration 구현 방식(코드 vs DSL vs LangGraph) | deferred |

## 10. Related canonical sources

- `harness-foundation-docs/docs/adr/ADR-0015-...` §7.4~7.6
- `finance-harness-docs/00-core/0-c-runtime-contract/finance-routing-contract.md`
- `finance-harness-docs/00-core/0-a-safety-core/finance-policyguard-rules.md`
