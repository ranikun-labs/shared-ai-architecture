# ADR-0004 — Execution Context·API Boundary·Tenant Isolation Contract

| 항목 | 값 |
| --- | --- |
| status | Accepted |
| jira | RPL-102 (parent RPL-98) |
| predecessor | RPL-101 / [ADR-0003](ADR-0003-retrieval-knowledge-runtime-ownership-lifecycle.md) |
| ADR namespace | shared-ai-architecture 로컬 (`ADR-000N`) |
| 상위 canonical | `harness-foundation-docs` ADR-0015 §7.2~7.4 / identity-token-contract / database-ownership-policy, `carelog-architecture` authenticated-context/customer-ownership, `finance-harness-docs` runtime/safety contracts |
| 해소 대상 | Shared AI Execution Context·Execute Request/Response 경계·tenant/resource isolation·query-time authorization invariant. OD-5는 부분 확정, OD-7은 security-boundary 부분만 확정 |

---

## 1. Status

**Accepted.**

4개 canonical repository의 fresh `origin/main`을 교차 확인한 결과, Shared AI의 provider-neutral
Execution Context/실행 경계와 Platform Identity·Product authorization·Finance/Carelog domain
ownership 사이에 blocking conflict가 없다. JWT/JWKS/OAuth/FastAPI/NATS/DTO를 선택하지 않고도
"무엇이 운반·신뢰·집행되어야 하는가"라는 logical authority/context contract를 독립적으로 확정할 수
있다. 교차 확인 SHA는 §18에 고정한다.

## 2. Context

선행 ADR이 확정한 경계는 다음과 같다.

- [ADR-0001](ADR-0001-skill-lens-workflow-execution-ownership.md): Product/Domain이 정의·선택·검증을,
  Shared AI가 generic execution mechanism을 소유한다. Phase 1 explicit Skill selection(라우팅 A).
- [ADR-0002](ADR-0002-evidence-provenance-core-domain-boundary.md): Evidence Core envelope는 `scope_ref`
  (product+tenant)를 필수로 보존하고, Shared AI는 scope를 전파할 뿐 authorization을 판단하지 않는다.
- [ADR-0003](ADR-0003-retrieval-knowledge-runtime-ownership-lifecycle.md): retrieval eligibility는
  **query 시점의 현재 authoritative Product/Tenant/Resource authorization/scope**로 판정하며,
  projection/index/cache에 저장된 과거 scope만으로 authorization이 충족된 것으로 간주하지 않는다.

이 세 결정이 전제하지만 아직 명시하지 않은 것은, Product Backend가 Shared AI를 호출할 때
**무엇을 운반하고 Shared AI가 무엇을 신뢰하며 무엇을 집행하는가**이다. 핵심 질문:

> Product Backend가 이미 인증·인가한 실행을 Shared AI가 대신 수행할 때, Shared AI는 Product
> authorization의 SoT가 되지 않으면서도 cross-tenant/cross-resource leakage와 stale-authorization
> 재사용을 어떻게 막는가?

Foundation은 이미 답의 뼈대를 갖고 있다. ADR-0015 §7.3은 `trusted authentication context →
Product-owned authorization`을, §7.2는 "security-sensitive 상태의 Fallback은 값을 추측하지 않고 fail
closed"를 확정했다. identity-token-contract는 product-specific permission을 shared token 기본 claim에서
금지하고 "authorization claim은 stale-token 동작이 허용될 때만 reviewed contract로" 도입하라고 못박았다.
Carelog는 "Shared Platform이 identity를 인증하고 Carelog가 product authorization을 결정하며, Customer
identifier만으로는 권한이 성립하지 않는다"고 소유한다. 이 ADR은 그 원칙들을 Shared AI 실행 경계로
연결할 뿐, 어느 것도 재정의하지 않는다.

이 ADR은 OD-5(Shared AI↔Product service-to-service 인증)의 **신뢰·context 계약(WHAT)** 을 확정하되
concrete credential mechanism(HOW)은 RPL-103으로 남긴다. OD-7은 **Shared AI 자신의 context 검증
security boundary(fail-closed)** 만 확정하고, Product/Identity 가용성에 대한 **domain fallback**은
Product(예: Carelog ADR-C03, 아직 미승인)에 남긴다.

## 3. Decision

1. **Product Backend가 authenticated·authorized Execution Context의 authority다.** Shared AI는 이
   context를 실행 입력으로 신뢰하되, Product authentication/authorization을 재구현하거나 대체하지 않는다.
2. **Shared AI는 provider-neutral Execution Context 계약과 그 무결성/일관성 검증을 소유한다.** 필수 scope
   부재, 내부 불일치, resource가 tenant scope를 벗어나는 요청은 실행 전에 기술적으로 거부한다(fail-closed).
3. **Execution Context는 control-plane metadata이고 Business Input과 분리된다.** raw 민감 payload(치료
   메모·고객명·journal 원문·대화 원문·전체 문서)를 Execution Context에 기본 복제하지 않는다.
4. **Identifier ≠ Authorization.** principal_ref/resource_ref가 context에 존재한다고 permission이
   자동 성립하지 않는다. authority scope는 authenticated principal에서 파생되며 caller가 제시한
   resource id로 확대되지 않는다.
5. **보호 대상 action/query의 eligibility는 query 시점의 현재 authority로 판정한다(ADR-0003 계승).**
   실행 시작 시점 context snapshot은 correlation/audit·original identity용이며 standing authorization
   grant가 아니다. long-running/async/retry/resume에서 과거 snapshot이 독립적으로 접근을 재부여할 수 없다.
6. **child capability(Tool/Retriever/Lens)는 parent execution context보다 넓은 scope를 요청할 수 없다.**
   같거나 좁은 scope만 허용하며, widening이 필요하면 Product-owned explicit decision을 요구한다.
7. **product와 tenant/workspace는 강한 fail-closed boundary다.** product 누락 시 default product,
   tenant 누락 시 global tenant fallback을 금지한다. Product A context로 Product B Skill을 실행하거나
   Tenant A evidence/retrieval을 Tenant B 실행에 사용하지 않는다.
8. **Shared AI는 generic execute 계약을 제공하고, Product-specific use-case endpoint는 Product Backend가
   소유한다.** free-text만 받아 Shared AI가 Product Skill을 임의 선택하는 구조는 금지한다(ADR-0001 A 유지).
9. 이 ADR은 logical authority/context/isolation 계약만 확정한다. JWT/JWKS/OAuth/header/DTO/transport/
   idempotency store/authz cache는 RPL-103 및 후속 owner decision으로 이월한다.

핵심 invariant:

```text
Authentication            = Shared Identity (platform) — 누구인지
Authorization / Scope SoT  = Product Backend — 무엇에 접근 가능한지 (query-time authoritative)
Execution Context Contract = Shared AI — 무엇이 운반·검증·격리되는가
Identifier / Snapshot      ≠ Authorization authority
Child scope                ≤ Parent scope (no implicit widening)
Security-sensitive gap     → fail-closed
```

## 4. Authority Model

**Product Backend가 소유한다:**

```text
- authenticated Product principal 해석과 domain authorization 결정
- semantic resource ownership (예: Customer는 Workspace scope 안에서만 접근)
- Product Skill/use-case 선택과 business request 구성
- authoritative Execution Context 구성(인증·인가 후)
- query-time 현재 authority 판정 seam
- domain 결과 최종 검증과 identity/provider 비가용 시 domain fallback
```

**Shared AI가 소유한다:**

```text
- provider-neutral Execution Context 계약과 무결성/일관성 검증
- product/tenant isolation enforcement seam
- opaque principal/resource propagation
- execution/correlation identifier 발급
- generic Execute Request/Response mechanism
- malformed/inconsistent/scope-widening context의 기술적 거부(fail-closed)
- child capability 실행 시 scope 비확대 집행
```

**Shared AI가 소유하지 않는다:**

```text
- Carelog Customer permission semantics / Finance journal access semantics
- Product business authorization policy와 규칙 원문
- Product workflow/Skill selection semantics
- Identity/provider 비가용 시 Product의 domain 가용성 판단(OD-7 domain 부분)
```

## 5. Execution Context vs Business Input

두 층을 분리한다.

```text
Execution Context (control plane):
  "누가 / 어느 Product / 어느 Tenant / 어떤 authorized scope에서 실행되는가"

Business Input (data plane):
  "무엇을 분석·요약·처리하는가"
```

예시(conceptual):

```text
context = { product=carelog, tenant=W1, principal_ref=P1, resource_ref=(customer:C123) }
input   = { target_visit_date, question, requested_review_mode }
```

raw treatment note 전체·고객명·journal 원문을 Execution Context에 넣지 않는다. Business Input이 민감
payload를 필요로 하면 Product-owned access/privacy 계약(ADR-0002 §10) 아래에서만 전달한다.

## 6. Execution Context Contract (logical fields)

정답 schema가 아니라 **logical field 계약**이다. JSON 이름/형식/cardinality/DTO는 RPL-103 이후 확정한다.

| Field(logical) | 필수성 | 발급자 | 신뢰자 | semantic owner | opaque | 비고 |
| --- | --- | --- | --- | --- | --- | --- |
| `context_version` | Required | Shared AI 계약 | Shared AI | Shared AI | — | context 계약 호환성 축 |
| `product_ref` | Required | Product Backend | Shared AI | Product | — | 누락 시 default 금지, fail-closed |
| `tenant_ref` | Required | Product Backend | Shared AI | Product | 예 | workspace 등. 누락 시 global fallback 금지 |
| `principal_ref` | Required | Product Backend | Shared AI | Product | 예 | identity ref(trace/scope 파생용). 권한 자체 아님 |
| `resource_refs[]` | Conditional | Product Backend | Shared AI | Product | 예 | typed opaque ref. tenant scope 안에 있어야 함 |
| `authority_ref` | Conditional | Product Backend | Shared AI | Product | 예 | 현재 authority 재검증 seam 참조(§9). protected action에 필요 |
| `request_id` | Required | Product Backend | 공통 | Product | — | 논리 요청 식별(멱등). 권한 증명 아님(§13) |
| `execution_id` | Required | Shared AI | 공통 | Shared AI | — | 실행/evidence correlation(ADR-0002 `execution_ref`) |
| `correlation_ref` | Optional | Product/Gateway | 공통 | 발급자 | — | 상위 요청 trace 연결 |

원칙:
- `principal_ref`/`resource_refs`는 opaque weak reference다(ADR-0002 §10, carelog identity contract).
  raw name/treatment text/journal text를 담지 않는다.
- Core가 `resource_refs`의 type namespace를 인지·전파할 수는 있으나 그 domain semantics를 해석하지 않는다.
- `authority_ref`는 "언제·어디에 현재 authority를 물어야 하는가"의 seam reference이며, authorization 자체를
  Shared AI가 계산한다는 뜻이 아니다.

## 7. Scope Model

```text
product     : cross-product 최상위 boundary (Required, fail-closed)
tenant       : product 내 격리 단위(Carelog Workspace, Finance user workspace) (Required, fail-closed)
principal    : 실행을 유발한 identity ref (trace/scope 파생; 권한 아님)
resource     : principal이 접근 대상으로 선언한 typed opaque ref (tenant scope 안이어야 함)
```

- product/tenant는 admission·registration·ingest·index·query·evidence·invalidation·delete·tool·retrieval
  전 경로에서 보존한다(ADR-0002/0003 계승).
- resource scope는 parent tenant scope를 **넓힐 수 없다.** resource가 tenant 밖이면 fail-closed.
- 물리 tenant DB 분리를 요구하지 않는다. logical scope 필드·필터와 future physicalization seam을 구분한다.

## 8. Identity Reference vs Authorization Authority

이 ADR의 핵심. 두 주장을 분리한다.

```text
"I am P1"          = identity assertion  (Shared Identity 인증, Product가 principal_ref로 표현)
"P1 may access C123" = authorization      (Product Backend 결정, query-time authoritative)
```

- Execution Context에 principal_ref/resource_ref가 있다고 permission이 자동 승인되지 않는다.
- Shared AI는 principal id를 보고 Product permission을 자체 추론하지 않는다.
- authority scope는 authenticated principal에서 파생된다. caller가 제시한 tenant/resource 값을 authority
  출처로 신뢰하지 않는다(carelog customer-ownership: "client-supplied workspace를 authority source로
  삼지 않는다").
- Shared AI가 request-time Product context를 신뢰하는 근거는 **호출자가 Product Backend(인증된 authority
  경계)라는 service-to-service 신뢰**이지, context 안 identifier의 자기주장이 아니다. 그 신뢰의 concrete
  mechanism(credential/token)은 OD-5 HOW로 deferred하되, WHAT(=Product Backend가 authority)은 여기서 확정한다.

## 9. Query-time Current Authorization

ADR-0003 invariant를 실행 경계로 확장한다.

- 보호 대상 action/query(retrieval, resource 접근 tool, mutation-adjacent 실행)의 eligibility는
  **query 시점의 현재 authoritative authorization/scope**로 판정한다.
- 실행 시작 시점 context snapshot은 correlation·audit·original identity 기록에 유효하지만, protected
  action 시점의 현재 authority를 **대체하지 않는다.**
- Shared AI는 authz 규칙을 계산하지 않는다. 대신 `authority_ref` seam을 통해 Product-owned 현재 authority를
  참조·집행한다. 이는 Product authz의 재구현이 아니라 "언제 현재값을 확인해야 하는가"의 실행 계약이다.
- revoked permission에서: `old snapshot ≠ current authorization`. 원 context가 남아 있어도 현재 authority가
  거부하면 protected action은 진행되지 않는다.

## 10. Confused Deputy / Authority Consistency

위협: 내부 caller가 `tenant=A`와 `resource=B의 id`를 조합해 Shared AI에 전달.

최소 invariant:
- context 구성요소는 authority-consistent해야 한다. resource_ref는 선언된 tenant scope 안에 있어야 하며,
  불일치하면 거부(fail-closed).
- resource scope는 parent tenant scope를 넓힐 수 없다.
- identifier만으로 permission을 생성할 수 없다.

단, Product business authz 규칙을 Shared AI에 복제하지 않는다. Shared AI는 **일관성/격리 seam**만
집행하고, "P1이 C123을 볼 수 있는가"의 최종 판정은 Product authority(§9)에 남긴다.

## 11. Execute Request Boundary (conceptual)

```text
ExecuteRequest
├── context        (§6 Execution Context)
├── skill_ref      (Product가 선택한 Skill/use-case; ADR-0001 explicit selection)
├── input          (Business Input; 필요 최소 payload)
└── correlation    (request_id, correlation_ref)
```

- Product가 `skill_ref`를 명시한다. free-text만 받아 Shared AI가 Skill을 임의 선택하지 않는다(ADR-0001 §6 A).
- input은 domain payload이며 context와 섞지 않는다.
- 실제 wire schema/endpoint는 후속 결정이다.

## 12. Execute Response Boundary (conceptual)

```text
ExecuteResponse
├── result         (Product가 검증·사용할 실행 결과; 최종 책임은 Product)
├── evidence_refs  (ADR-0002 Evidence envelope 참조; 재정의하지 않음)
├── warnings       (technical warning / partial 상태)
└── execution_ref  (correlation/provenance)
```

- Evidence envelope를 재정의하지 않는다(ADR-0002). response는 evidence를 **참조**한다.
- result는 Product 응답 schema가 아니다. Product가 최종 result/response contract를 소유한다(ADR-0002 §9).
- 실제 JSON/status code는 후속 결정이다.

## 13. Error Model

generic "AI_ERROR"로 합치지 않고 최소 semantic category를 구분한다(enum·HTTP code는 후속).

| category | 성격 | owner 판정 |
| --- | --- | --- |
| malformed_context | 필수 scope 부재·형식 오류 | Shared AI 기술 거부(fail-closed) |
| unauthorized_or_unverifiable_scope | scope 불일치·현재 authority 확인 불가 | Shared AI fail-closed; 최종 authz 의미는 Product |
| execution_technical_failure | adapter/model/runtime 실패 | Shared AI technical 실패 |
| retrieval_unavailable | projection/index 비가용 | Shared AI(ADR-0003 §12) |
| domain_validation_failure | Product/Domain validator 거부 | Product/Domain |
| insufficient_evidence | domain 기준 근거 부족 | Product/Domain(uncertainty) |

`request_id`는 same logical request 식별용이며 idempotency key는 authorization proof가 아니다(§17).

## 14. Fail-closed Isolation

다음에서 Shared AI는 "일단 실행"하지 않고 fail-closed 한다.

```text
- product 누락 / tenant 누락
- resource가 tenant scope 밖
- context 내부 불일치(authority-inconsistent)
- protected action에서 현재 authority를 확인할 수 없음
- stale authorization context를 protected action에 재사용하려는 경우
```

경계 분리(중요):
- 위는 **security boundary**로서 fail-closed다(foundation ADR-0015 §7.2와 정렬). 이 부분이 OD-7의
  security-boundary 몫이며 여기서 확정한다.
- 반면 "Identity/provider가 다운됐을 때 read를 어디까지 허용하는가" 같은 **domain 가용성 fallback**은
  Product 소유다(Carelog authenticated-context의 미승인 follow-up, OD-7 domain 몫). 이 ADR은 이를
  결정하지 않으며 bypass/grace/silent activation을 추론하지 않는다. 모든 downstream 장애를 무조건
  fail-closed로 확대하지도 않는다.

## 15. Sync Execution

```text
Product Backend (인증·인가 → authoritative context 구성)
→ Shared AI execute(context, skill_ref, input)
→ context 무결성/일관성 검증 (fail-closed)
→ capability invocation (Tool/Retriever/Lens; child scope ≤ parent)
→ protected action에서 query-time authority seam
→ result + evidence_refs
→ Product Backend 최종 validation·저장·책임
```

HTTP/framework는 결정하지 않는다(ADR-0015 §7.1 HTTP/JSON 기본값은 유지).

## 16. Async Execution

logical async job만 검토한다(NATS/queue는 RPL-103).

```text
Product request at T0 → queued → executed at T1
```

- T0 authorization snapshot만으로 T1 protected action을 허용하지 않는다.
- T0~T1 사이 권한이 revoke되면, T1의 protected action은 query-time 현재 authority(§9)에서 거부되어야 한다.

원칙:
- original context는 correlation/audit용으로 보존한다.
- protected action/query 시점의 현재 authorization 재검증은 여전히 필요하다.
- old context snapshot이 독립적으로 접근을 재부여할 수 없다.

## 17. Retry / Resume / Idempotency

- retry/resume에서 business request identity와 authorization freshness를 분리한다.
- `execution_id`/idempotency key 재사용이 과거 authorization의 자동 재사용을 의미하지 않는다.
- idempotency key는 same logical request 식별용이며 authorization proof가 아니다. "같은 request_id니까
  예전 권한으로 재실행"은 금지.
- 실제 idempotency store/구현은 후속 결정이다.

## 18. Child Capability Scope

Skill → Tool/Retriever/Lens 실행 중 하위 capability의 scope 규칙:

```text
child scope ⊆ parent execution context scope   (기본, 강제)
implicit scope widening 금지
widening이 실제로 필요하면 → Product-owned explicit decision (같은 seam으로 새 authority 판정)
```

- Tool 호출도 parent execution context를 넓힐 수 없다. Shared AI가 resource_ref를 임의 조합해 Product
  Tool에서 다른 tenant 데이터를 읽는 구조를 금지한다.
- Retriever는 context를 technical filtering/enforcement seam으로 쓰되, Product permission semantics는
  Product에 남긴다(ADR-0003). projection stored scope를 current authorization과 혼동하지 않는다.

## 19. Evidence / Retrieval / Tool Integration

- Evidence `scope_ref`(ADR-0002)는 execution context scope와 정합해야 한다. **Evidence가 execution
  context보다 넓은 scope를 가질 수 없다.** Evidence는 authorization proof가 아니라 scope/correlation
  reference다.
- Retrieval eligibility는 current authority(ADR-0003) 기준이며, context가 product/tenant/resource ref를
  운반하더라도 projection metadata scope를 authorization으로 승격하지 않는다.
- Tool invocation: Tool 의미·권한은 Product, generic invocation은 Shared AI. child scope 비확대(§18) 적용.
- ADR-0002 Evidence schema와 ADR-0003 retrieval lifecycle을 재정의하지 않는다.

## 20. Sensitive Data Minimization

- Execution Context는 metadata/control-plane 성격이다. 기본적으로 treatment note·customer name·medical
  detail·journal contents·conversation contents·full document를 담지 않는다.
- opaque ref 중심으로 구성한다. Business Input에 payload가 필요해도 Context와 분리하고 Product-owned
  access/privacy 계약을 적용한다(ADR-0002 §10, finance exposure policy).

## 21. Provider-neutral Contract & Generic Execute vs Product Endpoint

- Canonical 계약에 OpenAI/Anthropic message type, LangChain Runnable, LangGraph State, Pydantic-specific
  type, NATS header 같은 provider/framework type을 넣지 않는다. Shared AI 자체 logical 계약을 먼저 두고
  adapter가 framework/provider를 감싼다.
- Shared AI는 **generic execute** 계약을 제공한다. Product-specific use-case endpoint는 Product Backend가
  소유한다. 예: `Carelog Backend: POST /customers/{id}/treatment-prep` 내부에서 `execute(skill=Treatment
  Preparation)` 호출.
- Shared AI 자체에 `/carelog/...`·`/finance/...` 같은 Product API를 무분별하게 만들지 않되, 모든 것을 하나의
  untyped `/ai`로 뭉개지도 않는다(typed generic execute + Product-owned endpoint).

## 22. Context Versioning

축을 분리한다(migration framework는 만들지 않는다).

```text
Execution Context Contract Version = Shared AI context 계약 호환성 (context_version)
Skill Semantic Version             = Product/Domain Skill 정의 버전
Business Input Version             = Product domain input 계약 버전
Evidence Envelope Version          = ADR-0002 core version
```

- 모두 하나의 "API version"으로 묶지 않는다.
- context 계약 변경은 Skill/Input/Evidence semantic version bump를 자동 요구하지 않는다(그 역도 성립).

## 23. Threat Matrix

| # | Threat | Invariant | semantic owner | execution enforcement |
| --- | --- | --- | --- | --- |
| 1 | missing tenant | tenant 필수, global fallback 금지 | Product(scope 의미) | Shared AI fail-closed |
| 2 | spoofed tenant | authority는 authenticated principal 파생, caller 제시값 신뢰 안 함 | Product authority | Shared AI + Product authority seam |
| 3 | valid principal + wrong resource | resource ∈ tenant scope, 현재 authority 판정 | Product | Shared AI 일관성 + query-time seam |
| 4 | Product A context + Product B Skill | product boundary fail-closed | Product | Shared AI |
| 5 | revoked permission after queueing | query-time current authority(§9/§16) | Product | Shared AI seam 재검증 |
| 6 | retry with stale auth snapshot | snapshot ≠ authorization(§17) | Product | Shared AI |
| 7 | child Tool widens scope | child ⊆ parent(§18) | Product(widening 결정) | Shared AI 비확대 집행 |
| 8 | retrieval projection stale scope | projection scope ≠ current authz(ADR-0003) | Product | Shared AI query-time |
| 9 | Evidence from different tenant attached | Evidence scope ≤ execution scope(§19) | Product | Shared AI correlation 검증 |
| 10 | caller sends raw sensitive metadata | context data minimization(§20) | Product/Domain privacy | Shared AI 기본 미복제 |

exploit/protocol은 설계하지 않는다.

## 24. Ownership Matrix

| Concern | Product Backend | Shared AI | Shared Identity/Other |
| --- | --- | --- | --- |
| Authentication meaning | 인증 결과 소비 | context 신뢰 경계 | **Shared Identity 인증 발급** |
| Domain authorization | **소유(query-time authoritative)** | 판단 안 함, seam 집행 | — |
| Product scope | scope 의미 선언 | fail-closed 집행 | — |
| Tenant scope | 소유 | 전파·격리 집행 | — |
| Principal ref | authenticated principal 해석 | opaque 전파 | Identity가 stable id 발급 |
| Resource ref | semantic ownership | opaque 전파·일관성 검증 | — |
| Context contract | authoritative 구성 | **계약·검증 소유** | — |
| Context validation(무결성) | — | **소유(fail-closed)** | Gateway가 외부 header 무결성 선처리 |
| Skill selection | **소유(explicit)** | 실행만 | — |
| Tool/Retriever invocation | 의미·권한 | generic invocation·scope 비확대 | — |
| Current authz check seam | **authority 제공** | 참조·집행 | — |
| Evidence correlation | admissibility·validation | envelope·correlation(ADR-0002) | — |
| Final validation | **소유·최종 책임** | technical 결과 제공 | — |

Shared Identity를 Product authorization engine으로 만들지 않는다(identity-token-contract: permission은
기본 claim 금지).

## 25. Consumer Validation

### Carelog Therapy — PASS
- Carelog user 인증 → Workspace(W1) tenant → Customer(C123) resource authorization = Carelog.
- Product가 `TreatmentPreparation` skill_ref 명시 → Shared AI execute.
- customer id 단독 ≠ auth; workspace/customer mismatch → fail-closed(§10/§14).
- Treatment Retriever·Tool·Lens는 child scope ≤ parent(§18); query-time 현재 authz 유지(§9).
- raw customer/treatment payload는 Context metadata에 넣지 않음(§20). **PASS.**

### Carelog Rental — PASS
- landlord/workspace A 요청이 workspace B의 payment/conversation에 접근 불가(§7/§14).
- payment fact = Product Tool(structured_fact), conversation = Retriever(retrieved_content).
- Execution Context가 두 source를 **동일 authorized resource domain** 안에서 correlation.
- delete/permission change 후 old scope retrieval 금지(ADR-0003 계승). **PASS.**

### Finance — PASS
- Finance user/journal context → explicit Finance Skill(`InvestmentReview`) → Tool/Retriever/Lens → Evidence
  → Finance Validator.
- Finance context semantics(routing/policy/lens/admissibility/point-in-time/uncertainty)를 Shared AI가
  해석·소유하지 않음.
- private journal/raw-restricted knowledge가 global metadata/context로 이동하지 않음(§20, finance exposure
  policy). Shared AI는 실행·격리·correlation seam만 제공. **PASS.**

## 26. RPL-103 Handoff Criteria

RPL-103 Runtime Technology ADR가 이 ownership을 보존하며 선택할 수 있도록 넘기는 technical 기준:

```text
- typed context validation과 immutable execution context representation
- product/tenant/resource scope propagation을 모든 lifecycle operation에서 강제
- sync/async 실행 호환
- protected action에서 current authority revalidation seam
- child-scope 비확대 집행
- trace/evidence correlation(ADR-0002)
- provider/framework-neutral DTO boundary
- malformed/inconsistent context의 fail-closed 검증
```

이는 평가 기준이며 FastAPI/Pydantic/NATS/JWT/JWKS/OAuth 선택이 아니다.

## 27. Consequences

**긍정**
- Product authorization SoT를 침범하지 않으면서 cross-tenant/cross-resource isolation과 stale-auth 재사용
  방지를 실행 경계 invariant로 고정한다.
- ADR-0001~0003(Skill selection, Evidence scope, retrieval query-time authority)이 하나의 실행 계약으로 연결된다.
- generic execute + Product endpoint 경계로 Product별 if/else가 Core에 쌓이지 않는다.

**비용 / 트레이드오프**
- Product Backend는 authoritative context 구성과 query-time authority seam을 제공해야 한다(초기 계약 비용).
- protected action의 query-time 재검증은 async/long-running에서 추가 왕복·지연을 만들 수 있다(안전과의 교환).
- reference-first/data-minimization은 실제 payload가 필요한 경우 Product access 계약과의 조율을 요구한다.
- security-boundary fail-closed와 domain 가용성 fallback을 분리해 관리하는 규율이 필요하다.

## 28. Canonical Source Reality

| Repository | Canonical branch | fresh `origin/main` SHA |
| --- | --- | --- |
| `shared-ai-architecture` | `main` | `a23aa059901682dbf84ddd0e201524306866ad1f` |
| `harness-foundation-docs` | `main` | `69fceb267094756913f400f8b4045590bd0e8ac5` |
| `finance-harness-docs` | `main` | `5326977691b420f573a469c0bb7635de26fe2aba` |
| `carelog-architecture` | `main` | `6b581dd3cc92d5e8486172525898d674e8d61b7c` |

SHA는 재현 anchor이며 영구 invariant가 아니다. feature/review checkout은 canonical source로 사용하지 않았다.

## 29. Rejected Alternatives

- **caller-provided IDs를 그대로 authorization으로 신뢰** — confused deputy·cross-tenant 노출. carelog
  customer-ownership("identifier만으로 권한 아님")과 충돌. 기각.
- **Product authz logic을 Shared AI에 복제** — Product authorization ownership 침범, 규칙 이중 소유. 기각.
- **first-request auth snapshot을 async/retry에서 영구 신뢰** — ADR-0003 query-time invariant·revoke
  안전과 충돌. 기각.
- **tenant 없으면 default/global tenant 사용** — tenant isolation 붕괴. 기각.
- **child capability 암묵적 scope 확대** — privilege escalation seam. 기각.
- **raw business payload를 Execution Context에 복제** — data minimization·privacy 위반. 기각.
- **provider SDK type을 public 계약으로 사용** — ownership을 framework/provider에 결합. 기각.
- **free-text만 받고 Shared AI가 Product Skill 자동 결정** — ADR-0001 explicit selection(A)·고위험 domain
  안전과 충돌. 기각.
- **Evidence scope를 authorization proof로 사용** — Evidence는 scope/correlation reference이지 권한 증명
  아님(§19). 기각.

## 30. Non-goals와 Deferred Technology Decisions

```text
- JWT/JWKS/OAuth 형식, issuer/audience/claim schema 확정
- Shared Identity/Gateway/FastAPI/Pydantic DTO/NATS message schema 구현
- authz cache / idempotency store / database schema 구현
- Product의 Identity/provider 비가용 시 domain 가용성 fallback(OD-7 domain 몫; Carelog ADR-C03 등 Product 소유)
- concrete service-to-service credential mechanism(OD-5 HOW; RPL-103)
- HTTP status code/error enum 전체 표준화
- ADR-0002 Evidence schema / ADR-0003 retrieval lifecycle 재정의
- Finance routing/policy/lens/validator semantics, Carelog Customer/CustomerEvent/privacy 규칙 변경
- Product backend 코드 변경, RPL-103 runtime technology 결정
```

## 31. Open Decision Disposition

| # | 이전 상태 | 이 ADR 처리 |
| --- | --- | --- |
| OD-5 | deferred (Shared AI↔Product Tool 인증) | **부분 확정** — 신뢰/context 계약(WHAT: Product Backend=authority, context 무결성 검증)은 확정. concrete credential mechanism(HOW)은 RPL-103 deferred |
| OD-7 | NEEDS_ADR (fail-closed/fail-open) | **security-boundary 부분 확정** — Shared AI context 검증 fail-closed는 확정(§14). Product/Identity 가용성 domain fallback은 Product 소유로 유지(미해소) |

## 32. Related Canonical Sources

- `harness-foundation-docs/docs/adr/ADR-0015-platform-communication-messaging-scaling.md` §7.2, §7.3, §7.4
- `harness-foundation-docs/docs/contracts/backend-service-foundation/identity-token-contract.md`
- `harness-foundation-docs/docs/architecture/backend-service-foundation/database-ownership-and-reference-policy.md`
- `harness-foundation-docs/docs/adr/ADR-0013-target-deployment-and-data-boundaries.md`
- `carelog-architecture/contracts/authenticated-carelog-context.md`
- `carelog-architecture/contracts/customer-ownership.md`
- `carelog-architecture/architecture/ownership-map.md`
- `finance-harness-docs/00-core/0-c-runtime-contract/finance-routing-contract.md`
- `finance-harness-docs/00-core/0-a-safety-core/finance-market-data-grounding.md`
- `finance-harness-docs/00-core/0-d-knowledge-architecture/finance-document-exposure-policy.md`
- 본 repo: [ADR-0001](ADR-0001-skill-lens-workflow-execution-ownership.md), [ADR-0002](ADR-0002-evidence-provenance-core-domain-boundary.md), [ADR-0003](ADR-0003-retrieval-knowledge-runtime-ownership-lifecycle.md)
