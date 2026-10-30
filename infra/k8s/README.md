# Kubernetes 배포 구성

Kubernetes는 여러 컨테이너의 실행·연결·상태를 관리하는 플랫폼입니다. 이 폴더는 GOPS 애플리케이션과 데이터 계층을 실행할 리소스를 정의합니다.

## 폴더 구성

- `base/`: 공통 애플리케이션·플랫폼 구성입니다.
- `overlays/`: AWS 등 환경별 차이를 공통 구성 위에 덧붙입니다.

이 조합에는 Kustomize를 사용합니다. Kustomize는 원본 YAML을 복제하지 않고 환경별 설정을 합치는 도구입니다.

## 주요 리소스의 역할

| 리소스 | 역할 |
| --- | --- |
| Deployment | API·프론트엔드·처리 작업자의 실행과 교체 관리 |
| Service | Pod가 바뀌어도 같은 주소로 서비스 연결 |
| StatefulSet | 데이터 계층의 고정 식별·저장 공간 연결 |
| Job·CronJob | 일회성·정기 작업 실행 |
| ConfigMap·Secret·서비스 계정 | 설정·비밀값·작업별 권한 연결 |

프론트엔드·API·시장 데이터 수집/처리/저장·주문·에이전트를 역할별 워크로드로 분리합니다. 저장된 데이터 계층의 단일 복제본 구성을 다중 복제 고가용성으로 해석해서는 안 됩니다.

## 구성 확인

다음 명령은 리소스를 배포하지 않고 최종 YAML을 생성합니다.

```sh
kubectl kustomize infra/k8s/base
kubectl kustomize infra/k8s/overlays/aws
```

실제 배포에는 해당 환경의 클러스터·이미지·권한·설정이 필요합니다. 이 폴더와 Terraform 모듈만으로 VPC·EKS 전체를 새로 생성하는 것은 아닙니다.

- [로컬 EKS 배포](../../docs/LOCAL_EKS_DEPLOY.md)
- [Terraform 관리 범위](../aws/terraform/README.md)
- [Kafka 구성](../../docs/KAFKA_ARCHITECTURE.md)
