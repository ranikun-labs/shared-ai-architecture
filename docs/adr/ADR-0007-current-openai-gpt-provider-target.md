# ADR-0007 — Current OpenAI/GPT Implementation Target and Provider-Neutral Boundary

| 항목 | 값 |
| --- | --- |
| status | Accepted |
| predecessor | [ADR-0006](ADR-0006-runtime-technology-framework-dependency-boundary.md) |
| ADR namespace | shared-ai-architecture 로컬 (`ADR-000N`) |
| decision posture | Current implementation target; permanent/exclusive provider 선택 아님 |
| scope | 첫 production Shared AI model-inference slice의 현재 provider target과 neutral boundary |
| 상위 canonical | `harness-foundation-docs` ADR-0015 §7.4, §7.5, §10 |

---

## 1. Status

**Accepted.**

`SHARED_AI_OPENAI_TARGET_RATIFIED`로 이미 ratified된 architecture decision을 이 ADR에
durable하게 기록한다. Provider selection을 다시 열거나 영구 provider를 지정하는 ADR이 아니다.

[ADR-0006](ADR-0006-runtime-technology-framework-dependency-boundary.md)은 provider, model,
embedding model과 reranker를 의도적으로 deferred로 남겼다. 이 ADR은 그 historical Accepted
scope를 변경하지 않고, 현재 **model-inference provider target**만 후속 결정한다.

## 2. Context

현재 하나의 구체적인 model provider target이 필요한 이유는 다음과 같다.

- RPL-103은 Shared AI의 Phase 1 runtime placement와 runtime/framework boundary를 고정했다.
- RPL-104는 `ExecutionContext → Skill → Tool + Retriever → Model → Evidence + Result`의
  provider-neutral vertical-slice feasibility evidence를 제공했다.
- RPL-106은 `platform-services`에 canonical Shared AI neutral foundation을 만들었다.
- 실제 model inference 개발·평가·smoke baseline을 시작하려면 한 개의 concrete provider target이
  필요하다.
- 경쟁하는 Bedrock/OpenAI experimental 방향이 존재하는 상태에서 current target을 계속 defer하면
  adapter 구현과 평가 기준이 모호해진다.

따라서 현재 slice의 구현 target은 정하되, Product contract와 Shared AI의 architecture contract가
provider에 결합되지 않도록 boundary를 함께 고정한다.

## 3. Decision

첫 production Shared AI model-inference slice의 **CURRENT implementation target은 OpenAI API /
GPT family**다.

이 결정은 다음을 의미한다.

- 첫 concrete Spring AI model provider adapter의 target은 OpenAI provider/API다.
- OpenAI API/GPT를 real-provider development, evaluation과 smoke baseline으로 사용한다.
- model invocation은 Shared AI의 neutral contract를 통해 synchronous하게 수행한다.
- neutral contract가 지원하는 경우 streaming invocation도 target 범위에 포함한다.

이 결정은 다음을 의미하지 않는다.

- permanent provider choice 또는 exclusive provider choice
- Product-facing contract나 Product business rule
- embeddings provider/model 또는 reranker provider/model
- exact GPT model/version 선택

## 4. Four Separate Layers

Runtime host, integration framework, current provider target과 architecture contract는 서로 다른
layer이자 서로 다른 결정이다.

| Layer | 현재 의미 | 이 ADR과의 관계 |
| --- | --- | --- |
| A. Runtime host | Spring Boot / Spring MVC / JVM Platform Core | ADR-0005/0006의 runtime boundary를 사용하며 provider 결정이 아님 |
| B. AI integration framework | Spring AI | adapter/integration library이며 canonical application contract가 아님 |
| C. Current model-inference provider target | OpenAI API / GPT family | 이 ADR이 현재 target으로 결정 |
| D. Stable architecture contract | Shared AI-owned provider-neutral ports/contracts | Product와 application core가 의존하며 provider 교체 가능성을 보존 |

이 네 layer를 Spring AI, OpenAI와 Shared AI architecture의 단일 선택으로 합치지 않는다.

## 5. Provider-Neutral Boundary

Canonical dependency direction은 다음과 같다.

```text
Product
    ↓
Shared AI neutral application contract
    ↓
ModelGateway / application-owned neutral provider port
    ↓
Spring AI adapter
    ↓
OpenAI provider/API
```

다음 invariant를 지킨다.

- Product contract에는 OpenAI DTO/SDK type을 넣지 않는다.
- Product business rule에는 GPT model name을 넣지 않는다.
- Execution Context에는 OpenAI type을 넣지 않는다.
- Evidence에는 OpenAI-specific schema를 만들지 않는다.
- Skill/Lens/Workflow contract에는 OpenAI type을 넣지 않는다.
- Product가 OpenAI client를 직접 호출하지 않는다.
- provider-specific Product semantics를 만들지 않는다.
- OpenAI exception/failure taxonomy를 application 또는 Product contract로 leak하지 않는다.
- provider-specific retry contract를 application 또는 Product contract로 leak하지 않는다.

기존 generic Evidence field가 허용하는 경우에만 opaque provider/model operational reference를
담을 수 있다. 이것은 새로운 OpenAI-specific Evidence type을 만드는 근거가 아니다.

## 6. Decided Scope

이 ADR에서 현재 결정된 범위는 다음과 같다.

- 첫 production Shared AI model-inference provider target = OpenAI API
- 현재 model-family target = GPT
- 첫 concrete Spring AI model provider adapter
- real-provider development/evaluation/smoke baseline
- neutral contract를 통한 synchronous model invocation
- neutral contract가 지원하는 범위의 streaming invocation
- model identifier는 runtime configuration 또는 Shared AI-owned alias로 공급

여기서 model identifier를 공급할 수 있다는 것은 특정 GPT model/version이나 alias mapping을
선택했다는 뜻이 아니다.

## 7. Explicitly Deferred

다음은 이 ADR에서 결정하지 않는다.

- exact GPT model/version
- model alias mapping details
- embeddings provider/model
- reranker provider/model
- chunking/ranking policy
- image provider/capabilities
- audio provider/capabilities
- moderation provider/capabilities
- multi-provider routing
- fallback
- provider-specific retry policy
- permanent provider status
- exclusive provider status
- Bedrock/Anthropic/Gemini/local/open-model future adapters
- Pydantic AI
- Python runtime
- future agent runtime

PostgreSQL/pgvector retrieval persistence를 사용하는 것은 retrieval persistence 선택일 뿐이며,
OpenAI embeddings를 의미하지 않는다.

## 8. Future Providers and Contract Stability

Anthropic, Bedrock, Gemini, local/open model 또는 다른 provider는 향후 별도 adapter와 별도
decision을 통해 OpenAI를 대체하거나 OpenAI와 공존할 수 있다. 그 경우에도 Product contract,
Execution Context, Skill/Lens/Workflow contract와 neutral Evidence boundary를 provider type에
맞춰 변경하지 않는다.

## 9. RPL-107 Evidence Relationship

RPL-107의 current implementation/status projection은
[source-of-truth-map §2.1](../source-of-truth-map.md#21-current-implementation-projection)이
단일 owner로 유지한다. 이 ADR은 OpenAI/GPT provider target과 provider-neutral boundary를
소유하며, mutable PR/SHA/gate/smoke package를 복제하지 않는다.

현재 projection에서 OpenAI synchronous Slice A는 `IMPLEMENTED`다. Exact GPT model은 runtime
configuration으로 남고, `gpt-5.4-mini`는 smoke evidence이지 영구 architecture mandate가
아니다. Streaming Slice B, Gemini, routing/fallback, Product integration, Finance/Carelog
user-facing AI feature와 Python/Pydantic/Agent runtime은 각각 `DEFERRED`, `MISSING` 또는
미구현 범위로 source-of-truth map에 기록된다. 이 implementation evidence는 Product release,
runtime support 또는 deployment를 자동으로 의미하지 않는다.

## 10. Foundation Boundary

Foundation ADR-0015는 provider adapter, API key/secret, model alias와 provider operations를
Shared AI 책임으로 이미 위임한다. 따라서 이 decision에는 Foundation ADR delta가 필요하지 않으며,
`harness-foundation-docs`는 변경하지 않는다.

## 11. Status Reconciliation

Current implementation/status의 canonical projection은
[source-of-truth-map §2.1](../source-of-truth-map.md#21-current-implementation-projection)에
반영한다.

- PR #7은 canonical `shared-ai-architecture` `main`에 merge됐다.
- RPL-107 OpenAI synchronous Slice A는 위 current projection에서 `IMPLEMENTED`로 기록된다.
- README, execution-model과 Shared AI charter는 이 ADR과 source-of-truth map을 참조하는 최소
  projection만 가진다.

ADR-0006의 당시 branch/projection 상태와 그 Accepted history는 재작성하지 않는다.

## 12. Runtime Evolution Relationship

ADR-0007이 canonical이 된 뒤 별도 Runtime Evolution policy는 필요할 때 다음을 평가할 수 있다.

- bounded workflows
- agent-complexity triggers
- Pydantic AI
- same-workflow comparative PoC
- LLM-controlled-by-code evaluation criteria
- runtime/operations/resource trade-offs

이 평가는 ADR-0007의 범위가 아니며, Pydantic AI·Python·future agent runtime을 현재 결정하거나
commit하는 것이 아니다.

## 13. Related Records

- [ADR-0005](ADR-0005-runtime-placement-platform-core-modular-monolith-boundary.md) — Phase 1 runtime placement
- [ADR-0006](ADR-0006-runtime-technology-framework-dependency-boundary.md) — runtime technology와 Spring AI adapter boundary
- [ADR-0004](ADR-0004-execution-context-api-tenant-isolation.md) — provider-neutral Execution Context
- [ADR-0002](ADR-0002-evidence-provenance-core-domain-boundary.md) — neutral Evidence envelope와 domain extension
- [ADR-0003](ADR-0003-retrieval-knowledge-runtime-ownership-lifecycle.md) — retrieval/embedding ownership과 deferred technology
- `harness-foundation-docs` ADR-0015 — Shared AI provider operations delegation
- RPL-103 — runtime placement/technology decision evidence
- RPL-104 — provider-neutral vertical-slice evidence
- RPL-106 — canonical platform-services neutral foundation
- RPL-107 — implementation evidence only; canonical decision authority 아님
