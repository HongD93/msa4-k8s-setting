# 🎓 [교육용] Docker Desktop 기반 MSA 자동배포 실습 커리큘럼

## 1. 개요

훈련생 대상 실습 커리큘럼입니다. Docker/K8s/CI-CD 기본 개념 강의는 기완료된 상태를 전제로, **"코드 Push → 자동 빌드 → 자동 배포"** 전체 흐름을 개인 실습으로 체득하는 데 집중합니다.

프로덕션 인프라([00-k8s-architecture-overview.md](../situation/00-k8s-architecture-overview.md), [05-argocd-cicd-build.md](../situation/05-argocd-cicd-build.md))의 핵심 패턴(Gateway API 진입점, GitOps 배포)을 로컬 환경에 맞게 축소·변형하여 실습합니다.

### 1.1 진행 방식 — Day 구분 없이 파트(기술 스택) 단위로 진행

목표는 3일이지만, 훈련생 수준(비전공자 3개월 차)을 감안하면 4~5일까지 늘어날 수 있습니다. 그래서 시간표(Day1/Day2/Day3)로 못박지 않고, **아래 3개 파트를 순서대로, 각 파트의 체크포인트를 통과하면 다음 파트로 넘어가는 방식**으로 진행합니다.

| 파트 | 기술 스택 | 상세 문서 |
| :--- | :--- | :--- |
| **Part A** | CI — GitHub Actions + ghcr.io | [02-ci-github-actions.md](02-ci-github-actions.md) |
| **Part B** | 클러스터 진입점 — Gateway API + NGINX Gateway Fabric | [03-gateway-api-setup.md](03-gateway-api-setup.md) |
| **Part C** | GitOps — ArgoCD + 자동배포 파이프라인 완성 | [04-argocd-gitops-setup.md](04-argocd-gitops-setup.md) |

**중요 원칙 — 한 번 만든 건 나중에 다시 갈아엎지 않습니다.**

- Part A에서 만드는 GitHub Actions 워크플로우는 Part C에서 스텝을 **추가**할 뿐, 다시 만들지 않습니다.
- Part B에서 작성하는 Deployment/Service/HTTPRoute YAML은 `kubectl apply`로 먼저 적용해보고 나중에 ArgoCD로 갈아타는 게 아니라, **Part C에서 ArgoCD가 최초로 적용**합니다. 즉 Part B에서는 파일만 준비하고 아직 클러스터에 적용하지 않습니다.
- 따라서 각 파트를 반드시 이 순서(A → B → C)로 진행해야 합니다. Part B의 앱 매니페스트 작성은 Part A에서 최소 1번 이미지가 push된 뒤에만 가능합니다(태그가 실제로 존재해야 YAML에 적을 수 있음).

---

## 2. 전제 조건

| 항목 | 내용 |
| :--- | :--- |
| 실습 환경 | Docker Desktop 기반 kind 멀티노드 클러스터 (마스터+워커, 각자 PC에 사전 구성 완료) |
| 실습 형태 | 개인 실습, 각자 GitHub 개인 계정 사용 |
| 레포 구성 (1인당 6개, 모두 Public) | `scg`, `auth-service`, `post-service`, `client`, `k8s-manifests`(앱 매니페스트), `k8s-setting`(GatewayClass/Gateway/ArgoCD Application 등 클러스터 인프라) |
| 대상 프로젝트 | 훈련생이 사전에 구현한 MSA 프로젝트 (SCG + auth-service + post-service + client) |
| CI 도구 | GitHub Actions + GitHub Container Registry(ghcr.io) |
| 환경변수/Secret 관리 | `k8s-manifests`는 훈련생 포트폴리오 용도로 **Public 유지**. 비민감 설정값은 `ConfigMap`, JWT 서명키 등은 `Secret`(실습용 더미 값 — 상세 내용은 [03번 문서](03-gateway-api-setup.md) 6.1절 참고) |
| 매니페스트 관리 | ArgoCD (GitOps) |

### 2.1 제외 범위

이번 커리큘럼에서는 아래 항목을 다루지 않습니다.

| 제외 항목 | 사유 |
| :--- | :--- |
| 파일서버(NFS) | 별도 강의로 분리 |
| 모니터링(Prometheus/Grafana) | 별도 강의로 분리 |
| DB(MySQL) 연동 | 이번 커리큘럼은 순수 CI/CD 파이프라인(빌드→배포)에만 집중 |
| MinIO(오브젝트 스토리지) | 상동. 파일 업로드를 실제로 써보는 실습은 하지 않지만, 서비스 코드가 기동 시점에 `MINIO_*` 환경변수를 참조한다면 실제 연동 없이도 앱이 정상 기동하도록 더미 값을 `Secret`에 채워 넣습니다(코드 비활성화가 아니라 더미 환경변수로 기동만 보장) |
| cert-manager / Let's Encrypt (HTTPS) | 로컬 환경에 실도메인이 없어 HTTP-01 챌린지 자체가 불가능. Gateway는 HTTP 리스너만 사용 |
| Jenkins | 로컬 실습 부담을 줄이기 위해 GitHub Actions로 대체 (Part A 문서의 매핑표로 운영 환경 전환 대비) |
| ArgoCD ApplicationSet | 이번 기수는 Application을 각각 수동 생성. ApplicationSet은 [04번 문서](04-argocd-gitops-setup.md) 부록에 참고용으로만 남겨두고 진행하지 않음 |

---

## 3. 아키텍처

프론트엔드(`client`)는 SCG의 하위 서비스가 아니라, **SCG와 같은 Gateway에서 포트 기준으로 구분되어 직접 외부 노출**됩니다. Gateway에 리스너를 2개 두어(`client`→5173, `scg`→8080) 각 리스너에 HTTPRoute를 하나씩 붙이는 방식이며, 경로(path) 매칭은 쓰지 않습니다 — 어느 포트로 들어왔는지가 곧 라우팅 기준입니다.

포트가 다르면 브라우저 기준으로는 다른 origin이므로, **경로 기반 방식과 달리 CORS가 필요합니다.** SCG가 `globalcors` 설정(`CORS_ALLOW_ORIGIN` 환경변수)으로 client origin(`http://localhost:5173`)을 명시적으로 허용하는 방식으로 처리합니다.

```
[외부/브라우저]
    │ HTTP (kind extraPortMappings, localhost:5173 / localhost:8080)
    ▼
NGINX Gateway Fabric (Gateway API, HTTP 리스너 2개: client/5173, scg/8080)
    │
    ├── HTTPRoute (parentRefs.sectionName: scg)    ──▶ scg (Service, 외부 노출, 8080)
    │                                                    │ 내부망(Calico CNI, ClusterIP)
    │                                                    ├──▶ auth-service (ClusterIP만 — 외부 미노출)
    │                                                    └──▶ post-service (ClusterIP만 — 외부 미노출)
    │
    └── HTTPRoute (parentRefs.sectionName: client) ──▶ client (Service, 외부 노출, 5173, 정적 SPA)
```

- **scg, client는 외부에 노출**되고(각자 다른 포트로), `auth-service`/`post-service`는 CNI 내부망(ClusterIP)을 통해서만 접근 가능합니다.
- SCG는 애플리케이션 레벨에서 `auth-service`/`post-service`로 내부 라우팅을 수행하는 API 게이트웨이 역할을 합니다. 라우팅 대상(URI)·매칭 경로(predicate)는 SCG 코드에 하드코딩하지 않고 ConfigMap 환경변수(`AUTH_SERVICE_URI`, `AUTH_SERVICE_PREDICATE` 등)로 주입합니다.
- client는 정적 빌드 산출물을 nginx로 서빙하는 SPA이며, `msa4-meerkatgram` 네임스페이스 안에 다른 서비스와 함께 배치되어 **같은 CNI/네트워크**를 공유합니다("같은 프로젝트" 느낌).
- HTTPRoute는 `parentRefs.sectionName`으로 어느 리스너(포트)에 붙을지만 지정하며 `rules.matches`(path 매칭)는 사용하지 않습니다. 즉 경로 우선순위를 신경 쓸 필요가 없는 대신, **포트 2개를 각각 kind에 노출**해야 합니다.
- 프로덕션과 달리 cert-manager/TLS가 없으므로 Gateway는 HTTP 리스너만 사용합니다.

### 3.1 레포 구조

```
k8s-manifests/ (Public repo) — 앱 매니페스트
├── scg/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── httproute.yaml      ← sectionName: scg 리스너에 연결
│   ├── configmap.yaml      ← CORS_ALLOW_ORIGIN, AUTH_SERVICE_*/POST_SERVICE_* 등
│   └── secret.yaml         ← (필요 시) JWT_SECRET 등, 실습용 더미 값
├── auth-service/
│   ├── configmap.yaml      ← 비민감 설정값
│   ├── secret.yaml         ← JWT_SECRET 등, 실습용 더미 값
│   ├── deployment.yaml
│   └── service.yaml        ← ClusterIP
├── post-service/
│   ├── deployment.yaml
│   └── service.yaml        ← ClusterIP (필요 시 configmap.yaml/secret.yaml 추가)
└── client/
    ├── deployment.yaml
    ├── service.yaml
    └── httproute.yaml      ← sectionName: client 리스너에 연결

k8s-setting/ (Public repo) — 클러스터 1회성 인프라 (앱 매니페스트와 분리)
├── gateway-class.yaml
├── gateway.yaml             ← 리스너 2개: client(5173), scg(8080)
└── argocd/
    ├── argocd-app-scg.yaml
    ├── argocd-app-auth-service.yaml
    ├── argocd-app-post-service.yaml
    └── argocd-app-client.yaml
```

각 앱 레포(`scg`, `auth-service`, `post-service`, `client`)는 자신의 GitHub Actions 워크플로우에서 **자기 폴더만** 수정합니다. `k8s-setting`은 클러스터 부트스트랩용으로 한 번만 다루며, 이후 파트에서 반복 수정하지 않습니다.

---

## 4. Part A — CI (GitHub Actions + ghcr.io)

> 상세 절차: [02-ci-github-actions.md](02-ci-github-actions.md)

4개 서비스(scg, auth-service, post-service는 Gradle+Spring Boot / client는 Node 빌드+nginx 서빙) 각각 dockerfile을 작성하고, GitHub Actions로 build+push를 자동화합니다. 이 단계에서는 아직 k8s를 건드리지 않습니다.

**완료 체크포인트**
- [ ] 4개 서비스 모두 로컬 `docker build`/`docker run` 기동 확인
- [ ] 4개 레포 모두 GitHub Actions 워크플로우 작성, push 시 자동으로 ghcr.io에 이미지가 올라감
- [ ] ghcr.io 패키지 4개 모두 Visibility Public 전환 완료

---

## 5. Part B — Gateway API (클러스터 진입점)

> 상세 절차: [03-gateway-api-setup.md](03-gateway-api-setup.md)

**B-1 (클러스터 인프라 설치, `k8s-setting` 레포)**은 Part A와 무관하게 언제 해도 됩니다. **B-2 (앱 매니페스트 작성, `k8s-manifests` 레포)**는 Part A에서 이미지가 최소 1번 push된 이후에 진행합니다. 이 단계에서 작성하는 YAML은 아직 `kubectl apply` 하지 않고 **파일로만 준비**해둡니다 (Part C에서 ArgoCD가 최초로 적용).

**완료 체크포인트**
- [ ] `k8s-setting` 레포에 GatewayClass/Gateway(리스너 2개: client 5173, scg 8080) 작성, `GatewayClass` `Accepted: True`
- [ ] (해당 시) MinIO 접속 정보는 더미 값으로 Secret에 채워 기동만 보장 (기능 실습은 제외)
- [ ] auth-service ConfigMap/Secret/Deployment/Service(ClusterIP) YAML 작성 완료
- [ ] post-service Deployment+Service(ClusterIP, 필요 시 ConfigMap/Secret) YAML 작성 완료
- [ ] scg Deployment+Service+ConfigMap(CORS_ALLOW_ORIGIN, AUTH_SERVICE_*/POST_SERVICE_* 등)+HTTPRoute(`sectionName: scg`, 필요 시 Secret) YAML 작성 완료 (아직 미적용)
- [ ] client Deployment+Service+HTTPRoute(`sectionName: client`) YAML 작성 완료 (아직 미적용)
- [ ] SCG 내부 라우팅 설정(ConfigMap 환경변수 기반)을 k8s 서비스명 기준으로 반영

---

## 6. Part C — GitOps (ArgoCD + 자동배포 완성)

> 상세 절차: [04-argocd-gitops-setup.md](04-argocd-gitops-setup.md)

Part B에서 준비한 YAML을 `k8s-manifests` 레포에 올리고, ArgoCD가 이를 **최초로** 클러스터에 적용합니다. 이어서 Part A의 워크플로우에 "매니페스트 업데이트" 스텝을 추가해 CI/CD 전체 흐름을 완성합니다.

**완료 체크포인트**
- [ ] `k8s-manifests` 레포에 Part B YAML push, ArgoCD 설치 및 Repository 연결
- [ ] `k8s-setting` 레포의 GatewayClass/Gateway/Application 매니페스트로 Application 4개 생성 → 최초 Sync로 Pod 생성 확인 (`Synced`/`Healthy`)
- [ ] `localhost` 경유 외부 접근 성공 (API는 `:8080`, 화면은 `:5173`), auth/post 격리 확인
- [ ] selfHeal 동작 확인
- [ ] 4개 앱 레포 워크플로우에 "매니페스트 업데이트" 스텝 추가 (fine-grained PAT 1개만 사용)
- [ ] End-to-End 검증: 코드 push 한 번으로 해당 서비스만 자동 재배포
- [ ] 롤백 실습 + MSA 장애 격리 검증 (한 서비스 장애가 다른 서비스에 영향 없음 확인)

---

## 7. 진행 전 준비물 체크리스트

- [ ] 트레이니별 GitHub 개인 계정 (Public repo 6개 생성 권한 확인)
- [ ] SCG/auth-service/post-service/client 소스코드 (기존 MSA 프로젝트 코드 준비 완료 여부)
- [ ] client의 SPA fallback 설정(`nginx.conf`의 `try_files ... /index.html`)이 운영 환경에 이미 있다면 그대로 재사용
- [ ] Docker Desktop + kind 멀티노드 클러스터 각자 PC에 사전 세팅 완료 확인 (5173, 8080 두 포트 모두 `extraPortMappings`로 호스트에 매핑되어 `localhost:5173`(화면), `localhost:8080`(API) 접근 가능한 상태)
- [ ] fine-grained PAT 발급 절차 안내자료 (Part C에서 딱 1개만 필요 — `k8s-manifests` 레포 한정 Contents R/W)
