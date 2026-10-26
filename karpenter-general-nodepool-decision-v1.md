# Karpenter CPU 확장 풀(`general`) 결정 기록

> 상태: **진행 중 (0단계 사전 검증 완료, 노드 정의 main 머지 완료, controller 렌더링 완료)** · 작성일: 2026-10-26
> 근거 문서: [english-speaking-prd-architecture-v1.md](english-speaking-prd-architecture-v1.md) §6.4·§6.5·§7, [al2023-cpu-cluster-karpenter-guide.md](al2023-cpu-cluster-karpenter-guide.md) "이후 선택: Karpenter 설치"·13~17단계

---

## 0. 요약

- kubeadm 단일 기본 노드(t3.large) 위에 Karpenter를 설치하고, CPU 확장 풀 `general`을 추가한다. Karpenter 차트는 **로컬에서 `helm template`으로 렌더링한 YAML을 ss-k8s에 커밋**하고 Argo CD가 Git에서 배포한다(`platform/karpenter/controller/`).
- 기본 노드가 수용하지 못한 Pending 파드가 생기면 Karpenter가 **t3.large on-demand 워커를 최대 1대** 띄운다. 워커는 Custom AMI + userData로 `kubeadm join` 한다.
- 노드 정의(EC2NodeClass·NodePool)는 `say-scene/ss-k8s`의 `platform/karpenter/nodes/`에 두고 Argo CD Application `karpenter-nodes`로 배포한다.
- 아래 상수·조회값은 팀 리뷰로 하나씩 확정했다. 변경하려면 이 문서의 근거를 먼저 확인한다.

---

## 1. 진행 상태

| 단계 | 내용 | 상태 |
|---|---|---|
| 사전 | 가이드 0~12단계(AMI·IAM·SSM·수동 워커 조인·Argo CD) | 완료 |
| 0 | 사전 검증(태그·kubelet 설정·라벨·여유·SSM) | **완료** — §4 |
| 1 | ss-k8s 노드 정의 작성 (`platform/karpenter/nodes/`) | **완료**, main 머지 (`ecfec9d`) |
| 2 | ~~Argo CD에 Karpenter OCI Helm 저장소 연결 (가이드 K-1)~~ → controller 렌더링·커밋 (`platform/karpenter/controller/`) | **렌더링 완료** — §6 |
| 3 | `karpenter` Application 생성·Sync (Git 경로 `platform/karpenter/controller`, Server-Side Apply) → CRD 5개 `Established` | 대기 |
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
| Karpenter 차트를 **로컬 `helm template`으로 렌더링해 ss-k8s에 커밋**하고, Argo CD는 Git 경로를 배포 (가이드의 웹 콘솔 Helm 소스 방식 철회) | 웹 콘솔 VALUES는 Argo CD Application 객체에만 남아 리뷰·이력·재현이 안 된다. 렌더링본을 커밋하면 설정이 Git에 남고, 적용될 매니페스트를 Git diff로 확인할 수 있으며, `apps/`와 같은 평문 YAML 방식이 된다. Argo CD에 OCI Helm 저장소 연결도 필요 없다. |
| 사람이 values 변경 시에만 `render.sh`를 **수동 실행**, GitHub Actions 미사용 | 설정 변경 빈도가 낮아 자동 커밋 봇·충돌 처리나 CI 검사는 과하다. 렌더링 누락은 `values.yaml` 상단 주석으로 상기한다. |
| 렌더링 결과(`karpenter.yaml`)는 **직접 수정 금지** | 다시 렌더링하면 수동 수정이 사라진다. 설정은 항상 `values.yaml`에서 바꾼다. |
| Application 생성·Sync는 Argo CD **웹 콘솔**, 수동 Sync | Application YAML을 따로 관리하지 않는 기존 운영 방식 유지. 같은 리소스를 Helm CLI와 Argo CD가 중복 관리하지 않는다. |
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
- **Karpenter 설정 변경은 `values.yaml` 수정 → `./render.sh` → 두 파일 함께 커밋** 순서로 한다. `render.sh`만 잊으면 커밋은 됐는데 클러스터에는 반영되지 않는다.
- t3는 CPU 크레딧 unlimited 기본값이라 CPU를 오래 높게 쓰면 추가 과금된다.
- NodePool `limits`는 지출 하드 캡이 아니다. 병렬 생성 중 일시 초과할 수 있다.
- Karpenter는 kubeadm(non-EKS)을 공식 지원하지 않는다. PRD §7 통과 기준으로 직접 검증하며, D3에서 2일 이상 막히면 Cluster Autoscaler 대안을 검토한다(자동 전환 금지).

---

## 6. Karpenter controller 렌더링

ss-k8s `platform/karpenter/controller/`:

| 파일 | 역할 |
|---|---|
| `values.yaml` | 사람이 고치는 유일한 파일. 가이드 K-2 VALUES 대비 변경점은 `controller.resources.requests.cpu`(500m → 200m)뿐 |
| `render.sh` | `helm template karpenter oci://public.ecr.aws/karpenter/karpenter --version 1.15.0 --namespace kube-system --kube-version 1.35.9 --include-crds -f values.yaml` |
| `karpenter.yaml` | 렌더링 산출물(2,815줄). 직접 수정 금지 |
| `kustomization.yaml` | `resources: [karpenter.yaml]`만 지정. Argo CD 디렉터리 소스는 경로의 `.yaml`을 모두 적용하므로, 이 파일이 없으면 `kind`가 없는 `values.yaml`까지 매니페스트로 읽혀 Sync가 실패한다. `apps/`와 같은 kustomize 방식 |

렌더링 검증 결과(2026-10-26, 로컬 helm v4.3.1):

| 항목 | 결과 |
|---|---|
| 리소스 | CRD 5, Deployment, Service, ServiceAccount, PDB, ClusterRole 3, ClusterRoleBinding 2, Role 2, RoleBinding 2. Secret 없음 |
| CRD | `nodepools.karpenter.sh`, `nodeclaims.karpenter.sh`, `ec2nodeclasses.karpenter.k8s.aws`, `nodeoverlays.karpenter.sh`, `capacitybuffers.autoscaling.x-k8s.io` — 뒤의 둘은 1.15 차트 포함분으로 이번 구성에서 사용하지 않는다 |
| namespace | namespace 리소스는 모두 `kube-system` |
| 이미지 | `public.ecr.aws/karpenter/controller:1.15.0@sha256:ab0f953e…` (digest 고정) |
| 설정 반영 | requests cpu 200m / memory 512Mi, limit memory 1Gi, `CLUSTER_ENDPOINT=https://10.0.4.232:6443`, `EKS_CONTROL_PLANE=false` 1회, control-plane nodeSelector, `hostNetwork: true` |
| 재현성 | 재렌더링 결과가 바이트 단위로 동일 |
| kustomize | `kubectl kustomize` 결과도 같은 18개 리소스, CRD가 맨 앞 |

팀원마다 helm 버전이 다르면 출력이 미세하게 달라질 수 있다. 재렌더링 diff가 values 변경과 무관하게 크면 helm 버전부터 확인한다.

Argo CD `karpenter` Application 입력값: Path `platform/karpenter/controller`, Destination namespace `kube-system`, 수동 Sync, **Server-Side Apply**(큰 CRD의 annotation 크기 제한 회피), Prune·Force 미선택. 통과 기준은 controller Healthy, 로그에 AWS 권한 오류 없음, **CRD 5개 `Established=True`**.

---

## 7. 남은 작업과 미결 사항

- **5단계 replicas 재산정**: 가이드 15-2의 31개는 maxPods 기준이다. 이번 구성은 CPU가 먼저 부족하다(테스트 파드 100m 기준 기본 노드 1개 + 워커 약 16개). 31개면 약 14개가 계속 Pending이므로 검증 시점에 값을 다시 정한다.
- 16단계 private ECR pull: 테스트 이미지 확보 여부 미정. 없으면 미검증으로 기록한다.
- ss-k8s controller 렌더링본(`feature/karpenter-controller-render`) main 머지. 3·4단계 Application의 Revision은 `main`을 사용한다.
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
| Karpenter 설치 | 웹 콘솔에서 OCI Helm 저장소 연결(K-1) + Helm 소스 Application·VALUES 입력(K-2) | 로컬 `render.sh` 렌더링본을 ss-k8s에 커밋, Application은 Git 경로 지정 |
| 설치 후 확인할 CRD | 3개 | 5개 (`nodeoverlays`, `capacitybuffers` 추가) |

---

## 9. 관련 파일

- `say-scene/ss-k8s` `platform/karpenter/nodes/ec2nodeclass-general.yaml`
- `say-scene/ss-k8s` `platform/karpenter/nodes/nodepool-general.yaml`
- `say-scene/ss-k8s` `platform/karpenter/controller/` (`values.yaml`, `render.sh`, `karpenter.yaml`, `kustomization.yaml`)
- [al2023-cpu-cluster-karpenter-guide.md](al2023-cpu-cluster-karpenter-guide.md)
- [al2023-vpc-cni-network-prevention-guide.md](al2023-vpc-cni-network-prevention-guide.md)
- [aws-console-iam/worker-join-read-policy.template.json](aws-console-iam/worker-join-read-policy.template.json)
