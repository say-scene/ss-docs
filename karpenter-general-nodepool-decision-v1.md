# Karpenter CPU 확장 풀(`general`) 결정 기록

> 상태: **진행 중 (설치·확장·축소 검증 완료, 가이드 정정 대기)** · 작성일: 2026-10-26 · 갱신: 2026-10-27 (트러블슈팅 §9 추가)
> 근거 문서: [english-speaking-prd-architecture-v1.md](english-speaking-prd-architecture-v1.md) §6.4·§6.5·§7, [al2023-cpu-cluster-karpenter-guide.md](al2023-cpu-cluster-karpenter-guide.md) "이후 선택: Karpenter 설치"·13~17단계

---

## 0. 요약

- kubeadm 단일 기본 노드(t3.large) 위에 Karpenter를 설치하고, CPU 확장 풀 `general`을 추가한다. Karpenter 차트는 **로컬에서 `helm template`으로 렌더링한 YAML을 ss-k8s에 커밋**하고 Argo CD가 Git에서 배포한다(`platform/karpenter/controller/`).
- 기본 노드가 수용하지 못한 Pending 파드가 생기면 Karpenter가 **t3.large on-demand 워커를 최대 1대** 띄운다. 워커는 Custom AMI + userData로 `kubeadm join` 한다.
- 노드 정의(EC2NodeClass·NodePool)는 `say-scene/ss-k8s`의 `platform/karpenter/nodes/`에 두고, `controller/kustomization.yaml`이 `../nodes`를 참조해 **Argo CD Application `karpenter` 하나**로 본체와 함께 배포한다.
- 아래 상수·조회값은 팀 리뷰로 하나씩 확정했다. 변경하려면 이 문서의 근거를 먼저 확인한다.

---

## 1. 진행 상태

| 단계 | 내용 | 상태 |
|---|---|---|
| 사전 | 가이드 0~12단계(AMI·IAM·SSM·수동 워커 조인·Argo CD) | 완료 |
| 0 | 사전 검증(태그·kubelet 설정·라벨·여유·SSM) | **완료** — §4 |
| 1 | ss-k8s 노드 정의 작성 (`platform/karpenter/nodes/`) | **완료**, main 머지 (`ecfec9d`) |
| 2 | ~~Argo CD에 Karpenter OCI Helm 저장소 연결 (가이드 K-1)~~ → controller 렌더링·커밋 (`platform/karpenter/controller/`) | **렌더링 완료** — §6 |
| 3 | `karpenter` Application 생성·Sync (Git 경로 `platform/karpenter/controller`, Server-Side Apply) → CRD 5개 `Established` | **완료** |
| 4 | ~~`karpenter-nodes` Application 별도 생성~~ → `karpenter` Application이 `../nodes`까지 적용 (ss-k8s `9de74cb`) | **완료** — NodePool 미적용 장애 후 변경, §9-1 |
| 5 | 확장·축소 검증 (가이드 15~17) | **완료** — ss-be 롤아웃으로 워커 생성·가입·Running, replicas 0으로 워커 자동 종료. 노드 간 DNS 장애 해결 후 통과, §9-2 |
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
| 본체와 노드 정의를 **`karpenter` Application 하나**로 적용 (`../nodes` kustomize 참조) | 파일 이동 없이 폴더 구분(렌더링 산출물 vs 사람이 고치는 파일)을 유지하면서 Application 하나로 관리. 대가: 앱 삭제 시 NodePool도 삭제되고, 노드 설정만 바꿔도 앱 전체가 Sync 대상 (§5). |
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
- **노드 공통 SG(`sayscene-sg`)의 자기 참조 인바운드는 반드시 "모든 트래픽"이어야 한다.** CoreDNS가 기본 노드에만 있어 워커 파드의 DNS 질의(UDP 53)는 항상 노드를 넘는다. TCP만 허용하면 TCP 통신은 되는데 이름 해석만 실패해 원인을 찾기 어렵다(§9-2). SG를 새로 만들거나 규칙을 정리할 때 유지한다.
- **`karpenter` Application을 삭제·재생성하면 NodePool도 삭제**되어 Karpenter가 워커를 모두 정리한다. 노드 설정만 바꿔도 2,800줄짜리 앱 전체가 DIFF·Sync 대상이니 DIFF에서 노드 리소스 변경만 있는지 확인한다.
- 노드 정의를 볼 Application이 없으면 Karpenter는 `no dynamic nodepools found`만 남기고 워커를 만들지 않는다. 확장이 안 되면 먼저 `kubectl get nodepools,ec2nodeclasses`의 존재·`READY`를 본다(§9-1).
- 검증 중 `kubectl scale` 등으로 Argo CD 관리 리소스를 직접 바꾸면 Argo CD에서 OutOfSync로 보이고 다음 Sync 때 Git 값으로 되돌아간다. 테스트 후 Git 기준으로 원복을 확인한다.
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

Argo CD `karpenter` Application 입력값: Path `platform/karpenter/controller`, Destination namespace `kube-system`, 수동 Sync, **Server-Side Apply**(큰 CRD의 annotation 크기 제한 회피), Prune·Force 미선택. 통과 기준은 controller Healthy, 로그에 AWS 권한 오류 없음, **CRD 5개 `Established=True`**, `../nodes` 참조로 함께 적용되는 **EC2NodeClass·NodePool `general` `READY=True`**. kustomize 빌드 결과는 렌더링 18개 + 노드 정의 2개 = 20개 리소스다.

---

## 7. 남은 작업과 미결 사항

- 가이드 15-2 `cpu-test`(replicas 31)는 실행하지 않고 ss-be 롤아웃으로 확장을 검증했다. 다시 수행한다면 CPU가 먼저 부족하므로(테스트 파드 100m 기준 기본 노드 1개 + 워커 약 16개) replicas를 재산정한다.
- 워커 AMI(`ami-04e5c592ecea95a37`)에 networkd 재발 방지 설정(`/etc/systemd/networkd.conf.d/`)이 포함됐는지 확인. 없다면 워커에 보조 ENI가 붙을 때 DNS 장애가 재발할 수 있다.
- 매일 stop/start 후 워커가 남아 있지 않은지 확인하는 습관(§5). 자동화는 범위 밖.
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
| Application 구성 | `karpenter`(Helm) + `karpenter-nodes`(Git) 2개 | `karpenter` 1개가 `../nodes`까지 적용 |
| SG 자기 참조 | 1-1에 allow-all 기준 명시 | 실제 설정이 TCP만이었음 → 모든 트래픽으로 수정. 가이드에 "UDP(DNS) 포함" 이유와 검증 명령을 보강 |

---

## 9. 트러블슈팅 기록 (2026-10-26)

### 9-1. `no dynamic nodepools found` — 워커가 생성되지 않음

**증상:** ss-be 롤아웃 중 새 파드가 CPU 부족으로 Pending이 됐으나 워커가 생기지 않았다. Karpenter 로그에 `no dynamic nodepools found`.

**진단:**

| 확인 | 결과 |
|---|---|
| `kubectl get nodepools,ec2nodeclasses` | `No resources found` (CRD는 있음 — CRD가 없으면 `the server doesn't have a resource type` 오류) |
| `kubectl -n argocd get application karpenter-nodes` | `NotFound`. Application은 `karpenter` 하나만 있고 경로는 `platform/karpenter/controller` |

Karpenter 소스(`provisioner.go`) 기준으로 이 로그는 NodePool 목록에서 static·**Ready 아님**·삭제 중인 것을 제외한 뒤 **남은 NodePool이 없을 때** 출력된다.

**원인:** `platform/karpenter/nodes/`를 보는 Application이 없어 NodePool·EC2NodeClass가 클러스터에 적용되지 않았다. Application 이름은 무관하며, 문제는 **어느 경로를 보느냐**였다.

**해결 (ss-k8s `9de74cb`):** Application을 추가하는 대신 기존 `karpenter` Application 하나로 노드 정의까지 적용하도록 바꿨다.

- `nodes/kustomization.yaml` 추가, `controller/kustomization.yaml`의 resources에 `../nodes` 추가.
- kustomize는 보안 제한으로 루트 밖의 **파일**(`../nodes/nodepool-general.yaml`)을 직접 참조할 수 없다. **kustomization이 있는 디렉터리**(`../nodes`)는 참조할 수 있다.
- 새 클러스터 첫 Sync에서 CRD보다 NodePool dry-run이 먼저 실패하지 않도록 EC2NodeClass·NodePool에 `argocd.argoproj.io/sync-options: SkipDryRunOnMissingResource=true`.

### 9-2. 워커의 ss-be가 `PodInitializing`에서 멈춤 — 노드 간 DNS(UDP) 차단

**증상:** 워커 `ip-10-0-9-11`에 뜬 ss-be 파드가 `Init:0/1`(PodInitializing)에서 진행되지 않았다. 이미지 pull은 완료.

**진단:**

| 확인 | 결과 | 해석 |
|---|---|---|
| init container `wait-for-redis` 로그 | `Redis 연결 대기 중...` / `Terminated` 반복 | `timeout 3`이 `redis-cli`를 종료 → 연결 거부가 아니라 **응답 없음** |
| 워커의 `aws-node`·`kube-proxy` | Running | 워커 CNI 미준비는 아님 |
| 워커 테스트 파드: DNS `10.96.0.10` (UDP 53) | ❌ timeout | 이름 해석 불가 |
| 워커 테스트 파드: Redis 파드 IP `10.0.9.180:6379` (TCP) | ✅ open | Redis·노드 간 TCP 정상 (Redis는 기본 노드) |
| 워커 테스트 파드: 기본 노드 `10.0.4.232:10250` (TCP) | ✅ open | 노드 간 TCP 정상 |
| CoreDNS 위치 | 2개 모두 기본 노드 | 워커의 DNS 질의는 반드시 노드를 넘음 |
| `sayscene-sg` 자기 참조 인바운드 | `tcp 0-65535`만 존재 | **UDP·ICMP 미허용** |

**원인:** 노드 공통 SG의 자기 참조 규칙이 TCP만 허용해, 워커 파드 → 기본 노드 CoreDNS의 UDP 53 질의가 차단됐다. init container는 서비스 이름(`ss-redis`)으로 접속하므로 DNS 단계에서 막혔다. 단일 노드일 때는 모든 DNS 질의가 노드 안에서 끝나 SG를 거치지 않았기 때문에 드러나지 않았다. 가이드 1-1의 "자기 참조 allow-all" 기준과 실제 설정이 달랐다.

**해결:** `sayscene-sg` 인바운드에 **유형 모든 트래픽, 소스 `sg-0590bbcc59ea3fdcb`(자기 자신)** 규칙 추가. 재시작 없이 즉시 적용됐다.

**검증 결과:**

- ss-be 파드가 워커에서 Running. ECR private 이미지(ss-be)를 워커 kubelet이 imagePullSecrets 없이 pull해 실행 → 가이드 16단계(private ECR pull) 충족.
- ss-be replicas를 0으로 줄이자 워커 노드·NodeClaim이 자동 종료됐다(`WhenEmpty` consolidation) → 가이드 17단계 축소 동작 확인.

**배제한 가설:** AL2023 networkd의 VPC CNI 규칙 삭제 문제([재발 방지 가이드](al2023-vpc-cni-network-prevention-guide.md)). 노드 간 파드 TCP가 정상이라 라우팅 문제는 아니었다. 다만 워커 AMI(`ami-04e5c592ecea95a37`)에 networkd 재발 방지 설정이 들어 있는지는 **확인하지 않았다**(§7).

---

## 10. 관련 파일

- `say-scene/ss-k8s` `platform/karpenter/nodes/ec2nodeclass-general.yaml`
- `say-scene/ss-k8s` `platform/karpenter/nodes/nodepool-general.yaml`
- `say-scene/ss-k8s` `platform/karpenter/nodes/kustomization.yaml`
- `say-scene/ss-k8s` `platform/karpenter/controller/` (`values.yaml`, `render.sh`, `karpenter.yaml`, `kustomization.yaml`)
- [al2023-cpu-cluster-karpenter-guide.md](al2023-cpu-cluster-karpenter-guide.md)
- [al2023-vpc-cni-network-prevention-guide.md](al2023-vpc-cni-network-prevention-guide.md)
- [aws-console-iam/worker-join-read-policy.template.json](aws-console-iam/worker-join-read-policy.template.json)
