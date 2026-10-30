# 시장 데이터 시스템

Alpaca 시장 데이터 수집, 스트림 처리, 저장, 요청 시 누락 데이터 보충과 조회 도우미를 담당합니다. 스트림 처리는 계속 들어오는 개별 이벤트를 순서대로 가공하는 방식입니다.

차트 데이터 기준은 `docs/CHART_DATA_ARCHITECTURE.md`, 운영 절차는 `docs/CHART_DATA_OPERATIONS.md`입니다. 저장·전달 규약의 세부 값은 각 플랫폼 README에 있습니다.

## 폴더 구성

```text
pods/market-ingestor/          Alpaca 실시간 WebSocket 수집 진입점
pods/market-processor/         로컬·AWS 공용 Python 스트림 처리기
pods/s3-sink/                  Kafka 원본·가공 토픽의 S3 저장
pods/clickhouse-loader/        Kafka 가공 토픽의 ClickHouse 적재
pods/news-intelligence-worker/ 한국어 뉴스 요약·관련도 사전 계산
jobs/symbol-registry-sync/     종목 메타데이터 동기화
jobs/coverage-repair/          차트 데이터 범위·보충 이력 점검
jobs/news-backfill/            Alpaca 과거 뉴스 수집·원본 백업
jobs/news-intelligence-rebuild/ 한국어 뉴스 관련도 v2 재계산
config/                       수집 종목군·구독 정책
shared/alfaka/                시장 데이터 공용 패키지
tests/                       시장 데이터 테스트
```

## 실행 진입점

| 경로 | 역할 |
| --- | --- |
| `pods/market-ingestor/market_stream.py` | `alfaka.alpaca.websocket_collector` 실행 |
| `pods/market-processor/local_main.py` | `alfaka.streaming.processor` 실행 |
| `infra/k8s/base/app/deployment-market-processor.yaml` | 현재 Kubernetes 처리기 배포 정의 |
| `pods/s3-sink/processed_sink.py` | `alfaka.storage.processed_s3_sink` 실행 |
| `pods/s3-sink/raw_archive_sink.py` | `alfaka.storage.raw_s3_archive_sink` 실행 |
| `pods/clickhouse-loader/processed_loader.py` | `alfaka.storage.clickhouse_loader` 실행 |
| `pods/news-intelligence-worker/main.py` | 한국어 뉴스 분석 결과 사전 계산 |
| `jobs/symbol-registry-sync/main.py` | `alfaka.tools.sync_symbol_registry` 실행 |
| `jobs/coverage-repair/main.py` | `/api/charts/candles`의 보충 이력 점검 |
| `jobs/news-backfill/main.py` | `articleId`별 Alpaca 뉴스 원본을 한 번 저장 |
| `jobs/news-intelligence-rebuild/main.py` | 최근 한국어 뉴스의 관련도 v2 필드 재계산 |

## 컨테이너 이미지

```text
gops-market-ingestor    → market-ingestor
gops-market-processor   → market-processor, symbol-registry-sync, coverage-repair
gops-market-storage     → 가공 S3 저장기, 원본 S3 백업, ClickHouse 적재기,
                          뉴스 분석 작업자·뉴스 작업
```

## 실행 의존성

Kafka(이벤트 전달), Redis(최신 상태·캐시), ClickHouse(시계열 조회), S3(객체 보관), Secrets Manager와 Alpaca 자격 증명이 필요합니다.

`alfaka.*` 이름 공간은 유지합니다. Docker·Compose·Kubernetes·테스트·로컬 스크립트는 `systems/market-data/shared`를 `PYTHONPATH`에 넣어 같은 공용 패키지를 사용해야 합니다.

## 증시지도 조회 데이터 계약

증시지도는 시장 데이터를 읽는 소비자이며 재무 데이터 수집기가 아닙니다. `GET /api/market/heatmap?universe=sp500`은 `systems/fundamentals`의 Redis·ClickHouse 데이터를 읽고 `sharesOutstanding`(발행 주식 수)과 최신 가격·등락 정보를 조합합니다. 시장 데이터 시스템은 가격을 갱신하고, 재무 시스템은 SEC 수집·CIK(SEC 기업 식별번호) 연결·최신 재무 저장을 담당합니다.

재무 정보가 없으면 `systems/market-data/config/sp500-heatmap-seed.json`의 배치 초기값을 사용합니다. 가격이 없으면 마지막 캐시를 재사용하거나 초기 종목을 중립 색상·최소 시가총액 대체값으로 표시할 수 있습니다.

## 데이터 피드와 거래 세션

Alpaca 실시간 수집은 피드 프로필별로 분리합니다. 기본 v1은 미국 동부시간 04:00~20:00의 장전·정규장·장후에 `sip`, 20:00~04:00의 야간에 `boats`를 사용합니다. 필요한 곳에서는 `overnight`를 Alpaca 야간 피드의 별칭으로 유지합니다. Compose·Kubernetes는 한 프로세스에서 피드를 바꾸지 않고, 활성 프로필별 별도 수집기와 클라이언트 ID를 실행합니다.

원본 메시지·정규화 이벤트·Redis 최신 상태·ClickHouse 행·API 캔들·차트 스냅샷에는 `feedProfile`, `marketSession`을 포함합니다. 세션 값은 `pre/regular/after/overnight`입니다. 과거 행에 세션 메타데이터가 없으면 일·주·월봉 조회는 `regular`를 사용합니다.

기존 ClickHouse 볼륨에는 열을 바로 추가할 수 있지만, 여러 피드의 행을 함께 보존하려면 `infra/clickhouse/initdb/01-market-data.sql`의 피드·세션을 포함한 `ORDER BY`로 테이블을 재구성해야 합니다.

## 차트 수집 범위

과거 차트 데이터를 미리 모두 적재하지 않습니다. API는 차트나 명시적 구독에서 요청한 `symbol + timeframe + range + layer`만 읽습니다. SIP 수집기는 1분봉을 빠르게 보여주기 위해 S&P 500의 `bars/updatedBars/dailyBars/statuses` 기본 수집을 유지할 수 있지만, 이는 과거 차트 사전 적재나 전체 종목 체결·호가 수집이 아닙니다. 종목 레지스트리는 검증·검색을 돕는 자료입니다.

실행 정책은 다음과 같습니다.

- 고정 종목군의 과거 차트 전체 사전 적재를 하지 않습니다.
- SIP S&P 500 기본 수집은 `bars/updatedBars/dailyBars/statuses`로 제한합니다.
- 실시간 `trades/quotes`는 같은 SIP WebSocket에서 명시적으로 활성화한 구독만 받습니다.
- Redis는 종목·주기별로 화면이 요청한 최근 구간만 유지합니다.
- 과거 완료 봉은 ClickHouse에 해당 주기 행이 있으면 직접 읽습니다.
- 시간봉이 없으면 `10m`, 기존 `1m` 순으로 집계합니다. 다른 파생 주기는 문서에 정한 `1m` 또는 `1D`를 사용합니다.
- 작은 전경 요청이 불완전하면 해당 주기의 Alpaca REST 봉으로 보충할 수 있습니다. `ON_DEMAND_FILL_FOREGROUND_AUTO_INTERVALS`가 허용한 장중 주기도 포함합니다.
- 백그라운드 결측 보충은 S3 final·매니페스트를 먼저, Alpaca 과거 직접 조회를 다음으로 사용합니다.
- S3 원본 백업은 저용량 이벤트·봉만 보관하고 실시간 체결·호가는 제외합니다. 활성 조회·적재 입력으로 사용하지 않습니다.

수집기는 종목 목록을 코드에 고정하지 않고 Redis 제어 키에서 결정된 구독 등급을 읽어야 합니다.

## 실시간 전달 경로 점검

AWS나 로컬 Compose에서 실시간 데이터가 보이지 않으면 종목 하나를 끝까지 따라가고 최초로 끊긴 단계에서 원인을 확인합니다. 읽기 전용 점검 도우미:

```bash
PYTHONPATH=systems/market-data/shared python scripts/local/check-live-path.py NVDA --interval 1m
scripts/aws/check-live-path.sh NVDA
```

구성 요소 로그:

```bash
kubectl logs -n alfaka-market-data deploy/alfaka-alpaca-ingestor-sip --tail=100
kubectl logs -n alfaka-market-data deploy/alfaka-market-processor --tail=100
kubectl logs -n alfaka-market-data deploy/alfaka-clickhouse-loader --tail=100
```

Kafka 처리기 그룹이 원본 토픽을 읽고 있으며 소비 지연이 제한된 범위인지 확인합니다.

```bash
kafka-consumer-groups.sh --bootstrap-server "$KAFKA_BOOTSTRAP_SERVERS" --describe --group alfaka-market-processor
```

조회 경로가 같은 이름 공간을 사용하는지도 확인합니다.

```bash
redis-cli -u "$REDIS_URL" GET "gops:market:on-demand:v1:live:candle:NVDA:1m"
curl -fsS "$GOPS_API_BASE_URL/api/charts/candles?symbol=NVDA&interval=1m&limit=20"
curl -fsS "$GOPS_API_BASE_URL/api/monitor/market-data/realtime?symbol=NVDA&interval=1m"
```

필수 흐름은 다음과 같습니다.

```text
Alpaca → market.input.realtime.* Kafka → Python 처리기의 피드 검사
→ Redis 실시간 봉·체결·호가 + market.layer.* Kafka
→ ClickHouse/S3 final → API/WebSocket → 브라우저
```

차트는 `POST /api/charts/active-symbol`의 주기적 갱신으로 현재 종목을 실시간 구독 그룹에 유지합니다. 이 갱신은 화면 주기나 장중 WebSocket 연결 여부와 독립적입니다. API WebSocket 허브는 공통 Redis `market.events` 리스너 하나와 활성 세션의 묶음 조회를 사용합니다. 종목별 Redis 구독 리스너를 다시 만들지 않습니다.

장외 시간에는 통제된 재생·로컬 경로 추적으로 디버깅합니다. 사용자가 직접 AWS 검증을 다시 요청하지 않는 한, 장중 AWS 기본 동작 점검은 별도의 운영·실시장 검증 항목으로 둡니다.

## 데이터 범위 복구

새 로컬 볼륨 초기화·ClickHouse 복원·관심종목 변경 후 다음 작업을 사용합니다.

```bash
docker compose --profile repair run --rm coverage-repair
```

기본값은 dry-run(실제 작업 전 범위·결과를 확인하는 모드)입니다. 다만 작업이 `GET /api/charts/candles`를 호출하므로 API 자체는 요청 범위 안에서 누락 데이터를 보충하고 `fill` 이력을 보고할 수 있습니다.

```bash
COVERAGE_REPAIR_DRY_RUN=false docker compose --profile repair run --rm coverage-repair
```

작업은 Redis·ClickHouse에 직접 접근하지 않고 API를 사용하므로 화면과 같은 조회 규칙을 따릅니다. 실시간 파생 봉은 로컬 `1m/1D` 집계를, 과거 복구는 표준 Alpaca 원본 주기(`1m/5m/10m`는 `1Min`, `1h/4h`는 `10Min`)를 사용합니다.

## 1년치 캔들 초기 적재

`systems/market-data/jobs/candle-bootstrap/main.py`는 운영자가 저장소 S&P 500 종목군을 초기 적재하는 도구입니다. 기본은 dry-run이고 `--apply`가 있어야 저장합니다. 스트림 적재기와 같은 가공 캔들·ClickHouse 행 변환을 사용하고, 대상 `chart_candles`에 존재하는 열만 기록합니다.

두 종목과 월봉 별칭을 저장 없이 점검합니다.

```bash
PYTHONPATH=systems/market-data/shared .venv/bin/python \
  systems/market-data/jobs/candle-bootstrap/main.py \
  --symbols AAPL,NVDA \
  --intervals 1m,1D,1mo
```

사전 점검·저장 공간 확인 후 전체 종목군 1년치를 저장합니다.

```bash
PYTHONPATH=systems/market-data/shared .venv/bin/python \
  systems/market-data/jobs/candle-bootstrap/main.py \
  --intervals 1m,5m,10m,1h,4h,1D,1W,1M \
  --continue-on-error \
  --apply
```

기본 범위는 최근 완료한 미국 동부시간 20:00 장외 세션까지의 달력 기준 365일입니다. 기존 `v2/split` 시각에 `ma5/ma20/ma60`이 완전하면 건너뜁니다. 주기마다 시작 전 59개 준비 봉을 보수적으로 조회해 첫 저장 봉부터 이동평균을 계산하되, 실제 저장은 요청한 반개구간(시작 포함·끝 제외)만 수행합니다.

이동평균이 불완전한 기존 행은 다시 삽입해 `ReplacingMergeTree`가 새 완성 행을 유지하도록 합니다. 종목 자체의 원본 이력이 60개 미만이면 만들 수 없는 이동평균은 `null`로 둡니다. `--start`·`--end`는 함께 지정하고 `--max-symbols`로 적용 범위를 제한합니다. 자격 증명은 기존 환경·Secrets Manager에서 읽으며 명령 인자로 넣지 않습니다.

주식 `1D/1W/1M`의 ClickHouse 식별에는 항상 `market_session=regular`를 사용합니다. 공급자의 주·월봉 시각이 야간에 속해도 같은 장기 봉을 이중 생성하지 않습니다. 암호화폐는 `market_session=crypto`를 유지합니다.

## 요청에 따른 과거 데이터 보충

일반 차트 확장의 조회 순서:

```text
Redis 최근 요청 구간 → ClickHouse → 범위를 제한한 자동·일반 전경 Alpaca REST
→ 백그라운드 S3 final·매니페스트 → 백그라운드 Alpaca 과거 직접 조회
```

표준 과거 봉은 Alpaca `adjustment=split`을 쓰고 `priceAdjustment=split`, `canonicalVersion=v2`로 저장합니다. 조회는 기존·원본·출처 불명 행을 제외합니다. 단, 저장된 `1m`은 당일 실시간 기본 봉을 숨기지 않도록 `priceAdjustment=live`인 Alpaca 완료 봉도 읽습니다. 일봉·과거 표준 적재는 `split`만 사용합니다.

과거 복구 원본은 `1m=1Min`, `5m/10m=1Min`, `1h/4h=10Min`, `1D=1Day`입니다. 시간봉은 실제 `10m`를 저장한 뒤 09:30 ET 기준 목표 봉을 만듭니다. `10m` 도입 이전 범위는 조회 시 기존 `1m` 집계로 대체합니다. 주·월봉은 표준 일봉으로 집계합니다.

장중 주식 직접 보충은 Alpaca REST 호출 전에 세션을 나눕니다. `pre/regular/after`는 설정된 과거 피드를 사용합니다. `overnight`는 BOATS 실시간·요청 수집만 사용하며 활성 차트가 야간 봉을 모으기 전에는 `fill.feedRoutes`에 건너뛴 경로로 표시합니다.

의심되는 ClickHouse 행을 삭제·격리하기 전에 `python -m alfaka.tools.canonical_candle_audit`를 실행합니다. 선택 환경 변수는 `CANONICAL_AUDIT_SYMBOL`, `CANONICAL_AUDIT_INTERVAL`, `CANONICAL_AUDIT_LIMIT`이며 중복·비표준·잘못된 OHLC 행 수를 확인합니다.

이미 적재한 표준 S3 객체의 값이 잘못됐다는 근거가 있으면 명시적 운영 복구에서 해당 객체를 건너뛰고 Alpaca를 다시 읽을 수 있습니다. 분할일의 일봉 고·저가 이상치는 같은 날 분할 보정 `1m`으로 검증하고 이상 고·저가만 보정합니다. 시가·종가·거래량은 `dailyBars`를 유지합니다.

감사용 원본 백업을 부가 저장할 수 있지만 그 백업 누락 때문에 차트 요청·보충·ClickHouse 적재를 실패시키지 않습니다. 보충 판단은 Redis·ClickHouse·S3 final·매니페스트로 하며 원본 백업은 사용하지 않습니다.

로컬에서 AWS 규약으로 실행할 때는 `ALPACA_CREDENTIAL_SOURCE=aws-secrets-manager`, `S3_PROCESSED_FORMAT=parquet`를 사용합니다. Compose는 `APCA_API_KEY_ID`, `APCA_API_SECRET_KEY`도 전달하므로 Secrets Manager가 끊긴 명시적 로컬 점검은 `ALPACA_CREDENTIAL_SOURCE=local-env`로 실행할 수 있습니다. 기존 종목군·`jsonl` 출력은 요청 기반 재구성 규약에 넣지 않습니다.

차트 진입·왼쪽 이동의 과거 조회는 캔들 API를 먼저 사용합니다. 프론트엔드가 `interval`, `limit`, `start/end` 또는 `before`를 지정합니다. Redis·ClickHouse에서 그릴 수 있는 데이터가 없으면 API는 부분·빈 응답을 즉시 반환하면서 같은 범위를 S3 final·매니페스트와 Alpaca 과거 조회로 백그라운드 보충할 수 있습니다.

작은 요청이 Alpaca REST 응답을 기다려야 할 때만 `ON_DEMAND_FILL_FOREGROUND_ALPACA_ENABLED=true`를 켭니다. 응답의 `dataStatus`, `coverage`, `fill`에는 저장소별 적중·미적중·시간 초과·실패가 포함됩니다. 성긴 구간을 전체 범위 사전 적재로 확대하지 않습니다.

운영자가 승인한 초기 적재 전에는 final 캔들 객체 하나로 S3→ClickHouse 적재를 증명합니다.

```bash
S3_MATERIALIZE_KEYS=market-data/rebuild-20260702-lazy-v1/final/candles/.../canonical=v2.parquet python -m alfaka.storage.s3_materializer
```

## 과거 뉴스 보충과 자주 쓰는 캐시

뉴스는 사전 계산합니다. 사용자 질문에는 Redis·ClickHouse의 결과를 읽고 동기적으로 Alpaca·OpenAI를 호출하지 않습니다.

```text
Alpaca News API
→ alpaca-news-ingestor / news-backfill
→ Kafka market.news.alpaca.v1
→ news-intelligence-worker
→ ClickHouse news_article_localizations
→ Redis news:v2:latest:ko:{symbol}
→ API·에이전트 응답

NVDA 일회성 키워드 생성
→ ClickHouse의 2026-07-10~2026-07-14 기사
→ 수동 alfaka-news-daily-summary-rebuild Job
→ ClickHouse·Redis의 일별 뉴스 v2
→ NewsKeywordPanel이 5분마다 백그라운드 갱신
```

저장 책임:

- S3는 가능하면 `include_content=true` 원본을 `articleId`별 표준 객체 하나로 보관합니다.
- 종목별 S3 색인은 원본 객체를 가리키며 기사 본문을 종목마다 복사하지 않습니다.
- ClickHouse는 현재 30일치 앱 조회 행에 한국어 요약·핵심 내용·관련도 v2·감정·링크를 저장합니다.
- Redis는 30일 기사 캐시(`news:v2:latest:*`, `news:v2:topic:*`)와 종목별 일별 요약 30개를 둡니다. 한국어 요약·링크·관련도·일별 수집 범위만 저장하고 기사 본문·원본은 보관하지 않습니다.

AWS는 기사마다 기업·날짜별 요약을 다시 만들지 않습니다. `alfaka-news-daily-summary-worker` 복제 수는 0이고 `NEWS_DAILY_SUMMARY_EVENT_DRIVEN_ENABLED=false`입니다. 반복 뉴스 요약 CronJob은 없습니다.

수동 재생성은 `NVDA`, 양 끝 포함 `2026-07-10..2026-07-14`, 최대 5개 날짜 그룹으로 고정합니다. 직접 관련 기사가 없는 날짜는 건너뜁니다. 저장된 `mention` 행도 현재 규칙 기반 주제 분류기로 다시 확인해 제목에 Nvidia가 직접 나오는 오래된 행을 빠뜨리지 않습니다. 기사 해시·버전 검사로 저장된 v2 요약이 기사 집합과 같으면 OpenAI 호출을 생략합니다.

Kubernetes의 `alfaka-news-backfill`, `alfaka-news-intelligence-rebuild`, 수동 `alfaka-news-daily-summary-rebuild`는 기본 dry-run 환경값으로 구성합니다. 범위와 API·OpenAI 비용을 검토한 의도적인 일회성 작업에서만 `NEWS_BACKFILL_DRY_RUN=false`, `NEWS_INTELLIGENCE_REBUILD_DRY_RUN=false`, `NEWS_DAILY_SUMMARY_REBUILD_DRY_RUN=false`를 지정합니다.

`news-intelligence-rebuild`는 기본적으로 최근 ClickHouse 한국어 행으로 Redis를 채우며 ClickHouse는 다시 쓰지 않습니다. 의도적인 관련도 행 재작성에서만 `NEWS_INTELLIGENCE_REBUILD_REWRITE_CLICKHOUSE=true`를 지정합니다. GitHub Actions는 `rebuild_news_cache=true`일 때만 수동 뉴스 작업을 실행합니다. 일반 배포는 폐기된 `alfaka-news-daily-summary-nvda` CronJob도 삭제해 예약 실행이 남지 않도록 합니다.

로컬 소규모 점검:

```bash
NEWS_BACKFILL_SYMBOLS=AAPL,NVDA NEWS_BACKFILL_DAYS=2 NEWS_BACKFILL_CHUNK_DAYS=2 NEWS_BACKFILL_MAX_PAGES_PER_CHUNK=1 NEWS_BACKFILL_DRY_RUN=false docker compose --profile jobs --profile local-s3 run --rm news-backfill
```

AWS 검토 후 실행:

```bash
AWS_ACCOUNT_ID=993099901407 IMAGE_TAG=news-v2-smoke ./scripts/aws/build-and-push-images.sh market-storage
AWS_ACCOUNT_ID=993099901407 IMAGE_TAG=news-v2-smoke NEWS_BACKFILL_DRY_RUN=false NEWS_BACKFILL_SYMBOLS=AAPL,NVDA NEWS_BACKFILL_DAYS=2 NEWS_BACKFILL_MAX_PAGES_PER_CHUNK=1 ./scripts/aws/run-news-backfill-job.sh
```

S&P 500 전체의 1년치를 처리할 때는 `NEWS_BACKFILL_SYMBOLS`를 제거하고 `NEWS_BACKFILL_UNIVERSE=sp500`을 유지합니다. 먼저 소규모 점검으로 S3·ClickHouse·Redis 건수를 확인합니다. 긴 보충 작업은 `NEWS_BACKFILL_SHARD_INDEX`, `NEWS_BACKFILL_SHARD_COUNT`로 고정 분할합니다. 예를 들어 개수 `8`에 인덱스 `0..7`을 각각 실행합니다.
