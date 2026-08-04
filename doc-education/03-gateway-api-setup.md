# 🌐 [교육용] Part B — Gateway API 실습 가이드

Gateway API + NGINX Gateway Fabric을 설치하고, `scg`와 `client`를 같은 Gateway에서 **포트로 구분**해 노출하며, `auth-service`/`post-service`는 ClusterIP로만 존재하도록 매니페스트를 준비합니다.

- Gateway 리스너 `client`(5173) → `client` (화면)
- Gateway 리스너 `scg`(8080) → `scg` (API 요청)

포트가 다르면 브라우저 기준으로 origin이 다르므로, **경로 기반 방식과 달리 CORS 설정이 필요**합니다. SCG가 자신의 ConfigMap(`CORS_ALLOW_ORIGIN`)으로 client origin(`http://localhost:5173`)을 허용하는 방식으로 처리합니다.

이 파트는 두 성격이 다른 작업으로 나뉩니다.

- **B-1. 클러스터 인프라 설치** — Gateway API CRD, NGINX Gateway Fabric, GatewayClass, Gateway. `k8s-setting` 레포에 담습니다. 특정 앱 이미지와 무관한 **클러스터 전체 1회성 설정**이라 [Part A](02-ci-github-actions.md)와 순서 상관없이 진행 가능합니다.
- **B-2. 앱 매니페스트 작성** — Deployment/Service/HTTPRoute YAML. `k8s-manifests` 레포에 담습니다. 이미지 태그를 명시해야 하므로 **Part A에서 최소 1번 push가 끝난 뒤**에만 작성 가능합니다.

> ⚠️ **이 파트에서는 `kubectl apply`로 앱 매니페스트를 직접 적용하지 않습니다.** B-2에서 작성하는 YAML은 파일로만 준비해두고, [Part C](04-argocd-gitops-setup.md)에서 ArgoCD가 이 파일들을 Git 기준으로 **처음** 클러스터에 적용합니다. 여기서 미리 적용해버리면 Part C에서 ArgoCD로 갈아타는 과정이 "왜 또 똑같은 걸 하지?"로 느껴지게 됩니다.

> 전제: 클러스터가 kind로 생성될 때 5173, 8080 두 포트가 각각 `extraPortMappings`로 호스트에 이미 매핑되어 있어, `http://localhost:5173`(화면), `http://localhost:8080`(API)으로 접근 가능한 상태입니다. (프로덕션의 [00-k8s-architecture-overview.md](../situation/00-k8s-architecture-overview.md)와 달리 실도메인/443/cert-manager는 사용하지 않습니다.)

---

## B-1. 클러스터 인프라 설치

### 1. 사전 확인

```bash
kubectl get nodes
kubectl get pods -A
helm version
```

노드가 모두 `Ready`, `helm`이 설치되어 있는지 확인합니다. (컨트롤플레인 노드는 기본적으로 `node-role.kubernetes.io/control-plane:NoSchedule` taint가 걸려 있어 일반 워크로드는 자동으로 워커 노드에만 배치됩니다. 이번 실습은 프로덕션과 달리 마스터에 별도 컴포넌트를 올릴 필요가 없으므로 **taint를 제거하지 않습니다.**)

### 2. Gateway API CRD 설치

```bash
kubectl apply --server-side -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.1/standard-install.yaml

```

### 3. NGINX Gateway Fabric 설치 (HTTP만, 포트 5173/8080)

컨트롤플레인 노드의 5173, 8080 포트에 HostNetwork로 바인딩합니다. (443/TLS는 사용하지 않으므로 생략)

```bash
kubectl create namespace nginx-gateway

helm install ngf oci://ghcr.io/nginx/charts/nginx-gateway-fabric \
  --namespace nginx-gateway \
  --version 2.6.5 \
  --timeout 10m \
  --set nginx.kind=daemonSet \
  --set nginx.container.hostPorts[0].port=5173 \
  --set nginx.container.hostPorts[0].containerPort=5173 \
  --set nginx.container.hostPorts[1].port=8080 \
  --set nginx.container.hostPorts[1].containerPort=8080 \
  --set nginx.nodeSelector."node-role\\.kubernetes\\.io/control-plane"="" \
  --set-json 'nginx.pod.tolerations=[{"key":"node-role.kubernetes.io/control-plane","operator":"Exists","effect":"NoSchedule"}]'
```

**설치 확인**

```bash
kubectl get pods -n nginx-gateway
```

### 4. GatewayClass 생성

`k8s-setting/01-gateway-class.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: gateway.nginx.org/nginx-gateway-controller
```

```bash
kubectl apply -f k8s-setting/01-gateway-class.yaml
kubectl get gatewayclass
# ACCEPTED: True 여야 함
```

### 5. Gateway 생성 (HTTP 리스너 2개: client/5173, scg/8080)

`k8s-setting/02-gateway.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: gateway
  namespace: nginx-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: client
    port: 5173
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
  - name: scg
    port: 8080
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
```

```bash
kubectl apply -f k8s-setting/02-gateway.yaml
kubectl get gateway -n nginx-gateway
```

> 리스너 이름(`client`, `scg`)은 뒤에서 각 서비스의 HTTPRoute가 `parentRefs.sectionName`으로 지정할 값입니다. path 매칭 없이 **어느 리스너(포트)로 들어왔는지**만으로 트래픽이 갈립니다.

**B-1 완료 체크포인트**: `GatewayClass`가 `ACCEPTED: True`이면 인프라 설치는 끝입니다. 아직 앱이 없으므로 실제 트래픽 테스트는 Part C에서 합니다.

### 6. 네임스페이스 생성

```bash
kubectl create namespace msa4-meerkatgram
```

이후 모든 Deployment/Service/HTTPRoute는 `msa4-meerkatgram` 네임스페이스 기준으로 작성합니다.

---

## B-2. 앱 매니페스트 작성 (Part A 완료 후 진행)

> [Part A](02-ci-github-actions.md)에서 최소 1번 이미지가 push되어 `ghcr.io/<github-username>/scg:1` 같은 실제 태그가 존재해야 아래 YAML을 정확히 작성할 수 있습니다.
>
> ⚠️ 아래 YAML의 `<github-username>`은 **반드시 소문자**로 입력합니다. GitHub 프로필에 대문자가 섞여 있어도(`ByungjooPark` 등) ghcr.io 이미지 경로는 소문자만 허용합니다 ([Part A](02-ci-github-actions.md) 2절 참고). Packages 탭에서 본인 이미지의 실제 URL을 복사해오면 정확합니다.

`k8s-manifests` 레포는 [Part C](04-argocd-gitops-setup.md)에서 만들 것이므로, 지금은 로컬 작업 디렉토리(예: `k8s/`)에 파일만 작성해둡니다.

### 6.1 환경변수 전략 — ConfigMap / Secret, 그리고 MinIO 기능 비활성화

이 실습의 `k8s-manifests` 레포는 **Public을 유지**합니다(훈련생 포트폴리오 용도). 그래서 환경변수는 성격에 따라 나눠서 다룹니다.

| 값의 성격 | 사용할 오브젝트 | 예시 |
| :--- | :--- | :--- |
| 민감하지 않은 설정값 | `ConfigMap` | `SPRING_PROFILES_ACTIVE`, 내부 서비스 URL 등 |
| JWT 서명키처럼 "형태상" 민감한 값 | `Secret` | `JWT_SECRET` |

> **왜 Public인데도 `Secret`을 그대로 쓰는가?** 지금 쓰는 값은 실습용 더미 키라 Public repo에 있어도 실질적 피해는 없습니다. 하지만 **k8s `Secret`이라는 오브젝트 자체의 사용법(`stringData` 구조, 컨테이너에 주입하는 방식)은 운영 환경과 동일하게 익혀둡니다.** 실제 서비스라면 이 레포는 반드시 Private이어야 하고, 이 값도 진짜 비밀로 취급해야 합니다 — 지금은 학습 목적으로 예외를 두는 것뿐입니다.

> **참고 — Kakao OAuth2 소셜 로그인**: auth-service엔 카카오 로그인 연동(`KAKAO_CLIENT_ID`, `KAKAO_CLIENT_SECRET`, 로그인 성공 후 프론트로 리다이렉트할 `FRONTEND_CALLBACK_URI`)이 이미 구현되어 있을 수 있습니다. 이 커리큘럼은 CI/CD 파이프라인 자체가 목표라 카카오 로그인 실습은 다루지 않지만, 코드가 이 값들을 기동 시점에 참조한다면 Secret/ConfigMap에 더미 값이라도 채워야 auth-service가 정상 기동합니다.

> ⚠️ **MinIO 연동(파일 업로드) — 실습 스코프는 제외, 그러나 기동은 보장**: 이 커리큘럼은 MinIO 파일 업로드를 실제로 써보는 실습은 제외했습니다([01번 문서](01-msa-deploy-curriculum.md) 2.1절). 다만 서비스 코드가 기동 시점에 `MINIO_ENDPOINT`, `MINIO_ACCESS_KEY` 등을 참조해 커넥션을 맺는 로직이 있다면, 이 환경변수가 없을 때 애플리케이션 자체가 기동 실패할 수 있습니다. 코드를 수정(비활성화)하는 대신, **더미/미사용 값을 그대로 Secret에 채워** 컨테이너가 정상 기동하도록만 맞춥니다 — 파일 업로드 "기능"을 눌러보는 실습만 하지 않을 뿐, 코드는 그대로 둡니다.

> **client는 보통 ConfigMap/Secret이 필요 없습니다.** API 호출 주소(`http://localhost:8080` 같은 절대경로)는 k8s Secret이 아니라 [Part A](02-ci-github-actions.md)의 빌드 단계(Vite의 `.env`, `VITE_API_BASE_URL` 등)에서 이미 정적 산출물에 고정됩니다 — 빌드 시점에 굳어버린 값이라 배포 시점(k8s)에 바꿀 수 없기 때문입니다.

### 7. auth-service (ClusterIP만 — 외부 미노출)

`k8s/auth-service/configmap.yaml` — 비민감 설정값

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: auth-service-config
  namespace: msa4-meerkatgram
data:
  # <변경필요> 실제 프로젝트에 맞는 값
  APP_DOMAIN: "http://auth-service:80"
  APP_PORT: "8080"
  APP_DESCRIPTION: "Auth Server"
  # (해당 시) 카카오 로그인 성공 후 프론트로 리다이렉트할 주소
  FRONTEND_CALLBACK_URI: "http://localhost:5173/oauth2/callback"
  GATEWAY_URI: "http://localhost:8080"
```

> `SPRING_PROFILES_ACTIVE` 같이 프로필을 고정하는 값은 ConfigMap이 아니라 아래 Deployment의 `env`에 직접 넣습니다(서비스마다 항상 같은 값이라 ConfigMap으로 분리할 이유가 없음).

`k8s/auth-service/secret.yaml` — JWT 서명키 등 (실습용 더미 값, 6.1절 참고)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: auth-service-secret
  namespace: msa4-meerkatgram
type: Opaque
stringData:
  JWT_SECRET: "<변경필요-실습용-더미값>"
  # (해당 시) 카카오 OAuth2 — 로그인 실습은 안 하지만 코드가 참조하면 더미값이라도 필요
  KAKAO_CLIENT_ID: "<변경필요-실습용-더미값>"
  KAKAO_CLIENT_SECRET: "<변경필요-실습용-더미값>"
  # (해당 시) MinIO — 실습 스코프 제외, 기동 실패 방지용 더미값 (6.1절 참고)
  MINIO_ENDPOINT: "http://localhost:6601"
  MINIO_BUCKET: "dummy"
  MINIO_ACCESS_KEY: "dummy"
  MINIO_SECRET_KEY: "dummy"
```

> 여기 나열한 키 이름은 예시입니다. 본인 프로젝트 코드가 실제로 참조하는 환경변수 이름과 정확히 일치해야 합니다.

`k8s/auth-service/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth-service
  namespace: msa4-meerkatgram
spec:
  replicas: 1
  selector:
    matchLabels:
      app: auth-service
  template:
    metadata:
      labels:
        app: auth-service
    spec:
      containers:
      - name: auth-service
        # <변경필요> Part A에서 push한 태그
        image: ghcr.io/<github-username>/auth-service:1
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        envFrom:
        - configMapRef:
            name: auth-service-config
        - secretRef:
            name: auth-service-secret
```

`k8s/auth-service/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: auth-service
  namespace: msa4-meerkatgram
spec:
  type: ClusterIP
  selector:
    app: auth-service
  ports:
  - port: 80
    targetPort: 8080
```

### 8. post-service (ClusterIP만 — 외부 미노출)

`k8s/post-service/deployment.yaml` / `k8s/post-service/service.yaml`은 auth-service와 동일 패턴으로 이름만 `post-service`로 교체하여 작성합니다.

`configmap.yaml`/`secret.yaml`도 auth-service와 같은 이유로 대부분 필요합니다 — 예를 들어 post-service가 DB에 직접 접속하거나 MinIO로 게시글 이미지를 업로드한다면(6.1절 참고, 이번 실습에선 더미값) `post-service-config`/`post-service-secret`을 auth-service와 같은 패턴(`APP_DOMAIN`, `APP_PORT`, `APP_DESCRIPTION`, `GATEWAY_URI` 등 ConfigMap / `DB_*`, `MINIO_*` 등 Secret)으로 만듭니다. post-service가 JWT를 직접 검증한다면 auth-service와 **동일한 `JWT_SECRET` 값**으로 `post-service-secret`을 만들어야 토큰 검증이 일치합니다.

### 9. scg (외부 노출 대상 — API 경로)

SCG는 다른 두 서비스와 달리 **ConfigMap/Secret이 사실상 필수**입니다. CORS 허용 origin, auth/post-service로의 내부 라우팅 대상·경로가 전부 ConfigMap 환경변수로 외부화되어 있고(11절 참고), 게이트웨이 레벨의 JWT 검증(AuthFilter)도 모든 요청에 항상 적용되기 때문에 `JWT_SECRET`도 항상 필요합니다.

`k8s/scg/configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: scg-config
  namespace: msa4-meerkatgram
data:
  APP_DOMAIN: "http://localhost:8080"
  APP_PORT: "8080"
  APP_DESCRIPTION: "Gateway"
  # client가 노출되는 origin과 반드시 일치해야 함
  CORS_ALLOW_ORIGIN: "http://localhost:5173"
  # <변경필요> k8s Service DNS 이름
  AUTH_SERVICE_URI: "http://auth-service:80"
  AUTH_SERVICE_PREDICATE: "/api/auth/**"
  AUTH_SERVICE_NAME: "auth-service"
  # <변경필요> k8s Service DNS 이름
  POST_SERVICE_URI: "http://post-service:80"
  POST_SERVICE_PREDICATE: "/api/posts/**"
  POST_SERVICE_NAME: "post-service"
```

`k8s/scg/secret.yaml` — auth-service와 **동일한 `JWT_SECRET` 값**(토큰 검증 일치)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: scg-secret
  namespace: msa4-meerkatgram
type: Opaque
stringData:
  JWT_SECRET: "<변경필요-auth-service와-동일한-값>"
```

`k8s/scg/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: scg
  namespace: msa4-meerkatgram
spec:
  replicas: 1
  selector:
    matchLabels:
      app: scg
  template:
    metadata:
      labels:
        app: scg
    spec:
      containers:
      - name: scg
        # <변경필요> Part A에서 push한 태그
        image: ghcr.io/<github-username>/scg:1
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        envFrom:
        - configMapRef:
            name: scg-config
        - secretRef:
            name: scg-secret
```

`k8s/scg/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: scg
  namespace: msa4-meerkatgram
spec:
  type: ClusterIP
  selector:
    app: scg
  ports:
  - port: 8080
    targetPort: 8080
```

### 10. HTTPRoute 생성 — scg (`scg` 리스너 전체 매칭)

> **이 라우팅 설정은 NGF에 직접 하는 게 아닙니다.** `HTTPRoute`라는 선언적 K8s 오브젝트로 규칙을 만들어 Gateway에 붙여두면(`parentRefs`), NGF(Gateway 컨트롤러)가 이를 감지해서 내부 nginx 설정에 자동 반영합니다. cert-manager가 `Certificate` 오브젝트를 보고 알아서 인증서를 발급하는 것과 같은 패턴입니다.

`k8s/scg/httproute.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: scg-route
  namespace: msa4-meerkatgram
spec:
  parentRefs:
  - name: gateway
    namespace: nginx-gateway
    sectionName: scg
  rules:
  - backendRefs:
    - name: scg
      port: 8080
```

> `parentRefs.sectionName: scg`로 5절에서 만든 `scg` 리스너(8080)에만 이 라우트를 붙입니다. `rules.matches`(path 매칭)를 쓰지 않으므로 이 리스너로 들어오는 요청은 전부 `scg` Service로 갑니다 — path 기반 라우팅이 아니라 **포트(리스너) 기반 라우팅**입니다.
>
> auth-service/post-service에는 **HTTPRoute를 만들지 않습니다.** 이게 곧 "외부 미노출"을 만드는 방법입니다.

### 10.5 HTTPRoute 생성 — client (`client` 리스너 전체 매칭)

`k8s/client/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: client
  namespace: msa4-meerkatgram
spec:
  replicas: 1
  selector:
    matchLabels:
      app: client
  template:
    metadata:
      labels:
        app: client
    spec:
      containers:
      - name: client
        # <변경필요> Part A에서 push한 태그
        image: ghcr.io/<github-username>/client:1
        ports:
        - containerPort: 80
```

`k8s/client/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: client
  namespace: msa4-meerkatgram
spec:
  type: ClusterIP
  selector:
    app: client
  ports:
  - port: 5173
    targetPort: 80
```

`k8s/client/httproute.yaml`

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: client-route
  namespace: msa4-meerkatgram
spec:
  parentRefs:
  - name: gateway
    namespace: nginx-gateway
    sectionName: client
  rules:
  - backendRefs:
    - name: client
      port: 5173
```

> **경로 우선순위를 신경 쓸 필요가 없습니다**: scg와 client는 애초에 **서로 다른 리스너(포트)** 에 붙어 있어 경로가 겹칠 일이 없습니다. 대신 kind 클러스터에 5173, 8080 두 포트가 모두 `extraPortMappings`로 열려 있어야 각각 접근 가능합니다.

### 11. SCG 내부 라우팅 설정 (+ CORS)

각자 프로젝트의 `application.yml`(Spring Cloud Gateway 라우팅 설정)에서 대상 URI·매칭 경로를 **하드코딩하지 말고 환경변수로 외부화**합니다. 값은 9절의 `scg-config` ConfigMap에서 주입되므로, 로컬 개발/배포 환경이 바뀌어도 코드 수정 없이 값만 갈아 끼울 수 있습니다.

```yaml
spring:
  cloud:
    gateway:
      server:
        webflux:
          globalcors:
            cors-configurations:
              '[/**]':
                # <변경필요> client가 노출되는 origin
                allowed-origins: ${CORS_ALLOW_ORIGIN}
                allowed-methods: ["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"]
                allowed-headers: "*"
                allow-credentials: true
          routes:
            - id: ${AUTH_SERVICE_NAME}
              # <변경필요> k8s Service DNS 이름
              uri: ${AUTH_SERVICE_URI}
              predicates:
                - Path=${AUTH_SERVICE_PREDICATE}
            - id: ${POST_SERVICE_NAME}
              # <변경필요> k8s Service DNS 이름
              uri: ${POST_SERVICE_URI}
              predicates:
                - Path=${POST_SERVICE_PREDICATE}
```

> `auth-service`, `post-service`는 `msa4-meerkatgram` 네임스페이스 안의 Service 이름으로, 같은 네임스페이스 안에서는 이 짧은 이름만으로 DNS 해석이 됩니다(CoreDNS). 수정 후 SCG 이미지를 다시 빌드/push합니다 (Part A 워크플로우가 자동으로 처리).
>
> **`CORS_ALLOW_ORIGIN`은 client가 노출되는 origin(`http://localhost:5173`)과 정확히 일치해야 합니다** — 스킴/호스트/포트 중 하나라도 다르면 브라우저가 API 응답을 차단합니다(포트 기반 라우팅이라 SCG와 client가 서로 다른 origin이기 때문에 필요한 설정입니다. 3장 참고).
>
> client 코드의 API 호출 주소도 절대경로(`http://localhost:8080`) 기준으로 맞춰져 있어야 합니다 ([Part A](02-ci-github-actions.md) 1.2절 참고).

---

## 12. 이 시점에서 할 수 있는 검증 / 할 수 없는 검증

### 12.1. 확인 가능 범위

| 확인 항목 | 지금 가능? |
| :--- | :--- |
| `kubectl get gatewayclass` → `ACCEPTED: True` | ✅ 가능 |
| `kubectl get gateway -n nginx-gateway` | ✅ 가능 |
| YAML 문법(`kubectl apply --dry-run=client -f ...`)만 검사 | ✅ 가능 |
| `curl http://localhost:8080/...`(scg), `curl http://localhost:5173/`(client) 실제 응답 확인 | ❌ 아직 불가 — Pod가 없으므로. [Part C](04-argocd-gitops-setup.md)에서 ArgoCD가 배포한 뒤 확인합니다 |
| auth/post 외부 접근 격리 확인 | ❌ 아직 불가 — 상동 |

### 12.2. 확인 방법 예시
| 확인 항목 | 명령어 | 설명 |
| :--- | :--- | :--- |
| 문법 검증 | `kubectl apply --dry-run=client -f .` | 클러스터에 아무 것도 만들지 않고 YAML 문법/필드만 체크 |
| 네임스페이스 생성 | `kubectl create namespace msa4-meerkatgram` |  |
| 직접 적용 | `kubectl apply -f auth-service/` | configmap/secret/deployment/service를 로컬 클러스터에 실제로 생성 |
| 확인 | `kubectl get pods -n msa4-meerkatgram`, `kubectl describe pod ...`, `kubectl logs ...` | Pod가 뜨는지, 이미지 pull이 되는지, 환경변수가 제대로 들어갔는지 확인 |
| 접속 테스트 | `kubectl port-forward svc/auth-service 8080:80 -n msa4-meerkatgram` |  로컬에서 API 호출 |

---

## 13. 트러블슈팅

| 증상 | 원인 | 확인 명령 |
| :--- | :--- | :--- |
| `GatewayClass` `ACCEPTED: False` | 컨트롤러(NGF) 파드 미기동 | `kubectl get pods -n nginx-gateway`, `kubectl describe gatewayclass nginx` |
| NGF 파드가 `Pending` | 컨트롤플레인 노드 nodeSelector/toleration 불일치 | `kubectl describe pod <ngf-pod> -n nginx-gateway`의 Events 확인 |
| YAML `apply --dry-run` 오류 | 들여쓰기/필드명 오타 | 에러 메시지의 필드 경로 확인 |
| (Part A 스모크 테스트 시) 컨테이너가 즉시 종료됨 | MinIO 등 미사용 외부 연동의 환경변수 누락으로 앱 기동 실패 | 로그에서 `MINIO_*` 등 없는 환경변수 참조 여부 확인, 6.1절 참고해 더미값으로 채우기 |
| (Part C 배포 후) client는 되는데 scg(API) 접속 안 됨(또는 반대) | HTTPRoute의 `parentRefs.sectionName`이 Gateway 리스너 이름과 다르거나, kind `extraPortMappings`에 5173/8080 중 하나가 안 걸려 있음 | `kubectl get httproute -n msa4-meerkatgram -o yaml`로 `sectionName` 확인, kind 클러스터 설정의 `extraPortMappings` 확인 |
| (Part C 배포 후) 브라우저 콘솔에 CORS 에러 | scg `scg-config`의 `CORS_ALLOW_ORIGIN` 값이 실제 client 접속 origin과 다름(포트 불일치 등) | `kubectl get configmap scg-config -n msa4-meerkatgram -o yaml`로 값과 브라우저 주소창 origin이 정확히 일치하는지 확인 |
| (Part C 배포 후) 화면 새로고침 시 404 | client `nginx.conf`에 SPA fallback(`try_files ... /index.html`) 누락 | 이미지 안 `/etc/nginx/conf.d/default.conf` 내용 확인 |

---

## 14. 작업 체크리스트

| 단계 | 내용 | 완료 |
| :--- | :--- | :---: |
| B-1-1 | Gateway API CRD 설치 (`k8s-setting` 레포) | ☐ |
| B-1-2 | NGINX Gateway Fabric 설치 (포트 5173, 8080) | ☐ |
| B-1-3 | GatewayClass / Gateway(리스너 2개: client/5173, scg/8080) 생성 | ☐ |
| B-1-4 | `msa4-meerkatgram` 네임스페이스 생성 | ☐ |
| B-2-0 | MinIO 관련 환경변수는 더미값으로 채워 기동만 보장 (코드에 있는 경우) | ☐ |
| B-2-1 | auth-service ConfigMap/Secret/Deployment/Service YAML 작성 | ☐ |
| B-2-2 | post-service Deployment/Service(+필요시 ConfigMap/Secret) YAML 작성 | ☐ |
| B-2-3 | scg Deployment+Service+ConfigMap(CORS_ALLOW_ORIGIN 등)+Secret(JWT_SECRET)+HTTPRoute(`sectionName: scg`) YAML 작성 | ☐ |
| B-2-4 | client Deployment+Service+HTTPRoute(`sectionName: client`) YAML 작성 | ☐ |
| B-2-5 | SCG 내부 라우팅 설정(ConfigMap 환경변수 기반) 반영, client API 호출 주소(절대경로) 확인 | ☐ |

다음 단계: [04-argocd-gitops-setup.md](04-argocd-gitops-setup.md) — 지금 작성한 YAML을 `k8s-manifests` 레포에 올리고, ArgoCD가 처음으로 클러스터에 적용합니다.
