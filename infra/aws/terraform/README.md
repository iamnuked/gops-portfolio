# AWS Terraform 기반 리소스

이 폴더는 애플리케이션을 기존 AWS 인프라에 연결하는 데 필요한 리소스를 만듭니다. Terraform은 인프라 구성을 코드로 정의하고 적용하는 도구입니다. 이 모듈은 전체 네트워크·클러스터를 생성하지 않고 이미지 저장소·데이터 보관·권한 연결을 담당합니다.

## 만드는 것

```text
GOPS 전용 이미지의 ECR 저장소
기존 시장 데이터 S3 버킷 참조
비공개·버전 관리 AI 코치 스냅샷 S3 버킷
Secrets Manager의 Alpaca 비밀값 참조
시장 데이터·AI 코치 작업자의 IRSA 역할과 정책
```

## 만들지 않는 것

```text
EKS 클러스터
MSK 클러스터
ElastiCache Redis
VPC·서브넷·NAT
```

위 항목은 회사 공통 인프라 또는 별도 Terraform 모듈에서 만드는 쪽이 안전합니다. 이 폴더는 그 결과값을 받아 애플리케이션을 딱 붙이는 역할입니다.

## 적용 순서

```sh
cd infra/aws/terraform
cp terraform.tfvars.example terraform.tfvars
terraform init
terraform plan
terraform apply
```

적용 후 `terraform output` 값을 `infra/k8s/overlays/aws`의 자리표시자에 넣습니다.

대상 전용 이미지 저장소:

```text
gops-frontend
gops-api-server
gops-market-ingestor
gops-market-processor
gops-market-storage
gops-order-worker
gops-kis-adapter
gops-agent-orchestrator
```

빌드 스크립트 환경 변수 연결:

| Terraform 출력 | `scripts/aws/build-and-push-images.sh` 환경 변수 |
| --- | --- |
| `frontend_ecr_repository_url` | `ECR_FRONTEND_REPO` |
| `api_server_ecr_repository_url` | `ECR_API_SERVER_REPO` |
| `market_ingestor_ecr_repository_url` | `ECR_MARKET_INGESTOR_REPO` |
| `market_processor_ecr_repository_url` | `ECR_MARKET_PROCESSOR_REPO` |
| `market_storage_ecr_repository_url` | `ECR_MARKET_STORAGE_REPO` |
| `order_worker_ecr_repository_url` | `ECR_ORDER_WORKER_REPO` |
| `kis_adapter_ecr_repository_url` | `ECR_KIS_ADAPTER_REPO` |
| `agent_orchestrator_ecr_repository_url` | `ECR_AGENT_ORCHESTRATOR_REPO` |

이미지를 전부 다시 빌드하지 않고 변경된 서비스만 빌드/푸시할 수 있습니다.

```sh
AWS_ACCOUNT_ID=993099901407 scripts/aws/build-and-push-images.sh frontend
AWS_ACCOUNT_ID=993099901407 scripts/aws/build-and-push-images.sh backend market-storage
AWS_ACCOUNT_ID=993099901407 SERVICES=frontend,backend scripts/aws/build-and-push-images.sh
```

서비스 이름은 `scripts/aws/build-and-push-images.sh --help`로 확인합니다.

## Secrets Manager 비밀값

Secrets Manager는 API 키 같은 비밀값을 보관하는 AWS 서비스입니다. 기본값은 기존 `dev/alpaca` 비밀값을 참조합니다.

```hcl
alpaca_secret_name   = "dev/alpaca"
create_alpaca_secret = false
```

비밀값은 다음 JSON 키 형식을 사용해야 합니다.

```json
{"APCA_API_KEY_ID":"...","APCA_API_SECRET_KEY":"..."}
```

기존 비밀값이 없고 Terraform으로 내용이 비어 있는 보관 항목만 만들 때
`create_alpaca_secret = true`로 바꿉니다.

`google_oauth_secret_name`을 비워두면 IRSA 정책에 Google OAuth 비밀값을
추가하지 않습니다. Google 로그인 비밀값을 Secrets Manager에서 읽을 때는
아래처럼 값을 넣으면 gops-backend가 해당 비밀값을 읽을 수 있도록 같은 Pod 정책에 ARN이 포함됩니다. ARN은 AWS 리소스를 식별하는 이름입니다.

```hcl
google_oauth_secret_name = "oauth/google"
```

OpenAI API 키를 External Secrets로 동기화할 때는 `openai_secret_name`의 ARN도 Pod 정책에 포함됩니다. External Secrets는 외부 비밀값을 Kubernetes에 동기화하는 구성 요소입니다.

```hcl
openai_secret_name = "/gops/prod/agent-orchestrator/openai/api-key"
```

Terraform 대신 개발용 도우미 스크립트로 IRSA를 갱신할 때도 같은 비밀값 이름을
넘겨야 합니다.

```bash
GOOGLE_OAUTH_SECRET_NAME=oauth/google \
OPENAI_SECRET_NAME=/gops/prod/agent-orchestrator/openai/api-key \
./scripts/aws/create-irsa.sh
```

## S3 버킷

기본값은 이미 만들어진 `gops-market-data-993099901407-ap-northeast-2-an`
버킷을 참조합니다.

```hcl
s3_bucket_name   = "gops-market-data-993099901407-ap-northeast-2-an"
create_s3_bucket = false
```

버킷을 Terraform으로 새로 만들 때만 `create_s3_bucket = true`로 바꿉니다.

AI 코치 스냅샷 버킷은 이 모듈에서 항상 생성하며 다음 값을 별도로 출력합니다.

```text
ai_coach_snapshot_s3_bucket
ai_coach_worker_irsa_role_arn
```

첫 출력은 `AI_COACH_SNAPSHOT_S3_BUCKET`에 넣고, 두 번째 출력은 `ai-coach-worker-sa`의 역할 주석에 연결합니다. IRSA는 Kubernetes 서비스 계정에 AWS 역할을 연결해 작업별 권한을 부여하는 방식입니다.

이 역할은 사용자·날짜 범위의 `ai-coach/input/` 보관 데이터를 읽고, `ai-coach/snapshots/`와 `ai-coach/reports/`만 읽고 쓸 수 있습니다. 버킷 목록 조회와 객체 삭제 권한은 없습니다. 기존 시장 데이터 서비스 역할에는 `ai-coach/reports/` 읽기 권한만 주어 인증된 백엔드가 최신 코치 리포트를 제공하게 합니다.

조건부 쓰기는 재시도가 기존 객체를 덮어쓰지 않도록 합니다. HTTP 412로 불변 객체가 이미 존재함을 확인했을 때만 작업자가 처음 저장한 스냅샷을 읽고 검증해 재사용하며, 새로 만든 입력을 다시 분석하지 않습니다.

현재 버전은 `ai_coach_snapshot_retention_days`(기본 90일) 후 만료됩니다. 버전 관리 버킷에서는 이때 과거 버전으로 바뀌며, `ai_coach_snapshot_noncurrent_retention_days`(기본 1일, 허용 1~7일)가 지나면 영구 삭제 대상이 됩니다. 기본 설정에서 약 91일째 삭제 대상이 되며 180일이 아닙니다. S3 수명주기 처리는 비동기이므로 정확한 삭제 시각을 보장하는 값은 아닙니다.

애플리케이션 배포 전에 `terraform plan`으로 생성될 버킷·역할 이름이 Kubernetes 오버레이 값과 맞는지 확인합니다. AWS 앱 오버레이는 코치 보관을 필수 모드로 실행합니다. 분석 작업자 배포 후 `scripts/aws/verify-ai-coach-snapshot-s3.sh`가 해당 작업자에서 비민감 시험 객체를 조건부 저장하고 다이제스트를 검증하며 읽어 실제 IRSA·버킷 연결을 확인합니다. 목록 조회·삭제 권한은 추가하지 않습니다.
