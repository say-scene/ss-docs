# SayScene BE 관측성 연결 매뉴얼 — Prometheus ServiceMonitor·OpenTelemetry

> v1 · 대상: `ss-be`, `ss-k8s`, kube-prometheus-stack, Alloy·Tempo
> 관련 기준: [SayScene PRD §6.8](../english-speaking-prd-architecture-v1.md)

## 0. 목표와 완료 기준

BE 메트릭은 **Prometheus → ServiceMonitor → `/actuator/prometheus`**, BE 추적은 **OpenTelemetry Java agent → Alloy OTLP → Tempo** 경로로 연결한다. 두 경로는 독립적이다. ServiceMonitor를 만들었다고 Tempo 추적이 켜지지 않으며, agent를 켰다고 Prometheus가 메트릭을 자동 수집하지도 않는다.

완료 기준은 다음과 같다.

1. Prometheus의 Targets에 `ss-be`가 나타나고 `UP`이며, `jvm_memory_used_bytes`와 요청 메트릭이 조회된다.
2. BE 요청을 한 번 실행했을 때 Tempo에서 `service.name=ss-be`인 trace와 BE 서버·외부 HTTP 또는 Redis span을 찾을 수 있다.
3. 메트릭·trace의 라벨과 속성에 발화 원문, 세션 쿠키, API 키가 없는지 확인한다.

현재 로컬에서 확인된 것은 BE의 `/actuator/prometheus` 응답과 `ss-k8s/apps/ss-be`의 Kustomize 렌더링이다. **클러스터의 Prometheus selector, ServiceMonitor CRD, 실제 수집과 Tempo 수신은 아직 확인하지 않았다.**

## 1. 구성요소와 파일

| 구성요소 | 파일·설정 | 역할 |
|---|---|---|
| Spring Boot Actuator + Micrometer | `ss-be/pom.xml`, `src/main/resources/application.yml` | `/actuator/prometheus` 제공. 기존 Actuator에 Prometheus registry를 추가 |
| BE Service | `ss-k8s/apps/ss-be/05_service.yaml` | `app: ss-be` Pod의 `http` 포트(80 → 8080) 노출 |
| BE ServiceMonitor | `ss-k8s/apps/ss-be/07_servicemonitor.yaml` | 같은 네임스페이스의 `app: ss-be` Service에서 `/actuator/prometheus` 수집 지시 |
| OpenTelemetry Java agent | `ss-be/pom.xml`의 Jib 이미지에 `/opt/otel/opentelemetry-javaagent.jar` 포함 | JVM 시작 옵션으로 켰을 때 HTTP·Redis 등 자동 span 생성 |
| Alloy OTLP Service | 추후 Alloy 배포 설정 | BE가 보낸 trace를 받아 Tempo로 전달 |

`ServiceMonitor`는 Prometheus Operator가 제공하는 CRD다. 따라서 kube-prometheus-stack/Operator가 CRD를 제공하기 전에 BE 매니페스트를 클러스터에 동기화하면 리소스 종류를 인식하지 못할 수 있다. [Prometheus Operator 시작 가이드](https://prometheus-operator.dev/docs/developer/getting-started/).

## 2. 1번 담당자와 맞출 값

1번 담당자는 kube-prometheus-stack/Prometheus를, 이 문서의 작성자는 BE와 ServiceMonitor를 담당한다. **Prometheus가 ServiceMonitor를 발견하는 조건**과 **ServiceMonitor가 Service를 발견하는 조건**은 서로 다른 두 단계다.

| 함께 확인할 값 | 현재 BE 쪽 초안 | 왜 맞춰야 하나 / 어긋나면 |
|---|---|---|
| Helm release 이름과 Prometheus `serviceMonitorSelector` | ServiceMonitor 라벨 `release: kube-prometheus-stack`은 임시값 | 차트의 기본 selector가 Helm release 라벨을 요구할 수 있다. 실제 release 이름이 다르면 ServiceMonitor가 있어도 Prometheus가 선택하지 않는다. 담당자가 렌더링된 Prometheus CR의 `spec.serviceMonitorSelector`를 확인해 라벨을 맞춘다. |
| Prometheus `serviceMonitorNamespaceSelector` | BE ServiceMonitor는 BE 앱과 같은 네임스페이스에 둔다 | Prometheus가 다른 네임스페이스에서 실행된다면 그 네임스페이스를 탐색 대상으로 포함해야 한다. 그렇지 않으면 라벨이 맞아도 ServiceMonitor를 발견하지 못한다. |
| BE 앱의 실제 네임스페이스 | `sayscene`을 잠정 후보로 사용하되, 현재 `apps/ss-be/kustomization.yaml`에는 `namespace`가 없다 | 클러스터 이름 `sayscene`과 Kubernetes 네임스페이스는 별개다. ArgoCD Application의 destination namespace와 실제 Service·ServiceMonitor 위치를 확인한다. ServiceMonitor의 기본 Service 선택 범위는 자기 네임스페이스다. |
| ServiceMonitor의 `spec.selector`와 Service 포트 | Service 라벨 `app: ss-be`, 포트 이름 `http`, 경로 `/actuator/prometheus` | 이 셋이 다르면 Prometheus가 Service를 못 찾거나 잘못된 포트·경로를 호출한다. `release` 라벨은 Prometheus→ServiceMonitor 선택용이고 `app` 라벨은 ServiceMonitor→Service 선택용이다. |
| 네트워크 접근 | Prometheus Pod → BE Service:80, HTTP | NetworkPolicy 또는 보안 설정이 차단하면 target은 발견돼도 scrape가 실패한다. `/actuator`를 외부 Ingress로 공개할 필요는 없다. |
| 배포 순서 | Operator/CRD 준비 → BE 새 이미지와 ServiceMonitor 반영 → Targets 확인 | 메트릭이 없는 이전 BE 이미지에 ServiceMonitor만 붙이면 `/actuator/prometheus`가 404일 수 있다. |

특히 kube-prometheus-stack의 `serviceMonitorSelectorNilUsesHelmValues` 기본값과 실제 release 이름을 확인한다. Helm values의 `{}` 표기만 보고 모든 ServiceMonitor가 선택된다고 단정하지 말고, **배포된 Prometheus CR의 selector**를 확인해야 한다. [kube-prometheus-stack values](https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/values.yaml), [Prometheus Operator API](https://prometheus-operator.dev/docs/api-reference/api/).

### 담당자에게 전달할 메모

> BE에 `ServiceMonitor`를 추가했습니다. BE Service 라벨은 `app: ss-be`, 포트 이름은 `http`, 메트릭 경로는 `/actuator/prometheus`입니다. ServiceMonitor 라벨 `release: kube-prometheus-stack`은 임시값입니다. 실제 Helm release 이름, 배포된 Prometheus CR의 `serviceMonitorSelector`와 `serviceMonitorNamespaceSelector`, BE 앱의 destination namespace를 알려주세요. 해당 값이 맞아야 Prometheus가 ServiceMonitor를 발견합니다. Operator/ServiceMonitor CRD가 준비된 뒤 BE 매니페스트를 동기화하고 Targets에서 `ss-be`가 `UP`인지 함께 확인하면 됩니다.

## 3. 적용·검증 순서

1. 1번 담당자에게 §2의 값과 CRD 설치 상태를 확인받는다. 네임스페이스가 `sayscene`으로 정해지면 ArgoCD destination namespace와 실제 배포 위치를 동일하게 맞춘다.
2. 메트릭이 포함된 BE 이미지를 준비하고, Prometheus Operator/CRD가 준비된 뒤 BE의 ServiceMonitor를 동기화한다. 클러스터 변경 전에는 적용 범위와 되돌릴 방법을 팀과 확인한다.
3. `kubectl -n <BE_NAMESPACE> get service ss-be servicemonitor ss-be`로 두 리소스가 같은 네임스페이스에 있는지 확인한다.
4. Prometheus Targets에서 BE가 발견되고 `UP`인지 확인한다. 없으면 **Prometheus→ServiceMonitor의 네임스페이스/라벨**, **ServiceMonitor→Service의 라벨/포트**, **네트워크/HTTP 응답** 순서로 좁힌다.
5. Alloy OTLP 수신 Service가 준비되면 BE JVM 시작 옵션과 환경변수를 설정하고 trace를 확인한다. Alloy Service 주소·포트는 이 단계에서 확정한다.

ServiceMonitor만 되돌릴 때는 `07_servicemonitor.yaml`을 Kustomization에서 제거하고 동기화한다. BE 메트릭 코드까지 되돌려야 한다면 BE 이미지도 이전 버전으로 되돌린다. Java agent를 끄려면 BE 실행 설정에서 `JAVA_TOOL_OPTIONS`의 `-javaagent`를 제거한 뒤 Pod를 재시작한다.

## 4. Java agent와 구간 계측

현재 Jib 이미지는 agent JAR를 **포함**하지만, BE Deployment에는 아직 `-javaagent`와 Alloy 주소가 없으므로 **추적은 활성화되지 않았다**. 활성화할 때는 다음 값을 BE 컨테이너에 설정한다.

```text
JAVA_TOOL_OPTIONS=-javaagent:/opt/otel/opentelemetry-javaagent.jar
OTEL_SERVICE_NAME=ss-be
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_EXPORTER_OTLP_ENDPOINT=http://<ALLOY_OTLP_SERVICE>:4318
OTEL_METRICS_EXPORTER=none
OTEL_LOGS_EXPORTER=none
```

`OTEL_EXPORTER_OTLP_ENDPOINT`는 OTLP/HTTP **기본 주소**이므로 `/v1/traces`를 직접 붙이지 않는다. agent가 trace 경로를 붙여 전송한다. Prometheus가 메트릭을 직접 수집하고 Alloy가 Pod 로그를 수집한다면 agent의 metrics/logs OTLP 전송은 위처럼 꺼 중복을 피한다. [OpenTelemetry OTLP 설정](https://opentelemetry.io/docs/languages/sdk-configuration/otlp-exporter/).

Java agent는 Java `HttpClient`와 Redis Lettuce를 자동 계측한다. 현재 BE의 STT·LLM 호출은 `HttpClient`, 대화 이력 저장은 Spring Data Redis의 Lettuce 경로를 사용한다. 자동 span은 **HTTP 호출·Redis 명령**을 보여주지만, 그 호출이 학습 흐름의 `stt`인지 `llm`인지, 중간에 어떤 자체 처리를 했는지까지 업무 의미를 자동으로 붙여 주지는 않는다. 필요하면 그 구간을 감싸는 수동 span을 추가한다. [지원 라이브러리](https://opentelemetry.io/docs/zero-code/java/agent/supported-libraries/), [수동 span](https://opentelemetry.io/docs/zero-code/java/agent/annotations/).

### 나중에 추가하거나 빼는 방법

| 바꿀 대상 | 변경 위치 | 확인할 영향 |
|---|---|---|
| 특정 라이브러리 자동 span | Java agent 환경변수 `OTEL_INSTRUMENTATION_<NAME>_ENABLED` 등 | 환경변수·Pod 재시작으로 변경 가능. 종속된 자동 계측과 기존 대시보드·검색 결과를 확인 |
| `stt`, `llm`, Redis 조회/갱신 등 업무 구간 span | BE 코드의 OpenTelemetry API 또는 `@WithSpan` | 코드·이미지 배포가 필요. span 이름/속성을 바꾸면 Tempo 검색·대시보드도 수정 |
| `turn_latency_seconds{stage}` 같은 커스텀 메트릭 | BE의 Micrometer 계측 코드 | 코드·이미지 배포가 필요. 기존 PromQL·알림·대시보드와 라벨 수를 확인 |
| 수집 비율·보관 기간 | Java agent/Alloy/Tempo/Prometheus 설정 | 저장량과 비용, 과거 구간과 비교 가능한 정도가 바뀜 |

따라서 **PRD의 관측 구간은 처음부터 전부 고정된 코드 목록이 아니다.** 운영 중 유용한 구간을 더하고 불필요한 구간을 뺄 수 있다. 단, 이미 쓰는 메트릭·span 이름을 제거하기 전에는 이를 참조하는 대시보드와 알림을 확인하고, 발화 원문·세션 식별자·API 키를 span 속성이나 메트릭 라벨에 넣지 않는다. 계측을 끄더라도 과거에 저장된 데이터는 각 저장소의 retention이 끝날 때까지 남을 수 있다. agent는 개별 계측을 설정으로 끌 수 있으며, 세밀한 업무 구간은 코드로 정의한다. [agent 계측 제어](https://opentelemetry.io/docs/zero-code/java/agent/disable/).

PRD §6.8의 `TTS 시작/완료`는 현재 **BE가 아닌 브라우저 `speechSynthesis`**에서 실행되므로 BE Java agent로 측정할 수 없다. 필요하면 프론트 계측으로 별도 설계하거나 관측 범위를 조정한다. 또한 `rate_limited_total`은 PRD §6.9에서 초기 rate limit을 제외했으므로 현 기능에 맞춰 추후 도입 여부를 판단한다. 이 매뉴얼은 그 차이를 기록하며 PRD 자체는 변경하지 않는다.

## 5. 관측 스택 자원 계획: 짧은 시연과 안정적 검증

아래 두 열은 **배포 전 추정치이며 `resources.requests` 적용은 보류**한다. 제품의 공식 최소 사양이나 실제 사용량의 상한은 아니다. CPU 노드 1대(4 vCPU, 16 GiB, x86_64), 낮은 로그·추적 유입량, 각 구성요소 1개 복제본을 가정했다. node-exporter와 로그 수집 Alloy는 CPU 노드 1대에 실행되는 DaemonSet Pod 1개씩으로 계산했다. GPU 노드에 같은 DaemonSet을 배포하면 노드당 요청량이 추가된다. 현재 Alloy·Tempo·Loki values에는 명시적 자원 요청을 두지 않았으며, Helm 차트 기본 요청량이 있을 수 있다.

| 구성요소 | 짧은 시연 CPU | 짧은 시연 메모리 | 안정적 검증 CPU | 안정적 검증 메모리 |
|---|---:|---:|---:|---:|
| Prometheus | 300m | 1Gi | 500m | 2Gi |
| Grafana | 100m | 512Mi | 250m | 768Mi |
| Alertmanager | 50m | 128Mi | 100m | 256Mi |
| Prometheus Operator | 50m | 128Mi | 100m | 256Mi |
| kube-state-metrics | 75m | 192Mi | 100m | 256Mi |
| node-exporter (노드당) | 50m | 64Mi | 50m | 128Mi |
| Alloy 로그 수집 (노드당) | 150m | 192Mi | 250m | 384Mi |
| Alloy OTLP 수신 | 100m | 192Mi | 200m | 384Mi |
| Loki | 300m | 1Gi | 600m | 2Gi |
| Tempo | 250m | 768Mi | 500m | 1.5Gi |
| **합계 (CPU 노드 1대 기준)** | **1,425m** | **4,224Mi (약 4.1Gi)** | **2,650m** | **8,064Mi (약 7.9Gi)** |

수집·저장 경로인 Prometheus, 로그 수집 Alloy, Loki, OTLP 수신 Alloy, Tempo를 우선한다. Grafana 화면은 다소 느려도 되므로 시연 요청 CPU를 낮게 잡았다. Loki는 CPU만 소폭 낮추고 메모리 1Gi를 유지한다. Operator는 `ServiceMonitor` 등의 설정을 반영하는 제어 구성요소이므로 계속 실행하되, 매번 메트릭을 수집하는 Prometheus보다 적게 예약한다. kube-state-metrics에 CPU를 더 주는 것만으로 데이터가 즉시 갱신되지는 않는다. 객체 상태 갱신과 Prometheus 수집 간격을 함께 봐야 한다.

최근 노드 출력은 기존 요청량 `1600m (43%)`, 메모리 `1328Mi (8%)`였다. 이 `1600m`에 표의 Pod가 **하나도 포함되지 않았다고 가정하면**, 짧은 시연 추정치와의 단순 합계는 `3025m`이다. 이 계산은 배포 승인치가 아니다. 실제 Helm 렌더링에는 Loki gateway 같은 추가 Pod가 나타나므로 표의 합계가 실제 예약량보다 작을 수 있다. 차트 버전과 실제 Pod 목록·노드 `allocatable`·BE/Redis/Ingress 사용량을 확인한 뒤 요청량을 확정한다. 노드 배치는 Karpenter의 노드 공급과 별개로 각 Pod의 `nodeSelector`·`tolerations`가 결정한다. 현재 Tempo·OTLP Alloy는 control-plane 라벨과 taint를 전제로 하고, 로그 Alloy는 DaemonSet으로 실행된다. 실제 노드 라벨·taint를 확인하기 전에는 배치를 확정하지 않는다.

시연에서는 중요한 BE 메트릭을 약 15초, 클러스터 상태 메트릭을 약 30초 간격으로 수집하는 것을 시작점으로 삼고, 실제 수집 시간과 CPU 사용량을 보고 조정한다. Alertmanager는 Prometheus가 만든 경고의 전달·묶음·중복 억제를 담당한다. 알림 지연은 CPU 요청량뿐 아니라 수집 간격, 규칙 평가 간격, 규칙의 `for`, Alertmanager의 `group_wait`에 좌우된다. 로그용 Alloy가 멈추거나 Loki 전송이 오래 실패하면 새 로그의 조회가 늦어지거나 일부가 유실될 수 있다. Alloy의 재시도·큐는 무손실을 보장하지 않으므로 `loki_write_dropped_entries_total`과 전송 지연을 확인한다. Loki WAL은 Loki가 이미 받은 로그를 보호한다. [Alloy `loki.write`](https://grafana.com/docs/alloy/latest/reference/components/loki/loki.write/), [Loki WAL](https://grafana.com/docs/loki/latest/operations/storage/wal/), [Alertmanager 설정](https://prometheus.io/docs/alerting/latest/configuration/).

위 수치는 **검토용 용량 계획**이다. 현재 관측 스택의 설치 여부, 실제 Pod 요청량, 로그·추적 유입량, 보관 기간과 저장소 용량은 아직 확인하지 않았다. 차트 기본값까지 포함한 렌더링과 노드 현황을 확인한 뒤 요청량을 정하고, CPU throttling, OOM, 수집 실패, Alloy의 로그 폐기, Loki/Tempo 쓰기 지연을 보고 조정한다.

### Loki 초기 배포와 연결 값

`ss-k8s/platform/loki/values.yaml`은 `grafana-community/loki` 차트의 `deploymentMode: Monolithic` 단일 복제본 초안이다. 예전 차트의 `SingleBinary` 모드와 같은 구성이고, 현재 차트에서도 Pod 설정 키는 `singleBinary`다. Loki와 별도로 gateway Pod가 실행되며, 초기 시연에서는 canary·두 Memcached 캐시를 꺼 둔다. [Loki Monolithic 설치 문서](https://grafana.com/docs/loki/latest/setup/install/helm/install-monolithic/), [차트 구성 요소](https://grafana.com/docs/loki/latest/setup/install/helm/concepts/).

Loki는 처음부터 S3를 사용한다. **설치 전 필수 확인**: 실제 S3 chunks/ruler 버킷 이름과 리전, 버킷 접근 권한, Loki Pod에 임시 AWS 자격 증명을 제공하는 방식, S3 네트워크 연결. 현재 values의 `CHANGE_ME_*`는 미확정 표시이며 그대로 설치하면 안 된다. `ss-k8s/platform/karpenter/values.yaml`에는 `eksControlPlane: false`가 있으므로 EKS Pod Identity가 바로 사용 가능하다고 가정하지 않는다. AWS는 IRSA가 자체 관리형 Kubernetes에서도 가능하다고 설명하지만 OIDC 및 관련 구성이 있는지 확인해야 한다. 대안으로 EC2 인스턴스 프로파일을 쓰려면 Pod의 자격 증명 접근 범위도 검토한다. 액세스 키를 values에 넣지 않는다. [AWS 서비스 계정 안내](https://docs.aws.amazon.com/eks/latest/userguide/service-accounts.html), [Loki AWS 저장소 안내](https://grafana.com/docs/loki/latest/setup/install/helm/deployment-guides/aws/).

**버킷 분리 결정:** 기존 FE 정적 파일 버킷 `ss-fe-944296702922-ap-northeast-2-an`은 유지한다. Loki는 서울 리전(`ap-northeast-2`)에 **범용 S3 버킷** 두 개를 별도로 만들어 `chunks`(로그 데이터)와 `ruler`(규칙)에 각각 사용한다. 버킷 이름은 생성 후 values에 반영한다. FE 배포 권한과 Loki의 읽기·쓰기 및 보관 정책을 분리하기 위한 결정이다. [Grafana Loki AWS 설치 안내](https://grafana.com/docs/loki/latest/setup/install/helm/deployment-guides/aws/).

AWS 설정 순서는 (1) 사용 중인 리전과 기존 버킷 유무 확인, (2) Loki 전용 chunks/ruler 버킷 준비, (3) 선택한 Pod 자격 증명 방식으로 두 버킷에 대한 최소 IAM 권한 연결, (4) 클러스터에서 S3 접근 확인, (5) values의 버킷·리전 대체 후 Helm 렌더링·Argo CD 적용이다. Grafana의 AWS 예시는 `s3:ListBucket`, `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`를 버킷 ARN과 객체 ARN에 한정한다. 기존 S3 버킷/역할이 있으면 먼저 재사용 가능 여부를 확인한다. 실제 AWS 버킷·IAM 생성이나 권한 변경은 별도 승인 후 진행한다. [Loki AWS 배포 안내](https://grafana.com/docs/loki/latest/setup/install/helm/deployment-guides/aws/).

Loki S3 데이터와 달리 `singleBinary.persistence.enabled: false`인 초기 설정의 WAL 등 로컬 작업 데이터는 Pod 재생성 시 사라질 수 있다. Tempo도 현재 로컬 임시 저장소다. S3 버킷과 접근 방식을 확정한 뒤 retention 및 PVC 필요성을 정한다.

서비스 주소는 모두 배포 후 실제 release 이름과 네임스페이스로 다시 확인해 변경한다. 로그 Alloy의 Loki gateway URL(`platform/alloy/logs/values.yaml`), OTLP Alloy의 Tempo 수신 주소(`platform/alloy/otlp/values.yaml`), BE의 Alloy OTLP 수신 URL, Grafana의 Loki·Tempo 데이터소스 URL이 대상이다. 동일 네임스페이스일 때만 현재 짧은 DNS 이름이 유효하다.
