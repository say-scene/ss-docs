# 사용자 동작 tracing 변경 요약

> 2026-11-02 · 로컬 구현·검증 완료 / 운영 미배포 · 요구사항: [PRD §6.8](english-speaking-prd-architecture-v1.md#68-관측성)

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
| FE의 번역·추천·조회·리포트·데모 호출부 | 알려진 API를 공통 wrapper로 계측. 별도 동작은 별도 trace이며 업무 payload와 재시도 `requestId`는 유지 |
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

주소가 비어 있으면 relay는 비활성화(`204`), 비어 있지 않은 잘못된 주소는 `503`과 고정 경고로 처리한다. FE·BE·ConfigMap 변경을 함께 반영해야 하며, **이번 작업에서 운영 배포는 하지 않았다.**

Tempo에서 발화를 찾고, 해당 FE span의 `session.id`로 같은 페이지의 다른 API 흐름을 조회한다.

```traceql
{ resource.service.name = "ss-fe" && span:name = "speech.turn" }
{ resource.service.name = "ss-fe" && span.session.id = "관측용-UUID" }
```

## 검증과 남은 범위

- 구현 시 FE **73개**, BE **43개** 테스트와 양쪽 빌드, Kustomize 렌더링 통과.
- 실제 Chrome → FE SDK → BE Java agent → **모의** 추론 서버/수집기에서 동일 trace ID·부모/자식 span, 독립 동작 분리·페이지 ID 회전·FE 민감정보 제외 확인. 실제 마이크/실제 모델·운영 경로 전체 검증을 뜻하지 않는다.
- FE CLIENT 시간은 응답 헤더 수신까지, `speech.turn`은 이어지는 응답 처리까지다. 첫 TTS 음성/재생 완료 시간, 추론 내부 tracing, API p95용 histogram·대시보드는 이번 범위가 아니다.
- 운영 CloudFront/ALB의 전달과 수집 성공은 배포 후 검증 대상이다. 기존 개발 의존성 보안 경고 3건은 별도이며 구현 당시 운영 의존성 감사는 0건이었다.
