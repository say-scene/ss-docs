# SayScene 관측성 문서 통합 검토

> 2026-10-28 검토 초안. ①의 설치 기록과 ③의 BE 연결 매뉴얼을 비교했다. 클러스터 상태는 이 검토에서 직접 조회하지 않았으므로, 문서에 적힌 배포 완료와 현재 실행 상태를 구분한다.

## 비교 범위와 기준

- ① [kube-prometheus-stack 설치 문서](../monitoring/kube-prometheus-stack-install-v1.md): 메트릭 스택의 설치 기록과 2026-10-28 배포 후 리뷰.
- ③ [BE 관측성 연결 매뉴얼](ss-be-observability-integration-manual-v1.md): BE의 메트릭·추적 연결과 로그·추적 스택의 시연 계획.
- 실제 저장소의 `ss-k8s/platform/{kube-prometheus-stack,loki,alloy,tempo}`와 `ss-k8s/apps/ss-be`도 대조했다. **파일 내용은 확인했지만 Argo CD의 현재 Sync 상태와 Pod 동작은 확인하지 않았다.**
- 이 문서는 충돌과 결정 순서를 기록한다. 원본 문서, PRD, Helm values, Application, 클러스터 설정은 변경하지 않는다.

## 확인된 충돌과 상태 차이

| 항목 | ① 문서 / 현재 설정 | ③ 문서 / 현재 설정 | 영향과 정리 방향 |
|---|---|---|---|
| 관측 네임스페이스 | ① §9-1은 기존 스택의 `monitoring` 사용과 Loki·Alloy·Tempo의 합류를 **결정**으로 기록 | ③ 매뉴얼은 네임스페이스 조율을 미결로 설명하고 Loki·로그 Alloy values 주석은 `observability`를 임시값으로 표기 | **결정 서술이 충돌**한다. ③ 배포 전 팀 합의를 재확인하고, ①의 결정대로라면 두 Application의 destination을 `monitoring`으로 설정한다. values 주석과 서비스 주소도 갱신한다. 실제 배포를 이미 시작했다면 네임스페이스 이동 절차가 필요하다. |
| BE ServiceMonitor 선택 조건 | ① values는 `serviceMonitorSelectorNilUsesHelmValues: false`, `serviceMonitorNamespaceSelector: {}`로 설정 | ③ 매뉴얼은 `release: kube-prometheus-stack`을 임시값으로 보고 release 이름 확인을 선행 조건으로 적음 | **③의 선행 조건 설명이 현재 설정과 어긋난다.** 배포된 Prometheus CR의 실제 selector를 조회해 최종 판정한다. BE ServiceMonitor의 `app` 라벨·`http` 포트·경로는 별도로 확인한다. |
| BE ServiceMonitor 중복 | ① values의 `additionalServiceMonitors`에 `ss-be`와 `ss-triton`이 존재 | `apps/ss-be/07_servicemonitor.yaml`에도 `ss-be`가 존재하고 Kustomization에 포함 | 양쪽이 Sync되면 같은 BE Service를 두 번 수집할 수 있다. **①의 `ss-be` 항목만 제거**하는 방향이 합의 후보이며 `ss-triton`은 유지한다. 현재 중복이 실제로 발생했는지 클러스터 목록과 Targets로 먼저 확인한다. ① App에 Prune이 꺼져 있으므로 제거 뒤 남은 차트 생성 ServiceMonitor의 정리 방법도 정해야 한다. |
| 설치 상태 | ①은 kube-prometheus-stack 설치 완료(2026-10-27 `Healthy`·`Synced`)를 기록 | ③은 클러스터의 Operator/CRD/selector가 미확인이라고 기록 | ③의 상태 설명은 오래됐다. **①의 기록은 설치 당시 증거**이며 현재 상태 확인을 대신하지 않는다. 현재 Argo CD revision, CRD, Prometheus CR, Targets를 확인한 뒤 양쪽 문서의 상태 문구를 갱신한다. |
| BE 메트릭 적용 | ①은 BE target이 `micrometer-registry-prometheus` 미반영으로 DOWN일 수 있다고 기록 | ③은 로컬 `/actuator/prometheus` 확인과 BE 이미지/ServiceMonitor 배포를 다음 단계로 기록 | 두 문서의 시점이 다르다. BE 기능 브랜치의 이미지가 배포됐는지와 실제 endpoint 응답을 확인해야 UP 기대 시점을 정할 수 있다. |
| 용량 기준 | ①의 실제 Prometheus 요청은 `200m/1Gi`, Grafana `50m/128Mi` 등이며 설치 후 디스크 여유 `4.6G` 기록 | ③의 시연 표는 Prometheus `300m/1Gi`, Grafana `100m/512Mi` 등 **제안치**이고 `1600m` 기존 요청량을 전제로 계산 | 표를 현재 배포 설정이나 추가 요청량으로 취급하면 이중 계산된다. 노드 `allocatable`, 현재 Pod requests, 디스크 여유를 다시 측정하고 Loki gateway 등 부속 Pod와 노드당 Alloy DaemonSet을 더한다. 이전 `3025m` 합계는 현황값으로 사용하지 않는다. |
| Grafana 데이터소스 | ①은 sidecar가 `grafana_datasource` 라벨 ConfigMap을 읽도록 설정, 실제 ConfigMap 부재 기록 | ③은 Loki·Tempo 주소가 설치 후 확인 대상이나 데이터소스 파일은 아직 없음 | Loki·Tempo 설치만으로 Grafana 조회가 되지 않는다. 두 서비스의 실제 주소 확인 후 같은 네임스페이스에 데이터소스 ConfigMap을 배포하는 단계가 필요하다. |
| Git values 소스 | ① App의 Git source에는 `path: platform/kube-prometheus-stack`가 있어 그 경로의 최상위 YAML도 매니페스트로 렌더링 | ③의 Loki·Tempo `values.yaml`은 각 폴더 최상위에 있음 | **조건부 충돌**이다. ③ App의 Git source를 `ref: values`만 있는 값 공급용으로 만들면 최상위 values 파일을 매니페스트로 렌더링하지 않는다. 그 source에 `path`를 추가할 때는 렌더링 범위와 파일 배치를 재설계해야 한다. ①의 폴더 배치 규칙을 모든 App에 그대로 적용할 필요는 없다. |
| Loki S3와 로컬 저장 | ③ 매뉴얼에는 `CHANGE_ME_*` 버킷 및 IAM 미설정이라는 과거 안내가 남음 | 현재 `ss-k8s/platform/loki/values.yaml`에는 서울 리전의 chunks/ruler 버킷 이름이 입력돼 있음 | **③ 문서가 현재 파일보다 뒤처져 있다.** AWS 버킷과 IAM은 사용자 보고로 준비됐으나 Loki Pod의 IMDS·S3 읽기/쓰기는 아직 검증하지 않았다. Loki WAL은 persistence가 꺼져 있어 Pod 재생성 시 유실 가능성이 있다. |
| OTLP 추적 | ③은 Agent JAR가 BE 이미지에 포함되지만 `-javaagent`와 Alloy 주소는 미설정이라고 기록 | `ss-k8s/apps/ss-be`에도 현재 `OTEL_*`/`JAVA_TOOL_OPTIONS`가 없음 | 문서 간 직접 충돌은 없다. 다만 Alloy OTLP→Tempo 설치와 주소 확정 전에 BE trace가 보일 것으로 기대하면 안 된다. `monitoring`으로 통일하면 BE에서는 `alloy-otlp.monitoring.svc.cluster.local:4318` 같은 실제 Service 이름을 조회해 설정한다. |
| `ss-triton` 대상 | ①은 Service/ServiceMonitor가 추가됐으나 replicas 0이라 endpoint가 없다고 정정 | ③의 자원 표나 BE 연결 범위에는 Triton target이 없음 | 직접 충돌은 없다. 메트릭 스택 전체 완료 기준과 BE 메트릭 완료 기준을 구분한다. |

## 결정·검증 순서

공수는 **Kubernetes와 Argo CD UI에 익숙한 1명이 작업할 때의 대략적 시간**이다. 승인 대기, 이미지 빌드, 장애 대응, 리뷰 시간은 제외했다. 숫자는 일정 약속이 아니다.

| 순서 | 먼저 정하거나 확인할 것 | 선택이 미치는 영향 | 대략 공수 |
|---|---|---|---|
| 1. 실행 상태 확인 | ① App의 revision/Sync, `monitoring`·`sayscene`의 ServiceMonitor 목록, Prometheus CR selector, Targets, 노드 requests·디스크 여유 | 이미 적용된 것과 Git에만 있는 것을 구분한다. 이 확인 없이 중복 삭제·용량 결정을 하면 잘못된 리소스를 건드릴 수 있다. | 30~60분 |
| 2. 네임스페이스 재확인 | ①의 `monitoring` 통일 결정을 ③ 배포 기준으로 채택할지 확인 | 현재 ③의 짧은 DNS(`loki-gateway`, `tempo`)는 같은 네임스페이스에서만 유효하다. App destination, Grafana 데이터소스, BE OTLP 주소가 함께 결정된다. 이미 `observability`에 배포했다면 이동 비용이 커진다. | 결정 15~30분; 미배포 설정 반영 15~30분 |
| 3. BE 수집 소유권 | `ss-be` ServiceMonitor는 앱 폴더 것을 유지하고 ① 차트의 `ss-be` 항목만 제거할지, 기존 리소스를 누가 정리할지 결정 | 중복 scrape와 설정 소유권이 정리된다. `ss-triton` 수집을 보존해야 한다. Prune 비활성 상태에서 옛 리소스는 자동으로 지워지지 않는다. | 코드 검토·수정 30~60분; 적용·검증 별도 |
| 4. 배포 가능한 용량 | Loki·gateway·Alloy 로그/OTLP·Tempo의 실제 렌더링 Pod, requests/limits, 이미지와 임시 저장량을 기준으로 시연 자원값 확정 | 단일 CPU 노드의 스케줄링과 루트 디스크 위험에 영향. ③ 시연 표는 출발점으로 쓰되 ①의 실행 중 요청량을 다시 합산하지 않는다. | 1~2시간; 자원 부족 시 추가 작업 |
| 5. 로그 경로 | Loki20.0.2/Alloy1.14.3, Git `ref: values`와 `path` 유무, release 이름, Loki S3 연결, Alloy 전송 주소 확인 | Helm 렌더 실패, 중복 리소스, S3 접근 실패, 로그 유실을 막는다. S3 권한 범위와 Pod에서의 실제 접근은 배포 후 검증한다. | 설정·렌더 1~2시간; 배포·로그 검증 1~2시간 |
| 6. Grafana 로그 데이터소스 | `grafana_datasource` 라벨을 가진 ConfigMap의 소유 App·위치·Loki URL 결정 | Loki에 로그가 있어도 Grafana에서 안 보이는 상태를 해소한다. 같은 `monitoring` 네임스페이스라면 sidecar 범위 변경은 필요 없다는 ① 문서의 계획을 확인한다. | 30~60분 |
| 7. BE 메트릭 활성화 | BE 계측 이미지의 main 반영·배포 시점과 `/actuator/prometheus` 응답 확인 | ServiceMonitor가 존재해도 이전 BE 이미지면 target이 DOWN일 수 있다. `ss-be` Targets가 하나이고 UP인지 확인한다. | 구성 확인 30분; 빌드·배포 시간 별도 |
| 8. 추적 경로 | Tempo 차트 버전·임시 저장 허용, Alloy OTLP release/Service 이름, BE Agent 환경변수와 샘플링 결정 | BE→Alloy→Tempo trace가 이어진다. Agent 활성화는 BE 재배포가 필요하고 Tempo 로컬 저장은 Pod 재생성 시 trace를 잃을 수 있다. | 설정·렌더 1~2시간; 배포·trace 확인 1~2시간 |
| 9. Grafana 추적 데이터소스와 문서 갱신 | Tempo URL ConfigMap, 실제 namespace/release/자원/검증 결과 반영 | 화면에서 trace 확인이 가능해지고, ①·③ 매뉴얼의 과거 전제와 상태 문구가 정리된다. PRD 수정은 별도 승인 대상으로 둔다. | 1~2시간 |

### 적용 전에 확인할 증거

1. ① 담당자에게 **현재** Argo CD `kube-prometheus-stack` revision/Sync, `kubectl -n monitoring get servicemonitor`, 배포된 Prometheus CR의 `serviceMonitorSelector` 및 `serviceMonitorNamespaceSelector` 결과를 확인한다.
2. ③ 담당자는 로그·추적 Application의 Git source가 values만 공급하는지(`ref`만) 매니페스트도 생성하는지(`path` 있음) 확인하고, 렌더된 Pod/Service 이름과 자원 요청량을 본다.
3. 기존 ① 문서의 `4.6G` 여유는 설치 당시 기록이다. 새 Pod를 올리기 전에 노드 디스크와 메모리 여유를 다시 측정한다.
4. 각 Sync 후 Argo CD `Healthy`/`Synced`뿐 아니라 Prometheus Targets, Loki 쓰기·조회, S3 접근, Tempo trace 조회를 각각 확인한다.

### 변경 범위와 복구

이 문서는 검토 결과만 추가한다. 이후 ① values, BE ServiceMonitor, ③ Application을 바꿀 때는 변경 리소스와 현재 revision을 기록하고 팀 합의 후 Sync한다. 중복 ServiceMonitor 정리는 **정확한 이름을 조회한 뒤 해당 리소스만** 대상으로 한다. 로그·추적 배포 실패 시에는 해당 Application의 Sync를 중단하고 이전 Git revision의 설정으로 되돌리는 경로를 준비한다. 데이터 저장소나 네임스페이스 삭제는 이 검토의 범위가 아니다.
