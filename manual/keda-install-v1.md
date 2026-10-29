# KEDA 설치 매뉴얼

> 상태: **설치 완료 (`Synced` + `Healthy`), ss-triton 자동 확장 동작 확인 (1 → 2 → 1, 노드 회수까지)** · 작성일: 2026-10-29 · 실측일: 2026-10-29
> 관련: [CPU Whisper STT 세팅 매뉴얼](stt-cpu-whisper-setup-v1.md) · [kube-prometheus-stack 설치 문서](../monitoring/kube-prometheus-stack-install-v1.md)
> ss-k8s 파일: `platform/keda/application.yaml` · `platform/keda/values.yaml` · `apps/ss-triton/03_scaledobject.yaml`

---

## 0. 요약

- ss-triton(CPU Whisper STT)을 부하에 따라 늘리고 줄이기 위해 **KEDA**를 설치했다.
- KEDA가 **Prometheus에 PromQL을 직접** 실행해 Pod 수를 정한다. **prometheus-adapter와 metrics-server는 필요 없다.**
- 설치는 Argo CD Application(Helm 차트 `kedacore/keda` **2.21.0**)으로 한다. Application은 웹 콘솔이 아니라 **`kubectl apply`로 만들었다**(§4).
- KEDA 구성요소 3개는 다른 platform 구성요소처럼 **control plane 노드**에 둔다.

---

## 1. 구성

| 항목 | 값 |
|---|---|
| 차트 | `kedacore/keda` `2.21.0` (repo `https://kedacore.github.io/charts`) |
| 클러스터 | Kubernetes v1.35 (KEDA 2.21 지원 범위 v1.34~v1.36) |
| namespace | `keda` |
| Argo CD | Application `keda` (namespace `argocd`), 수동 sync, **Server-Side Apply** |
| values | ss-k8s `platform/keda/values.yaml` (ref 소스, path 없음) |
| 배치 | control plane (`node-role.kubernetes.io/control-plane` nodeSelector + toleration) |

| 구성요소 | 역할 | request (cpu / memory) | memory limit |
|---|---|---|---|
| `keda-operator` | ScaledObject를 감시하고 HPA를 만든다 | 20m / 64Mi | 256Mi |
| `keda-operator-metrics-apiserver` | HPA에 external metrics를 제공한다 | 20m / 64Mi | 256Mi |
| `keda-admission-webhooks` | ScaledObject를 검증한다 | 10m / 32Mi | 128Mi |

- 차트 기본 request는 구성요소마다 100m / 100Mi(합계 300m)다. control plane에 부담이라 줄였다. **실측 전 추정값**이므로 Grafana에서 사용량을 보고 조정한다.

### 버전 선택 근거

- 2026-10-29 기준 최신 차트는 `2.21.0`(2026-09-23 배포)이다.
- `2.20.x`는 지원 범위가 v1.33~v1.35라 클러스터 버전(v1.35)이 범위의 끝에 걸린다. `2.21`은 v1.35가 범위 가운데에 있다.
- 출처: [KEDA Kubernetes 호환표](https://keda.sh/docs/latest/operate/cluster/)

### 왜 prometheus-adapter·metrics-server가 필요 없나

| 정보를 얻는 곳 | 알려 주는 것 | 이 클러스터 |
|---|---|---|
| metrics-server | Pod별 CPU·메모리 사용량 | **없음** (`kubectl top` 안 됨) |
| Prometheus | STT 서버가 기록한 대기 시간·503·처리 시간 | 있음, ss-triton `/metrics` 수집 중 |

- STT는 요청을 처리하는 순간 CPU를 항상 100% 쓴다. "1건 처리 중"과 "1건 처리 + 5건 대기"가 CPU로는 똑같이 보여서 CPU 기준으로는 늘릴 시점을 잡을 수 없다.
- 그래서 Prometheus 지표를 쓰고, KEDA의 `prometheus` trigger가 Prometheus에 직접 묻는다. metrics-server가 필요 없다.
- `external.metrics.k8s.io` APIService는 클러스터에 **하나만** 둘 수 있다. prometheus-adapter를 함께 설치하면 충돌한다.

---

## 2. 사전 확인

control plane에서 실행한다.

```bash
# external metrics 자리가 비어 있어야 한다. 설치 전 결과: 빈 값
kubectl get apiservice | grep -e metrics.k8s.io

# Argo CD namespace
kubectl get ns | grep -i argo
```

---

## 3. 설치 절차

### 3-1. namespace 만들기

`CreateNamespace=true`를 넣었지만 namespace가 자동으로 만들어지지 않았다(원인 미확인, §5-2). 먼저 만든다.

```bash
kubectl create namespace keda
```

### 3-2. Application 만들기

ss-k8s `platform/keda/application.yaml`을 control plane으로 복사해 적용한다.

```bash
kubectl apply -f application.yaml
kubectl get application keda -n argocd -o jsonpath='{.status.conditions}{"\n"}'   # 비어 있으면 정상
```

### 3-3. sync

1. 웹 콘솔에서 `keda` 앱의 **APP DIFF**를 확인한다. CRD, RBAC, Service, Deployment 3개, APIService가 새로 추가되는 내용만 있어야 한다.
2. **SYNC → ALL**. Server-Side Apply가 켜져 있는지 확인한다.
3. CRD가 등록되는 동안 잠깐 `Progressing`이 보인다. 이어서 ServiceAccount, Service, Deployment가 하나씩 적용된다.

---

## 4. 설치 확인

```bash
kubectl get pods -n keda -o wide                           # Pod 3개, control plane 노드, Running
kubectl get apiservice v1beta1.external.metrics.k8s.io     # AVAILABLE True
kubectl get crd | grep keda.sh                             # scaledobjects 등
```

웹 콘솔에서 앱이 `Synced` + `Healthy`면 끝이다.

---

## 5. 트러블슈팅 (설치 중 실제로 겪은 것)

### 5-1. 웹 콘솔 CREATE가 실패하고 앱이 목록에 없다

- 원인: EDIT AS YAML에 붙여 넣은 Application에 `metadata.namespace: argocd`가 없었다.
- 해결: `namespace: argocd`를 넣고 `kubectl apply`로 만들었다. 실패 이유가 터미널에 남아서 웹 콘솔보다 원인 파악이 쉽다.
- 브라우저 콘솔의 `Blocked aria-hidden on an element ...` 경고는 Argo CD 화면의 접근성 경고일 뿐 원인이 아니다.

### 5-2. `namespaces "keda" not found`로 sync 실패

- 증상: CRD만 들어가고 ServiceAccount·Service·Deployment가 모두 `Missing`. 앱은 `OutOfSync` + `Progressing`.
- 확인:
  ```bash
  kubectl get application keda -n argocd -o jsonpath='{.status.operationState.phase}{"\n"}{.status.operationState.message}{"\n"}'
  ```
- 해결: `kubectl create namespace keda` 후 다시 **SYNC → ALL**.
- `CreateNamespace=true`가 동작하지 않은 원인은 확인하지 못했다. 웹 콘솔 SYNC 창의 **AUTO-CREATE NAMESPACE** 체크가 꺼진 채로 실행됐을 가능성이 있다.

### 5-3. APIService가 `Progressing`, `ServiceNotFound: service/keda-operator-metrics-apiserver in "keda" is not present`

- 원인이 아니라 **결과**다. APIService가 가리키는 Service와 Pod가 아직 없어서 생긴다(5-2 때문).
- Service와 metrics-apiserver Pod가 뜨면 저절로 `Healthy`가 된다. 화면이 갱신되지 않으면 **REFRESH**.

---

## 6. ss-triton 연결 (ScaledObject)

ss-k8s `apps/ss-triton/03_scaledobject.yaml`.

| 항목 | 값 | 근거 |
|---|---|---|
| Pod 수 | 최소 1, 최대 2 | 테스트용 상한. stt NodePool limits도 t3.large 2대분(cpu 4, 16Gi) |
| 줄일 때 안정화 | 300초 | 새 노드 기동과 모델 로드에 몇 분이 걸린다 |
| 주 기준 | 처리 중 + 대기 중 요청 수, `threshold: "0.8"` | Pod 1개 용량 = 1.0(동시 처리 1건). 여유를 두어 0.8 |
| 보조 기준 | 503(`STT_BUSY`) 비율, `threshold: "0.05"` | 평균에 가려지는 순간 몰림을 잡는다 |
| Prometheus | `http://kube-prometheus-stack-prometheus.monitoring.svc:9090` | |

- KEDA는 `Pod 수 = 올림(쿼리 값 ÷ threshold)`으로 계산하고, trigger가 여러 개면 가장 큰 값을 따른다.
- 사용자 1명이 잠깐 썼을 때 주 기준 값은 **0.175**였다(2026-10-29). Pod 1개당 사용자 4명 남짓에서 2번째 Pod가 뜬다.
- 쿼리 결과가 비면 KEDA는 0으로 본다(`ignoreNullValues` 기본값 `true`). 503 비율 쿼리는 0÷0(NaN)을 피하려고 분모에 `> 0`을 붙였다.
- Deployment에는 `replicas`를 두지 않는다. 적으면 Argo CD sync마다 KEDA가 정한 값을 덮어쓴다.

### 언제 늘고 줄어드나

`threshold: "0.8"`은 "쿼리 값이 0.8 이상이면 늘린다"가 아니라 **Pod 1개가 감당할 목표 값**이다. 실제 동작에는 세 가지 조건이 붙는다.

1. **전체 값 ÷ 0.8**: `필요한 Pod 수 = 올림(쿼리 값 ÷ 0.8)`. 쿼리 값은 모든 Pod의 합계다.
2. **HPA 허용 오차 10%**: 쿠버네티스 HPA는 계산 결과가 현재와 10% 넘게 차이 날 때만 Pod 수를 바꾼다(기본 tolerance 0.1). 경계 근처에서 늘었다 줄었다 하는 것을 막는다.
3. **최근 2분 평균**: 쿼리가 `rate(...[2m])`라 순간 값이 아니다. KEDA는 30초마다 확인한다.

| 현재 Pod | 쿼리 값 | 계산 | 결과 |
|---|---|---|---|
| 1개 | 0.85 | 0.85 ÷ 0.8 = 1.06배, 10% 이내 | 그대로 1개 |
| 1개 | 0.9 | 0.9 ÷ 0.8 = 1.125배, 10% 초과 | **2개로 늘림** |
| 2개 | 1.2 | 올림(1.2 ÷ 0.8) = 2 | 그대로 2개 |
| 2개 | 0.7 | 올림(0.7 ÷ 0.8) = 1 | 300초 동안 유지되면 **1개로 줄임** |

- **늘어나는 시점**: Pod 1개일 때 쿼리 값이 약 **0.88**을 넘으면(0.8 × 1.1).
- **줄어드는 시점**: Pod 2개일 때 쿼리 값이 **0.8 이하**로 300초 동안 유지되면.
- 503 비율 trigger도 같은 허용 오차가 붙어 약 **5.5%**부터 늘린다.
- 실제로 0.8부터 늘리고 싶으면 threshold를 약 0.73으로 낮춘다(0.73 × 1.1 ≈ 0.8).
- 테스트 중에는 `kubectl get hpa -n sayscene -w`의 TARGETS 칸(`현재값/목표값`, 예: `880m/800m`)을 본다.

### Grafana 확인용 PromQL

Explore → 데이터소스 Prometheus → Code 모드에 붙여 넣는다. 요청이 없으면 결과가 비므로 Grafana용은 `or vector(0)`을 붙여 0으로 보이게 했다. ScaledObject 쿼리에는 붙이지 않는다(KEDA가 빈 결과를 0으로 본다).

**① 처리 중 + 대기 중 요청 수 (주 기준, threshold 0.8)**: 최근 2분 동안 평균 몇 건이 서버 안에 있었나. 1.0이면 Pod 1개가 꽉 찬 상태.

```promql
(
  sum(rate(stt_inference_seconds_sum{namespace="sayscene"}[2m]))
  +
  sum(rate(stt_queue_wait_seconds_sum{namespace="sayscene"}[2m]))
) or vector(0)
```

- 원리: 초당 쌓인 처리 시간과 대기 시간의 합은 그 시간 동안 서버 안에 있던 요청 수의 평균과 같다(리틀의 법칙).
- 벤치마크 기준으로 동시 요청 1이면 약 1.0, 2면 약 2.0이 나온다.

**② 503(`STT_BUSY`) 비율 (보조 기준, threshold 0.05)**: 요청 중 몇 %를 거절했나.

```promql
(
  sum(rate(stt_requests_total{namespace="sayscene", result="busy"}[2m]))
  /
  (sum(rate(stt_requests_total{namespace="sayscene"}[2m])) > 0)
) or vector(0)
```

- 분모의 `> 0`은 요청이 없을 때 0÷0(NaN)을 막는다. NaN은 값으로 취급돼 `or vector(0)`이 대신하지 못한다.
- Pod가 시작된 뒤 503이 한 번도 없으면 `result="busy"` 시계열이 아직 없어서 결과가 빈다. 정상이다.

**③ 대기 시간 p90 (참고)**: 처리 순서를 얼마나 기다렸나.

```promql
histogram_quantile(0.9, sum by (le) (rate(stt_queue_wait_seconds_bucket{namespace="sayscene"}[2m]))) or vector(0)
```

- 처리 순서를 받은 요청만 기록한다. 10초를 기다리다 503이 된 요청은 빠지므로, 503이 나기 시작하면 실제보다 낮게 보인다. ②와 함께 본다.

### 단일 Pod 기준값 (t3.large, `small.en`, 2026-10-29 실측)

| `WHISPER_CPU_THREADS`, `STT_CONCURRENCY` | 처리량 | 동시 1건 p50 | 동시 4건 p90 |
|---|---|---|---|
| 2, 1 | 약 0.16 rps | 5.94s | 15.42s |
| **1, 1 (채택)** | 약 0.16 rps | 6.07s | 15.40s |
| 1, 2 | 약 0.16 rps | 6.14s | 21.99s |

- t3.large의 vCPU 2개는 물리 1코어로 알고 있다. 스레드나 동시 처리 수를 늘려도 처리량이 늘지 않았다. **처리량은 노드 수에 비례해서만 늘어난다.**
- 메모리 피크 1.19GiB(working set). request·limit은 CPU 1500m, 메모리 2Gi(Guaranteed).
- c7i.large는 general AMI에서 Pod DNS가 실패해 보류했다. 주 NIC 이름 문제([이슈 01](../issue/20261027/01-llm-ami-ens5-primary-nic.md))와 같은 계열로 추정하지만 확인하지 못했다.

### 동작 확인

```bash
kubectl get scaledobject,hpa -n sayscene      # READY True, HPA keda-hpa-ss-triton의 TARGETS에 값이 보여야 한다
kubectl describe scaledobject ss-triton -n sayscene   # READY False면 Events에 Prometheus·쿼리 오류가 나온다
```

부하 테스트는 control plane에서 `bench.py`(동시 요청 수 `-c`, 시간 `-d`)로 한다. `-c 2`로 5분 주면 2번째 Pod와 t3.large가 뜨고, 멈추고 약 5분 뒤 1개로 줄어야 한다.

**결과 (2026-10-29)**: 정상 동작. 부하 시 Pod 2개·stt 노드 2대로 늘었고, 부하를 멈춘 뒤 Pod 1개로 줄고 빈 노드가 회수되는 것까지 확인했다.

---

## 7. 업그레이드·롤백

### 업그레이드

1. [호환표](https://keda.sh/docs/latest/operate/cluster/)에서 클러스터 버전이 지원 범위에 드는지 확인한다.
2. ss-k8s `platform/keda/application.yaml`의 `targetRevision`을 바꿔 커밋한다.
3. control plane에서 `kubectl apply -f application.yaml` 후 웹 콘솔에서 APP DIFF 확인, SYNC.

### 롤백 (KEDA 제거)

1. ScaledObject를 먼저 지운다. ss-k8s `apps/ss-triton/kustomization.yaml`에서 `03_scaledobject.yaml`을 빼고 sync한다. Pod 수는 마지막 값으로 남으므로 필요하면 Deployment에 `replicas: 1`을 다시 넣는다.
2. Application을 지운다.
   ```bash
   kubectl delete application keda -n argocd
   kubectl delete namespace keda
   ```
3. APIService가 남았으면 지운다: `kubectl delete apiservice v1beta1.external.metrics.k8s.io`
