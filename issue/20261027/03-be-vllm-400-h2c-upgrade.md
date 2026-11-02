# 03. BE → vLLM 400 Bad Request — Java HttpClient의 h2c 업그레이드

> 상태: **코드 수정 완료, 커밋·배포 대기** (ss-be) · 발생: 2026-10-27

## 증상

로컬 Mac에서 CloudFront 경유로 `POST /api/v1/demo/cafe?scenarioId=cafe-order` 호출 시:

```json
{"code":"INFERENCE_FAILED","requestId":"bc9dd099-635d-48a7-aaa5-07efc7dafe97"}
```

같은 vLLM에 Pod 안에서 `curl`로 보낸 `/v1/chat/completions` 요청은 정상 응답했다.

## 진단

| 확인 | 결과 |
|---|---|
| ss-be 로그 (`requestId`) | `inference_failed requestId=bc9dd099-... status=400` → 연결 성공, vLLM이 400 반환 |
| vLLM 로그 | `WARNING: Unsupported upgrade request.` → `"POST /v1/chat/completions HTTP/1.1" 400 Bad Request` → `WARNING: Invalid HTTP request received.` |
| BE 요청 본문 (로컬 소스) | `model`, `stream:false`, `max_tokens`, `temperature`, `top_p`, `messages`만 전송. 프롬프트 약 1KB |

## 원인

ss-be는 `HttpClient.newBuilder().connectTimeout(...).build()`로 클라이언트를 만든다. Java HttpClient의 기본 버전은 **HTTP/2**라서, `http://`(평문) 주소에는 `Upgrade: h2c` 헤더로 HTTP/2 업그레이드를 시도한다. vLLM의 uvicorn은 h2c를 지원하지 않고, 업그레이드 요청에 POST 본문이 붙으면 요청을 해석하지 못해 400을 반환한다.

- `curl`은 기본 HTTP/1.1이라 같은 요청이 성공했다.
- 로컬 개발에서 쓰던 Ollama(Go)는 업그레이드 요청을 무시하고 처리해 드러나지 않은 것으로 본다(추정).

## 해결

ss-be의 HttpClient 두 곳에 HTTP/1.1을 고정했다.

```java
HttpClient.newBuilder().version(HttpClient.Version.HTTP_1_1).connectTimeout(Duration.ofSeconds(5)).build();
```

| 파일 | 이유 |
|---|---|
| `src/main/java/com/sayscene/InferenceController.java` | vLLM 호출 |
| `src/main/java/com/sayscene/SttController.java` | STT 서버도 Python(uvicorn 등)이면 같은 문제 |

## 검증

- 로컬에 Maven이 없어 컴파일·테스트는 실행하지 못했다. 커밋 전 테스트 또는 CI 결과로 확인한다.
- 배포 후 `/api/v1/demo/cafe`가 정상 응답하고, vLLM 로그에 `Unsupported upgrade request`가 더 이상 찍히지 않는지 확인한다.

## 남은 일

- ss-be 커밋 → `ecr-push` 빌드 → ss-k8s `apps/ss-be/kustomization.yaml` `newTag` 갱신 → `sayscene` sync.
- `INFERENCE_TIMEOUT_SECONDS`(45초)와 CloudFront 오리진 응답 대기(기본 30초)가 CPU 추론 시간에 충분한지 실측한다. 리포트(`max_tokens` 4096)·번역(2048)은 초과 가능성이 높다.
