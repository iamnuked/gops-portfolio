# 로컬에서 EKS에 배포하기

팀원은 GitHub Actions 대기열을 쓰지 않고 자기 로컬 컴퓨터 리소스로 dev EKS에
배포할 수 있다. 기본 명령어는 하나다.

```bash
AWS_PROFILE=gops-dev ./scripts/aws/deploy-dev-local.sh
```

이 스크립트는 현재 로컬 브랜치나 미커밋 변경을 배포하지 않는다. 기본값은
`git fetch origin dev`로 원격 `origin/dev` 최신 커밋을 가져오고, 임시
`git worktree`에서 그 커밋만 체크아웃해서 빌드한다. 검증용 원격 브랜치는
`REMOTE_BRANCH`, 업로드하지 않은 로컬 커밋은 `LOCAL_REF`로 명시한다. 어느 경우든
미커밋 변경은 배포에 포함하지 않는다.

## 필요한 로컬 도구

- Docker Desktop: 애플리케이션을 Docker 이미지로 빌드한다. Docker 이미지는 앱
  코드와 실행 환경을 한 덩어리로 포장한 배포 단위다.
- AWS CLI v2: AWS API를 터미널에서 호출하는 공식 도구다. 여기서는 ECR 로그인,
  EKS kubeconfig 갱신, Secrets Manager 조회에 사용한다.
- kubectl: Kubernetes 클러스터에 명령을 보내는 CLI다. 여기서는 EKS에 리소스 정의를
  dry-run/적용하고 배포 교체 상태를 확인한다.
- git: 원격 `origin/dev` 커밋을 가져오고 임시 분리 작업 폴더를 만든다.

## 최초 설정

각 팀원은 공유 액세스 키 대신 자기 AWS 프로필을 사용한다.

```bash
aws configure --profile gops-dev
aws sts get-caller-identity --profile gops-dev
```

배포 권한에는 최소한 ECR 업로드, EKS 클러스터 조회, dev 이름 공간 Kubernetes 배포,
`icon/logodev` Secrets Manager 읽기 권한이 필요하다.

Docker가 켜져 있는지도 확인한다.

```bash
docker info
```

## 일반 배포

원격 `dev`에 업로드된 코드만 배포된다.

```bash
AWS_PROFILE=gops-dev ./scripts/aws/deploy-dev-local.sh
```

스크립트 동작:

1. `origin/dev` 최신 SHA를 고정한다.
2. EKS `ConfigMap/gops-dev-deploy-state`에서 서비스별 마지막 성공 SHA를 읽는다.
3. 서비스별 비교 기준과 `origin/dev` 사이의 변경 차이를 기존 서비스 연결 규칙으로 분석한다.
4. 변경된 서비스 이미지만 로컬 Docker로 빌드해 ECR에 업로드한다.
5. 임시 분리 작업 폴더의 Kustomize 환경별 구성만 수정해 EKS에 서버 측 사전 검증·적용한다.
6. 선택된 Deployment와 상시 시뮬레이터 배포 교체, 프론트엔드·백엔드 기본 동작 점검를 확인한다.
7. 성공하면 선택된 서비스의 `service.<name>.lastSuccessfulSha`를 최신 SHA로 갱신한다.

서비스별 배포 상태가 없으면 먼저 기존 `lastSuccessfulSha`를 확인한다. 단,
기존 `lastSuccessfulServices`에 해당 서비스가 들어 있을 때만 그 SHA를 비교 기준으로
사용한다. 그래도 비교 기준이 없으면 현재 EKS 주요 Deployment의 이미지 태그를 읽어
비교 기준으로 사용한다. 이 대체 조회도 실패하면 해당 서비스는 안전하게 재빌드 대상이
된다.

이 구조에서는 백엔드만 성공 배포해도 프론트엔드의 미배포 변경이 사라지지 않는다.
다음 실행 때 프론트엔드의 서비스별 비교 기준부터 다시 변경 차이를 계산하기 때문이다.

## 사전 검증

실제 이미지 업로드나 Kubernetes 적용 없이 대상·변경 차이·서버 측 사전 검증을 확인한다.

```bash
DRY_RUN=true AWS_PROFILE=gops-dev ./scripts/aws/deploy-dev-local.sh
```

원격 기능 브랜치를 검증 배포할 때도 먼저 dry-run한다. 로컬 체크아웃이나
미커밋 파일이 아니라 지정한 `origin/<branch>` 커밋이 대상이다.

```bash
REMOTE_BRANCH=codex/expand-chart-patterns \
FORCE_SERVICES=frontend,agent-orchestrator \
CHART_INTERPRETATION_ONLY=true \
DRY_RUN=true \
AWS_PROFILE=gops-dev \
./scripts/aws/deploy-dev-local.sh
```

업로드하지 않은 로컬 `dev` 커밋을 배포할 때도 먼저 같은 검증을 거친다.

```bash
LOCAL_REF=dev \
FORCE_SERVICES=frontend,backend,simulator \
DRY_RUN=true \
AWS_PROFILE=gops-dev \
./scripts/aws/deploy-dev-local.sh
```

실제 배포는 위 명령에서 `DRY_RUN=true`만 제거한다.

`CHART_INTERPRETATION_ONLY=true`는 기존 Geometry 자산을 읽는 조회기 호환 확인 전용
배포 경계다. 전체 Kustomize 환경별 구성를 적용하지 않고 다음 Deployment의 이미지만
교체한다.

```text
gops-frontend
agent-analysis-worker
agent-orchestrator
```

이 경로는 `chart-asset-builder`를 갱신하지 않으므로 해설 생성기·프롬프트 변경을
검증하거나 개발 패널에서 자산을 재생성하는 용도로 사용하지 않는다. 생성기 변경은 일반 `agent-orchestrator` 배포로 검증한다. 일반 배포는 작도 자산 생성기 교체 직후
`scripts/aws/preflight-chart-commentary-aws.sh`를 자동 실행하며, 현재 이미지 태그와
`chart-commentary.ko.v5` 런타임 계약이 맞지 않으면 기존 되돌리기 경로로 실패한다.

따라서 같은 에이전트 이미지를 공유하는 `chart-asset-builder`, `chart-geometry-build`
CronJob, 마이그레이션·유지보수 Job과 다른 에이전트 워크로드는 변경하지 않는다. 이 모드는
`FORCE_SERVICES=frontend,agent-orchestrator`를 반드시 함께 사용하며 마이그레이션, 캐시
재생성, 플랫폼 적용 옵션이 하나라도 켜져 있으면 시작 전에 실패한다. 실제 배포도
위 명령에서 `DRY_RUN=true`만 제거해 같은 경계를 유지한다.

## 비상시 강제 지정

전체 앱 이미지를 강제로 다시 빌드/배포한다.

```bash
FORCE_SERVICES=all AWS_PROFILE=gops-dev ./scripts/aws/deploy-dev-local.sh
```

일부 서비스만 강제로 배포할 수도 있다.

```bash
FORCE_SERVICES=frontend,backend AWS_PROFILE=gops-dev ./scripts/aws/deploy-dev-local.sh
```

시뮬레이터 이미지만 강제로 다시 빌드하고, 상시 단일 복제본 Deployment를 갱신할 수도 있다.

```bash
FORCE_SERVICES=simulator AWS_PROFILE=gops-dev ./scripts/aws/deploy-dev-local.sh
```

일반 앱 배포가 시뮬레이터 Pod와 백엔드 연결을 같이 올리므로 별도 시작 명령은 없다.
시뮬레이터 이미지가 선택되면 새 이미지로 실제 틱 데이터셋을 앱 배포 교체 전에 자동 적재한다.
수동 복구나 사전 점검이 필요할 때만 아래 명령을 직접 실행한다.

```bash
AWS_PROFILE=gops-dev ./scripts/aws/run-simulator-replay-import.sh
```

기존 `READY` 데이터셋이 있으면 이 명령도 다시 실행하지 않는다. 필요할 때
`start-dev-simulator.sh`는 준비 상태 점검, `stop-dev-simulator.sh`는 LIVE 전환에만 쓰며
둘 다 복제 수나 백엔드 연결을 변경하지 않는다.

주문 마이그레이션은 `order-worker` 선택 시, 차트 마이그레이션은 `agent-orchestrator`
선택 시 앱 배포 교체 전에 자동 실행된다. `agent-orchestrator`를 선택하면 두 마이그레이션
이미지가 함께 선택된다. 뉴스 캐시 재생성만 명시적 설정가 필요하다.
단, 위 `CHART_INTERPRETATION_ONLY` 경로는 이 일반 결합을 적용하지 않는다.

```bash
FORCE_SERVICES=order-worker AWS_PROFILE=gops-dev ./scripts/aws/deploy-dev-local.sh
FORCE_SERVICES=agent-orchestrator AWS_PROFILE=gops-dev ./scripts/aws/deploy-dev-local.sh
REBUILD_NEWS_CACHE=true FORCE_SERVICES=market-storage AWS_PROFILE=gops-dev ./scripts/aws/deploy-dev-local.sh
```

## 실패 시 동작

- Docker build나 ECR 업로드가 실패하면 EKS 적용 전이므로 클러스터 상태는 바뀌지 않는다.
- 자동 order/chart 마이그레이션이 실패하면 앱 워크로드 적용 전에 중단한다.
- agent 배포 교체 뒤 IRSA 스냅샷 시험 저장가 실패하면 배포를 실패 처리하고 되돌리기을 시도한다.
- EKS 적용 이후 배포 교체, 기본 동작 점검, 재생성 단계가 실패하면 선택된 Deployment에
  `kubectl rollout undo`를 시도한다.
- 실패한 실행은 `gops-dev-deploy-state`를 갱신하지 않는다.
- 로컬 작업 폴더는 수정하지 않고, 임시 분리 작업 폴더는 종료 시 삭제한다.

GitHub Actions `.github/workflows/deploy-dev.yml`은 비상용 백업 경로로 유지한다.
