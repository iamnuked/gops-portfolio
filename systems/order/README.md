# 주문 시스템

KIS 해외주식 모의투자 주문, 영속 모의계좌, 보유종목 조회, Outbox 발행, 증권사 어댑터, 데이터베이스 마이그레이션과 정합성 확인을 담당합니다. Outbox는 주문 변경과 발행할 이벤트를 함께 저장해 전달 누락을 줄이는 방식입니다.

## 폴더 구성

```text
pods/order-outbox/        계속 실행되는 Outbox 발행기
pods/paper-order-matcher/ 실시간 호가로 영속 모의주문을 체결하는 작업자
pods/kis-adapter/         계속 실행되는 KIS 연동 어댑터
jobs/migrations/         PostgreSQL 스키마 변경 작업
jobs/reconciler/         범위를 제한해 수동 실행하는 정합성 확인 작업
shared/kis_trader/       주문 공용 패키지
tests/kis_trader/       주문 테스트
```

SQL 마이그레이션은 `shared/kis_trader/migrations`에 있습니다. 실행 코드에서 `kis_trader.persistence.migrations`를 통해 가져오기 때문입니다.

## 실행 진입점

| 경로 | 역할 |
| --- | --- |
| `pods/order-outbox/main.py` | `kis_trader.cli outbox-publish --limit 100`을 2초 간격으로 실행합니다. |
| `pods/paper-order-matcher/main.py` | `market.layer.quotes.v1`을 소비해 모의 지정가 주문을 체결합니다. |
| `pods/kis-adapter/main.py` | `kis_trader.cli broker-adapter --timeout-seconds 1.0`을 실행합니다. |
| `jobs/migrations/main.py` | `kis_trader.cli migrate`를 실행합니다. |
| `jobs/reconciler/main.py` | `kis_trader.cli reconcile --rows-json []`를 실행합니다. |

`kis-adapter`는 기본적으로 실제 KIS 모의투자 HTTP 클라이언트를 사용합니다. 명시적인 로컬 가짜 응답 점검에서만 `KIS_BROKER_ADAPTER_ARGS=--fake-kis success`를 지정합니다.

v1은 해외주식 모의투자 지정가 주문만 받으며 주문 화면에 주문 가능 외화 금액을 제공합니다. KIS 자격 증명은 기본적으로 AWS Secrets Manager의 `tead/gops/kis`에서 읽습니다.

## 컨테이너 이미지

```text
gops-order-worker   → order-outbox, paper-order-matcher, migrations, reconciler
gops-kis-adapter    → kis-adapter
```

## 실행 의존성

- PostgreSQL: 주문·원장 상태 저장
- Kafka: 주문 이벤트 전달
- Redis: 실시간 구독 제어
- Secrets Manager와 KIS 모의투자 자격 증명
- KIS 모의투자 API

`kis_trader.*` 이름 공간을 유지합니다. Docker·Compose·Kubernetes·테스트·로컬 스크립트는 `systems/order/shared`를 `PYTHONPATH`에 포함해야 합니다. `KIS_ENV=real`은 v1에서 비활성화 상태를 유지합니다.

## 영속 모의투자

`kis_trader.paper`는 KIS와 일시적인 GOPS 시연 시뮬레이터 양쪽에서 독립된 모듈입니다. `POST /api/paper/orders`는 `Idempotency-Key`가 필요하며, PostgreSQL에 모의 현금이나 정수 수량 주식을 예약합니다. 멱등성 키인 `Idempotency-Key`는 같은 주문 요청이 반복되어도 주문이 중복 생성되지 않도록 식별하는 값입니다. 이 경로는 KIS 주문 Outbox에 쓰지 않습니다.

매수는 현재 매도호가가 지정가 이하(`ask <= limit`), 매도는 현재 매수호가가 지정가 이상(`bid >= limit`)일 때 체결합니다. 주문은 체결·취소 또는 명시적 계좌 초기화로 제거될 때까지 대기합니다.

`0006_paper_trading.sql`은 계좌 세대, 포지션, 주문, 이벤트와 추가 기록만 허용하는 현금 원장을 만듭니다. `paper-order-matcher` 전에 적용해야 합니다. 작업자는 미체결 주문 종목을 Alpaca 실시간 구독의 최우선 그룹에 유지하며, 기본 최대 종목 수는 100개입니다.

보유종목 화면은 `GET /api/account/holdings`를 사용합니다. 내부의 `kis_trader.kis.client.DemoKisHttpClient`는 KIS 모의투자 잔고 API를 호출하는 프로젝트 클라이언트이며 기존 `dev/kis` Secrets Manager 규약을 재사용합니다.

해외주식 잔고 응답의 `frcr_buy_amt_smtl1`은 외화 매입금액 합계이며 현금으로 취급하지 않습니다. KIS가 실제 외화 현금 필드를 제공하지 않으면 `account.cashForeign`은 `null`을 유지합니다. 주식·전체 외화 가치는 보유종목 평가금액 합계로 구하며 임의의 현금 비중을 만들지 않습니다.
