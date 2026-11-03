# 사용자 동작 tracing 변경 요약

> 구현 기록: 2026-11-02 · 운영 상태 갱신: 2026-11-03 · 현재 명세: [PRD 관측성](english-speaking-prd-architecture-v1.md#6-관측성)
> 사용자 확인: Grafana·Tempo의 FE→BE trace 조회, Alloy의 OTel·로그 수집, Loki 로그 조회와 Alertmanager→Discord 알림 수신을 운영에서 확인했다. 아래 로컬 테스트 기록은 당시의 검증 범위를 보존한다.

## 무엇을 바꿨나

기존에는 BE에 들어온 HTTP 요청별 trace만 확인할 수 있었다. FE에 OpenTelemetry를 추가해 **한 번의 발화에서 이어지는 STT와 대화 생성 요청을 하나의 trace로 연결**했다. 추론 서버 내부는 상세 span을 추가하지 않고 메트릭으로 관측한다.

```text
speech.turn (FE 부모 span, 같은 trace ID)
├─ POST /api/v1/stt (FE CLIENT)
│  └─ BE SERVER → Whisper 호출 CLIENT
└─ POST /api/v1/conversation/turn (FE CLIENT)
   └─ BE SERVER → Redis / LLM 호출 CLIENT
```

레거시 대화 경로는 `/api/v1/demo/cafe/reply`를 사용한다. 위 구조는 추론 서버 자체의 SERVER/내부 span을 추가했다는 뜻이 아니다.

## 어떻게 연결했나

| 변경 위치 | 역할 |
|---|---|
| `ss_fe/src/telemetry.ts`, `main.tsx` | SDK 초기화, 동작 부모 span, API별 CLIENT span, W3C `traceparent` 전달. 전역 fetch 패치 없이 context를 명시적으로 전달 |
| `useRecorder.ts` → `ChatRoom.tsx` → `useConversation.ts` / `conversation.ts` | 같은 동작 context를 STT 콜백과 후속 대화·자동 재시도로 전달. 마이크/UI는 먼저 해제하고 부모 span만 후속 응답까지 유지 |
| FE의 번역·추천·조회·리포트 호출부 | 알려진 API를 공통 wrapper로 계측. 별도 동작은 별도 trace이며 업무 payload와 재시도 `requestId`는 유지 |
| `ss_be/src/main/java/com/sayscene/BrowserTelemetryController.java` | `POST /api/telemetry/traces`의 OTLP/JSON을 검증·정제해 고정된 Alloy 주소로 전달 |
| `ss_be` 설정, `ss_k8s/apps/ss-be/01_configmap.yaml` | `BROWSER_TELEMETRY_OTLP_ENDPOINT`로 비공개 수집 경로 설정. Collector 공개·CORS 확대 없음 |

- **Trace는 사용자 전체/대화 전체가 아니라 유한한 동작 단위**다. 시작/이어하기의 동기화와 이후 첫 인사, 종료와 리포트는 별도 trace다. 자동 재시도는 같은 동작, 나중의 수동 재시도는 새 동작이다.
- **흐름 연결용 `session.id`는 페이지 메모리의 임의 UUID**다. 모든 FE span에 기록하지만 실제 대화 `sessionId`·업무 `requestId`·사용자 계정과 연결하지 않는다. 새로고침·새 페이지에서는 바뀌며 FE span을 통해 조회한다.
- 발화·오디오·body·URL query·쿠키·인증 헤더·실제 대화 식별자·예외 원문은 새 FE span/relay 전달 대상에서 제외한다. BE Java agent의 기존 HTTP 메타데이터 수집 정책 자체를 변경한 작업은 아니다.
- Relay 제한은 요청당 **128 KiB / 128 spans**, BE 인스턴스당 **분당 60회 / 동시 4회**, upstream 전체 응답 대기 **2초**다. 익명 수집이며 인증·서비스 전체 quota는 아니다.
- 초기화·전송 실패는 업무 요청을 막지 않는다. 수집은 best effort이며 FE span 유실 시 해당 익명 ID로 흐름을 찾지 못할 수 있다.

## 설정과 조회

BE에 다음 환경변수를 설정한다. 브라우저는 same-origin `/api/telemetry/traces`에만 전송한다.

```text
BROWSER_TELEMETRY_OTLP_ENDPOINT=http://alloy-otlp.monitoring.svc.cluster.local:4318/v1/traces
```

주소가 비어 있으면 relay는 비활성화(`204`), 비어 있지 않은 잘못된 주소는 `503`과 고정 경고로 처리한다. FE·BE·ConfigMap 변경을 함께 반영해야 한다.

FE main merge는 GitHub Actions의 자동 배포를 실행하지만, BE의 Argo CD Application `sayscene`은 수동 sync 방식(`targetRevision: HEAD`)이다. Git merge·이미지 태그 갱신만으로 BE 파드가 반영되지는 않는다. `envFrom` ConfigMap 변경도 기존 파드를 자동 재시작하지 않으므로, 이후 변경도 **수집 주소 ConfigMap 적용 → 새 BE 이미지/파드 rollout → FE 연결 확인**을 함께 검증한다. 이 문서의 갱신 작업에서는 클러스터 sync나 배포를 실행하지 않았다.

Tempo에서 발화를 찾고, 해당 FE span의 `session.id`로 같은 페이지의 다른 API 흐름을 조회한다.

```traceql
{ resource.service.name = "ss-fe" && span:name = "speech.turn" }
{ resource.service.name = "ss-fe" && span.session.id = "관측용-UUID" }
```

## 구현 시 검증과 현재 범위

- 구현 시 FE **73개**, BE **43개** 테스트와 양쪽 빌드, Kustomize 렌더링 통과.
- 실제 Chrome → FE SDK → BE Java agent → **모의** 추론 서버/수집기에서 동일 trace ID·부모/자식 span, 독립 동작 분리·페이지 ID 회전·FE 민감정보 제외 확인. 실제 마이크/실제 모델·운영 경로 전체 검증을 뜻하지 않는다.
- FE CLIENT 시간은 응답 헤더 수신까지, `speech.turn`은 이어지는 응답 처리까지다. 첫 TTS 음성/재생 완료 시간과 추론 내부 tracing은 계측하지 않는다. 현재 Kubernetes ConfigMap은 BE HTTP histogram을 활성화하며 Grafana 대시보드·패널도 운영에서 확인됐다. 브라우저 동작 trace와 API p95 지표는 다른 측정이다.
- FE→BE trace의 운영 조회·수집 성공은 2026-11-03 사용자 확인을 반영했다. 노드별 로그 수집 범위와 저장 설정은 현재 [PRD 관측성](english-speaking-prd-architecture-v1.md#6-관측성) 및 ss-k8s values를 따른다. 과거 테스트 수·의존성 감사 결과는 구현 당시 기록이며 현재 상태를 보증하는 수치는 아니다.
