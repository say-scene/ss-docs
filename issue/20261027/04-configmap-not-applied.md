# 04. ConfigMap 변경이 ss-be에 반영되지 않음 — Argo CD 미동기화 + envFrom

> 상태: **해결** (수동 sync·재시작) · 발생: 2026-10-27

## 증상

ss-k8s `29e2420`에서 `apps/ss-be/01_configmap.yaml`의 추론 주소를 바꾸고 main에 머지했으나, BE 호출이 계속 실패했다.

```json
{"code":"INFERENCE_NOT_CONFIGURED", ...}
```

`INFERENCE_NOT_CONFIGURED`는 BE의 `INFERENCE_MODEL`이 비어 있을 때 반환된다.

## 진단

| 확인 | 결과 |
|---|---|
| `kubectl -n sayscene get cm ss-be-config` | `INFERENCE_BASE_URL=http://ss-triton:8000`, `INFERENCE_MODEL` 없음 → **예전 값** |
| Argo CD `sayscene` 앱 | main 최신 커밋 미반영 |

## 원인

두 가지가 겹쳤다.

1. **Argo CD가 동기화하지 않았다.** `sayscene` 앱이 main의 새 커밋을 반영하지 않아 ConfigMap 자체가 예전 값이었다.
2. **ConfigMap이 바뀌어도 Pod는 재시작되지 않는다.** ss-be는 `envFrom`으로 ConfigMap을 읽는다. 환경변수는 컨테이너 시작 시 한 번 복사되고, Deployment Pod 템플릿은 바뀌지 않았으므로 rollout도 일어나지 않는다.

| ConfigMap 사용 방식 | Pod 안 값 갱신 | 앱 반영 |
|---|---|---|
| `env` / `envFrom` | 안 됨 | 재시작 필요 |
| 볼륨 마운트 | kubelet이 주기 갱신 (수십 초~1·2분) | 앱이 파일을 다시 읽어야 함 |
| 볼륨 + `subPath` | 안 됨 | 재시작 필요 |

ss-be(Spring Boot)는 설정을 시작 시 한 번 읽으므로 어느 방식이든 재시작이 필요하다.

## 해결

1. Argo CD 웹 콘솔에서 `sayscene` 앱 App Diff 확인 후 Sync → ConfigMap `http://llm:8000  qwen3-4b` 확인.
2. `kubectl -n sayscene rollout restart deploy/ss-be`.
3. `kubectl -n sayscene exec deploy/ss-be -- printenv INFERENCE_BASE_URL INFERENCE_MODEL`로 Pod 반영 확인.

이후 호출은 `INFERENCE_FAILED`로 바뀌었고, 별도 원인([03](03-be-vllm-400-h2c-upgrade.md))으로 이어졌다.

## 재발 방지 후보

- kustomize `configMapGenerator`로 전환: 내용이 바뀌면 ConfigMap 이름에 해시가 붙고 Deployment 참조가 바뀌어 자동 rollout된다. 추가 구성요소가 필요 없다. 예전 ConfigMap 정리는 Argo CD prune 설정에 따른다(확인 필요).
- 대안: Reloader(stakater) 컨트롤러 설치.
- `sayscene` 앱의 자동 sync 여부를 확인하고, 수동이라면 머지 후 Sync를 절차에 포함한다.
