# ss-fe CI/CD 개념 설명

절차는 [ss-fe-cicd-runbook-v1.md](ss-fe-cicd-runbook-v1.md) 참조. 이 문서는 "왜 그렇게 하는가"만 다룬다.

---

## 1. 세 레포가 갈라지는 이유

| 레포 | 산출물 | 보관 | 실행 주체 | k8s |
|---|---|---|---|---|
| `ss-fe` | 정적 파일 | S3 | 사용자 브라우저 | ✗ |
| `ss-be` | 컨테이너 이미지 | ECR | CPU 노드 | ✓ |
| `ss-triton` | 컨테이너 이미지 | ECR | GPU 노드 | ✓ |

근본 차이는 하나다.

- **FE는 "실행"되지 않고 "다운로드"된다.** 브라우저가 받아서 실행한다. 서버 쪽에 돌고 있는 프로세스가 없다 → 파일만 놓으면 끝.
- **BE는 24시간 돌고 있는 프로세스다.** "돌고 있는 걸 교체"해야 한다 → 이미지로 포장 → k8s가 교체.

그래서 `ss-fe`만 ArgoCD를 거치지 않고, Actions 초록불 = 배포 완료다.
`ss-be`는 초록불 = **이미지와 지시서까지만**이고 실제 반영은 ArgoCD가 나중에 한다.

### 배치를 정하는 곳

코드도 이미지도 자기가 어디서 돌지 모른다. `ss-k8s` 매니페스트가 정한다.

```yaml
# backend — GPU 요청 없음 → taint 없는 general 노드로 자연히 스케줄
resources:
  requests: { cpu: 500m, memory: 1Gi }

# Triton — 이 한 줄이 GPU 노드를 고른다
resources:
  limits: { nvidia.com/gpu: 1 }
```

그래서 "레플리카 늘려라", "메모리 올려라", "되돌려라"는 소스 레포를 안 건드리고 `ss-k8s` 커밋 하나로 끝난다.

---

## 2. 단계를 나눈 이유

| 단계 | 확인하는 것 | AWS |
|---|---|---|
| 1단계 | 트리거가 걸리는가 | ✗ |
| 2단계 | 코드가 빌드·테스트를 통과하는가 | ✗ |
| 3단계 A | S3에 올라가는가 (OIDC 인증) | ✓ |
| 3단계 B | 사용자가 볼 수 있는가 (CloudFront) | ✓ |

**한 번에 하면 실패 원인이 섞인다.** 1~2단계에서 Actions 자체를 익히고, A구간에서 인증만, B구간에서 배달만 따로 잡는다. S3 업로드는 CloudFront 없이 완결되므로 나눌 수 있다.

---

## 3. VM이 매번 새 것이라는 점

```
VM 안에 있는 것:  git, docker, aws CLI, 각종 SDK (미리 설치)
VM 안에 없는 것:  내 코드, Node, node_modules, AWS 자격증명
```

여기서 따라오는 결과:

- `actions/checkout`이 **항상 첫 줄**이다. 코드조차 없다.
- `cache: npm`이 필요하다. 다만 캐시되는 건 `~/.npm`(다운로드)이고 `node_modules`가 아니라서 `npm ci`는 여전히 매번 돈다.
- 내 맥북 설정이 하나도 안 묻어온다. **"내 로컬에선 됐는데"가 구조적으로 불가능해진다.** 이게 CI의 핵심 가치다.
- 워크플로가 끝나면 `dist/`도 사라진다. 그래서 2단계는 "깨지지 않는다"만 확인하고 끝난다.

### `npm install`이 아니라 `npm ci`

| | `npm install` | `npm ci` |
|---|---|---|
| lock 파일 | 필요하면 **수정** | 그대로만 따름 |
| lock ↔ package.json 불일치 | 조용히 맞춰 수정 | **에러로 중단** |

세 번째 줄이 핵심이다. `npm install`은 누가 `package.json`만 고치고 lock을 안 갱신한 상태를 조용히 메꾼다. 그러면 CI와 로컬이 다른 버전으로 도는데 아무도 모른다. CI에서는 멈추는 쪽이 맞다.

---

## 4. 게이트 — 검증 실패 시 배포 차단

GitHub Actions의 기본 동작이 그대로 게이트다.

> 스텝이 실패하면 그 뒤의 스텝은 전부 건너뛴다.

```
npm test  ✗
    ↓
npm run build        건너뜀
AWS 인증             건너뜀
S3 업로드            건너뜀    ← S3에 아무것도 안 올라감
```

**배포 스텝을 검증 스텝 뒤에 두는 것 자체가 게이트다.** 별도 장치가 없다.

주의: 어떤 스텝에도 `continue-on-error: true`를 붙이면 무력화된다.

### 브랜치 보호를 걸지 않기로 한 결정

`main` 직접 push는 가능해야 한다. 그러면 막는 대상이 바뀐다.

```
브랜치 보호:  깨진 코드가 main에 들어가는 것을 막는다     ← 안 함
CI 게이트:    깨진 코드가 S3로 나가는 것을 막는다        ← 함
```

| 상황 | 검증 | 배포 | 결과 |
|---|---|---|---|
| PR, 정상 | ✓ | 건너뜀 | 머지해도 안전하다는 신호 |
| PR, 깨짐 | ✗ | 건너뜀 | 머지 전에 알게 됨 |
| main push, 정상 | ✓ | 실행 | 배포 완료 |
| main push, **깨짐** | ✗ | **건너뜀** | **main엔 들어갔지만 배포 안 됨** |

**부작용:** 깨진 코드가 `main`에 남아 다음 사람이 그 위에서 작업한다. 완화책은 알림 — `main` push 실패 시 GitHub이 커밋 작성자에게 메일을 보낸다. 4명 규모면 이걸로 충분하다.

### `if: github.ref == 'refs/heads/main'`

PR에서는 배포 스텝을 건너뛴다. 이게 없으면 **PR이 빨간불이 된다** — IAM 신뢰 관계가 `main`만 허용하므로 PR의 `refs/pull/N/merge`는 인증에서 거부된다.

---

## 5. 역할 분담이 줄 번호로 나뉜다

코드 책임은 개발자, 검증·배포 책임은 파이프라인 담당.

```
checkout / setup-node / npm ci / npm test / npm run build    ← 실패 시 개발자
AWS 인증 / S3 업로드 / invalidation                          ← 실패 시 담당자
```

| 실패 지점 | 책임 | 담당자가 할 일 |
|---|---|---|
| `npm ci` lock 불일치 | 개발자 | 로그 전달 |
| `npm test` 실패 | 개발자 | 로그 전달 |
| 타입 오류·번들 실패 | 개발자 | 로그 전달 |
| `setup-node` 버전 | 담당자 | 워크플로 수정 |
| OIDC 인증 | 담당자 | IAM 설정 |
| S3 업로드 | 담당자 | 권한·경로 |
| 캐시 때문에 안 바뀜 | 담당자 | 캐시 정책 |

**검증 스텝 실패는 고치지 않고 막는 것이 담당자의 일이다.**

품질 기준은 "CI가 통과하느냐"가 아니라 **"CI가 정확하게 판정하느냐"**. 가장 나쁜 실패는 깨졌는데 초록불이 뜨는 것(false green)이다. `npm test`를 넣지 않았다면 테스트가 깨진 코드도 배포됐을 것이다.

---

## 6. OIDC — 액세스 키 없이 AWS에 붙기

```
옛 방식:  액세스 키를 secrets에 저장 → 만료 없음, 유출되면 끝
지금:     GitHub이 발급한 1시간 신분증을 AWS에 제시 → 저장할 비밀이 없음
```

이 설계의 결과로 **`secrets`를 하나도 쓰지 않는다.** GitHub에 등록한 건 `vars` 2개(계정 ID, 버킷 이름)뿐이고, 유출돼도 그 자체로 아무것도 못 한다.

### IAM 롤에는 면이 두 개 있다 — 여기가 핵심

```
┌─────────── IAM 롤  gha-ss-fe-deploy ───────────┐
│                                                 │
│  ① 신뢰 관계              ② 권한 정책           │
│  "누가 빌릴 수 있나"        "빌리면 뭘 할 수 있나"│
│                                                 │
│  sub: repo:say-scene@.../  s3:PutObject         │
│       ss-fe@...:ref:...    s3:ListBucket        │
│       ↑                          ↓              │
└───────┼──────────────────────────┼──────────────┘
        │                          │
   OIDC provider              S3 버킷
```

**두 면은 서로를 모른다.**

- ①만 맞고 ②가 비면 → 인증 성공, 업로드에서 `AccessDenied`
- ②만 맞고 ①이 틀리면 → 인증 단계에서 거부

AWS 콘솔에서 역할을 열면 `[권한]` `[신뢰 관계]` 탭이 나뉘어 있는 이유다.

### 롤은 "사용자"가 아니다

```
IAM 사용자: 영구 계정, 액세스 키를 가짐        ← 안 씀
IAM 롤:     빌려 쓰는 권한 꾸러미, 주인 없음    ← 이걸 씀
```

GitHub Actions는 AWS에 계정이 없다. 신분증을 보여주고 롤을 1시간 빌린다.

### 전체 흐름 8단계

```
①  워크플로 → GitHub     "신분증 발급해줘"       (permissions: id-token: write 필요)
②  GitHub → 워크플로     서명된 JWT
                           iss: token.actions.githubusercontent.com
                           sub: repo:say-scene@345080911/ss-fe@1426408350:ref:refs/heads/main
                           aud: sts.amazonaws.com
③  워크플로 → AWS STS    JWT + 롤 ARN 제시
④  STS 검사 1            iss가 등록된 provider인가?    ← OIDC provider
⑤  STS 검사 2            롤의 신뢰 관계와 sub 일치?    ← 신뢰 관계
⑥  STS → 워크플로        1시간 임시 자격증명
                           로그: Authenticated as assumedRoleId AROA...:GitHubActions
⑦  워크플로 → S3         aws s3 sync/cp
                           → 롤의 권한 정책 확인        ← 권한 정책
⑧  소멸                  VM 삭제 + 1시간 만료
```

**④⑤는 "누구냐"(인증), ⑦은 "뭘 해도 되냐"(인가).** 이 경계가 롤의 두 탭으로 나뉘어 있다.

⑥에서 받은 자격증명이 환경변수에 심기므로 **이후 `aws` 명령은 아무 설정 없이 자동 인증된다.**

### AWS는 GitHub에 물어보지 않는다

```
AWS가 하는 일:    받은 문자열 == 정책에 적힌 문자열  →  비교만
AWS가 안 하는 일: GitHub에 "이 레포 맞나?" 조회
```

AWS는 `say-scene`이 뭔지 모른다. **글자가 같은지만 본다.** 그래서 방향이 한쪽이다.

```
GitHub이 보내는 값 = 기준 (우리가 못 고름)
AWS 정책에 적는 값 = 그 기준에 맞춰야 함
```

열쇠는 GitHub이 깎는다. 우리는 자물쇠 홈을 맞출 수밖에 없다.

### 비교가 두 개다

| | 값 | 어디에 | 질문 |
|---|---|---|---|
| ① 롤 ARN | `arn:aws:iam::944296702922:role/gha-ss-fe-deploy` | 워크플로 | 어느 롤인가 (주소) |
| ② `sub` | `repo:say-scene@345080911/ss-fe@1426408350:ref:refs/heads/main` | 롤의 신뢰 정책 | 빌릴 자격이 있나 (신원) |

**주소가 맞았다는 게 들어갈 수 있다는 뜻은 아니다.** 주소는 맞게 찾아갔지만 문 앞에서 신분증이 명단과 안 맞아 못 들어간 상황.

그래서 에러가 `Not authorized`(자격 없음)였고 `NoSuchEntity`(없음)가 아니었다. ARN을 먼저 확인한 것은 ①을 배제해 ②로 범위를 좁히는 순서였다.

### GitHub과 AWS는 다른 두 세계다

```
GitHub          AWS
조직 say-scene   계정 944296702922
레포 ss-fe       리전 ap-northeast-2
```

둘을 잇는 건 신뢰 정책의 `sub` 문자열 하나뿐이다.

AWS 계정 번호는 **계정 전체**(청구·리소스 소유 단위)를 가리키고, 콘솔 로그인 ID는 그 안의 IAM 사용자다. 리소스는 개인 소유가 아니라 계정 소유이므로, 팀원에게 접근을 주려면 같은 계정 안에 IAM 사용자를 만든다.

버킷 이름에 계정 번호가 들어간 건 관례다 — 버킷 이름은 전 세계에서 유일해야 해서 겹치지 않게 섞는다. AWS가 그 안의 숫자를 해석하지는 않는다.

---

## 7. 실제로 막혔던 것 — immutable subject claim

### 증상

provider 정상, 롤 ARN 정상, 신뢰 정책도 "일반적으로 맞는 형식"이었는데 계속 거부됐다.

```
Could not assume role with OIDC: Not authorized to perform sts:AssumeRoleWithWebIdentity
```

### 원인

```bash
gh api repos/say-scene/ss-fe/actions/oidc/customization/sub
```

```json
{"use_default":true,"use_immutable_subject":true,
 "sub_claim_prefix":"repo:say-scene@345080911/ss-fe@1426408350"}
```

이 레포는 **immutable subject claim**이 켜져 있어서 `sub`에 이름과 **숫자 ID가 함께** 들어간다.

```
정책에 적어둔 값: repo:say-scene/ss-fe:ref:refs/heads/main
실제로 오는 값:   repo:say-scene@345080911/ss-fe@1426408350:ref:refs/heads/main
                               ↑ 조직 ID          ↑ 레포 ID
```

> 사고 당시 레포명은 `ss_fe`였고 2026-10-23에 `ss-fe`로 리네임했다.
> 그때 `sub`도 함께 바뀌어 신뢰 정책을 다시 고쳐야 했다 — 아래 "언제 바뀌나" 참고.

### 값 조립 방식

| 필드 | 뜻 |
|---|---|
| `sub_claim_prefix` | `sub`의 앞부분이 이 문자열로 고정 |
| `use_default: true` | 뒷부분은 기본 템플릿 그대로 (`:ref:refs/heads/<브랜치>`) |

```
repo:say-scene@345080911/ss-fe@1426408350  +  :ref:refs/heads/main
└───────── sub_claim_prefix ──────────┘      └─ 기본 템플릿 ─┘
```

### 왜 ID를 쓰나 — 이름 가로채기 방어

```
이름 기반의 구멍:
  조직 say-scene 을 삭제하거나 이름을 바꾸면 그 이름이 빈다
  → 제3자가 같은 이름의 조직·레포를 만들어
  → 똑같은 sub 를 발급받아
  → 당신의 AWS 롤을 그대로 사용
```

이름은 반납되고 재사용된다. 조직 ID `345080911`, 레포 ID `1426408350`은 GitHub 전체에서 영구 고유하고 재사용되지 않는다.

**대가:** 정책을 사람이 읽기 어렵다. 그리고 `sub`에는 ID뿐 아니라 **이름도 함께** 들어가므로, 레포나 조직 이름을 바꾸면 ID가 그대로여도 `sub`가 달라져 정책을 고쳐야 한다. 레포를 새로 만들어 옮기면 ID까지 달라진다.

### 언제 바뀌나

`sub_claim_prefix`는 이름과 ID가 섞여 있다. 이름 쪽이 리네임에 반응한다.

| 부분 | 언제 바뀌나 |
|---|---|
| `say-scene` (조직 이름) | 조직 이름을 바꿀 때 |
| `@345080911` (조직 ID) | 조직을 새로 만들 때만 |
| `ss-fe` (레포 이름) | **레포 이름을 바꿀 때** — 2026-10-23에 실제로 겪음 |
| `@1426408350` (레포 ID) | 레포를 새로 만들 때만 |
| `:ref:refs/heads/main` | `main` push면 항상 이 값 |

한 번 맞추면 모든 `main` push에서 작동한다.
단 **레포나 조직 이름을 바꾸면 다시 맞춰야 한다.**

### 교훈

- IAM 롤을 만들기 **전에** 조회한다. 레포마다 ID가 달라서 접두사도 다르다.
- **레포나 조직 이름을 바꾸면 반드시 다시 조회해서 신뢰 정책을 고친다.**
  이름이 `sub`에 들어가므로 코드를 한 줄도 안 건드려도 리네임만으로 배포가 막힌다.
  증상은 처음과 동일한 `Not authorized to perform sts:AssumeRoleWithWebIdentity`다.

```bash
gh api repos/say-scene/<레포>/actions/oidc/customization/sub --jq .sub_claim_prefix
```

---

## 8. 업로드가 명령 두 개인 이유

`vite build` 결과:

```
dist/index.html                   ← 이름 고정
dist/assets/index-2934mxgb.js     ← 해시 포함
dist/assets/index-CXwGsm7x.css    ← 해시 포함
```

| | 파일명 | 캐시 정책 | 근거 |
|---|---|---|---|
| asset | 해시 포함 | `max-age=31536000, immutable` | 내용이 바뀌면 파일명도 바뀌므로 1년 캐시해도 안 상함 |
| `index.html` | 고정 | `no-cache` | 캐시하면 사용자가 옛 버전을 계속 받음 |

`aws s3 sync`는 전체에 같은 `--cache-control`을 적용하므로 한 번에 못 한다.

`no-cache`는 "저장 금지"가 아니라 **"쓸 때마다 서버에 물어봐"** 다. 이 파일만 매번 확인하면 안에 적힌 asset 참조가 최신으로 바뀌면서 나머지가 따라온다.

### 순서: asset → index.html

```
올바름:  asset → index.html    새 HTML이 가리키는 asset이 이미 있음        ✓
거꾸로:  index.html → asset     그 사이 몇 초간 없는 파일을 가리킴 → 깨짐  ✗
```

### `--delete`를 쓰지 않는다

옛 asset이 S3에 남는다. 그래서

- 배포 중 열려 있던 탭이 옛 asset을 요청해도 404가 안 난다
- 롤백할 때 옛 번들이 살아 있다 → `git revert`만으로 복구된다 (S3 버저닝 불필요)

번들 하나가 260KB 정도라 쌓여도 무시할 수준이다. 나중에 lifecycle rule로 정리한다.

### 매 런마다 전체가 다시 올라간다

`sync`가 변경분만 올린다고 생각하기 쉽지만 아니다. 비교 기준이 **크기와 수정 시각**이고, CI는 매번 새로 빌드하므로 로컬 파일의 수정 시각이 항상 S3 객체보다 최신이다. 그래서 `dist/` 전체가 매번 재업로드된다.

실측 — 코드 변경이 전혀 없는 `workflow_dispatch` 런인데도 동일 파일이 다시 올라갔다.

```
run 39759206744   index-BbsUUV-p.css  index-CMwohPtJ.js  sayscene-logo.png
run 39899674140   index-BbsUUV-p.css  index-CMwohPtJ.js  sayscene-logo.png
```

`--delete`를 쓰지 않는 것과 합치면 버킷이 두 부류로 갈린다.

| 객체 | 매 배포에서 |
|---|---|
| 현재 `dist/`에 있는 것 | 재업로드된다 |
| `dist/`에 없는 구 asset | 다시 손대지 않는다 |

### 객체에 커밋 SHA를 심는다

해시 파일명만으로는 어느 배포에서 나온 것인지 알 수 없다. 업로드할 때 출처를 메타데이터로 박는다.

```
--metadata "commit=${{ github.sha }},run=${{ github.run_id }}"
```

**S3 콘솔 → 객체 클릭 → 메타데이터** 에서 본다. 목록 화면에는 안 보이고 상세에만 나온다.

```
시스템 정의   Cache-Control      no-cache
사용자 정의   x-amz-meta-commit  62e7ae0246855a52325ebde7e0cf4fcbbbcbc5a8
사용자 정의   x-amz-meta-run     39900645101
```

`x-amz-meta-run` 값은 Actions 런 ID와 일치한다. 즉 객체 하나만 보고 어느 커밋·어느 런에서 나왔는지 AWS 안에서 자족적으로 확인된다.

위의 재업로드 동작과 맞물려 **메타데이터가 없는 객체 = 현재 배포에 포함되지 않은 잔여물**이라는 신호가 공짜로 생긴다. lifecycle rule로 정리할 때의 판별 기준이 된다.

### invalidation은 `/index.html`만 (B구간)

CloudFront는 S3 앞에서 따로 캐시한다. S3를 바꿔도 모른다. 그래서 `index.html`만 "버려라"고 알려준다.

asset은 파일명이 새로워서 CloudFront가 **처음 보는 파일**로 취급하고 알아서 가져온다. 무효화할 게 없다. 월 1,000건까지 무료이고, `/*`로 전체를 무효화하면 과금 대상이 되고 느리다.

---

## 9. 권한이 삼중으로 좁다

```
누가:   say-scene/ss-fe 의 main 브랜치만         (신뢰 관계)
무엇을: s3:PutObject, s3:ListBucket 만           (권한 정책)
어디에: 그 버킷 하나만                            (권한 정책 Resource)
```

이 롤이 탈취돼도 그 버킷에 파일을 올리는 것 외에는 못 한다. PR 워크플로는 `sub`가 달라 AWS에 아예 붙을 수 없다.

`DeleteObject`를 넣지 않은 것은 `--delete`를 쓰지 않기 때문이다.

### `Variables`와 `Secrets`

| | `vars.X` | `secrets.X` |
|---|---|---|
| 로그 표시 | 값 그대로 | `***` 마스킹 |
| 나중에 다시 보기 | 콘솔에서 보임 | 불가 |

계정 ID와 버킷 이름은 비밀이 아니고, `AccessDenied`가 났을 때 로그에서 버킷 이름을 눈으로 확인할 수 있어야 한다. 그래서 `vars`다.

주의: `vars` 이름을 틀리면 **조용히 빈 문자열**이 되어 `arn:aws:iam:::role/...` 처럼 계정 자리가 빈다. 에러 메시지가 "ARN 형식 오류"로 나와 원인 찾기가 어렵다.

---

## 10. 왜 워크플로가 레포 안의 파일인가

옛날에는 빌드 설정이 Jenkins 서버의 웹 화면에 있었다. 지금은 전부 레포 안의 파일이다.

| 이유 | |
|---|---|
| 코드와 빌드 방식이 같이 움직인다 | 3개월 전 커밋을 체크아웃하면 그 시점의 워크플로도 따라온다 |
| 브랜치마다 다를 수 있다 | 브랜치에서 빌드 방식을 실험하고 머지한다 |
| 리뷰와 이력이 공짜 | `git log .github/workflows/` 로 누가 언제 뭘 바꿨는지 나온다 |
| 되돌릴 수 있다 | `git revert` |

ArgoCD와 같은 논리다.

```
옛 방식                  지금
Jenkins 웹 화면    →    레포 안 .github/workflows/
kubectl 명령어     →    레포 안 ss-k8s 매니페스트
```

둘 다 한 문장으로 요약된다 — **"조작"이 아니라 "git에 적힌 상태"로 다룬다. 그러면 이력·리뷰·롤백이 공짜로 생긴다.**

로컬에 파일이 있는 이유는 "그래야 해서"가 아니라 에디터가 편해서다. GitHub 웹에서 직접 고쳐도 똑같이 동작한다.

---

## 11. `ss-be`가 다른 점

| | `ss-fe` | `ss-be` |
|---|---|---|
| 1단계 | 동일 | 동일 (런타임만 Java 21) |
| 산출물 | `dist/` | 컨테이너 이미지 |
| 어디에 | S3 | ECR |
| 배포 주체 | Actions (직접) | ArgoCD (간접) |
| 초록불의 뜻 | **배포 완료** | **배포 시작 전** |
| 인증 | OIDC 1개 | OIDC + **GitHub App 토큰** 2개 |
| 소스 레포 변경 | 워크플로만 | **`build.gradle.kts`에 Jib 플러그인도** |
| 확인할 곳 | Actions 탭 | Actions 탭 + **ArgoCD UI** |

`ss-be`는 `ss-k8s`에 커밋해야 하는데 기본 `GITHUB_TOKEN`으로는 남의 레포에 못 쓴다 → GitHub App 토큰이 필요하다.

이미지는 **Jib으로 Dockerfile 없이** 만든다. 단 Jib은 인증 정보를 `~/.docker/config.json`에서 읽으므로 `docker login`이 필요하다(데몬은 안 띄움). `ss-triton`은 반대로 Dockerfile + `docker buildx`를 쓴다 — Jib은 Java 앱 전용이다.

### 순서 의존성 — 이미지가 매니페스트보다 먼저

```
올바름:  jib → ECR → ss-k8s 커밋 → ArgoCD → kubelet pull            ✓
거꾸로:  ss-k8s 커밋 → ArgoCD → pull → 이미지 없음 → ImagePullBackOff ✗
```

매니페스트는 **문자열 하나**만 들고 있다. 이미지 내용은 모른다. "이 주소에서 꺼내 와라"는 참조일 뿐이다. 워크플로 스텝이 순차 실행되므로 jib 스텝을 커밋 스텝보다 위에 두면 자동으로 보장된다.

### `ss-triton` 주의점

- **`paths` 필터가 필요하다.** 빌드가 수십 분이고 디스크 부족 위험이 있어서 README만 바꾼 push에 돌면 안 된다. 소스 레포인 `ss-fe`/`ss-be`는 필요 없다(빠뜨리는 쪽이 더 위험).
- **도커 레이어 캐시가 안 먹는다.** VM이 매번 새 것이라 매번 베이스 이미지 전체를 받는다. Actions 캐시는 레포당 10GB 제한이라 수십 GB 이미지에 안 맞는다.
- PoC에선 로컬/EC2에서 수동 빌드하고 자동화를 미루는 것이 현실적이다.
- 배포 단위가 `이미지 SHA + AMI ID + 모델 revision + 모델 설정` 묶음이다. 이미지 SHA만 되돌려도 복구가 보장되지 않는다.

---

## 12. 알아둘 것 — 배포 후 `/api` 호출은 실패한다

```
개발:  브라우저 → vite dev server → 프록시(/api → localhost:8080)
운영:  브라우저 → CloudFront → /* → S3 → /api/v1 없음 → 실패
```

`vite.config.ts`의 프록시는 dev 전용이다. CloudFront behavior가 `/*` → S3 하나뿐이라 API 호출이 S3에서 파일을 찾다가 실패한다.

`ss-be`가 뜨고 ALB가 생긴 뒤 `/api/*` behavior를 추가해야 해결된다. 그때까지 "화면은 뜨는데 API가 안 된다"가 나오면 **코드 문제가 아니라 아직 안 만든 인프라** 때문이다.

덤으로, CloudFront same-origin이라 FE가 API를 `/api/v1` 상대 경로로 부른다 → **빌드 타임 환경변수가 필요 없다.** 번들이 환경 독립적이고 CI secret이 줄어든다.

---

## 13. 런타임 요청 흐름 (배포와 다른 경로)

```
브라우저 → CloudFront ─ /*     → S3            (ss-fe)
                      └ /api/* → ALB → ss-be 파드 (general 노드)
                                            ↓ 클러스터 내부
                                       Triton 파드 (gpu 노드)
                                         ├ vLLM (Qwen3-8B)
                                         └ faster-whisper
```

외부 접점은 `ss-be` 하나뿐이다. 브라우저는 Triton의 존재를 모르고, Triton에는 ALB도 인그레스도 없다. 같은 도메인이라 CORS 설정도 필요 없다.

---

## 14. 미해결 항목

| | 내용 |
|---|---|
| git 커밋 신원 | 작성자가 `ai-ncw <likencw@naver.com>`. GitHub 계정은 `namchn` → contribution 그래프에 안 잡힐 수 있음 |
| `.DS_Store` | `ss-fe`는 `.gitignore`에 있음. `ss-be`, `ss-triton`, `ss-k8s`, `ss-docs`는 아직 |
| PRD §6.10 레포 이름 | PRD는 `frontend`/`backend`/`triton`/`k8s-manifests`, 실제는 `ss-*`. `ss-docs`는 표에 없음 |
| CI 경고 2개 | `checkout@v4`·`setup-node@v4` Node 20 deprecation (Node 24로 강제 실행 중) / `ubuntu-latest` → Ubuntu 26 이전 예고. **둘 다 동작 영향 없음.** 깨지면 `@v5` / `ubuntu-24.04`로 고정 |
