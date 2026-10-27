# ss-fe CI/CD 작업 절차

이 문서를 위에서 아래로 그대로 따라 하면 같은 결과가 만들어진다.

- 레포: `say-scene/ss-fe`
- AWS 계정: `944296702922` / 리전 `ap-northeast-2`
- 버킷: `ss-fe-944296702922-ap-northeast-2-an`

---

## 1단계 — 트리거

### 1-1. 워크플로 파일 생성

```bash
mkdir -p .github/workflows && cat > .github/workflows/build.yml <<'EOF'
name: build

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: echo "트리거 확인"
EOF
```

### 1-2. 커밋 + push

```bash
git add .github/workflows/build.yml && git commit -m "ci: 빌드 워크플로 트리거 추가" && git push origin main
```

### 1-3. 확인

```bash
gh run list --repo say-scene/ss-fe --limit 3
```

| 기대 | |
|---|---|
| 상태 | `success` |
| 트리거 | `push` / 브랜치 `main` |
| 소요 | 10초 내외 |

안 뜨면: 경로 오타 / YAML 들여쓰기 / `main` 아닌 브랜치.

---

## 2단계 — 검증

### 2-1. `steps:` 교체

```bash
cat > .github/workflows/build.yml <<'EOF'
name: build

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm

      - run: npm ci

      - run: npm test

      - run: npm run build
EOF
```

`node-version`은 `package.json`의 `engines.node`에 맞춘다.

### 2-2. 커밋 + push

```bash
git add .github/workflows/build.yml && git commit -m "ci: 빌드 검증 스텝 추가" && git push origin main
```

### 2-3. 확인

```bash
RUN=$(gh run list --repo say-scene/ss-fe --limit 1 --json databaseId --jq '.[0].databaseId') && gh run view $RUN --repo say-scene/ss-fe
```

6스텝 전부 초록. 첫 실행은 캐시가 없어 2~3분.

```bash
gh run view $RUN --repo say-scene/ss-fe --log | grep -iE "Test Files|Tests |built in|dist/"
```

빌드 산출물의 해시 파일명을 확인해 둔다 (3단계 캐시 정책의 근거).

---

## 3단계 A구간 — S3 업로드

### 3-0. OIDC sub 접두사 조회 ★ 가장 먼저

IAM 롤의 신뢰 정책에 들어갈 값을 먼저 확인한다. 이걸 건너뛰면 인증이 거부된다.

```bash
gh api repos/say-scene/ss-fe/actions/oidc/customization/sub
```

```json
{"use_default":true,"use_immutable_subject":true,
 "sub_claim_prefix":"repo:say-scene@345080911/ss-fe@1426408350"}
```

신뢰 정책에 쓸 `sub` 값 = `sub_claim_prefix` + `:ref:refs/heads/main`

```
repo:say-scene@345080911/ss-fe@1426408350:ref:refs/heads/main
```

`use_immutable_subject`가 `false`면 이름 기반(`repo:say-scene/ss-fe:ref:refs/heads/main`)이다.
레포마다 ID가 다르므로 `ss-be`, `ss-triton`, `ss-k8s`도 각각 조회한다.

> **레포·조직 이름을 바꾸면 이 값을 다시 조회해서 3-2의 신뢰 정책을 고친다.**
> `sub`에는 ID와 **이름이 함께** 들어가므로, ID가 그대로여도 리네임만으로 인증이 거부된다.
> 2026-10-23 `ss_fe` → `ss-fe` 리네임 때 실제로 겪었다.
> 코드는 한 줄도 안 바뀌므로 `build.yml`을 들여다봐도 원인이 보이지 않는다.

### 3-1. S3 버킷 (AWS 콘솔)

`S3` → `버킷 만들기`

| 항목 | 값 |
|---|---|
| 리전 | 아시아 태평양(서울) `ap-northeast-2` |
| 버킷 이름 | `ss-fe-944296702922-ap-northeast-2-an` |
| 퍼블릭 액세스 차단 | **4개 전부 체크 유지** |
| 그 외 | 전부 기본값 |

정적 웹사이트 호스팅은 건드리지 않는다(기본 꺼짐).

### 3-2. OIDC provider (AWS 콘솔)

`IAM` → `자격 증명 공급자`

**목록에 `token.actions.githubusercontent.com`이 이미 있으면 건너뛴다.** 계정당 1개만 존재할 수 있고 5개 레포가 공유한다. 있으면 **대상(Audiences)에 `sts.amazonaws.com`이 있는지만** 확인하고, 없으면 추가한다.

없으면 `공급자 추가`:

| 항목 | 값 |
|---|---|
| 공급자 유형 | OpenID Connect |
| 공급자 URL | `https://token.actions.githubusercontent.com` |
| 대상 | `sts.amazonaws.com` |

`지문 가져오기` 버튼이 보이면 클릭 → `공급자 추가`

### 3-3. 권한 정책 (AWS 콘솔)

`IAM` → `정책` → `정책 생성` → `JSON` 탭 → 전체 교체

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::ss-fe-944296702922-ap-northeast-2-an/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::ss-fe-944296702922-ap-northeast-2-an"
    }
  ]
}
```

이름: `ss-fe-s3-upload` → `정책 생성`

`DeleteObject`는 넣지 않는다(`--delete`를 쓰지 않음). `cloudfront:CreateInvalidation`은 B구간에서 추가.

### 3-4. IAM 롤 (AWS 콘솔)

`IAM` → `역할` → `역할 생성`

1. 신뢰할 수 있는 엔터티: **웹 자격 증명**
2. 공급자 `token.actions.githubusercontent.com`, Audience `sts.amazonaws.com`
3. 권한: `ss-fe-s3-upload` 선택
4. 이름: `gha-ss-fe-deploy` → `역할 생성`

생성 후 **`신뢰 관계` 탭 → `신뢰 정책 편집`** 으로 전체 교체 (3-0에서 조회한 `sub` 사용):

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
          "token.actions.githubusercontent.com:sub": "repo:say-scene@345080911/ss-fe@1426408350:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

`정책 업데이트`

**확인:** 역할 요약의 ARN이 `arn:aws:iam::944296702922:role/gha-ss-fe-deploy` 인지(경로 없음).

### 3-5. GitHub Variables

```bash
gh variable set AWS_ACCOUNT_ID --repo say-scene/ss-fe --body "944296702922" && gh variable set S3_BUCKET --repo say-scene/ss-fe --body "ss-fe-944296702922-ap-northeast-2-an"
```

```bash
gh variable list --repo say-scene/ss-fe
```

`Secrets`가 아니라 `Variables`. 비밀이 아니고 로그에 보여야 디버깅이 된다.

### 3-6. 워크플로에 배포 스텝 추가

```bash
cat > .github/workflows/build.yml <<'EOF'
name: build

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

permissions:
  id-token: write
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm

      - run: npm ci

      - run: npm test

      - run: npm run build

      - uses: aws-actions/configure-aws-credentials@v4
        if: github.ref == 'refs/heads/main'
        with:
          role-to-assume: arn:aws:iam::${{ vars.AWS_ACCOUNT_ID }}:role/gha-ss-fe-deploy
          aws-region: ap-northeast-2

      - name: asset 업로드 (1년 캐시)
        if: github.ref == 'refs/heads/main'
        run: |
          aws s3 sync dist/ s3://${{ vars.S3_BUCKET }}/ \
            --exclude index.html \
            --cache-control "public,max-age=31536000,immutable" \
            --metadata "commit=${{ github.sha }},run=${{ github.run_id }}"

      - name: index.html 업로드 (캐시 금지)
        if: github.ref == 'refs/heads/main'
        run: |
          aws s3 cp dist/index.html s3://${{ vars.S3_BUCKET }}/index.html \
            --cache-control "no-cache" \
            --metadata "commit=${{ github.sha }},run=${{ github.run_id }}"
EOF
```

바뀐 것: `permissions` 3줄 + AWS 스텝 3개. 기존 줄은 그대로.

### 3-7. 커밋 + push

```bash
git add .github/workflows/build.yml && git commit -m "ci: S3 업로드 스텝 추가" && git push origin main
```

### 3-8. 확인

```bash
RUN=$(gh run list --repo say-scene/ss-fe --limit 1 --json databaseId --jq '.[0].databaseId') && gh run view $RUN --repo say-scene/ss-fe
```

8스텝 전부 초록이어야 한다.

```bash
gh run view $RUN --repo say-scene/ss-fe --log | grep -iE "Authenticated|upload:"
```

```
Authenticated as assumedRoleId AROA...:GitHubActions
upload: dist/assets/index-*.css  →  s3://.../assets/index-*.css
upload: dist/assets/index-*.js   →  s3://.../assets/index-*.js
upload: dist/sayscene-logo.png   →  s3://.../sayscene-logo.png
upload: dist/index.html          →  s3://.../index.html        ← 맨 마지막
```

`index.html`이 맨 마지막이어야 한다(순서가 지켜진 증거).

### 3-9. S3 메타데이터 확인 ★ A구간 최종 검증

`S3` → 버킷 → `객체` → 파일 클릭 → **메타데이터** 섹션

| 파일 | `Cache-Control` |
|---|---|
| `assets/index-*.js` | `public,max-age=31536000,immutable` |
| `assets/index-*.css` | `public,max-age=31536000,immutable` |
| `sayscene-logo.png` | `public,max-age=31536000,immutable` |
| `index.html` | **`no-cache`** |

`index.html`에 `immutable`이 붙어 있으면 CloudFront를 붙인 뒤 배포해도 화면이 안 바뀐다.

**화면 확인은 여기서 불가.** 버킷이 퍼블릭 차단이므로 B구간(CloudFront)이 필요하다.

---

## 실패 시 조치

```bash
gh run view $RUN --repo say-scene/ss-fe --log-failed
```

| 에러 | 원인 | 고칠 곳 |
|---|---|---|
| Actions 탭에 실행 없음 | 경로·YAML·브랜치 | 워크플로 |
| `Unable to get OIDC token` | `permissions: id-token: write` 누락 | 워크플로 |
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | provider 미등록 또는 **`sub` 불일치** | 3-0 재조회 → 3-4 |
| `AccessDenied` | 권한 정책의 버킷 ARN | 3-3 |
| `npm ci`/`npm test`/`build` 실패 | 코드 | 개발자에게 전달 |

AWS 설정만 고친 경우 코드 변경 없이 재실행:

```bash
gh run rerun $RUN --repo say-scene/ss-fe
```

---

## 롤백

```bash
git revert <커밋> && git push origin main
```

재빌드 → `index.html` 덮어쓰기. `--delete`를 쓰지 않아 옛 asset이 남아 있으므로 성립한다.

S3 객체를 지운 경우: Actions 탭 → `Run workflow`(branch: main). `workflow_dispatch`도 `github.ref`가 `refs/heads/main`이라 업로드까지 수행한다.

---

## 남은 작업 — B구간 (CloudFront)

| | 할 일 |
|---|---|
| 3-10 | CloudFront 배포 + OAC → 생성된 버킷 정책을 S3에 붙이기 |
| 3-11 | CloudFront Function (viewer-request) — SPA 라우팅 |
| 3-12 | IAM 롤에 `cloudfront:CreateInvalidation` 추가 |
| 3-13 | `gh variable set CF_DISTRIBUTION_ID` |
| 3-14 | 워크플로에 invalidation 스텝 추가 |

SPA 라우팅 Function (없으면 새로고침 시 404):

```js
function handler(event) {
  var uri = event.request.uri;
  if (!uri.includes('.')) { event.request.uri = '/index.html'; }
  return event.request;
}
```

"사용자 지정 오류 응답(403/404 → index.html)"은 쓰지 않는다 — 나중에 `/api/*`의 정상 404까지 바꿔버린다.

invalidation 스텝:

```yaml
      - name: CloudFront invalidation
        if: github.ref == 'refs/heads/main'
        run: |
          aws cloudfront create-invalidation \
            --distribution-id ${{ vars.CF_DISTRIBUTION_ID }} \
            --paths "/index.html"
```
