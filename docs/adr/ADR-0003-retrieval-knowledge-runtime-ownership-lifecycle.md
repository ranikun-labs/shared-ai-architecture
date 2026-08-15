# ADR-0003 — Retrieval/Knowledge Runtime Ownership과 Projection Lifecycle

| 항목 | 값 |
| --- | --- |
| status | Accepted |
| jira | RPL-101 (parent RPL-98) |
| predecessor | RPL-100 / [ADR-0002](ADR-0002-evidence-provenance-core-domain-boundary.md) |
| ADR namespace | shared-ai-architecture 로컬 (`ADR-000N`) |
| 상위 canonical | `harness-foundation-docs` ADR-0013/ADR-0015, `finance-harness-docs` knowledge/routing/grounding contracts, `carelog-architecture` ownership/context contracts |
| 해소 대상 | OD-2 (Retriever/RAG runtime 소유), OD-6 (corpus projection 소유·lifecycle) |

---

## 1. Status

**Accepted.**

4개 canonical repository의 fresh `origin/main`을 교차 확인한 결과, Product-owned Knowledge와
Shared AI-owned retrieval execution substrate 사이에 blocking conflict가 없다. 물리 저장소,
embedding model, framework 또는 wire schema를 선택하지 않고도 logical ownership, isolation,
projection lifecycle을 독립적으로 확정할 수 있다. 교차 확인 SHA는 §18에 고정한다.

## 2. Context

[ADR-0001](ADR-0001-skill-lens-workflow-execution-ownership.md)은 Product/Domain이 capability의
정의·선택·검증을 소유하고 Shared AI가 generic execution mechanism을 소유한다고 확정했다.
[ADR-0002](ADR-0002-evidence-provenance-core-domain-boundary.md)는 Shared AI가 evidence correlation
mechanism을 제공하되 evidence admissibility와 domain meaning은 Product/Domain에 남긴다고 확정했다.

RPL-97은 다음을 미결로 남겼다.

- **OD-2**: Retriever/RAG, embedding, index infrastructure의 Shared AI 소유 범위.
- **OD-6**: tenant-scoped corpus의 저장·소유와 lifecycle 경계.

핵심 질문은 저장 제품이 아니라 authority다.

> Product source가 chunk, embedding 또는 index row로 투영됐을 때 무엇이 canonical이고, 누가
> corpus admission·검색 의미·갱신·삭제·권한 변경을 결정하며, 누가 그 결정을 실행하는가?

Foundation은 각 Product가 자기 데이터·migration을 소유하고 다른 service의 DB 직접 접근,
cross-service FK와 JOIN을 금지한다. Finance는 Markdown을 semantic Authoring SoT로 유지하고 향후
DB도 검증된 Runtime Projection으로만 허용한다. Carelog는 Workspace/Customer/CustomerEvent 의미와
Product authorization을 소유한다. 따라서 Shared AI가 runtime projection을 소유하더라도 Product
source나 authorization을 흡수할 수 없다.

## 3. Decision

1. **Product/Domain은 Source Knowledge의 Semantic SoT를 소유한다.** corpus admission·membership,
   source authorization, semantic chunk constraints, corpus selection, retrieval intent/policy 의미,
   freshness/effective-time 의미와 최종 사용·검증도 Product/Domain 소유다.
2. **Shared AI는 provider-neutral Retrieval/RAG execution substrate를 소유한다.** 승인된 source의
   registration projection, parse/chunk/embedding/index 실행, lexical/vector/hybrid query와 filter/rerank
   실행, projection lifecycle·기술 상태, staleness reporting, evidence/provenance correlation을 담당한다.
3. **Runtime Projection은 Semantic SoT가 아니다.** chunk, embedding, lexical/vector index와 runtime
   registry는 source로부터 재생성 가능한 기술 투영이다. projection이 source보다 오래되면 source가
   이기며 projection은 canonical knowledge로 주장할 수 없다.
4. Shared AI는 Product DB를 직접 crawl하거나 corpus를 임의 구성하지 않는다. Product/Domain이
   승인하고 scope·source/version·policy reference를 제공한 등록 요청만 실행한다.
5. exact structured fact는 Product Tool/structured SSOT에서 얻는다. RAG narrative를 금액·날짜·상태·
   count·payment status·pain score·appointment·market price의 authoritative source로 사용하지 않는다.
6. product와 tenant/workspace scope는 admission, registration, ingest, index, query, evidence,
   invalidation, rebuild, delete 전 경로에서 보존한다. unscoped global corpus는 금지한다.
7. source 삭제, 권한 철회 또는 scope 변경이 확인되면 기존 projection은 **즉시 retrieval-ineligible**가
   되어야 한다. 물리 삭제·rebuild가 비동기일 수 있어도 접근 차단은 지연을 정당화하지 않는다.
8. 이 ADR은 logical ownership과 lifecycle만 확정한다. persistence, transport, framework, model,
   index schema와 concrete API는 RPL-102/RPL-103 및 구현 decision으로 이월한다.

핵심 invariant:

```text
Source Knowledge / Semantic SoT = Product / Domain
Runtime Projection              = Shared AI-owned technical projection
Retrieval Execution             = Shared AI-owned mechanism
Admission / Selection / Meaning = Product / Domain
```

## 4. Semantic SoT vs Runtime Projection

```text
Product-owned source
  ├─ Finance Markdown / journal narrative
  └─ Carelog-authorized treatment memo / conversation
          ↓ Product admission + semantic constraints
Shared AI Runtime Projection
  ├─ parsed representation
  ├─ chunks
  ├─ embeddings
  └─ lexical/vector index entries
          ↓ Product-declared corpus/scope/policy
Shared AI Retrieval Execution
          ↓
Evidence reference + Product validation
```

- Product source 변경은 semantic source version의 변경이다. 기존 projection은 새 source를 대체하지 않는다.
- embedding/index 기술 변경은 projection rebuild 사유이지 Product source 의미 변경이 아니다.
- projection 삭제는 source 삭제를 의미하지 않는다. 반대로 source 삭제·권한 철회는 projection을
  retrieval에서 제거해야 한다.
- Shared AI registry가 source descriptor를 보관하더라도 그것은 runtime projection이며 Product의
  canonical corpus definition을 대체하지 않는다.
- projection을 durable하게 저장하는 경우 logical data·schema·migration owner는 Shared AI다. Product는
  source storage의 owner로 유지된다. 같은 physical engine에 배치해도 ownership은 합쳐지지 않으며,
  cross-service FK·cascade delete 대신 명시적 lifecycle contract로 동기화한다.

## 5. Ownership Matrix

| Concern | Product/Domain | Shared AI |
| --- | --- | --- |
| Source SoT | source content·business meaning·canonical version | opaque reference와 version awareness |
| Corpus admission | 포함/제외 결정, membership 의미 | 승인된 admission을 등록·집행 |
| Source authorization | principal/resource 접근 판정 | 결정된 scope·authorization constraint 전파·집행 |
| Registration | owner-approved descriptor와 scope 제공 | provider-neutral runtime registration projection |
| Parse/chunk semantic constraints | section/clause/domain boundary 선언 | constraint를 보존하는 parse/chunk mechanism |
| Chunk execution | — | generation, overlap/tokenizer adaptation mechanism |
| Embedding execution | model 의미를 domain authority로 사용하지 않음 | embedding 생성·교체·rebuild mechanism |
| Index projection | source authority 유지 | lexical/vector projection build·health·delete |
| Corpus selection | workflow별 corpus/resource/source class 선택 | 선언된 selection을 임의 변경 없이 실행 |
| Retrieval policy semantics | time, source class, admissibility, domain filter 의미 | lexical/vector/hybrid/filter/top-k의 technical execution |
| Retrieval execution | 결과의 domain 사용 조건 | query, filter, rerank, timeout, technical failure normalization |
| Rerank execution | 허용 후보와 domain constraint | 선언된 후보 범위의 technical mechanism |
| Evidence correlation | admissibility·domain extension·최종 검증 | ADR-0002 Core reference와 execution correlation |
| Source update | 새 canonical content/version 발행 | stale 판정 seam, invalidation/rebuild 실행 |
| Invalidation | 무효화 사유·fallback 의미 결정 | projection eligibility 차단·상태 보고 |
| Delete propagation | source deletion과 domain retention 결정 | retrieval 차단 후 projection 삭제 실행 |
| Final validation | 충분성·freshness·uncertainty·업무 사용 판정 | technical result와 상태 제공 |

`Registration`은 공동 소유를 뜻하지 않는다. Product는 무엇을 등록할지 결정하고, Shared AI는 그
결정을 runtime projection으로 materialize한다. 어느 한쪽도 상대 영역을 대신하지 않는다.

## 6. Ingestion Boundary

### 6.1 Admission

“이 source를 AI retrieval corpus에 넣을 것인가?”는 Product/Domain decision이다. admission에는
최소한 source owner, product/tenant scope, source reference/version, 허용 resource 범위와 적용할
Product-owned policy reference가 따라야 한다. exact descriptor/wire schema는 RPL-102 scope다.

### 6.2 Registration

Shared AI는 owner-approved descriptor를 runtime registry에 투영할 수 있다. 이 registration은:

- Product DB table 위치나 credential을 포함하지 않는다.
- source를 의미상 승인하거나 Finance/Carelog classification을 판정하지 않는다.
- opaque source reference와 scope를 사용하며 raw customer name, treatment note, journal text를
  공통 metadata로 복제하지 않는다.

### 6.3 Parse와 Chunk

Product/Domain은 section, clause, chronology처럼 의미 보존에 필요한 constraint를 선언한다.
Shared AI는 그 constraint 안에서 parsing, normalization, chunk generation, overlap과 tokenizer
adaptation을 실행한다. exact chunk size·algorithm은 이 ADR에서 정하지 않는다.

Shared AI가 semantic boundary를 알 수 없거나 constraint를 만족하지 못하면 임의 chunk로 성공 처리하지
않고 technical failure/not-ready 상태를 반환한다. Product source 자체는 영향받지 않는다.

### 6.4 Embed와 Index

embedding과 lexical/vector index는 재생성 가능한 projection이다. producer 또는 index 기술 교체는
새 projection build를 만들 수 있지만 source artifact를 다시 작성하거나 semantic version을 자동으로
올리지 않는다.

## 7. Retrieval Boundary

Product/Domain semantic policy:

```text
- 어떤 corpus와 resource scope를 검색하는가
- 어떤 source class와 기준시각이 허용되는가
- 어떤 evidence type이 admissible한가
- 결과를 Domain에서 어떻게 사용·검증하는가
```

Shared AI technical execution:

```text
- lexical / vector / hybrid query execution
- declared metadata and scope filter execution
- declared candidate set 안의 technical reranking
- top-k, timeout, adapter failure와 technical diagnostics
```

“hybrid retrieval을 실행한다”는 기술 mechanism이지 “어떤 Finance corpus를 검색한다”는 semantic
decision이 아니다. Shared AI는 LLM inference나 전역 heuristic으로 corpus를 임의 선택하지 않는다.
Product가 선언한 bounded corpus와 policy reference 안에서만 실행한다.

## 8. Structured Fact vs Narrative Retrieval

| 데이터 | authoritative 경로 | Retrieval 사용 |
| --- | --- | --- |
| 금액·미납 상태·계약일 | Product Tool / structured SSOT | narrative 보조만 |
| appointment·pain score·count·상태 | Product Tool / structured SSOT | memo/history 보조만 |
| market price·기준시각 데이터 | Finance-approved structured Tool/source | 문서 검색으로 대체 금지 |
| 상담·치료 memo, conversation, journal narrative | Product-admitted Retriever corpus | source reference가 있는 narrative context |
| Finance canonical knowledge text | Finance Semantic SoT의 승인된 projection | Finance corpus/routing/admissibility 안에서만 |

retrieved narrative에 정확한 숫자가 포함돼도 그것만으로 current structured fact가 되지 않는다.
Product가 그 숫자를 authoritative fact로 사용할 필요가 있으면 Product Tool/SSOT로 별도 확인한다.

## 9. Logical Lifecycle

| 단계 | Semantic owner | Execution owner | Authority / scope invariant |
| --- | --- | --- | --- |
| Source admitted | Product/Domain | — | source owner가 corpus membership·scope 승인 |
| Registered | Product descriptor | Shared AI | 승인된 ref/version/scope만 runtime projection으로 등록 |
| Ingestion | semantic constraint=Product | Shared AI | scope 유지, raw DB crawl 금지 |
| Projection build | source authority=Product | Shared AI | source version에 연결된 rebuildable projection |
| Retrieval | corpus/policy=Product | Shared AI | product+tenant+resource constraint 안에서만 query |
| Evidence correlation | admissibility=Product | Shared AI Core | source/projection/execution/scope refs 연결 |
| Source update | Product | Shared AI invalidation/rebuild | old projection은 새 source authority가 아님 |
| Invalidation | 사유·fallback=Product | Shared AI | invalidated projection retrieval 금지 |
| Source delete | Product | Shared AI propagation | eligibility 즉시 차단, projection 삭제 실행 |
| Permission/scope change | Product authorization | Shared AI enforcement | old scope에서 즉시 retrieval-ineligible |

이 표는 구현 state machine이나 enum을 확정하지 않는다.

## 10. Update, Delete와 Permission Change

### Source update

새 source version이 발행되면 그 version에 대응하는 projection이 준비됐는지 구분할 수 있어야 한다.
old projection은 자동으로 최신 source가 되지 않는다. Product policy가 stale projection 사용을 허용하지
않으면 Shared AI는 fail closed/not-ready로 반환한다.

### Source delete

source owner가 삭제를 확정하면 해당 source projection을 새 retrieval 결과에 노출하지 않는다.
Shared AI의 physical cleanup이 별도 작업이어도 logical retrieval 차단이 먼저다. Product source의
retention·복구 의미를 Shared AI가 결정하지 않는다.

### Permission revoke와 scope reassignment

권한 철회 또는 tenant/resource reassignment가 확인된 순간 기존 scope projection은 retrieval-ineligible다.
재색인, cache 만료 또는 manual cleanup까지 계속 검색 가능하게 두는 것은 security failure다. transport나
event mechanism은 deferred지만, 누락·지연을 탐지하고 reconciliation할 seam은 필요하다.

## 11. Isolation과 Data Minimization

1. 모든 lifecycle operation은 product와 tenant/workspace scope를 필수로 보존한다.
2. principal/resource/customer는 필요할 때 opaque weak reference로 전달한다. identifier 단독은 권한이 아니다.
3. Shared AI는 scope constraint를 집행하지만 Product-local authorization을 발명하거나 대체하지 않는다.
4. customer name, full treatment note, symptom text, conversation, private journal은 Core registration/index
   metadata에 기본 복제하지 않는다. source content 처리 자체도 Product admission·access contract 안에서만 한다.
5. Product A의 source/projection이 Product B query에 나타나는 구조와 tenant scope 없는 global vector
   corpus를 금지한다.
6. cross-service FK, direct Product DB read와 credential 공유를 금지한다.

## 12. Failure와 Staleness Boundary

| 상황 | 분류 | Owner response |
| --- | --- | --- |
| projection/adapter/index unavailable | technical failure | Shared AI가 명시적 failure/not-ready와 diagnostics 제공 |
| query 성공, match 없음 | empty retrieval result | system failure로 가장하지 않음; Product가 의미·fallback 결정 |
| source/projection version mismatch | stale projection | Shared AI가 감지·표시하고 Product policy에 따라 차단 가능하게 함 |
| Domain 기준 evidence 부족 | domain insufficiency | Product/Domain이 uncertainty·사용 가능성 판정 |
| 권한/scope 확인 불가 | security boundary failure | retrieval fail closed; 임의 grace/bypass 금지 |

Shared AI는 stale projection을 canonical source처럼 주장하지 않는다. “stale일 때 사용자에게 무엇을
말할지”와 point-in-time/freshness의 업무 의미는 Product/Domain 소유다.

## 13. Retrieval → Evidence Integration

Retriever 결과는 [ADR-0002](ADR-0002-evidence-provenance-core-domain-boundary.md)의
`retrieved_content`로 correlation한다. 최소 logical linkage는 다음을 보존한다.

```text
opaque source reference
+ execution/step correlation
+ product/tenant scope
+ available source artifact version reference
+ projection/producer version reference
```

source ref는 raw payload나 Product DB FK가 아니다. projection provenance는 source evidence를 대체하지
않고, Finance/Carelog admissibility도 판정하지 않는다. Evidence payload/storage/wire schema는
ADR-0002와 RPL-102 경계를 따른다.

## 14. Versioning

각 version 축은 독립이다.

```text
Source Artifact Version       = Product/Domain semantic source version
Projection Version            = Shared AI runtime projection compatibility/identity
Embedding Producer Version    = embedding 생성 mechanism version
Projection Build Version      = parse/chunk/index build mechanism version
Retriever Runtime Version     = query/filter/rerank execution mechanism version
```

- source가 version을 제공하지 않으면 Shared AI가 가짜 semantic version을 만들지 않는다.
- 모든 축을 모든 source에 필수화하지 않는다. 재현·staleness 판단에 존재하는 version만 연결한다.
- embedding/index/retriever 교체는 Product semantic source version bump를 자동 요구하지 않는다.
- source 의미 변경은 projection version 변경만으로 숨길 수 없다.

## 15. Consumer Validation

### Carelog Therapy — PASS

- Appointment와 pain score `7→5→3`은 Tool/structured fact다.
- Treatment memo admission·access·customer scope는 Carelog가 소유한다.
- Shared AI는 승인된 memo의 retrieval projection과 execution만 소유한다.
- “개선 추세”는 Lens/Model interpretation이며 source fact가 아니다.
- deletion/permission change 후 old customer scope retrieval을 금지한다.

### Carelog Rental — PASS

- `700,000원 미납`, 계약일과 상태는 Product Tool/structured SSOT다.
- “20일까지 납부” 대화는 Carelog가 admission한 retrieved narrative다.
- index의 숫자를 현재 미납 사실로 승격하지 않는다.
- Workspace scope와 대화 접근 권한을 ingest/query/delete 전 경로에서 유지한다.

### Finance — PASS

- Journal semantics와 corpus admission은 Finance 소유다.
- Finance Markdown은 Finance Semantic/Authoring SoT이며 projection으로 승격되지 않는다.
- market fact와 point-in-time source는 Finance-approved structured Tool/source가 담당한다.
- Finance가 corpus/routing/admissibility·기준시각·uncertainty·PolicyGuard·Validator 의미를 소유한다.
- Shared AI는 승인된 knowledge projection의 technical retrieval과 ADR-0002 correlation만 제공한다.
- 이 ADR은 현재 Finance MVP의 WAS private resource를 재분류하거나 ingestion을 활성화하지 않는다.
  향후 Finance가 source를 명시적으로 admission할 때만 Shared AI projection을 만들 수 있다.

세 consumer 모두 동일한 logical substrate를 쓰며 Finance/Carelog 전용 core field가 필요하지 않다.

## 16. RPL-103 Technology ADR Handoff

RPL-103은 이 ownership을 보존하는 기술을 선택해야 한다. Phase 1 candidate는 최소 다음을 지원해야 한다.

- product/tenant/resource-scoped filtering을 모든 lifecycle operation에서 강제할 수 있음
- lexical과 vector retrieval을 contract 뒤에서 제공하거나 단계적으로 추가할 수 있음
- source를 변경하지 않고 projection rebuild와 producer 교체가 가능함
- invalidation/delete/permission revoke 후 retrieval eligibility를 fail closed로 차단할 수 있음
- source/projection version과 staleness를 추적·보고할 수 있음
- evidence에 opaque source ref와 projection/execution provenance를 반환할 수 있음
- provider/framework-neutral contract 뒤에 숨길 수 있음

이는 평가 기준이지 PostgreSQL, pgvector, dedicated vector DB, LlamaIndex, RAGFlow, LangGraph, embedding
vendor 또는 reranker 선택이 아니다.

## 17. Consequences

**긍정**

- Product semantic authority를 보존하면서 ingestion/retrieval lifecycle의 중복 구현을 줄인다.
- projection을 교체·재생성할 수 있어 기술 변경이 Product knowledge rewrite로 번지지 않는다.
- update/delete/permission change가 retrieval isolation과 직접 연결된다.
- ADR-0002 evidence가 source와 projection provenance를 혼동하지 않고 연결된다.

**비용 / 트레이드오프**

- Product는 admission, semantic constraints, authorization, freshness와 fallback contract를 제공해야 한다.
- Shared AI는 build 성공뿐 아니라 invalidation, staleness, delete propagation과 reconciliation을 운영해야 한다.
- reference-first 경계는 source가 삭제·비가용일 때 Product와 Shared AI 간 lifecycle coordination을 요구한다.
- fail-closed eligibility는 stale data 노출을 막는 대신 projection 재구축 중 retrieval availability를 낮출 수 있다.

## 18. Canonical Source Reality

| Repository | Canonical branch | fresh `origin/main` SHA |
| --- | --- | --- |
| `shared-ai-architecture` | `main` | `aa56c2b52aaf3f7350b95e11f73b059576b0fe87` |
| `harness-foundation-docs` | `main` | `69fceb267094756913f400f8b4045590bd0e8ac5` |
| `finance-harness-docs` | `main` | `5326977691b420f573a469c0bb7635de26fe2aba` |
| `carelog-architecture` | `main` | `6b581dd3cc92d5e8486172525898d674e8d61b7c` |

SHA는 재현 anchor이며 영구 invariant가 아니다. feature/review checkout은 canonical source로 사용하지
않았다.

## 19. Rejected Alternatives

- **Shared AI가 Product DB를 직접 crawl** — DB ownership·authorization을 우회하므로 기각.
- **vector index를 canonical knowledge DB로 취급** — stale projection을 source authority로 승격하므로 기각.
- **모든 Product data를 embedding** — admission·privacy·cost·deletion 경계를 파괴하므로 기각.
- **Product마다 완전히 별도 retrieval framework** — semantic ownership 보존에는 도움이 되지만 제품 중립
  execution substrate와 lifecycle을 중복 구현하므로 기각.
- **Shared AI가 corpus semantics까지 소유** — Product Workflow/Corpus/Domain Validation ownership과 충돌.
- **exact structured fact를 vector search로 조회** — narrative를 Product SSOT처럼 사용하게 하므로 기각.
- **deletion/permission change를 manual cleanup에만 의존** — 철회 후 cross-scope 노출을 허용하므로 기각.
- **framework type을 canonical contract로 사용** — ownership을 특정 implementation에 결합하므로 기각.

## 20. Non-goals와 Deferred Technology Decisions

```text
- PostgreSQL/pgvector 또는 dedicated vector DB 선택
- LlamaIndex/RAGFlow/LangGraph/GraphRAG adoption
- embedding model, reranker, exact chunk size/algorithm 선정
- DB table/index schema, migration, retention job
- FastAPI/Python runtime 구현
- NATS/event subject/transport 구현
- API/DTO/wire schema (RPL-102)
- Runtime technology 결정 (RPL-103)
- universal knowledge graph/ontology, plugin marketplace, autonomous crawler, global memory platform
- Finance knowledge/evidence/routing/PolicyGuard/Validator semantics 변경
- Carelog Customer/CustomerEvent/privacy/business rule 변경
```

OD-2/OD-6은 ownership과 logical lifecycle 관점에서 해소한다. 물리 기술 선택은 해소 조건이 아니며
RPL-103으로 명시적으로 deferred한다.

## 21. Related Canonical Sources

- `harness-foundation-docs/docs/adr/ADR-0013-target-deployment-and-data-boundaries.md`
- `harness-foundation-docs/docs/adr/ADR-0015-platform-communication-messaging-scaling.md` §7.4, §7.8, §10
- `harness-foundation-docs/docs/architecture/backend-service-foundation/database-ownership-and-reference-policy.md`
- `finance-harness-docs/00-core/0-d-knowledge-architecture/finance-knowledge-storage-decision.md`
- `finance-harness-docs/00-core/0-d-knowledge-architecture/finance-knowledge-artifact-metadata-standard.md`
- `finance-harness-docs/00-core/0-d-knowledge-architecture/finance-document-exposure-policy.md`
- `finance-harness-docs/00-core/0-c-runtime-contract/finance-routing-contract.md`
- `finance-harness-docs/00-core/0-a-safety-core/finance-market-data-grounding.md`
- `finance-harness-docs/06-professional-standards/finance-evidence-and-uncertainty-standard.md`
- `carelog-architecture/architecture/ownership-map.md`
- `carelog-architecture/contracts/authenticated-carelog-context.md`
- `carelog-architecture/contracts/customer-ownership.md`
- `carelog-architecture/contracts/customer-event-lifecycle.md`
