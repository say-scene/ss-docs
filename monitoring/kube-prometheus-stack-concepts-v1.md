# kube-prometheus-stack 개념 정리

> 작성 2026-10-29 · 갱신 2026-10-29 (설치 문서와 사실 대조) · 대상: `ss-k8s/platform/kube-prometheus-stack` 을 처음 읽는 팀원
> 설치 과정과 결정 배경은 [kube-prometheus-stack-install-v1.md](./kube-prometheus-stack-install-v1.md) 에 있다. 이 문서는 그 설치 문서와 values 파일을 **읽기 위한 개념**만 다룬다.
>
> `monitoring/` 의 문서 셋은 역할이 다르다.
>
> | 문서 | 답하는 질문 |
> |---|---|
> | **concepts** (이 문서) | PV·PVC·바인딩·values 를 **어떻게 읽나** |
> | [install](./kube-prometheus-stack-install-v1.md) | **왜 이렇게 했나** · 무엇이 막혔나 · 어떻게 쓰나 |
> | [handover](./loki-tempo-alloy-handover-v1.md) | ③ 담당에게 **무엇을 전달했나** |

## 1. 한 줄 요약

- Prometheus 저장소는 **hostPath 가 아니라 local PV** 다. 기본 노드(control-plane)의 `/var/lib/prometheus` 를 PV 로 노출하고 PVC 로 받아 쓴다.
- **PV 는 공급, PVC 는 요청서**다. PV 를 미리 만들어 두면 조건이 맞는 PVC 와 1:1 로 바인딩된다(정적 프로비저닝).
- 바인딩은 `WaitForFirstConsumer` 때문에 **파드가 스케줄되는 과정 안에서** 일어난다. `nodeSelector`(목적지) · toleration(출입증) · PV `nodeAffinity` 가 함께 노드를 정한다.
- 나머지 컴포넌트는 같은 패턴으로 기본 노드에 고정한다. node-exporter 만 DaemonSet + key 없는 `operator: Exists` 로 모든 노드에 뜬다.

## 2. local PV 와 hostPath

둘 다 노드 디스크의 디렉터리를 파드에 붙인다. 차이는 **쿠버네티스가 그 디렉터리의 노드 위치를 아느냐**다.

| | hostPath | local PV |
|---|---|---|
| 선언 위치 | 파드 `spec.volumes` 에 직접 | `PersistentVolume` 리소스 → PVC 로 받음 |
| 스케줄러가 노드 위치를 아나 | 모름 | 앎 (`nodeAffinity` 필수) |
| 파드가 다른 노드로 가면 | 그 노드에서 **빈 디렉터리로 조용히 뜸** | **Pending 으로 멈춤** |
| 디렉터리 생성 | `DirectoryOrCreate` 로 자동 생성 가능 | 안 만들어줌 — 미리 있어야 함 |
| `fsGroup` 소유권 조정 | 적용 안 됨 | 적용됨 |
| 용량 (`capacity`) | 개념 없음 | 선언하지만 **강제되지 않음** |

- **노드 자체가 사라지면 둘 다 데이터를 잃는다.** local PV 는 데이터를 지켜주는 것이 아니라, 엉뚱한 노드에서 빈 채로 뜨는 것을 막는다.
- 이 구성이 local PV 인 이유: Prometheus Operator 는 저장소를 `storageSpec.volumeClaimTemplate`(PVC) 으로만 받는다. `spec.volumes` 로 hostPath 를 넣으면 Operator 가 붙이는 `/prometheus` 볼륨과 충돌해 거부된다(설치 문서 §12-1). 클러스터에 StorageClass 가 없으니 PVC 가 받을 PV 를 직접 만든 것이다.
- `PersistentVolume` 안에 `hostPath:` 를 쓰는 hostPath PV 도 있지만, `nodeAffinity` 가 없어 같은 문제가 남는다.

## 3. PV 와 PVC

```
파드  ──참조──▶  PVC  ──바인딩(1:1)──▶  PV  ──가리킴──▶  실제 저장소
                (요청)                (공급)            /var/lib/prometheus
 namespaced    namespaced           cluster 범위
```

- 파드는 **PVC 만 안다.** 그 뒤가 로컬 디스크인지 EBS 인지 몰라도 된다.
- 그래서 저장소를 EBS 로 옮길 때 **PV 만 바꾸고 `storageSpec`(PVC 쪽)은 그대로 둔다.**
- 짝짓기는 PV 컨트롤러가 자동으로 한다. 조건은 아래가 모두 맞는 것이다.

```
storageClassName   같아야 함            local-prometheus = local-prometheus
용량               PV ≥ PVC 요청        5Gi ≥ 5Gi
accessModes        PV 가 요청을 포함     ReadWriteOnce
노드 조건          PV nodeAffinity 충족
```

### 정적 vs 동적 프로비저닝

| | 정적 (지금) | 동적 |
|---|---|---|
| PV 를 누가 만드나 | 사람이 미리 (`storage.yaml`) | 공급자가 PVC 를 보고 |
| provisioner | `kubernetes.io/no-provisioner` | 예: `ebs.csi.aws.com` |
| 맞는 PV 가 없으면 | PVC 가 계속 `Pending` | 새로 만들어 줌 |

### 상태 변화

```
PV:  Available ──바인딩──▶ Bound ──PVC 삭제──▶ Released
                                              Retain 이라 데이터 유지, claimRef 를 비워야 재사용
```

StatefulSet 을 지워도 그것이 만든 PVC 는 기본적으로 남는다. 데이터가 남는 이유는 두 층이다 — **PVC 가 남고**, PVC 가 지워져도 **PV 가 Retain** 이다.

## 4. `storage.yaml`

### StorageClass `local-prometheus`

| 필드 | 의미 |
|---|---|
| `provisioner: kubernetes.io/no-provisioner` | 동적 공급자가 없다. PV 는 아래처럼 직접 선언해야 한다 |
| `volumeBindingMode: WaitForFirstConsumer` | PVC 가 생겨도 바로 묶지 않고, **파드가 스케줄될 때** 노드 조건까지 보고 묶는다 |

PV 를 만들어주지 않는데도 StorageClass 를 두는 이유는 둘이다 — PVC 와 PV 를 짝짓는 **이름표**, 그리고 `WaitForFirstConsumer` 를 걸 **자리**.

### PV `prometheus-local`

| 필드 | 값 | 의미 |
|---|---|---|
| `capacity.storage` | `5Gi` | PVC 요청과 맞추는 숫자. **실제 사용 상한이 아니다** — 상한은 `retentionSize` 와 디스크 알림이 맡는다 |
| `accessModes` | `ReadWriteOnce` | **노드** 하나에서 읽기·쓰기. 파드 하나로 제한하는 것은 `ReadWriteOncePod` |
| `persistentVolumeReclaimPolicy` | `Retain` | **PVC 가 지워졌을 때** PV 와 데이터를 남긴다. 노드가 사라지는 것과는 무관하다 |
| `local.path` | `/var/lib/prometheus` | 미리 `sudo mkdir -p` 해야 한다. 소유권은 `fsGroup` 이 맞춘다 |
| `nodeAffinity` | `kubernetes.io/hostname In [ip-10-0-4-232...]` | **hostname** 으로 노드를 고른다 |

- `nodeAffinity` 는 control-plane 역할이 아니라 **hostname** 으로 고른다. 지금은 결과적으로 같은 노드일 뿐이다. 기본 노드를 교체하면 hostname 이 바뀌어 **PV 쪽만 깨지므로** 같이 고쳐야 한다.
- StorageClass 와 PV 는 **클러스터 범위**라 네임스페이스가 없다. `monitoring` 을 지워도 남는다.
- 이 파일이 적용되는 이유는 **폴더 최상위**에 있어서다. Argo CD 는 `ref` 소스 `path` 의 최상위 `.yaml` 만 매니페스트로 읽고, 하위 폴더의 values 는 읽지 않는다.

## 5. `prometheus/values.yaml`

### 저장소와 바인딩 시점

PVC 를 직접 만드는 파일은 없다.

```
values.yaml   storageSpec.volumeClaimTemplate      PVC 의 틀
   ↓ 차트가 Prometheus CR 에 넣음
Operator      StatefulSet 생성 시 volumeClaimTemplates 로 전달
   ↓
StatefulSet   파드마다 PVC 자동 생성 → local PV 와 바인딩
```

바인딩 순서는 아래와 같다. 파드가 노드에 올라간 **뒤**가 아니라 노드를 **정하는 과정 안에서** 묶인다.

```
1. PVC 생성 → Pending (WaitForFirstConsumer)
2. 스케줄러가 노드를 거름
     nodeSelector 통과?  taint 견딤?  이 노드에서 PVC 에 맞는 PV 를 쓸 수 있나?
3. 셋 다 통과한 노드(기본 노드) 선택
4. PVC ↔ PV 바인딩 → 파드 배정
```

2단계의 세 번째 조건 때문에 **진짜 고정 장치는 PV 의 `nodeAffinity`** 다. `nodeSelector` 가 없어도 toleration 만 있으면 스케줄러가 기본 노드로 보낸다. `nodeSelector` 는 의도를 드러내는 이중 안전장치다. (일반 동작 기준이며 이 클러스터에서 직접 실험하지는 않았다.)

### `nodeSelector` 와 `tolerations` — 역할이 다르다

| | 하는 일 | 비유 |
|---|---|---|
| `nodeSelector` | 노드 **라벨** `node-role.kubernetes.io/control-plane=""` 인 노드로**만** 간다 | 목적지 |
| `tolerations` | 그 노드의 **taint**(`NoSchedule`)를 견딘다. 노드를 고르지는 않는다 | 출입증 |

- 라벨과 taint 는 key 이름이 같을 뿐 **다른 것**이다. kubeadm 이 control-plane 노드에 둘 다 붙인다.
- toleration 의 주어는 **파드**다 — "이 파드는 key 가 `node-role.kubernetes.io/control-plane` 인 taint 를 값과 무관하게(`Exists`) 견딘다. 단 `NoSchedule` 만."
- `nodeSelector` 만 있으면 출입증이 없어 Pending, toleration 만 있으면 워커로도 갈 수 있다.

### 보존

| 설정 | 값 | 의미 |
|---|---|---|
| `retention` | `7d` | 기간 상한 |
| `retentionSize` | `1GiB` | 용량 상한 |

- **먼저 닿는 쪽**에서 오래된 블록부터 지운다. 지금은 디스크 여유 때문에 1GiB 가 먼저 닿아 7일을 다 못 채운다.
- 블록 단위로 지우므로 WAL 과 최근 데이터 때문에 순간적으로 1GiB 를 넘을 수 있다.

### 수집 대상 선택

| 설정 | 의미 |
|---|---|
| `serviceMonitorSelectorNilUsesHelmValues: false` | selector 를 비워두면 release 라벨로 거르지 않고 **전부** 선택 |
| `serviceMonitorNamespaceSelector: {}` | **모든** 네임스페이스 |
| `ruleSelectorNilUsesHelmValues: false` | PrometheusRule 도 release 라벨과 무관하게 선택 |

- 그래서 다른 담당자가 자기 폴더에 만든 ServiceMonitor 도 수집된다.
- **PodMonitor · Probe 는 아직 기본값(`true`)** 이다. 나중에 PodMonitor 를 release 라벨 없이 만들면 조용히 무시된다.
- Helm release 이름은 Argo CD 에 `releaseName` 이 비어 있어 **Application 이름 `kube-prometheus-stack`** 을 쓴다. `kube-prometheus-stack-operator` 같은 리소스 이름이 그 증거다.

### 리소스

requests `cpu 200m · memory 1Gi`, limits `memory 3Gi` 만. CPU limit 은 쓰로틀링으로 쿼리를 느리게 만들어 걸지 않고, 메모리만 막아 이웃 파드의 OOM 을 막는다.

### `additionalServiceMonitors` — ss-triton

```yaml
- name: ss-triton
  namespaceSelector: { matchNames: [sayscene] }
  selector: { matchLabels: { app: ss-triton } }
  endpoints: [{ port: http, path: /metrics }]
```

- `port: http` 는 프로토콜이 아니라 **Service 포트의 이름**이다. `apps/ss-triton/02_service.yaml` 의 `name: http, port: 8001` 을 이름으로 찾아간다. 프로토콜은 `scheme` 필드가 정하고 기본값이 http 다.
- ss-be 는 이미 `apps/ss-be/07_servicemonitor.yaml` 로 옮겼다(설치 문서 §13-1). 같은 이유로 ss-triton 도 앱 폴더로 옮기는 것을 검토할 수 있다. 옮긴다면 바꿀 곳은 아래다.

```
apps/ss-triton/03_servicemonitor.yaml    추가
apps/ss-triton/kustomization.yaml        resources 에 추가
prometheus/values.yaml                   ss-triton 항목 삭제
클러스터                                  Prune 이 꺼져 있어 monitoring 의 ss-triton ServiceMonitor 를 손으로 삭제
```

### 디스크 알림 `NodeDiskFillingUp`

```
100 - (남은 공간 / 전체 × 100) > 85      → 루트 디스크 사용률 85% 초과
for: 10m                               → 10분 동안 계속 넘어야 발화
```

- 넘으면 먼저 `Pending`, 10분 유지되면 `Firing` 이 된다.
- `severity: warning` 은 **라벨일 뿐**이다. 어디로 갈지는 Alertmanager 의 라우팅이 정한다. **[10-28 갱신] 수신처가 Discord 로 연결되어 실제로 전송된다**(설치 문서 §15-8).
- node-exporter 가 모든 노드에 있으므로 **워커 노드 디스크도 대상**이다. summary 의 "기본 노드" 문구와 달리 어느 노드인지는 `instance` 라벨로 구분한다.
- **[10-28 실증]** WSL 노드가 조인한 뒤 그 PC 의 윈도우 드라이브(`/mnt/e`)로 `NodeFilesystemAlmostOutOfSpace` 가 발화했다. 이 지적이 맞다는 증거다. summary 문구는 수정 대상이다.
- `fstype!~"tmpfs|overlay"` 는 컨테이너용 가상 파일시스템을 빼고 진짜 디스크만 보는 조건이다.

## 6. 나머지 values 의 공통 패턴

operator · grafana · alertmanager · kube-state-metrics 는 **`nodeSelector` + control-plane toleration + 메모리 limit** 이 공통이다.

### 고정 이유

공통 배경은 **Karpenter 가 만들고 지우는 워커에 두면 같이 흔들린다**는 것이다.

| 컴포넌트 | 기본 노드에 고정하는 이유 |
|---|---|
| Prometheus | PV 가 거기 있다 |
| Operator | 워커가 정리되면 ServiceMonitor 반영이 멈춘다 |
| kube-state-metrics | 어디서든 클러스터 전체가 보이지만 흔들리지 않게 |
| Grafana · Alertmanager | 같은 이유로 안정적인 노드에 |

### 키 경로가 다르다 — CR vs 일반 subchart

```
prometheus.prometheusSpec.nodeSelector          Prometheus CR
alertmanager.alertmanagerSpec.nodeSelector      Alertmanager CR
grafana.nodeSelector                            Deployment (subchart)
kube-state-metrics.nodeSelector                 〃
prometheusOperator.nodeSelector                 〃
```

Prometheus 와 Alertmanager 는 차트가 CR 을 만들고 **Operator 가 StatefulSet 을 만든다.** 그래서 `*Spec` 아래에 적는다. 경로를 틀리면 차트가 모르는 키라 **에러 없이 무시된다.**

### Grafana sidecar

`sidecar.datasources` · `dashboards` 를 켜 두면 `grafana_datasource` · `grafana_dashboard` 라벨이 붙은 ConfigMap 을 자동으로 불러온다. ③ 담당이 이 파일을 건드리지 않고 Loki · Tempo 데이터소스를 붙이기 위한 장치다. **[10-28 갱신] 의도대로 동작했다** — ③ 가 `platform/loki/datasource.yaml` · `platform/tempo/datasource.yaml` 을 추가했고(`12bacd7`), ① 의 파일은 한 줄도 바뀌지 않았다.

### node-exporter

```yaml
tolerations:
  - operator: Exists     # key · effect 없음
```

- key 와 effect 가 없으면 **모든 taint 를 견딘다** — control-plane, llm 풀의 `sayscene.io/workload`, NotReady 때 붙는 taint 까지.
- 모든 노드에 보내는 것은 **DaemonSet** 이고, toleration 은 막지 않게 할 뿐이다. 주석의 "보내라가 아니라 막지 마라" 가 이 뜻이다.
- 호스트의 `9100` 포트를 직접 써서 노드당 하나만 뜬다. 옛 `monitor` 네임스페이스의 node-exporter 가 `hostPort 9100` 을 쥐고 있어 새 것이 Pending 이 된 사건(설치 문서 §12-3)이 이 때문이다.

## 7. 자주 헷갈리는 점

| 헷갈리기 쉬운 이해 | 바른 이해 |
|---|---|
| hostPath 로 저장한다 | **local PV** 로 저장한다. 결과는 같지만 실패 방식이 다르다 |
| `no-provisioner` 면 StorageClass 가 필요 없다 | 이름표 + `WaitForFirstConsumer` 자리로 필요하다 |
| `capacity: 5Gi` 가 사용 상한이다 | 상한은 `retentionSize` 와 디스크 알림이 맡는다 |
| `Retain` 이면 노드를 안 지우는 한 데이터가 남는다 | **PVC 가 지워져도** 남는다는 뜻이다. 노드가 사라지면 데이터도 사라진다 |
| PV `nodeAffinity` 가 control-plane 을 고른다 | **hostname** 을 고른다. 지금은 같은 노드일 뿐이다 |
| `nodeSelector` + toleration 이 함께 taint 있는 노드를 고른다 | `nodeSelector` 는 목적지, toleration 은 출입증. toleration 은 노드를 고르지 않는다 |
| PVC 는 파드가 노드에 올라간 뒤 바인딩된다 | 노드를 **정하는 과정 안에서** 바인딩된다 |
| `port: http` 는 http 방식이다 | Service 포트의 **이름**이다 |
| warning 알림이 어딘가로 간다 | 수신처가 없어 UI 에만 보인다 |
| 디스크 알림은 기본 노드만 본다 | 모든 노드를 본다 |

## 8. 기억할 것

- 저장 구조의 진짜 고정 장치는 **PV `nodeAffinity`** 다.
- PV 에 **hostname 이 박혀 있다.** 기본 노드를 교체하면 `storage.yaml` 을 고친다.
- 루트 디스크를 Argo CD · Karpenter · CoreDNS 와 함께 쓴다. 가득 차면 노드 전체가 위험하다.
- Operator 가 관리하는 리소스는 `spec.volumes` 같은 하위 필드로 우회하지 않는다. CRD 가 제공하는 필드(`storageSpec`)를 쓴다.
- values 키 경로가 틀리면 **에러 없이 무시된다.**
- **`Prune` 이 꺼져 있다.** 깃에서 지워도 클러스터에 남으므로 손으로 정리한다.

## 9. 이 문서가 아직 다루지 않는 것

- **수집 흐름** — ServiceMonitor → Operator 가 scrape 설정 생성 → Prometheus 가 endpoint 를 scrape → TSDB(PV) 저장 → Grafana 가 PromQL 로 조회
- **`common/values.yaml`** — `kubeControllerManager` · `kubeScheduler` · `kubeProxy` · `kubeEtcd` 를 끈 이유
- **Argo CD Application** — multi-source, `valueFiles` 병합 순서, `ServerSideApply`, `Prune` 꺼짐
- **운영** — 확인 방법, 알려진 한계(설치 문서 §11), Grafana 비밀번호(설치 문서 §12-6)

### 스스로 점검하기

1. 다른 담당자가 `sayscene` 에 ServiceMonitor 를 새로 만들면 수집까지 어떤 일이 일어나나?
2. 기본 노드를 새 인스턴스로 교체하면 무엇이 깨지고 무엇을 고쳐야 하나?
3. values 에서 ss-triton 항목을 지우고 Sync 하면 클러스터의 ServiceMonitor 는 어떻게 되나?
4. 루트 디스크가 90% 가 되면 누가 어떤 경로로 알게 되나?
5. `nodeSelector` 를 `prometheusSpec` 바깥에 적으면 어떻게 되나?
