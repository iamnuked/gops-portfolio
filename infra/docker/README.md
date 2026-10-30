# Docker 이미지 구성

Docker는 코드와 실행 환경을 컨테이너 이미지로 묶는 도구입니다. 이 폴더는 서비스별 이미지를 만드는 Dockerfile과 프론트엔드 웹 서버 설정을 보관합니다.

## 서비스별 이미지

| Dockerfile | 담당 |
| --- | --- |
| `Dockerfile.gops-frontend` | React 화면을 빌드하고 Nginx로 제공 |
| `Dockerfile.gops-backend` | FastAPI 차트·주문·실시간 API |
| `Dockerfile.gops-market-ingestor` | Alpaca 시장 데이터 수집 |
| `Dockerfile.gops-market-processor` | 이벤트 가공·실시간 상태 갱신 |
| `Dockerfile.gops-market-storage` | S3·ClickHouse 저장과 뉴스 처리 |
| `Dockerfile.gops-order-worker` | 주문 Outbox·모의체결·마이그레이션·정합성 작업 |
| `Dockerfile.gops-kis-adapter` | KIS 모의투자 연동 |
| `Dockerfile.gops-agent-orchestrator` | 에이전트 분석·전달·작도 자산 관련 작업 |
| `Dockerfile.gops-simulator` | 저장된 실제 시장 데이터 재생 |

`nginx/`에는 프론트엔드 정적 파일과 `/api`, `/ws` 요청 전달 설정이 있습니다. Nginx는 웹 파일을 제공하고 API 서버로 요청을 중계하는 웹 서버입니다.

## 로컬 실행과 배포

로컬 서비스 조합은 루트 `docker-compose.yml`, Kubernetes 배포 조합은 [Kubernetes 폴더](../k8s/README.md)에서 정의합니다. Dockerfile만으로 시장 데이터나 운영 자격 증명이 준비되지는 않습니다.

- [과거 로컬 실행 절차](../../docs/LOCAL_RUNTIME_REFERENCE.md)
- [EKS 배포 절차](../../docs/LOCAL_EKS_DEPLOY.md)
- [Terraform 리소스 범위](../aws/terraform/README.md)
