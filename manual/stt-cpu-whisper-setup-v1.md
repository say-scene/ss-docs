# CPU Whisper STT 세팅 매뉴얼

> 상태: **운영 중 (실도메인 연동 확인, 간헐 오류 원인 미확인)** · 작성일: 2026-10-28 · 갱신 2026-10-29 (전용 노드 이전 · 스레드 조합 실측 · KEDA 자동 확장)
> 관련: [2026-10-27 이슈 기록](../issue/20261027/README.md) · [KEDA 설치 매뉴얼](keda-install-v1.md) · ss-triton tag `triton-gpu-v1`(교체 전 GPU Triton 원본)

---

## 0. 요약

- GPU quota 미승인으로 Triton(Qwen3-8B FP8 + Whisper GPU)을 걷어내고, **ss-triton 레포 내용을 CPU Whisper STT 서버로 교체**했다.
- **이름은 `ss-triton`을 유지한다.** 레포·ECR·IAM·CI·ss-k8s 경로(`apps/ss-triton`)를 그대로 쓰기 위해서다. Pod·Service·이미지 이름은 `ss-triton`이지만 내용은 Whisper STT다.
- STT는 **전용 Karpenter NodePool `stt`(t3.large)** 에 노드 1대당 Pod 1개로 올린다. **KEDA가 부하에 따라 1~2개로 늘리고 줄인다.** LLM(vLLM)은 별도 m7i 노드(`apps/llm`)에서 돈다.
  - 2026-10-28까지는 control plane(t3.xlarge)에 CPU limit 2로 두었다. 2026-10-29 전용 노드로 옮겼다(§2).
- ss-be와의 계약(KServe v2 JSON, `POST /v2/models/whisper_stt/infer`)은 Triton 시절과 같아 **BE 코드 변경은 없다.** 주소만 `STT_BASE_URL`로 바꿨다.

---

## 1. 구성

```
브라우저 ─(multipart audio)─▶ CloudFront ─▶ ALB ─▶ ss-be  POST /api/v1/stt
                                                     │ base64 → KServe v2 JSON
                                                     ▼
                                    Service ss-triton:8001 (ClusterIP)
                                                     ▼
                                    Pod ss-triton × 1~2 (stt 노드 t3.large, whisper_server.py)
                                      ├ POST /v2/models/whisper_stt/infer
                                      ├ GET  /health
                                      └ GET  /metrics  ◀── Prometheus (ServiceMonitor ss-triton)
                                                            ▲
                                    KEDA ScaledObject ss-triton ─ PromQL로 Pod 수 결정
```

| 위치 | 파일 | 역할 |
|---|---|---|
| ss-triton | `whisper_server.py` | faster-whisper(CPU, int8) + HTTP 서버. 대기열·동시 처리 수·지표 포함 |
| ss-triton | `Dockerfile` | `python:3.12-slim`, 비root(uid 10001), 모델은 기동 시 HF에서 받음 |
| ss-triton | `requirements.txt` | `faster-whisper==1.2.1`, `av==16.0.1`, `prometheus-client==0.21.1` |
| ss-triton | `.github/workflows/ecr-push.yml` | main push → ECR `ss-triton:<SHA7>` → ss-k8s 태그 자동 갱신 |
| ss-k8s | `apps/ss-triton/` | Deployment·Service·ScaledObject(`03_scaledobject.yaml`)·kustomization (이미지 태그는 CI가 갱신) |
| ss-k8s | `platform/karpenter/nodepool-stt.yaml` | 전용 NodePool `stt`: t3.large, taint `sayscene.io/workload=stt:NoSchedule`, 최대 2대 |
| ss-k8s | `platform/keda/` | KEDA Application·values ([KEDA 설치 매뉴얼](keda-install-v1.md)) |
| ss-k8s | `apps/ss-be/01_configmap.yaml` | `STT_BASE_URL: http://ss-triton:8001`, `STT_TIMEOUT_SECONDS: 120` |
| ss-k8s | `platform/kube-prometheus-stack/prometheus/values.yaml` | `additionalServiceMonitors`의 `ss-triton` 항목 |

---

## 2. 결정 사항과 근거

| 결정 | 근거 |
|---|---|
| Triton 대신 경량 Python 서버 | CPU에서는 Triton의 동적 배칭·GPU 공유 이점이 없고, 공식 이미지가 CUDA 기반으로 크다. 대기열·지표는 서버에 직접 넣었다 |
| ~~control plane에 배치~~ → **전용 NodePool `stt`(t3.large)** (2026-10-29) | STT는 처리 중 물리 코어를 꽉 쓴다. control plane에 두면 부하 시 etcd·apiserver와 코어를 다투고, 늘릴 자리도 없다. taint로 STT만 올리고 Karpenter가 노드를 늘리게 했다 |
| 다른 Pod와 섞지 않음 (전용 노드 유지) | t3.large는 물리 1코어라 STT가 일할 때 남는 CPU가 거의 없다. 섞으면 STT·다른 Pod 모두 느려지고 Pod당 용량이 흔들려 KEDA 기준이 무너진다 |
| CPU 1500m·메모리 2Gi, request = limit (Guaranteed) | t3.large allocatable 1700m에서 DaemonSet 몫을 뺀 값. 노드당 STT 1개라 더 낮출 이유가 없다. 메모리는 실측 피크 1.19GiB 기준 |
| ~~c7i.large~~ 보류 | general AMI에서 Pod DNS가 실패했다. 주 NIC 이름 문제([이슈 01](../issue/20261027/01-llm-ami-ens5-primary-nic.md))와 같은 계열로 추정, 미확인. 일정상 t3.large로 확정 |
| 기본 모델 `small.en` | `large-v3-turbo`는 encoder가 large와 같아 CPU에서 회화용으로 느릴 것으로 추정. 환경변수로 바꿔 비교한다(§6) |
| 스레드 1 + 동시 계산 1 + 대기열 10초 | t3.large 실측(§5-4)에서 (2,1)·(1,1)·(1,2) 처리량이 같았다. 가장 단순한 (1,1)을 쓴다. 겹친 요청은 즉시 거절 대신 최대 10초 기다린다 |
| KEDA로 1~2개 자동 확장 | 처리량은 노드 수에만 비례한다. 기준은 CPU가 아니라 서버 안의 요청 수(처리 + 대기)와 503 비율([KEDA 설치 매뉴얼 §6](keda-install-v1.md#6-ss-triton-연결-scaledobject)) |
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
| `WHISPER_CPU_THREADS` | `2` | `1` | 요청 1건의 계산 스레드 수. t3.large에서는 2로 올려도 빨라지지 않았다(§5-4) |
| `STT_CONCURRENCY` | `1` | `1` | 동시 계산 수. faster-whisper `num_workers`에도 같은 값. 물리 코어 수보다 올려도 처리량이 늘지 않는다 |
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
| 배치 | `nodeSelector: sayscene.io/workload: stt` + toleration `sayscene.io/workload=stt:NoSchedule` |
| replicas | **Deployment에 두지 않는다.** KEDA(`03_scaledobject.yaml`)가 1~2로 정한다. 적으면 Sync마다 KEDA 값을 덮어쓴다 |
| 배포 전략 | RollingUpdate `maxSurge: 1`, `maxUnavailable: 50%`. Pod 1개면 새 Pod를 2번째 노드에 먼저 띄워 끊김이 없고, 2개면 하나씩 내리고 띄운다(노드 최대 2대) |
| 포트 | `http` 8001 (API·`/health`·`/metrics` 공용) |
| resources | requests = limits: cpu 1500m / memory 2Gi (Guaranteed). turbo 비교 시 메모리를 다시 잰다 |
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

1. KEDA가 먼저 설치돼 있어야 한다([KEDA 설치 매뉴얼](keda-install-v1.md)). ScaledObject CRD가 없으면 Sync가 실패한다.
2. Argo CD에서 `karpenter`(NodePool `stt`) → `ss-triton` 순서로 Sync. KEDA가 Pod를 1개로 맞추고, Karpenter가 stt 노드를 띄운다.
3. 기동 확인:
   ```bash
   kubectl -n sayscene get pod -l app=ss-triton -o wide
   kubectl -n sayscene get scaledobject,hpa
   kubectl -n sayscene logs deploy/ss-triton | head
   ```
   `Whisper 준비 완료: http://0.0.0.0:8001 (threads=1, concurrency=1, queue_timeout=10.0s)`가 나오고, ScaledObject가 `READY True`면 준비 완료.

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

Mac 수치는 참고용이다. 실제 노드 실측은 §5-4.

### 5-4. t3.large 단일 Pod 실측 (2026-10-29)

전용 t3.large 노드에 Pod 1개(CPU 1500m Guaranteed, `small.en`)를 띄우고, control plane에서 `bench.py`로 Service ClusterIP에 부하를 줬다. 음성은 `say`로 만든 3.5·9.1·18.4초 영어 WAV 3개를 돌려 썼다. 칸당 2분, 약 20건.

| threads, concurrency | 동시 요청 | 처리량 | p50 | p90 | 503 |
|---|---|---|---|---|---|
| 2, 1 | 1 / 2 / 4 | 0.165 / 0.163 / 0.161 rps | 5.94 / 12.29 / 13.96s | 7.20 / 13.20 / 15.42s | 0 / 0 / 20(+연결 끊김 2) |
| **1, 1 (채택)** | 1 / 2 / 4 | 0.164 / 0.161 / 0.165 rps | 6.07 / 12.44 / 12.95s | 7.07 / 13.28 / 15.40s | 0 / 0 / 22 |
| 1, 2 | 1 / 2 / 4 | 0.162 / 0.154 / 0.158 rps | 6.14 / 12.75 / 17.66s | 7.21 / 15.00 / 21.99s | 0 / 0 / 15(+연결 끊김 1) |

- **처리량은 조합과 무관하게 약 0.16 rps(분당 약 10건)다.** t3.large의 vCPU 2개는 물리 1코어의 하이퍼스레드로 알고 있고, int8 행렬 연산은 두 번째 하이퍼스레드의 덕을 거의 못 본다(추정).
- 동시 2건이면 p50이 두 배가 된다. Pod가 1건씩 차례로 처리한다는 뜻이다.
- 처리 약 6초, `STT_QUEUE_TIMEOUT` 10초라 **대기 요청이 2건을 넘으면 503이 난다.**
- "연결 끊김"(`URLError`)은 서버가 본문을 읽기 전에 503을 보내고 닫아서 생긴다. 사실상 503과 같다.
- 메모리 피크(working set) **1.19GiB**. 사용자 1명이 잠깐 썼을 때 "처리 + 대기 요청 수"는 **0.175**.

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
| t3 CPU 크레딧 | stt 노드(t3.large)도 burstable이다. 2026-10-29 확인 시 **unlimited**라 성능은 유지되지만 기준 성능(vCPU당 30%)을 넘긴 만큼 추가 요금이 붙는다 |
| 동시 사용자 | Pod 1개 처리량은 약 0.16 rps다. 대기열은 "끊기지 않고 기다리게"만 할 뿐 처리량을 늘리지 않는다. 늘리는 방법은 노드(Pod) 수뿐이다 |
| replicas | Deployment에 `replicas`를 다시 넣지 않는다. Sync마다 KEDA가 정한 값을 덮어쓴다 |
| 로그 수집 | Alloy(logs) DaemonSet에 `stt` taint toleration이 없어 **stt 노드의 ss-triton 로그가 Loki에 수집되지 않는다.** `kubectl logs`로 본다. 필요하면 Alloy values에 toleration을 추가한다 |
| 노드 접속 | stt 노드는 Karpenter가 띄워 SSH 키가 없다. 노드 안은 `kubectl debug node/<노드> -it --image=busybox:1.36`으로 본다 |
| 이름 혼동 | Pod·Service·이미지·Argo 앱은 `ss-triton`, 지표 이름은 `stt_*`다. 대시보드에 설명을 단다 |
| ServiceMonitor 중복 | `apps/ss-be/07_servicemonitor.yaml`(팀원 추가)과 values의 `additionalServiceMonitors.ss-be`가 같은 대상을 본다. 중복 수집 여부 정리 필요 |

---

## 8. 미결 사항

- **실도메인에서 간헐적 STT 오류**(2026-10-28 확인, RESTARTS 0). 원인 미확인. 관측성 세팅 완료 후 `stt_requests_total{result}`, 대기·처리 시간, 브라우저의 실패 응답 코드(CloudFront 504 여부)로 재검토한다. 대기열이 원인이면 Redis 대기열 + SSE 비동기 구조를 검토한다.
- `small.en` ↔ `large-v3-turbo` 실측 비교(§6).
- KEDA 자동 확장 동작 테스트(1 → 2 → 1, 노드 추가·회수). 테스트 후 최대 Pod 수와 stt NodePool limits를 운영값으로 정한다.
- c7i.large(7세대 Intel) Pod DNS 실패 원인 확인. general AMI의 networkd 주 NIC 매칭이 원인이면 AMI를 고쳐 다시 비교한다.
- 모델을 이미지에 넣을지 검토. 지금은 Pod가 뜰 때마다 HF에서 받아 새 노드 기동이 느리고, 외부 네트워크 문제 시 Pod가 뜨지 못한다.

---

## 9. 되돌리기

- **설정만 되돌리기**: ss-k8s에서 해당 커밋을 revert하고 Sync.
- **Triton GPU 구성으로 복귀**: ss-triton tag `triton-gpu-v1`(`f75c574`)에서 브랜치를 만들어 복원한다. GPU 노드(NodePool)·GPU quota·ss-k8s `apps/ss-triton` GPU 매니페스트가 함께 필요하다.
