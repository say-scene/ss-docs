# Loki · Tempo · Alloy 설치 전 전달 사항

> 보내는 쪽 ① (kube-prometheus-stack 담당) · 받는 쪽 ③ (로그 · 트레이스 담당)
> 작성 2026-10-28 · **갱신 2026-10-28 (처리 현황 반영)** · 기준 커밋 `ss-k8s` `c7cec30`
> 설치 문서: [kube-prometheus-stack-install-v1.md](kube-prometheus-stack-install-v1.md)

`platform/loki` · `platform/tempo` · `platform/alloy` 의 values 를 읽고, **설치 전에 맞춰야 할 것**을 정리했다. 파일에 적어둔 `1번 담당자와 확인` 항목의 답이기도 하다.

---

## 0. 먼저 — 네임스페이스는 `monitoring` 이다

values 주석의 `임시 destination.namespace=observability` 를 **`monitoring` 으로 바꾼다.**

```
platform/loki/values.yaml:2
platform/alloy/logs/values.yaml:2
```

### 왜 통일했나

네임스페이스가 실제로 주는 것은 RBAC 경계 · ResourceQuota · NetworkPolicy · 삭제 단위인데, **`ss-k8s` 에서 쓰이는 것이 하나도 없다** (실측 0 건). quota 를 걸지 않으면 네임스페이스는 이름표일 뿐이고, 노드 배치나 리소스 격리와는 무관하다.

반대로 분리하면 비용이 생긴다. `monitoring` 쪽은 이미 `Healthy` 로 돌고 있어 옮기려면 재설치해야 하고, Argo CD 는 네임스페이스 이동을 하지 못한다.

자세한 근거와 **나중에 분리할 때 쓸 축**은 설치 문서 §9-1 에 있다.

### 통일해서 좋아지는 점 — 주소를 안 고쳐도 된다

values 에 적어둔 짧은 주소가 **그대로 동작한다.**

```
alloy/logs/values.yaml:49    http://loki-gateway:80/loki/api/v1/push
alloy/otlp/values.yaml:26    tempo:4317
```

같은 네임스페이스면 짧은 이름이 해석된다. 분리했다면 전부 FQDN 으로 고쳐야 했다.

### Prometheus 쪽은 손댈 것이 없다

`serviceMonitorSelectorNilUsesHelmValues: false` · `serviceMonitorNamespaceSelector: {}` 로 두었다. **release 라벨이 없어도, 어느 네임스페이스에 있어도 수집된다.** Loki · Tempo · Alloy 의 ServiceMonitor 를 만들면 그대로 잡힌다.

---

## 조치 목록

| | 항목 | 왜 |
|---|---|---|
| A | Grafana 데이터소스 ConfigMap 추가 | ✅ **완료** — `12bacd7` |
| B | `values.yaml` 위치 규칙 맞추기 | ✅ **완료** — `directory.include` 로 해결 |
| C | `resources.requests` 명시 | ❌ **미처리** — 상황 악화 (아래) |
| D | Tempo 디스크 | ✅ **완료** — S3 로 전환 |
| E | Loki S3 권한 검증 | ⚠️ **미처리 · 범위 2배** |

### 2026-10-28 15:59 기준 처리 현황

③ 이 전달 후 작업했다. `12bacd7`(14:13) · `e9f530e`(14:49) · `ab38336`(15:59).

```
처리됨    A 데이터소스 · B 구조 · D Tempo 디스크
남음      C resources · E S3 권한
```

**남은 둘은 모두 "보류" 라고 적힌 채 그대로다.** C 는 Tempo `metricsGenerator` 활성화로 **더 급해졌고**, E 는 Tempo 도 S3 를 쓰게 되어 **승인 대상이 둘로 늘었다.**

---

## A. Grafana 데이터소스 ConfigMap — ✅ 완료

**`12bacd7` 에서 양쪽 다 추가됐다.**

```
platform/loki/datasource.yaml     grafana_datasource: "1"
platform/tempo/datasource.yaml
```

주소도 FQDN 으로 정확하다.

```
http://loki-gateway.monitoring.svc.cluster.local:80
http://tempo.monitoring.svc.cluster.local:3200
```

Tempo 쪽 주석이 헷갈리기 쉬운 지점을 짚어두었다 — **Grafana 조회는 3200, Alloy 전송은 4317.**

아래는 당시 전달 내용이다. 기록으로 남긴다.

### (전달 당시) 데이터소스 ConfigMap 이 없다

Grafana 쪽 sidecar 는 **이미 켜두었다.**

```yaml
# platform/kube-prometheus-stack/grafana/values.yaml:20
sidecar:
  datasources:
    enabled: true
    label: grafana_datasource
  dashboards:
    enabled: true
    label: grafana_dashboard
```

이 라벨이 붙은 ConfigMap 을 `monitoring` 에 만들면 **Grafana 를 재시작하지 않아도 데이터소스가 붙는다.** ① 의 파일을 고칠 필요도 없다.

그런데 **그 라벨을 가진 ConfigMap 이 저장소에 없다.**

```
grep -rl grafana_datasource .
  → grafana/values.yaml  (라벨을 정의한 쪽뿐)
```

차트가 만들어주는 종류가 아니므로 ③ 이 매니페스트로 추가해야 한다. 바로 쓸 수 있는 형태는 아래와 같다.

```yaml
# platform/loki/datasource.yaml  (Tempo 도 같은 방식)
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasource-loki
  labels:
    grafana_datasource: "1"
data:
  loki.yaml: |
    apiVersion: 1
    datasources:
      - name: Loki
        type: loki
        access: proxy
        url: http://loki-gateway
        isDefault: false
```

```yaml
# platform/tempo/datasource.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasource-tempo
  labels:
    grafana_datasource: "1"
data:
  tempo.yaml: |
    apiVersion: 1
    datasources:
      - name: Tempo
        type: tempo
        access: proxy
        url: http://tempo:3200
        isDefault: false
```

`namespace` 를 적지 않은 것은 Application 의 `destination.namespace` 가 정하기 때문이다. **그 값이 `monitoring` 이어야 Grafana 의 sidecar 가 읽는다.**

Service 이름은 release 이름에 따라 달라지므로, 설치 후 `kubectl -n monitoring get svc` 로 실제 이름을 확인하고 맞춘다.

---

## B. `values.yaml` 위치 — ✅ 완료 (다른 방법으로)

**파일을 옮기지 않고 Argo CD 설정으로 풀었다.**

```
# platform/loki/datasource.yaml 첫 줄
# Git source: path=platform/loki, directory.include=datasource.yaml, ref=values.
```

`directory.include` 로 **`datasource.yaml` 만 매니페스트로 렌더링**하고 `values.yaml` 은 제외한다. ① 이 제안한 "폴더 깊이로 구분" 보다 **의도가 명시적이고 폴더 구조를 바꾸지 않아도 된다.** 이쪽이 낫다.

아래는 당시 전달 내용이다.

### (전달 당시) values.yaml 이 폴더 최상위에 있다

```
platform/loki/values.yaml                            ← 폴더 최상위
platform/tempo/values.yaml                           ← 폴더 최상위
platform/kube-prometheus-stack/grafana/values.yaml   ← 하위 폴더
platform/alloy/logs/values.yaml                      ← 하위 폴더
```

multi-source Application 에서 `ref` 소스의 `path` 는 **매니페스트로도 렌더링된다.** 최상위 `.yaml` 이 그 대상이고, 그래서 ① 의 `storage.yaml` 이 거기 있다.

`values.yaml` 에는 `apiVersion` · `kind` 가 없으므로, **같은 방식으로 Application 을 만들면 매니페스트로 읽히려다 실패한다.** `loki` · `tempo` 의 Application 설정을 확인해 주면 좋겠다 — 이미 다른 방식이면 이 항목은 넘어가도 된다.

그리고 A 를 하려면 어차피 매니페스트를 둘 자리가 필요하다. 규칙을 이렇게 맞추면 둘 다 풀린다.

```
폴더 하나 = 차트 하나

최상위 .yaml      차트가 만들지 못하는 매니페스트   storage.yaml · datasource.yaml
하위 폴더/values   차트에 넘기는 입력값
```

**깊이로 "Argo CD 가 그대로 올릴 것" 과 "차트에 넘길 것" 이 갈린다.**

```
platform/loki/
├─ datasource.yaml        ← Argo CD 가 그대로 올린다
└─ loki/values.yaml       ← helm valueFiles 로 넘긴다
```

하위 폴더 이름은 `alloy/logs` · `alloy/otlp` 처럼 **무엇을 설치하는지 드러나는 이름**이면 된다.

---

## C. `resources.requests` — ❌ 미처리 · **더 급해졌다**

**네 파일 모두 여전히 0 건**이고 주석의 `적용 보류` 도 그대로다.

그사이 Tempo 에 `metricsGenerator` 가 켜졌다.

```yaml
# platform/tempo/values.yaml
metricsGenerator:
  enabled: true
  processor.local_blocks:
    flush_to_storage: true
```

**metrics-generator 는 들어오는 모든 span 을 받아 집계하므로 메모리를 많이 쓴다.** 거기에 BE Java Agent 추적까지 붙어(`e9f530e`) 유입량도 늘었다.

**제한이 없는 컴포넌트가 노드 메모리를 밀어낼 수 있다.** 같은 노드에 ① 의 Prometheus 와 Alertmanager 가 있다.

아래는 당시 전달 내용이다.

### (전달 당시) resources.requests 가 네 파일 모두 없다

```
platform/loki/values.yaml          0 건
platform/tempo/values.yaml         0 건
platform/alloy/logs/values.yaml    0 건
platform/alloy/otlp/values.yaml    0 건
```

주석에 `적용 보류` 라고 적어둔 것을 보았다. **네임스페이스를 통일했으니 ① 스택과 같은 노드의 같은 여유를 나눠 쓴다.** requests 가 없으면 스케줄러가 남은 용량을 계산하지 못하고, 나중에 파드가 밀려날 때 원인을 찾기 어려워진다.

① 이 잡아둔 양은 이렇다.

```
prometheus        200m / 1Gi
operator           50m / 128Mi
grafana            50m / 128Mi
alertmanager       10m / 64Mi
kube-state-metrics 10m / 64Mi
node-exporter      10m / 32Mi
                 ─────────────
                  330m / 약 1.4Gi
```

노드는 4 vCPU · 16 GiB 이고 설치 전 여유가 CPU 약 2,120m 였다. **CPU 는 넉넉하다.** 시작값으로 이 정도면 충분해 보인다.

```
loki (singleBinary)   200m / 512Mi
loki gateway           10m / 32Mi
tempo                 100m / 512Mi
alloy-logs             50m / 128Mi
alloy-otlp             50m / 128Mi
```

`limits` 는 메모리만 걸고 **CPU 에는 걸지 않기를 권한다.** 쓰로틀링이 쿼리를 느리게 만든다. ① 도 같은 이유로 Prometheus 에 CPU limits 를 두지 않았다.

차트 기본값은 `helm template` 으로 확인하고 조정하면 된다.

---

## D. Tempo 디스크 — ✅ 완료

**S3 로 전환됐다.**

```yaml
storage.trace:
  backend: s3        # local → s3
  local: null
  s3.bucket: ss-tempo-traces-...
```

루트 디스크를 먹는 문제가 사라졌다. WAL 만 로컬이고 그건 제한적이다.

다만 **S3 를 쓰게 되면서 E 의 승인 대상이 하나 늘었다.**

아래는 당시 전달 내용이다.

### (전달 당시) Tempo 가 Prometheus 와 같은 디스크를 쓴다

values 를 읽다 찾은 것으로, 아직 공유되지 않은 내용이다.

```yaml
# platform/tempo/values.yaml
tempo:
  storage:
    trace:
      backend: local
      local:  { path: /var/tempo/traces }
      wal:    { path: /var/tempo/wal }
persistence:
  enabled: false        # ← PVC 없음 → emptyDir
```

`persistence.enabled: false` 면 **emptyDir** 이고, emptyDir 은 **노드의 루트 디스크**에 올라간다. 즉 trace 가 쌓이는 곳이 Prometheus 와 **같은 16G 디스크**다.

그 디스크는 이미 빡빡하다.

```
루트 디스크   16G 중 설치 후 여유 약 4.6G
Prometheus   retentionSize 1GiB   ← 여유가 줄어 2GiB 에서 낮춘 값이다
```

그리고 ① 이 걸어둔 알림이 있다.

```yaml
# additionalPrometheusRulesMap
alert: NodeDiskFillingUp      # 루트 디스크 85% 초과 10분 지속
```

**Tempo 에 보존 설정이 없으면 trace 가 계속 쌓여 이 알림이 발화한다.** 디스크가 더 차면 Prometheus 쓰기도 함께 위험해진다.

시연 기간이 짧으면 그대로 두어도 되지만, **의식하고 두는 것과 모르고 두는 것은 다르다.** 선택지는 셋이다.

```
1  짧은 보존 기간을 건다              compactor 의 block_retention
2  띄울 때 디스크를 보고 결정한다       df -h / 로 남은 양 확인
3  전용 EBS 볼륨으로 옮긴다            ① 도 같은 전환 기준을 세워두었다 (§5-0)
```

Loki 는 chunk 를 S3 에 보내므로 이 문제가 작지만, `singleBinary.persistence.enabled: false` 라 **WAL 은 같은 루트 디스크**를 쓴다.

---

## E. S3 권한 — ⚠️ 미처리 · **대상이 둘로 늘었다**

Tempo 도 S3 를 쓰게 되면서 **승인이 필요한 IAM 정책이 둘**이 됐다.

```
ss-loki-s3-access     버킷 2개 (chunks · ruler)
ss-tempo-s3-access    버킷 1개 (traces)          ← 새로 추가
```

양쪽 주석이 같다.

```
# Pod 의 IMDS·S3 접근은 배포 후 검증한다
```

역할은 `sayscenecontrolplanerole` **하나**다. 붙이면 **기본 노드의 모든 파드**(Argo CD · Karpenter 포함)가 그 S3 권한을 얻는다. **버킷 범위를 세 개로 제한하는 것이 유일한 방어선이다.**

아래는 당시 전달 내용이다.

### (전달 당시) Loki 의 S3 접근이 검증되지 않았다

```
# platform/loki/values.yaml:5
# Pod 의 IMDS 접근과 S3 읽기·쓰기는 배포 후 검증한다.
```

`serviceAccount` 에 `role-arn` 지정이 없어 **기본 노드 IAM 역할의 임시 자격 증명**에 의존한다. 확장 워커는 `httpPutResponseHopLimit: 1` 때문에 파드가 IMDS 에 닿지 못하므로, **Loki 를 기본 노드에 고정한 것은 맞는 선택**이다.

다만 그 역할에 S3 권한을 붙이는 것은 **권한 변경이라 승인이 필요하다.** 진행 전에 범위와 롤백 방법을 함께 공유해 주면 좋겠다.

설치 후 가장 빠른 확인은 Loki 파드 로그다. 권한이 없으면 `AccessDenied` 가 바로 나온다.

```bash
kubectl -n monitoring logs -l app.kubernetes.io/name=loki --tail=50
```

---

## ① 쪽에서 이미 끝난 것

```
네임스페이스              monitoring 확정 · 설치 문서 §9-1
Grafana sidecar          datasources · dashboards 둘 다 켜둠
ServiceMonitor 수집 조건   라벨 · 네임스페이스 제한 없음
ss-be ServiceMonitor     중복이던 ① 쪽을 제거 (8bd5da8) — apps/ss-be 것만 남는다
```

마지막 항목은 ③ 이 만든 `apps/ss-be/07_servicemonitor.yaml` 과 ① 의 `additionalServiceMonitors` 가 같은 Service 를 가리켜 **중복 수집되고 있던 것**이다. 앱 메트릭은 앱 폴더에서 관리하는 쪽이 맞다고 보아 ① 쪽을 뺐다. `07_servicemonitor.yaml` 은 그대로 두면 된다.

---

## 설치 후 확인

```bash
# 파드가 기본 노드에 떴는지
kubectl -n monitoring get pods -o wide

# Grafana 데이터소스가 붙었는지 (A 가 되어 있어야 보인다)
kubectl -n monitoring get cm -l grafana_datasource

# Prometheus 가 ServiceMonitor 를 잡았는지
kubectl -n monitoring get servicemonitors

# 디스크 — D 의 감시 지점
df -h /
```

막히는 것이 있으면 ① 에게 알려주면 된다. 설치 중 겪은 실패와 원인은 설치 문서 §12 에 남겨두었다.
