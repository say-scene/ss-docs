# ss-fe — main push 한 장 요약

## 미리 준비 (1회성)

| 구분 | 내용 |
|---|---|
| **AWS** | ① S3 버킷 (퍼블릭 차단)<br>② OIDC provider (계정당 1개)<br>③ IAM 롤 `gha-ss-fe-deploy` = 신뢰관계(누가) + 권한정책(무엇을) |
| **GitHub** | Variables 2개 (`AWS_ACCOUNT_ID`, `S3_BUCKET`) · Secrets 0개 |
| **코드** | `.github/workflows/build.yml` |

---

## 실행 흐름

```text
git push origin main
 │
 ├─ [GitHub] build.yml 읽음 → on: branches[main] 일치 → 빈 VM 생성
 │           permissions: id-token: write  (⑥에서 쓸 토큰 범위)
 │
 │  ── VM 안 ── [검증 구간: 실패 시 개발자 책임] ──
 │
 ├─ ① checkout        코드 clone — VM이 비어 있어 첫 줄
 ├─ ② setup-node      Node 22 + npm 캐시
 ├─ ③ npm ci          lock 그대로 설치 (불일치면 중단)
 ├─ ④ npm test        vitest run → 14개
 ├─ ⑤ npm run build   tsc && vite → dist/
 │                      ├─ index.html                (이름 고정)
 │                      └─ assets/index-2934mxgb.js  (해시)
 │
 │  ── if main ── [배포 구간: 실패 시 내 책임] ──
 │
 ├─ ⑥ AWS 인증
 │      GitHub에 JWT 요청 → STS에 JWT + 롤ARN 제시
 │      → provider 등록 확인   (발급기관)
 │      → 신뢰관계 sub 비교    (신원 = 누구냐)
 │      → 1시간 자격증명 → 환경변수
 ├─ ⑦ asset 올림       immutable 1년  ← 권한정책 검사
 ├─ ⑧ index.html 올림  no-cache       ← asset 다음에
 │
 └─ VM 삭제 (dist/·자격증명 소멸)
      남는 것: S3 파일 + Actions 로그
```

### 단계별 정리

| 단계 | 스텝 | 내용 | 구간 |
|---|---|---|---|
| ① | checkout | 코드 clone — VM이 비어 있어 첫 줄 | 검증 (실패 시 개발자 책임) |
| ② | setup-node | Node 22 + npm 캐시 | 검증 |
| ③ | npm ci | lock 그대로 설치 (불일치면 중단) | 검증 |
| ④ | npm test | vitest run → 14개 | 검증 |
| ⑤ | npm run build | tsc && vite → `dist/`<br>`index.html` (이름 고정)<br>`assets/index-2934mxgb.js` (해시) | 검증 |
| ⑥ | AWS 인증 | GitHub에 JWT 요청 → STS에 JWT + 롤 ARN 제시<br>→ provider 등록 확인 (발급기관)<br>→ 신뢰관계 `sub` 비교 (신원 = 누구냐)<br>→ 1시간 자격증명 → 환경변수 | 배포 (main만, 실패 시 내 책임) |
| ⑦ | asset 올림 | `immutable` 1년 ← 권한정책 검사 | 배포 |
| ⑧ | index.html 올림 | `no-cache` ← asset 다음에 | 배포 |

---

## 이 순서가 전부 의미를 갖는 이유

### ⑤ → ⑥ : 검증이 배포 앞 = 게이트
- ④ 실패 → ⑤⑥⑦⑧ 건너뜀 → S3에 안 올라감
- main 직접 push는 허용. 깨진 코드는 main에 들어가도 운영엔 안 나감

### ⑥ → ⑦ : "누구냐" → "뭘 해도 되냐"
- 신뢰관계(누구냐) → 권한정책(뭘 해도 되냐)
- 롤의 두 탭. 서로 모름

### ⑦ → ⑧ : 캐시 정책이 다르므로 명령이 둘로 나뉨
- 해시 파일명 = `immutable` 가능 / 고정 이름 = `no-cache`
- asset 먼저여야 새 HTML이 가리키는 파일이 이미 존재
- `--delete` 없음 → 옛 asset 남아 열린 탭·롤백 보호

### PR
- ①~⑤만 실행, ⑥⑦⑧ skipped (신뢰관계가 main만 허용하므로)

---

## 현재 위치

| 상태 | 내용 |
|---|---|
| ✅ **완료** | 1단계 트리거 / 2단계 검증 / 3단계 A구간 S3 업로드 |
| ⏭️ **다음** | 3단계 B구간: CloudFront + OAC + SPA Function + invalidation<br>화면을 볼 수 있는 건 이때부터 |
| 🔜 **이후** | `ss-be` → `ss-triton` → `ss-k8s` |
