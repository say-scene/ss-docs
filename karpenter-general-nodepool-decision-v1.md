# Karpenter CPU 확장 풀(`general`) 결정 기록

> 상태: **진행 중 (0단계 사전 검증 완료, 1단계 매니페스트 작성 완료 · ss-k8s push 완료, main 머지 전)** · 작성일: 2026-10-26
> 근거 문서: [english-speaking-prd-architecture-v1.md](english-speaking-prd-architecture-v1.md) §6.4·§6.5·§7, [al2023-cpu-cluster-karpenter-guide.md](al2023-cpu-cluster-karpenter-guide.md) "이후 선택: Karpenter 설치"·13~17단계

---

## 0. 요약

- kubeadm 단일 기본 노드(t3.large) 위에 Karpenter를 **Argo CD 웹 콘솔**로 설치하고, CPU 확장 풀 `general`을 추가한다.
- 기본 노드가 수용하지 못한 Pending 파드가 생기면 Karpenter가 **t3.large on-demand 워커를 최대 1대** 띄운다. 워커는 Custom AMI + userData로 `kubeadm join` 한다.
- 노드 정의(EC2NodeClass·NodePool)는 `say-scene/ss-k8s`의 `platform/karpenter/nodes/`에 두고 Argo CD Application `karpenter-nodes`로 배포한다.
- 아래 상수·조회값은 팀 리뷰로 하나씩 확정했다. 변경하려면 이 문서의 근거를 먼저 확인한다.

---

## 1. 진행 상태

| 단계 | 내용 | 상태 |
|---|---|---|
| 사전 | 가이드 0~12단계(AMI·IAM·SSM·수동 워커 조인·Argo CD) | 완료 |
| 0 | 사전 검증(태그·kubelet 설정·라벨·여유·SSM) | **완료** — §4 |
| 1 | ss-k8s 매니페스트 작성 | **작성 완료**, 브랜치 `feature/karpenter-general-nodepool` 커밋 `8ddf53a` push 완료 (main 머지 전) |
| 2 | Argo CD에 Karpenter OCI Helm 저장소 연결 (가이드 K-1) | 대기 |
| 3 | `karpenter` Application 생성·Sync (가이드 K-2, VALUES는 §6) | 대기 |
| 4 | `karpenter-nodes` Application 생성·Sync → NodeClass/NodePool `Ready=True` | 대기 |
| 5 | 확장·축소 검증 (가이드 15~17) | 대기 — replicas 재산정 필요(§7) |
| 6 | 가이드 문서 정정 (§8) | 대기 |

---

## 2. 확정 값

### 2-1. 고정 상수

| # | 항목 | 확정값 | 적용 위치 |
|---|---|---|---|
| 1 | 클러스터 이름 / discovery 태그 | `sayscene` | 서브넷·SG 태그 `karpenter.sh/discovery`, userData, SSM 경로 |
| 2 | 워커 공인 IP | `associatePublicIPAddress: true` | EC2NodeClass |
| 3 | 루트 디스크 | `20Gi` | EC2NodeClass |
| 4 | maxPods | `30` | EC2NodeClass `kubelet.maxPods`, userData `MAX_PODS`, kubelet-config |
| 5 | systemReserved | cpu `100m`, memory `256Mi` | EC2NodeClass, kubelet-config |
| 6 | kubeReserved | cpu `200m`, memory `256Mi` | EC2NodeClass, kubelet-config |
| 7 | 디스크 종류 | `gp3`, `encrypted: true`, `deleteOnTermination: true` | EC2NodeClass |
| 7-1 | IMDS | `httpTokens: required`, `httpPutResponseHopLimit: 1` | EC2NodeClass |
| 8 | 인스턴스 타입 | `t3.large` 단일 | NodePool |
| 9 | 구매 방식 | `on-demand` | NodePool |
| 10 | 풀 상한 | cpu `2`, memory `8Gi` (워커 최대 1대) | NodePool `limits` |
| 11 | 축소 정책 | `WhenEmpty`, `consolidateAfter: 1m` | NodePool `disruption` |
| 12 | 워커 라벨 | `sayscene.io/workload: general` | NodePool |
| 13 | Karpenter 버전 | `1.15.0` 고정 (controller·CRD 동일 차트) | Argo CD `karpenter` App |
| 14 | 리전 | `ap-northeast-2` | Karpenter VALUES, userData |
| 15 | controller 자원 | requests cpu `200m` / memory `512Mi`, limit memory `1Gi` | Karpenter VALUES |

### 2-2. 조회로 확정한 값

| 항목 | 값 | 확인 방법 |
|---|---|---|
| 워커 AMI | `ami-04e5c592ecea95a37` (루트 `/dev/xvda`, 원본 16GiB) | `aws ec2 describe-images --owners self` |
| 워커 인스턴스 프로파일 | `SaySceneWorkerRole` | IAM 콘솔 → 역할 → 인스턴스 프로파일 ARN |
| clusterEndpoint | `https://10.0.4.232:6443` (기본 노드 사설 IP) | `kubectl get nodes -o wide` |
| 워커 서브넷 | `subnet-0aa08e2c3de6fa4e6` (ap-northeast-2d, 10.0.0.0/20) | discovery 태그로 자동 선택 |
| 공통 노드 SG | `sg-0590bbcc59ea3fdcb` (`sayscene-sg`) | discovery 태그로 자동 선택 |
| 기본 노드 | `i-042a82bd6ddd4e20c` | `kubernetes.io/cluster/sayscene=owned` |

---

## 3. 결정 사항과 근거

### 진행 방식

| 결정 | 근거 |
|---|---|
| Karpenter·노드 정의 모두 **Argo CD 웹 콘솔**로 생성·Sync (수동 Sync) | 가이드의 운영 원칙(Application YAML 직접 작성·`kubectl apply` 지양). 같은 리소스를 Helm CLI와 Argo CD가 중복 관리하지 않게 한다. |
| 노드 정의 경로 `ss-k8s/platform/karpenter/nodes/` | 앱(`apps/`)과 클러스터 인프라를 분리. 이후 GPU 풀·KEDA도 `platform/` 아래에 추가한다. |
| NodeClass 값은 `settings.env` 없이 콘솔·AWS CLI로 하나씩 조회 | 가이드 0단계의 `settings.env`를 만들지 않았다. 생성 스크립트 대신 확정값을 직접 매니페스트에 기록해 리뷰 가능하게 했다. |
| **userData 원본은 Git**(ss-k8s의 EC2NodeClass)으로 한다 | 그동안 Notion에서 관리했으나 Karpenter가 실제로 쓰는 것은 Git 본이다. Notion을 고쳐도 워커에 반영되지 않는다. |
| userData의 `KARPENTER_MANAGED=true` | Notion 본은 11단계 수동 조인용(`false`)이었다. Karpenter 워커는 `karpenter.sh/unregistered:NoExecute` taint로 가입해야 Karpenter가 자기 노드로 인식하고 taint를 제거한다. 그 외 내용은 동일하다. |
| 퇴실 전 워커 정리 절차는 **이번 범위에서 제외** | 워커는 테스트 때만 잠깐 뜨므로 수동으로 챙긴다. 위험은 §5에 기록. |

### 상수별 근거

| # | 결정 | 근거 / 대안 |
|---|---|---|
| 1 | `sayscene` | userData·SSM(`/sayscene/kubeadm/*`)·IAM 경로와 같은 이름이어야 한다. 서브넷·SG 태그는 2026-10-26에 부착·확인. |
| 2 | 공인 IP `true` | NAT Gateway·VPC 엔드포인트가 없어 공인 IP 없이는 SSM·ECR·AWS API에 접근 못 해 가입이 실패한다. 서브넷의 `MapPublicIpOnLaunch=false`라 NodeClass에서 명시가 필수. 가입·노드 간 통신은 사설 IP를 쓴다. |
| 3 | `20Gi` | AMI 원본 16GiB 이상이어야 한다. general 풀은 일반 CPU 앱용이라 GPU 이미지를 받지 않는다. 디스크 압박 eviction이 나오면 늘린다. |
| 4 | `30` | t3.large의 VPC CNI 상한은 `3 ENI × (12−1) + 2 = 35`라 30은 안전. 기본 노드·userData·kubelet-config와 동일. **m6i/c6i.large(상한 29)로 바꾸면 30은 초과**하므로 함께 재계산해야 한다. |
| 5·6 | 100m/256Mi, 200m/256Mi | 워커는 `kubeadm join` 시 클러스터 `kubelet-config`를 받는다. NodeClass 값은 Karpenter의 용량 계산용이므로 실제 kubelet 설정과 같아야 계산이 어긋나지 않는다. t3.large allocatable ≈ 1.7 vCPU / 7.0Gi. |
| 7 | gp3, 암호화, 종료 시 삭제 | gp3는 gp2보다 저렴하고 기본 3000 IOPS. 기본 `aws/ebs` 키는 추가 비용·권한 불필요. `false`면 고아 EBS 비용 발생. |
| 7-1 | IMDSv2 필수, hop 1 | 일반 Pod가 워커 IAM 역할(SSM 가입 토큰 읽기 포함)을 쓰지 못하게 한다. hostNetwork인 aws-node·kubelet ECR pull은 영향 없음. 현재 ss-be·ss-triton은 AWS SDK를 쓰지 않는다. 기본 노드는 LBC 때문에 hop 2. |
| 8 | t3.large 단일 | 기본 노드와 같은 타입이라 테스트 결과 예측이 쉽다. 단점: CPU 크레딧(unlimited 시 추가 과금), 재고 부족(ICE) 시 대체 타입 없음 → 필요하면 `t3a.large` 추가. |
| 9 | on-demand | 첫 검증에서 가입 실패와 Spot 회수를 구분하기 위해. `interruptionQueue`가 비어 있어 Spot 회수에 대비하지 못한다. PRD도 "첫 검증 on-demand, 이후 Spot". |
| 10 | cpu 2 / 8Gi | 검증용 최소(워커 1대). 기본 노드는 상한에 포함되지 않는다. 지출 하드 캡은 아니므로 비용 상한은 AWS Budgets로 별도 관리. |
| 11 | WhenEmpty, 1m | 실행 중인 앱을 옮기지 않아 예측 가능. 워커에 뜬 Pod는 기본 노드로 자동 복귀하지 않으므로 필요 시 rollout restart. |
| 12 | `sayscene.io/workload: general` | 기본 노드와 워커가 공유하는 공통 라벨. 앱 `nodeSelector`가 이 라벨과 맞아야 Karpenter가 워커를 만든다. |
| 13 | `1.15.0` 고정 | 공식 호환성 표 기준 K8s 1.35는 Karpenter ≥ 1.9 필요, 1.15는 범위 안. 범위 지정 대신 정확한 버전으로 고정해 원치 않는 업그레이드를 막는다. |
| 14 | `ap-northeast-2` | kubeadm 환경에서는 리전 자동 감지가 없어 명시. 노드·AMI·SSM·userData와 같아야 한다. |
| 15 | cpu 200m | 차트 기본(1 / 1Gi)과 가이드(500m)보다 낮춤. 기본 노드 CPU 여유가 300m뿐이라 500m은 들어가지 않는다(§4). |

---

## 4. 0단계 사전 검증 결과 (2026-10-26)

| 항목 | 결과 |
|---|---|
| 서브넷 discovery 태그 | ✅ `subnet-0aa08e2c3de6fa4e6`, 남은 IP 4054 |
| SG discovery 태그 | ✅ `sg-0590bbcc59ea3fdcb` 1개, 기본 노드 SG와 동일 |
| 기본 EC2 소속 태그 | ✅ `i-042a82bd6ddd4e20c`, `10.0.4.232`, running |
| kubelet-config | ✅ maxPods 30, systemReserved 100m/256Mi, kubeReserved 200m/256Mi |
| 기본 노드 라벨 | ✅ `sayscene.io/workload=general`, `sayscene.io/node-role=base`, nodepool 라벨 없음 |
| 기본 노드 여유 | ⚠️ CPU allocatable 1700m 중 requests 1400m(여유 300m), memory 816Mi/7.0Gi, pods 19/30 |
| SSM endpoint | ✅ `10.0.4.232:6443` |
| 워커 AMI 루트 | ✅ `/dev/xvda`, 16GiB ≤ 20GiB |

Karpenter controller(200m) 설치 후 기본 노드 CPU 여유는 **100m**이 된다.

---

## 5. 팀 공유 · 주의 사항

- **새 앱은 `resources.requests`를 반드시 지정한다.** Karpenter는 실제 사용률이 아니라 requests로 판단한다. 기본 노드 CPU 여유가 100m뿐이라 새 파드는 대부분 워커로 가고, 워커 상한(1대)을 넘으면 Pending으로 남는다.
- 앱 배치 조건: 필수는 `nodeSelector: sayscene.io/workload: general`만 둔다. 기본 노드 선호는 `sayscene.io/node-role=base` **soft affinity**로 표현한다. control-plane 라벨을 필수로 걸면 워커에 뜨지 못한다.
- **기본 노드 stop/start와 워커 비용**: 기본 노드를 꺼도 워커 EC2는 계속 과금된다. Karpenter도 기본 노드에서 돌기 때문에 꺼진 동안 워커를 정리하지 못한다. 퇴실 전 `kubectl get nodeclaims`가 비어 있는지, 콘솔에 Karpenter 워커가 없는지 확인한다.
- **Loki·Tempo(S3 저장소)는 기본 노드에 고정한다.** 워커는 hop limit 1이라 Pod에서 IAM 자격증명을 받을 수 없다. 또한 S3 권한을 IAM 역할에 별도로 추가해야 한다(현재 미부여로 알고 있으나 미확인).
- **userData·NodeClass 수정은 Git에서** 한다. NodeClass를 바꾸면 Karpenter가 drift로 기존 워커를 교체한다.
- t3는 CPU 크레딧 unlimited 기본값이라 CPU를 오래 높게 쓰면 추가 과금된다.
- NodePool `limits`는 지출 하드 캡이 아니다. 병렬 생성 중 일시 초과할 수 있다.
- Karpenter는 kubeadm(non-EKS)을 공식 지원하지 않는다. PRD §7 통과 기준으로 직접 검증하며, D3에서 2일 이상 막히면 Cluster Autoscaler 대안을 검토한다(자동 전환 금지).

---

## 6. Karpenter Helm VALUES (확정본)

Argo CD `karpenter` Application(차트 `karpenter` `1.15.0`, 저장소 `public.ecr.aws/karpenter` OCI, namespace `kube-system`, 수동 Sync, Server-Side Apply)의 **HELM → VALUES**에 입력한다. 가이드 K-2 대비 변경점은 `controller.resources.requests.cpu`(500m → 200m)뿐이다.

```yaml
replicas: 1
hostNetwork: true
dnsPolicy: ClusterFirstWithHostNet
nodeSelector:
  kubernetes.io/os: linux
  node-role.kubernetes.io/control-plane: ""
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
  - key: CriticalAddonsOnly
    operator: Exists
topologySpreadConstraints: []
settings:
  clusterName: sayscene
  clusterEndpoint: https://10.0.4.232:6443
  eksControlPlane: false
  enableZonalShift: false
  interruptionQueue: ""
controller:
  env:
    - name: AWS_REGION
      value: ap-northeast-2
    - name: EKS_CONTROL_PLANE
      value: "false"
  resources:
    requests:
      cpu: 200m
      memory: 512Mi
    limits:
      memory: 1Gi
```

---

## 7. 남은 작업과 미결 사항

- **5단계 replicas 재산정**: 가이드 15-2의 31개는 maxPods 기준이다. 이번 구성은 CPU가 먼저 부족하다(테스트 파드 100m 기준 기본 노드 1개 + 워커 약 16개). 31개면 약 14개가 계속 Pending이므로 검증 시점에 값을 다시 정한다.
- 16단계 private ECR pull: 테스트 이미지 확보 여부 미정. 없으면 미검증으로 기록한다.
- ss-k8s 매니페스트(`feature/karpenter-general-nodepool`) main 머지. 4단계 `karpenter-nodes`의 Revision은 머지 후 `main`을 사용한다.
- Loki/Tempo용 S3 권한과 배치는 관측성 구성 때 결정.

---

## 8. 가이드와 달라진 점 (가이드 정정 대상)

| 항목 | 가이드 | 이번 결정 |
|---|---|---|
| maxPods 표기 | 일부 20, 일부 30 (60·602·1241·1313줄, 15-2 스크립트는 20) | 30으로 통일 |
| 워커 인스턴스 타입 | m6i.large / c6i.large | t3.large |
| NodePool limits | cpu 4 / 16Gi | cpu 2 / 8Gi |
| 루트 디스크 | 40Gi | 20Gi |
| controller CPU requests | 500m | 200m |
| userData 관리 | 기본 노드의 `worker-userdata.sh` | ss-k8s EC2NodeClass(Git)가 원본 |
| NodeClass 생성 | `settings.env` + 스크립트 | 확정값 직접 기록 |

---

## 9. 관련 파일

- `say-scene/ss-k8s` `platform/karpenter/nodes/ec2nodeclass-general.yaml`
- `say-scene/ss-k8s` `platform/karpenter/nodes/nodepool-general.yaml`
- [al2023-cpu-cluster-karpenter-guide.md](al2023-cpu-cluster-karpenter-guide.md)
- [al2023-vpc-cni-network-prevention-guide.md](al2023-vpc-cni-network-prevention-guide.md)
- [aws-console-iam/worker-join-read-policy.template.json](aws-console-iam/worker-join-read-policy.template.json)
