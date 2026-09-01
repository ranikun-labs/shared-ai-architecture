# Skill / Lens / Tool / Retriever / Model

| 항목 | 값 |
| --- | --- |
| status | draft (RPL-97 Foundation) |
| owner | shared-ai-architecture |
| 상위 canonical | `ADR-0015` §7.4, [ADR-0001](adr/ADR-0001-skill-lens-workflow-execution-ownership.md), [ADR-0007](adr/ADR-0007-current-openai-gpt-provider-target.md), `finance-lens-authoring-standard.md`, `finance-knowledge-storage-decision.md` |

---

## 1. Purpose

Shared AI 실행에 등장하는 capability를 **하나로 합치지 않고** 정의한다. 각 개념의 working
definition, 소유 주체, non-goal을 고정해 "Prompt=Skill", "Tool=Skill", "모든 데이터=embedding"
같은 개념 붕괴를 막는다.

## 2. Capability 정의와 소유

| Capability | Working definition | 소유(정의/내용) | 소유(실행 계약) |
| --- | --- | --- | --- |
| **LLM Invocation** | 단순 모델 호출 | Shared AI | Shared AI |
| **Structured Data** | 정확한 Product 사실값 (미납·예약·통증 score·journal metadata·market data) | Product Backend / DB | Shared AI는 Tool로 소비 |
| **Tool** | 결정론적 capability (Product API, DB-backed query, 계산, aggregation, external market data) | Product (tool 의미·권한) | Shared AI (호출 규약) |
| **Retriever / RAG** | 비정형 knowledge/history 검색 (상담 메모, 치료 기록, 계약서, 금융 문서, 과거 narrative) | Product (semantic SoT·admission·corpus·retrieval policy) | Shared AI (tenant-scoped projection lifecycle·retrieval execution) |
| **Skill** | 업무 단위 Application Use Case (= 무엇을 수행) | Product (정의) | Shared AI (실행 계약) |
| **Lens** | 동일 context를 보는 관점 (= 어떤 관점으로 분석) | Domain (Finance=finance-harness-docs) | Shared AI (실행 계약) |
| **Workflow** | 사전 정의된 실행 조합 | Product | Shared AI |
| **Agent** | 실행 중 next action 동적 판단 | Product | Shared AI (예외적) |
| **Policy / Guardrail** | 허용/금지 범위 | Product/Domain (규칙 원문) | Shared AI (gate 호출) |
| **Evidence / Provenance** | 결과의 근거 추적 | core=Shared AI / 확장=Domain | Shared AI |

## 3. Skill vs Lens (핵심 구분)

```text
Skill = 무엇을 수행할 것인가   (Treatment Preparation, Rental Payment Review, Investment Review)
Lens  = 어떤 관점으로 분석할 것인가 (Therapy Progress, Risk, Earnings Revision, Semiconductor Cycle)
```

- 하나의 Skill이 여러 Lens를 조합할 수 있다.
- Finance의 lens authoring standard가 이미 못박은 원칙과 정렬:
  **"렌즈는 무엇을 보는가만 정의한다. 그래서 무엇을 해야 하는가는 정의하지 않는다."**
- 따라서 Lens의 판단 내용/문체/금지표현은 Domain(Finance) 소유이고, Shared AI가 소유할 수 있는
  것은 최대한 **Lens를 runtime-loadable artifact로 실행하는 계약**뿐이다(candidate, OD-1).

## 4. Skill = Application Service (Product 소유)

`ADR-0015` §7.4는 Product Workflow·System Prompt·Domain Policy를 Product에 귀속시켰다.
따라서 Skill의 **정의**(어떤 Tool/Retriever/Lens/Prompt를 어떤 순서로 쓰는가, 도메인 결과를
어떻게 검증하는가)는 **Product 소유**다. Shared AI는 Skill을 *실행할 수 있는 계약과 substrate*를
제공한다.

> 이 지점이 RPL-97 working hypothesis와 기존 canonical의 긴장점이다. 가설은 "Skill이 Shared AI
> Execution Core 안에 산다"였으나, ADR-0015는 Workflow/Prompt/Policy를 Product에 귀속시킨다.
> 결론: **Skill 정의 = Product, Skill 실행 계약 = Shared AI** 로 분리한다.
> 실행 mechanism 소유는 [ADR-0001](adr/ADR-0001-skill-lens-workflow-execution-ownership.md)에서
> 확정됐고, 정의·선택·domain validation semantics는 계속 Product/Domain에 남는다.

## 5. Retriever / RAG vs Knowledge Artifact (혼동 금지)

두 가지를 반드시 구분한다.

```text
(A) 사용자 대면 RAG corpus
    = 상담 메모 / 치료 기록 / 계약서 / journal / 과거 narrative
    = per-tenant 비정형 user data
    → Retriever 대상. tenant-scoped. corpus·policy는 Product 소유.
      projection lifecycle·retrieval execution은 Shared AI 소유([ADR-0003](adr/ADR-0003-retrieval-knowledge-runtime-ownership-lifecycle.md)).
      물리 저장·index technology는 RPL-103으로 deferred.

(B) Finance Knowledge Artifact
    = Lens / PolicyGuard / Routing / Validator 문서 그 자체
    = 저작·검수·안전 임계 자산
    → RAG corpus 아님. finance-harness-docs 결정상 Markdown authoring SoT + WAS private
      resource이며 MVP에서 DB화하지 않는다. Shared AI가 이를 흡수하거나 embedding하지 않는다.
```

Vector RAG(A)를 정확한 사실값의 SSOT로 쓰지 않는다. 사실값은 Tool(Product SSOT)로 얻는다.

## 6. Tool 원칙

- Tool은 결정론적이며 Product Tool/API adapter를 통해 Product Backend·DB에 도달한다
  (Shared AI는 Product DB를 직접 알지 않는다).
- Tool 의미와 사용 권한은 Product 소유(ADR-0015 §7.4). Shared AI는 호출 규약·timeout·retry·trace만.

## 7. Model Gateway

- 소유: Shared AI (provider adapter, key/secret, model alias, timeout/retry/fallback,
  rate limit/concurrency, usage/cost, observability, provider 장애 처리). ADR-0015 §7.4 정렬.
- Prompt 내용은 Product 소유. Model Gateway는 prompt/model **version 인지**만 한다.
- 현재 첫 model-inference target은 [ADR-0007](adr/ADR-0007-current-openai-gpt-provider-target.md)의
  OpenAI API / GPT family다. Spring AI는 adapter/integration library이며, exact GPT model/version,
  embeddings와 reranker는 deferred다.
- LLM output은 structured shape를 가져도 untrusted output이다. schema/mechanical validation 뒤에
  Product/Domain semantic validation을 거쳐야 Product result가 된다([execution-model](execution-model.md)).

## 8. Explicit non-goals

```text
- Skill Marketplace / Dynamic Plugin Loader / Workflow DSL 정의
- 자유 LLM routing 또는 generic Agent Framework를 Phase 1 기본값으로 도입
- Pydantic AI/Python type을 neutral contract에 노출
- Lens 실행 계약의 wire schema 확정
- exact embedding/reranker/chunk/ranking policy 및 dedicated vector DB 선택
- Finance Lens semantics를 이 repo로 복사
- Universal Tool/Adapter interface를 모든 Product에 강제 (foundation의 "Global Enum 강제 금지" 정렬)
```

## 9. Open decisions

| # | Open decision | 분류 |
| --- | --- | --- |
| OD-1 | Skill/Lens 실행 계약을 Shared AI가 소유 vs Model Gateway까지만 소유 | **RESOLVED — [ADR-0001](adr/ADR-0001-skill-lens-workflow-execution-ownership.md)** (실행 mechanism = Shared AI; 정의·선택·검증 = Product/Domain) |
| OD-6 | RAG corpus(A) source와 runtime projection의 소유·lifecycle·tenant 격리 경계 | **RESOLVED — [ADR-0003](adr/ADR-0003-retrieval-knowledge-runtime-ownership-lifecycle.md)** (Source SoT=Product, projection/execution=Shared AI, technology=RPL-103 deferred) |
| OD-10 | Tool 선언(capability declaration) 형식이 provider/product-neutral하게 필요한가 | deferred |

## 10. Related canonical sources

- `harness-foundation-docs/docs/adr/ADR-0015-...` §7.4
- `harness-foundation-docs/docs/architecture/shared-core-and-extensions.md` §4.10~4.11, §19
- `finance-harness-docs/00-core/0-b-lens-infra/finance-lens-authoring-standard.md`
- `finance-harness-docs/00-core/0-d-knowledge-architecture/finance-knowledge-storage-decision.md`
