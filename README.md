# ss-docs

SayScene의 제품 명세, 하이브리드 아키텍처, 구축·운영·장애·성능 실험을 기록한다. 현재 서비스는 **AWS CPU Whisper STT + 로컬 GPU vLLM(Qwen3-4B-FP8)** 구성이다.

## 먼저 읽기

| 문서 | 내용 |
|---|---|
| [제품 명세 & 운영 아키텍처](english-speaking-prd-architecture-v1.md) | 사용자 흐름, API·데이터, 현재 AI 서빙, 관측성, 배포 경계 |
| [사용자 동작 tracing](user-action-tracing.md) | FE 발화부터 BE까지 trace 연결·익명 페이지 ID·수집 제한·조회 |
| [통합 아키텍처 이미지](01%20%C2%B7%20AWS%20%2B%20Kubernetes%20%ED%86%B5%ED%95%A9%20%EC%95%84%ED%82%A4%ED%85%8D%EC%B2%98.png) | AWS와 로컬 GPU, GitOps의 전체 구성 개념도 |
| [관측성 아키텍처 이미지](02%20Observability%20Architecture.png) | 메트릭·로그·트레이스·알림의 데이터 흐름 개념도 |

이미지는 구성 개념도이며 모든 배포값을 담지는 않는다. 현재 모델·인스턴스·포트·임계값은 PRD와 각 저장소의 코드·매니페스트를 기준으로 확인한다.

## 구축·운영 문서

| 영역 | 문서 |
|---|---|
| AMI·클러스터 | [AL2023 CPU AMI](al2023-cpu-ami-setup-guide.md) · [kubeadm·Karpenter 구성](al2023-cpu-cluster-karpenter-guide.md) |
| 네트워크 | [VPC CNI 주소 중복·라우팅 규칙 삭제 방지](al2023-vpc-cni-network-prevention-guide.md) |
| 노드·STT 확장 | [general NodePool 결정 기록](karpenter-general-nodepool-decision-v1.md) · [KEDA](manual/keda-install-v1.md) · [CPU Whisper 실측·확장 기록](manual/stt-cpu-whisper-setup-v1.md) |
| 관측성 | [읽는 순서](monitoring/README.md) · [kube-prometheus-stack 설치](monitoring/kube-prometheus-stack-install-v1.md) · [구성 개념](monitoring/kube-prometheus-stack-concepts-v1.md) · [Loki·Tempo·Alloy 운영](monitoring/sayscene-observability-manual-v1.md) · [DCGM Exporter](monitoring/dcgm-exporter-install-v1.md) |
| FE 배포 | [읽는 순서](cicd/ss-fe/README.md) · [CI/CD 한 장 요약](cicd/ss-fe/ss-fe-cicd-overview-v1.md) · [실행 절차](cicd/ss-fe/ss-fe-cicd-runbook-v1.md) · [설계 근거](cicd/ss-fe/ss-fe-cicd-concepts-v1.md) |
| 이미지 CI | [AWS OIDC 결정](cicd/gha/gha-ecr-aws-auth-decision-v1.md) · [BE Jib → ECR](cicd/gha/gha-ecr-jib-push-manual-v1.md) · [STT 저장소 Buildx → ECR](cicd/gha/gha-ecr-triton-push-manual-v1.md) |
| 성능·장애 | [E2E 응답 지연 대응·슬롯 4 운영 기준](monitoring/e2e-latency-response-v1.md) · [로컬 GPU vLLM 동시성 부하테스트](loadtest/vllm-concurrency-loadtest-v1.md) · [부하 분석·알림 설계 기록](monitoring/vllm-load-analysis-v1.md) · [슬롯 1–32 누적 대화 실험·슬롯 여유 대기 검토](monitoring/vllm-loadtest-results-20261103.md) · [2026-10-27 장애 기록](issue/20261027/README.md) |

설치 매뉴얼과 실험 문서는 **작성 시점의 과정·측정 조건**을 보존한다. GPU Triton·CPU LLM·이전 인스턴스 사양이나 당시의 미검증 상태가 남아 있을 수 있으므로 현재 상태는 PRD를 먼저 읽는다. [LLM AMI 제작 기록](manual/llm-ami-v2-build-v1.md)은 이전 AWS CPU LLM 경로의 기록이다.

## 저장소별 역할

| 저장소 | 책임 |
|---|---|
| [ss-fe](https://github.com/say-scene/ss-fe) | React UI, 녹음·브라우저 TTS, 로컬 저장, 사용자 동작 추적 |
| [ss-be](https://github.com/say-scene/ss-be) | Spring Boot API, Redis 대화 상태, LLM·STT 연결, 브라우저 trace relay |
| [ss-triton](https://github.com/say-scene/ss-triton) | 이름만 유지한 CPU Whisper STT 서버·이미지 |
| [ss-k8s](https://github.com/say-scene/ss-k8s) | 앱·노드 확장·관측 스택의 Kubernetes/GitOps 설정 |
| ss-docs | 제품·아키텍처 명세와 운영 기록 |

문서 자체를 빌드하거나 배포하는 절차는 없다. GitHub의 Markdown·Mermaid 렌더링으로 읽는다. 런타임 동작은 코드·설정에서, 운영 확인 결과는 날짜와 확인 근거를 구분해 기록하며 자격 증명·webhook·개인키는 포함하지 않는다.
