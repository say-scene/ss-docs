# CPU Whisper STT 세팅 매뉴얼

> 상태: **운영 중 (실도메인 연동 확인, 간헐 오류 원인 미확인)** · 작성일: 2026-10-28
> 관련: [2026-10-27 이슈 기록](../issue/20261027/README.md) · ss-triton tag `triton-gpu-v1`(교체 전 GPU Triton 원본)

---

## 0. 요약

- GPU quota 미승인으로 Triton(Qwen3-8B FP8 + Whisper GPU)을 걷어내고, **ss-triton 레포 내용을 CPU Whisper STT 서버로 교체**했다.
- **이름은 `ss-triton`을 유지한다.** 레포·ECR·IAM·CI·ss-k8s 경로(`apps/ss-triton`)를 그대로 쓰기 위해서다. Pod·Service·이미지 이름은 `ss-triton`이지만 내용은 Whisper STT다.
- STT는 **control plane 노드(t3.xlarge)** 에 CPU limit 2로 올린다. LLM(vLLM)은 별도 m7i 노드(`apps/llm`)에서 돈다.
- ss-be와의 계약(KServe v2 JSON, `POST /v2/models/whisper_stt/infer`)은 Triton 시절과 같아 **BE 코드 변경은 없다.** 주소만 `STT_BASE_URL`로 바꿨다.

---

## 1. 구성

```
브라우저 ─(multipart audio)─▶ CloudFront ─▶ ALB ─▶ ss-be  POST /api/v1/stt
                                                     │ base64 → KServe v2 JSON
                                                     ▼
                                    Service ss-triton:8001 (ClusterIP)
                                                     ▼
                                    Pod ss-triton (control plane, whisper_server.py)
                                      ├ POST /v2/models/whisper_stt/infer
                                      ├ GET  /health
                                      └ GET  /metrics  ◀── Prometheus (ServiceMonitor ss-triton)
```

| 위치 | 파일 | 역할 |
|---|---|---|
| ss-triton | `whisper_server.py` | faster-whisper(CPU, int8) + HTTP 서버. 대기열·동시 처리 수·지표 포함 |
| ss-triton | `Dockerfile` | `python:3.12-slim`, 비root(uid 10001), 모델은 기동 시 HF에서 받음 |
| ss-triton | `requirements.txt` | `faster-whisper==1.2.1`, `av==16.0.1`, `prometheus-client==0.21.1` |
| ss-triton | `.github/workflows/ecr-push.yml` | main push → ECR `ss-triton:<SHA7>` → ss-k8s 태그 자동 갱신 |
| ss-k8s | `apps/ss-triton/` | Deployment·Service·kustomization (이미지 태그는 CI가 갱신) |
| ss-k8s | `apps/ss-be/01_configmap.yaml` | `STT_BASE_URL: http://ss-triton:8001`, `STT_TIMEOUT_SECONDS: 120` |
| ss-k8s | `platform/kube-prometheus-stack/prometheus/values.yaml` | `additionalServiceMonitors`의 `ss-triton` 항목 |

---

## 2. 결정 사항과 근거

| 결정 | 근거 |
|---|---|
| Triton 대신 경량 Python 서버 | CPU에서는 Triton의 동적 배칭·GPU 공유 이점이 없고, 공식 이미지가 CUDA 기반으로 크다. 대기열·지표는 서버에 직접 넣었다 |
| control plane에 배치 | STT는 요청 시 수 초만 CPU를 쓰는 패턴이라 burstable(t3)과 맞는다. 별도 노드 비용·기동 지연이 없다 |
| CPU limit 2 (LLM과 달리 limit 설정) | etcd·apiserver·CoreDNS·Argo CD와 노드를 공유한다. 클러스터 안정성이 STT 지연보다 우선 |
| 기본 모델 `small.en` | `large-v3-turbo`는 encoder가 large와 같아 CPU 2코어에서 회화용으로 느릴 것으로 추정. 환경변수로 바꿔 비교한다(§6) |
| 동시 계산 1 + 대기열 10초 | CPU 2코어에서 동시 계산을 늘려도 처리량은 같고 지연만 늘어난다. 겹친 요청은 즉시 거절 대신 최대 10초 기다린다 |
| 모델을 이미지에 굽지 않음 | 모델 비교를 쉽게 하려고. `small.en`은 수백 MB라 기동 시 다운로드를 감수 |
| 이름 `ss-triton` 유지 | 미니 프로젝트라 레포·ECR·IAM 변경 없이 CI/CD를 그대로 쓰는 쪽을 택함 |

---

## 3. 서버 동작 (`whisper_server.py`)

### 3-1. API

| 경로 | 요청 / 응답 |
|---|---|
| `POST /v2/models/whisper_stt/infer` | 요청 `{"inputs":[{"name":"AUDIO","shape":[1],"datatype":"BYTES","data":["<base64>"]}]}` → 응답 `{"outputs":[{"name":"TEXT","shape":[1],"datatype":"BYTES","data":["<text>"]}]}` |
| `GET /health` | 모델 로드 후 200 `{"ready": true}`. 서버는 모델 로드가 끝난 뒤 포트를 연다 |
| `GET /metrics` | Prometheus 텍스트 형식 |

입력 제약: 디코딩 후 음성 길이 0.5초 이상 31초 이하, 오디오 2MiB 이하, 요청 본문 3MiB 이하. 인식은 `language='en'`, `vad_filter=True`, `beam_size=1`, `temperature=0`.

### 3-2. 응답 코드와 BE 매핑

| STT 서버 | 상황 | ss-be 응답 (`SttController`) |
|---|---|---|
| 200 | 성공 | 200 `{"text": ...}`. 빈 문자열이면 422 `NO_SPEECH` |
| 503 `STT_BUSY` | 슬롯을 `STT_QUEUE_TIMEOUT`초 넘게 기다림 | 503 `STT_BUSY` |
| 422 `INVALID_AUDIO` | 형식·길이·base64 오류 | 422 `INVALID_AUDIO` |
| 422 `STT_FAILED` | 그 밖의 예외 | 422 `INVALID_AUDIO` (BE가 422를 구분하지 않음) |
| 413 `AUDIO_TOO_LARGE` | 크기 초과 | 502 `STT_FAILED` (BE는 업로드 단계에서 2MiB를 먼저 막음) |
| (응답 없음) | BE timeout 120초 초과 | 504 `STT_TIMEOUT` |

### 3-3. 환경변수

| 변수 | 이미지 기본값 | 운영값 (Deployment) | 설명 |
|---|---|---|---|
| `WHISPER_MODEL` | `small.en` | `small.en` | 모델 이름. HF에서 받는다 |
| `WHISPER_CPU_THREADS` | `2` | `2` | CPU limit과 맞춘다 |
| `STT_CONCURRENCY` | `1` | `1` | 동시 계산 수. faster-whisper `num_workers`에도 같은 값 |
| `STT_QUEUE_TIMEOUT` | `10` | `10` | 슬롯 대기 상한(초) |
| `STT_HOST` | `0.0.0.0` | — | 스크립트 직접 실행 시 기본값은 `127.0.0.1` |
| `HF_HOME` | `/cache/huggingface` | — | emptyDir `/cache`에 모델 저장 |

### 3-4. 지표

| 지표 | 종류 | 내용 |
|---|---|---|
| `stt_requests_total{result}` | counter | `ok`, `busy`, `invalid`, `failed` |
| `stt_queue_wait_seconds` | histogram | 처리 슬롯 대기 시간 |
| `stt_inference_seconds` | histogram | Whisper 처리 시간 (디코딩 제외) |
| `stt_audio_seconds` | histogram | 입력 음성 길이 |

PromQL 예시:

```promql
# 처리 시간 p95
histogram_quantile(0.95, sum by (le) (rate(stt_inference_seconds_bucket[5m])))
# RTF (처리 시간 ÷ 음성 길이)
rate(stt_inference_seconds_sum[5m]) / rate(stt_audio_seconds_sum[5m])
# 거절 비율
sum(rate(stt_requests_total{result="busy"}[5m])) / sum(rate(stt_requests_total[5m]))
```

로그에는 요청마다 `stt model=... audio_s=... elapsed_s=... rtf=...` 한 줄이 남는다. **발화 텍스트와 음성은 로그에 남기지 않는다.**

---

## 4. 세팅 절차

### 4-1. 이미지 빌드 (CI)

ss-triton `main`에 push하면 `ecr-push` workflow가 실행된다.

1. `push` job: `py_compile` 검증 → `linux/amd64` 빌드 → `944296702922.dkr.ecr.ap-northeast-2.amazonaws.com/ss-triton:<SHA7>` push
2. `update-tag` job: ss-k8s `apps/ss-triton/kustomization.yaml`의 `newTag`를 `<SHA7>`로 바꿔 커밋 (`say-scene-ss-k8s-writer[bot]`)

`update-tag`는 kustomization의 이미지 이름이 **정확히 `ss-triton` 하나**이고 `    newTag:` 줄이 **정확히 하나**인지 검사한다. 이 구조를 바꾸면 CI가 실패한다.

첫 실행 결과(2026-10-27): run `40504601480` 성공, push 55초 / update-tag 6초, ss-k8s `fdb1857`에서 `newTag: "e47e872"`.

### 4-2. ss-k8s 매니페스트

`apps/ss-triton/01_deployment.yaml` 주요 값:

| 항목 | 값 |
|---|---|
| 배치 | `nodeSelector: node-role.kubernetes.io/control-plane: ""` + 해당 toleration |
| 포트 | `http` 8001 (API·`/health`·`/metrics` 공용) |
| resources | requests cpu 1 / memory 1Gi, limits cpu 2 / memory 3Gi (turbo 비교 고려) |
| 볼륨 | emptyDir `/cache` (sizeLimit 4Gi). Pod 재시작 시 모델 재다운로드 |
| probe | startup `/health` 5초 × 60회(최대 5분), readiness 10초, liveness 10초·timeout 5초·6회 |

### 4-3. Argo CD Application

Application 정의는 Git에 없고 웹 콘솔에서 등록한다.

| 항목 | 값 |
|---|---|
| Application | `ss-triton` |
| Repository / Revision | `https://github.com/say-scene/ss-k8s.git` / `main` |
| Path | `apps/ss-triton` |
| Destination namespace | `sayscene` (ss-be와 같은 namespace여야 `http://ss-triton:8001`이 해석됨) |

### 4-4. 기동

1. Argo CD에서 `ss-triton` Application Sync.
2. replicas를 1로 올린다. **Git의 `replicas`는 아직 `0`이다.** Argo CD UI에서만 올렸다면 다음 Sync 때 0으로 돌아가므로, 확인 후 Git도 1로 커밋한다.
3. 기동 확인:
   ```bash
   kubectl -n sayscene get pod -l app=ss-triton
   kubectl -n sayscene logs deploy/ss-triton | head
   ```
   `Whisper 준비 완료: http://0.0.0.0:8001 (threads=2, concurrency=1, queue_timeout=10.0s)`가 나오면 준비 완료.

### 4-5. ss-be 연결

1. Argo CD에서 `sayscene` Application Sync → ConfigMap `STT_BASE_URL` 반영 확인:
   ```bash
   kubectl -n sayscene get cm ss-be-config -o jsonpath='{.data.STT_BASE_URL}{"\n"}'
   ```
2. **ss-be 재시작이 필요하다.** `envFrom`은 컨테이너 시작 시에만 읽힌다([이슈 04](../issue/20261027/04-configmap-not-applied.md)).
   ```bash
   kubectl -n sayscene rollout restart deploy/ss-be && kubectl -n sayscene rollout status deploy/ss-be
   ```
3. ss-be의 STT HttpClient는 HTTP/1.1로 고정돼 있어야 한다(ss-be `72847af` 이후). HTTP/2(h2c) 업그레이드 요청은 Python 서버가 거부할 수 있다([이슈 03](../issue/20261027/03-be-vllm-400-h2c-upgrade.md)).

### 4-6. Prometheus

`kube-prometheus-stack` Application은 수동 Sync다. `additionalServiceMonitors`의 `ss-triton` 항목(`port: http`, `path: /metrics`)을 반영하려면 Sync한다.

---

## 5. 검증

### 5-1. 클러스터 내부에서 직접 호출

이미지에 `curl`이 없으므로 Python으로 호출한다.

```bash
kubectl -n sayscene exec deploy/ss-triton -- python -c "import urllib.request;print(urllib.request.urlopen('http://localhost:8001/health').read().decode())"
kubectl -n sayscene exec deploy/ss-triton -- python -c "import urllib.request;print(urllib.request.urlopen('http://localhost:8001/metrics').read().decode())" | grep -E "^stt_"
```

### 5-2. 로컬 Mac에서 BE 경유 호출

```bash
say -o hello.wav --data-format=LEI16@16000 "Hi, can I get an iced latte with oat milk, please?"
curl -s -X POST "https://<도메인>/api/v1/stt" -F "audio=@hello.wav;type=audio/wav" -w '\nHTTP %{http_code}  total %{time_total}s\n'
```

### 5-3. 로컬 컨테이너 검증 기록 (2026-10-27, Mac Docker `--cpus 2`)

| 시험 | 결과 |
|---|---|
| 4.13초 합성 음성 (`small.en`, arm64) | 1.62~1.81초 (RTF 약 0.4) |
| 같은 음성 (`linux/amd64` 에뮬레이션 빌드) | 정상 응답 |
| 인식 결과 | "Hi, can I get an **ice-d** with **auto** milk, please?" (원문 iced latte / oat milk) |
| 동시 4건, 대기 한도 3초 | 200 2건(1건은 1.65초 대기 후 처리), 503 2건. 지표 `ok 2`, `busy 2`와 일치 |

Mac 수치는 참고용이다. control plane(t3, x86)에서는 더 느릴 수 있으므로 §3-4 지표로 실측한다.

---

## 6. 모델 비교 절차 (`small.en` ↔ `large-v3-turbo`)

1. `apps/ss-triton/01_deployment.yaml`의 `WHISPER_MODEL` 값을 바꿔 커밋·Sync. 이미지 재빌드는 필요 없다.
2. 기동 시 모델을 새로 받는다(turbo는 약 1.6GB로 추정). startupProbe 한도(5분) 안에 끝나는지 확인한다.
3. 같은 녹음 파일 묶음으로 각각 요청하고 로그의 `elapsed_s`, `rtf`와 인식 결과를 비교한다.
4. 판단 기준 예: 10초 이하 발화에서 처리 시간 p95 3초 이하. CPU 크레딧(CloudWatch `CPUCreditBalance`)도 함께 본다.

---

## 7. 주의 사항

| 항목 | 내용 |
|---|---|
| CloudFront 30초 | CloudFront 오리진 응답 대기 기본값은 30초다. 대기(최대 10초) + 처리 시간이 30초를 넘으면 BE timeout(120초)보다 먼저 504가 난다. turbo 비교나 대기 한도 조정 시 이 합을 넘지 않게 한다 |
| t3 CPU 크레딧 | control plane은 burstable이다. 크레딧이 줄면 STT 지연 측정값이 오염된다 |
| 동시 사용자 | 처리량은 CPU 2코어에 묶인다. 대기열은 "끊기지 않고 기다리게"만 할 뿐 처리량을 늘리지 않는다 |
| replicas | Git 값(`0`)과 실제 값이 다르면 Sync 시 0으로 돌아간다(§4-4) |
| 이름 혼동 | Pod·Service·이미지·Argo 앱은 `ss-triton`, 지표 이름은 `stt_*`다. 대시보드에 설명을 단다 |
| ServiceMonitor 중복 | `apps/ss-be/07_servicemonitor.yaml`(팀원 추가)과 values의 `additionalServiceMonitors.ss-be`가 같은 대상을 본다. 중복 수집 여부 정리 필요 |

---

## 8. 미결 사항

- **실도메인에서 간헐적 STT 오류**(2026-10-28 확인, RESTARTS 0). 원인 미확인. 관측성 세팅 완료 후 `stt_requests_total{result}`, 대기·처리 시간, 브라우저의 실패 응답 코드(CloudFront 504 여부)로 재검토한다. 대기열이 원인이면 Redis 대기열 + SSE 비동기 구조를 검토한다.
- `small.en` ↔ `large-v3-turbo` 실측 비교(§6).
- Git `replicas` 1 반영 여부 확인.

---

## 9. 되돌리기

- **설정만 되돌리기**: ss-k8s에서 해당 커밋을 revert하고 Sync.
- **Triton GPU 구성으로 복귀**: ss-triton tag `triton-gpu-v1`(`f75c574`)에서 브랜치를 만들어 복원한다. GPU 노드(NodePool)·GPU quota·ss-k8s `apps/ss-triton` GPU 매니페스트가 함께 필요하다.
