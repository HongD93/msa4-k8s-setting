# 🐳 [교육용] Part A — CI (GitHub Actions + ghcr.io) 실습 가이드

`scg`, `auth-service`, `post-service`, `client` 4개 레포 각각에 dockerfile을 작성하고, **GitHub Actions로 처음부터 자동화된** 빌드+push 파이프라인을 만듭니다. 이 단계에서는 아직 k8s를 건드리지 않습니다.

---

## 0. Jenkinsfile ↔ GitHub Actions 문법 매핑

| 개념 | Jenkinsfile (Groovy DSL) | GitHub Actions (YAML) |
| :--- | :--- | :--- |
| 파이프라인 정의 파일 | `Jenkinsfile` | `.github/workflows/*.yml` |
| 최상위 블록 | `pipeline { ... }` | (파일 전체) |
| 실행 환경 지정 | `agent any` | `runs-on: ubuntu-latest` |
| 트리거 | Build Triggers (webhook 체크박스) | `on: push: branches: [...]` |
| 단계 묶음 | `stages { stage('이름') { ... } }` | `jobs: <id>: steps: [...]` |
| 개별 명령 | `steps { sh "..." }` | `- run: ...` |
| 환경 변수 | `environment { VAR = "..." }` | `env: VAR: ...` |
| 자격증명 사용 | `withCredentials([...]) { ... }` | `secrets.<NAME>` |
| 후처리 | `post { always { ... } }` | `if: always()` |

**핵심 차이 — 자격증명 소스**: Jenkins는 모든 자격증명을 Credentials 화면에 동일하게 등록하지만, GitHub Actions는 **같은 레포 안에서의 작업(이번 파트의 ghcr.io push)은 자동 주입되는 `secrets.GITHUB_TOKEN`으로 충분**합니다. 별도 토큰 발급이 필요 없습니다. (다른 레포에 push해야 하는 경우에만 PAT가 필요한데, 그건 Part C에서 다룹니다.)

---

## 1. dockerfile 작성

> 파일명은 대문자 `Dockerfile`이 아니라 **소문자 `dockerfile`** 로 저장합니다(이 프로젝트 전체 컨벤션). `docker build`는 대소문자 구분 없이 이 파일을 인식합니다.

### 1.1 scg / auth-service / post-service (Gradle + Spring Boot 공통 패턴)

`scg`, `auth-service`, `post-service` 레포 루트에 각각 동일한 패턴으로 작성합니다. (Spring Boot 4.1 + Java 21 기준)

```dockerfile
# --- 1단계: 빌드 ---
FROM gradle:8-jdk21-alpine AS builder
WORKDIR /app
COPY . .
RUN gradle bootJar --no-daemon -x test

# --- 2단계: 실행 ---
FROM eclipse-temurin:21-jre
WORKDIR /app
ENV TZ=Asia/Seoul
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
COPY --from=builder /app/build/libs/*.jar app.jar
EXPOSE 8080
CMD ["java", "-jar", "app.jar"]

```

> 서비스별로 Gradle 모듈 구조나 빌드 명령이 다르면 `RUN gradle ...` 줄만 조정합니다.

### 1.2 client (Node 빌드 + nginx 정적 서빙)

`client` 레포 루트에 작성합니다. 정적 빌드 산출물만 서빙하면 되는 SPA이므로 Node 프로세스를 계속 띄워두지 않고, 빌드 후 nginx로 넘깁니다 (운영 [05-argocd-cicd-build.md](../situation/05-argocd-cicd-build.md) 6.2절과 동일 패턴).

```dockerfile
# --- 1단계: 빌드 ---
FROM node:24-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# --- 2단계: 실행 ---
FROM nginx:alpine
RUN apk add --no-cache tzdata && \
    ln -snf /usr/share/zoneinfo/Asia/Seoul /etc/localtime && \
    echo "Asia/Seoul" > /etc/timezone
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]

```

`nginx.conf`는 **운영 환경에 이미 있는 SPA fallback 설정을 그대로 가져와 씁니다** (새로 작성할 필요 없음). 클라이언트 라우팅(`/some-page` 새로고침 시 404 방지)을 위해 아래 형태여야 합니다.

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

> API 호출 주소: [Part B](03-gateway-api-setup.md)에서 client(5173)와 scg(8080)는 Gateway의 **서로 다른 포트**로 노출됩니다. 즉 브라우저 기준 origin이 다르므로, client는 API를 **절대경로**(`http://localhost:8080/...`)로 호출해야 하고 SCG가 CORS를 허용해야 합니다. Vite 프로젝트라면 빌드 시점에 `VITE_API_BASE_URL` 같은 환경변수로 이 주소를 주입해두면(`.env`), 로컬/실습 환경이 바뀌어도 코드 수정 없이 값만 바꿀 수 있습니다.

### 1.3 로컬 빌드 및 기동 스모크 테스트

각 레포 디렉토리에서 실행합니다. (아직 ghcr.io에 push하지 않습니다 — 이 테스트는 "이미지 자체가 제대로 뜨는지"만 확인하는 목적입니다.)

```bash
# 예시: scg 레포
docker build -t scg:local .
docker run --rm -p 8080:8080 --env-file .env scg:local

# 다른 터미널에서 헬스체크 (엔드포인트는 각자 프로젝트에 맞게)
curl http://localhost:8080/actuator/health
```

```bash
# client 레포
docker build -t client:local .
docker run --rm -p 5173:80 --env-file .env client:local

# 브라우저 또는 curl로 확인
curl http://localhost:5173
```

4개 서비스 모두 동일하게 반복합니다. (포트는 로컬에서 겹치지 않게 바꿔서 확인해도 무방)

---

## 2. GitHub Actions 워크플로우 작성 (build + push만)

### 2.0 ⚠️ 레포 설정 — Workflow permissions (필수, 4개 레포 모두)

워크플로우 파일만 push하면 대부분 그냥 동작하지만, **딱 하나 미리 바꿔둬야 하는 레포 설정**이 있습니다.

GitHub은 레포마다 `GITHUB_TOKEN`이 가질 수 있는 권한의 **상한선**을 별도로 관리합니다. 이 상한선이 "읽기 전용"으로 되어 있으면, 아래 2절 워크플로우 YAML에 `permissions: packages: write`를 아무리 선언해도 그 상한을 넘을 수 없어서 `docker push`가 `denied`로 실패합니다. YAML의 `permissions:`는 이 레포 설정 **범위 안에서만** 유효합니다.

`scg`, `auth-service`, `post-service`, `client` 레포 각각에서:

1. 레포 → **Settings → Actions → General**
2. 맨 아래 **Workflow permissions** 항목으로 스크롤
3. **Read and write permissions** 선택
4. **Save**

> 이건 레포 생성 시 GitHub이 기본으로 "Read-only"를 걸어두기 때문에 필요한 일회성 설정입니다. Actions 자체(Settings → Actions → General → Actions permissions)는 개인 레포라면 기본적으로 켜져 있어 별도로 켤 필요는 없습니다.

이 설정을 4개 레포 모두에서 마친 뒤 아래 워크플로우를 작성합니다.

`scg` 레포에 `.github/workflows/deploy.yml`:

```yaml
# Actions 탭 목록에 표시될 워크플로우 이름 (자유롭게 지정 가능)
name: Build and Deploy

on:
  # "push 이벤트가 발생하면" 트리거
  push:
    # 그중에서도 main 브랜치로의 push만 (다른 브랜치 push는 무시)
    branches: [main]

env:
  # <변경필요> ghcr.io에 올릴 "이미지 이름만" (예: scg). 태그(:v1 등)는 절대 붙이지 않기 — 태그는 아래 build 단계에서 실행 번호로 자동 결정됨. 여기에 태그를 붙이면 최종 이미지 참조가 "이름:태그:번호"가 되어 콜론이 2개가 되고 invalid tag 에러가 남
  IMAGE_NAME: scg

# 이 워크플로우 실행 중 GITHUB_TOKEN에 부여할 권한
permissions:
  # 이 레포 코드를 checkout하기 위한 최소 권한
  contents: read
  # ghcr.io(GitHub Packages)에 이미지를 push하기 위한 권한 (2.0절 레포 설정이 선행되어야 실제로 적용됨)
  packages: write

jobs:
  # job의 식별자 (임의로 지정 가능, Actions 로그에 표시됨)
  build-and-push:
    # 이 job을 실행할 GitHub 호스팅 러너의 OS/이미지
    runs-on: ubuntu-latest
    steps:
      # Actions 로그에 표시될 step 이름
      - name: Checkout
        # 현재 레포 코드를 러너로 내려받는 공식 액션 (버전 선택 이유는 위 안내 참고)
        uses: actions/checkout@v7

      - name: Log in to ghcr.io
        # Docker 레지스트리 로그인을 대신 처리해주는 공식 액션
        uses: docker/login-action@v4
        # 액션에 넘기는 입력값들
        with:
          # 로그인할 대상 레지스트리 주소
          registry: ghcr.io
          # 이 워크플로우를 트리거한(=push한) 사용자의 GitHub ID
          username: ${{ github.actor }}
          # 매 실행마다 GitHub이 자동 발급/주입하는 임시 토큰 — 별도 PAT 발급 불필요
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push image
        # 아래 셸 명령들을 순서대로 실행 (이 블록 안은 전부 bash 환경변수로 통일)
        run: |
          # 이미지 전체 경로 조립: ghcr.io/계정명(소문자)/이미지이름. IMAGE_NAME은 위 env: 블록 값이 자동으로 셸 변수로도 주입됨
          IMAGE=ghcr.io/${GITHUB_REPOSITORY_OWNER,,}/$IMAGE_NAME
          # GITHUB_RUN_NUMBER는 GitHub이 기본 제공하는 환경변수(=github.run_number와 동일한 값)
          docker build -t $IMAGE:$GITHUB_RUN_NUMBER .
          # 위에서 빌드한 이미지를 ghcr.io로 push
          docker push $IMAGE:$GITHUB_RUN_NUMBER

      # TODO (Part C에서 추가): 여기 아래에 k8s-manifests 레포를 checkout해서
      # 이미지 태그를 갱신하고 commit/push하는 "매니페스트 업데이트" 스텝을 추가할 예정입니다.
      # 지금은 build+push까지만 완성합니다.
```

> **버전을 `@v7`, `@v4`로 고정한 이유**: 특별한 기능이 필요해서가 아니라, 각 액션의 **현재 최신 메이저 버전**을 골라 고정(pin)한 것입니다. 메이저 버전 단위로 고정해두면 패치/마이너 업데이트는 자동으로 받으면서 호환성이 깨지는 변경은 막을 수 있습니다. 강의 시점엔 더 최신 메이저 버전이 나와 있을 수 있으니, [actions/checkout](https://github.com/actions/checkout/releases)·[docker/login-action](https://github.com/docker/login-action/releases) 릴리스 페이지에서 한 번 확인해보는 걸 권장합니다.
>
> `GITHUB_RUN_NUMBER`를 이미지 태그로 사용해 push할 때마다 태그가 자동으로 올라갑니다 (프로덕션의 `${BUILD_NUMBER}`와 동일한 역할). PAT 발급이나 `docker login` 같은 수동 작업이 전혀 없다는 점에 주목하세요 — 같은 레포 안 작업은 `GITHUB_TOKEN`만으로 충분합니다. `client` 레포도 dockerfile만 다를 뿐 이 워크플로우 구조는 완전히 동일합니다.
>
> **`${{ }}` vs `$VAR` 사용 규칙 (이 문서 전체에 일관 적용)**: `github.repository_owner`(표현식)와 `GITHUB_REPOSITORY_OWNER`(환경변수)처럼, GitHub은 주요 context 값들을 `${{ }}` 표현식과 `GITHUB_` + 대문자 스네이크케이스 환경변수 양쪽으로 항상 동시에 제공합니다. 이름만 다를 뿐 같은 값입니다. `env:` 블록에 우리가 직접 정의한 값(`IMAGE_NAME` 등)도 마찬가지로 `run:` 안에서 별도 선언 없이 바로 `$IMAGE_NAME`으로 접근할 수 있습니다. 그래서 이 문서에서는 아래 규칙으로 통일합니다.
> - **`run:` 블록 안 (bash가 실제로 실행)** → 항상 `$VAR`/`${VAR}` (환경변수). bash 문자열 조작(`${VAR,,}` 소문자 변환 등)도 여기서만 가능합니다.
> - **`uses:`, `with:`, `on:`, `permissions:` 같은 워크플로우 YAML 필드** → 항상 `${{ }}` (표현식). 이 필드들은 셸이 개입하기 **전에** GitHub Actions 엔진이 직접 처리하므로 애초에 bash 환경변수 문법이 통하지 않습니다. 아래 `docker/login-action`의 `with:` 블록(`${{ github.actor }}`, `${{ secrets.GITHUB_TOKEN }}`)이 그 예입니다.
>
> ghcr.io 이미지 경로는 **소문자만 허용**하는데 GitHub 계정명은 대문자를 포함할 수 있어(훈련생마다 다름), `GITHUB_REPOSITORY_OWNER`를 `${VAR,,}` bash 문법으로 소문자 변환합니다.
>
> **주의**: 사람이 직접 터미널에 입력하는 명령(4.1절의 `docker pull` 등)에서는 이 자동 변환이 적용되지 않으므로, `<github-username>` 자리에 **본인 GitHub ID를 소문자로** 입력해야 합니다.

### 2.1 auth-service / post-service / client 레포에 동일하게 적용

위 워크플로우를 그대로 복사해 나머지 3개 레포에 추가하되 `env.IMAGE_NAME`만 바꿉니다.

```yaml
env:
  IMAGE_NAME: auth-service
```

```yaml
env:
  IMAGE_NAME: post-service
```

```yaml
env:
  IMAGE_NAME: client
```

---

## 3. 첫 실행 확인 (4개 레포 모두)

```bash
git add .github/workflows/deploy.yml
git commit -m "Add CI workflow"
git push
```

레포 → **Actions** 탭에서 워크플로우 실행 상태 확인. 성공하면 ghcr.io에 새 태그(예: `scg:1`)가 생성됩니다. GitHub 프로필 → **Packages** 탭에서 4개 패키지(`scg`, `auth-service`, `post-service`, `client`) 확인.

---

## 4. ⚠️ 패키지 Visibility Public 전환 (필수, 4회 반복)

레포가 Public이어도 **처음 push된 ghcr.io 패키지는 기본값이 Private**입니다. 이 단계를 건너뛰면 나중에(Part C) K8s가 이미지를 pull하지 못해 `ImagePullBackOff`가 발생합니다.

패키지 4개 각각에 대해 반복합니다.

1. GitHub 프로필 → **Packages** → 해당 패키지 클릭 (예: `scg`)
2. 우측 **Package settings**
3. 맨 아래 **Danger Zone → Change package visibility**
4. **Public** 선택 → 확인창에 패키지 이름 입력 → **I understand the consequences, change package visibility**

### 4.1 검증

```bash
# 로그인 없이(익명) pull이 되는지 확인
docker logout ghcr.io
docker pull ghcr.io/<github-username>/scg:1
docker pull ghcr.io/<github-username>/auth-service:1
docker pull ghcr.io/<github-username>/post-service:1
docker pull ghcr.io/<github-username>/client:1
```

4개 모두 인증 없이 pull 성공하면 완료입니다.

---

## 5. 트러블슈팅

| 증상 | 원인 | 확인 |
| :--- | :--- | :--- |
| Actions에서 ghcr.io push 실패 (`denied`) | ① 레포의 **Workflow permissions**가 Read-only로 남아있음(2.0절) ② 워크플로우 파일의 `permissions.packages: write` 누락 | ①번을 먼저 확인 (가장 흔한 원인) → 그다음 워크플로우 파일 상단 `permissions` 블록 확인 |
| push해도 Actions 탭에 아무 실행 기록이 안 생김 | 레포의 기본 브랜치명이 `main`이 아님(`master` 등) | `git branch`로 현재 브랜치명 확인 후 워크플로우의 `on.push.branches` 값과 일치시키기 |
| `invalid tag "...:v1:2"` 같은 콜론 2개짜리 에러 | `env.IMAGE_NAME`에 태그(`:v1` 등)가 같이 들어감 | `IMAGE_NAME`은 태그 없이 순수 이미지 이름만 (예: `scg`) |
| `invalid reference format` (계정명에 대문자 포함) | ghcr.io 경로는 소문자만 허용, GitHub 계정명은 대문자 가능 | `${{ github.repository_owner }}` 대신 `${GITHUB_REPOSITORY_OWNER,,}` 사용했는지 확인 (본문 참고) |
| 워크플로우는 성공했는데 Packages 탭에 안 보임 | `IMAGE_NAME` 오타 | Actions 로그의 `docker push` 출력 URL 확인 |
| 로컬 `docker build` 실패 (scg/auth/post) | Gradle 빌드 자체 오류 (dockerfile 문제 아님) | `./gradlew bootJar`를 로컬에서 직접 실행해 원인 분리 |
| 로컬 `docker build`/`run` 실패 (client) | `npm run build` 산출물 경로가 `dist`가 아님(Vite 외 도구 사용 시) | 빌드 도구의 실제 출력 폴더명 확인 후 dockerfile의 `COPY --from=builder /app/dist` 경로 수정 |

---

## 6. 작업 체크리스트

| 단계 | 내용 | scg | auth-service | post-service | client |
| :--- | :--- | :---: | :---: | :---: | :---: |
| 1 | dockerfile 작성 | ☐ | ☐ | ☐ | ☐ |
| 2 | 로컬 build & run 스모크 테스트 | ☐ | ☐ | ☐ | ☐ |
| 3 | 레포 설정: Workflow permissions → Read and write (2.0절) | ☐ | ☐ | ☐ | ☐ |
| 4 | GitHub Actions 워크플로우 작성 (build+push) | ☐ | ☐ | ☐ | ☐ |
| 5 | push 후 Actions 성공, ghcr.io 이미지 확인 | ☐ | ☐ | ☐ | ☐ |
| 6 | Visibility Public 전환 | ☐ | ☐ | ☐ | ☐ |
| 7 | 익명 pull 검증 | ☐ | ☐ | ☐ | ☐ |

다음 단계: [03-gateway-api-setup.md](03-gateway-api-setup.md) — 이 이미지들을 클러스터에 배포할 준비(YAML 작성)를 합니다.
