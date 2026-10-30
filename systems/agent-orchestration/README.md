# GOPS 에이전트 오케스트레이션

`agent-orchestration`은 v1 주식 분석 에이전트의 실행을 담당합니다. 오케스트레이션은 여러 분석 역할의 실행 순서와 결과 결합을 조율하는 작업입니다. 분석 리포트·화면 배치 제안·시장 이벤트 해설·알림 여부를 생성하며, 주문 실행이나 계좌 제어는 호출하지 않습니다.

인수인계 시 `docs/AGENT_ARCHITECTURE.md`를 먼저 읽고 연동 대상에 따라 `docs/AGENT_BACKEND_INTEGRATION.md`, `docs/AGENT_FRONTEND_INTEGRATION.md`, `docs/AGENT_AWS_BUILD.md`를 확인합니다. 이 문서는 코드 책임과 실행 명령을 정리합니다.

## 실행 구성 요소

| 구성 요소 | 실행 단위 | 필수 여부 | 역할 |
| --- | --- | --- | --- |
| `agent-orchestrator` | Pod | 필수 | 직접 분석·리포트 조회용 HTTP 호환 진입점 |
| `agent-analysis-worker` | Pod | 필수 | 대기열의 빠른 분석 요청을 처리하고 리포트 저장 |
| `agent-delivery-gateway` | Pod | 비동기·SSE에서 필수 | 결과 이벤트를 리포트 저장소와 Redis 갱신 채널에 반영 |
| `agent-intent-classifier` | Pod | 선택 | 여러 의도가 섞인 모호한 요청을 저비용으로 분류 |
| `deep-analysis-worker` | Pod | 선택 | 빠른 분석 대기열 밖에서 사용자가 선택한 심층 후속 분석 처리 |
| `event-detector` | Pod | 선택 | 시장 토픽에서 이상 이벤트 감지·발행 |
| `notification-publisher` | Pod | 선택 | 알림 결정을 Redis·WebSocket 구독자에게 전달 |
| `graph-expansion-refresh` | Job | 선택 | GraphDB 관계 힌트를 Redis·ClickHouse 캐시에 구체화 |
| 기본 동작·평가 작업 | Job | 선택 | 대기열·저장소·그래프·지연·분배·검색·근거 일치 검증 |

Pod는 Kubernetes에서 컨테이너를 실행하는 기본 단위이며, Job은 한 번의 작업 완료를 목표로 실행하는 단위입니다. 모두 `gops-agent-orchestrator` 이미지를 사용합니다. SSE는 서버가 브라우저로 갱신 내용을 연속 전송하는 방식입니다.

## 패키지 구성

```text
config/                 화면 용어 사전과 대체 종목 별칭
shared/gops_agents/
  contracts/            리포트·근거·경로·스냅샷·런타임 데이터 계약
  query_understanding/  한국어 중심 기업·테마 해석
  intent_understanding/ 내용·화면 조작 의도 분리와 분류기 어댑터
  orchestration/        흐름·라우팅·시간 측정·캐시·추적
  runtime/              요청 수용 제어·대기열·작업자·리포트 저장·전달
  retrieval/            관계 확장·스냅샷·교차 신호·장애 격리
  providers/            뉴스·온톨로지·거시 지표 어댑터와 캐시
  roles/                역할별 에이전트와 AgentContext
  synthesis/            최종 답변 종합
  events/               시장 이벤트 감지와 알림 발행
pods/                   실행 진입점
jobs/                   기본 동작 점검·갱신·성능 측정·평가 진입점
tests/                  단위·계약 테스트
```

`shared/gops_agents/orchestrator.py`는 기존 가져오기 경로를 유지하는 호환 파일입니다. 실제 흐름은 `shared/gops_agents/orchestration/`에 있습니다.

## 요청 처리 흐름

```text
POST /api/agents/analyze
  → AgentAnalysisRequestEnvelope
  → agents.analysis-requests.v1
  → agent-analysis-worker
  → AgentOrchestrator
  → 요청 해석 + 입력 스냅샷 + 답변 종합
  → 리포트 저장소
  → agents.analysis-results.v1
  → agent-delivery-gateway
  → Redis 리포트 갱신 채널
```

`AGENT_ASYNC_ANALYSIS_ENABLED=false`이면 백엔드가 호환 경로로 `agent-orchestrator`를 HTTP 호출할 수 있습니다.

## 요청 해석

빠른 응답 경로에서는 다음 작업을 제한된 수로 병렬 실행합니다.

- 한국어 기업·테마 해석
- 규칙 기반 분석 작업 분류
- 규칙 기반 화면 조작 작업 분류
- 선택적인 분류기 Pod 또는 OpenAI 분류기 호출

통합된 실행 모드는 다음 중 하나입니다.

```text
analysis
ui_layout
hybrid
clarify
```

각각 분석, 화면 배치, 두 작업의 결합, 추가 확인 요청을 뜻합니다. 기업·테마는 카탈로그로 해석합니다. `config/entity-aliases.seed.json`과 초기 상수는 시작 시 대체 자료이며 운영의 기준 원본이 아닙니다.

## 데이터 제공자 상태

- 뉴스는 Redis·ClickHouse에 사전 저장한 분석 결과를 사용합니다. 명시적으로 켜지 않으면 Alpaca 직접 조회를 사용하지 않습니다.
- 온톨로지(기업·개념 사이의 관계 모델)는 프로젝트의 `GraphDBOntologyProvider`가 관계 스냅샷으로 제공합니다.
- 거시 지표 제공자 어댑터는 v1에서 의도적으로 비어 있습니다.
- 관계 기반 검색을 켜면 Redis를 먼저, ClickHouse를 다음으로 읽습니다.
- 교차 신호 종합은 기능 설정으로 제어합니다.

Kafka 입출력·ClickHouse/Redis 조회·뉴스 정규화·선택적 대체 경로에는 `systems/market-data/shared`의 `alfaka.*`를 사용합니다.

## Kafka 토픽

토픽은 Kafka가 메시지를 종류별로 저장·전달하는 통로입니다.

```text
agents.market-events.v1
agents.analysis-requests.v1
agents.deep-analysis-requests.v1
agents.analysis-results.v1
agents.query-understanding-events.v1
agents.notification-decisions.v1
agents.dlq.v1
```

## 리포트 저장

실행 리포트는 프로젝트 저장소 추상화인 `ReportStore`를 사용합니다. 기본 Redis 키·채널은 다음과 같습니다.

```text
agent:report:{analysisId}
agent:report:latest:{SYMBOL}
agent:report:latest
agent:request:idempotency:{userHash}:{keyHash}
agent.reports
agent.reports:{analysisId}
```

Redis 저장 실패가 최신 리포트 보관 때문에 분석 생성까지 차단해서는 안 됩니다. v1 에이전트 런타임은 장기 PostgreSQL 이력을 포함하지 않습니다.

## 주요 환경 변수

```text
AGENT_ASYNC_ANALYSIS_ENABLED
AGENT_SHARED_REPORT_STORE_ENABLED
AGENT_ANALYSIS_QUEUE_BACKEND
AGENT_REPORT_STORE_BACKEND
AGENT_REPORT_TTL_SECONDS
AGENT_REPORT_STREAM_REDIS_ENABLED
AGENT_ANALYSIS_REQUESTS_TOPIC
AGENT_DEEP_ANALYSIS_REQUESTS_TOPIC
AGENT_ANALYSIS_RESULTS_TOPIC
AGENT_QUERY_UNDERSTANDING_EVENTS_TOPIC
AGENT_NOTIFICATION_DECISIONS_TOPIC
AGENT_MARKET_EVENTS_TOPIC
AGENT_DLQ_TOPIC
AGENT_DEEP_ANALYSIS_ENABLED
KAFKA_BOOTSTRAP_SERVERS
REDIS_URL
CLICKHOUSE_HTTP_URL
GRAPHDB_SPARQL_URL
AGENT_GRAPH_PATH_CACHE_BACKEND
AGENT_GRAPH_PATH_CACHE_TTL_SECONDS
AGENT_GRAPH_PATH_CACHE_NO_DATA_TTL_SECONDS
AGENT_GRAPH_PATH_CACHE_KEY_PREFIX
OPENAI_API_KEY
```

배포·환경·플랫폼 계약은 `docs/AGENT_AWS_BUILD.md`, 실행 계약은 `docs/AGENT_ARCHITECTURE.md`를 참고합니다.

## 검증

단위 테스트:

```sh
.venv/bin/python -m unittest discover -s systems/agent-orchestration/tests -p 'test_*.py'
.venv/bin/python -m unittest discover -s systems/fundamentals/tests -p 'test_*.py'
```

백엔드 연결 테스트:

```sh
.venv/bin/python -m unittest discover -s systems/api-server/tests -p 'test_agent_routes.py'
```

기본 동작 점검 작업:

```sh
PYTHONPATH=systems/agent-orchestration/shared:systems/market-data/shared \
  .venv/bin/python systems/agent-orchestration/jobs/agent-queue-smoke/main.py

PYTHONPATH=systems/agent-orchestration/shared:systems/market-data/shared \
  .venv/bin/python systems/agent-orchestration/jobs/report-store-smoke/main.py

PYTHONPATH=systems/agent-orchestration/shared:systems/market-data/shared \
  .venv/bin/python systems/agent-orchestration/jobs/graph-expansion-smoke/main.py
```

AWS 인수 검증에서는 다음 비동기 왕복 흐름을 모두 확인합니다.

```text
API 202 → Kafka 요청 → 작업자 리포트 → Redis 저장 → 리포트 조회 → SSE 갱신
```
