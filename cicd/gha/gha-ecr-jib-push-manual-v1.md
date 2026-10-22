# GitHub Actions + Jib → ECR 이미지 자동 push 매뉴얼

> 상태: **운영 중 (ECR push + ss_k8s 태그 갱신 성공)** · 작성일: 2026-10-22
> 근거 문서: [gha-ecr-aws-auth-decision-v1.md](gha-ecr-aws-auth-decision-v1.md) (② GitHub OIDC + IAM Role 채택)

---

## 0. 요약

- `ss_be` 레포의 `main`에 push하면 GitHub Actions가 백엔드 이미지를 빌드해서 ECR에 push한다.
- 이어서 `ss_k8s` 레포의 `apps/ss-be/kustomization.yaml`에 있는 `newTag`를 새 이미지 태그로 바꿔 커밋한다. ArgoCD는 이 커밋을 보고 배포한다(ArgoCD 구성은 아직 범위 밖).
- 이미지 빌드는 **Jib**(Maven 플러그인)이 한다. **Dockerfile과 Docker 데몬이 필요 없다.**
- AWS 인증은 **GitHub OIDC + IAM Role**로 한다. GitHub에 저장하는 AWS 비밀 값은 없다.
- `ss_k8s` 쓰기 인증은 **GitHub App**으로 한다. 실행마다 `ss_k8s` Contents 쓰기만 가진 1시간짜리 토큰을 새로 받는다.
- 결과물
  - 이미지: `944296702922.dkr.ecr.ap-northeast-2.amazonaws.com/ss_be:<짧은 git SHA 7자리>`
  - `ss_k8s` 커밋: `ss_be 이미지 태그 <짧은 SHA>로 갱신` (작성자 `say-scene-ss-k8s-writer[bot]`)

---

## 1. 확정 값

| 항목 | 값 |
|---|---|
| GitHub 레포 | `say-scene/ss_be` |
| 실행 조건 | `main` push, 수동 실행(`workflow_dispatch`) |
| AWS 계정 / 리전 | `944296702922` / `ap-northeast-2` |
| OIDC Provider | `token.actions.githubusercontent.com`, Audience `sts.amazonaws.com` |
| IAM Role | `gha-ss-be-ecr-push` (`arn:aws:iam::944296702922:role/gha-ss-be-ecr-push`) |
| ECR 리포지토리 | `ss_be` (tag mutable) |
| 이미지 태그 | 짧은 git SHA 7자리 |
| CPU 아키텍처 | `amd64` (Jib 기본값) |
| 베이스 이미지 | `eclipse-temurin:21-jre` (Docker Hub) |
| Role ARN 보관 | GitHub Repository Variable `AWS_ROLE_ARN` |
| push 전 테스트 | 실행하지 않음 (추후 결정) |
| `main` 브랜치 보호 | 없음 (미니 프로젝트, 직접 push) |
| 태그 갱신 대상 | `say-scene/ss_k8s`의 `apps/ss-be/kustomization.yaml` → `images[ss_be].newTag` |
| GitHub App | `say-scene-ss-k8s-writer` (org 소유, `ss_k8s`에만 설치, Contents 쓰기) |
| App ID 보관 | Environment `ss-k8s-write`의 Variable `SS_K8S_APP_ID` |
| App 개인키 보관 | Environment `ss-k8s-write`의 Secret `SS_K8S_APP_PRIVATE_KEY` (`main` 브랜치만 허용) |

---

## 2. 전체 흐름

workflow는 job 두 개로 나뉜다.

| job | 하는 일 | 인증 |
|---|---|---|
| `push` | Jib으로 이미지를 빌드해서 ECR에 push | AWS OIDC |
| `update-tag` (`push` 성공 후) | `ss_k8s`의 `newTag`를 갱신해 commit, push | GitHub App 토큰 |

```mermaid
sequenceDiagram
  participant Dev as 개발자
  participant GH as GitHub Actions 러너
  participant IdP as GitHub OIDC 발급자
  participant STS as AWS STS
  participant Hub as Docker Hub
  participant ECR as Amazon ECR
  participant API as GitHub API
  participant K8S as ss_k8s 레포
  Dev->>GH: 1. ss_be main에 push
  Note over GH: push job
  GH->>GH: 2. checkout, JDK 21 설치
  GH->>IdP: 3. ID 토큰 요청
  IdP-->>GH: 4. 서명된 토큰 (sub=repo:say-scene@345080911/ss_be@1426408654:ref:refs/heads/main)
  GH->>STS: 5. AssumeRoleWithWebIdentity(gha-ss-be-ecr-push)
  STS-->>GH: 6. 임시 자격증명 (기본 1시간)
  GH->>ECR: 7. 로그인 → ~/.docker/config.json에 기록
  GH->>GH: 8. mvn compile (소스 컴파일)
  GH->>Hub: 9. Jib: 베이스 이미지 정보 조회
  GH->>ECR: 10. Jib: 바뀐 레이어 업로드 + ss_be:<sha> 태그 등록
  Note over GH: update-tag job (environment ss-k8s-write)
  GH->>API: 11. App 개인키로 설치 토큰 요청 (ss_k8s, contents:write)
  API-->>GH: 12. 1시간짜리 설치 토큰
  GH->>K8S: 13. 토큰으로 checkout
  GH->>K8S: 14. newTag를 <sha>로 바꿔 commit, push
```

### 2.1 Jib의 역할

- Jib은 별도 서버가 아니라 `pom.xml`에 등록된 **Maven build plugin**이다. GitHub 호스티드 러너(`ubuntu-latest`)에서 `mvn` 명령의 일부로 실행된다.
- 컨테이너 이미지는 **레이어 파일(tar)과 메타데이터(JSON)의 묶음**이다. Jib은 이것을 Java 코드로 직접 만들고, 레지스트리 API로 바로 업로드한다. 그래서 Dockerfile과 Docker 데몬이 필요 없다.
- 만들어진 이미지는 Docker나 Kubernetes가 그대로 실행할 수 있는 표준 이미지(Docker/OCI 형식)다.
- 앱을 아래 세 레이어로 나눈다. 코드만 바뀐 커밋이면 클래스 레이어만 다시 올라간다.
  1. 의존성 JAR
  2. 리소스
  3. 컴파일된 클래스
- ECR에 이미 있는 레이어는 업로드하지 않는다.

### 2.2 Jib 목표별 차이

| 목표 | Docker 데몬 | 용도 |
|---|---|---|
| `jib:build` | 필요 없음 | 레지스트리로 직접 push. **CI에서 쓰는 목표** |
| `jib:dockerBuild` | 필요함 | 로컬 Docker에 이미지 적재 |
| `jib:buildTar` | 필요 없음 | tar 파일로 저장 |

### 2.3 `ss_k8s` 쓰기 인증으로 GitHub App을 고른 이유

| 방식 | 실행마다 새 토큰 | 저장하는 장기 비밀 | 판단 |
|---|---|---|---|
| Deploy key | 아니요 (같은 키를 계속 씀) | SSH 개인키 | `say-scene` org 정책으로 deploy key가 꺼져 있음(`deploy_keys_enabled_for_repositories: false`) |
| Fine-grained PAT | 아니요 | PAT | 개인 계정에 묶이고 만료일을 관리해야 함 |
| **GitHub App** | **예 (1시간)** | App 개인키 | **채택.** GitHub 공식 기능 중 OIDC에 가장 가까운 최소 권한·최소 시간 |
| octo-sts (OIDC 교환) | 예 | 없음 | 외부 서비스(Chainguard) 의존. 미니 프로젝트에는 과함 |
| `GITHUB_TOKEN` | 예 | 없음 | 자기 레포에만 쓸 수 있어 `ss_k8s`에 push 불가 |

- App 개인키는 만료되지 않는 장기 비밀이다. 대신 이 키로는 **토큰 발급만** 할 수 있고, 키 자체를 `main` 실행에서만 꺼낼 수 있게 environment에 둔다(§5.2).

---

## 3. 레포 구성

### 3.1 `ss_be`: `pom.xml`의 Jib 설정

```xml
<plugin>
  <groupId>com.google.cloud.tools</groupId>
  <artifactId>jib-maven-plugin</artifactId>
  <version>3.4.5</version>
  <configuration>
    <from><image>eclipse-temurin:21-jre</image></from>
    <container>
      <ports><port>8080</port></ports>
      <user>10001</user>
    </container>
  </configuration>
</plugin>
```

- `<to><image>`(push 대상)는 pom에 적지 않는다. workflow에서 `-Dimage`로 넘긴다.

### 3.2 `ss_be`: `.github/workflows/ecr-push.yml`

```yaml
name: ecr-push

on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

# 연속 push 시 ss_k8s 태그 커밋이 엇갈리지 않도록 한 번에 하나씩 실행한다.
concurrency:
  group: ecr-push
  cancel-in-progress: false

jobs:
  push:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '21'
          cache: maven
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ vars.AWS_ROLE_ARN }}
          aws-region: ap-northeast-2
      - id: ecr
        uses: aws-actions/amazon-ecr-login@v2
      # Jib은 Docker 데몬 없이 ~/.docker/config.json의 ECR 로그인 정보로 직접 push한다.
      - run: mvn -B compile jib:build -Dimage=${{ steps.ecr.outputs.registry }}/ss_be:${GITHUB_SHA::7}

  # environment를 이 job에만 둔다. push job에 두면 OIDC sub가 environment 형식으로 바뀌어 AWS 신뢰 정책과 어긋난다.
  update-tag:
    needs: push
    runs-on: ubuntu-latest
    environment: ss-k8s-write
    steps:
      - id: app
        uses: actions/create-github-app-token@v2
        with:
          app-id: ${{ vars.SS_K8S_APP_ID }}
          private-key: ${{ secrets.SS_K8S_APP_PRIVATE_KEY }}
          owner: say-scene
          repositories: ss_k8s
          permission-contents: write
      - uses: actions/checkout@v4
        with:
          repository: say-scene/ss_k8s
          token: ${{ steps.app.outputs.token }}
      - name: Update ss_k8s image tag
        run: |
          # ponytail: 이미지가 하나라 newTag 줄을 sed로 바꾼다. 이미지가 늘면 yq로 name을 골라 바꾼다.
          sed -i "s/^\( *newTag:\).*/\1 ${GITHUB_SHA::7}/" apps/ss-be/kustomization.yaml
          git diff --quiet && exit 0
          git config user.name "${{ steps.app.outputs.app-slug }}[bot]"
          git config user.email "${{ steps.app.outputs.app-slug }}[bot]@users.noreply.github.com"
          git commit -am "ss_be 이미지 태그 ${GITHUB_SHA::7}로 갱신"
          git push
```

**`push` job**

| 단계 | 하는 일 |
|---|---|
| `permissions: id-token: write` | OIDC 토큰 발급을 허용한다. `push` job에만 준다. 빠지면 인증이 실패한다 |
| `actions/checkout` | 레포 소스를 러너에 내려받는다 |
| `actions/setup-java` | JDK 21을 설치하고 Maven 의존성을 캐시한다 |
| `configure-aws-credentials` | OIDC 토큰을 IAM Role의 임시 자격증명으로 바꾼다 |
| `amazon-ecr-login` | ECR 로그인 정보를 `~/.docker/config.json`에 기록한다. `registry` 출력에 ECR 주소가 담긴다 |
| `mvn compile jib:build` | 컴파일한 뒤 Jib으로 이미지를 만들어 ECR에 push한다. 테스트는 실행하지 않는다 |

**`update-tag` job**

| 단계 | 하는 일 |
|---|---|
| `needs: push` | 이미지 push가 성공한 뒤에만 태그를 바꾼다. 없는 이미지를 가리키는 일을 막는다 |
| `environment: ss-k8s-write` | `main` 실행에서만 App 개인키 Secret을 꺼낼 수 있다 |
| `create-github-app-token` | App 개인키로 설치 토큰을 발급받는다. `repositories`와 `permission-contents`로 `ss_k8s` Contents 쓰기만 가진 토큰으로 좁힌다. 토큰은 1시간 뒤 만료되고, job이 끝나면 action이 토큰을 폐기한다 |
| `actions/checkout` (`ss_k8s`) | 발급받은 토큰으로 `ss_k8s`를 내려받는다. 이후 `git push`도 이 토큰으로 인증된다 |
| `sed` | `newTag:` 줄을 짧은 SHA로 바꾼다 |
| `git diff --quiet && exit 0` | 같은 SHA로 재실행해서 바뀐 내용이 없으면 커밋하지 않는다 |
| `git commit`, `git push` | App 봇 이름으로 `ss_k8s` `main`에 커밋한다 |

- **job을 나눈 이유**: job에 `environment:`를 지정하면 그 job의 OIDC `sub`가 `repo:...:environment:ss-k8s-write` 형식으로 바뀐다. 같은 job에 두면 AWS 신뢰 정책(브랜치 형식)과 맞지 않아 ECR push가 실패한다.
- **`concurrency`**: 연속 push가 들어와도 실행을 하나씩 처리해서 `ss_k8s` 태그 커밋끼리 충돌하지 않게 한다. 대기 중인 실행은 취소하지 않는다.
- Maven은 `ubuntu-latest` 러너에 기본으로 설치돼 있다. 레포에 Maven Wrapper(`mvnw`)는 없다.

### 3.3 `ss_k8s`: `apps/ss-be/kustomization.yaml`

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Deployment, Service 등은 클러스터 베이스 작업 후 추가한다.
resources: []

# newTag는 ss_be의 ecr-push workflow가 main push마다 짧은 SHA로 갱신한다.
images:
  - name: ss_be
    newName: 944296702922.dkr.ecr.ap-northeast-2.amazonaws.com/ss_be
    newTag: 366bff4
```

- 나중에 추가할 Deployment에서는 `image: ss_be`라고만 적는다. kustomize가 `newName:newTag`로 바꿔 준다.
- `newTag` 값은 workflow가 바꾼다. 사람이 직접 고치지 않는다.

---

## 4. AWS 설정 절차 (계정당 1회)

### 4.1 IAM OIDC Provider 등록

IAM → 자격 증명 공급자 → 공급자 추가

| 항목 | 값 |
|---|---|
| 공급자 유형 | OpenID Connect |
| 공급자 URL | `https://token.actions.githubusercontent.com` |
| 대상 (Audience) | `sts.amazonaws.com` |

- "대상" 칸의 설명은 "앱의 ID 제공업체가 발행한 클라이언트 ID"로 되어 있다. GitHub Actions에는 발급된 클라이언트 ID가 따로 없으므로, 토큰의 `aud` 값인 `sts.amazonaws.com`을 넣는다.

### 4.2 IAM Role 생성

IAM → 역할 → 역할 생성 → **웹 자격 증명**

| 항목 | 값 |
|---|---|
| 자격 증명 공급자 | `token.actions.githubusercontent.com` |
| Audience | `sts.amazonaws.com` |
| GitHub organization | `say-scene` |
| GitHub repository | `ss_be` |
| GitHub branch | `main` |
| 역할 이름 | `gha-ss-be-ecr-push` |
| 설명 | `Allows GitHub Actions to push ss_be images to ECR` |
| 권한 경계 | 사용하지 않음 |

- 권한 정책 선택 화면에서는 관리형 정책을 고르지 않는다. `AmazonEC2ContainerRegistryPowerUser` 같은 관리형 정책은 계정의 모든 ECR 리포지토리에 권한을 준다.
- 권한 경계를 쓰지 않는 이유: 인라인 정책이 이미 최소 권한이고, Role에 IAM 권한이 없어 스스로 권한을 늘릴 수 없다.

#### 신뢰 정책 (최종본)

```json
{
	"Version": "2012-10-17",
	"Statement": [
		{
			"Effect": "Allow",
			"Principal": {
				"Federated": "arn:aws:iam::944296702922:oidc-provider/token.actions.githubusercontent.com"
			},
			"Action": "sts:AssumeRoleWithWebIdentity",
			"Condition": {
				"StringEquals": {
					"token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
					"token.actions.githubusercontent.com:sub": "repo:say-scene@345080911/ss_be@1426408654:ref:refs/heads/main"
				}
			}
		}
	]
}
```

- **`sub`에 숫자 ID가 들어간다.** 이 레포는 GitHub OIDC **immutable subject**(`use_immutable_subject: true`)가 켜져 있어서, `sub`가 `repo:<org>@<org ID>/<repo>@<repo ID>:...` 형식으로 발급된다. 콘솔 마법사가 만든 `repo:say-scene/ss_be:...` 형식으로는 인증이 실패한다(첫 실행 실패 원인).
  - ECR의 tag mutability와는 관계없는 GitHub 쪽 설정이다.
  - 실제 prefix는 아래 명령으로 확인한다.
    ```bash
    gh api repos/say-scene/ss_be/actions/oidc/customization/sub
    ```
- org 이름은 **`say-scene`**(하이픈 포함)이다. 처음에 `sayscene`으로 잘못 입력해서 고쳤다. 틀리면 인증이 실패한다.
- 콘솔 마법사가 `sub` 조건을 `StringLike`로 만들거나 같은 값을 두 번 넣을 수 있다. 와일드카드(`*`)가 없으면 `StringEquals` 하나로 정리한다.
- `sub`에 `*`를 넣으면 다른 브랜치나 레포에서도 이 Role을 쓸 수 있으므로 넣지 않는다.

#### 권한 정책 (인라인)

역할 → 권한 추가 → 인라인 정책 생성 → JSON

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage",
        "ecr:BatchGetImage"
      ],
      "Resource": "arn:aws:ecr:ap-northeast-2:944296702922:repository/ss_be"
    }
  ]
}
```

- `ecr:GetAuthorizationToken`은 리소스 범위를 좁힐 수 없다. 반드시 `"Resource": "*"`로 둔다.

### 4.3 ECR 리포지토리 생성

ECR → 프라이빗 리포지토리 → 생성 (`ap-northeast-2`)

| 항목 | 값 |
|---|---|
| 이름 | `ss_be` |
| 태그 변경 불가능성 | Mutable (같은 SHA로 재실행하면 태그를 덮어쓴다) |

### 4.4 수명 주기 정책 (선택, 첫 이미지 push 후 적용)

최근 이미지 30개만 보관한다.

```json
{"rules":[{"rulePriority":1,"description":"keep last 30","selection":{"tagStatus":"any","countType":"imageCountMoreThan","countNumber":30},"action":{"type":"expire"}}]}
```

---

## 5. GitHub 설정

### 5.1 AWS Role ARN 등록

Role ARN을 Repository Variable로 등록한다. 비밀 값이 아니므로 Secret이 아니라 **Variable**에 넣는다.

```bash
gh variable set AWS_ROLE_ARN --repo say-scene/ss_be --body "arn:aws:iam::944296702922:role/gha-ss-be-ecr-push"
```

- 콘솔에서는 레포 Settings → Secrets and variables → Actions → **Variables** 탭에서 등록한다.
- **Secrets** 탭에 넣으면 workflow의 `vars.AWS_ROLE_ARN`으로 읽을 수 없다.

### 5.2 `ss_k8s` 태그 갱신용 GitHub App 설정

#### 1) GitHub App 생성 (org 소유)

`https://github.com/organizations/say-scene/settings/apps/new`

| 항목 | 값 |
|---|---|
| GitHub App name | `say-scene-ss-k8s-writer` (GitHub 전체에서 중복 불가) |
| Homepage URL | `https://github.com/say-scene` (필수 입력, 아무 URL 가능) |
| Webhook → Active | **체크 해제** |
| Repository permissions → Contents | **Read and write** (Metadata는 자동으로 Read-only) |
| 나머지 권한 | 모두 No access |
| Where can this GitHub App be installed? | **Only on this account** |

- **반드시 org 설정 화면에서 만든다.** 개인 계정 Settings → Developer settings에서 만들면 앱이 개인 계정 소유가 되고, "Only on this account" 때문에 Install App 목록에 개인 계정만 보인다.
  - 이미 개인 계정으로 만들었다면 앱 설정 → **Advanced → Transfer ownership of this GitHub App**에서 `say-scene`으로 이전한다. App ID와 개인키는 그대로 유지된다.
  - 앱 설정 주소로 소유자를 구분한다. `github.com/settings/apps/...`는 개인 계정, `github.com/organizations/say-scene/settings/apps/...`는 org 소유다.

#### 2) App ID 확인, 개인키 생성

- 앱 General 페이지 상단의 **App ID**(숫자)를 적어 둔다. `Iv1.`로 시작하는 Client ID와 다르다.
- 같은 페이지 **Private keys → Generate a private key**를 누르면 `.pem` 파일이 다운로드된다.

#### 3) 앱을 `ss_k8s`에만 설치

앱 페이지 왼쪽 **Install App** → `say-scene` → **Install** → **Only select repositories** → `ss_k8s`

#### 4) `ss_be`에 environment 생성

`ss_be` 레포 Settings → Environments → **New environment**

| 설정 | 값 | 설명 |
|---|---|---|
| 이름 | `ss-k8s-write` | workflow의 `environment:` 값과 정확히 같아야 한다 |
| Deployment branches and tags | **Selected branches and tags** → `main` | 이 environment(=App 개인키)를 `main` 실행에서만 쓸 수 있게 한다. **핵심 보안 장치** |
| Required reviewers | 끔 | 켜면 push마다 태그 갱신이 승인을 기다리며 멈춘다 |
| Wait timer | 끔 | 쓸 이유 없음 |
| Enable custom rules with GitHub Apps | 사용 안 함 | 외부 앱으로 배포 허용을 판단하는 기능(Preview) |
| Allow administrators to bypass | 켜져 있음(기본값) | 승인자·대기 규칙이 없어 사실상 영향이 없다. 브랜치 제한까지 우회되는지는 확인하지 못했다. 엄격하게 하려면 끈다 |

- `ss_be`는 public 레포라서 무료 플랜에서도 environment의 브랜치 제한을 쓸 수 있다.

#### 5) App ID와 개인키 등록

| 값 | 위치 | 이름 |
|---|---|---|
| App ID | Settings → Environments → `ss-k8s-write` → **Environment variables** → Add environment variable | `SS_K8S_APP_ID` |
| 개인키(`.pem` 전체) | Settings → Environments → `ss-k8s-write` → **Environment secrets** → Add environment secret | `SS_K8S_APP_PRIVATE_KEY` |

- App ID와 개인키를 모두 `ss-k8s-write` environment에 둔다. App 관련 값이 한 곳에 모여, App을 교체하거나 정리할 때 environment 하나만 보면 된다. App ID는 비밀 값이 아니므로 Secret이 아니라 Variable에 넣는다.
- workflow는 그대로 `vars.SS_K8S_APP_ID`로 읽는다. `update-tag` job에 `environment: ss-k8s-write`가 있어서 environment variable이 읽힌다. 같은 이름의 Repository variable이 있으면 environment 값이 우선하므로, 헷갈리지 않게 Repository variable에는 두지 않는다.

CLI로는 아래처럼 등록한다.

```bash
gh variable set SS_K8S_APP_ID --repo say-scene/ss_be --env ss-k8s-write --body "<App ID 숫자>"
```

```bash
gh secret set SS_K8S_APP_PRIVATE_KEY --repo say-scene/ss_be --env ss-k8s-write < ~/Downloads/<다운로드된 파일>.pem
```

- 웹에서 붙여넣을 때는 `pbcopy < <파일>.pem`으로 복사한다. `-----BEGIN RSA PRIVATE KEY-----`부터 `-----END RSA PRIVATE KEY-----`까지 줄바꿈을 포함해 전부 넣는다.
- 개인키를 **Repository secrets**에 넣으면 안 된다. `main`이 아닌 브랜치의 workflow도 꺼낼 수 있게 된다.

#### 6) 등록 확인, 로컬 개인키 삭제

```bash
gh api repos/say-scene/ss_be/environments/ss-k8s-write --jq '{deployment_branch_policy, can_admins_bypass}'
```

```bash
gh variable list --repo say-scene/ss_be --env ss-k8s-write
```

```bash
gh secret list --repo say-scene/ss_be --env ss-k8s-write
```

```bash
gh api orgs/say-scene/installations --jq '.installations[] | {app_slug, repository_selection, permissions}'
```

- environment Secret에 `SS_K8S_APP_PRIVATE_KEY`가 보이면 다운로드한 `.pem` 파일은 삭제한다. 키를 잃어버리면 앱 페이지에서 새로 만들면 된다.
- 설치 대상 레포가 `ss_k8s` 하나인지는 org Settings → GitHub Apps → `say-scene-ss-k8s-writer` → Configure 화면에서 확인한다(일반 `gh` 토큰으로는 조회 불가).

---

## 6. 실행과 확인

1. 변경 사항을 `ss_be` `main`에 push한다. 수동으로 실행하려면 Actions 탭 → `ecr-push` → Run workflow를 누른다.
2. GitHub Actions 탭에서 `ecr-push`의 `push`, `update-tag` job이 모두 성공했는지 확인한다.
3. ECR에 이미지가 올라갔는지 확인한다.

```bash
aws ecr describe-images --repository-name ss_be --region ap-northeast-2 --query 'sort_by(imageDetails,&imagePushedAt)[-1].imageTags'
```

4. `ss_k8s`에 태그 커밋이 생겼는지 확인한다.

```bash
gh api 'repos/say-scene/ss_k8s/commits?per_page=1' --jq '.[0] | "\(.sha[0:7]) \(.commit.author.name) \(.commit.message)"'
```

```bash
gh api repos/say-scene/ss_k8s/contents/apps/ss-be/kustomization.yaml --jq .content | base64 -d | grep newTag
```

---

## 7. 문제 해결

| 증상 | 원인 후보 | 조치 |
|---|---|---|
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | 신뢰 정책 `sub` 불일치(immutable subject의 숫자 ID 누락, org·repo·branch 오타), `aud` 불일치 | §4.2의 `gh api` 명령으로 prefix를 확인하고 최종본과 비교한다 |
| `push` job에 `environment:`를 넣은 뒤 위 오류 발생 | `sub`가 `...:environment:<이름>` 형식으로 바뀜 | `environment`는 `update-tag` job에만 둔다 |
| `Credentials could not be loaded` | `permissions: id-token: write` 누락 | `push` job의 `permissions`를 확인한다 |
| `Input required and not supplied: role-to-assume` | `AWS_ROLE_ARN` Variable 미등록, 또는 Secrets 탭에 등록함 | §5.1대로 Variables 탭에 등록한다 |
| 수동 실행만 실패 | `main`이 아닌 브랜치에서 실행함 | 신뢰 정책과 environment가 `main`만 허용한다. `main`에서 실행한다 |
| ECR push 단계에서 `name unknown` / `repository does not exist` | ECR 리포지토리 이름이나 리전 불일치 | `ss_be` 리포지토리가 `ap-northeast-2`에 있는지 확인한다 |
| ECR push 단계에서 `AccessDenied` | 권한 정책 Resource ARN 불일치 | §4.2 권한 정책의 ARN을 확인한다 |
| `create-github-app-token`에서 app-id나 private-key 입력 누락 오류 | `SS_K8S_APP_ID` Variable이나 개인키 Secret을 `ss-k8s-write` environment가 아닌 곳에 등록함 | §5.2-5대로 등록 위치를 확인한다 |
| `create-github-app-token`에서 설치를 찾지 못함 | 앱이 `say-scene` org에 설치되지 않았거나 `ss_k8s`가 설치 대상에 없음 | §5.2-1, 3을 확인한다. 개인 계정 소유 앱이면 org로 소유권을 이전한다 |
| `ss_k8s` checkout이나 push에서 `403` / `Permission denied` | App의 Contents 권한이 Read only | App 권한을 Read and write로 바꾸고, org 설치 화면에서 변경된 권한을 승인한다 |
| `update-tag`가 성공했는데 `ss_k8s` 커밋이 없음 | 같은 SHA로 재실행해서 `newTag`가 이미 같음 | 정상 동작이다(`git diff --quiet`로 건너뜀) |
| `newTag`가 바뀌지 않음 | `kustomization.yaml`의 `newTag:` 줄 형식이 바뀜 | `newTag:` 줄이 하나이고 공백 들여쓰기로 시작하는지 확인한다 |
| 베이스 이미지 조회 중 `429 Too Many Requests` | Docker Hub 익명 pull 횟수 제한 | `<from><image>`를 `public.ecr.aws/docker/library/eclipse-temurin:21-jre`로 바꾼다 |
| Pod가 `exec format error`로 실행되지 않음 | 노드가 ARM(Graviton)인데 이미지가 `amd64` | Jib `<platforms>` 설정에 `arm64`를 추가한다 |

---

## 8. 알려진 제약과 추후 결정 사항

- **Docker Hub 의존**: Jib은 빌드할 때마다 베이스 이미지 정보를 Docker Hub에서 조회한다. GitHub 호스티드 러너는 매번 새로 만들어지므로 Jib 로컬 캐시가 남지 않는다. 레이어 본체는 ECR에 이미 있으면 다운로드를 건너뛰는 것으로 알고 있지만, 실제로 확인하지는 않았다.
- **테스트 미실행**: `mvn compile jib:build`는 테스트를 실행하지 않는다. 테스트를 넣기로 하면 `verify`를 추가한다.
- **브랜치 보호 없음**: `main`에 push할 수 있는 사람은 누구나 ECR과 `ss_k8s`에도 반영할 수 있다.
- **Maven 버전 미고정**: `mvnw`가 없어서 러너에 설치된 Maven 버전을 쓴다.
- **App 개인키는 장기 비밀**: 만료되지 않는다. 유출이 의심되면 앱 페이지에서 키를 삭제하고 새로 만들어 §5.2-5대로 다시 등록한다.
- **태그 치환은 sed**: `kustomization.yaml`에 이미지가 하나라서 `newTag:` 줄을 sed로 바꾼다. 이미지가 늘면 `yq`로 `name`을 골라 바꾸도록 고친다.
- **`ss_k8s` push 경합**: workflow끼리는 `concurrency`로 직렬화하지만, 사람이 같은 시점에 `ss_k8s` `main`에 push하면 태그 push가 거절될 수 있다. 그때는 실행을 다시 돌린다.
- **Action 버전**: `checkout@v4`, `setup-java@v4`, `configure-aws-credentials@v4`, `create-github-app-token@v2`는 Node.js 20 기반이라 실행 시 deprecated 경고가 나온다. `setup-java`는 v5로 옮기라는 안내도 있다. 지금 동작에는 영향이 없다.
- **러너 이미지**: `ubuntu-latest`가 2026-10-19부터 Ubuntu 26으로 넘어간다는 안내가 나온다.
- **범위 밖 (다음 단계)**: `ss_k8s`의 실제 매니페스트(Deployment, Service 등, 지금은 `resources: []`), namespace와 Secret(`INFERENCE_API_KEY`), ArgoCD 설치와 Application(`apps/ss-be` 경로, `automated` sync) 등록.

---

## 9. 검증 상태

- AWS 설정(OIDC Provider, Role 신뢰·권한 정책, ECR 리포지토리)과 GitHub Variable 등록은 끝났다.
- workflow를 `main`에 push했다(`6d33334`).
- 첫 실행은 `sub` 형식 불일치(immutable subject)로 인증 단계에서 실패했다. 신뢰 정책을 고친 뒤 재실행해서 **성공했다** ([run 39712790582](https://github.com/say-scene/ss_be/actions/runs/39712790582)).
- GitHub App 태그 갱신을 추가한 workflow(`366bff4`)를 `main`에 push했고, `push`와 `update-tag` job이 모두 **성공했다** ([run 39737304461](https://github.com/say-scene/ss_be/actions/runs/39737304461)).
  - Jib 로그: `Built and pushed image as 944296702922.dkr.ecr.ap-northeast-2.amazonaws.com/ss_be:366bff4`
  - `ss_k8s` 커밋 `6d06130` (`say-scene-ss-k8s-writer[bot]`): `newTag`가 `6d33334` → `366bff4`로 바뀌었다.
- `SS_K8S_APP_ID`를 Repository variable에서 `ss-k8s-write` environment variable로 옮긴 뒤 수동 실행(`workflow_dispatch`)이 **성공했다** ([run 39758974204](https://github.com/say-scene/ss_be/actions/runs/39758974204)). 같은 SHA라 `ss_k8s` 커밋은 생기지 않았다(정상).
- 콘솔 화면의 항목 이름은 AWS·GitHub UI 변경으로 달라질 수 있다.
