# monitoring

Prometheus · Grafana · Alertmanager 로 클러스터와 vLLM 을 관측한다.

## 읽는 순서

| | 문서 | 무엇 |
|---|---|---|
| 1 | [kube-prometheus-stack-concepts](kube-prometheus-stack-concepts-v1.md) | 무엇을 왜 이렇게 구성했나 |
| 2 | [kube-prometheus-stack-install](kube-prometheus-stack-install-v1.md) | 설치 절차 |
| 3 | [dcgm-exporter-install](dcgm-exporter-install-v1.md) | GPU 지표 수집 추가 |
| 4 | [sayscene-observability-manual](sayscene-observability-manual-v1.md) | 운영 매뉴얼 |
| 5 | [vllm-load-analysis](vllm-load-analysis-v1.md) | vLLM 부하 측정·분석과 알람 설계 |

## 관련

- [loadtest/vllm-concurrency-loadtest-v1.md](../loadtest/vllm-concurrency-loadtest-v1.md) — 5번의 짝. 부하를 거는 방법
