# GOPS 프론트엔드

실제 미국 주식 차트에 패턴·추세·지지·저항을 표시하고 차트 해설·기업 정보·모의투자를 연결하는 웹 화면입니다.

## 주요 기술과 역할

- React 19: 화면을 재사용 가능한 구성 요소로 만드는 라이브러리입니다.
- TypeScript: JavaScript 코드에 자료형을 지정해 잘못된 값 사용을 개발 중에 확인합니다.
- Vite: 개발 서버와 배포용 파일 빌드를 담당합니다.
- 자체 차트 엔진: 캔들·지표·작도를 Canvas 2D에 그립니다. 상태·좌표 계산은 [차트 엔진](../chart-engine/README.md)에 있습니다.

## 화면 구성

- 증시지도·추천종목·기업 분석·차트·포트폴리오 패널을 조합합니다.
- 차트 위 패턴·추세·지지·저항 레이어를 선택하고 저장된 분석 근거와 연결합니다.
- REST API로 과거 데이터를, WebSocket으로 실시간 갱신을 받습니다.
- 시장 데이터 저장소나 증권사 API를 브라우저에서 직접 호출하지 않습니다.

## 개발·빌드 명령

저장소 루트에서 실행합니다.

```sh
npm ci --prefix apps/gops-frontend
npm run dev --prefix apps/gops-frontend
npm run build --prefix apps/gops-frontend
npm run test:chart --prefix apps/gops-frontend
```

`npm ci`는 잠금 파일에 고정된 의존성을 설치합니다. `dev`는 개발 서버를 시작하고, `build`는 자료형 검사 후 배포 파일을 만들며, `test:chart`는 차트 관련 검사를 실행합니다. UI만 실행해도 과거 시장 데이터나 외부 API가 자동으로 복원되는 것은 아닙니다.

## 관련 문서

- [차트 데이터 설계](../../docs/CHART_DATA_ARCHITECTURE.md)
- [차트 패턴·작도 자산](../../docs/CHART_ANALYSIS_ASSETS.md)
- [백엔드 API](../../systems/api-server/README.md)
- [로컬 실행 참고](../../docs/LOCAL_RUNTIME_REFERENCE.md)
