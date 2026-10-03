# msa4-k8s-setting

Docker Desktop 기반 MSA 배포 실습에서 Gateway API와 Argo CD를 준비하는 클러스터 설정 저장소입니다. 앱 이미지 생성, 외부 진입점 구성, GitOps 동기화까지 이어지는 교육 문서가 포함되어 있습니다.

## 구성 요소의 역할

| 구성 | 역할 |
| --- | --- |
| GatewayClass·Gateway | NGINX Gateway 컨트롤러를 연결하고 프런트엔드·API용 HTTP 리스너를 정의합니다. |
| Argo CD Application | 앱별 매니페스트 경로를 읽고 대상 클러스터와 네임스페이스에 동기화합니다. |
| [msa4-k8s-manifest](https://github.com/HongD93/msa4-k8s-manifest) | 앱 Deployment, Service, 환경 설정과 HTTPRoute를 관리하는 별도 저장소입니다. |

이 저장소에는 DB나 앱 Deployment가 없습니다. 클러스터 진입점과 GitOps 연결을 준비한 뒤 별도 앱 매니페스트를 사용합니다.

## 문서와 설정 탐색

| 문서·파일 | 알 수 있는 내용 |
| --- | --- |
| [전체 커리큘럼](doc-education/01-msa-deploy-curriculum.md) | 실습 전제조건, 서비스·저장소 관계와 전체 진행 순서 |
| [CI 실습](doc-education/02-ci-github-actions.md) | 앱 이미지 빌드와 GitHub Actions 구성 |
| [Gateway API 실습](doc-education/03-gateway-api-setup.md) | CRD, NGINX Gateway Fabric, Gateway와 앱 라우트 준비 |
| [Argo CD 실습](doc-education/04-argocd-gitops-setup.md) | 저장소 인증, Application과 동기화 흐름 |
| [01-gateway-class.yaml](01-gateway-class.yaml) | `nginx` GatewayClass와 사용할 컨트롤러 |
| [02-gateway.yaml](02-gateway.yaml) | `nginx-gateway` 네임스페이스의 `gateway`와 HTTP 리스너 |
| [argocd/](argocd/) | 인증·게시물·게이트웨이·프런트엔드용 Application 4개 |

## 시작 순서

1. 커리큘럼에서 실습 환경과 저장소 관계를 확인합니다.
2. CI 문서를 따라 사용할 앱 이미지를 준비하고 앱 매니페스트의 이미지·환경 설정을 확인합니다.
3. Gateway API 문서에 따라 CRD와 NGINX Gateway Fabric을 준비한 뒤 GatewayClass·Gateway를 적용합니다.
4. Argo CD를 설치하고 매니페스트 저장소에 접근할 수 있도록 구성합니다.
5. Application의 원본·대상을 확인한 뒤 앱별 동기화를 연결합니다.

문서의 명령과 설정은 교육 환경을 전제로 합니다. 클러스터 버전과 실제 서비스 설정에 맞는지 확인한 뒤 필요한 단계를 적용합니다.

## Gateway 설정

Kubernetes 클러스터, kubeconfig와 `kubectl`이 필요합니다. Gateway API `v1` CRD, NGINX Gateway Fabric 컨트롤러와 `nginx-gateway` 네임스페이스를 먼저 준비합니다. GatewayClass는 `gateway.nginx.org/nginx-gateway-controller`를 참조합니다.

저장소 루트에서 대상 context를 확인한 뒤 적용합니다. `apply`는 해당 클러스터의 리소스를 생성하거나 갱신합니다.

```sh
kubectl config current-context
kubectl apply -f 01-gateway-class.yaml
kubectl get gatewayclass
kubectl apply -f 02-gateway.yaml
kubectl get gateway -n nginx-gateway
```

| 리스너 | 포트 | 연결할 앱 라우트 |
| --- | --- | --- |
| `client` | `5173` | 프런트엔드 HTTPRoute |
| `scg` | `8080` | API 게이트웨이 HTTPRoute |

GatewayClass의 승인 상태, Gateway의 준비 상태와 컨트롤러의 노출 주소를 확인합니다. `allowedRoutes.namespaces.from`은 `All`로 설정되어 있어 다른 네임스페이스의 라우트도 연결할 수 있으므로 사용 환경에 맞게 범위를 검토합니다.

## Argo CD 설정

Argo CD와 Application CRD를 먼저 설치합니다. Application은 매니페스트 저장소의 `master` 브랜치에서 각각 `auth`, `post`, `scg`, `client` 경로를 읽고 현재 클러스터의 `msa4-meerkatgram` 네임스페이스에 동기화하도록 설정되어 있습니다.

적용 전에 Application의 다음 항목을 확인합니다.

- `repoURL`: 읽을 매니페스트 저장소
- `targetRevision`, `source.path`: 동기화할 브랜치와 앱 경로
- 대상 클러스터와 네임스페이스
- 비공개 저장소를 읽기 위한 Argo CD의 저장소 인증 설정

```sh
kubectl apply -f argocd/
kubectl get applications -n argocd
```

Application에는 자동 동기화, `prune: true`, `selfHeal: true`, `CreateNamespace=true`가 설정되어 있습니다. 적용하면 Argo CD가 앱 리소스를 생성·수정할 수 있고, Git에서 제거한 관리 리소스를 정리하거나 수동 변경을 원본 상태로 되돌릴 수 있습니다.
