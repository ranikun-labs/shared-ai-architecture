# Consumer Use-Case Matrix

| 항목 | 값 |
| --- | --- |
| status | draft (RPL-97 Foundation) |
| owner | shared-ai-architecture |
| purpose | 3개 실존/근접 consumer 요청을 같은 Core abstraction으로 설명 가능한지 검증 |

---

## 1. Purpose

Charter/실행 모델이 실제로 성립하는지, 세 요청을 **동일한 Core abstraction**으로 end-to-end
trace해서 검증한다. 각 요청은 다음 골격을 통과해야 한다.

```text
Request → Product Backend → Shared AI → Skill → Tool/Retriever/Lens/Policy
        → Model → Evidence → Response
```

세 use case 모두 PASS해야 이 Foundation이 성립한다.

## 2. Carelog Therapy

> "내일 오는 김OO 고객 상태, 이전 치료 반응과 내일 확인할 내용을 정리해줘."

| 단계 | 소유 | 내용 |
| --- | --- | --- |
| Product Backend | Carelog | 요청 수신, workspace/customer authz, scope 확정 |
| Skill | Carelog(정의) | `TreatmentPreparation` |
| Tool | Carelog API | Appointment(내일 예약), Measurement/Trend(통증 score 등) — 정확한 사실값 |
| Retriever | tenant-scoped | Treatment History(상담·치료 narrative) — 비정형 |
| Lens | Carelog domain | Therapy Progress |
| Policy | Carelog domain | Therapy Safety Policy (진단/처방 대체 금지) |
| Model | Shared AI | 요약 생성 |
| Evidence | Shared AI core + Carelog 확장 | 사용한 appointment/measurement tool, 검색된 기록 ref, lens/version |
| Response | Carelog | 검증 후 사용자에게 정리 제공 |

Composition:

```text
TreatmentPreparation Skill
  + Appointment Tool + Measurement/Trend Tool
  + Treatment History Retriever
  + Therapy Progress Lens
  + Therapy Safety Policy
  + Evidence
```

- 경계 준수: Shared AI는 Carelog DB를 직접 알지 않고 Tool로만 사실값 취득. 의료진을 대체하는
  진단/처방 Agent가 되지 않는다(Workflow, Policy로 차단). tenant scope=workspace+customer 전파.

**판정: PASS**

## 3. Carelog Rental

> "이번 달 월세 미납자와 이전 연락 내용을 정리해줘."

| 단계 | 소유 | 내용 |
| --- | --- | --- |
| Product Backend | Carelog | authz, scope(workspace) 확정 |
| Skill | Carelog(정의) | `RentalPaymentReview` |
| Tool | Carelog API | Payment(정확한 미납 상태/금액) — **사실값은 Tool, RAG 아님** |
| Tool/Context | Carelog | Contract Context (계약 정보) |
| Retriever | tenant-scoped | Conversation/연락 기록 — 비정형 |
| Model | Shared AI | 정리 생성 |
| Evidence | Shared AI core | payment tool 결과 ref, 검색된 대화 ref |
| Response | Carelog | 검증 후 제공 |

Composition:

```text
RentalPaymentReview Skill
  + Payment Tool + Contract Context
  + Conversation Retriever
  + Evidence
```

- 경계 준수: **Vector RAG가 정확한 금액의 SSOT가 되지 않는다** — 미납 상태/금액은 Payment Tool
  (Product SSOT)에서만. 이전 연락은 Retriever. anti-pattern("모든 데이터 embedding") 회피.

**판정: PASS**

## 4. Finance Investment Review

> "내 투자 판단에서 놓친 리스크를 복기해줘."

| 단계 | 소유 | 내용 |
| --- | --- | --- |
| Product Backend | Finance Harness | authz, scope(user/journal) 확정 |
| Skill | Finance(정의) | `InvestmentReview` |
| Context/Tool | Finance | Journal Context, Point-in-time Evidence(당시 판단 시점), Market Data Tool |
| Retriever | tenant-scoped | 과거 journal/narrative |
| Lens | finance-harness-docs | Risk Lens, Journal-Review Lens (semantics는 Finance 소유) |
| Policy | finance-harness-docs | PolicyGuard (매수/매도·목표가·손절가 미제공, 과거 평가 금지) |
| Model | Shared AI | 복기 질문/체크리스트 생성 |
| Evidence | Shared AI core + Finance 확장 | lens version, context snapshot, market data timestamp |
| Response | Finance | Output Validator 통과 후 제공 |

Composition:

```text
InvestmentReview Skill
  + Journal Context + Point-in-time Evidence + Market Data Tool
  + Risk Lens + Journal-Review Lens
  + Finance PolicyGuard
  + Evidence
```

- 경계 준수: **금융 Lens 판단 내용은 finance-harness-docs 소유.** Shared AI는 실행 계약/Model
  Gateway/trace만(OD-1/OD-4, NEEDS_ADR). Finance의 canonical pipeline(Input Guard → intent →
  guard action → lens/redirect → template → Output Validator)을 Shared AI가 재정의하지 않는다.
  point-in-time grounding으로 사후 결과와 당시 판단을 분리.

**판정: PASS (경계 보존 기준). Lens 실행 계약 소유는 OD-1/OD-4로 이월되며 이 use case의 경계 유효성에는 영향 없음.**

## 5. Cross-use-case 검증 요약

| 검증 항목 | Therapy | Rental | Finance |
| --- | --- | --- | --- |
| 동일 Core 골격으로 trace 가능 | PASS | PASS | PASS |
| Shared AI가 Product DB 직접 접근 안 함 | PASS | PASS | PASS |
| 사실값=Tool, history=Retriever 분리 | PASS | PASS | PASS |
| tenant scope 전파 | PASS | PASS | PASS |
| Domain Lens/Policy semantics=Product/Domain | PASS | PASS | PASS |
| LLM 결과 검증 후 반영(자동 mutation 없음) | PASS | PASS | PASS |
| Agent 강제 안 함(Workflow로 충분) | PASS | PASS | PASS |

Ownership 회귀 검증:

```text
- Finance Lens semantics가 Shared AI로 복사되지 않음         ✔
- Carelog business rule이 Shared AI로 이동하지 않음          ✔
- Platform 통신/배포 규칙이 중복 정의되지 않음               ✔
- Shared AI 전용 execution abstraction이 명확함(Model Gateway·Tool/Retriever 실행 계약·Evidence) ✔
```

## 6. Phase 1 필수 vs 후순위 abstraction

| Abstraction | Phase 1 필수 | 후순위 |
| --- | --- | --- |
| Model Gateway (provider/key/alias/timeout/usage/trace) | ✔ | |
| Tool orchestration 계약 (Product API adapter) | ✔ | |
| Tenant scope 전파 | ✔ | |
| Execution Context 계약 + tenant/resource isolation (identity≠authz, query-time authority) | ✔ — [ADR-0004](adr/ADR-0004-execution-context-api-tenant-isolation.md) 확정 | |
| Evidence core envelope | ✔ | |
| Retriever 실행 계약 + tenant-scoped projection lifecycle | ✔ (use case 필요) — ownership은 [ADR-0003](adr/ADR-0003-retrieval-knowledge-runtime-ownership-lifecycle.md) 확정 | |
| Skill 실행 계약 (interface) | ✔ (정의는 Product) | |
| Lens 실행 계약 | | ▶ Finance runtime contract 정합 후 (NEEDS_ADR) |
| Agent runtime | | ▶ 동적 판단 use case 등장 시 |
| Vector DB / embedding / reranker 확정 | | ▶ RPL-103 Runtime Technology ADR |
| Workflow DSL / Skill Marketplace / Plugin Loader | | ▶ 훨씬 후순위 |

## 7. Related canonical sources

- `carelog-architecture/contracts/customer-event-lifecycle.md`, `contracts/authenticated-carelog-context.md`
- `finance-harness-docs/00-core/0-c-runtime-contract/finance-routing-contract.md`
- `finance-harness-docs/01-core-analysis-lenses/finance-lens-risk-checklist.md`, `finance-lens-journal-review.md`
- [execution-model](execution-model.md), [product-ai-responsibility](product-ai-responsibility.md)
