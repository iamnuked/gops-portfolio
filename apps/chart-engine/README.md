# GOPS 차트 엔진

브라우저에서 동작하는 TypeScript 차트 엔진입니다. 차트 상태, 명령 처리, 지표, 좌표·배율 계산과 화면 렌더링을 담당합니다. React 화면 코드는 `apps/gops-frontend`에 둡니다.

## 담당 범위

- `ChartDocument`: 종목·캔들·지표·작도 등 차트의 현재 상태를 담는 문서 모델입니다.
- 차트 명령 리듀서: 차트 조작 명령을 받아 다음 상태를 계산합니다.
- 과거 캔들 스냅샷과 실시간 이벤트의 형식을 정규화합니다.
- 지표와 뷰포트(현재 화면에 보이는 가격·시간 범위)를 관리합니다.
- 렌더링 장면을 만들고 Canvas 2D로 그립니다. Canvas 2D는 브라우저의 2차원 그래픽 그리기 기능입니다.
- 종목·관심종목을 조회하는 클라이언트의 응답 형식을 정규화합니다.

## 직접 수정하는 경로

```text
apps/chart-engine/src/
apps/chart-engine/package.json
apps/chart-engine/tsconfig.json
shared/chart-contract/
```

## 변경 전에 연동을 확인할 경로

```text
apps/gops-frontend/src/
systems/api-server/
systems/market-data/
```

차트 엔진은 Alpaca, Kafka, S3, Redis, ClickHouse를 직접 읽지 않습니다. 과거 데이터는 백엔드 REST API에서 받고, 실시간 데이터는 WebSocket 이벤트로 받습니다.
