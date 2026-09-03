# Execution Model

| 항목 | 값 |
| --- | --- |
| status | draft (RPL-97 Foundation + Runtime Evolution follow-up) |
| owner | shared-ai-architecture |
| decision posture | Phase 1 bounded workflow default; LLM reasoning is controlled by runtime code; future agent/framework evaluation only |
| 상위 canonical | `ADR-0015` §7.4~7.6, [ADR-0001](adr/ADR-0001-skill-lens-workflow-execution-ownership.md), [ADR-0002](adr/ADR-0002-evidence-provenance-core-domain-boundary.md), [ADR-0004](adr/ADR-0004-execution-context-api-tenant-isolation.md), [ADR-0006](adr/ADR-0006-runtime-technology-framework-dependency-boundary.md), [ADR-0007](adr/ADR-0007-current-openai-gpt-provider-target.md) |

---

## 1. Purpose

Shared AI가 하나의 요청을 어떻게 실행하는지의 **공통 실행 골격**과 Runtime Evolution 원칙을
정의한다. 목표는 API endpoint마다 별도 AI pipeline을 하드코딩하지 않고, Product가 선택한
Skill/Workflow와 허용 capability를 Shared AI가 bounded·typed하게 실행하는 것이다.

이 문서는 구현 framework나 generic agent framework를 선택하지 않는다. 현재 첫 model-inference
target은 [ADR-0007](adr/ADR-0007-current-openai-gpt-provider-target.md)에 따라 OpenAI API / GPT
family이고, Spring AI는 integration adapter다. Exact GPT model/version, embeddings, reranker,
Pydantic AI와 Python runtime은 이 문서에서도 선택하지 않는다.

현재 implementation/status의 primary projection은
[source-of-truth-map §2.1](source-of-truth-map.md#21-current-implementation-projection)이다.
그 projection에서 RPL-107 OpenAI synchronous Slice A는 neutral `ModelGateway` 뒤의
`IMPLEMENTED` bounded provider path로 기록된다. 이 문서는 실행 mechanics를 소유하며,
Product integration·user-facing feature·runtime support·deployment·release 또는 Streaming
Slice B를 구현됐다고 주장하지 않는다.

## 2. Core execution shape

Execution Context 계약, identity≠authz, query-time 현재 authorization, tenant/resource isolation과
fail-closed 경계는 [ADR-0004](adr/ADR-0004-execution-context-api-tenant-isolation.md)가 확정한다.
Product가 Skill/Workflow와 허용 capability를 선택하고, Shared AI가 그 선언을 실행 mechanism으로
집행한다([ADR-0001](adr/ADR-0001-skill-lens-workflow-execution-ownership.md)).

```text
Input
  ↓
Context / Candidate Construction
  ↓
LLM Reasoning                         (probabilistic, untrusted)
  ↓
Typed / Structured Result              (schema-shaped, not yet trusted)
  ↓
Runtime / Mechanical Validation        (Shared AI)
  ↓
Product Semantic / Policy Validation   (Product/Domain)
  ↓
SUCCESS / ABSTAIN-UNKNOWN / REJECTED / FAILED
  ↓
Only authorized side effects
  ↓
Evidence + Execution Trace
```

실행에 `Tool`, `Retriever`, `Lens`, `Policy`가 필요하면 Context/Candidate Construction과 선언된
Workflow plan 안에서 준비한다.

```text
Product request
  → authoritative Execution Context + selected Skill/Workflow
  → bounded typed plan / explicit handler
      ├─ Tool       (결정론적 사실값 / Product API·계산 / market data)
      ├─ Retriever  (비정형 history/knowledge, tenant-scoped)
      ├─ Lens       (관점; 정의·semantics는 Domain 소유)
      └─ Policy     (허용/금지; 규칙 원문은 Product/Domain 소유)
  → Model Gateway → LLM
  → structured result → validation gates → result/effect decision
```

핵심:

```text
Skill ≈ Application Service / Use Case
Shared AI Core는 "실행 골격"을 제공하고,
Skill/Lens/Workflow/Policy의 의미와 내용은 Product/Domain이 채운다.
```

## 3. LLM controlled by code

LLM은 확률적 reasoning component다. Prompt가 원하는 행동을 설명할 수는 있지만, Prompt만으로
시스템 보장이 생기지는 않는다.

```text
Prompt says X
!=
System guarantees X
```

LLM output은 **untrusted probabilistic output**으로 취급한다. 코드/runtime이 다음 enforceable
boundary를 소유한다.

```text
- correctness boundary와 불변식
- Execution Context와 tenant/resource isolation
- authorization enforcement seam과 execution-scoped capability 확인 (의미=Product, mechanism=runtime)
- tool permission enforcement mechanism
- deterministic calculation과 Product SSOT 접근 경로
- provider-neutral Model Gateway를 통한 model invocation
- Tool/Retriever invocation mechanism
- schema/parsing/validation
- side-effect 실행 authority의 mechanical gate
- execution/step state lifecycle mechanics
- timeout / cancellation / retry safety / duplicate invocation handling
- failure handling, outcome mapping과 operational traceability
```

Prompt는 reasoning guidance, context 설명, 표현 형식과 abstention 지침에 유용하다. 그러나
Prompt, model tool choice, structured-output 선언 어느 것도 authorization, semantic truth,
side-effect permission 또는 Product mutation authority를 대신하지 않는다.

## 4. Structured result와 validation boundary

**Structured Output exists**와 **semantic correctness is guaranteed**는 서로 다른 주장이다.
Typed/structured 결과는 parsing 이후의 transport shape일 뿐, Product 내부의 신뢰된 domain
object로 자동 승격되지 않는다.

### 4.1 Runtime / Mechanical Validation — Shared AI

Shared AI는 provider-neutral mechanics로 다음을 검사하고, 위반 시 결과나 capability 실행을
진행하지 않는다.

```text
- input/output schema와 parsing
- required field
- enum / discriminated-union validity
- malformed result rejection
- tool argument schema
- declared execution/tool reference existence
- provider-response shape
- Execution Context / tenant / resource consistency
- child capability scope non-widening
- evidence/source/execution reference correlation
```

이 검사는 universal domain schema나 사실 판정기가 아니다. 예를 들어 `required`라는 기계적
필드 존재는 Finance가 요구하는 evidence sufficiency를 의미하지 않는다.

### 4.2 Product Semantic / Policy Validation — Product/Domain

Product/Domain은 결과가 해당 업무에서 충분하고 허용되는지 판단한다.

| 구분 | Shared AI가 검사할 수 있는 것 | Product/Domain이 판단하는 것 |
| --- | --- | --- |
| Finance | result schema, allowed tool, valid context, referenced evidence/tool result existence | Earnings evidence가 충분한가, Lens set이 의미상 충분한가, Finance policy를 위반했는가, `UNKNOWN`으로 내려야 하는가 |
| Carelog | customer/tool reference shape, tenant context consistency, valid tool-call shape | customer가 실제 follow-up 대상인가, contact/send가 허용되는가, duplicate business action인가, draft와 send를 어떻게 구분하는가 |

Shared AI는 Product validator/policy hook을 선언된 위치에서 호출하고 그 결과를 집행하지만,
검증 규칙의 의미나 domain schema를 소유하지 않는다.

### 4.3 Safety / Policy 위치

- **기술적 Safety**(provider-level safety, streaming 안정성, resource limit)는 Shared AI가
  집행한다.
- **Domain Policy / Guardrail 규칙 원문**과 semantic gate는 Product/Domain이 소유한다. Shared AI는
  Product가 선언한 gate를 호출·순서대로 집행할 뿐 규칙을 정의하거나 완화하지 않는다.
- Finance의 canonical order인 `Input Guard → intent → guard action → lens/redirect → template →
  Output Validator`는 Finance가 소유한다. Shared AI가 이 pipeline을 실행 substrate로 제공할 수는
  있지만, 순서·규칙·validator 의미를 재정의하지 않는다([ADR-0001](adr/ADR-0001-skill-lens-workflow-execution-ownership.md)).

## 5. Execution outcome semantics

Product UX와 retry 판단에 충분한 최소 4개 first-class outcome을 사용한다. 세부 원인은
operational trace에 남기되 Product-facing exception hierarchy를 거대하게 만들지 않는다.

| Outcome | 의미 | 기본 처리 방향 |
| --- | --- | --- |
| `SUCCESS` | 필요한 runtime·semantic validation을 통과한 결과 | Product가 표시·저장·허용된 후속 action을 결정 |
| `ABSTAIN / UNKNOWN` | evidence/context 부족, model abstention/refusal, 또는 의도적으로 no decision | 안전한 비결정으로 표시; 자동 mutation/side effect 금지; 추가 context 여부는 Product가 결정 |
| `REJECTED` | policy, validation, tool permission, invariant 또는 capability gate가 거부 | 원인 보정 없는 blind retry 금지; Product가 사용자/흐름을 안내 |
| `FAILED` | provider/tool/retrieval/runtime infrastructure failure, timeout/cancellation, 또는 unrecoverable malformed execution | 안전한 경우에만 제한 retry; side effect는 성공 확인 전 재실행하지 않음 |

예를 들어 retrieval **empty**는 Product semantic 판단에 따라 `UNKNOWN`이 될 수 있지만,
retrieval **failed**는 보통 `FAILED`다. schema rejection처럼 처리 가능한 validation 거부는
`REJECTED`로, 응답을 해석할 수 없어 execution을 계속할 수 없는 malformed execution은
`FAILED`로 기록할 수 있다. 이 차이는 세부 public error type을 강제하는 것이 아니라 UX·retry·
observability·safe non-execution을 위한 최소 의미 구분이다.

## 6. Tool control

Model tool selection은 authorization이 아니다. Tool 요청은 다음 gate를 통과해야 한다.

```text
LLM Tool Request
  ↓
Allowed Tool Check
  ↓
Argument Schema Validation
  ↓
Execution Context / Tenant Enforcement
  ↓
Product Authorization / Policy Hook
  ↓
Budget / Timeout / Side-effect Rules
  ↓
Tool Execution
```

- allowlist는 Product가 선택한 Skill/Workflow와 execution-scoped capability에서 파생되고,
  Shared AI가 기계적으로 집행한다.
- model-generated argument는 untrusted input이다. Shared AI는 schema/형식을 검사하고,
  Product는 business 의미와 authorization을 검사한다.
- child capability scope는 parent Execution Context보다 넓어질 수 없다. tenant/resource
  mismatch는 fail-closed다.
- read tool과 write/side-effecting tool을 구분한다. write는 Product가 명시적으로 허용한
  capability여야 하고, idempotency 또는 duplicate-safe contract 없이는 retry하지 않는다.
- timeout과 cancellation은 tool invocation까지 전파한다. retry는 provider/tool과 side effect의
  안전성이 확인된 경우에만 budget 안에서 수행한다.
- 중복 invocation은 정상 경로로 간주하지 않는다. idempotency key, deduplication 또는 Product
  command semantics로 보호하며, 모델이 같은 요청을 반복했다는 이유로 자동 승인하지 않는다.

Shared AI는 allowlist, context, schema, timeout, cancellation, retry와 실행 budget의 enforcement
mechanism을 소유한다. Product는 "이 고객에게 이 메시지를 보내도 되는가" 같은 business
permission과 command/mutation semantics를 소유한다. Shared AI는 모델의 요청만으로 Product
상태를 mutation하지 않는다.

## 7. Evidence / Provenance boundary

RPL-100의 경계를 유지한다. 상세 logical field contract와 ownership은
[ADR-0002](adr/ADR-0002-evidence-provenance-core-domain-boundary.md)가 확정한다.

```text
Evidence Core Envelope (logical):
  evidence identity / coarse kind / source refs
  producer + execution/step correlation
  product + tenant/workspace scope
  observed/effective time separation
  core version + typed Domain extension ref
```

Shared AI는 Evidence mechanism, integrity와 execution correlation을 소유한다. Product/Domain은
source truth, admissibility, temporal meaning, uncertainty, evidence sufficiency와 semantic
validation을 소유한다.

Shared AI는 다음 mechanically invalid provenance를 허용하지 않는다.

```text
- nonexistent source/tool/execution reference
- failed execution을 valid evidence로 표현
- model-invented execution/source identity
- 다른 execution 또는 tenant의 evidence를 잘못 연결
```

그러나 Shared AI는 universal truth/fact-check engine이 아니다. Evidence reference가 존재한다는
사실도 Product의 semantic sufficiency나 사실성 검증을 대신하지 않는다. Source Evidence와
Model Interpretation, Evidence와 Provenance/Trace/Log/Audit/Result를 합치지 않으며 raw Product
payload, unrestricted prompt/PII와 chain-of-thought를 기본 저장·export하지 않는다.

## 8. Failure and execution trace

최종 outcome만으로 세부 원인을 Product contract에 모두 노출하지 않는다. 대신 Shared AI trace가
다음 차이를 운영적으로 보존해야 한다.

| Trace reason | Outcome와의 관계 |
| --- | --- |
| provider/model failure | 보통 `FAILED`; retry 가능성은 error와 budget에 따라 판단 |
| model abstention/refusal | 보통 `ABSTAIN / UNKNOWN`; Product policy가 별도 거부로 판단할 수 있음 |
| retrieval empty | Product semantic validation 결과에 따라 `UNKNOWN` 또는 `REJECTED` |
| retrieval failed | `FAILED` |
| tool rejected | `REJECTED` |
| tool failed | `FAILED` |
| runtime/mechanical validation rejected | `REJECTED`, unrecoverable malformed execution이면 `FAILED` |
| timeout / cancellation | 보통 `FAILED`와 operational reason으로 기록 |
| Product policy rejection | `REJECTED` |

Trace는 최소 다음을 reconstruct할 수 있어야 한다.

```text
executionId
Product / tenant / workspace context
workflow / skill / lens identity와 version
provider/model operational reference
latency / token usage / retry / timeout / cancellation
retrieval calls와 empty/failed 상태
tool calls와 allowed/rejected/failed 상태
mechanical validation와 Product validator 결과
final outcome
```

이 정보로 운영자는 "왜 이 execution이 SUCCESS / UNKNOWN / REJECTED / FAILED로 끝났는가"를
재구성할 수 있어야 한다. Chain-of-thought를 저장하거나 unrestricted prompt/PII를 관측에
복제하는 것은 요구하지 않는다. 민감 payload는 별도 Product policy, redaction과 명시적 enablement
없이는 trace/export 대상이 아니다.

## 9. Phase 1 bounded workflow default

Phase 1의 canonical default는 **Product가 선택한 bounded workflow를 Shared AI가 실행하는 것**이다.

```text
Product selects use case / Skill / Workflow / allowed capabilities
  ↓
Shared AI executes bounded typed plan
  ↓
explicit handlers + controlled ordering
  ↓
Model / Tool / Retriever / Product policy-validator hooks
```

- Product가 Skill/Workflow와 허용 capability를 선택한다. free-form request만 받아 Shared AI가
  업무 의미나 Skill을 임의로 결정하지 않는다.
- Typed handler/application service에서 순서, gate, side-effect 경계를 보이게 한다.
- LLM은 bounded plan 안에서 reasoning하거나 Product가 선언한 allowed set 안에서 제안할 수
  있지만, model tool choice가 allowlist·authorization·policy를 대체하지 않는다.
- 자유로운 LLM routing을 Phase 1 default로 만들지 않는다.
- 현재 model-inference target은 OpenAI API / GPT family이며 Spring AI adapter를 통해 neutral
  contract 뒤에서 사용한다. 이는 permanent/exclusive provider 선택이 아니다. Exact GPT
  model/version·alias mapping·embeddings·reranker는 [ADR-0007](adr/ADR-0007-current-openai-gpt-provider-target.md)대로 deferred다.

기존 RPL-97 consumer use case 3종은 모두 Workflow로 표현 가능하다
([consumer-use-case-matrix](consumer-use-case-matrix.md)).

## 10. Workflow vs Agent and complexity triggers

```text
Workflow : known execution graph + known allowed capabilities + bounded branching
Agent    : runtime에서 다음 action을 동적으로 선택하고 state를 유지
```

Agentic runtime은 다음 요구가 실제로 load-bearing하다는 evidence가 있을 때만 재검토한다.

```text
- dynamic next-action selection이 결과 품질/기능에 필수
- durable graph checkpoint와 process restart/restart-from-state
- long-running execution
- human pause/resume 또는 승인 대기
- replay/recovery가 persistent state를 필요로 함
- persistent branching/state가 bounded plan으로 구조적으로 표현되지 않음
- bounded workflow가 요구사항을 구조적으로 충족하지 못함
```

다음만으로는 Agent framework를 정당화하지 않는다.

```text
- model call이 여러 개임
- tool이 여러 개임
- prompt orchestration이 불편함
- framework marketing이 feature를 agent라고 부름
```

이 trigger가 충족되어도 먼저 same-workflow evidence, state/replay 요구, resource/failure
impact와 side-effect gate를 검토한다. premature multi-agent orchestration은 도입하지 않는다.

## 11. Pydantic AI future-candidate policy

Pydantic AI는 **FUTURE CANDIDATE**일 뿐 현재 선택된 architecture가 아니다.

- Pydantic AI type을 Shared AI neutral contract, Product contract, Evidence 또는 Execution
  Context에 넣지 않는다.
- 이 pass로 Python worker/service를 만들거나 승인하지 않는다. Phase 1은 ADR-0006의
  Kotlin/JVM Core와 Spring AI adapter 경계를 유지한다.
- Spring AI → Pydantic AI를 mandatory runtime chain으로 정의하지 않는다. 두 선택지는 neutral
  port/adapter 뒤의 alternative 또는 필요 시 complementary implementation으로만 평가한다.
- Pydantic AI 도입은 demo speed나 developer ergonomics만으로 결정하지 않는다. 동일 workflow
  비교 gate를 통과하고, JVM-first 운영/유지보수와 resource trade-off가 정당화될 때 별도 review로
  재평가한다.

## 12. Same-workflow Spring AI vs Pydantic AI PoC gate

미래에 Spring AI와 Pydantic AI를 비교하려면 두 구현이 **동일한 representative workflow**를
구현해야 한다. Provider target, input/candidate set, Product policy, allowed tools, acceptance
criteria, timeout/retry budget을 가능한 한 동일하게 고정해 framework 차이와 workflow 차이를
혼동하지 않는다.

| 비교 영역 | 양쪽 구현에서 확인할 것 |
| --- | --- |
| 결과/검증 | typed structured output, malformed output rejection, runtime validation hooks |
| Tool control | tool execution interception, allowlist enforcement, argument validation |
| Context/policy | tenant/context enforcement, Product validator integration |
| Reliability | retry/error semantics, cancellation/timeout behavior |
| Evidence/trace | evidence correlation, execution trace completeness and redaction |
| Testability | unit/contract/integration testing ergonomics and failure injection |
| Operations | operational complexity, memory/runtime footprint, resource isolation and saturation behavior |
| Delivery | deployment complexity, JVM-first platform maintainability, upgrade and rollback burden |

PoC 결과는 성공 데모가 아니라 위의 pass/fail evidence와 비용/운영 측정으로 기록한다. 어느
framework도 이 gate만으로 현재 Product ownership이나 neutral contract를 바꾸지 않는다.

## 13. Finance walkthrough

질문:

> "NVIDIA 투자를 계속 가져가도 되는지 분석"

Finance는 다음을 소유한다.

```text
- Lens 선택과 Lens semantics
- 필요한 evidence와 기준시각/point-in-time 의미
- investment-advice policy와 PolicyGuard
- domain semantic validation / Output Validator
- final synthesis와 Product policy, 사용자에게 허용할 결과
```

Shared AI는 Finance가 선언한 `InvestmentReview` 또는 동등한 bounded analysis plan을 실행한다.

```text
Finance-owned selected plan
  ↓
Shared AI Execution Context / allowed capabilities
  ↓
declared Lens + Retriever / Market/Earnings/Rate/Valuation/FX Tool calls
  ↓
OpenAI/GPT model reasoning through neutral Model Gateway
  ↓
typed result → runtime validation → Evidence correlation
  ↓
Finance semantic validator / policy gate
  ↓
SUCCESS / UNKNOWN / REJECTED / FAILED
```

여기서 필요한 Tool/Lens 집합, required evidence와 순서는 Finance가 선언하는 workflow
requirement다. 단일 prompt에 `check rates, earnings, valuation, FX...`라고 쓰는 것만으로 required evidence,
순서, permission, validation이 보장되지는 않는다. evidence가 부족하면 Finance validator가
`UNKNOWN`을 선택할 수 있고, Shared AI는 그 결과를 자동 투자 mutation이나 거래 side effect로
승격하지 않는다.

## 14. Carelog sanity check

질문:

> "최근 방문 고객 중 follow-up이 필요한 사람을 찾아 메시지 초안을 만들어줘."

이 요청도 동일한 runtime abstraction으로 표현할 수 있다.

```text
Carelog가 bounded follow-up workflow와 allowed read/draft capabilities를 선택
  ↓
Shared AI가 tenant/workspace/customer context를 검증
  ↓
최근 방문/기록 후보를 Product Tool + tenant-scoped Retriever로 구성
  ↓
LLM이 후보 reasoning과 draft language를 typed result로 제안
  ↓
Shared AI mechanical validation
  ↓
Carelog semantic eligibility / contact policy / duplicate validation
  ↓
draft 결과만 반환; send는 별도 authorized Product command
```

LLM은 후보를 reasoning하고 문구를 생성할 수 있지만 다음 경계를 결정하지 않는다.

```text
- real Customer identity와 tenant membership
- contact eligibility와 current authorization
- allowed send tool 및 send authority
- duplicate-send protection / idempotency
- Draft와 Send의 business distinction
```

실제 follow-up eligibility나 메시지 전송 여부는 Carelog가 판단한다. 모델이 `send`를 제안해도
Shared AI가 자동으로 발송하지 않는다. Finance-specific Lens semantics를 Shared AI로 옮기지
않으면서도 동일한 control model이 Carelog의 안전 경계를 지키므로, 이 abstraction은 Finance
전용이 아니다.

## 15. Streaming / async

- 사용자 대면 token streaming: SSE (`Shared AI → Product → Gateway → Client`). Streaming 중에도
  typed/semantic validation과 side-effect gate를 우회하지 않는다.
- 장시간 job / usage·cost 기록 / batch ingestion: NATS JetStream 후보 (deferred, 첫 use case 시).
  도입 시 at-least-once + idempotent consumer + DB commit 후 ACK (ADR-0015 §7.6).
- async/retry/resume에서 original context는 correlation/audit용이고, protected action은
  query-time current authorization을 다시 확인한다([ADR-0004](adr/ADR-0004-execution-context-api-tenant-isolation.md)).

## 16. Anti-pattern and overengineering guard

```text
- 모든 질문 → RAG
- 모든 요청 → Agent
- 모든 데이터 → embedding
- endpoint마다 별도 pipeline 하드코딩
- Product별 if/else를 Core에 누적
- Vector RAG를 정확한 사실값(금액/날짜/score)의 SSOT로 사용
- LLM 결과 → validation 없이 Product fact/mutation으로 승격
- Tenant scope 없는 retrieval
- Prompt = Skill, Tool = Skill (개념 혼합; skill-lens-tool-retriever-model 참조)
- LangGraph clone 또는 custom generic Agent Framework
- universal Workflow DSL / universal Product schema
- giant distributed state machine
- custom generic permission language 또는 generic policy engine
- universal automatic claim fact-checker
- premature multi-agent runtime
```

Framework 도입 여부와 무관하게 Product business policy, Skill/Lens meaning, retrieval meaning,
required Evidence semantics와 final mutation authority를 Shared AI로 이동하지 않는다.

## 17. Explicit non-goals

```text
- 실제 runtime code/API schema/production wire contract 구현
- exact GPT model/version 또는 model alias mapping 확정
- embeddings provider/model, reranker와 chunk/ranking policy 확정
- Pydantic AI/Python worker/service 선택 또는 구현
- 자유 LLM router, generic Agent Framework, LangGraph/RAGFlow adoption
- Evidence의 domain semantics를 universalize하거나 truth/fact-check engine화
- Finance runtime contract를 Shared AI로 대체
- Carelog business rule 또는 Finance policy/lens semantics의 Shared AI 이전
```

## 18. Open decisions

| # | Open decision | 분류 |
| --- | --- | --- |
| OD-4 | Finance 기존 runtime pipeline과 Shared AI 실행 골격의 관계 | **RESOLVED — [ADR-0001](adr/ADR-0001-skill-lens-workflow-execution-ownership.md)** (병존 + opt-in; Shared AI는 선언된 순서·gate를 재정의 없이 집행) |
| OD-8 | Evidence core envelope vs domain extension 경계와 schema | **RESOLVED — [ADR-0002](adr/ADR-0002-evidence-provenance-core-domain-boundary.md)** (Core envelope·correlation=Shared AI, admissibility·semantics·validation=Product/Domain) |
| OD-9 | Workflow orchestration 구현 방식(코드 vs DSL vs LangGraph) | deferred — bounded typed handler를 Phase 1 default로 사용; agent/graph adoption은 §10 evidence trigger 이후 |
| ADR-0007 follow-up | exact GPT model/version, embeddings/reranker, provider fallback | deferred — 이 문서는 current OpenAI API / GPT family target만 사용 |

## 19. Related canonical sources

- `harness-foundation-docs/docs/adr/ADR-0015-...` §7.4~7.6
- [ADR-0001](adr/ADR-0001-skill-lens-workflow-execution-ownership.md) — definition/selection/validation vs execution mechanism
- [ADR-0002](adr/ADR-0002-evidence-provenance-core-domain-boundary.md) — Evidence core/domain boundary
- [ADR-0004](adr/ADR-0004-execution-context-api-tenant-isolation.md) — context, authority와 isolation
- [ADR-0006](adr/ADR-0006-runtime-technology-framework-dependency-boundary.md) — Kotlin/JVM, Spring AI adapter와 future extraction boundary
- [ADR-0007](adr/ADR-0007-current-openai-gpt-provider-target.md) — current OpenAI/GPT target과 provider-neutral contract
- `finance-harness-docs/00-core/0-c-runtime-contract/finance-routing-contract.md`
- `finance-harness-docs/00-core/0-a-safety-core/finance-policyguard-rules.md`
- [product-ai-responsibility](product-ai-responsibility.md)
- [source-of-truth-map](source-of-truth-map.md)
