# 장애 유형별 점검 및 대응 방법

## E2E 응답 시간이 증가한 경우

**원칙: 알림 확인 → 느린 요청 찾기 → 느린 구간 확인 → 해당 구간만 조치 → 다시 측정.** 알림이 오지 않아도 사용자가 느리다고 하면 같은 순서로 확인한다.

### 기준과 현재 상태

아래 수치는 **운영하기로 결정한 기준**이다. `speech.turn` p95는 **STT 요청 직전부터 대화 응답 처리까지, 발화 1회**를 뜻한다. 녹음·TTS 시간은 포함하지 않는다. 평시 5초 이내를 목표로 보고, 5초를 넘으면 원인을 살피며, **10초 이상이면 알림 대상**으로 본다.

| 항목 | 평시 기준 | 알림·점검 기준 |
|---|---:|---:|
| vLLM 슬롯 수 | 4개(현재 매니페스트) | 실제 실행·대기 수와 함께 확인 |
| 동시 사용자 | 4명 | 8명 이상 |
| 컨트롤 플레인 CPU / 메모리 / 디스크 | 미정 / 미정 / 미정 | 각각 80% 이상 |
| GPU KV 캐시 사용률 | 75% 이하 | 75% 이상 |
| vLLM TTFT p99 / TPOT p99 | 0.164초 / 0.2초 | 3.2초 이상 / 0.4초 이상 |
| BE API 오류율 | 1% 미만 | 1% 이상 |
| 발화 1회 `speech.turn` p95 | 5초 이내 | 10초 이상 |

75%는 평시·알림 표기의 경계가 겹치므로 **알림 기준을 우선**한다. 평시 CPU·메모리·디스크의 빈 값은 임의로 채우지 않았다. 동시 사용자 수와 BE 오류율의 집계 방식은 알림 규칙 구현 시 확정해야 한다.

**현재 설정과의 차이:** `ss-k8s` 매니페스트의 vLLM 슬롯은 **4**다. 현재 `NodeDiskFillingUp`은 디스크 **85% 초과 10분**, `VllmTtftHigh`는 **TTFT 5분 평균 1.2초 초과 2분**을 사용한다. 위 표의 디스크 80%·TTFT p99 3.2초를 비롯한 기준을 실제 Discord 알림이 이미 모두 적용한다고 해석하지 않는다. 현재 Discord 알림은 평일 09:00~19:00(KST)에만 발송된다. [현재 구성과 알림 규칙](../english-speaking-prd-architecture-v1.md#6-관측성)을 확인한다.

### 볼 대시보드

| 대시보드 | 먼저 볼 패널 |
|---|---|
| **Sayscene BE Service** | API별 요청·오류, ALB/STT/vLLM 오류, BE 로그, TraceID·Tracing |
| **vLLM Monitoring** | 실행·대기 요청, TTFT·TPOT p99, E2E 지연, KV 캐시, 토큰 처리량 |
| **Node Exporter / Nodes** | 컨트롤 플레인 CPU·메모리·루트 디스크·네트워크 |
| **NVIDIA DCGM Exporter Dashboard** | GPU 사용률·메모리·온도·클럭 |

대시보드의 패널값은 같은 시간 범위에서 비교한다. vLLM 패널의 E2E는 직접 추론 요청 지연일 수 있으므로 `speech.turn` p95와 혼동하지 않는다.

### 1. 어디서 느린지 찾기

1. **Discord 알림**에서 발생 시각, 대상, 지표를 확인한다. 알림이 없으면 사용자 제보 시각을 기준으로 진행한다. 위의 결정 기준과 실제 알림 규칙을 구분한다.
2. **Grafana → Explore → Tempo**에서 해당 시각의 `speech.turn` trace를 연다. `POST /api/v1/stt`와 `POST /api/v1/conversation/turn` 중 긴 요청을 찾고, 그 안의 BE·Redis·STT/LLM 호출 시간을 비교한다. 여러 요청이 느린지 Grafana의 API별 p95와 오류율도 확인한다. trace가 없으면 같은 시간대의 지표와 로그로 확인한다.
3. **긴 구간에 따라 아래 표로 이동한다.** 이 trace만으로 vLLM 내부 대기 시간을 직접 볼 수는 없다.

### 2. 구간별 점검과 대응

| 느린 구간 | 먼저 확인 | 대응 |
|---|---|---|
| **Web/BE**: FE 요청 또는 BE 처리 | `Sayscene BE Service`의 API별 p95·오류율, ALB 오류, BE Pod 상태·재시작, Redis 호출 시간. FE 요청만 길면 브라우저·CloudFront·ALB 경로도 확인 | 오류 구간을 먼저 복구하고 다시 요청한다. Redis가 느리거나 `CONVERSATION_STORE_UNAVAILABLE`이면 Redis 상태부터 확인한다. 원인이 확인되기 전에는 BE 증설로 해결된다고 가정하지 않는다. |
| **STT**: STT 호출 또는 대기 | `stt_queue_wait_seconds`, `STT_BUSY`(503), STT Pod의 Ready 수, KEDA/HPA와 노드 상태 | Pod가 1개이고 대기·503이 늘면 자동 확장이 진행되는지 확인한다. 현재 STT는 KEDA가 1→2 Pod로 늘리며 새 노드 준비까지 약 3분 걸릴 수 있다. 이미 2개라면 동시 요청을 낮추거나 재시도 안내를 우선하고, 용량 변경은 별도 검토한다. |
| **vLLM**: 대화 생성의 LLM 호출 | `vLLM Monitoring`의 실행·대기 요청, TTFT·TPOT p99, KV 캐시, preemption, 입력 토큰 길이. `NVIDIA DCGM Exporter Dashboard`의 GPU 상태 | 대기와 TTFT가 함께 늘면 요청 집중과 긴 대화 이력을 확인한다. KV 사용량이나 preemption이 높으면 슬롯만 늘리지 말고 입력 길이·동시 요청·GPU 용량을 함께 검토한다. 현재 서비스는 로컬 GPU이므로 VPN 연결과 vLLM Pod 상태도 확인한다. |

**AWS GPU에서 vLLM을 운영하는 경우(가정):** 위 vLLM 지표에 더해 GPU 노드와 vLLM Pod가 Ready인지, 모델 로딩이 끝났는지, BE에서 추론 API까지 연결되는지 확인한다. 새 GPU 노드·Pod가 생겨도 준비 전에는 요청을 처리할 수 없다. 확장 결정을 내리기 전에 대기열 증가가 용량 부족인지, 모델 준비·연결 장애인지 먼저 구분한다. 현재 배포 구성은 AWS vLLM이 아니다.

### 3. 조치 후 확인

같은 시간 범위에서 **발화 `speech.turn` p95, BE API 오류율, STT 대기 또는 vLLM 대기/TTFT**가 회복됐는지 확인한다. 목표는 각각 **5초 이내, 1% 미만**이다. 발생·조치·회복 시각과 근거 지표를 기록한다. 한 구간이 회복됐는데도 E2E가 느리면 1단계부터 다시 확인한다.

## 자세한 문서

- [관측성 매뉴얼](sayscene-observability-manual-v1.md) · [사용자 동작 tracing](../user-action-tracing.md): Grafana·Tempo·로그 조회
- [현재 서비스 구성과 지표](../english-speaking-prd-architecture-v1.md#6-관측성): trace 범위, API p95, Discord 알림 조건
- [STT 확장 실측](../manual/stt-cpu-whisper-setup-v1.md) · [KEDA 설정](../manual/keda-install-v1.md): 대기·503·Pod 확장
- [vLLM 부하 분석](vllm-load-analysis-v1.md) · [누적 대화 부하 실험](vllm-loadtest-results-20261103.md): 대기·TTFT·KV와 긴 대화 이력 해석
