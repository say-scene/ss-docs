# kube-prometheus-stack 설치 문서

> 담당 ① · 작성 2026-10-27 · 갱신 2026-10-28 (네임스페이스 결정 + 배포 후 리뷰 + ss-triton 정정) · 상태: **설치 완료 (2026-10-27 20:25, `Healthy` + `Synced`)**
> 클러스터 전반은 [al2023-cpu-cluster-karpenter-guide.md](../al2023-cpu-cluster-karpenter-guide.md)
> 배경과 팀 분담 검토(`관측성-검토.md`)·클러스터 구성 정리(`클러스터구성.md`)는 ① 담당 로컬 문서로, 이 저장소에 없다.

## 1. 범위

이 문서는 `kube-prometheus-stack` 차트 하나의 설치만 다룬다. 차트에 포함되는 것은 다섯이다.

```
Prometheus · Alertmanager · Grafana · node-exporter · kube-state-metrics
Prometheus Operator 와 CRD 도 함께 들어온다
```

**범위 밖:** DCGM(②), Alloy·Loki·Tempo(③), KEDA, `ss-be` 애플리케이션 계측.

## 2. 전제 — 클러스터 실측 (2026-10-27)

| 항목 | 값 | 설치에 미치는 영향 |
|---|---|---|
| 기본 노드 | `ip-10-0-4-232` · 4 vCPU · 16 GiB | 전부 여기에 올린다 |
| CPU 여유 | 약 2,120m | 스택 전체가 들어간다 |
| 메모리 여유 | 약 14.9 GiB | 〃 |
| CPU limits | **0** — 상한 건 파드가 없다 | **메모리 limits 를 명시한다** (CPU 는 §5-2) |
| **루트 디스크** | **16G 중 7.3G 여유 (55% 사용)** | **여기가 가장 빡빡하다** — §5-0 |
| StorageClass | **없음** | PVC 불가 |
| metrics-server | 없음 | 영향 없음. 스케줄링은 requests 기준 |
| 기본 노드 taint | `node-role.kubernetes.io/control-plane:NoSchedule` | **toleration 필수** |
| 앱 네임스페이스 | `sayscene` | ServiceMonitor 의 `namespaceSelector` |
| `ss-triton` Service | 설치 당시 없었으나 **10-27 21:29 에 추가됨** | §13-8 참고 — `replicas: 0` 이라 endpoint 는 여전히 0 개 |

## 3. 폴더 구조

```
ss-k8s/platform/kube-prometheus-stack/
├─ README.md
├─ storage.yaml                          StorageClass + local PV (유일한 최상위 .yaml)
├─ common/values.yaml                    전역 값만
├─ prometheus-operator/values.yaml       CRD · ServiceMonitor 반영 담당
├─ prometheus/values.yaml                보존 · hostPath · ServiceMonitor · 알림 규칙
├─ grafana/values.yaml                   sidecar datasources
├─ alertmanager/values.yaml              알림 규칙
├─ prometheus-node-exporter/values.yaml  DaemonSet toleration
└─ kube-state-metrics/values.yaml
```

**폴더명은 `values.yaml` 안의 최상위 키와 같게 둔다.** 그래서 `node-exporter` 가 아니라 `prometheus-node-exporter` 다. 폴더만 보면 안에 어떤 키가 있는지 알 수 있다.

**values 는 하위 폴더에만 둔다.** Argo CD 는 `ref` 소스의 `path` 를 매니페스트로도 렌더링하는데(Karpenter 에서 확인) 디렉터리 스캔이 비재귀라, 한 층 내려두면 values 가 매니페스트로 읽히지 않는다.

`storage.yaml` 은 반대로 **의도적으로 적용되는 매니페스트**라 최상위에 둔다. 설치 후 확인한 결과 Argo CD 가 `StorageClass` 와 `PersistentVolume` 을 관리 대상으로 잡았다.

## 4. Argo CD Application

Karpenter 와 같은 multi-source 형태다.

```yaml
sources:
- chart: kube-prometheus-stack
  repoURL: https://prometheus-community.github.io/helm-charts
  targetRevision: <버전 고정>
  helm:
    valueFiles:
      - $values/platform/kube-prometheus-stack/common/values.yaml
      - $values/platform/kube-prometheus-stack/prometheus-operator/values.yaml
      - $values/platform/kube-prometheus-stack/prometheus/values.yaml
      - $values/platform/kube-prometheus-stack/grafana/values.yaml
      - $values/platform/kube-prometheus-stack/alertmanager/values.yaml
      - $values/platform/kube-prometheus-stack/prometheus-node-exporter/values.yaml
      - $values/platform/kube-prometheus-stack/kube-state-metrics/values.yaml
- path: platform/kube-prometheus-stack
  ref: values
```

| 웹 콘솔 입력 | 값 | 이유 |
|---|---|---|
| 차트 버전 (`targetRevision`) | **`92.13.1`** | 최신(92.14.0, 10-26) 바로 아래. 설치 직전 릴리스 노트 확인 |
| Destination namespace | `monitoring` | **비우면 `default` 로 간다** — Karpenter 전례 |
| Auto-create namespace | 켬 | |
| Sync policy | 수동 | 현재 운영 방식 |
| **Server-Side Apply** | **켬** | CRD 가 수 MB — annotation 256KB 한도 회피 |
| Prune · Force | 끔 | |

**`valueFiles` 는 순서대로 병합되고 뒤가 이긴다.** `common` 을 맨 앞에 둔다.

## 5. values 파일 초안

### 5-0. 저장소 — 기본 노드 루트 디스크의 local PV

**결정:** 전용 볼륨을 붙이지 않고 기본 노드의 `/var/lib/prometheus` 를 쓴다. 필요해지면 옮긴다.

**근거는 전환 비용이 낮다는 점이다.** 보존이 7일이면 옮길 때 버릴 데이터가 7일치뿐이다. PoC 에서 지표 유실은 치명적이지 않고, 기본 노드는 교체 계획이 없다.

**방식은 local PersistentVolume 이다.** `hostPath` 를 `spec.volumes` 로 직접 붙이려다 두 번 실패했다 (§12). Prometheus Operator 는 저장소를 `storageSpec` 으로만 받는다.

```
storage.yaml          StorageClass(no-provisioner, WaitForFirstConsumer) + PV
prometheus/values.yaml  storageSpec.volumeClaimTemplate 이 그 StorageClass 를 쓴다
```

PV 는 `nodeAffinity` 로 기본 노드에 묶는다. 그 경로가 거기에만 있기 때문이다. `reclaimPolicy: Retain` 이라 Application 을 지워도 지표가 남는다.

**설치 전에 기본 노드에서 한 번 실행해야 한다.**

```bash
sudo mkdir -p /var/lib/prometheus
```

local PV 는 `hostPath` 의 `DirectoryOrCreate` 와 달리 **디렉터리를 만들어주지 않는다.** 없으면 `FailedMount: path does not exist` 로 멈춘다. 소유권은 `fsGroup` 이 맞추므로 `chown` 은 필요 없다 — 설치 후 `drwxrwsr-x root 2000` 으로 확인했다.

**대가는 디스크 위험이다.** 디스크가 차면 Prometheus 만 죽는 것이 아니라 **같은 노드의 Argo CD · Karpenter · CoreDNS 가 전부 위험해진다.** 그래서 조건 둘을 붙인다.

**조건 ① 보존 상한을 디스크 여유에 맞춘다.** `retention: 7d` + **`retentionSize: 1GiB`**. `retentionSize` 는 TSDB 블록 기준이라 **정확한 상한이 아니고** WAL 이 일시적으로 더 쓴다.

**조건 ② 디스크 알림을 함께 건다.** node-exporter 가 잔량을 수집하고 Prometheus 가 85% 규칙을 평가한다. **설치하는 스택이 자기 위험을 감시한다.**

### 디스크 현황 — 전환 기준에 근접

```
설치 전   16G 중 7.3G 여유 (55%)
설치 후   16G 중 4.6G 여유 (72%)    ← 이미지가 2.7G
```

`retentionSize: 2GiB` 로 두면 여유가 **2.6G** 까지 내려가 아래 전환 기준의 첫 줄에 닿고, 85% 알림이 상시 발화한다.

**결정: `1GiB` 로 낮췄다** (`7bfe529`). 여유가 3.6G 선에서 유지되어 알림이 의미를 잃지 않는다. 대가는 보존 기간이다 — **7d 를 다 못 채우고 용량에서 먼저 잘린다.** 전용 볼륨으로 옮길 때 다시 올린다.

### 전환 기준

셋 중 하나가 오면 전용 볼륨으로 옮긴다. **기준이 없으면 "나중에" 가 오지 않는다.**

```
루트 디스크 여유가 3G 미만으로 내려간다      ← 근접
보존을 7d 이상으로 늘리고 싶어진다
기본 노드 교체 계획이 생긴다
```

옮길 때는 EBS 볼륨을 만들어 기본 노드에 붙이고 `/var/lib/prometheus` 로 마운트한 뒤, PV 의 `capacity` 만 키우면 된다. **`storageSpec` 구조는 그대로 쓴다.**

### 5-0-1. prometheus-operator/values.yaml

```yaml
prometheusOperator:
  nodeSelector:
    node-role.kubernetes.io/control-plane: ""
  tolerations:
    - key: node-role.kubernetes.io/control-plane
      operator: Exists
      effect: NoSchedule
  resources:
    requests: { cpu: 50m, memory: 128Mi }
    limits:   { memory: 512Mi }
```

Operator 는 차트가 함께 설치하는 **여섯 번째 구성요소**다. CRD 를 관리하고 ServiceMonitor 를 Prometheus 설정에 반영한다. 고정하지 않으면 확장 워커로 갈 수 있고, **그 워커가 정리될 때 ServiceMonitor 반영이 멈춘다.**


### 5-1. common/values.yaml

```yaml
global:
  imageRegistry: ""
```

**여기에 고정 설정을 넣지 않는다.** 최상위 `nodeSelector` 는 subchart 에 전파되지 않는다. 각 subchart 는 자기 키 아래만 읽으므로, 고정 블록은 아래 파일마다 반복해서 쓴다. 중복이지만 차트 구조상 피할 수 없다.

### 5-2. prometheus/values.yaml

```yaml
prometheus:
  prometheusSpec:
    retention: 7d
    retentionSize: 1GiB          # 전용 볼륨으로 옮기면 올린다
    serviceMonitorSelectorNilUsesHelmValues: false
    serviceMonitorNamespaceSelector: {}    # 모든 네임스페이스
    nodeSelector:
      node-role.kubernetes.io/control-plane: ""
    tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
    resources:
      requests: { cpu: 200m, memory: 1Gi }
      limits:   { memory: 3Gi }
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: local-prometheus
          accessModes: [ReadWriteOnce]
          resources:
            requests:
              storage: 5Gi
    additionalServiceMonitors:
      - name: ss-be
        namespaceSelector:
          matchNames: [sayscene]
        selector:
          matchLabels: { app: ss-be }
        endpoints:
          - port: http
            path: /actuator/prometheus
      # [10-28 정정] 아래 주석은 설치 당시(10-27 20:25)의 상태다.
      # 10-27 21:29 에 다른 담당자가 Service 와 함께 주석을 풀었다. §13-8 참고.
      # ss-triton 은 Service 가 아직 없다. 생기면 추가한다.
      # - name: ss-triton
      #   namespaceSelector: { matchNames: [sayscene] }
      #   selector: { matchLabels: { app: ss-triton } }
      #   endpoints: [{ port: metrics }]
```

루트 디스크를 쓰므로 **디스크 알림을 같은 파일에 넣는다.**

```yaml
additionalPrometheusRulesMap:
  disk:
    groups:
      - name: node-disk
        rules:
          - alert: NodeDiskFillingUp
            expr: >-
              100 - (node_filesystem_avail_bytes{mountpoint="/"}
              / node_filesystem_size_bytes{mountpoint="/"} * 100) > 85
            for: 10m
            labels: { severity: warning }
            annotations:
              summary: "기본 노드 루트 디스크 85% 초과"
```

**`retentionSize` 를 반드시 `retention` 과 함께 건다.** 둘 중 먼저 닿는 쪽에서 삭제된다. 기간만 걸면 디스크가 먼저 찬다.

**`serviceMonitorNamespaceSelector: {}`** 로 모든 네임스페이스를 본다. ServiceMonitor 는 `monitoring` 에 생기고 대상 Service 는 `sayscene` 에 있다.

**`serviceMonitorSelectorNilUsesHelmValues: false` 가 중요하다.** 기본값(`true`)이면 Prometheus 는 **이 차트의 release 라벨이 붙은 ServiceMonitor 만** 수집한다. ③ 담당이 자기 폴더에서 ServiceMonitor 를 만들면 라벨이 없어 **조용히 무시된다.** Grafana sidecar 와 같은 종류의 경계 문제이고, ① 설치 시점에 정해야 한다.

**저장소와 노드 고정은 세트로 움직인다.** PV 의 `nodeAffinity` 가 기본 노드를 가리키므로, 고정이 빠지면 파드가 다른 노드로 가서 `WaitForFirstConsumer` 바인딩이 실패한다.

**`spec.volumes` 로 저장소를 바꾸려 하지 않는다.** Operator 가 `/prometheus` 에 `prometheus-<CR이름>-db` 볼륨을 항상 추가하므로, 같은 이름은 중복이고 다른 이름은 `mountPath` 중복이다. 둘 다 StatefulSet 생성이 거부된다 (§12).

**`cpu limits` 는 의도적으로 비웠다.** CPU 상한은 쓰로틀링을 일으켜 쿼리를 느리게 만든다. 메모리만 막아 이웃 파드의 OOM 을 방지한다. §2 의 "메모리 limits 를 명시한다" 가 이 뜻이다.

### 5-3. grafana/values.yaml

```yaml
grafana:
  nodeSelector:
    node-role.kubernetes.io/control-plane: ""
  tolerations:
    - key: node-role.kubernetes.io/control-plane
      operator: Exists
      effect: NoSchedule
  resources:
    requests: { cpu: 50m, memory: 128Mi }
    limits:   { memory: 512Mi }
  sidecar:
    datasources:
      enabled: true
      label: grafana_datasource
```

`grafana.ini` 의 `root_url` · `serve_from_sub_path` 는 **외부 노출(2 단계)에서 더한다.** 1 단계에서는 `port-forward` 로 보므로 필요 없다.

**`sidecar.datasources` 는 반드시 켠다.** 이것이 켜져 있으면 ③ 담당이 Loki·Tempo 데이터소스를 **자기 폴더의 ConfigMap 으로** 붙일 수 있다. 꺼져 있으면 ③ 이 이 파일을 고쳐야 해서 담당 경계가 겹친다.

### 5-4. alertmanager/values.yaml

```yaml
alertmanager:
  alertmanagerSpec:
    nodeSelector:
      node-role.kubernetes.io/control-plane: ""
    tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
    resources:
      requests: { cpu: 10m, memory: 64Mi }
      limits:   { memory: 256Mi }
```

**설치는 하되 수신처(Slack·이메일)는 비운다.** 경보는 Grafana 와 Alertmanager 화면에서 볼 수 있다. 수신처 연결은 별도 작업이고 지금 범위를 늘릴 이유가 없다.

### 5-5. prometheus-node-exporter/values.yaml

```yaml
prometheus-node-exporter:
  tolerations:
    - operator: Exists
  resources:
    requests: { cpu: 10m, memory: 32Mi }
    limits:   { memory: 128Mi }
```

**노드 고정을 하지 않는다.** DaemonSet 이라 모든 노드에 떠야 한다. 대신 기본 노드의 taint 를 견디도록 `operator: Exists` 로 전부 허용한다. 확장 워커가 뜨면 자동으로 따라붙고, 정리되면 함께 사라진다.

### 5-6. kube-state-metrics/values.yaml

```yaml
kube-state-metrics:
  nodeSelector:
    node-role.kubernetes.io/control-plane: ""
  tolerations:
    - key: node-role.kubernetes.io/control-plane
      operator: Exists
      effect: NoSchedule
  resources:
    requests: { cpu: 10m, memory: 64Mi }
    limits:   { memory: 256Mi }
```

## 6. 사전 점검 — 2026-10-27 통과

Sync 전에 기본 노드에서 **Argo CD 가 할 일을 그대로 먼저 해봤다.** `helm` 이 기본 노드에 있으므로 가능하다.

```bash
rm -rf /tmp/kps && git clone -q -b feature/kube-prometheus-stack --depth 1 \
  https://github.com/say-scene/ss-k8s.git /tmp/kps
cd /tmp/kps/platform/kube-prometheus-stack

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update prometheus-community

helm template kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --version 92.13.1 --namespace monitoring --include-crds \
  -f common/values.yaml \
  -f prometheus-operator/values.yaml \
  -f prometheus/values.yaml \
  -f grafana/values.yaml \
  -f alertmanager/values.yaml \
  -f prometheus-node-exporter/values.yaml \
  -f kube-state-metrics/values.yaml \
  > /tmp/rendered.yaml
```

### 결과

| 확인 | 결과 |
|---|---|
| 쿠버네티스 | 서버 `v1.35.9` · 차트 제약 `kubeVersion >=1.25.0-0` → **충족** |
| 차트 | `92.13.1` 존재 · appVersion `v0.94.1` |
| 렌더링 | 성공 · 83,275 줄 (대부분 CRD) |
| 디스크 | 16G 중 **7.3G 여유** |
| Docker Hub | `401` — 도달 가능 (busybox 수급 가능) |
| 서버 dry-run | CRD · ClusterRole · Service 적용 가능, 에러 없음 |

### 내 설정이 먹었는지

```bash
for k in node-role.kubernetes.io/control-plane hostPath /var/lib/prometheus \
         chown-data NodeDiskFillingUp ss-be; do
  printf '%-40s %s\n' "$k" "$(grep -c -- "$k" /tmp/rendered.yaml)"
done
grep -cE 'kube-controller-manager|kube-scheduler|kube-proxy|kube-etcd' /tmp/rendered.yaml
```

```
node-role.kubernetes.io/control-plane   12      노드 고정 5 쌍
hostPath                                12      Prometheus + node-exporter
/var/lib/prometheus                      1
chown-data                               1      initContainer
NodeDiskFillingUp                        1      디스크 알림 규칙
ss-be                                    2      ServiceMonitor
꺼둔 컴포넌트 4 종                        0      의도대로 빠짐
```

**이 점검이 핵심이다.** `valueFiles` **경로**가 틀리면 Helm 이 에러를 내므로 금방 드러난다. 조용한 실패는 파일 안의 **키 이름**이 틀렸을 때다 — 차트가 모르는 키라 무시하고 기본값으로 설치한다. 파드는 정상이고 `Healthy` 로 보이는데 노드 고정도 `hostPath` 도 안 들어간 상태가 된다. 위 `grep` 이 그 경우를 잡는다.

### 확정된 리소스 이름

렌더링에서 직접 확인했다. 추측하지 않는다.

```
Deployment   kube-prometheus-stack-operator
Deployment   kube-prometheus-stack-grafana
Deployment   kube-prometheus-stack-kube-state-metrics
DaemonSet    kube-prometheus-stack-prometheus-node-exporter
```

**Prometheus 와 Alertmanager 의 StatefulSet 은 렌더링 결과에 없다.** 차트가 만드는 것은 `Prometheus` · `Alertmanager` CR 이고, StatefulSet 은 **Operator 가 그 CR 을 보고 뒤에 만든다.** 이름은 CR 이름 앞에 종류가 붙는다.

```
prometheus-kube-prometheus-stack-prometheus
alertmanager-kube-prometheus-stack-alertmanager
```

Sync 직후 StatefulSet 이 없다가 잠시 뒤 생기는 것이 정상이다.

## 7. 설치 순서

**1 단계(설치·검증)와 2 단계(외부 노출)를 나눈다.** 함께 하면 실패했을 때 차트 문제인지 엣지 문제인지 구분되지 않는다.

### 1 단계 — 설치와 검증

```
0  기본 노드 셸    df -h /          여유가 3G 아래면 멈추고 전용 볼륨부터
1  로컬           커밋 → 푸시      (완료: main c65c71a)
2  Argo CD 웹      Application 생성
3  Argo CD 웹      Sync
4  확인            §8
```

**디렉터리 준비는 필요 없다.** kubelet 이 만들고 initContainer 가 소유권을 맞춘다.

#### 2-1. 브랜치 — `main` 으로 확정

`feature/kube-prometheus-stack` 을 **`main` 에 병합했다** (`c65c71a`, 2026-10-27). Application 은 `main` 을 본다.

검증용 브랜치를 유지하지 않은 이유는 **§6 사전 점검이 실제 클러스터에서 이미 끝났기** 때문이다. 렌더링과 서버 dry-run 으로 확인할 수 있는 것은 다 확인했다.

#### 2-2. Application 정의

`+ NEW APP` → `EDIT AS YAML` 에 붙여넣는 편이 빠르다.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack
  namespace: argocd
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
  sources:
    - chart: kube-prometheus-stack
      repoURL: https://prometheus-community.github.io/helm-charts
      targetRevision: 92.13.1
      helm:
        valueFiles:
          - $values/platform/kube-prometheus-stack/common/values.yaml
          - $values/platform/kube-prometheus-stack/prometheus-operator/values.yaml
          - $values/platform/kube-prometheus-stack/prometheus/values.yaml
          - $values/platform/kube-prometheus-stack/grafana/values.yaml
          - $values/platform/kube-prometheus-stack/alertmanager/values.yaml
          - $values/platform/kube-prometheus-stack/prometheus-node-exporter/values.yaml
          - $values/platform/kube-prometheus-stack/kube-state-metrics/values.yaml
    - repoURL: https://github.com/say-scene/ss-k8s.git
      targetRevision: main
      path: platform/kube-prometheus-stack
      ref: values
```

**`ServerSideApply=true` 가 빠지면 CRD 가 커서 실패한다.** `namespace` 를 비우면 Karpenter 처럼 `default` 로 간다.

#### 2-3. Sync

`SYNCHRONIZE` 를 누른다. 수동 sync 라 누르기 전까지 아무 일도 일어나지 않는다. **첫 Sync 는 몇 분 걸린다** — CRD 5 개를 먼저 올리고 나머지를 적용한다.

### 2 단계 — 외부 노출 (보류)

Argo CD 와 같은 ALB 를 공유한다. **`group.order` 가 겹치면 안 된다.**

```
10  argocd    init/Ingress.yaml
20  ss-be     apps/ss-be/06_ingress.yaml
30  grafana   ← 새로 추가
```

CloudFront 에는 behavior 를 **두 개** 만든다 — `/grafana` 와 `/grafana/*`. 후자는 전자를 포함하지 않는다. Argo CD 에서 겪은 것과 같다.

노출할 때 `grafana/values.yaml` 에 아래를 더한다. 1 단계에서는 넣지 않는다.

```yaml
grafana:
  grafana.ini:
    server:
      root_url: "https://<CloudFront 도메인>/grafana"
      serve_from_sub_path: true
```

`serve_from_sub_path` 가 빠지면 ALB health check 가 404 를 받아 target 이 전부 unhealthy 가 된다. Argo CD 의 `rootpath` 와 같은 문제다.

**Ingress 파일을 어디에 두고 검증 헤더를 어떻게 다룰지가 미결이다.** `ss-k8s` 는 **공개 저장소**라 `X-Origin-Verify` 값을 커밋할 수 없다. 현재 `init/Ingress.yaml` 에도 헤더 조건이 없고 prefix list 만 있다.

## 8. 확인 순서

### ① Application 카드 — 큰 상태

```
Synced    깃과 클러스터가 같다
Healthy   파드가 실제로 떴다
```

**`Synced` + `Progressing` 이 한동안 정상이다.** 파드가 기동 중이라는 뜻이다. `Degraded` 로 멈추면 ② 로 간다.

### ② 리소스 트리 — 무엇이 안 되는지

Application 을 클릭하면 리소스가 트리로 보인다. 빨간 것을 눌러 `EVENTS` 탭을 본다.

| 증상 | 원인 |
|---|---|
| `Pending` | 노드 고정 또는 toleration 누락 |
| `CrashLoopBackOff` | `hostPath` 권한 — initContainer 로그를 본다 |
| `ComparisonError` | `valueFiles` 경로 오타 |
| CRD 적용 실패 | `ServerSideApply` 가 꺼져 있다 |

### ③ APP DIFF — 내 values 가 먹었는지

`APP DIFF` 에서 Prometheus 리소스를 보면 **최종 렌더링 결과**가 나온다. `nodeSelector` 와 `hostPath` 가 보이면 values 가 적용된 것이다.

**경로가 틀리면 `ComparisonError` 로 드러난다.** 조용한 실패는 경로가 아니라 **키 이름**이 틀렸을 때이고, §6 사전 점검이 그 경우를 잡는다. 여기서는 Sync 전후가 같은지만 본다.

### ④ 노드 셸 — 실제 확인

```bash
kubectl -n monitoring get pods -o wide
kubectl -n monitoring exec sts/prometheus-kube-prometheus-stack-prometheus \
  -c prometheus -- df -h /prometheus
```

StatefulSet 은 Operator 가 CR 을 보고 만든다. **Sync 직후에는 없을 수 있으니** 잠시 기다린다.

두 번째가 **가장 불확실한 지점**이다. `/dev/nvme0n1p1` 이면 성공이고, `tmpfs` 나 `overlay` 면 Operator 가 붙인 emptyDir 이 이긴 것이다.

initContainer 가 실패했다면 여기서 드러난다.

```bash
kubectl -n monitoring logs sts/prometheus-kube-prometheus-stack-prometheus -c chown-data
```

### ⑤ Grafana

```bash
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80

kubectl -n monitoring get secret kube-prometheus-stack-grafana \
  -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

Targets 화면에서 `node-exporter` 와 `kube-state-metrics` 가 UP 이면 성공이다.

### 통과 기준

```
파드가 전부 Running 이고 NODE 가 기본 노드다
  kubectl -n monitoring get pods -o wide

CRD 가 Established 다
  kubectl get crd | grep -E 'monitoring.coreos.com'

Prometheus 의 Targets 에서 node-exporter·kube-state-metrics 가 UP 이다

Argo CD Application 이 Synced + Healthy 다

파드를 지웠다 살려도 지표가 남아 있다   ← hostPath 가 동작한다는 증거

디스크 여유가 줄지 않는다
  df -h /              루트가 아니라 전용 볼륨에 쌓이는지

https://<도메인>/grafana/ 로 로그인된다
```

`node-exporter` 만 NODE 가 다를 수 있다 — DaemonSet 이라 모든 노드에 뜬다. 지금은 노드가 하나라 차이가 없다.

**`ss-be` Target 은 당분간 DOWN 이 정상이다.** 애플리케이션에 `micrometer-registry-prometheus` 가 들어가기 전까지 `/actuator/prometheus` 가 없다. `ss-triton` 은 **설치 당시** Service 가 없어 주석 처리해 두었으나, 10-27 21:29 에 Service 와 함께 활성화되었다. 다만 `replicas: 0` 이라 endpoint 가 없다 — §13-8.

`kubeControllerManager` · `kubeScheduler` · `kubeProxy` · `kubeEtcd` 는 **꺼두었다.** kubeadm 에서 `127.0.0.1` 에만 바인딩되어 수집에 실패하고, 켜두면 Target 4 개가 계속 DOWN 이면서 기본 알림이 발화해 **진짜 문제가 소음에 묻힌다.**

## 9. 결정 사항

**미결 없음. 1 단계는 이대로 실행할 수 있다.**

결정된 것:

```
네임스페이스   monitoring 하나로 통일 · 근거는 §9-1
저장소        기본 노드 루트 디스크로 시작 · 전환 기준은 §5-0
차트 버전     92.13.1 · 설치 직전 릴리스 노트 확인
Alertmanager  설치는 한다. 수신처는 비운다
Grafana       1 단계는 port-forward 로 확인
외부 노출     보류. X-Origin-Verify 취급도 함께 보류
비밀번호      보류. Sync 마다 어긋나므로 필요할 때 reset 한다 (§12-6)
              외부 노출과 묶어서 existingSecret 으로 처리한다
```

### 9-1. 네임스페이스 — `monitoring` 하나로 통일 (2026-10-28)

관측성 전체를 **`monitoring` 네임스페이스 하나**로 둔다. ③ 담당의 Loki · Tempo · Alloy 도 여기에 설치한다.

#### 왜 분리하지 않나

네임스페이스가 실제로 주는 것은 다섯이다. **그중 이 저장소에서 쓰이는 것이 하나도 없다.**

```
RBAC 경계          Role / RoleBinding       실측 0 건
ResourceQuota      네임스페이스 단위 제한     실측 0 건
NetworkPolicy      트래픽 제한               실측 0 건
삭제 단위          kubectl delete ns
이름 충돌 회피      같은 이름 Service 두 개
```

네임스페이스는 **노드 배치나 리소스 격리와 무관하다.** quota 를 걸지 않으면 이름표일 뿐이다. 지금 분리하면 수정 비용만 치르고 이름 정리 효과만 남는다.

#### 교환비

```
monitoring 로 통일      팀원 주석 2 줄 수정
observability 로 통일    돌아가는 스택을 재설치
```

`observability` 는 ③ 담당의 **주석에만** 존재한다. 본인들도 `임시` 라고 적어두었다.

```
platform/loki/values.yaml:2        # 임시 destination.namespace=observability
platform/alloy/logs/values.yaml:2  # 임시 destination.namespace=observability
```

반대로 `monitoring` 으로 옮기려면 Argo CD Application 의 destination 만 고쳐서는 안 된다. **Argo CD 는 네임스페이스 이동을 하지 못한다** — `monitor` → `monitoring` 때 옛 리소스가 남아 `hostPort 9100` 을 쥐고 있던 것과 같은 상황이 된다 (§12-3).

그리고 **PV 를 다시 건드린다.** `storage.yaml` 의 `persistentVolumeReclaimPolicy: Retain` 때문에 PVC 를 지우면 PV 가 `Released` 로 가고, `claimRef` 를 손으로 비워야 다시 바인딩된다. 실패 #3 · #4 로 두 번 막혔던 지점이다 (§12-1).

#### 통일로 사라지는 것

```
Grafana sidecar    searchNamespace 설정이 불필요하다 — 같은 네임스페이스는 기본값으로 읽는다
Alloy 주소         짧은 주소가 그대로 해석된다   loki-gateway:80 · tempo:4317
```

③ 담당의 Alloy 는 네임스페이스 없는 짧은 주소를 쓰고 있다. 통일하지 않으면 FQDN 으로 모두 고쳐야 한다.

`Prometheus` 는 어느 쪽이든 손댈 것이 없다. `serviceMonitorNamespaceSelector: {}` 라서 이미 모든 네임스페이스를 본다 (§5-2).

#### 분리할 때 쓸 축 — 재검토 시점을 위해

분리한다면 **신호 종류별(metrics / logs / traces)로 가르지 않는다.** 경계가 맞지 않는다.

```
Grafana   메트릭 · 로그 · 트레이스를 다 읽는다   →  둘 곳이 없다
Alloy     로그와 트레이스를 다 받는다            →  한 차트가 두 네임스페이스에 걸린다
```

맞는 축은 **소유자 / 수명주기**다.

```
 monitoring                      observability
 ┌────────────────────────┐      ┌────────────────────────┐
 │ Prometheus             │      │ Loki        S3 백엔드   │
 │ Alertmanager           │      │ Tempo       로컬 저장   │
 │ node-exporter          │      │ alloy-logs             │
 │ kube-state-metrics     │      │ alloy-otlp             │
 │ Grafana          UI    │      │                        │
 └────────────────────────┘      └────────────────────────┘
   ① 담당 · 영구 저장               ③ 담당 · 임시 저장
```

Grafana 는 읽는 쪽이고 차트가 설치하므로 왼쪽이다.

#### 재검토 조건

아래 중 **하나라도 실제로 할 계획이 생기면** 분리를 다시 판단한다.

```
③ 담당에게 네임스페이스 단위 권한만 주고 싶다        RBAC
로그 저장소를 통째로 지우고 다시 깔 일이 잦다        삭제 단위
로그가 메트릭의 노드 용량을 잡아먹는 것을 막고 싶다   ResourceQuota
```

지금이 분리 비용이 가장 싼 시점이기는 하다 — Loki · Tempo 가 아직 떠 있지 않아 왼쪽을 건드리지 않고 오른쪽만 만들면 된다. 그래도 **PoC 기간이고 RBAC 계획이 없어 통일로 간다.**

### 보류 — Grafana 외부 노출

`https://<도메인>/grafana/` 공개는 **2 단계로 미룬다.** 1 단계를 끝내고 다시 판단한다.

미루면서 함께 미뤄지는 것이 `X-Origin-Verify` 다. `ss-k8s` 가 **공개 저장소**라 검증 헤더 값을 커밋할 수 없다. 재개할 때 선택지는 둘이다.

```
Secret 으로 빼서 참조
prefix list 만으로 가고 헤더 검증 포기   ← 현재 argocd Ingress 가 이 상태
```

확인된 것:

```
네임스페이스      앱은 sayscene, 이 스택은 monitoring
ss-be Service     port name http · /actuator/prometheus
ss-triton         설치 당시 Service 없음 → 10-27 21:29 추가됨 (§13-8)
루트 디스크        16G 중 7.3G 여유
ss-k8s            공개 저장소 — 비밀값 커밋 불가
```

`ss-be` 애플리케이션 계측(`micrometer-registry-prometheus`)은 분담표상 ③ 이지만 ① 의 수집 전제다. **순서 조정이 필요하다** — 그전까지 Target 은 DOWN 이고 Grafana 는 노드·파드 지표만 보인다.

## 10. 작업 분담 — 누가 실행하나

문서·파일 작성은 로컬에서 끝나지만, **적용은 전부 기본 노드 셸과 Argo CD 웹 콘솔에서 한다.** 이 맥에는 클러스터 kubeconfig 도 AWS CLI 도 없다.

| 단계 | 실행 위치 | 누가 |
|---|---|---|
| 폴더 · values 파일 작성 | 로컬 | 대행 가능 |
| 커밋 | 로컬 | 대행 가능 (푸시는 승인 후) |
| Application 생성 | Argo CD 웹 콘솔 | **직접** |
| Sync | Argo CD 웹 콘솔 | **직접** |
| §7 검증 명령 | 기본 노드 셸 | **직접** |
| `port-forward` 로 Grafana 확인 | 기본 노드 셸 | **직접** |

**1 단계에 AWS 콘솔 작업은 없다.** 볼륨을 붙이지 않기로 했고 외부 노출을 보류했으므로, 콘솔을 건드릴 일이 2 단계까지 없다.

직접 하실 일은 **Argo CD 웹 2 종류 + 노드 셸 확인 명령**뿐이다. 디렉터리 준비는 initContainer 가 대신한다.

### 2 단계에서 생기는 AWS 작업

```
CloudFront    behavior 2 개 추가 (/grafana, /grafana/*)
              origin 은 기존 ALB 재사용
```

전용 볼륨으로 옮길 때는 여기에 **EBS 볼륨 생성·연결**이 더해진다(§5-0 전환 기준).

## 11. 알려진 한계

**Prometheus 와 Alertmanager 가 기본 노드에 있다.** 그 노드가 통째로 멈추면 **알림을 보낼 주체도 함께 멈춘다.** 2026-10-27 00:44 에 argocd 파드 7 개가 동시 재시작한 것이 그런 사건이었고, 그때 알림은 오지 않았을 것이다.

PoC 구조상 피할 수 없다. 외부 감시(CloudWatch 등)가 있어야 메워진다. 범위 밖으로 두되 **한계로 기록한다.**

**`hostPath` 는 노드 교체를 견디지 못한다.** 기본 노드는 Karpenter 관리 대상이 아니라 임의로 교체되지 않지만, 사람이 교체하면 지표가 사라진다. 교체 계획이 생기면 EBS CSI 로 옮긴다.

---

## 12. 설치 기록 — 2026-10-27

**결과: `Healthy` + `Synced` (20:25).** 사전 점검은 전부 통과했지만, 실제 설치에서 **다섯 번 막혔다.** 같은 길을 다시 돌지 않도록 원인과 교훈을 남긴다.

| # | 증상 | 원인 | 해결 |
|---|---|---|---|
| 1 | 파드가 `monitor` 에 생성 | Application 생성 시 네임스페이스 오타 | 수정 후 재생성 |
| 2 | node-exporter `Pending` | 옛 `monitor` 파드가 `hostPort 9100` 점유 | `kubectl delete namespace monitor` |
| 3 | Prometheus StatefulSet 생성 거부 | `volumeMounts` 의 `/prometheus` 중복 | 볼륨 이름을 Operator 것으로 → 실패 |
| 4 | 같은 거부, 다른 메시지 | `volumes` 이름 중복 | **local PV + `storageSpec`** |
| 5 | `FailedMount` | local PV 는 디렉터리를 안 만든다 | `sudo mkdir -p /var/lib/prometheus` |

### 12-1. 저장소 설계가 세 번 바뀌었다

가장 많은 시간을 쓴 지점이다. **Prometheus Operator 는 저장소를 `storageSpec` 으로만 받는다.**

```
시도 1   volumes(name: data) + volumeMounts(/prometheus)
         -> containers[0].volumeMounts[3].mountPath: "/prometheus": must be unique
         Operator 가 이미 /prometheus 에 마운트한다

시도 2   volumes(name: prometheus-<CR이름>-db) 로 덮어쓰기
         -> spec.template.spec.volumes[8].name: Duplicate value
         이름을 맞춰도 덮어쓰기가 아니라 중복이다

시도 3   storageSpec.volumeClaimTemplate + local PV      성공
```

**교훈:** Operator 가 관리하는 리소스는 `spec.volumes` 같은 하위 필드로 우회하지 않는다. CRD 가 제공하는 필드(`storageSpec`)를 쓴다. 우회가 통하는 것처럼 보여도 Operator 가 자기 것을 항상 덧붙인다.

### 12-2. 사전 점검이 잡지 못한 것

`helm template` 렌더링은 **차트가 만드는 매니페스트**까지만 검증한다. 시도 1·2 의 실패는 **Operator 가 CR 을 읽고 StatefulSet 을 만드는 단계**에서 났다. 그 단계는 클러스터에 CR 이 들어가야 일어나므로, 렌더링으로는 알 수 없다.

```
helm template 이 검증하는 것   차트 -> CR·Deployment·Service
검증하지 못하는 것             CR -> Operator -> StatefulSet
```

Operator 패턴을 쓰는 차트는 **렌더링을 통과해도 이 단계에서 막힐 수 있다.** 다음에 ②③ 을 설치할 때도 같다.

### 12-3. 네임스페이스 오타의 대가

`monitor` 로 한 번 Sync 된 뒤 `monitoring` 으로 고쳤는데, **`Prune` 이 꺼져 있어 옛 리소스가 남았다.** 그것이 `hostPort 9100` 을 쥐고 있어 새 node-exporter 가 뜨지 못했다.

**교훈:** 네임스페이스는 Sync 전에 확정한다. 이미 Sync 했다면 옛 네임스페이스를 통째로 지우는 것이 가장 깨끗하다. CRD 와 ClusterRole 은 네임스페이스 밖이라 영향받지 않는다.

### 12-4. 훅 타임아웃은 재시도로 끝난다

마지막에 `Sync failed: waiting for completion of hook batch/Job/kube-prometheus-stack-admission-create` 가 났다. 훅 Job 이 끝나기 전에 Argo CD 가 기다리다 실패로 처리한 것이고, **다시 Sync 하니 바로 끝났다.** 파드는 이미 `Healthy` 였다.

### 12-5. 확인된 최종 상태

```
파드          Prometheus 2/2, Grafana 3/3, Alertmanager 2/2,
              kube-state-metrics 1/1, Operator 1/1, node-exporter 1/1 x 2 노드
저장소        PVC Bound -> prometheus-local (5Gi)
              /var/lib/prometheus/prometheus-db  drwxrwsr-x root 2000
디스크        16G 중 4.6G 여유 (72%)
Application   Healthy + Synced
```

`node-exporter` 가 **노드 2 대에 각각 떴다.** 확장 워커(`ip-10-0-8-247`)에도 자동으로 따라붙은 것이 DaemonSet 이 의도대로 동작한다는 증거다.

### 12-6. Grafana 비밀번호가 Sync 마다 어긋난다 — 보류

Secret 에 적힌 값으로 로그인이 안 됐다. **차트가 렌더링할 때마다 새 비밀번호를 생성하는데, Grafana 는 처음 기동 때 받은 것만 자기 DB 에 저장**하기 때문이다.

```
Sync 1 회차   Secret: A   Grafana 가 A 를 저장
Sync 2 회차   Secret: B   Grafana 는 여전히 A      ← 어긋남
```

설치 중 여러 번 Sync 했으므로 Secret 의 값은 쓸 수 없다. **Sync 할 때마다 다시 어긋난다.**

#### 지금 푸는 법

```bash
kubectl -n monitoring exec -it deploy/kube-prometheus-stack-grafana -c grafana -- \
  grafana cli admin reset-admin-password '<새 비밀번호>'
```

#### 근본 해결 — Secret 을 고정한다

`ss-k8s` 는 **공개 저장소**라 비밀번호를 values 에 직접 쓸 수 없다. Secret 을 손으로 만들고 **이름만 참조**한다.

```bash
# 기본 노드에서 한 번만. 깃에는 올리지 않는다
kubectl -n monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password='<비밀번호>'
```

```yaml
# grafana/values.yaml
grafana:
  admin:
    existingSecret: grafana-admin
    userKey: admin-user
    passwordKey: admin-password
```

#### 결정: 보류

**적용하지 않는다.** 이유는 셋이다.

```
Argo CD 가 관리하지 않는 Secret 에 의존하게 된다
네임스페이스를 다시 만들면 손으로 다시 만들어야 한다
지금 Grafana 는 port-forward 로만 접근하고 사용자는 혼자다
```

`/var/lib/prometheus` 디렉터리와 같은 성격의 **수동 전제가 하나 더 생기는데**, 지금 얻는 것이 그만큼 크지 않다.

**다시 볼 시점은 외부 노출(§7 2 단계) 때다.** 그때는 여러 사람이 쓰게 되고 비밀번호가 Sync 마다 바뀌는 상태를 유지할 수 없다. 외부 노출과 묶어 한 번에 처리한다.

그때까지는 Sync 후 로그인이 안 되면 **`reset-admin-password` 를 다시 실행한다.** 그것이 현재 운영 방식이다.

### 12-7. Grafana 접속 주의

`port-forward` 를 **기본 노드에서 실행하면 그 노드의 `127.0.0.1` 에만 묶인다.** 브라우저에서 바로 열리지 않는다. 로컬 PC 에서 보려면 SSH 터널이 필요하다.

EIP 는 `kubectl get nodes -o wide` 의 `EXTERNAL-IP` 열에서 본다. 이 저장소는 공개이므로 값을 적지 않는다.

```bash
ssh -i <키> -L 3000:localhost:3000 ec2-user@<기본 노드 EIP>
```

터널을 띄운 뒤 **그 노드에서 `port-forward` 가 돌고 있어야** 브라우저의 `localhost:3000` 이 연결된다.

외부 노출(§7 2 단계)을 하면 이 과정이 없어진다. 현재는 보류 상태다.

---

## 13. 배포 후 리뷰 — 2026-10-28

`Healthy` 가 된 뒤 ① 의 파일과 ③ 담당의 파일을 함께 검토했다. §12 는 **설치 중에 막힌 것**이고, 여기는 **설치가 끝난 뒤 남은 것**이다.

| # | 이슈 | 소유 | 상태 |
|---|---|---|---|
| 1 | 관측 네임스페이스 미결 | ① | **해소** — `monitoring` 확정 (§9-1) |
| 2 | ServiceMonitor `ss-be` 중복 | ① | 수정 대기 |
| 3 | Grafana sidecar `searchNamespace` | ① | **불필요해짐** (§9-1) |
| 4 | Grafana 데이터소스 ConfigMap 없음 | ③ | 전달 대기 |
| 5 | `resources:` 미선언 4 개 | ③ | 전달 대기 |
| 6 | `values.yaml` 이 폴더 최상위 | ③ | 확인 필요 |
| 7 | Loki S3 접근 미검증 | ③ | 배포 후 검증 |
| 8 | `ss-triton` 서술이 낡음 | ① | **해소** — 문서 정정 (§13-7) |
| 9 | 깃과 클러스터 어긋남 가능 | ① | **확인 필요** (§13-8) |

**§13-8 이 13-1 의 선행 조건이다.** 설치 후 다른 담당자가 `prometheus/values.yaml` 을 바꿨고 sync 여부가 확인되지 않았다.

기본 노드 집중은 §11 에 한계로 적어두었다. Loki · Tempo · Alloy 가 올라가면 더 심해진다.

### 13-1. ServiceMonitor `ss-be` 가 둘이다 — ①

```
platform/kube-prometheus-stack/prometheus/values.yaml:50   additionalServiceMonitors: ss-be
apps/ss-be/07_servicemonitor.yaml                          ServiceMonitor ss-be
```

③ 담당이 만든 쪽은 `apps/ss-be/kustomization.yaml` 의 `resources` 에 이미 들어 있어 **실제로 올라간다.** 둘 다 같은 Service 를 같은 경로로 가리켜 **같은 타깃을 두 번 수집한다.**

```yaml
# 양쪽이 동일
selector: { matchLabels: { app: ss-be } }
endpoints: [{ port: http, path: /actuator/prometheus }]
```

**① 의 것을 지운다.** 앱 메트릭은 앱 소유자가 앱 폴더에서 관리하는 쪽이 경계가 맞다. 수집 측이 앱의 포트 이름과 경로를 알고 있어야 할 이유가 없고, 앱이 바뀔 때 ① 의 파일을 같이 고쳐야 하는 결합이 생긴다.

#### 지울 범위 — 블록 전체가 아니다

`additionalServiceMonitors` 에는 **항목이 둘** 있다.

```
prometheus/values.yaml:51   ss-be      ← 중복. 제거 대상
prometheus/values.yaml:64   ss-triton  ← 중복 아님. 지우면 수집이 끊긴다
```

처음 이 절을 적을 때 `ss-be` 만 보고 "`additionalServiceMonitors` 삭제" 라고 썼다. **블록째 지우면 `ss-triton` 까지 사라진다.** 지울 것은 `ss-be` 항목뿐이다.

#### `Prune` 이 꺼져 있어 자동 삭제되지 않는다

항목을 지우면 차트가 더는 렌더링하지 않지만, **Argo CD 가 기존 리소스를 지우지는 않는다.** §4 에서 `Prune` 을 꺼두었다. §12-3 에서 `monitor` 네임스페이스 잔존 리소스가 `hostPort` 를 쥐고 있던 것과 같은 구조다.

수정 후 손으로 지운다.

```bash
kubectl -n monitoring get servicemonitors
kubectl -n monitoring delete servicemonitor <차트가 만든 ss-be 이름>
```

지우지 않으면 **고치기 전과 똑같이 중복 수집이 계속된다.**

### 13-2. Grafana sidecar `searchNamespace` — ① · 해소

sidecar 는 **기본적으로 자기 네임스페이스만** 본다. 네임스페이스를 분리했다면 `searchNamespace: ALL` 이 필요했다. §9-1 로 통일했으므로 **설정 없이 동작한다.** 분리로 되돌리면 다시 필요하다.

### 13-3. Grafana 데이터소스 ConfigMap 이 없다 — ③

sidecar 는 켜져 있다.

```yaml
# grafana/values.yaml:20
sidecar:
  datasources:
    enabled: true
    label: grafana_datasource
```

그런데 **그 라벨을 붙인 ConfigMap 이 저장소에 없다.**

```
grep -rl grafana_datasource .
  → platform/kube-prometheus-stack/grafana/values.yaml  (라벨을 정의한 쪽뿐)
```

읽을 것이 없으므로 **Loki · Tempo 를 설치해도 Grafana 에서 보이지 않는다.** 차트가 만들어주지 않는 종류라 ③ 이 매니페스트를 추가해야 한다. 이 구멍은 13-5 와 묶여 있다 — 둘 자리가 없는 것이 원인이다.

### 13-4. `resources:` 가 네 파일 모두 없다 — ③

```
platform/loki/values.yaml          0 건
platform/tempo/values.yaml         0 건
platform/alloy/logs/values.yaml    0 건
platform/alloy/otlp/values.yaml    0 건
```

본인들도 주석에 `resources.requests 는 적용 보류` 라고 적어두었다. 네임스페이스를 통일했으니 **이 넷이 ① 의 스택과 같은 노드의 같은 여유를 나눠 쓴다.** requests 가 없으면 스케줄러가 남은 용량을 계산하지 못한다.

§2 의 실측 여유(CPU 약 2,120m)를 어떻게 나눌지 숫자가 필요하다. 디스크는 더 빡빡하다 — 7.3G 여유에서 Prometheus 가 이미 쓰고 있다.

### 13-5. `values.yaml` 이 폴더 최상위에 있다 — ③

```
platform/loki/values.yaml                            ← 폴더 최상위
platform/tempo/values.yaml                           ← 폴더 최상위
platform/kube-prometheus-stack/grafana/values.yaml   ← 하위 폴더
```

§4 의 multi-source 에서 `ref` 소스의 `path` 는 **매니페스트로도 렌더링된다.** 최상위 `.yaml` 이 그 대상이고, 그래서 `storage.yaml` 이 거기 있다. 같은 방식으로 Application 을 만들면 **`values.yaml` 이 매니페스트로 읽히려다 실패한다** — `apiVersion` · `kind` 가 없기 때문이다.

③ 의 Application 설정 확인이 필요하다. 그리고 13-3 을 풀려면 ③ 도 매니페스트를 두어야 하므로, 규칙을 맞춰둔다.

```
폴더 하나 = 차트 하나

최상위 .yaml      차트가 만들지 못하는 매니페스트   storage.yaml · datasource.yaml
하위 폴더/values   차트에 넘기는 입력값
```

**깊이로 "Argo CD 가 그대로 올릴 것" 과 "차트에 넘길 것" 이 갈린다.** 예외가 아니라 일관된 규칙이 된다.

### 13-6. Loki 의 S3 접근이 미검증 — ③

```
# loki/values.yaml:5
# Pod 의 IMDS 접근과 S3 읽기·쓰기는 배포 후 검증한다.
```

`ec2nodeclass-general.yaml` 의 `httpPutResponseHopLimit: 1` 때문에 **확장 워커의 파드는 IMDS 에 도달하지 못한다.** Loki 는 기본 노드에 고정되어 있어 될 가능성이 높지만 **확인된 바 없다.**

`serviceAccount` 의 `role-arn` 지정이 없어 **기본 노드 IAM 역할의 임시 자격 증명**에 의존한다. 그 역할에 S3 권한을 붙이는 것은 **권한 변경이라 승인이 필요하다.**

### 13-7. `ss-triton` 서술이 낡았다 — ① · 문서 수정으로 해소

문서는 "`ss-triton` 은 Service 가 없어 ServiceMonitor 를 주석 처리해 두었다" 고 적고 있었다. **설치 당시에는 맞았고, 지금은 둘 다 틀리다.**

```
d523bc0  10-27        ① 최초 커밋 — ss-triton 항목을 주석으로 둠      문서 서술이 맞던 시점
         10-27 20:25  ① 설치 완료 (Healthy + Synced)
9e86d25  10-27 21:29  asecdycom — Service 추가 + 주석 해제           문서가 낡은 시점
```

Service 와 ServiceMonitor 는 서로 **맞는다.**

```yaml
# apps/ss-triton/02_service.yaml      # prometheus/values.yaml:64
labels: { app: ss-triton }              selector: { app: ss-triton }   O
ports: [{ name: http, port: 8001 }]     endpoints: [{ port: http }]    O
```

다만 **`apps/ss-triton/01_deployment.yaml:9` 이 `replicas: 0`** 이다. 파드가 없으니 Service 에 endpoint 가 없고, **Target 은 여전히 비어 있다.** "Service 가 없어서 대상 0 개" 가 "파드가 없어서 대상 0 개" 로 바뀐 것이고, 보이는 결과는 같다.

§2 · §5-2 · §8 · §9 의 해당 서술을 정정했다.

### 13-8. 깃과 클러스터가 어긋나 있을 수 있다 — ① · 확인 필요

**설치가 끝난 뒤에 `prometheus/values.yaml` 이 한 번 더 바뀌었다.**

```
10-27 20:25  ① 설치 완료 — 이 시점의 values 가 클러스터에 있다
10-27 20:32  ai-ncw    7bfe529  retentionSize 1GiB          ① 가 sync 함
10-27 21:17  jisukim587 c64aba0  ss-be ServiceMonitor · alloy/loki/tempo values
10-27 21:29  asecdycom  9e86d25  ss-triton ServiceMonitor 활성화   ← values 변경
```

Argo CD 는 **수동 sync** 다. 20:32 이후로 Sync 를 누르지 않았다면 **돌고 있는 Prometheus 는 `ss-triton` 이 주석인 상태**이고, 깃과 다르다.

이것이 13-1 의 선행 조건이다. 중복 수집도 **양쪽이 모두 적용된 뒤에야** 실제로 발생한다.

```bash
# 어긋났는지 보는 가장 빠른 방법
kubectl -n monitoring get servicemonitors
kubectl -n sayscene  get servicemonitors
```

| 보이는 것 | 해석 |
|---|---|
| `monitoring` 에 `…-ss-be` 하나뿐 | 20:32 상태. 아직 sync 안 됨 → 중복 없음 |
| `monitoring` 에 `…-ss-be` · `…-ss-triton` + `sayscene` 에 `ss-be` | 모두 sync 됨 → **중복 수집 중** |

Argo CD 카드의 `Synced` 는 **Application 기준**이다. 다른 Application 의 커밋은 이 카드에 나타나지 않으므로, 카드만 보고 판단하면 안 된다.

### 13-9. 다음 할 일

```
①  13-8 확인      servicemonitors 실제 목록으로 깃·클러스터 일치 여부 판단
①  13-1 수정      ss-be 항목만 제거 → 커밋 → sync → 손으로 옛 ServiceMonitor 삭제
③  전달           13-3 · 13-4 · 13-5 · 13-6 · 네임스페이스 monitoring 확정 공지
```

**13-8 을 먼저 한다.** 어긋난 상태에서 13-1 을 고치면 무엇이 반영된 결과인지 구분할 수 없다.
