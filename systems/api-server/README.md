# API 서버 시스템

차트·주문 API, 주문 가능 금액 조회, WebSocket 통신을 중계하는 FastAPI 서버입니다. FastAPI는 Python으로 HTTP API를 만드는 프레임워크로, 이 프로젝트에서는 브라우저와 내부 데이터·주문 시스템 사이의 진입점을 담당합니다.

## 폴더 구성

```text
pods/api-server/gops-backend/   FastAPI 애플리케이션
tests/                        API 서버 테스트
.env.example                   백엔드 단독 실행용 환경 변수 예시
```

`jobs/`와 `shared/`는 향후 API 서버가 직접 소유할 코드를 위한 예약 경로입니다. 이 시스템의 책임에 해당할 때만 코드를 추가합니다.

## 실행 구성

```text
image:   gops-api-server
docker:  infra/docker/Dockerfile.gops-backend
command: uvicorn app.main:app --host 0.0.0.0 --port 8000
```

백엔드만 로컬 프로세스로 실행할 때는 저장소 루트에서 아래 명령을 사용합니다. Git에서 제외된 로컬 환경 파일을 명시해 모든 차트 모듈이 같은 환경 변수를 받게 합니다. Uvicorn은 FastAPI 앱을 실행하는 서버이며, 아래 명령에서 앱 위치·수신 주소·포트·환경 파일을 지정합니다.

```bash
PYTHONPATH=systems/market-data/shared:systems/order/shared:systems/order:systems/api-server/pods/api-server/gops-backend \
  .venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8000 \
  --env-file systems/api-server/.env
```

`systems/api-server/.env.example`은 저장소에 보존된 환경 변수 기준입니다. Docker Compose는 루트 `.env.example`을 기준으로 하며, 실제 `.env`는 Git에 포함하지 않습니다.

## 로그인과 인증

Google OAuth 로그인은 API 서버가 담당합니다. OAuth는 사용자의 비밀번호를 직접 받지 않고 외부 로그인 서비스로 신원을 확인하는 방식입니다. `AUTH_ENABLED=true`이면 `/api/orders`, `/ws/orders/{order_id}`, `/api/llm/*`를 보호합니다. 차트·시장 데이터 API는 공개 상태를 유지합니다.

세션은 Redis의 `AUTH_REDIS_KEY_PREFIX` 아래 저장합니다. 브라우저에는 JavaScript에서 읽을 수 없는 HttpOnly 쿠키로 세션 식별자만 전달합니다.

백엔드는 먼저 환경 변수 `GOOGLE_OAUTH_CLIENT_ID`, `GOOGLE_OAUTH_CLIENT_SECRET`, `AUTH_SESSION_SECRET`을 읽습니다. 인증이 활성화됐는데 값이 비어 있으면 `GOOGLE_OAUTH_SECRET_NAME`에 지정한 AWS Secrets Manager의 JSON 비밀값에서 누락된 항목을 읽을 수 있습니다.

## 차트의 뉴스·실적 이벤트

`GET /api/charts/events`는 로드된 캔들 범위의 저장된 실적·일별 뉴스 마커를 반환합니다. 요청 중에는 ClickHouse만 읽습니다. S&P 500 실적은 `yahoo_earnings_estimates`, 뉴스는 `news_company_daily_summaries`에서 조회합니다. S&P 500 밖의 종목은 실적 상태를 빈 값으로 두고 뉴스를 반환합니다. 이 요청은 Yahoo·Alpaca를 호출하지 않습니다.

틱 재생에서는 시뮬레이터의 가상시각인 `virtualTime`을 받습니다. 날짜별 최신 행을 고르기 전에 `generated_at <= virtualTime`을 적용하고, 커서 이후에 수집한 실적은 제외합니다.

SIM의 `GET /api/market/news/latest`는 `published_at`과 `localized_at`이 모두 커서 이하인 ClickHouse 한국어 기사만 읽습니다. 실시간 Redis 캐시를 읽거나 미리 채우지 않습니다. `GET /api/market/news/daily`도 ClickHouse만 읽으며 최신 일간 등락률을 덧붙이지 않습니다. 최신 상태만 제공하는 나머지 시장·관심종목 뉴스 경로는 계속 `simulation_data_unavailable`을 반환합니다.

## 시뮬레이터 빠른 주문

SIM 모드의 `GET /api/simulator/quote?symbol=NVDA`는 현재 재생 중인 매수·매도 호가를 제공합니다. 빠른 주문 화면은 차단된 실시간 주문흐름 경로 대신 이 호가를 사용하고, 기존 `POST /api/orders`로 주문을 제출합니다. 체결·잔고·주문 이력은 현재 시뮬레이터의 `userId + runId` 원장에 남습니다. 재생 목록 밖의 종목에는 합성 호가를 만들지 않습니다.

## 증시지도 조회

`GET /api/market/heatmap?universe=sp500`는 브라우저 TreeMap에 필요한 조회 전용 데이터를 제공합니다. TreeMap은 값의 크기에 따라 사각형 면적을 나누는 시각화입니다.

`systems/fundamentals`가 만든 Redis 요약 키 `gops:fundamentals:summary:v1:{SYMBOL}`을 먼저 읽고, 없으면 ClickHouse의 `sec_financial_facts`, `sec_company_tickers`를 조회합니다. 발행 주식 수인 `sharesOutstanding`과 시장 데이터 가격을 조합해 Redis에 캐시합니다. SEC 공시 수집은 별도 작업자·저장소의 책임입니다.

공개 응답은 종목 식별 정보, 기업·섹터·산업명, `marketCap`, `layoutMarketCap`, 현재가 관련 필드인 `lastPrice`, `previousClose`, `changePercent`, 호버 화면의 거래량 필드로 제한합니다. Redis 내부 데이터에 추가 재무 정보가 있어도 이 API로 보내지 않습니다. 상세 재무·시계열은 기업 선택 시 종목별 API로 조회합니다.

## 공용 패키지 가져오기

```text
alfaka.*      → systems/market-data/shared
kis_trader.*  → systems/order/shared
```

구조만 정리하는 작업에서는 라우트 동작과 API 규약을 변경하지 않습니다.
