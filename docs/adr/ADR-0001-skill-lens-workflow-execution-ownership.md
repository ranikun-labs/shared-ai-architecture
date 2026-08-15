# ADR-0001 — Skill/Lens/Workflow Execution Ownership

| 항목 | 값 |
| --- | --- |
| status | Accepted |
| jira | RPL-99 (parent RPL-98) |
| predecessor | RPL-97 Foundation (canonical main `53fffee`) |
| ADR namespace | shared-ai-architecture 로컬 (`ADR-000N`). foundation의 전역 `ADR-00NN`와 별개다 |
| 상위 canonical | `harness-foundation-docs` `ADR-0015` §7.4, `finance-harness-docs` routing-contract / lens-authoring-standard / knowledge-storage-decision, `carelog-architecture` ownership-map |
| 해소 대상 | OD-1 (Skill/Lens 실행 계약 소유), OD-4 (Finance runtime contract 관계) |

---

## 1. Status

**Accepted.**

이 ADR은 RPL-97에서 `NEEDS_ADR`로 남긴 OD-1·OD-4를 확정한다. 4개 canonical repository
(`harness-foundation-docs`, `finance-harness-docs`, `carelog-architecture`, 본 repo)의 fresh
`origin/main`을 교차 확인한 결과 **blocking conflict가 없고, owner repository 규칙 변경이 선행될
필요가 없다.** 따라서 Proposed가 아니라 Accepted로 확정한다. (교차 확인 SHA는 §11에 고정한다.)

## 2. Context

RPL-97 Foundation은 Shared AI를 "AI를 어떻게 실행하는가", Product를 "AI가 이 제품에서 무엇을
해야 하는가"로 나눴다(`ADR-0015` §7.4 정렬). 그러나 두 경계 결정을 열어 두었다.

- **OD-1**: Shared AI가 Skill/Lens **실행 계약**을 소유하는가, 아니면 Model Gateway까지만
  소유하고 Skill/Lens 실행은 각 Product runtime에 두는가.
- **OD-4**: Finance가 이미 소유한 자체 runtime contract(`finance-routing-contract.md`)와 Shared AI
  실행 골격의 관계(채택/병존/충돌).

핵심 질문은 하나다.

> **Shared AI가 sequence를 실행한다는 이유만으로 Product Workflow semantics를 소유하게 되는가?**

canonical source의 답은 **아니오**다. `ADR-0015` §7.4는 Product Workflow·System Prompt·Domain
Policy·Domain Context 선택·Tool 의미/권한·Domain 결과 검증·최종 책임을 모두 Product에 귀속시키고,
"공통 AI Orchestration과 공통 AI Job Runtime"은 **제품 중립 기술 Capability 후보**로만 인정하며
"실제 Use Case가 생긴 뒤 후속 Decision으로" 소유를 정하라고 명시했다. 본 ADR이 그 후속 Decision이다.

## 3. Decision (요지)

1. **OD-1 확정** — Shared AI는 Skill/Lens/Workflow의 **generic execution mechanism(실행 계약)을
   소유한다.** 이는 "Model Gateway만" 소유보다는 넓지만, 엄격히 **실행 mechanism에 한정**된다:
   capability invocation, sequence 실행, gate 호출, timeout/retry, model 호출, trace, evidence
   correlation, streaming. Shared AI는 Skill/Lens/Workflow의 **정의(Definition)·선택(Selection)·
   도메인 검증(Validation) semantics를 소유하지 않는다.**

2. **OD-4 확정** — **병존 + opt-in 채택.** Finance의 `finance-routing-contract.md`는 workflow·
   policy 정의와 선택 권한(routing)의 canonical owner로 **그대로 유지**된다. Shared AI는 Product가
   선언한 순서 있는 pipeline(pre-model policy gate + post-model output-validator gate 포함)을
   실행할 수 있는 generic engine을 제공한다. Finance는 pipeline semantics를 넘기지 않고 Shared AI를
   실행 substrate로 **선택적으로** 채택하거나, 자체 runtime을 유지할 수 있다. 강제 migration은 없다.

3. Shared AI 실행 mechanism은 **Product가 선언한 필수 실행 순서**(예: Input Guard가 lens보다
   먼저, Output Validator 통과 전 노출 불가)를 mechanism 차원에서 보장해야 한다. 이 순서의 **내용과
   규칙**은 Domain 소유이고, Shared AI는 그 순서를 **재정의하지 않고 집행**한다.

## 4. Ownership Matrix (4축)

각 capability를 Definition / Selection / Execution / Validation 네 축으로 분리한다.

| Capability | Definition Owner | Selection Owner | Execution Owner | Validation Owner |
| --- | --- | --- | --- | --- |
| **Skill** | Product/Domain | Product/Domain | **Shared AI** (mechanism) | Product/Domain (+Shared AI 기술 seam) |
| **Lens** | Domain (Finance=`finance-harness-docs`) | Product/Domain (Finance=routing-contract) | **Shared AI** (LensExecutor: runtime-loadable artifact 실행) | Domain (Output Validator 규칙) — Shared AI가 gate로 호출 |
| **Workflow** | Product | Product | **Shared AI** (sequenced execution engine) | Product (도메인) + Shared AI (기술) |
| **Tool** | Product (의미·권한) | Product (workflow가 step에서 선언) | **Shared AI** (호출 규약·timeout·retry·trace) → Product Tool adapter | Product (사실값 = Product SSOT) |
| **Retriever** | Product (corpus·retrieval policy) | Product | **Shared AI** (tenant-scoped substrate; 물리 저장은 OD-6 deferred) | Product |
| **Policy/Guardrail** | Product/Domain (규칙 원문) | Product/Domain (routing이 guard action 결정) | **Shared AI** (선언된 순서의 gate로 호출) | Domain (policy 자체가 검증 규칙); Shared AI는 gate 위치를 집행 |

핵심 3분할(모든 축의 상위 원칙):

```text
Semantic Source of Truth   = Product/Domain 소유 (정의·선택 규칙·검증 규칙)
Runtime Projection         = Shared AI가 읽는 실행 투영 (registry/descriptor 투영)
Execution Mechanism        = Shared AI 소유 (호출·순서·gate·timeout·trace·model·evidence)
```

## 5. Execution Boundary (무엇을 소유하고 무엇을 안 하는가)

**Shared AI가 소유(실행 mechanism):**

```text
- declared capability(Tool/Retriever/Lens/Skill) invocation
- Product가 선언한 sequence/graph의 실행 (순서·분기 집행, 재정의 아님)
- Product가 선언한 gate 실행: pre-model policy gate, post-model output-validator gate
- model 호출 (Model Gateway)
- timeout / 제한 retry / fallback / rate limit / concurrency
- technical error normalization
- trace / usage / cost / observability
- evidence correlation (core envelope; core/domain 경계는 OD-8 deferred)
- streaming (SSE transport 구현)
```

**Shared AI가 소유하지 않음:**

```text
- Skill/Lens/Workflow 의 업무 의미 정의
- 어떤 Skill/Lens/Tool/Retriever 를 쓸지에 대한 선택 semantics
- 임의(arbitrary) Skill/Lens 자동 선택
- Domain business rule / Domain policy 규칙 원문
- Domain 결과의 최종 검증과 책임
- Product workflow 의 순서·분기 규칙 자체 (집행만; 정의는 Product)
```

## 6. Routing Model (Phase 1 권장)

네 후보 중 하나로 좁힌다.

| 옵션 | 내용 | Phase 1 판정 |
| --- | --- | --- |
| A. Product Explicit Routing | Product가 `skill_id`/`lens_id`/`workflow_id`를 명시 | **채택(기본값)** — deterministic·auditable·testable |
| B. Shared AI Deterministic Router | Shared AI가 metadata로 선택 | **소유로는 기각** — Shared AI가 selection semantics를 소유하지 않는다. Product가 선언·위임한 bounded step을 *실행*하는 것은 허용 |
| C. LLM Router | LLM이 Skill/Lens 자유 선택 | **기각(Phase 1)** — 고위험 domain(Finance/Carelog)에서 위험. 임의 선택 금지 |
| D. Hybrid (bounded) | Product가 Skill 선택 → Skill 계약이 허용한 lens 후보 집합 내에서 bounded 선택 | **허용(제한적)** — 선택은 Product/Domain이 선언한 allowed set 안에서만 |

**Phase 1 규칙**: 기본은 A(명시적 라우팅). D는 Product/Domain이 선언한 allowed set 안에서만 허용.
**Selection ownership은 라우터를 쓰더라도 Product/Domain에 남는다** — Shared AI는 선언·위임된 선택을
*실행*할 뿐 임의 선택을 소유하지 않는다. C(자유 LLM 라우팅)는 Phase 1에서 금지.

> Finance의 intent 분류 + guard action 결정은 Finance가 소유하는 deterministic selection step이다.
> 이 step이 물리적으로 Shared AI engine 안에서 실행되더라도, **규칙과 순서 불변식은 Finance 소유**이며
> Shared AI는 선택 결과를 바꿀 수 없다.

## 7. Workflow vs Agent

canonical 구분으로 고정한다.

```text
Workflow : known execution graph + known allowed capabilities + bounded branching
Agent    : runtime에서 다음 action을 동적으로 선택
```

- Phase 1 기본값은 **Workflow**. Agent는 동적 next-action 판단이 실제로 필요한 use case에만.
- RPL-97 consumer 3종(Therapy/Rental/Finance)은 전부 Workflow로 충분하다 → Phase 1에 Agent runtime 불필요.
- **본 ADR은 LangGraph 등 orchestration 구현 technology를 결정하지 않는다**(별도 implementation ADR, OD-9).

## 8. Versioning / Registry Principle

schema를 구현하지 않고 **compatibility principle**만 확정한다. 세 version 축은 독립이다.

```text
Semantic owner version      : Skill/Lens/Workflow/Policy 정의 버전 — Product/Domain 소유
Prompt/template artifact ver: prompt 산출물 버전 — Product/Domain 소유
Runtime executor version    : 실행 mechanism 버전 — Shared AI 소유
```

호환 규칙:

1. Runtime executor 변경은 semantic 결과를 바꾸면 안 된다(순수 실행 mechanism).
2. Semantic version 변경은 contract 변경이며 반드시 선언되어야 한다.
3. Prompt 변경이 자동으로 semantic contract 변경은 아니다 — 선언된 behavior를 바꿀 때만 semantic
   version bump이며, 그 판단은 Product/Domain이 한다.
4. Shared AI는 version을 **인지(aware)**할 뿐 semantic version을 **소유하지 않는다**.

**Registry 원칙**: `Runtime Registry = Runtime Projection ≠ Semantic Source of Truth`.
Finance canonical(`knowledge-storage-decision`)이 이미 못박았다 — Markdown = Authoring SoT,
향후 PostgreSQL은 "Runtime Projection 또는 Control Plane"일 뿐 SoT가 아니며, gate pass/fail·
routing_enabled·실행 시각 같은 runtime 필드는 lens metadata(정의)와 분리된다. Shared AI가 registry를
갖더라도 그것은 Product-owned descriptor의 runtime projection이다.

**Domain Pack** (logical packaging concept로만 인정): Product가 소유하는 canonical bundle
(Skill 정의 + Lens 정의 + Policy + allowed Tool/Retriever + workflow descriptor). Shared AI는 이를
runtime projection으로 읽는다. **Dynamic Plugin Loader는 구현하지 않는다** — 논리적 packaging까지만.

## 9. Consumer Validation

RPL-97 consumer 3종을 본 ADR ownership으로 재검증한다.

### A. Carelog Therapy — PASS
`TreatmentPreparation` 정의·선택=Carelog, 실행=Shared AI(mechanism), 검증=Carelog Therapy Safety
Policy(gate). 사실값은 Appointment/Measurement Tool(Product SSOT), history는 Retriever. Shared AI는
진단/처방을 선택하지 않는다(임의 선택 금지). **PASS.**

### B. Carelog Rental — PASS
`RentalPaymentReview`. 미납 금액은 Payment Tool(Product SSOT)에서만, **RAG가 금액을 결정하지 않는다.**
정의·선택=Carelog, 실행=Shared AI. **PASS.**

### C. Finance Investment Review — PASS
`InvestmentReview`. Finance routing-contract가 selection·순서·policy·validator의 owner로 유지되고,
Shared AI는 선언된 순서(Input Guard → intent → guard action → lens/redirect → template → Output
Validator)를 **재정의 없이 집행**한다. Lens semantics·PolicyGuard 규칙은 finance-harness-docs 소유.
**PASS** — OD-4 병존 모델이 Finance canonical을 침범하지 않음을 확인.

Ownership 회귀: Finance Lens semantics 미복사 ✔ / Carelog business rule 미이동 ✔ / Platform 통신·
배포 규칙 미중복 ✔ / 임의 LLM·Agent 강제 없음 ✔.

## 10. Consequences

**긍정**

- 여러 Product가 하나의 안정적 execution runtime을 재사용하되 업무 의미는 각자 소유한다.
- Finance는 검증된 routing contract를 잃지 않고 실행 substrate만 선택적으로 공유할 수 있다.
- Selection/Definition/Validation이 Product/Domain에 고정되어 고위험 domain의 안전 불변식이 보존된다.

**비용 / 트레이드오프**

- Shared AI 실행 mechanism은 **Product 선언 순서 있는 gate**(pre-model, post-model)를 1급 기능으로
  지원해야 한다. 이 제약을 못 지키면 Finance를 host할 수 없다 — 실행 계약 설계의 필수 요구사항.
- Product는 Skill/Lens/Workflow를 **선언적 execution request**로 표현해야 하므로 초기 declarative
  descriptor 설계 부담이 생긴다(coupling은 낮추지만 선언 비용은 존재).
- Registry가 projection이라는 원칙은 SoT drift 방지 규율을 요구한다(descriptor 변경 → projection 재생성).

## 11. Canonical Source Reality (교차 확인 anchor)

| Repository | Canonical branch | fresh `origin/main` SHA |
| --- | --- | --- |
| `shared-ai-architecture` | `main` | `53fffee740ae642f658bcea065fc53ce86e74055` |
| `harness-foundation-docs` | `main` | `69fceb267094756913f400f8b4045590bd0e8ac5` |
| `finance-harness-docs` | `main` | `5326977691b420f573a469c0bb7635de26fe2aba` |
| `carelog-architecture` | `main` | `6b581dd3cc92d5e8486172525898d674e8d61b7c` |

> SHA는 재현 anchor이며 영구 invariant가 아니다. 각 repo의 canonical은 `main`이고, feature branch는
> canonical source가 아니다.

## 12. Rejected Alternatives

- **Shared AI가 Skill/Lens/Workflow 의 정의·선택을 소유** — `ADR-0015` §7.4(Workflow=Product)와
  직접 충돌. 기각.
- **Model Gateway만 소유(실행 계약 전부 Product runtime)** — 재사용 가능한 공통 실행 substrate라는
  Shared AI 존재 이유를 약화. sequence·gate·evidence를 각 Product가 재구현하게 됨. 기각.
- **자유 LLM 라우팅(C)** — 고위험 domain 안전 불변식 위반 위험. Phase 1 기각.
- **Finance runtime contract를 Shared AI로 흡수/대체** — Finance canonical 침범, owner repo 변경
  선행 필요. 병존/opt-in으로 대체.

## 13. Non-goals

```text
- OD-6 Retrieval/RAG 물리 저장 결정
- OD-8 Evidence core/domain schema 확정
- OD-3 Shared AI physical runtime 위치·시점
- FastAPI/Python scaffold, pgvector schema, embedding/chunk/reranker 선정
- LangGraph/RAGFlow/LightRAG/GraphRAG/NATS adoption (implementation technology)
- Agent runtime, Dynamic Plugin Loader, Workflow DSL, Skill Marketplace
- Finance Lens semantics / Carelog domain rule 변경
- Product backend 코드 변경, Jira 후속 ticket 생성
```

## 14. Follow-up Decisions

```text
OD-2/OD-6 : Retriever/RAG·Embedding·Vector infra 소유·범위·저장
OD-8      : Evidence core envelope vs domain extension schema
OD-3      : Shared AI physical runtime 위치·도입 시점
OD-5      : Shared AI ↔ Product Tool service-to-service 인증
OD-7      : Provider/Identity 비가용 시 fail-closed/fail-open
OD-9      : Workflow orchestration 구현 방식(코드 vs DSL vs LangGraph)
OD-10     : Tool capability declaration 형식의 product/provider-neutral 필요성
```

## 15. Related canonical sources

- `harness-foundation-docs/docs/adr/ADR-0015-platform-communication-messaging-scaling.md` §7.4
- `finance-harness-docs/00-core/0-c-runtime-contract/finance-routing-contract.md`
- `finance-harness-docs/00-core/0-b-lens-infra/finance-lens-authoring-standard.md`, `finance-lens-catalog.md`
- `finance-harness-docs/00-core/0-d-knowledge-architecture/finance-knowledge-storage-decision.md`
- `carelog-architecture/architecture/ownership-map.md`, `contracts/authenticated-carelog-context.md`
- 본 repo: [shared-ai-charter](../shared-ai-charter.md), [skill-lens-tool-retriever-model](../skill-lens-tool-retriever-model.md), [execution-model](../execution-model.md), [source-of-truth-map](../source-of-truth-map.md)
</content>
</invoke>
