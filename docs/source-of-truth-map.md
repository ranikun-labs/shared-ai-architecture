# Source-of-Truth Map

| 항목 | 값 |
| --- | --- |
| status | draft (RPL-97 Foundation) |
| owner | shared-ai-architecture |
| purpose | Shared AI 관련 개념별 canonical owner를 고정하고, 기존 결정과의 충돌을 분류 |

---

## 1. Purpose

Shared AI 설계가 상위 canonical source의 기존 결정을 조용히 재해석·덮어쓰지 않도록,
개념별 **canonical owner**와 **충돌 상태**를 명시한다. 이 문서는 판정만 하고, 실제 규칙
원문은 각 owner repository가 소유한다.

## 2. Canonical source reality (RPL-97 preflight, fresh 확인)

| Repository | Canonical branch | 이번 preflight에서 읽은 SHA | 참고 |
| --- | --- | --- | --- |
| `harness-foundation-docs` | `main` | `origin/main` = `69fceb26` | local checkout은 `docs/RPL-72-g1-runtime-authority`(`449c0232`)였고 read 대상 파일 tree는 origin/main과 동일 |
| `carelog-architecture` | `main` | `6b581dd3` (local=remote 정렬) | clean |
| `finance-harness-docs` | `main` | `origin/main` = `53269776` | local checkout은 `docs/service-policy-foundation`(`0cbed02b`, 20 ahead/4 behind); read한 00-core lens/routing/knowledge 파일 tree는 origin/main과 동일 |
| `shared-ai-architecture` | `main` | bootstrap 이전 empty remote (ls-remote 무결과) | 이번 세션에서 root commit 생성 |

> SHA는 재현 anchor이며 영구 business invariant가 아니다. finance/foundation의 feature branch는
> 이번 작업의 canonical source가 아니다 — canonical은 각 repo의 `main`이다.

## 3. Existing decision inventory (영역별)

### Platform-owned (`harness-foundation-docs`)

```text
- "AI Runtime Server"는 목표 Deployment Unit 중 하나 (ADR-0013, boundaries §9)
- Shared AI ↔ Product 책임 경계 (ADR-0015 §7.4): 아래 §4 표의 근거
- AI Token Streaming = SSE, flow: Shared AI → Product → Gateway → Client (ADR-0015 §7.5)
- 비동기 AI Job = NATS JetStream, 첫 use case 발생 후 도입 (ADR-0015 §7.6, deferred)
- 내부 통신 기본값 HTTP/JSON, 내부 호출은 Gateway 우회 안 함 (ADR-0015 §7.1)
- 각 Service가 자기 DB 소유, cross-service DB 직접 접근·FK·JOIN 금지 (ADR-0015 §7.8)
- ai_runtime_db = AI Runtime Server 소유 논리 DB (boundaries §10.1)
- Shared AI는 platform-services 밖 future independent Python Runtime (boundaries §13.3)
- Shared AI physical runtime = deferred / not_started (master §22)
- Candidate before promotion, Cloud/AI 결과는 자동 Truth 아님 (master §6.4~6.5)
```

### Carelog-owned (`carelog-architecture`)

```text
- Workspace / Product Membership / Customer ownership / CustomerEvent 의미·시간 불변식
- Product authorization, cross-workspace 금지
- identityUserId는 weak reference (Identity DB 직접 접근·물리 FK 금지)
- CRM Customer != Identity User
- Shared AI provider/model/DTO는 이 identity contract 밖 = Deferred
```

### Finance-owned (`finance-harness-docs`)

```text
- Lens semantics / 본문 / catalog / lens_tier / trigger
- PolicyGuard 규칙(intent 12종, guard action 7종) 원문
- Routing Contract (Input Guard → intent → action → lens/redirect → template → validator)
- Output Validator 규칙, Prompt Template 원문
- Market Data Grounding, Professional Standards, Human Review Protocol
- Knowledge Artifact 저장 결정: Markdown authoring SoT + WAS private resource,
  MVP에서 PostgreSQL 미도입 (knowledge-storage-decision)
- Lens activation gate / runtime_loadable / routable / fixture_refs 의미
```

### Potential Shared-AI-owned (RPL-97 제안)

```text
- Model Gateway (provider adapter, key, alias, timeout/retry, rate limit, usage/cost, trace)
- Tool orchestration 실행 계약 (Product Tool/API adapter 호출 규약)
- Retriever 실행 계약 + tenant-scoped RAG substrate (candidate)
- Skill/Lens 실행 계약 (interface만; 정의·semantics는 Product/Finance)
- Evidence / Provenance envelope
- Tenant scope 전파
```

## 4. Ownership / conflict matrix

| Concept | Existing owner | Proposed owner | Conflict | Action |
| --- | --- | --- | --- | --- |
| Provider 실행·key·model alias·timeout·rate limit·usage·trace | Shared AI (ADR-0015 §7.4) | Shared AI | No | Keep |
| AI Token Streaming transport (SSE) | Platform (ADR-0015 §7.5) | Platform 유지, Shared AI는 기술 구현 | No | Preserve |
| 제품별 System Prompt / Workflow / Domain Policy | Product (ADR-0015 §7.4) | Product | No | Preserve |
| Finance Lens **semantics** | Finance | Finance | No | Keep |
| Finance PolicyGuard / Output Validator / Routing **규칙** | Finance | Finance | No | Keep |
| Lens/Skill **실행 계약**(interface) | Finance 자체 runtime contract 보유; Platform은 "공통 AI Orchestration"을 candidate로만 인정 | Shared AI (실행 mechanism) | Resolved | **RESOLVED — [ADR-0001](adr/ADR-0001-skill-lens-workflow-execution-ownership.md)** (실행 mechanism=Shared AI, 정의·선택·검증=Product/Domain; Finance는 병존+opt-in) |
| Retriever / RAG · Embedding · Vector infra | Platform이 candidate로만 인정 (ADR-0015 §10 follow-up) | Shared AI Core (tenant-scoped) | Possible | **NEEDS_ADR** — 소유·범위·저장 결정 |
| Finance **Knowledge Artifact** 저장(Lens/PolicyGuard 문서) | Finance (Markdown+WAS, no DB) | Finance 유지 | No | Preserve — RAG corpus와 구분 |
| Product structured 사실(미납·예약·통증 score·journal metadata) | Product Backend / DB | Product (Shared AI는 Tool로 소비) | No | Preserve — direct DB 금지 |
| Evidence contract | Finance provenance 일부 존재 | Shared AI Core envelope + Domain extension | Possible | **NEEDS_ADR** — core/domain 경계 정의 |
| Tenant/identity context | Platform(Identity) + Product(authz) | Shared AI는 scope 전파만 | No | Preserve — Shared AI는 authz 판단 안 함 |
| Product DB ownership | Product/Foundation | Product | No | Preserve |
| Shared AI physical runtime 위치/시점 | Platform (deferred) | Platform 유지 | No | Defer |

충돌 분류 요약:

```text
NO_CONFLICT              : Provider 실행 경계, Prompt/Workflow/Policy=Product,
                          Lens semantics=Finance, Product SSOT direct-DB 금지,
                          tenant scope=전파만, physical runtime=deferred
RESOLVED (ADR-0001)     : Lens/Skill 실행 계약 소유(OD-1), Finance runtime contract 관계(OD-4)
NEEDS_ADR               : Retriever/RAG·Vector 소유(OD-2/OD-6), Evidence core/domain 경계(OD-8)
NEEDS_OWNER_REPO_UPDATE : (이번 Slice 없음 — 기존 owner 문서 수정 불필요)
BLOCKING_CONFLICT       : 없음
```

## 5. 재해석 방지 규칙

1. 이 repo의 어떤 문서도 Finance Lens 판단 내용, Carelog business rule, Platform 통신/배포/데이터
   결정을 **복사하거나 재정의하지 않는다.** 필요하면 owner repo의 canonical path를 참조한다.
2. `NEEDS_ADR` 항목은 이 repo가 단독으로 확정하지 않는다. 독립 Architecture Review + 해당 owner repo
   합의로 ADR을 만든 뒤 확정한다.
3. 상위 canonical과 이 repo 문서가 충돌하면 상위 canonical이 이긴다
   (`ADR-0015` > boundaries/master > 이 repo).

## 6. Related canonical sources

- `harness-foundation-docs/docs/adr/ADR-0015-...` §7.1, §7.4~7.6, §7.8, §10
- `harness-foundation-docs/docs/adr/ADR-0013-...`
- `harness-foundation-docs/docs/architecture/repository-service-boundaries.md` §9, §10, §13.3
- `carelog-architecture/architecture/ownership-map.md`, `contracts/authenticated-carelog-context.md`
- `finance-harness-docs/00-core/0-d-knowledge-architecture/finance-knowledge-storage-decision.md`
