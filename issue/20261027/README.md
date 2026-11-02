# 2026-10-27 이슈 기록

> 작성일: 2026-10-27 · 범위: GPU quota 미승인에 따른 CPU vLLM 전환 작업 중 발생한 이슈

## 배경

- AWS GPU 인스턴스(G·VT) vCPU quota 증가 요청이 승인되지 않아, STT·LLM을 Triton+GPU 대신 **CPU 노드로 분리**하기로 했다.
- LLM은 vLLM CPU 공식 이미지 + `Qwen/Qwen3-4B-Instruct-2507`, 노드는 Karpenter `llm` NodePool(`m7i.2xlarge` 최대 1대)로 띄운다.
- 모델 가중치와 vLLM 이미지는 llm 전용 AMI에 미리 넣는다(기동 시 외부 다운로드 없음).

| 구성 | 값 |
|---|---|
| 모델 | `Qwen/Qwen3-4B-Instruct-2507` @ `cdbee75f17c01a7cc42f958dc650907174af0554` (`/opt/models`, 7.6G) |
| 이미지 | `docker.io/vllm/vllm-openai-cpu:v0.32.0-x86_64` (digest `sha256:827ef381…0701`) |
| AMI | `sayscene-worker-llm-v1` `ami-0e8d33b291f90152d` (폐기 예정) → `sayscene-worker-llm-v2` `ami-019b11c51aea1b31d` |
| ss-k8s | `18519f0` llm NodePool·EC2NodeClass · `29e2420` `apps/llm`·ss-be 추론 주소 · `53a51c7` AMI v2 |
| Argo CD | Application `llm` (path `apps/llm`, namespace `sayscene`) 웹 콘솔에서 등록 |

## 이슈 목록

| # | 이슈 | 상태 |
|---|---|---|
| 01 | [llm-v1 AMI로 m7i 노드 조인 실패 — 주 NIC 매칭이 `ens5`로 고정](01-llm-ami-ens5-primary-nic.md) | 해결 (AMI v2) |
| 02 | [새 노드 vLLM 기동 13분 — EBS lazy loading](02-ebs-lazy-loading-cold-start.md) | 원인 확인, 개선 미정 |
| 03 | [BE → vLLM 400 Bad Request — Java HttpClient의 h2c 업그레이드](03-be-vllm-400-h2c-upgrade.md) | 코드 수정, 배포 대기 |
| 04 | [ConfigMap 변경이 ss-be에 반영되지 않음 — Argo CD 미동기화 + envFrom](04-configmap-not-applied.md) | 해결 (수동 sync·재시작) |

## 경미한 관찰

- **`FailedCreatePodSandBox ... aws-cni ... failed to setup network policy`**: llm 노드가 Ready 직후 한 번 발생. 노드의 network policy agent(`aws-node`의 `aws-eks-nodeagent`)가 준비되기 전 호출된 것으로 보이며, kubelet 재시도로 17초 안에 해결됐다. 반복되면 해당 노드의 `aws-eks-nodeagent` 로그(`/sys/fs/bpf` 마운트 등)를 확인한다.
- **`apps/llm`이 배포되지 않음**: 기존 Application `sayscene`의 path는 `apps/ss-be` 하나뿐이다. 매니페스트의 namespace는 Application의 `destination.namespace`가 정한다. llm은 비용 노드를 띄우므로 독립 Application으로 분리했다.
- **수동 EC2 조인 불가**: EC2NodeClass userData는 `karpenter.sh/unregistered` taint를 붙인다. Karpenter가 만들지 않은 노드는 이 taint가 제거되지 않으므로, 노드는 NodePool로만 띄운다.
