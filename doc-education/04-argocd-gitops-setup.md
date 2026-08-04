# 🔄 [교육용] Part C — GitOps (ArgoCD) 실습 가이드

[Part B](03-gateway-api-setup.md)에서 준비한 YAML을 `k8s-manifests` 레포에 올리고, ArgoCD가 이를 **최초로** 클러스터에 적용합니다. 이어서 [Part A](02-ci-github-actions.md)의 워크플로우에 "매니페스트 업데이트" 스텝을 추가해 전체 자동배포 파이프라인을 완성합니다.

---

## 1. `k8s-manifests` 레포 구조 작성

GitHub에 **Public** 레포 `k8s-manifests`를 생성하고, [Part B](03-gateway-api-setup.md)에서 작성한 매니페스트를 폴더별로 옮겨 담습니다.

```
k8s-manifests/
├── scg/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── httproute.yaml
│   ├── configmap.yaml      ← CORS_ALLOW_ORIGIN, AUTH_SERVICE_*/POST_SERVICE_* 등 (사실상 필수)
│   └── secret.yaml         ← JWT_SECRET (auth-service와 동일 값, 사실상 필수)
├── auth-service/
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── deployment.yaml
│   └── service.yaml
├── post-service/
│   ├── deployment.yaml
│   └── service.yaml
└── client/
    ├── deployment.yaml
    ├── service.yaml
    └── httproute.yaml
```

```bash
git add .
git commit -m "Add initial k8s manifests"
git push
```

> 이 YAML들은 아직 클러스터에 한 번도 적용된 적이 없습니다. 다음 절의 ArgoCD Application 생성이 **최초 적용**입니다.
>
> `k8s-manifests`는 Public repo이므로 `Secret` 매니페스트가 포함되어도 4절의 Repository 연결 방식(인증 없이 HTTPS)은 **바뀌지 않습니다.** ([03번 문서](03-gateway-api-setup.md) 6.1절 참고 — 이번 실습은 더미 값이라 Public 유지, 실제 서비스라면 Private + 별도 인증이 필요합니다.)
>
> [Part B](03-gateway-api-setup.md)에서 만든 GatewayClass/Gateway는 `k8s-manifests`가 아니라 **별도의 `k8s-setting` 레포**에 있습니다(클러스터 1회성 인프라라 앱 매니페스트와 성격이 다름). 아래 5절에서 만들 ArgoCD Application 매니페스트(`argocd-app-*.yaml`)도 이 `k8s-setting` 레포에 함께 둡니다. `k8s-setting`은 ArgoCD 스스로 부트스트랩하는 리소스를 담고 있어 ArgoCD Application으로 관리하지 않고, GatewayClass/Gateway처럼 **`kubectl apply`로 직접 적용**합니다.

---

## 2. ArgoCD 설치

> **실행 위치: 로컬 클러스터 (kubectl context가 kind 클러스터를 가리키는 상태)**

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace

kubectl get pods -n argocd
```

모든 Pod가 `Running`이 될 때까지 대기합니다.

---

## 3. ArgoCD 접속 (도메인 없이 port-forward)

프로덕션은 `https://argocd.meerkat.p-e.kr`처럼 실도메인+Gateway로 접속하지만, 로컬 실습에는 도메인이 없으므로 `kubectl port-forward`로 접속합니다.

```bash
kubectl port-forward svc/argocd-server -n argocd 6500:443
```

브라우저에서 `https://localhost:6500` 접속 → 자체 서명 인증서 경고는 "고급 → 계속 진행"으로 넘어갑니다.

### 3.1 초기 admin 비밀번호 확인

```bash
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

- ID: `admin`
- PW: 위 출력값

### 3.2 argocd CLI 설치 (선택)

```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

argocd login localhost:8080 --insecure
```

---

## 4. ArgoCD ↔ GitHub Repository 연결 (private repo — 인증 필요)

ArgoCD가 `k8s-manifests` 레포를 읽을 수 있도록 Deploy Key를 등록합니다.

```bash
# SSH 키 생성
ssh-keygen -t ed25519 -C "argocd" -f ~/.ssh/argocd_github -N ""

# 공개키 확인 (GitHub에 등록할 값)
cat ~/.ssh/argocd_github.pub
```

GitHub `k8s-manifests` 레포 → Settings → Deploy keys → Add deploy key
- Title: `argocd`
- Key: 위에서 출력한 공개키 붙여넣기
- Allow write access: **체크 안 함** (읽기 전용)

ArgoCD에 개인키 등록 (UI 방식):

`https://localhost:6500` 로그인 후 (3. ArgoCD 접속 참고) 진행

- **Settings → Repositories → Connect Repo (SSH)**
- name: 자유롭게 작성
- project: 자유롭게 작성 (단, argocd Application Manifest의 spec.project와 동일)
- Repository URL: `git@github.com:<org>/k8s-manifests.git`
- SSH private key data: 마스터에서 아래 명령으로 출력한 내용을 그대로 붙여넣기
  ```bash
  cat ~/.ssh/argocd_github
  ```
- **CONNECT** 클릭 후 연결 상태(Successful) 확인

> `argocd` CLI가 마스터에 설치되어 있다면 `argocd login` 후 `argocd repo add git@github.com:<org>/k8s-manifests.git --ssh-private-key-path ~/.ssh/argocd_github` 명령으로도 동일하게 등록 가능

---

## 5. Application 4개 생성 — 여기서 처음으로 배포됩니다

네 서비스 각각에 대해 Application을 만듭니다. `source.path`만 다르고 나머지는 동일한 패턴입니다. 아래 `argocd-app-*.yaml` 파일들은 `k8s-setting` 레포에 보관합니다(1절 참고 — GatewayClass/Gateway와 같은 성격의 클러스터 인프라 리소스라 앱 매니페스트 레포와 분리합니다).

### 5.1 scg-app

`argocd-app-scg.yaml`

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: scg-app
  namespace: argocd
spec:
  project: default
  source:
    # <변경필요>
    repoURL: https://github.com/<github-username>/k8s-manifests.git
    targetRevision: main
    path: scg
  destination:
    server: https://kubernetes.default.svc
    namespace: msa4-meerkatgram
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### 5.2 auth-service-app / post-service-app / client-app

`argocd-app-auth-service.yaml`, `argocd-app-post-service.yaml`, `argocd-app-client.yaml`을 위와 동일하게 작성하되 `metadata.name`과 `spec.source.path`만 각각 `auth-service`, `post-service`, `client`로 바꿉니다.

### 5.3 적용

```bash
kubectl apply -f argocd-app-scg.yaml
kubectl apply -f argocd-app-auth-service.yaml
kubectl apply -f argocd-app-post-service.yaml
kubectl apply -f argocd-app-client.yaml

kubectl get application -n argocd
```

---

## 6. Sync 상태 확인

```bash
kubectl describe application scg-app -n argocd
# 또는
argocd app get scg-app
```

4개 Application 모두 `Sync Status: Synced`, `Health Status: Healthy`인지 확인합니다.

```bash
kubectl get pods -n msa4-meerkatgram
```

**이 시점에 처음으로 Pod 4개가 뜹니다.** [Part B](03-gateway-api-setup.md)에서 준비만 해뒀던 YAML이 지금 처음 클러스터에 반영된 것입니다.

---

## 7. 외부 접근 검증

```bash
# scg 경유(API, 8080 포트) — 정상 응답이 와야 함
curl http://localhost:8080/api/auth/health
curl http://localhost:8080/api/posts/health

# client 경유(화면, 5173 포트) — index.html이 내려와야 함
curl http://localhost:5173/
```

---

## 8. 격리 검증 (핵심 체크포인트)

`auth-service`/`post-service`는 **HTTPRoute가 없으므로** Gateway를 통한 외부 경로 자체가 존재하지 않습니다. ClusterIP 타입이라 애초에 클러스터 밖에서 IP로 직접 접근할 수도 없습니다.

```bash
kubectl get svc auth-service -n msa4-meerkatgram -o jsonpath='{.spec.clusterIP}'
# 위에서 나온 IP로 외부(호스트)에서 curl 시도 → 연결 자체가 안 됨 (라우팅 경로 없음)
```

> **참고**: `kubectl port-forward svc/auth-service -n msa4-meerkatgram 8080:80` 같은 명령은 `kubectl` API 접근 권한이 있는 관리자만 쓸 수 있는 별도의 통로이지, 외부 사용자(브라우저)가 임의로 쓸 수 있는 경로가 아닙니다. 격리 검증의 핵심은 **"HTTPRoute가 없으면 외부 사용자 입장에서는 도달할 방법이 없다"**는 것 자체입니다.

---

## 9. selfHeal 실습

ArgoCD가 관리 중인 리소스를 수동으로 건드리면 자동으로 원복되는지 확인합니다.

```bash
kubectl scale deployment post-service -n msa4-meerkatgram --replicas=3

# 잠시 후 다시 확인 — 1로 되돌아가 있어야 정상
kubectl get deployment post-service -n msa4-meerkatgram -w
```

ArgoCD UI에서도 해당 Application에 일시적으로 `OutOfSync`가 떴다가 다시 `Synced`로 돌아오는 것을 관찰할 수 있습니다.

---

## 10. PAT 발급 — `k8s-manifests` 레포 전용 (이 커리큘럼에서 필요한 유일한 PAT)

Part A의 워크플로우는 지금까지 ghcr.io push만 했습니다(같은 레포 작업이라 `GITHUB_TOKEN`으로 충분). 이제 **다른 레포(`k8s-manifests`)에 커밋을 push**해야 하므로 PAT가 필요합니다.

1. GitHub → Settings → Developer settings → **Personal access tokens → Fine-grained tokens** → **Generate new token**
2. Token name: `manifest-repo-token`
3. Expiration: 교육 기간에 맞춰 설정 (예: 7일)
4. Repository access: **Only select repositories** → `k8s-manifests`
5. Permissions → Repository permissions → **Contents: Read and write**
6. **Generate token** → 값 복사

> fine-grained 토큰이라 **`k8s-manifests` 레포 하나만** 건드릴 수 있습니다. 분실/유출되어도 피해 범위가 제한됩니다.

### 10.1 Actions Secret 등록 (4개 앱 레포 모두 동일하게 반복)

`scg`, `auth-service`, `post-service`, `client` 레포 각각에서:

1. 레포 → **Settings → Secrets and variables → Actions → New repository secret**
2. Name: `MANIFEST_REPO_TOKEN`
3. Secret: 위에서 발급한 PAT 값
4. **Add secret**

4개 레포 모두 등록했는지 확인합니다.

---

## 11. Part A 워크플로우 확장 — TODO였던 부분을 채웁니다

[Part A 2절](02-ci-github-actions.md#2-github-actions-워크플로우-작성-build--push만)에서 만들었던 워크플로우에 남겨둔 TODO 주석 자리에, 아래 스텝을 **추가**합니다 (새로 만드는 게 아니라 기존 파일을 이어서 씁니다).

`scg` 레포의 `.github/workflows/deploy.yml` 전체 (추가된 부분은 주석 표시):

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

env:
  # <변경필요> 서비스별로 다르게
  IMAGE_NAME: scg
  # <추가> k8s-manifests 안의 폴더명, 서비스별로 다르게
  MANIFEST_PATH: scg
  # <추가> 매니페스트를 올릴 대상 레포 (owner/repo 형식)
  MANIFEST_REPO: <github-username>/k8s-manifests

permissions:
  contents: read
  packages: write

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v7

      - name: Log in to ghcr.io
        uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push image
        run: |
          IMAGE=ghcr.io/${GITHUB_REPOSITORY_OWNER,,}/$IMAGE_NAME
          docker build -t $IMAGE:$GITHUB_RUN_NUMBER .
          docker push $IMAGE:$GITHUB_RUN_NUMBER

      # <추가> 여기부터가 Part A에서 TODO로 남겨뒀던 부분입니다
      # 이번엔 "다른" 레포(k8s-manifests)를 추가로 내려받는 step — with: 블록이라 ${{ }} 필수 (본문 규칙 참고)
      - name: Checkout k8s-manifests
        uses: actions/checkout@v7
        with:
          # 내려받을 대상 레포
          repository: ${{ env.MANIFEST_REPO }}
          # 다른 레포이므로 GITHUB_TOKEN 대신 10절에서 등록한 PAT 사용
          token: ${{ secrets.MANIFEST_REPO_TOKEN }}
          # 이 레포를 받아둘 하위 폴더 이름 (임의 지정 가능)
          path: manifests

      - name: Update image tag
        # 여기부터는 다시 run: 블록이므로 $VAR로 통일
        run: |
          # 위 build 스텝과 동일하게 이미지 경로 재조립
          IMAGE=ghcr.io/${GITHUB_REPOSITORY_OWNER,,}/$IMAGE_NAME
          # k8s-manifests 안에서 이 서비스 폴더로 이동 (자기 폴더만 건드림)
          cd manifests/$MANIFEST_PATH
          # deployment.yaml의 image 태그 줄을 방금 push한 태그로 치환
          sed -i "s|image: ${IMAGE}:.*|image: ${IMAGE}:${GITHUB_RUN_NUMBER}|" deployment.yaml
          # 커밋 작성자 이메일 (자유롭게 지정 가능한 값)
          git config user.email "actions@github.com"
          # 커밋 작성자 이름
          git config user.name "GitHub Actions"
          # 변경된 deployment.yaml 커밋
          git commit -am "Deploy ${IMAGE_NAME}:${GITHUB_RUN_NUMBER}"
          # k8s-manifests에 push → ArgoCD가 감지해 자동 배포
          git push
```

> `${{ }}` vs `$VAR` 구분 규칙은 [Part A 2절](02-ci-github-actions.md#2-github-actions-워크플로우-작성-build--push만)과 동일합니다: `run:` 안은 `$VAR`(환경변수), `with:`처럼 셸이 개입하지 않는 필드는 `${{ }}`(표현식).

### 11.1 auth-service / post-service / client 레포에 동일하게 확장

같은 방식으로 `env.MANIFEST_PATH`만 바꿔 나머지 레포의 워크플로우도 확장합니다.

```yaml
env:
  IMAGE_NAME: auth-service
  MANIFEST_PATH: auth-service
  MANIFEST_REPO: <github-username>/k8s-manifests
```

```yaml
env:
  IMAGE_NAME: post-service
  MANIFEST_PATH: post-service
  MANIFEST_REPO: <github-username>/k8s-manifests
```

```yaml
env:
  IMAGE_NAME: client
  MANIFEST_PATH: client
  MANIFEST_REPO: <github-username>/k8s-manifests
```

### 11.2 확장 후 첫 실행 확인

```bash
git add .github/workflows/deploy.yml
git commit -m "Add manifest update step"
git push
```

순서대로 관찰합니다.

1. 각 레포 **Actions** 탭 → 워크플로우 성공 확인
2. `k8s-manifests` 레포 → 해당 폴더(`scg/`, `auth-service/`, `post-service/`, `client/`)만 수정된 새 커밋 생성 확인 (`git log --stat`으로 변경 파일 확인, 다른 폴더는 안 건드렸는지 체크)
3. ArgoCD UI → 해당 Application이 자동으로 `Syncing` → `Synced`

---

## 12. End-to-End 검증 (핵심 실습)

`post-service` 코드를 간단히 수정합니다 (예: 응답 문자열 변경).

```bash
# post-service 레포에서
git add .
git commit -m "test: change response message"
git push
```

순서대로 관찰합니다.

1. `post-service` 레포 **Actions** 탭 → 워크플로우 성공 확인
2. `k8s-manifests` 레포 → `post-service/deployment.yaml`만 변경된 새 커밋 생성 확인
3. ArgoCD UI → `post-service` Application이 자동으로 `Syncing` → `Synced`
4. 클러스터 확인:
   ```bash
   kubectl get pods -n msa4-meerkatgram
   ```
   `post-service` Pod만 `AGE`가 리셋(재생성)되고, `scg`/`auth-service` Pod는 그대로인지 확인
5. 기능 확인:
   ```bash
   curl http://localhost:8080/api/posts/health
   ```
   수정한 응답이 반영됐는지 확인

> client만 수정해 push해도 동일하게 확인할 수 있습니다: `client` Pod만 재생성되고, `curl http://localhost:5173/`의 응답(화면 HTML)만 바뀝니다.

이 과정에서 **scg, auth-service는 전혀 건드리지 않았다는 점**이 MSA 독립배포의 핵심입니다.

---

## 13. 롤백 실습

`k8s-manifests` 레포에서 방금 반영된 커밋을 되돌립니다.

```bash
# post-service 배포 커밋 확인
git log --oneline
git revert <커밋해시>
git push
```

ArgoCD가 감지해 이전 이미지 태그로 자동 롤백합니다. ArgoCD UI에서 `History and Rollback` 메뉴로 직접 이전 버전을 선택해 되돌리는 방법도 함께 시연합니다.

---

## 14. 장애 격리 검증 (핵심 실습)

`auth-service`의 `deployment.yaml` 이미지 태그를 존재하지 않는 값으로 바꿔 의도적으로 장애를 만듭니다.

```bash
# k8s-manifests 레포에서
cd auth-service
# 존재하지 않는 태그로 변경
sed -i 's|:[0-9]*$|:9999|' deployment.yaml
git commit -am "break auth-service on purpose"
git push
```

**관찰**

```bash
kubectl get pods -n msa4-meerkatgram
# auth-service Pod만 ImagePullBackOff / ErrImagePull
```

```bash
# auth 쪽 기능은 실패하지만
curl http://localhost:8080/api/auth/health

# post 쪽 기능은 영향 없이 정상 동작해야 함
curl http://localhost:8080/api/posts/health

# 화면(client)도 영향 없이 정상 동작해야 함
curl http://localhost:5173/
```

확인 후 복구합니다.

```bash
git revert HEAD
git push
```

ArgoCD가 자동으로 정상 이미지로 되돌리는 것까지 확인합니다.

---

## 15. 트러블슈팅

| 증상 | 원인 | 확인 |
| :--- | :--- | :--- |
| Repository 연결 `Unknown`/실패 | URL 오타, 아직 Public 전환 전 | `Settings → Repositories`에서 URL 재확인 |
| Application `OutOfSync` 지속 | `path`가 실제 레포 구조와 다름 | `kubectl describe application <name> -n argocd`의 Events |
| Application은 있는데 Pod가 없음 | `destination.namespace` 오타 또는 CreateNamespace 누락 | `kubectl get ns`, `syncOptions.CreateNamespace=true` 확인 |
| `k8s-manifests` checkout/push 실패 (`403`) | `MANIFEST_REPO_TOKEN` Secret 미등록 또는 권한 범위 오류 | 레포 Settings → Secrets, PAT의 Repository permissions 재확인 |
| 워크플로우는 성공했는데 ArgoCD가 반영 안 함 | ArgoCD polling 주기(기본 3분) 대기 필요 | `argocd app get <app>`으로 `Sync Status` 재확인 |
| 다른 서비스 폴더까지 같이 변경됨 | `MANIFEST_PATH` 오타 | 워크플로우 `env.MANIFEST_PATH` 확인 |

---

## 16. 작업 체크리스트

| 단계 | 내용 | 완료 |
| :--- | :--- | :---: |
| 1 | `k8s-manifests` 레포 구조 작성 및 push | ☐ |
| 2 | ArgoCD Helm 설치 | ☐ |
| 3 | port-forward로 UI 접속, 초기 비밀번호 확인 | ☐ |
| 4 | Repository 연결 (Public, 인증 없이) | ☐ |
| 5 | Application 4개 생성 → 최초 Sync 확인 | ☐ |
| 6 | 외부 접근 검증 (API `:8080` + 화면 `:5173`) | ☐ |
| 7 | auth/post 격리 검증 | ☐ |
| 8 | selfHeal 실습 | ☐ |
| 9 | `manifest-repo-token` PAT 발급 (공용 1개) | ☐ |
| 10 | 4개 레포 Secret 등록 + 워크플로우 확장 | ☐ |
| 11 | End-to-End 검증 (post-service 기준) | ☐ |
| 12 | 롤백 실습 | ☐ |
| 13 | 장애 격리 검증 (auth-service 기준) | ☐ |

여기까지 완료하면 [01-msa-deploy-curriculum.md](01-msa-deploy-curriculum.md)의 최종 목표 — "코드 push 한 번으로 해당 서비스만 자동 재배포되는 개인별 MSA 파이프라인"이 완성됩니다.

---

## 부록 — ApplicationSet (이번 기수는 진행하지 않음)

3개 Application을 개별 생성하는 대신 하나의 ApplicationSet으로 자동 생성하는 방법입니다. **이번 기수 커리큘럼에는 포함하지 않으며, 다음 기수 준비용으로만 남겨둡니다.**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: msa4-meerkatgram-apps
  namespace: argocd
spec:
  generators:
  - git:
      repoURL: https://github.com/<github-username>/k8s-manifests.git
      revision: main
      directories:
      - path: "*"
  template:
    metadata:
      name: '{{path.basename}}-app'
    spec:
      project: default
      source:
        repoURL: https://github.com/<github-username>/k8s-manifests.git
        targetRevision: main
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: msa4-meerkatgram
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
        - CreateNamespace=true
```

> 적용 전 5절에서 만든 개별 Application 4개는 먼저 삭제해야 합니다(같은 이름 규칙을 관리하면 충돌).
