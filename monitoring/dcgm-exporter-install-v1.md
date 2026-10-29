# DCGM Exporter 설치 — GPU 메트릭 수집

> 2026-10-29 설치. 설치부터 검증까지 한 번에 끝났고, 막힌 곳이 없었다.
>
> | 문서 | 답하는 질문 |
> |---|---|
> | **dcgm-exporter** (이 문서) | GPU 를 **어떻게 보게 했나** · WSL 에서 무엇이 나오나 |
> | [kube-prometheus-stack-install](./kube-prometheus-stack-install-v1.md) | 수집·저장·알림 스택 전반 |
> | [kube-prometheus-stack-concepts](./kube-prometheus-stack-concepts-v1.md) | PV·PVC·바인딩·values 를 어떻게 읽나 |
> | [sayscene-observability-manual](./sayscene-observability-manual-v1.md) | Loki · Tempo · Alloy |

---

## 목차

```
1.  왜 필요했나
2.  구성
3.  values.yaml — 세 가지만 덮는다
4.  Argo CD Application
5.  검증 절차 — 다섯 단계
6.  WSL2 에서 실제로 나온 것 · 안 나온 것
7.  ⚠️ VRAM 이 이미 89% 다 — 알림을 만들기 전에 읽을 것
8.  쓰는 법
9.  알림 규칙을 아직 안 만든 이유
10. 롤백
11. 남은 것
```

---

## 1. 왜 필요했나

`node-exporter` 는 **GPU 를 전혀 모른다.** CPU·메모리·디스크·네트워크만 본다.

```
node-exporter     /proc 을 읽는다            GPU 없음
DCGM Exporter     NVIDIA 드라이버에 묻는다    GPU 사용률 · VRAM · 온도
```

`ss-vllm` 이 GPU 를 쓰기 시작하면서 **사용률도 VRAM 여유도 볼 수 없는 상태**였다. GPU 가 포화됐는지, 모델이 메모리를 얼마나 잡았는지 알 방법이 없었다.

구조상 자리는 `node-exporter` 와 같다. DaemonSet 으로 떠서 `/metrics` 를 열고, ServiceMonitor 로 Prometheus 가 긁어간다.

```
DCGM Exporter (DaemonSet)  :9400/metrics
         ▲
    ServiceMonitor  ──  Prometheus 가 긁는다
```

---

## 2. 구성

```
차트          dcgm-exporter 4.8.4
레포          https://nvidia.github.io/dcgm-exporter/helm-charts
이미지        nvcr.io/nvidia/k8s/dcgm-exporter:4.6.1-4.8.4-distroless
네임스페이스   monitoring            Prometheus·Grafana 와 공유
배치          sayscene-gpu-wsl 1대   GPU 노드에만
파일          platform/dcgm-exporter/values.yaml
Application   dcgm-exporter          multi-source, Manual sync
```

**`kube-prometheus-stack` 과 달리 별개 차트다.** `node-exporter` 는 그 차트의 서브차트라 values 만 더하면 됐지만, DCGM 은 Application 을 새로 만들어야 한다.

```
node-exporter    kube-prometheus-stack 의 서브차트   →  Application 불필요
DCGM Exporter    독립 차트                          →  Application 신규
```

### 설치 전 클러스터 상태

이미 갖춰져 있었다. 새로 설치한 것은 없다.

```
nvidia-device-plugin   kube-system 에서 Running
RuntimeClass nvidia    존재 (handler: nvidia)
GPU 노드               sayscene-gpu-wsl · nvidia.com/gpu: 1
```

---

## 3. `values.yaml` — 세 가지만 덮는다

차트 기본값이 대부분 맞아서 **세 줄만 바꿨다.**

```yaml
# GPU 노드에만 띄운다. CPU 노드에 뜨면 드라이버가 없어 바로 죽는다.
nodeSelector:
  sayscene.com/accelerator: nvidia

# 차트 기본값은 control-plane taint 라 GPU 노드에 안 뜬다. 반드시 덮는다.
# value 는 따옴표가 필요하다. 없으면 YAML 이 불리언으로 읽어 거부당한다.
tolerations:
  - key: sayscene.com/gpu
    operator: Equal
    value: "true"
    effect: NoSchedule

# ss-vllm 과 같은 런타임. kubectl get runtimeclass 로 존재를 확인했다.
runtimeClassName: nvidia
```

### 적지 않은 것들 — 차트 기본값이 이미 맞다

```
serviceMonitor.enabled   true        그대로 쓴다
resources                선언됨       requests 100m/128Mi · limits 200m/512Mi
메트릭 목록 ConfigMap      차트가 만듦   exporter-metrics-config-map
securityContext          SYS_ADMIN   프로파일링 지표용
priorityClassName        system-node-critical
```

**기본값이 맞는 것을 다시 적지 않는다.** 적어두면 차트가 올라갈 때 옛 값에 묶인다.

### ⚠️ 기본 toleration 이 함정이다

```yaml
# 차트 기본값
tolerations:
  - key: node-role.kubernetes.io/control-plane   ← GPU 노드의 taint 가 아니다
```

GPU 노드의 taint 는 `sayscene.com/gpu=true:NoSchedule` 이다. **덮지 않으면 아예 안 뜬다.**

`alloy-logs` 가 WSL 노드에 없는 것(설치 문서 이슈 14)과 **같은 종류의 함정**이다. toleration 은 "막지 마라" 는 선언이라 조용히 어긋난다. 에러가 나지 않는다.

### ServiceMonitor 에 `release` 라벨이 없다 — 그래도 된다

차트가 만드는 ServiceMonitor 에는 `release: kube-prometheus-stack` 라벨이 없다. 보통이면 Prometheus 가 무시한다.

```yaml
# prometheus/values.yaml
serviceMonitorSelectorNilUsesHelmValues: false
```

이 설정 덕에 라벨 없이도 수집된다. ③ 담당의 ServiceMonitor 를 받기 위해 열어둔 설정이 여기서도 효과를 봤다. **설치 후 실제로 수집되는 것을 확인했다** (§5-5).

---

## 4. Argo CD Application

UI 의 `EDIT AS YAML` 로 만들었다. 폼을 하나씩 채우는 것보다 빠르고 오타가 줄어든다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: dcgm-exporter
  namespace: argocd
spec:
  project: default
  sources:
    - repoURL: https://nvidia.github.io/dcgm-exporter/helm-charts
      chart: dcgm-exporter
      targetRevision: 4.8.4
      helm:
        releaseName: dcgm-exporter
        valueFiles:
          - $values/platform/dcgm-exporter/values.yaml
    - repoURL: https://github.com/say-scene/ss-k8s.git
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

### `repoURL` 철자가 정확해야 한다

처음에 `.git` 없이 쓸 뻔했다. **등록된 주소와 한 글자라도 다르면 소스를 못 찾는다.**

```bash
kubectl -n argocd get application kube-prometheus-stack \
  -o jsonpath='{range .spec.sources[*]}{.repoURL}{"  ref="}{.ref}{"\n"}{end}'
```

```
https://prometheus-community.github.io/helm-charts   ref=
https://github.com/say-scene/ss-k8s.git              ref=values
https://github.com/say-scene/ss-k8s.git              ref=
```

기존 앱의 주소를 그대로 가져다 쓰면 된다.

### 소스가 둘인 이유

`kube-prometheus-stack` 은 셋이다.

```
1  차트
2  ss-k8s · ref=values      valueFiles 용
3  ss-k8s · ref 없음        path 아래 .yaml 을 직접 렌더 (storage.yaml)
```

DCGM 은 직접 작성한 매니페스트가 없어 **둘이면 된다.** 나중에 PersistentVolume 같은 걸 더하면 그때 세 번째를 추가한다.

### namespace 가 둘이다

```yaml
metadata:
  namespace: argocd           # Application CR 자체가 사는 곳
spec:
  destination:
    namespace: monitoring     # 만들어진 리소스가 갈 곳
```

Argo CD 는 기본적으로 **자기 네임스페이스의 Application 만 본다.** `argocd` 가 아니면 인식하지 못한다.

### UI 의 CREATE 가 안 눌릴 때

```
EDIT AS YAML  →  붙여넣기  →  SAVE  →  CREATE
                              ^^^^ 이걸 눌러야 폼에 반영된다
```

그래도 안 되면 구버전 UI 가 `sources`(복수)를 파싱하지 못하는 경우다. `kubectl apply -f -` 로 만들어도 **결과는 완전히 같다.** Application 은 CR 이라 어느 쪽으로 만들든 etcd 에 같은 것이 들어간다.

---

## 5. 검증 절차 — 다섯 단계

### 5-1. 푸시 전 렌더 검증

`helm template` 으로 **배치 조건이 제대로 덮였는지** 먼저 본다. Sync 까지 가서 실패를 알면 늦다.

```bash
cat > /tmp/dcgm-values.yaml <<'EOF'
nodeSelector:
  sayscene.com/accelerator: nvidia
tolerations:
  - key: sayscene.com/gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
runtimeClassName: nvidia
EOF

helm repo add nvidia-dcgm https://nvidia.github.io/dcgm-exporter/helm-charts
helm repo update nvidia-dcgm
helm template dcgm-exporter nvidia-dcgm/dcgm-exporter --version 4.8.4 \
  -n monitoring -f /tmp/dcgm-values.yaml > /tmp/dcgm-rendered.yaml

grep "^kind:" /tmp/dcgm-rendered.yaml | sort | uniq -c
grep -A6 "runtimeClassName:\|nodeSelector:\|tolerations:" /tmp/dcgm-rendered.yaml
```

실제 결과 — 리소스 7종:

```
ConfigMap · DaemonSet · Role · RoleBinding · Service · ServiceAccount · ServiceMonitor
```

```yaml
runtimeClassName: nvidia
nodeSelector:
  sayscene.com/accelerator: nvidia
tolerations:
  - effect: NoSchedule
    key: sayscene.com/gpu
    operator: Equal
    value: "true"          # ← 따옴표가 유지됐다
```

`value: true` (따옴표 없음) 로 나오면 **거기서 멈춘다.** taint 값은 문자열이어야 한다.

### 5-2. 파드가 GPU 노드에 떴나

```bash
kubectl -n monitoring get pod -l app.kubernetes.io/name=dcgm-exporter -o wide
```

```
NAME                  READY   STATUS    RESTARTS   AGE   NODE
dcgm-exporter-rtbcf   1/1     Running   0          92s   sayscene-gpu-wsl
```

DaemonSet 이지만 **노드 하나에만 뜬다.** `nodeSelector` 가 GPU 노드만 고르기 때문이다. CPU 노드 2대에는 뜨지 않는 것이 정상이다.

```
node-exporter    tolerations 만 넓게   →  모든 노드 (3대)
dcgm-exporter    nodeSelector 로 좁힘  →  GPU 노드만 (1대)
```

`ContainerCreating` 이 몇 분 이어질 수 있다. 이미지가 수백 MB 다. Running 이 돼도 `livenessProbe.initialDelaySeconds: 45` 라 Ready 까지 45초를 더 기다린다.

### 5-3. 안 떴을 때

```bash
kubectl -n monitoring describe pod -l app.kubernetes.io/name=dcgm-exporter | tail -25
```

```
Pending              배치 조건 문제. nodeSelector·toleration 을 본다
FailedMount          pod-gpu-resources (/var/lib/kubelet/pod-resources)
                     WSL 의 kubelet 경로가 표준이 아닐 때
CrashLoopBackOff     떴는데 죽는다. DCGM 이 드라이버에 못 붙은 것
```

**이번 설치에서는 셋 다 발생하지 않았다.** WSL 노드의 kubelet 경로는 표준(`/var/lib/kubelet`)이었다.

### 5-4. 실제로 어떤 지표가 나오나 ← 핵심

이미지가 distroless 라 **컨테이너 안에 `curl` 이 없다.** port-forward 로 간다.

```bash
kubectl -n monitoring port-forward ds/dcgm-exporter 9400:9400 >/dev/null 2>&1 &
sleep 3
curl -s localhost:9400/metrics | grep "^DCGM" | cut -d'{' -f1 | sort -u
curl -s localhost:9400/metrics | grep -E "^DCGM_FI_DEV_(GPU_UTIL|FB_USED|FB_FREE|GPU_TEMP)"
kill %1
```

### 5-5. Prometheus 가 주워갔나

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090 >/dev/null 2>&1 &
sleep 3
curl -s 'http://localhost:9090/api/v1/query?query=DCGM_FI_DEV_GPU_UTIL'
kill %1
```

```json
{"status":"success","data":{"resultType":"vector","result":[{"metric":{
  "__name__":"DCGM_FI_DEV_GPU_UTIL","job":"dcgm-exporter",
  "namespace":"monitoring","instance":"...:9400",
  "hostname":"sayscene-gpu-wsl","modelName":"NVIDIA GeForce RTX 4060", ...
```

여기까지 나오면 **깃 → Argo CD → DaemonSet → 수집 → 저장**이 전부 연결된 것이다.

---

## 6. WSL2 에서 실제로 나온 것 · 안 나온 것

> NVIDIA 문서는 DCGM Exporter 의 지원 플랫폼을 **Linux 로 명시**한다. WSL2 는
> 런타임 지원 선언이 없다. 설치 전에는 전력·온도가 안 나올 것으로 예상했으나,
> **온도는 나왔다.** 실측 결과를 그대로 남긴다.

### 나온 것 — 16개

```
DCGM_FI_DEV_GPU_UTIL                    GPU 사용률 %
DCGM_FI_DEV_FB_USED / FB_FREE           VRAM 사용 / 여유 (MiB)
DCGM_FI_DEV_GPU_TEMP                    GPU 온도 °C        ← 예상과 달리 나왔다
DCGM_FI_DEV_MEMORY_TEMP                 메모리 온도
DCGM_FI_DEV_SM_CLOCK / MEM_CLOCK        클럭
DCGM_FI_DEV_MEM_COPY_UTIL               메모리 복사 사용률
DCGM_FI_DEV_ENC_UTIL / DEC_UTIL         인코더 / 디코더
DCGM_FI_DEV_PCIE_REPLAY_COUNTER         PCIe 재시도
DCGM_FI_DEV_CORRECTABLE_REMAPPED_ROWS
DCGM_FI_DEV_UNCORRECTABLE_REMAPPED_ROWS
DCGM_FI_DEV_ROW_REMAP_FAILURE
DCGM_FI_DEV_VGPU_LICENSE_STATUS
```

값 예시 (유휴 상태):

```
DCGM_FI_DEV_FB_USED{modelName="NVIDIA GeForce RTX 4060", hostname="sayscene-gpu-wsl"} 7057
DCGM_FI_DEV_FB_FREE{...}  898
DCGM_FI_DEV_GPU_TEMP{...}  43
DCGM_FI_DEV_GPU_UTIL{...}   0
```

라벨에 `modelName` · `hostname` · `UUID` · `pci_bus_id` 가 붙는다. GPU 가 여러 장이면 `gpu="0"` 으로 구분된다.

### 안 나온 것

```
DCGM_FI_DEV_POWER_USAGE                 전력        WSL 제약으로 보인다
DCGM_FI_DEV_TOTAL_ENERGY_CONSUMPTION    누적 에너지  같은 이유
DCGM_FI_DEV_PCIE_TX/RX_THROUGHPUT       PCIe 대역폭
ECC 카운터 (SBE · DBE)                   RTX 4060 은 ECC 미지원. 당연하다
NVLink 계열                              단일 소비자 GPU. 해당 없음
DCGM_FI_PROF_*                           차트 기본 목록에서 주석 처리되어 있다
```

**전력만 아쉽고 나머지는 하드웨어 특성상 없는 것들이다.** 16개면 운영에 충분하다.

전력이 꼭 필요해지면 `customMetrics` 로 필드를 추가해 볼 수 있지만, WSL 드라이버가 값을 채우지 못하면 필드만 생기고 값은 비게 된다.

---

## 7. ⚠️ VRAM 이 이미 89% 다 — 알림을 만들기 전에 읽을 것

설치 직후 유휴 상태에서 이렇게 나왔다.

```
FB_USED  7057 MiB
FB_FREE   898 MiB
합계     7955 MiB  ≈ 8GB

사용률 = 7057 / 7955 = 88.7%
GPU_UTIL = 0
```

**놀고 있는데 VRAM 은 꽉 차 있다.** vLLM 이 시작할 때 `gpu_memory_utilization` 만큼(기본 0.9) 통째로 선점하기 때문이다. KV 캐시용이다.

### 그래서 이런 규칙을 만들면 안 된다

```promql
# ❌ vLLM 이 떠 있는 내내 상시 발화한다
DCGM_FI_DEV_FB_USED / (DCGM_FI_DEV_FB_USED + DCGM_FI_DEV_FB_FREE) * 100 > 90
```

디스크 알림 문구가 노드를 단정했던 것(설치 문서 §13-11 이슈 12)과 **같은 종류의 실수**다. 지표의 의미를 확인하지 않고 임계값부터 거는 것이다.

### 봐야 할 것

```
GPU_UTIL          실제로 계산하고 있나
vLLM 자체 지표     대기 큐 길이 · 처리량 · KV 캐시 사용률
```

vLLM 은 자기 메트릭을 따로 내보낸다. VRAM 포화 여부는 **DCGM 이 아니라 vLLM 쪽에서** 보는 것이 맞다.

---

## 8. 쓰는 법

```promql
# GPU 사용률
DCGM_FI_DEV_GPU_UTIL

# 온도
DCGM_FI_DEV_GPU_TEMP

# VRAM 사용률 % — §7 의 주의사항을 읽고 쓸 것
DCGM_FI_DEV_FB_USED / (DCGM_FI_DEV_FB_USED + DCGM_FI_DEV_FB_FREE) * 100

# GPU 가 보이고 있나
up{job="dcgm-exporter"}

# 5분 평균 사용률
avg_over_time(DCGM_FI_DEV_GPU_UTIL[5m])
```

### Grafana 대시보드

NVIDIA 공식 대시보드가 있다. Grafana → Dashboards → Import → **ID `12239`** 를 넣으면 된다. 데이터소스는 Prometheus 를 고른다.

10-29 에 가져왔다. 태그는 `gpu` · `dcgm` 이다.

> **전제 — Grafana 의 `persistence` 가 켜져 있어야 한다.**
> 꺼져 있으면 `grafana.db` 가 `emptyDir` 위에 있어, Import 해도 파드가 다시
> 만들어질 때 사라진다. 10-29 에 실제로 한 번 잃었다. 같은 날 `grafana-local`
> PV 를 붙여 해소했다 (`kube-prometheus-stack-install-v1.md`).

---

## 9. 알림 규칙을 아직 안 만든 이유

**기준이 없다.** 며칠 데이터를 쌓고 정하기로 했다.

```
VRAM        vLLM 이 89% 를 선점한다. 이 상태가 정상인지 위험인지 아직 모른다
GPU_UTIL    평소 부하가 어느 정도인지 모른다
온도        43°C 가 유휴값이다. 부하 시 어디까지 오르는지 모른다
```

임계값을 추측으로 정하면 상시 발화하거나 아예 안 울린다. **먼저 보고 나중에 건다.**

알림을 만들게 되면 조건은 `platform/kube-prometheus-stack/prometheus/values.yaml` 의 `additionalPrometheusRulesMap` 에 넣는다. 수신처는 `alertmanager/values.yaml` 이다.

---

## 10. 롤백

```
Argo CD Application 삭제   →  DaemonSet · Service · ServiceMonitor 전부 사라짐
values.yaml 삭제           →  새 파일이라 지우면 끝
```

**기존 구성에 영향이 없다.** Prometheus·Grafana 설정을 건드리지 않았고, GPU 노드에 파드 하나(100m CPU · 128Mi)가 추가됐을 뿐이다.

Prune 이 꺼져 있으면 Application 을 지워도 리소스가 남을 수 있다. 그때는 직접 지운다.

```bash
kubectl -n monitoring delete daemonset,service,servicemonitor,configmap,serviceaccount,role,rolebinding \
  -l app.kubernetes.io/name=dcgm-exporter
```

---

## 11. 남은 것

```
✅  Grafana 대시보드 Import (ID 12239) — 10-29. 태그 gpu · dcgm
⬜  알림 규칙 — 며칠 데이터를 쌓은 뒤 기준을 정한다 (§9)
⬜  전력 지표 — WSL 에서 가능한지 추가 조사. 우선순위 낮음
⬜  vLLM 자체 메트릭 수집 — VRAM·큐 길이는 그쪽이 정확하다 (§7)
```

### 배운 것

```
1.  차트 기본 toleration 을 확인한다. 덮지 않으면 조용히 안 뜬다
2.  taint value 는 따옴표를 쓴다. 불리언이 되면 거부당한다
3.  푸시 전에 helm template 으로 배치 조건을 본다
4.  repoURL 은 등록된 철자 그대로 쓴다 (.git 포함)
5.  지표가 나온다고 의미를 아는 것은 아니다. VRAM 89% 가 그 예다
```
