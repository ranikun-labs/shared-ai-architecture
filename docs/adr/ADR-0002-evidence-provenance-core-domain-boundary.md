# ADR-0002 — Evidence/Provenance Core Envelope와 Domain Extension 경계

| 항목 | 값 |
| --- | --- |
| status | Accepted |
| jira | RPL-100 (parent RPL-98) |
| predecessor | RPL-99 / [ADR-0001](ADR-0001-skill-lens-workflow-execution-ownership.md) |
| ADR namespace | shared-ai-architecture 로컬 (`ADR-000N`) |
| 상위 canonical | `harness-foundation-docs` ADR-0013/ADR-0015, `finance-harness-docs` evidence/grounding/validator/routing contracts, `carelog-architecture` ownership/context contracts |
| 해소 대상 | OD-8 (Evidence core envelope vs Domain extension 경계) |

---

## 1. Status

**Accepted.**

4개 canonical repository의 fresh `origin/main`을 교차 확인한 결과, Shared AI의 제품 중립
execution mechanism과 Product/Domain의 evidence semantics 사이에 blocking conflict가 없다.
물리 저장, API wire schema, Audit 통합 없이도 logical ownership과 compatibility 원칙을 독립적으로
확정할 수 있으므로 OD-6을 기다리지 않는다. 교차 확인 SHA는 §17에 고정한다.

## 2. Context

[ADR-0001](ADR-0001-skill-lens-workflow-execution-ownership.md)은 다음 경계를 확정했다.

```text
Semantic Source of Truth = Product / Domain
Runtime Projection       = Shared AI execution projection
Execution Mechanism      = Shared AI
```

Shared AI는 Tool/Retriever/Lens/Policy/Model 실행과 evidence correlation mechanism을 제공할 수
있다. 그러나 Finance는 기준시각·admissibility·불확실성·Output Validator 의미를 소유하고,
Carelog는 Customer/Workspace/CustomerEvent·민감정보·Product authorization 의미를 소유한다.
따라서 다음 두 질문을 분리해야 한다.

```text
Core contract : 근거와 실행을 제품 중립적으로 어떻게 식별·연결하는가?
Domain meaning: 그 근거가 이 업무 판단에 유효한가, 어떻게 해석·검증·보존하는가?
```

## 3. Decision

1. Shared AI는 모든 Product가 사용할 **logical Evidence Core Envelope**와 execution correlation
   mechanism을 소유한다.
2. Product/Domain은 evidence admissibility, source truth, temporal meaning, uncertainty,
   domain classification, validation, result persistence와 business use를 소유한다.
3. Source Evidence와 Model Interpretation을 분리한다. Model output은 원본 근거로 승격되지 않는다.
4. Evidence와 Provenance를 연결하되 같은 개념으로 합치지 않는다. Evidence는 응답의 근거이고,
   Provenance는 그 근거와 결과가 어떤 실행 과정에서 사용·생성됐는지 설명한다.
5. Evidence는 Trace, Log, Audit 또는 Result schema가 아니다. 상호 참조할 수 있지만 서로의
   Source of Truth를 대체하지 않는다.
6. Core는 raw business payload를 기본 복제하지 않는다. tenant scope를 유지하는 opaque reference,
   최소 classification, timestamp와 correlation을 우선한다.
7. Domain extension은 Product/Domain-owned, typed, versioned contract를 reference한다. 임의 JSON을
   담는 `domain_metadata` catch-all은 허용하지 않는다.
8. 이 ADR은 logical contract만 확정한다. persistence, transport, concrete JSON schema와 retention은
   후속 owner decision이다.

핵심 ownership invariant:

> **Shared AI는 evidence를 운반하고 correlation할 공통 mechanism을 제공하되, 무엇이 도메인에서
> 유효한 근거인지 판단하는 권한은 Product/Domain에 남긴다.**

## 4. Evidence Type Model

Core의 `kind`는 source/result 성격을 혼동하지 않기 위한 최소 classification이다.

| Kind | 의미 | Source Evidence 여부 | Semantic owner |
| --- | --- | --- | --- |
| `structured_fact` | Tool 또는 구조화된 외부 source가 반환한 정확한 값 | Yes, 단 Product/Domain의 source validation 필요 | Product/External SSOT |
| `retrieved_content` | 문서·record·narrative에서 검색된 context | Yes, source reference와 retrieval context 필요 | Product corpus/retrieval policy |
| `model_interpretation` | source evidence를 바탕으로 Model이 만든 해석 | **No — 단독 source evidence 불가** | Product/Domain validation |
| `policy_result` | Product/Domain policy gate 실행 결과 | Business fact가 아니라 gate result | Policy owner |
| `validation_result` | Output/domain validator 실행 결과 | Business fact가 아니라 validation result | Validator owner |

`external_data`, `retrieved_document`, `retrieved_record` 같은 세부 taxonomy는 전역 enum으로
선제 표준화하지 않는다. 필요한 구분은 source descriptor 또는 versioned Domain extension이 소유한다.

### 4.1 Source Evidence와 Model Interpretation

```text
pain_score 7 → 5 → 3                 = structured_fact
"개선 추세로 보입니다"               = model_interpretation

Payment Tool의 700,000원 미납         = structured_fact
대화 기록의 "20일까지 납부"           = retrieved_content

당시 시점의 market data              = structured_fact
"금리 리스크를 충분히 고려하지 않음" = model_interpretation
```

Interpretation은 자신이 해석한 `source_refs`를 가져야 한다. source가 없거나 유효성을 확인할 수
없으면 Product/Domain 규칙에 따라 unavailable/uncertain으로 처리하며 사실처럼 승격하지 않는다.

## 5. Core Evidence Envelope

본 표는 **logical field contract**다. JSON 이름, scalar 형식, cardinality와 API DTO는 RPL-102 등
후속 API contract에서 확정한다.

| Field | 필수성 | Core인 이유 | Domain으로 두지 않는 이유 |
| --- | --- | --- | --- |
| `evidence_id` | Required | 한 evidence item의 immutable correlation identity | Product별 식별 방식이면 공통 provenance 연결이 깨짐 |
| `kind` | Required | fact/retrieval/interpretation/gate result 혼동 방지 | 제품별 세부 의미가 아니라 최소 공통 분류 |
| `source_refs[]` | Required | 원본 또는 입력 evidence를 opaque하게 연결 | Shared AI가 raw source payload나 business 의미를 소유하지 않게 함 |
| `producer_ref` | Required | Tool/Retriever/Policy/Validator/Model 등 생성 mechanism 식별 | producer의 업무 의미는 Domain 소유지만 실행 상관관계는 공통 필요 |
| `observed_at` | Required | 실행이 source/result를 관측한 시각 | `effective_at`이나 업무 발생시각과 구분되는 기술적 공통 시각 |
| `execution_ref` | Required | evidence를 사용·생성한 execution과 연결 | Evidence와 Provenance를 분리하면서 상호 추적하려면 공통 필요 |
| `scope_ref` | Required | product + tenant/workspace isolation 유지 | tenant 없는 global evidence pool을 금지하기 위한 공통 invariant |
| `schema_version` | Required | Core envelope compatibility 판정 | Domain semantic version과 독립적으로 진화해야 함 |
| `step_ref` | Optional | workflow step 수준 correlation이 필요할 때 사용 | 단일-step 실행에 강제하면 불필요한 결합 발생 |
| `effective_at` | Optional | source가 표현하는 업무/시장 유효시각 전달 | 존재 여부와 의미·freshness 판정은 Domain 소유 |
| `source_artifact_version` | Conditional | versioned source를 사용했을 때 재현성 제공 | 모든 source가 artifact version을 갖는다고 가정하지 않음 |
| `producer_version` | Conditional | versioned producer 실행을 재현할 때 필요 | Tool/Model/Policy별 version model을 Core가 정의하지 않음 |
| `domain_extension_ref` | Optional | typed Domain metadata 계약으로 연결 | Finance/Carelog 필드를 Core에 하드코딩하지 않기 위함 |

### 5.1 Field semantics 제한

- `source_refs[]`는 opaque logical reference다. Product DB 직접 접근, cross-service FK 또는 raw payload
  복제를 뜻하지 않는다.
- `observed_at`은 Shared AI가 값을 관측한 시각이다. Finance의 기준시각, Carelog의 `occurredAt`·
  `scheduledAt`, event `occurred_at`을 재정의하지 않는다.
- `scope_ref`는 최소한 product와 tenant/workspace를 보존한다. principal과 resource/customer는
  필요한 경우 opaque reference로만 추가하며 raw name·treatment text·journal text를 포함하지 않는다.
- `effective_at`이 없다고 Shared AI가 현재값으로 간주하지 않는다. 시간 유효성은 Domain validation이
  판정한다.

## 6. Core와 Domain Extension

Domain extension은 다음 compatibility principle을 따른다.

```text
domain_extension_ref
  → extension_type       (Product/Domain-owned stable identifier)
  → extension_version    (해당 Domain contract version)
  → extension payload    (owner 경계 안에서 검증·노출·보존)
```

- extension은 Product/Domain의 canonical contract가 정의·version·검증한다.
- Core는 `extension_type`과 `extension_version`을 인지하고 전달할 수 있지만 의미를 해석하거나
  임의 변경하지 않는다.
- `domain_metadata: { anything... }` 형태의 unbounded dump는 금지한다. schema ownership,
  compatibility, privacy와 redaction 책임이 사라지기 때문이다.
- Core가 extension payload를 반드시 저장해야 한다는 의미가 아니다. reference-only, inline,
  ephemeral 여부는 후속 API/storage decision에서 정한다.

Finance extension 후보 의미는 point-in-time semantics, market source classification,
uncertainty와 evidence quality다. Carelog extension 후보 의미는 treatment/clinical context
classification이다. **이 ADR은 해당 필드·enum·판정 규칙을 정의하지 않는다.**

## 7. Ownership Matrix

| Concern | Shared AI Core | Product/Domain | Other |
| --- | --- | --- | --- |
| Evidence envelope | logical fields, coarse kind, compatibility | extension contract와 사용 규칙 | API wire는 후속 contract |
| Evidence semantics | 운반·구분만 | admissibility, quality, uncertainty, temporal meaning | External source는 자기 사실 소유 |
| Source truth | source reference 유지 | Product Tool/corpus/domain document | External provider는 원천 데이터 소유 |
| Domain metadata | typed/versioned ref seam | schema, value, validation, redaction | — |
| Correlation | evidence/execution/step 연결 mechanism | workflow/business correlation 의미 | Foundation은 cross-service correlation 원칙 소유 |
| Validation | gate 실행·결과 연결 | policy/validator 규칙, 최종 판정과 책임 | Human/Legal review는 각 owner |
| Trace | AI execution trace mechanism | Product가 필요한 domain context 범위 결정 | Observability stack은 후속 구현 |
| Persistence | 결정하지 않음 | 결과·업무 반영·domain retention | 저장 owner/기술은 후속 decision |

## 8. Evidence vs Provenance

```text
Evidence   = 무엇을 근거로 말했는가
Provenance = 그 근거와 결과가 어떤 실행 과정에서 사용·생성됐는가
```

예를 들어 Rental 응답의 Evidence는 Payment Tool result와 Conversation source reference다.
Provenance는 `RentalPaymentReview` execution, Payment Tool invocation, retrieval invocation,
workflow step과 producer version을 연결한다.

Provenance는 source evidence를 대체하지 않는다. Tool 호출 trace가 존재해도 Tool 결과가 Product
SSOT에서 유효했는지는 Product가 검증한다. 반대로 evidence reference가 존재해도 execution debug
정보 전체를 Evidence에 복사하지 않는다.

## 9. Evidence vs Trace / Log / Audit / Result

| 개념 | 목적 | Owner | 금지되는 대체 |
| --- | --- | --- | --- |
| Evidence | Product 응답의 근거 reference | Core envelope + Domain semantics | Trace/Log를 근거 SoT로 사용 |
| Trace | latency, token, model, tool timing 등 실행 관측 | Shared AI technical mechanism | Evidence payload 전체 저장 |
| Log | runtime 운영·오류 진단 | 실행 service | raw sensitive evidence 기본 기록 |
| Audit | 보안·규제·업무 변경 이력과 보존 | Product/Service/Shared Audit의 별도 contract | Evidence를 자동 Audit record로 승격 |
| Result | 사용자·Product에 반환할 업무 결과 | Product final responsibility | Evidence envelope를 response schema로 사용 |

Conceptual response separation은 다음과 같다.

```text
Result
+ Evidence references
+ Warnings
+ TraceRef (필요 시)
```

이는 실제 API JSON schema가 아니다. Trace, Log, Audit은 evidence와 correlation reference를 공유할
수 있지만 lifecycle, access control, retention과 Source of Truth를 각각 소유한다.

## 10. Scope와 Privacy

1. 모든 Evidence 경로는 product와 tenant/workspace scope를 잃지 않는다.
2. Shared AI는 scope를 전파할 뿐 Product authorization을 판단하지 않는다.
3. principal/resource/customer reference는 stable opaque weak reference를 사용한다. email, customer
   name 또는 mutable display value를 canonical identifier로 사용하지 않는다.
4. Core envelope는 full treatment text, customer name, raw medical note, private journal, user decision
   text, full prompt/response를 기본 복제하지 않는다.
5. Retrieved content는 raw content 대신 source reference와 필요한 최소 metadata를 우선한다.
   실제 content 전달이 필요하면 Product-owned access/privacy contract를 적용한다.
6. Domain extension의 exposure, redaction, retention과 deletion은 Product/Domain owner가 결정한다.
7. tenant scope 없는 global evidence pool과 cross-service physical FK를 금지한다.

## 11. Versioning

네 version 축은 독립이다.

```text
Core Evidence Envelope Version = Shared AI logical envelope compatibility
Domain Extension Version       = Product/Domain semantic contract
Source Artifact Version        = source owner가 정의한 artifact/data version
Execution/Producer Version     = 실행한 Tool/Retriever/Policy/Model/Skill version
```

- Core의 하위 호환 optional field 추가가 Domain semantic version bump를 자동 요구하지 않는다.
- Core가 extension 해석 또는 필수 전달 방식을 바꾸면 Core compatibility review가 필요하다.
- Domain extension 의미 변경은 Domain owner의 version policy를 따른다.
- Source/producer version이 없는 source에 가짜 version을 만들지 않는다.
- version awareness는 ownership transfer가 아니다.

## 12. Consumer Validation — Carelog Therapy

요청: “내일 오는 고객 상태와 이전 치료 반응, 내일 확인할 내용을 정리해줘.”

| Item | Evidence Kind | Semantic Owner | Core Metadata | Domain Extension |
| --- | --- | --- | --- | --- |
| Appointment Tool result | `structured_fact` | Carelog | source/producer/scope/time/execution refs | appointment meaning if needed |
| pain score `7→5→3` | `structured_fact` | Carelog Measurement | source refs + observed/effective time | measurement/trend semantics |
| Treatment History retrieval | `retrieved_content` | Carelog | record refs + scope + retrieval producer | treatment context classification |
| “개선 추세” | `model_interpretation` | Carelog validation | supporting source refs + model producer | therapy interpretation semantics |
| Therapy Safety gate | `policy_result` / `validation_result` | Carelog | policy/step/execution refs | gate rule/version meaning |

**PASS.** Customer/Workspace scope와 Product authorization은 Carelog에 남고, raw treatment history나
customer name을 Core metadata에 복제하지 않는다. `7→5→3`과 “개선 추세”를 분리한다.

## 13. Consumer Validation — Carelog Rental

요청: “이번 달 월세 미납자와 이전 연락 내용을 정리해줘.”

| Item | Evidence Kind | Semantic Owner | Core Metadata | Domain Extension |
| --- | --- | --- | --- | --- |
| Payment Tool `700,000원 미납` | `structured_fact` | Carelog Payment | result ref + scope + producer + time | payment semantics if needed |
| Conversation “20일까지 납부” | `retrieved_content` | Carelog | conversation ref + scope + retrieval provenance | conversation classification if needed |
| Model summary | `model_interpretation` | Carelog validation | both source refs + model execution | Product response semantics |

**PASS.** 미납 금액은 Tool/Product SSOT이고 대화는 narrative retrieval이다. 둘을 동일한 fact level로
취급하지 않으며, 대화 원문을 Core envelope에 기본 복제하지 않는다.

## 14. Consumer Validation — Finance

요청: “내 투자 판단에서 놓친 리스크를 복기해줘.”

| Item | Evidence Kind | Semantic Owner | Core Metadata | Domain Extension |
| --- | --- | --- | --- | --- |
| Journal context | source 성격에 맞는 기존 `kind` 값 | Finance | journal source ref + user scope + execution | user-input/evidence-quality/기타 Finance-specific 분류는 Finance 도메인 extension(`typed/versioned`) |
| Point-in-time Market Tool | `structured_fact` | Finance/external source | source/producer/observed/effective refs | 기준시각·source class·admissibility |
| Finance Knowledge retrieval | `retrieved_content` | Finance | artifact ref/version + execution | Finance evidence quality |
| Risk Lens interpretation | `model_interpretation` | Finance | supporting refs + lens/model producer | uncertainty/interpretation 의미 |
| PolicyGuard | `policy_result` | Finance | policy/step/execution refs | intent/action 의미 |
| Output Validator | `validation_result` | Finance | validator/step/execution refs | severity/pass/rewrite 의미 |

**PASS.** Market data의 기준시각과 admissibility, uncertainty, Lens/PolicyGuard/Validator 의미는
Finance canonical에 남는다. Shared AI는 reference, coarse kind, scope, version awareness와
execution correlation만 제공하며 Finance 규칙을 복사하지 않는다.

## 15. Consequences

**긍정**

- Product가 다른 semantics를 유지하면서도 공통 Evidence/Provenance mechanism을 재사용한다.
- structured fact, retrieval, interpretation과 gate result의 의미 붕괴를 막는다.
- tenant scope와 data minimization이 envelope 수준의 invariant가 된다.
- OD-6 저장 결정 없이 API/storage-neutral contract를 먼저 안정화할 수 있다.

**비용 / 트레이드오프**

- Product는 extension contract와 evidence admissibility/validation을 직접 관리해야 한다.
- Core와 Domain version을 독립 관리하므로 compatibility matrix와 owner review가 필요하다.
- reference-first 설계는 원본 조회 실패·삭제·retention 불일치를 후속 계약에서 다뤄야 한다.
- Core taxonomy를 의도적으로 작게 유지해 일부 Product는 extension 정의 비용을 부담한다.

## 16. Rejected Alternatives

- **모든 evidence semantics를 Shared AI가 소유** — Finance/Carelog canonical ownership을 침범하고
  Product별 admissibility·privacy·time semantics를 전역화하므로 기각.
- **Evidence = Trace/Log/Audit 단일 record** — lifecycle, access, retention, debugging과 business
  meaning을 결합해 raw data 복제와 ownership 혼선을 만들므로 기각.
- **unbounded `domain_metadata` JSON** — schema/version/privacy owner가 사라지는 escape hatch이므로 기각.
- **Finance/Carelog field를 Core에 직접 추가** — 특정 Product 규칙을 global contract로 승격하므로 기각.
- **storage 기술부터 결정** — logical ownership을 PostgreSQL/Redis/NATS에 종속시키고 OD-6을 불필요한
  선행조건으로 만들므로 기각.

## 17. Canonical Source Reality

| Repository | Canonical branch | fresh `origin/main` SHA |
| --- | --- | --- |
| `shared-ai-architecture` | `main` | `449d142cd7efbf28666f5ec63afa0148dcfe4c95` |
| `harness-foundation-docs` | `main` | `69fceb267094756913f400f8b4045590bd0e8ac5` |
| `finance-harness-docs` | `main` | `5326977691b420f573a469c0bb7635de26fe2aba` |
| `carelog-architecture` | `main` | `6b581dd3cc92d5e8486172525898d674e8d61b7c` |

SHA는 재현 anchor이며 영구 invariant가 아니다. feature/review checkout은 canonical source로 사용하지
않았다.

## 18. Non-goals

```text
- actual API/JSON/DTO schema와 endpoint
- PostgreSQL/Redis/NATS 저장·전송 결정, DB table, migration, retention job
- RAG/vector storage, embedding, chunking, reranker (OD-2/OD-6)
- OpenTelemetry 또는 Audit 구현
- Finance evidence/grounding/PolicyGuard/Validator semantics 변경
- Carelog business rule, privacy policy, CustomerEvent schema 변경
- Shared AI runtime/framework/Python/FastAPI 구현
- Result schema 또는 Product response contract 확정
```

## 19. Deferred Decisions

```text
OD-2/OD-6 : Retriever/RAG·Vector infra 소유·범위·물리 저장
OD-3      : Shared AI physical runtime 위치·도입 시점
OD-5      : Shared AI ↔ Product Tool service-to-service 인증
OD-7      : Provider/Identity 비가용 정책
OD-9      : Workflow orchestration 구현 technology
RPL-102   : 실제 API/wire contract 후보
후속 owner decision: persistence, retention, deletion, raw source resolution
```

## 20. Related Canonical Sources

- `harness-foundation-docs/docs/adr/ADR-0013-target-deployment-and-data-boundaries.md`
- `harness-foundation-docs/docs/adr/ADR-0015-platform-communication-messaging-scaling.md` §7.4, §7.8
- `harness-foundation-docs/docs/architecture/backend-service-foundation/database-ownership-and-reference-policy.md`
- `finance-harness-docs/00-core/0-a-safety-core/finance-market-data-grounding.md`
- `finance-harness-docs/00-core/0-a-safety-core/finance-output-validator.md`
- `finance-harness-docs/00-core/0-c-runtime-contract/finance-routing-contract.md`
- `finance-harness-docs/06-professional-standards/finance-evidence-and-uncertainty-standard.md`
- `carelog-architecture/architecture/ownership-map.md`
- `carelog-architecture/contracts/authenticated-carelog-context.md`
- `carelog-architecture/contracts/customer-ownership.md`
- `carelog-architecture/contracts/customer-event-lifecycle.md`
