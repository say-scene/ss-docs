# AL2023 Custom AMI 이후 CPU 클러스터 실행 가이드 — AWS 콘솔 중심

선행 문서: [CPU AMI 설치·확인](./al2023-cpu-ami-setup-guide.md)

**목표:** 최초 EC2 한 대가 컨트롤 플레인과 일반 CPU 워커를 겸한다. 이 노드의 배치 가능 용량을 초과하는 파드가 생기면 Karpenter가 Custom AMI로 워커를 추가하고, 빈 워커는 정리한다. EKS가 아닌 kubeadm 구성이다.

**현재 네트워크:** private subnet 없이 기존 public subnet을 사용한다. 기본 노드에는 EIP가 연결되어 있고 확장 워커도 이 public subnet에 생성한다. **public subnet에서도 모든 노드는 사설 IP를 가지며, 노드 간 통신에는 사설 IP를 사용한다.** 워커의 공인 IPv4는 NAT 없이 외부 AWS API·이미지 저장소에 접근하는 데 사용한다.

AMI가 `available`이고, 최초 EC2에서 아직 `kubeadm init`을 하지 않은 상태부터 시작한다. 이미 클러스터를 만들었다면 완료한 단계를 확인하고 이어서 진행한다. 기존 클러스터에 `init/reset`을 다시 실행하지 않는다.

## 실행 위치·구조·버전

필요한 이유: AWS 자원을 만드는 작업과 Kubernetes를 설정하는 작업은 실행 위치가 다르다. 기본 노드가 관리 역할과 앱 실행 역할을 함께 맡는다는 기준으로 위치를 통일한다.

이 문서에는 별도의 “관리 터미널”이나 추가 관리용 EC2가 없다.

| 표기 | 실제 위치·담당 작업 |
|---|---|
| **AWS 콘솔** | 브라우저에서 AWS에 로그인. AMI·VPC·SG·태그 확인, IAM 역할·정책, SSM 파라미터 생성, EC2 상태 확인 |
| **기본 노드 셸** | 최초 AL2023 EC2에 SSH 또는 Session Manager로 접속한 터미널. kubeadm·kubectl·Helm 실행 및 YAML·userData 파일 작성 |
| **확장 워커 내부** | Karpenter가 만든 새 EC2. cloud-init이 userData를 자동 실행. 문제 진단 때만 직접 접속 |

AWS CLI는 기본 노드에서 **현재 역할 신원을 확인**할 때와 워커 userData가 **SSM 값을 자동 조회**할 때만 사용한다. IAM·태그·파라미터 생성은 콘솔에서 처리한다. Karpenter의 EC2 생성·종료는 설치된 controller가 자동 수행하므로 사람이 콘솔에서 매번 인스턴스를 만들 필요는 없다.

기본 노드의 **컨트롤 플레인**은 API 서버·etcd·scheduler 등 클러스터를 관리하는 역할이다. 같은 EC2의 kubelet이 일반 앱 파드도 실행하므로 이 노드는 **워커 역할도 겸한다**. 아래에서는 이 EC2를 “기본 노드”, 변수에서는 `CP_...`로 표기한다.

```mermaid
flowchart LR
  P["앱 파드 배치 요청"] --> B["기본 EC2: 컨트롤 플레인 + 일반 CPU 앱"]
  B -->|"남은 용량에 배치 불가 → Pending"| K["기본 EC2의 Karpenter"]
  K --> W["Custom AMI로 확장 워커 생성"]
  W --> J["SSM 조회 → kubeadm join → 앱 실행"]
```

- CCM·Karpenter는 기본 노드에 고정한다. 기본 노드는 Karpenter 관리 대상에 넣지 않아 워커가 0대여도 유지된다.
- 일반 CPU 앱은 기본 노드와 확장 워커에 공통 라벨을 사용한다. 기본 노드를 선호하는 배치는 15단계에서 다룬다.
- **Karpenter는 실제 CPU 사용률이 100%가 됐다는 이유만으로 확장하지 않는다.** 스케줄러가 파드의 `resources.requests`·노드의 allocatable·배치 조건을 보고 배치 불가로 판단한 파드가 있어야 확장한다. 부하에 따라 파드 수를 늘리는 기능은 HPA/KEDA로 별도 구성한다.
- 기본 노드가 앱도 실행하므로 앱의 requests/limits를 지정하고 관리 프로세스에 여유를 둔다. 이 예시는 kubelet에 시스템·Kubernetes 프로세스용 예약량도 설정한다.

| 통신 | 사용할 주소 |
|---|---|
| 외부 PC의 kubectl → 기본 노드 API | 기본 노드 EIP:6443. SG 소스는 관리자 공인 IP/32 |
| 확장 워커의 join·kubelet → 기본 노드 API | 기본 노드 **사설 IP:6443** |
| Karpenter·CNI → 기본 노드 API | 기본 노드 **사설 IP:6443** |
| 기본 노드 ↔ 워커, Pod 간 통신 | VPC **사설 IP** |
| 각 노드·일반 Pod → 인터넷의 AWS API·이미지 저장소 | 기본 노드는 EIP, 워커는 자동 할당 공인 IPv4를 통한 외부 통신 |

| 구성 | 기준 |
|---|---|
| OS·아키텍처 | Amazon Linux 2023, x86_64 |
| Kubernetes | 선행 AMI에 설치한 1.35.x 그대로 |
| AWS CCM | v1.35.2, 소스 커밋·이미지 tag 고정 |
| VPC CNI | v1.21.1 소스 커밋 고정 |
| Karpenter controller·CRD | 1.15.0으로 일치 |
| Helm | v3.19.0 설치 예시 |
| 확장 워커 | m6i.large/c6i.large, on-demand |
| 첫 테스트 상한 | 확장 풀 전체 CPU 4개·메모리 16Gi, 노드당 maxPods 20 |

최신 버전이라는 뜻은 아니다. CNI 해당 릴리스의 README 호환성 표에는 1.35 인증이 명시되어 있지 않아 실제 PoC 검증이 필요하다. 다른 Kubernetes minor라면 CCM·provider·Karpenter 호환성을 다시 맞춘다.

## 0. 콘솔에서 값을 확인하고 기본 노드에 기록한다

필요한 이유: 클러스터 이름은 태그·SSM 경로·IAM 조건·Karpenter 설정을 연결한다. 같은 값을 기록해야 콘솔과 노드 설정이 서로 다른 자원을 가리키지 않는다.

먼저 콘솔 오른쪽 위에서 사용할 리전과 계정 ID를 확인한다. 이 문서의 예시는 서울 `ap-northeast-2`, 클러스터 이름 `sayscene`이다. 다른 이름이면 이후의 태그 키·값과 SSM 경로까지 함께 바꾼다.

| 기록할 값 | 콘솔 확인 위치 |
|---|---|
| 기본 EC2 ID | EC2 → 인스턴스 → 최초 EC2 → 인스턴스 ID |
| 기본 EC2 EIP | EC2 → 인스턴스 → 최초 EC2 → 네트워킹 / 탄력적 IP 주소 |
| CPU AMI ID | EC2 → AMI → 내 소유 → 생성한 AMI |
| 워커 서브넷 ID·VPC CIDR | VPC → 서브넷 / VPC |
| 공통 노드 SG ID | EC2 → 보안 그룹 |
| AMI 루트 장치·원본 크기 | EC2 → AMI → 해당 이미지 → 세부 정보·스토리지/블록 디바이스 매핑. 예: `/dev/xvda`, 30GiB |
| 워커 인스턴스 프로파일 이름 | 2단계에서 생성·확인. IAM 콘솔 EC2 역할 신규 생성 시 역할과 같은 이름 |

**기본 노드 셸**에서 작업 폴더와 설정 파일을 만든다. `REPLACE` 항목을 실제 값으로 수정한다.

```bash
mkdir -p "$HOME/sayscene-bootstrap"
cd "$HOME/sayscene-bootstrap"

cat > settings.env <<'EOF'
export AWS_REGION="ap-northeast-2"
export CLUSTER_NAME="sayscene"
export CONTROL_PLANE_INSTANCE_ID="i-REPLACE_BASE_NODE"
export CP_EIP="REPLACE_BASE_NODE_EIP"
export CPU_AMI_ID="ami-REPLACE_CPU_AMI"
export SUBNET_ID="subnet-REPLACE_WORKER_SUBNET"
export NODE_SG_ID="sg-REPLACE_SHARED_NODE_SG"
export CP_ROLE="SaySceneControlPlaneRole"
export WORKER_ROLE="SaySceneWorkerRole"
export WORKER_PROFILE="SaySceneWorkerRole"
export ROOT_DEVICE="REPLACE_AMI_ROOT_DEVICE"
export ROOT_VOLUME_GIB="40"
export SERVICE_CIDR="10.96.0.0/12"
export MAX_PODS="20"
export WORKER_PUBLIC_IP="true"
export HELM_VERSION="v3.19.0"
export KARPENTER_VERSION="1.15.0"
export CCM_COMMIT="1c494fbc3edacf55e2a1e06df21858c03ff9c457"
export CNI_COMMIT="e082bae7c03ddc28357b01595f3746b117ac08c7"
export AWS_PAGER=""
EOF

if grep -q REPLACE settings.env; then
  echo "실제 자원 값을 먼저 입력하세요."
else
  . ./settings.env
fi
```

`ROOT_VOLUME_GIB`는 최소 원본 AMI 루트 크기 이상으로 정한다. 40GiB는 예시다.

현재 구성에서는 `WORKER_PUBLIC_IP=true`를 유지한다. 별도 NAT·엔드포인트 경로가 없으므로 워커는 자동 할당 공인 IPv4로 외부 서비스에 접근한다. 워커마다 EIP를 만들 필요는 없다. **이 설정이 워커의 join·노드 간 통신에 공인 IP를 사용한다는 뜻은 아니다.** `SERVICE_CIDR`은 VPC의 기본·추가 CIDR 및 연결 네트워크의 대역과 겹치면 안 된다.

새 셸을 열 때마다 같은 **기본 노드**에서 실행한다. 파일을 다른 관리 PC로 복사할 필요는 없다.

```bash
cd "$HOME/sayscene-bootstrap"
. ./settings.env
if [[ -f cluster-runtime.env ]]; then
  . ./cluster-runtime.env
fi
```

## 1. 콘솔에서 네트워크와 태그를 준비한다

필요한 이유: 태그는 Karpenter가 사용할 자원을 고르고 CCM이 클러스터를 식별하게 한다. 실제 통신 허용은 별도로 SG·라우팅·DNS에서 설정해야 한다.

### 1-1. AMI·VPC·보안 그룹 확인

**AWS 콘솔**에서 다음 순서로 확인한다.

1. **EC2 → AMI → 내 소유**: CPU AMI가 `available`, 아키텍처 `x86_64`인지 확인한다. init 이전의 AMI를 선택한다.
2. **VPC → 서브넷**: 기본 노드가 있는 기존 public subnet을 워커 후보로 사용한다. VPC·AZ·사용 가능한 IPv4 수를 확인한다. 기본 노드·서브넷·노드 SG가 같은 VPC여야 한다.
3. **VPC → VPC → 해당 VPC → 작업 → VPC 설정 편집**: DNS 확인과 DNS 호스트 이름을 활성화한다. VPC의 모든 IPv4 CIDR과 `SERVICE_CIDR`이 겹치지 않는지 확인한다.
4. **EC2 → 인스턴스 → 기본 노드 → 작업 → 보안 → 보안 그룹 변경**: 기존 필요한 SG를 유지하면서 공통 노드 SG를 연결한다. 기존 SG를 공통 노드 SG로 재사용하면 추가 SG를 만들거나 중복 연결할 필요가 없다.
5. **EC2 → 보안 그룹 → 공통 노드 SG → 인바운드 규칙 편집**: 유형 **모든 트래픽**, 소스 **동일 SG의 ID**로 자기 참조 규칙을 추가한다. 이미 있으면 추가하지 않는다.
6. SSH는 자신의 접속 IP 범위를 유지한다. 외부 kubectl용으로 **사용자 지정 TCP / 6443 / 내 공인 IP/32** 인바운드를 추가한다. `0.0.0.0/0`으로 전체 공개하지 않는다. 자기 참조 규칙은 별도로 유지하여 기본 노드↔워커의 사설 IP 통신을 허용한다.
7. 서브넷의 연결된 라우팅 테이블·인터넷 경로·SG 아웃바운드·NACL을 확인한다. AWS API·ECR·`registry.k8s.io`·`public.ecr.aws`·GitHub 등에 접근할 수 있어야 한다.

**VPC → 서브넷 → 해당 public subnet → 라우팅 테이블**에서 `0.0.0.0/0 → 인터넷 게이트웨이(igw-...)`를 확인한다. 기본 노드는 EIP, 워커는 NodeClass의 `associatePublicIPAddress: true`로 공인 IPv4를 사용한다. public subnet에 있다는 사실만으로 공인 IP 없는 워커의 인터넷 연결이 생기지는 않는다.

현재 경로에는 NAT Gateway가 필요하지 않다. 일반 Pod는 사설 IP를 사용하며 VPC CNI의 기본 SNAT와 노드의 공인 IPv4/EIP를 통해 인터넷으로 나간다. `AWS_VPC_K8S_CNI_EXTERNALSNAT=true`로 변경하지 않는다. VPC CNI 파드가 서브넷 IP를 소비하므로 EC2 대수보다 많은 IP 여유가 필요하다.

VPC CIDR은 실제 노드·Pod 주소 범위이고 Service CIDR은 ClusterIP용 가상 주소 범위다. 예를 들어 VPC `10.0.0.0/16`과 Service `10.96.0.0/12`는 겹치지 않는다. VPC에 `10.96.0.0/16`이 추가 연결되어 있다면 Service 범위와 겹치므로 다른 Service 대역을 선택해야 한다.

### 1-2. 직접 붙일 AWS 태그 — 현재 단계에서는 세 대상

아래 예시는 `CLUSTER_NAME=sayscene`이다. 키와 값은 대소문자까지 정확히 입력한다.

| 대상 | 콘솔 경로 | 태그 키 | 태그 값 | 붙이는 이유 |
|---|---|---|---|---|
| 워커 후보 서브넷 | VPC → 서브넷 → 대상 → 태그 → 태그 관리 | `karpenter.sh/discovery` | `sayscene` | NodeClass의 subnetSelectorTerms가 이 값으로 후보를 검색 |
| 공통 노드 SG | EC2 → 보안 그룹 → 대상 → 태그 → 태그 관리 | `karpenter.sh/discovery` | `sayscene` | NodeClass의 securityGroupSelectorTerms가 이 SG를 검색 |
| 기본 EC2 | EC2 → 인스턴스 → 최초 EC2 → 태그 → 태그 관리 | `kubernetes.io/cluster/sayscene` | `owned` | AWS CCM이 이 EC2의 클러스터 소속을 식별 |

기존 태그는 유지하고 **새 태그 추가 → 키·값 입력 → 저장**한다. 확인할 때는 각 자원의 태그 탭을 다시 연다. `Name` 태그는 사람이 보는 이름이며, 이번 자동 확장의 필수 입력은 아니다.

`karpenter.sh/discovery`는 **이 가이드가 선택한 검색 규칙**이다. 태그를 붙이는 것만으로 통신이 열리거나 노드가 가입하지 않는다. 같은 discovery 값을 불필요한 서브넷·SG에 붙이지 않는다. 일치하는 SG가 여러 개면 모두 연결될 수 있다. AMI는 ID로 지정하므로 AMI에 discovery 태그를 붙이지 않는다.

### 1-3. Karpenter가 자동으로 붙일 태그 — 직접 입력하지 않음

| 생성된 확장 EC2의 태그 키 | 값 예시 | 용도 |
|---|---|---|
| `kubernetes.io/cluster/sayscene` | `owned` | 클러스터 소속·IAM의 생성/종료 권한 조건 |
| `karpenter.sh/nodepool` | `general` | 이 EC2를 만든 NodePool |
| `karpenter.k8s.aws/ec2nodeclass` | `general` | EC2 설정을 제공한 NodeClass |
| `karpenter.sh/nodeclaim` | 자동 생성된 이름 | Kubernetes의 NodeClaim과 EC2 연결 |
| `eks:eks-cluster-name` | `sayscene` | AWS provider의 기본 클러스터 식별. EKS를 생성한다는 의미가 아님 |

이 표는 EC2 인스턴스 기준이다. 볼륨·Launch Template 등 자원별 태그 집합은 다를 수 있다. Karpenter 예약 소유권 태그를 NodeClass의 `spec.tags`에 수동 복제하지 않는다. **기본 노드에는 `karpenter.sh/nodepool` 소유권 태그나 Kubernetes 라벨을 붙이지 않는다.**

### 1-4. SSM 경로·IAM·Kubernetes 라벨은 AWS 태그와 별개

| 항목 | 설정 위치 | 이번 구성에서 의미 |
|---|---|---|
| `/sayscene/kubeadm/token` 등 | SSM Parameter Store의 **이름** | 파라미터 조회 경로. 태그가 아님 |
| SSM 파라미터의 태그 | Parameter Store → 파라미터 → 태그 | 선택적 관리 정보. 이 가이드에서는 불필요 |
| IAM 역할·프로파일의 discovery 태그 | IAM | 미리 만든 프로파일 이름을 사용하므로 불필요 |
| `sayscene.io/workload=general` | kubectl / NodePool YAML | 앱을 받을 노드의 **Kubernetes 라벨**. AWS 콘솔 태그로 추가해도 스케줄링에는 반영되지 않음 |
| `node-role.kubernetes.io/control-plane:NoSchedule` | kubectl로 조회·삭제 | 일반 파드를 막는 **Kubernetes taint**. 8단계에서 기본 노드에 한해 해제 |
| `kubernetes.io/role/elb` 등 | 나중에 ALB/LBC 도입 시 서브넷 | 현재 기본 노드U 확장 단계에서는 추가하지 않음 |

**통과 기준:** AMI·네트워크·IP 여유 확인, 위 세 대상에 정확한 태그 저장, 불필요한 discovery 후보 없음.

## 2. 콘솔에서 EC2 역할과 프로파일을 준비한다

필요한 이유: EC2는 역할의 임시 자격 증명으로 ECR·CNI·SSM에 접근한다. 기본 노드의 EC2 생성·종료 권한과 확장 워커의 조회 권한을 분리한다.

**AWS 콘솔 → IAM → 역할 → 역할 생성**에서 각각 만든다. 이미 만든 역할·프로파일이 있으면 사용처·정책을 확인하고 재사용한다.

1. 신뢰할 수 있는 엔터티: **AWS 서비스**.
2. 서비스/사용 사례: **EC2**.
3. 권한 정책에서 아래 공통 정책을 선택한다.
4. 역할 이름을 지정해 생성한다.

| 역할 이름 | 연결 대상 | 공통 AWS 관리형 정책 |
|---|---|---|
| `SaySceneControlPlaneRole` | 기본 노드만 | `AmazonEC2ContainerRegistryReadOnly`, `AmazonEKS_CNI_Policy` |
| `SaySceneWorkerRole` | 확장 워커만 | `AmazonEC2ContainerRegistryReadOnly`, `AmazonEKS_CNI_Policy` |

CNI 정책 이름에 EKS가 있어도 self-managed 클러스터의 aws-node가 사용할 수 있다. Session Manager로 접속한다면 해당 역할에 `AmazonSSMManagedInstanceCore`도 추가하고 SSM Agent·통신 경로를 준비한다. 이는 **Session Manager 접속용**이며, 워커의 가입 파라미터 읽기 권한은 3단계에서 별도로 준다.

IAM 콘솔에서 **EC2 사용 사례로** 역할을 만들면 **동일 이름의 인스턴스 프로파일이 자동 생성**된다. 따라서 신규 콘솔 구성의 `WORKER_PROFILE`은 `SaySceneWorkerRole`이다. 이전 CLI 절차로 `SaySceneWorkerInstanceProfile`을 만들었다면 실제 그 프로파일을 유지하고 `settings.env`에 그 이름을 적는다. 역할 이름과 프로파일 이름이 항상 같은 것은 아니다.

각 역할의 **신뢰 관계**에서 `ec2.amazonaws.com`에 `sts:AssumeRole`이 허용되는지 확인한다. 이 단계에서 Karpenter용 IAM 역할·프로파일 discovery 태그를 추가할 필요는 없다.

**통과 기준:** 역할 둘의 공통 정책·신뢰 관계 확인, 실제 워커 프로파일 이름 기록.

## 3. 콘솔에서 추가 IAM 정책을 입력하고 기본 노드에 역할을 연결한다

필요한 이유: Karpenter는 EC2 생성·태그·조회·종료와 지정 워커 역할 전달 권한이 필요하다. 워커는 지정된 SSM 가입 경로만 읽으면 되므로 권한을 따로 연결한다.

### 3-1. 정책 JSON에서 실제 값 치환

문서와 함께 제공하는 파일을 편집기로 연다. 파일 편집은 로컬 PC에서 해도 되며 노드에 업로드할 필요는 없다.

- [기본 노드의 Karpenter·CCM 정책](./aws-console-iam/base-node-controller-policy.template.json)
- [워커의 가입 정보 읽기 정책](./aws-console-iam/worker-join-read-policy.template.json)

파일의 아래 문자열을 **모두 찾아 바꾸기**한다. 이 파일은 EKS 기본 예제의 역할 생성을 실행하지 않고, 고정 Karpenter 소스의 NodeLifecyclePolicy·ResourceDiscoveryPolicy에 필요한 권한만 조합한 것이다.

| 치환할 문자열 | 입력값 예시 |
|---|---|
| `REPLACE_REGION` | `ap-northeast-2` |
| `REPLACE_ACCOUNT_ID` | 자신의 12자리 AWS 계정 ID |
| `REPLACE_CLUSTER_NAME` | `sayscene` |
| `REPLACE_WORKER_ROLE_NAME` | `SaySceneWorkerRole` — 프로파일 이름이 아닌 **역할 이름** |

원본 템플릿에 실제 계정 값을 저장할 필요는 없다. 편집한 JSON을 다음 콘솔 입력에 사용한다. 역할을 별도 IAM path 아래 만들었다면 PassRole의 역할 ARN도 실제 path가 포함된 ARN으로 맞춘다.

### 3-2. 콘솔의 역할별 인라인 정책 생성

1. **IAM → 역할 → SaySceneControlPlaneRole → 권한 → 권한 추가 → 인라인 정책 생성**.
2. **JSON** 편집기에서 기존 내용을 지우고, 치환한 기본 노드 정책 전체를 붙여 넣는다.
3. 검증 메시지와 `REPLACE_` 잔존 여부를 확인하고 정책 이름 `SaySceneKarpenterAndCCM`으로 저장한다.
4. **SaySceneWorkerRole**에도 같은 방식으로 치환한 워커 정책을 넣고 `SaySceneJoinRead`로 저장한다.

| 추가 정책 | 핵심 내용 | 적용 대상 |
|---|---|---|
| `SaySceneKarpenterAndCCM` | 공식 lifecycle·discovery 권한, 지정 WorkerRole에 대한 PassRole, CCM 노드 조회 | 기본 노드 역할 |
| `SaySceneJoinRead` | `ssm:GetParameter`, `arn:aws:ssm:리전:계정:parameter/sayscene/kubeadm/*` | 워커 역할 |

워커 정책의 권한은 **파라미터 이름 경로**로 제한하며 SSM 태그 조건을 사용하지 않는다. 기본 `alias/aws/ssm` KMS 키를 기준으로 한다. 고객 관리 키를 선택하면 그 키 정책과 `kms:Decrypt`도 준비한다.

기본 노드 정책의 공개 SSM 경로 읽기는 Karpenter의 AMI 발견용 기본 권한이다. 사용자의 `/sayscene/kubeadm/*` 가입 경로와는 다르다. 가입 파라미터는 사람이 콘솔에서 생성하므로 기본 노드에 `ssm:PutParameter` 권한을 부여하지 않는다.

워커 프로파일을 미리 만들었으므로 Karpenter에 프로파일 생성·삭제 권한을 주지 않는다. EKS 조회·SQS interruption·Zonal Shift 정책은 이번 예제에서 사용하지 않는다. CCM은 노드 controller 두 개만 사용하는 정책이며 Service LB·route·EBS 전체 권한을 뜻하지 않는다.

이전 CLI 절차의 inline 정책이 이미 있으면 중복 추가 전에 내용을 대조한다. 동일 내용은 재사용하고, `SaySceneJoinManage` 같은 기존 가입 관리 정책은 다른 사용처가 없는지 확인한 뒤 정리한다. 조직 SCP·permissions boundary 및 역할의 inline 총량 제한도 적용된다.

### 3-3. 기본 EC2에 역할 연결

**EC2 → 인스턴스 → 기본 노드 → 작업 → 보안 → IAM 역할 수정**에서 기본 노드 프로파일을 선택하고 업데이트한다. 신규 콘솔 구성이라면 `SaySceneControlPlaneRole`로 표시된다. 기존 역할을 바꿀 때 필요한 운영 권한을 보존한다.

전파 후 **기본 노드 셸**에서 확인한다.

```bash
aws sts get-caller-identity
```

ARN이 `assumed-role/SaySceneControlPlaneRole/...` 형태여야 한다. 이 명령은 **로그인이 아니라 현재 신원 조회**다. `aws configure`나 액세스 키 저장은 필요 없다. 예상하지 못한 사용자 ARN이면 환경 변수·기존 AWS 설정 파일의 정적 자격 증명이 EC2 역할보다 우선하는지 확인한다.

기본 노드의 일반 앱도 호스트 자격 증명 접근을 막는 별도 구성이 없으면 높은 권한에 접근할 수 있다. 이 문서는 신뢰하는 앱을 실행하는 PoC 구성이며, 운영에서는 파드별 신원·접근 격리를 설계한다.

**통과 기준:** 각 역할에 해당 JSON·공통 정책 연결, 기본 EC2의 역할 ARN 확인. 이 조회만으로 모든 AWS 권한을 검증한 것은 아니다.

## 4. 원래 EC2에서 kubeadm init을 실행한다

필요한 이유: API 서버·etcd·scheduler가 있어야 워커가 가입하고 파드를 배치한다. kubelet의 `cloud-provider=external`은 AWS 노드 초기화를 CCM에 맡기는 설정이며 이미지 인증과 별개다.

**기본 노드 셸**의 같은 터미널에서 진행한다. AMI가 available이고 아직 init/join을 실행하지 않은 상태가 전제다.

```bash
. ./settings.env

IMDS_TOKEN="$(curl -fsS --retry 5 --connect-timeout 3 --max-time 15 \
  -X PUT -H 'X-aws-ec2-metadata-token-ttl-seconds: 3600' \
  http://169.254.169.254/latest/api/token)"

metadata() {
  curl -fsS --retry 5 --connect-timeout 3 --max-time 15 \
    -H "X-aws-ec2-metadata-token: $IMDS_TOKEN" \
    "http://169.254.169.254/latest/meta-data/$1"
}

export CP_IP="$(metadata local-ipv4)"
export CP_NODE_NAME="$(metadata local-hostname)"
export CP_INSTANCE_ID="$(metadata instance-id)"
export CP_AZ="$(metadata placement/availability-zone)"
export K8S_VERSION="$(kubeadm version -o short)"

test "$CP_INSTANCE_ID" = "$CONTROL_PLANE_INSTANCE_ID"
test "$(metadata placement/region)" = "$AWS_REGION"

case "$K8S_VERSION" in
  v1.35.*) printf 'Kubernetes: %s\n' "$K8S_VERSION" ;;
  *) echo "Kubernetes 1.35 기준과 다릅니다."; false ;;
esac
```

위 확인이 실패하면 진행하지 않는다. 노드 이름은 EC2 실제 사설 hostname을 사용하고 임의의 `master` 같은 이름으로 바꾸지 않는다.

```bash
cat > kubeadm-init.yaml <<EOF
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: "${CP_IP}"
  bindPort: 6443
nodeRegistration:
  name: "${CP_NODE_NAME}"
  criSocket: unix:///run/containerd/containerd.sock
  kubeletExtraArgs:
    - name: cloud-provider
      value: external
    - name: provider-id
      value: "aws:///${CP_AZ}/${CP_INSTANCE_ID}"
    - name: node-ip
      value: "${CP_IP}"
---
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration
clusterName: "${CLUSTER_NAME}"
kubernetesVersion: "${K8S_VERSION}"
controlPlaneEndpoint: "${CP_IP}:6443"
apiServer:
  certSANs:
    - "${CP_IP}"
    - "${CP_NODE_NAME}"
    - "${CP_EIP}"
networking:
  serviceSubnet: "${SERVICE_CIDR}"
  dnsDomain: cluster.local
---
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
cgroupDriver: systemd
maxPods: ${MAX_PODS}
systemReserved:
  cpu: 100m
  memory: 256Mi
kubeReserved:
  cpu: 200m
  memory: 256Mi
EOF

sudo kubeadm config validate --config kubeadm-init.yaml
sudo kubeadm init --config kubeadm-init.yaml

mkdir -p "$HOME/.kube"
sudo cp /etc/kubernetes/admin.conf "$HOME/.kube/config"
sudo chown "$(id -u):$(id -g)" "$HOME/.kube/config"
chmod 600 "$HOME/.kube/config"

kubectl get nodes -o wide
kubectl get pods -A

printf 'export CP_IP=%q\nexport CP_NODE_NAME=%q\nexport K8S_VERSION=%q\n' \
  "$CP_IP" "$CP_NODE_NAME" "$K8S_VERSION" > cluster-runtime.env
```

`admin.conf`는 관리자 권한이므로 AMI·userData에 넣지 않는다. 새 기본 노드 셸에서는 `settings.env` 다음 `cluster-runtime.env`도 읽는다.

외부 PC에서 직접 kubectl을 사용할 경우 kubeconfig를 안전하게 준비하고, **외부 PC용 파일의** `clusters[].cluster.server`를 `https://기본노드EIP:6443`으로 설정한다. 위 `certSANs`에 EIP를 포함한 것은 이 접속의 TLS 검증을 위한 것이다. 기본 노드의 kubeconfig·SSM endpoint·Karpenter endpoint는 사설 IP를 유지한다.

이미 init한 클러스터에서는 YAML에 EIP를 추가해도 기존 인증서에 자동 반영되지 않는다. 해당 EIP가 SAN에 없다면 기존 클러스터의 인증서 재발급 절차가 필요하며 `kubeadm init`을 다시 실행하지 않는다.

**통과 기준:** init 성공, kubectl로 API 조회 가능. CNI·CCM 전 Node NotReady와 CoreDNS Pending은 예상 상태다. 아직 CoreDNS Ready를 기다리지 않고 다음으로 진행한다.

## 5. Helm을 설치한다

필요한 이유: Helm chart는 controller 파드와 ServiceAccount·RBAC를 함께 배포한다. 관리 도구이므로 기본 노드에 설치해도 워커 AMI를 다시 만들 필요는 없다.

**기본 노드 셸**에서 실행한다. Helm 3이 있으면 설치 블록을 생략하고 버전을 확인한다.

```bash
HELM_BUILD_DIR="$(mktemp -d)"
(
  set -e
  cd "$HELM_BUILD_DIR"
  curl -fsSLO "https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz"
  curl -fsSLO "https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz.sha256sum"
  sha256sum -c "helm-${HELM_VERSION}-linux-amd64.tar.gz.sha256sum"
  tar -xzf "helm-${HELM_VERSION}-linux-amd64.tar.gz"
  sudo install -m 0755 linux-amd64/helm /usr/local/bin/helm
)
helm version --short
```

**통과 기준:** checksum 성공, 의도한 Helm 버전 출력.

## 6. AWS VPC CNI를 설치한다

필요한 이유: containerd만으로는 파드에 VPC IP를 할당하고 통신을 연결하지 못한다. aws-node가 ENI·보조 IP를 관리하고 각 노드의 컨테이너 네트워크를 준비한다.

**기본 노드 셸**에서 고정 매니페스트를 적용한다.

```bash
. ./settings.env
. ./cluster-runtime.env

curl -fsSLo aws-k8s-cni.yaml \
  "https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/${CNI_COMMIT}/config/master/aws-k8s-cni.yaml"

kubectl apply -f aws-k8s-cni.yaml
kubectl -n kube-system set env daemonset/aws-node \
  CLUSTER_NAME="$CLUSTER_NAME" CLUSTER_ENDPOINT="https://${CP_IP}:6443"

kubectl -n kube-system rollout status daemonset/aws-node --timeout=300s
kubectl -n kube-system get pods -l k8s-app=aws-node -o wide
```

매니페스트의 hostNetwork·toleration으로 일반 파드 네트워크와 초기 taint가 남아 있어도 기동한다. IPv4 보조 IP 모드를 사용하고 prefix delegation·network policy는 별도로 켜지 않는다.

CNI 이미지가 AWS private ECR 형태 주소를 사용하므로 기본 노드에도 ECR provider·권한이 필요하다.

**통과 기준:** aws-node 롤아웃 성공. 실패 시 해당 파드의 `describe`에서 이미지 pull·IAM·init 오류를 확인한다. cloud 초기화 taint가 남았다면 CCM 설치를 계속한다.

## 7. AWS Cloud Controller Manager를 설치한다

필요한 이유: CCM은 Kubernetes Node를 실제 EC2와 연결하고 AWS 주소·영역·초기화 상태를 반영한다. ECR provider는 이미지 인증만 하므로 CCM을 대신하지 못한다.

**기본 노드 셸**에서 고정 소스를 가져온다. 이미 디렉터리가 있으면 clone을 다시 하지 말고 checkout을 확인한다.

```bash
git clone --no-checkout \
  https://github.com/kubernetes/cloud-provider-aws.git cloud-provider-aws-ccm
git -C cloud-provider-aws-ccm checkout --detach "$CCM_COMMIT"
test "$(git -C cloud-provider-aws-ccm rev-parse HEAD)" = "$CCM_COMMIT"

cat > aws-ccm-values.yaml <<EOF
namespace: kube-system
nameOverride: aws-cloud-controller-manager
image:
  repository: registry.k8s.io/provider-aws/cloud-controller-manager
  tag: v1.35.2
hostNetworking: true
dnsPolicy: Default
nodeSelector:
  node-role.kubernetes.io/control-plane: ""
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
  - key: node.cloudprovider.kubernetes.io/uninitialized
    operator: Exists
    effect: NoSchedule
args:
  - --cloud-provider=aws
  - --cluster-name=${CLUSTER_NAME}
  - --controllers=cloud-node-controller,cloud-node-lifecycle-controller
  - --configure-cloud-routes=false
  - --use-service-account-credentials=false
  - --leader-elect=true
  - --v=2
EOF

helm template aws-cloud-controller-manager \
  ./cloud-provider-aws-ccm/charts/aws-cloud-controller-manager \
  --namespace kube-system --values aws-ccm-values.yaml > aws-ccm-rendered.yaml

helm upgrade --install aws-cloud-controller-manager \
  ./cloud-provider-aws-ccm/charts/aws-cloud-controller-manager \
  --namespace kube-system --values aws-ccm-values.yaml

kubectl -n kube-system rollout status \
  daemonset/aws-cloud-controller-manager --timeout=300s
kubectl -n kube-system logs daemonset/aws-cloud-controller-manager --tail=100
```

chart 기본 이미지가 소스 릴리스보다 오래될 수 있어 `tag: v1.35.2`를 명시한다. `hostNetworking`은 이 chart의 실제 옵션명이다. node controller 두 개만 사용하고 Service LB·route controller는 켜지 않는다.

hostNetwork·기본 DNS·초기 taint toleration은 CoreDNS 준비 전에도 CCM을 띄우기 위한 설정이다. 기본 노드 역할의 임시 자격 증명을 사용하며 정적 키를 파드에 넣지 않는다.

**통과 기준:** CCM 실행, 지속적인 AWS 권한·인스턴스 조회 오류 없음. 클러스터 태그·노드 이름·providerID도 확인한다.

## 8. 기본 노드에 일반 앱 배치를 허용하고 준비 상태를 확인한다

필요한 이유: kubeadm은 기본적으로 컨트롤 플레인에 일반 파드를 막는 taint를 붙인다. 이 아키텍처는 같은 EC2에서 앱도 실행하므로 해당 taint를 해제하고 공통 앱 라벨을 붙여야 한다.

**기본 노드 셸**에서 먼저 CCM 초기화가 정상 완료됐는지 확인한다.

```bash
kubectl wait --for=condition=Ready "node/$CP_NODE_NAME" --timeout=300s
kubectl get node "$CP_NODE_NAME" \
  -o jsonpath='{.spec.providerID}{"\n"}{.spec.taints}{"\n"}'
```

`node.cloudprovider.kubernetes.io/uninitialized`가 남아 있으면 CCM의 태그·IAM·providerID·로그를 먼저 해결한다. 이 taint는 수동 삭제하지 않는다.

정상 초기화 후 **기본 노드 한 대에만** 다음을 적용한다.

```bash
if kubectl get node "$CP_NODE_NAME" -o json \
  | python3 -c 'import json,sys; n=json.load(sys.stdin); sys.exit(0 if any(t["key"]=="node-role.kubernetes.io/control-plane" and t["effect"]=="NoSchedule" for t in n["spec"].get("taints",[])) else 1)'; then
  kubectl taint node "$CP_NODE_NAME" \
    node-role.kubernetes.io/control-plane:NoSchedule-
fi

kubectl label node "$CP_NODE_NAME" \
  sayscene.io/workload=general sayscene.io/node-role=base --overwrite

kubectl -n kube-system rollout status deployment/coredns --timeout=300s
kubectl -n kube-system rollout status daemonset/kube-proxy --timeout=300s
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl describe node "$CP_NODE_NAME"
sudo grep -A 4 -E '^(cgroupDriver|systemReserved|kubeReserved):' \
  /var/lib/kubelet/config.yaml
```

`node-role.kubernetes.io/control-plane` **라벨은 유지**한다. CCM·Karpenter가 이 라벨로 기본 노드에 고정된다. 해제한 것은 같은 이름의 NoSchedule **taint**이며, 라벨과 다른 항목이다. `sayscene.io/node-role=base`는 일반 앱의 배치 선호도에 사용한다.

4단계 예약량은 시스템 100m/256Mi + Kubernetes 프로세스 200m/256Mi인 예시다. 예약량은 파드에 노출되는 allocatable을 줄이며, CCM·Karpenter 등 **파드**의 requests는 이와 별도로 계산된다. 실제 컨트롤 플레인·앱 부하에 맞춰 여유를 늘리고 앱 requests/limits도 지정한다.

**통과 기준:** 기본 노드 Ready, CNI·CCM·DNS·kube-proxy 정상, 올바른 providerID, cloud 초기화 taint와 control-plane NoSchedule taint 없음, 공통 앱 라벨 있음. 기본 노드에는 `karpenter.sh/nodepool` 라벨이 없어야 한다.

## 9. 기본 노드에서 가입 정보를 만들고 콘솔의 SSM에 저장한다

필요한 이유: 토큰을 AMI·userData에 넣지 않으면 AMI 재사용과 토큰 교체를 분리할 수 있다. 새 워커는 자신의 역할로 부팅 시점의 가입 정보를 읽는다.

### 9-1. 기본 노드에서 생성

**기본 노드 셸**에서 실행한다. `--ttl 0`은 자동 만료되지 않는 PoC 설정이다. 토큰을 교체하거나 종료할 때 Kubernetes 토큰과 SSM 값을 함께 관리한다.

```bash
JOIN_TOKEN="$(sudo kubeadm token create --ttl 0)"
CA_HASH="$(sudo openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt \
  | openssl pkey -pubin -outform DER \
  | openssl dgst -sha256 -hex | awk '{print "sha256:" $2}')"

printf 'endpoint: %s:6443\nca-hash: %s\ntoken: %s\n' \
  "$CP_IP" "$CA_HASH" "$JOIN_TOKEN"
```

마지막 출력은 본인이 콘솔에 옮겨 적기 위한 것이다. token을 공유 로그·스크린샷·Git에 남기지 않는다. 스크립트나 명령 인수에 실제 토큰을 적을 필요는 없다.

### 9-2. AWS 콘솔에서 세 파라미터 생성

**같은 리전 → Systems Manager → Parameter Store → 파라미터 생성**으로 각각 저장한다.

| 이름 — 클러스터 이름이 sayscene인 예시 | 계층·데이터 형식 | 유형 | 값 |
|---|---|---|---|
| `/sayscene/kubeadm/endpoint` | Standard · text | String | 위 endpoint 출력. `https://` 없이 사설 IP:6443 |
| `/sayscene/kubeadm/token` | Standard | SecureString | 위 token 출력. KMS 키는 기본 `alias/aws/ssm` |
| `/sayscene/kubeadm/ca-hash` | Standard · text | String | 위 ca-hash 출력. `sha256:` 접두사 포함 |

기존 파라미터가 있으면 새로 중복 생성하지 않고 해당 항목의 **편집**에서 값을 갱신한다. endpoint·ca-hash의 데이터 형식은 `aws:ec2:image`가 아닌 일반 `text`다. SecureString은 암호화용 KMS 키를 선택한다.

**이 단계에서 SSM 태그는 붙이지 않는다.** 워커는 이름 경로로 조회하고, IAM도 `parameter/sayscene/kubeadm/*` 리소스 ARN으로 허용한다. 태그를 붙여도 이름·값·읽기 권한을 대신하지 않는다.

목록과 각 세부 정보에서 이름·리전·유형·최신 버전을 확인하고, 입력한 endpoint·CA hash를 대조한다. token은 SecureString인지 확인하며 검증을 위해 다시 출력·공유하지 않는다. 입력을 마쳤으면 **기본 노드 셸**에서 지운다.

```bash
unset JOIN_TOKEN CA_HASH
```

최종 조회 성공 여부는 11단계 수동 가입 또는 15단계 Karpenter 가입으로 검증한다. 기본 노드의 IP 변경은 SSM뿐 아니라 API 인증서·Karpenter endpoint에도 영향을 주므로 사설 주소를 안정적으로 유지한다.

**통과 기준:** 같은 리전에 세 이름·타입·값을 저장, token은 SecureString, 워커 IAM의 경로와 일치.

## 10. 워커 가입 userData를 작성한다

필요한 이유: AMI는 공통 프로그램만 제공한다. 새 EC2의 실제 이름·IP·providerID와 현재 토큰을 부팅 때 결합해야 각 워커가 고유 신원으로 가입한다.

**기본 노드 셸**에서 파일만 만든다. 바깥 `<<'EOF'` 따옴표를 유지해야 워커용 변수가 현재 기본 노드에서 치환되지 않는다. 파일을 기본 노드에서 직접 실행하지 않는다.

```bash
cat > worker-userdata.sh <<'EOF'
#!/bin/bash
set -euo pipefail
umask 077

CLUSTER_NAME="__CLUSTER_NAME__"
EXPECTED_REGION="__AWS_REGION__"
MAX_PODS="__MAX_PODS__"
KARPENTER_MANAGED=true

if [[ -s /etc/kubernetes/kubelet.conf ]]; then
  echo "이미 가입한 노드입니다. 자동 재가입을 생략합니다."
  exit 0
fi

IMDS_TOKEN="$(curl -fsS --retry 5 --connect-timeout 3 --max-time 15 \
  -X PUT -H 'X-aws-ec2-metadata-token-ttl-seconds: 3600' \
  http://169.254.169.254/latest/api/token)"

metadata() {
  curl -fsS --retry 5 --connect-timeout 3 --max-time 15 \
    -H "X-aws-ec2-metadata-token: $IMDS_TOKEN" \
    "http://169.254.169.254/latest/meta-data/$1"
}

AWS_REGION="$(metadata placement/region)"
NODE_IP="$(metadata local-ipv4)"
NODE_NAME="$(metadata local-hostname)"
INSTANCE_ID="$(metadata instance-id)"
AZ="$(metadata placement/availability-zone)"
test "$AWS_REGION" = "$EXPECTED_REGION"

ssm_value() {
  aws ssm get-parameter --region "$AWS_REGION" \
    --name "$1" --with-decryption --query Parameter.Value --output text
}
API_ENDPOINT="$(ssm_value "/${CLUSTER_NAME}/kubeadm/endpoint")"
JOIN_TOKEN="$(ssm_value "/${CLUSTER_NAME}/kubeadm/token")"
CA_HASH="$(ssm_value "/${CLUSTER_NAME}/kubeadm/ca-hash")"

[[ "$JOIN_TOKEN" =~ ^[a-z0-9]{6}\.[a-z0-9]{16}$ ]]
[[ "$CA_HASH" =~ ^sha256:[0-9a-f]{64}$ ]]
[[ "$API_ENDPOINT" != *$'\n'* ]]

JOIN_FILE=/run/sayscene-kubeadm-join.yaml
trap 'rm -f "$JOIN_FILE"' EXIT

cat > "$JOIN_FILE" <<JOIN
apiVersion: kubeadm.k8s.io/v1beta4
kind: JoinConfiguration
discovery:
  bootstrapToken:
    apiServerEndpoint: "${API_ENDPOINT}"
    token: "${JOIN_TOKEN}"
    caCertHashes:
      - "${CA_HASH}"
nodeRegistration:
  name: "${NODE_NAME}"
  criSocket: unix:///run/containerd/containerd.sock
JOIN

if [[ "$KARPENTER_MANAGED" == true ]]; then
  cat >> "$JOIN_FILE" <<'TAINTS'
  taints:
    - key: karpenter.sh/unregistered
      effect: NoExecute
TAINTS
else
  printf '  taints: []\n' >> "$JOIN_FILE"
fi

cat >> "$JOIN_FILE" <<JOIN
  kubeletExtraArgs:
    - name: cloud-provider
      value: external
    - name: provider-id
      value: "aws:///${AZ}/${INSTANCE_ID}"
    - name: node-ip
      value: "${NODE_IP}"
    - name: max-pods
      value: "${MAX_PODS}"
JOIN

systemctl start containerd
kubeadm join --config "$JOIN_FILE"
unset JOIN_TOKEN
EOF

python3 - <<'PY'
import os
import re
from pathlib import Path
values = {
    "__CLUSTER_NAME__": os.environ["CLUSTER_NAME"],
    "__AWS_REGION__": os.environ["AWS_REGION"],
    "__MAX_PODS__": os.environ["MAX_PODS"],
}
if not re.fullmatch(r"[A-Za-z0-9][A-Za-z0-9_-]*", values["__CLUSTER_NAME__"]):
    raise SystemExit("클러스터 이름 오류")
if not re.fullmatch(r"[a-z0-9-]+", values["__AWS_REGION__"]):
    raise SystemExit("리전 오류")
if not values["__MAX_PODS__"].isdigit():
    raise SystemExit("MAX_PODS 오류")
path = Path("worker-userdata.sh")
body = path.read_text()
for key, value in values.items():
    body = body.replace(key, value)
path.write_text(body)
PY

bash -n worker-userdata.sh
wc -c worker-userdata.sh
```

userData 원문은 16KB 안에 있어야 한다. 토큰·정적 키를 파일에 넣지 않고, 조회 후 만드는 임시 파일은 root만 읽고 종료 시 삭제한다. `set -x`나 토큰 출력은 추가하지 않는다.

AMI의 `/etc/sysconfig/kubelet`에 있는 ECR provider 설정은 덮어쓰지 않는다. maxPods는 NodeClass와 실제 kubelet에 동일하게 반영한다.

| 등록 taint | 처리 주체 |
|---|---|
| `node.cloudprovider.kubernetes.io/uninitialized` | CCM |
| `karpenter.sh/unregistered` | Karpenter |

`kubelet.conf` 생성 뒤 가입이 일부 실패했다면 단순 재실행으로 복구된다고 가정하지 않는다. 로그를 확인하고 실패한 테스트 워커를 교체해 새 부팅으로 검증한다.

**통과 기준:** Bash 문법 통과, 16KB 미만, 토큰·CP IP가 고정되지 않은 파일.

## 11. 선택: 콘솔에서 수동 워커를 띄워 가입 경로를 검증한다

필요한 이유: AMI·SSM·통신·가입 문제를 Karpenter와 분리해 확인할 수 있다. 수동 노드는 NodeClaim이 없으므로 Karpenter 전용 등록 taint를 넣지 않는다.

이미 가입 경로를 검증했다면 생략한다. **기본 노드 셸**에서 수동 검증용 userData만 만든다.

```bash
sed 's/^KARPENTER_MANAGED=true$/KARPENTER_MANAGED=false/' \
  worker-userdata.sh > worker-manual-userdata.sh
bash -n worker-manual-userdata.sh
cat worker-manual-userdata.sh
```

**AWS 콘솔 → EC2 → 인스턴스 시작**에서 설정한다.

| 항목 | 입력 |
|---|---|
| 이름 | `sayscene-manual-test` |
| AMI | 내 AMI → 앞에서 만든 CPU Custom AMI ID |
| 유형 | `m6i.large` |
| 키 페어 | SSH가 필요하면 본인의 키 페어. Session Manager 사용 시 그 접속 조건 준비 |
| 네트워크 | 기존 VPC·워커 후보 서브넷·공통 노드 SG |
| 퍼블릭 IP | 자동 할당 활성화. 워커에는 별도 EIP 불필요 |
| 스토리지 | 원본 AMI 크기 이상, 예시 40GiB gp3 |
| 고급 세부 정보 → IAM 인스턴스 프로파일 | 실제 워커 프로파일. 신규 콘솔 구성은 `SaySceneWorkerRole` |
| 고급 세부 정보 → 메타데이터 | 메타데이터 접근 활성, IMDSv2 필수, hop limit 1 |
| 고급 세부 정보 → 사용자 데이터 | 위 `worker-manual-userdata.sh` 전체를 평문으로 붙여 넣음. 이미 base64 인코딩한 데이터 옵션은 선택하지 않음 |
| 태그 추가 | 대상 인스턴스, 키 `kubernetes.io/cluster/sayscene`, 값 `owned` |

인스턴스 1대를 시작하고 ID를 기록한다. 수동 노드에 `karpenter.sh/nodepool` 등 소유권 태그는 추가하지 않는다.

**기본 노드 셸**에서 새 이름을 확인한다.

```bash
kubectl get nodes -o wide
# 실제 새 워커 이름으로 실행:
# kubectl wait --for=condition=Ready node/NEW_NODE_NAME --timeout=600s
# kubectl describe node NEW_NODE_NAME
```

수동 노드가 Ready이고 providerID가 기록한 EC2 ID인지 확인한다. 검증 후 **기본 노드 셸**에서 해당 워커의 일반 파드를 확인하고 drain한다.

```bash
# 실제 수동 워커 이름으로 실행:
# kubectl drain NEW_NODE_NAME --ignore-daemonsets
```

그다음 **EC2 콘솔 → 기록한 수동 검증 인스턴스 선택 → 인스턴스 상태 → 인스턴스 종료**로 정리한다. 기본 노드를 선택하지 않는다. 이 수동 워커는 Karpenter가 자동 인수하거나 정리하지 않는다.

**통과 기준:** 수동 워커 Ready·고유 providerID 확인, 검증 후 EC2 종료 및 CCM의 Node 정리 확인.

## 12. Karpenter controller·CRD를 설치한다

필요한 이유: Karpenter는 배치할 수 없는 파드 수요를 계산해 EC2를 생성·정리한다. 기본 노드에 두어 워커가 0대여도 다시 확장할 수 있게 한다.

**기본 노드 셸**에서 실행한다. Kubernetes 1.35는 Karpenter 1.9 이상이 필요하고 이 가이드는 1.15.0을 사용한다.

```bash
. ./settings.env
. ./cluster-runtime.env

cat > karpenter-values.yaml <<EOF
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
  clusterName: ${CLUSTER_NAME}
  clusterEndpoint: https://${CP_IP}:6443
  eksControlPlane: false
  enableZonalShift: false
  interruptionQueue: ""
controller:
  env:
    - name: AWS_REGION
      value: ${AWS_REGION}
    - name: EKS_CONTROL_PLANE
      value: "false"
  resources:
    requests:
      cpu: 500m
      memory: 512Mi
    limits:
      memory: 1Gi
EOF

helm show chart oci://public.ecr.aws/karpenter/karpenter \
  --version "$KARPENTER_VERSION"

helm template karpenter oci://public.ecr.aws/karpenter/karpenter \
  --version "$KARPENTER_VERSION" --namespace kube-system \
  --values karpenter-values.yaml > karpenter-rendered.yaml

grep -nE 'image:|hostNetwork:|CLUSTER_ENDPOINT|CLUSTER_NAME|EKS_CONTROL_PLANE' \
  karpenter-rendered.yaml
```

OCI 패키지에서 렌더링된 controller 이미지가 의도한 버전인지 확인한다. 소스 chart에 오래된 기본 이미지가 남을 수 있으므로 이전 digest를 복사하지 않는다. 확인 항목은 실제 API 주소·`EKS_CONTROL_PLANE=false`·기본 노드 배치·hostNetwork다. chart의 기본 `karpenter.sh/nodepool DoesNotExist` affinity는 유지한다.

```bash
helm upgrade --install karpenter-crd \
  oci://public.ecr.aws/karpenter/karpenter-crd \
  --version "$KARPENTER_VERSION" --namespace kube-system --wait

helm upgrade --install karpenter \
  oci://public.ecr.aws/karpenter/karpenter \
  --version "$KARPENTER_VERSION" --namespace kube-system --skip-crds \
  --values karpenter-values.yaml --wait --timeout 5m

kubectl -n kube-system get pods \
  -l app.kubernetes.io/name=karpenter -o wide
kubectl -n kube-system logs deployment/karpenter --tail=100
```

hostNetwork는 IMDS hop limit 1에서도 기본 노드 역할을 이용하기 위한 선택이다. 다른 호스트 프로세스·hostNetwork 파드도 해당 역할에 접근할 수 있는 한계가 있다. 기본 노드 역할을 확장 워커에 부착하지 않는다.

**통과 기준:** controller가 기본 노드에서 실행되고 반복적인 인증·AWS API 오류가 없다. CRD와 controller 버전은 일치한다.

## 13. CPU EC2NodeClass를 생성한다

필요한 이유: NodeClass는 EC2의 AMI·네트워크·IAM·디스크·부트스트랩을 정의한다. Custom AMI에서는 Karpenter가 가입 스크립트를 자동 보완하지 않으므로 전체 userData를 연결한다.

**기본 노드 셸**에서 실행한다. 루트 장치와 크기는 0단계에서 AMI 콘솔을 보고 `settings.env`에 기록한 값을 사용한다. `ROOT_DEVICE`를 추측하지 않고 원본 AMI의 루트 매핑과 일치시킨다.

```bash
. ./settings.env
test -n "$ROOT_DEVICE"
test "$ROOT_VOLUME_GIB" -ge 1

cat > ec2nodeclass-general.yaml <<EOF
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: general
spec:
  amiFamily: Custom
  amiSelectorTerms:
    - id: ${CPU_AMI_ID}
  instanceProfile: ${WORKER_PROFILE}
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: ${CLUSTER_NAME}
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: ${CLUSTER_NAME}
  associatePublicIPAddress: ${WORKER_PUBLIC_IP}
  metadataOptions:
    httpEndpoint: enabled
    httpTokens: required
    httpPutResponseHopLimit: 1
  blockDeviceMappings:
    - deviceName: ${ROOT_DEVICE}
      ebs:
        volumeSize: ${ROOT_VOLUME_GIB}Gi
        volumeType: gp3
        encrypted: true
        deleteOnTermination: true
  kubelet:
    maxPods: ${MAX_PODS}
    systemReserved:
      cpu: 100m
      memory: 256Mi
    kubeReserved:
      cpu: 200m
      memory: 256Mi
EOF

printf '  userData: |\n' >> ec2nodeclass-general.yaml
sed 's/^/    /' worker-userdata.sh >> ec2nodeclass-general.yaml

kubectl apply --dry-run=server -f ec2nodeclass-general.yaml
kubectl apply -f ec2nodeclass-general.yaml
kubectl wait --for=condition=Ready ec2nodeclass/general --timeout=300s
kubectl describe ec2nodeclass general
```

NodeClass의 예약량은 kubeadm을 통해 워커가 받는 kubelet 설정과 일치시켰다. Custom AMI에서는 NodeClass 설정만으로 kubelet 파일이 자동 수정되지 않는다.

루트 40Gi는 용량 예시이며 원본 AMI가 더 크면 그 크기를 사용한다. CPU AMI를 루트 EBS 하나로 만들었다는 전제다. 추가 EBS에 필요한 데이터가 있다면 그 매핑·스냅샷도 보존한다. 고객 관리 KMS 키를 사용한다면 시작에 필요한 키 권한도 준비한다.

**통과 기준:** NodeClass Ready, 실제 AMI·서브넷·SG·프로파일 조회 성공. server dry-run만으로 EC2 부팅 성공이 검증되지는 않는다. Karpenter 예약 소유권 태그는 controller가 부여하므로 수동 추가하지 않는다.

## 14. CPU NodePool을 생성한다

필요한 이유: NodePool은 허용 인스턴스 유형·구매 방식·라벨·용량 상한을 정한다. 첫 검증은 on-demand로 진행해 Spot 중단 처리와 권한 문제를 분리한다.

**기본 노드 셸**에서 실행한다.

```bash
cat > nodepool-general.yaml <<'EOF'
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: general
spec:
  template:
    metadata:
      labels:
        sayscene.io/workload: general
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: general
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["m6i.large", "c6i.large"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
  limits:
    cpu: "4"
    memory: 16Gi
  disruption:
    consolidationPolicy: WhenEmpty
    consolidateAfter: 1m
EOF

kubectl apply --dry-run=server -f nodepool-general.yaml
kubectl apply -f nodepool-general.yaml
kubectl wait --for=condition=Ready nodepool/general --timeout=300s
kubectl describe nodepool general
```

두 유형의 리전·AZ 가용성과 On-Demand CPU 쿼터가 필요하다. maxPods 20은 이 유형의 IPv4 ENI 수용량 안에서 사용하는 예시다. 다른 유형·CNI 모드를 사용하면 다시 계산한다.

limits는 풀 전체 EC2 용량이다. 병렬 생성 중 일시 초과할 수 있어 절대적 지출 하드 캡이 아니다. AMI drift·기본 노드 만료는 WhenEmpty consolidation과 다른 동작이다.

**통과 기준:** NodePool Ready. 수요가 없으면 워커가 아직 없어도 정상이다.

## 15. 기본 노드 앱 실행 → 용량 부족 → 워커 확장을 검증한다

필요한 이유: 일반 앱이 기본 노드에서 먼저 실행되고 용량 부족 때만 확장되는 흐름을 확인해야 한다. 기본 노드와 워커에 공통 라벨을 사용하고, 기본 노드 선호는 강제가 아닌 soft affinity로 표현한다.

### 15-1. 기본 노드에서 앱 한 개 실행

**기본 노드 셸**에서 여유와 라벨을 확인한다. 가능하면 다른 일반 앱을 올리기 전 수행한다.

```bash
kubectl get nodes -L sayscene.io/workload,sayscene.io/node-role,karpenter.sh/nodepool
kubectl describe node "$CP_NODE_NAME"
```

`Allocated resources`의 **Requests**와 `Allocatable`을 비교한다. 기준 노드에 100m CPU·64Mi 메모리와 파드 슬롯 여유가 있어야 다음 파드 한 개가 먼저 배치된다. 여유가 이미 없으면 첫 파드부터 확장되는 것도 정상이다.

```bash
cat > cpu-test.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cpu-test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cpu-test
  template:
    metadata:
      labels:
        app: cpu-test
    spec:
      nodeSelector:
        sayscene.io/workload: general
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: sayscene.io/node-role
                    operator: In
                    values: ["base"]
      containers:
        - name: pause
          image: registry.k8s.io/pause:3.10
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 100m
              memory: 64Mi
EOF

kubectl apply -f cpu-test.yaml
kubectl rollout status deployment/cpu-test --timeout=900s
kubectl get pods -l app=cpu-test -o wide
kubectl get nodeclaims
```

처음 워커가 없고 기본 노드에 여유가 있다면 기본 노드에서 실행되며 NodeClaim이 없어도 정상이다. 이 파드는 CPU를 실제로 소모하는 부하 발생기가 아니라 **스케줄러의 자원 요청·노드 용량을 확인하는 테스트**다.

앱의 `nodeSelector`에는 공통 라벨만 둔다. `node-role.kubernetes.io/control-plane`이나 `sayscene.io/node-role=base`를 **필수 조건**으로 넣으면 확장 워커에서 실행할 수 없다. 위 preference는 기본 노드 점수를 높이는 설정이며 다른 스케줄링 점수까지 포함한 배치를 항상 보장하지는 않는다.

### 15-2. 기본 노드 단독 수용량을 넘기는 파드 수로 확장

이 PoC는 기본 노드의 `maxPods=20`을 사용한다. 테스트 파드를 기본 노드의 전체 파드 슬롯보다 하나 더 요청하면, 기존 시스템 파드를 고려하지 않아도 기본 노드 한 대로는 모두 수용할 수 없다.

```bash
BASE_POD_CAPACITY="$(kubectl get node "$CP_NODE_NAME" \
  -o jsonpath='{.status.allocatable.pods}')"
if [[ "$BASE_POD_CAPACITY" != "$MAX_PODS" || "$MAX_PODS" != 20 ]]; then
  echo "이 테스트는 maxPods=20 기준입니다. 실제 용량과 테스트 예산을 확인하세요."
else
  kubectl scale deployment cpu-test --replicas="$((BASE_POD_CAPACITY + 1))"
fi

kubectl get pods -l app=cpu-test -o wide
kubectl get nodeclaims
kubectl get nodes -o wide
kubectl rollout status deployment/cpu-test --timeout=900s
kubectl get pods -l app=cpu-test -o wide

kubectl get nodeclaims \
  -o custom-columns='NAME:.metadata.name,NODE:.status.nodeName,PROVIDER_ID:.status.providerID'
# 실제 이름으로 실행:
# kubectl describe nodeclaim ACTUAL_NODECLAIM_NAME
```

21개 파드의 요청 합계는 CPU 2.1개·메모리 약 1.31Gi다. 실제로는 기본 노드의 **CPU requests 부족 또는 파드 슬롯 부족 중 먼저 도달한 조건**에서 확장된다. 기본 노드가 CPU를 실제로 100% 쓸 때까지 기다리는 테스트가 아니다.

```text
기본 노드의 남은 용량에 앱 배치
→ 남은 파드 Pending
→ NodeClaim → EC2 생성
→ userData → SSM 조회 → kubeadm join
→ CCM·Karpenter 등록 → CNI 준비 → 추가 파드 Running
```

NodeClaim의 Launched·Registered·Initialized·Ready가 True인지 확인한다. Node와 NodeClaim의 providerID가 같은 EC2를 가리켜야 한다. **AWS 콘솔 → EC2 → 인스턴스**에서도 해당 ID·워커 프로파일·서브넷·SG·자동 태그를 대조한다.

현재 NodePool은 확장 워커 총 CPU 4개까지다. 다른 앱이 이미 용량을 쓰고 있으면 테스트를 줄이거나 먼저 그 요청량을 확인한다. 기본 노드 용량은 이 NodePool 한도에 포함되지 않는다.

앱 배포에도 위 공통 selector·requests·선택적 기본 노드 preference를 적용한다. 이미 워커에서 실행 중인 파드는 기본 노드에 여유가 생겨도 자동으로 이동하지 않는다. `WhenEmpty`는 빈 워커만 축소하며, 앱이 남은 워커에서 기본 노드로 재배치하는 정책은 아니다.

**통과 기준:** 기본 노드의 앱 실행 확인, 수용량 초과 시 워커 생성·가입·앱 Running, EC2 자동 태그·프로파일 확인.

## 16. 실제 private ECR pull을 검증한다

필요한 이유: 공개 pause 이미지는 ECR 인증 경로를 입증하지 못한다. kubelet이 private 이미지를 받아야 provider·역할·저장소 권한이 함께 확인된다.

이미 접근 가능한 private ECR 테스트 이미지가 있다면 실제 값을 넣어 **기본 노드 셸**에서 실행한다. 아래는 셸과 sleep이 있는 작은 이미지 기준이며, 확장 워커에서 확인하도록 이 진단 파드에만 NodePool 라벨을 요구한다. 일반 앱의 공통 selector와는 구분한다.

```bash
# 실제 이미지로 바꿔 실행:
# export ECR_TEST_IMAGE="ACCOUNT.dkr.ecr.REGION.amazonaws.com/test:IMMUTABLE_TAG"
# kubectl run ecr-pull-test --image="$ECR_TEST_IMAGE" \
#   --image-pull-policy=Always --restart=Never \
#   --overrides='{"spec":{"nodeSelector":{"karpenter.sh/nodepool":"general"}}}' \
#   --command -- sh -c 'sleep 300'
# kubectl wait --for=condition=Ready pod/ecr-pull-test --timeout=300s
# kubectl describe pod ecr-pull-test
# kubectl delete pod ecr-pull-test
```

이미지에 셸이 없다면 정상 실행 명령에 맞춘다. imagePullSecrets나 수동 토큰을 추가하지 않아야 이번 provider 경로를 확인할 수 있다. `crictl pull`·`ctr pull`은 kubelet의 exec provider 연결 검증을 대신하지 않는다.

**통과 기준:** kubelet의 private ECR pull과 컨테이너 실행. 테스트 이미지가 없다면 이 항목은 미검증으로 기록한다.

## 17. 스케일인과 콘솔의 EC2 종료 상태를 검증한다

필요한 이유: 테스트 수요가 사라지면 확장 워커·NodeClaim·실제 EC2가 정리되어야 비용과 생명주기 검증이 끝난다. 기본 노드는 관리 기능과 앱 실행을 계속할 수 있어야 한다.

**기본 노드 셸**에서 **Karpenter 워커만** 기록하고 테스트를 삭제한다. 공통 workload 라벨에는 기본 노드도 포함되므로 소유권 라벨로 조회한다.

```bash
kubectl get nodes -l karpenter.sh/nodepool=general \
  -o custom-columns='NAME:.metadata.name,PROVIDER_ID:.spec.providerID'

kubectl delete deployment cpu-test
kubectl get pods -A -o wide
kubectl get nodeclaims
kubectl get nodes -o wide
```

`consolidateAfter: 1m`과 drain·종료 처리 시간을 기다린 뒤 다시 확인한다. 워커에 DaemonSet 외의 일반 앱·테스트 파드가 남아 있으면 `WhenEmpty` 조건이 충족되지 않는다.

**AWS 콘솔 → EC2 → 인스턴스**에서 기록한 워커 ID를 검색한다. 실행 중 필터가 있으면 해제하고 **종료됨(terminated)** 상태인지 확인한다. 인스턴스를 직접 종료하지 않고 Karpenter가 정리하는지 관찰한다. 기본 노드 ID는 계속 **실행 중(running)**이어야 한다.

**통과 기준:** 테스트 외 앱이 없는 경우 확장 워커 Node·NodeClaim 삭제 및 해당 EC2 terminated 확인. 기본 노드는 Ready이며 Karpenter도 계속 실행된다. IAM·AMI·EBS snapshot은 consolidation이 삭제하지 않는다.

## 18. 실패 지점에 따라 진단한다

필요한 이유: provisioning·가입·초기화·이미지 pull은 서로 다른 단계다. 첫 실패 조건을 기준으로 확인하면 정상 설정을 반복 설치하거나 클러스터를 초기화하는 일을 줄일 수 있다.

| 증상 | 우선 확인 |
|---|---|
| NodeClaim 없음 | 파드 Pending 사유, selector·requests, NodePool/NodeClass Ready |
| EC2 생성 실패 | lifecycle·discovery·PassRole, 프로파일, 타입·쿼터·KMS |
| EC2만 있고 Node 없음 | userData, SSM·IMDS, API 6443·DNS, 토큰·CA hash |
| Registered/Initialized 대기 | providerID, 등록 taint, CCM·CNI·실제 kubelet 설정 |
| cloud uninitialized 잔존 | CCM 권한·EC2 태그·노드 이름·로그 |
| ImagePullBackOff | ECR provider·역할·저장소 정책·통신 |
| 유휴 노드 잔존 | 일반 파드·drain/PDB·disruption·종료 권한 |
| CoreDNS Pending | CNI·CCM·taint·자원 부족 |
| crictl 설정 없음 경고 | crictl 기본 설정 파일과 RuntimeReady를 구분 |

**기본 노드 셸**:

```bash
kubectl get events -A --sort-by=.metadata.creationTimestamp
kubectl -n kube-system logs deployment/karpenter --tail=100
kubectl -n kube-system logs daemonset/aws-cloud-controller-manager --tail=100
# 실제 이름으로 실행:
# kubectl describe nodeclaim NODECLAIM_NAME
# kubectl describe node NODE_NAME
# kubectl describe pod POD_NAME
```

**문제가 난 워커 내부**:

```bash
sudo tail -n 100 /var/log/cloud-init-output.log
sudo journalctl -u kubelet --no-pager -n 100
sudo journalctl -u containerd --no-pager -n 100
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock info
```

crictl 설정 파일 경고는 다음 기본값으로 없앨 수 있다. 서비스 재시작은 필요 없다.

```bash
sudo tee /etc/crictl.yaml >/dev/null <<'EOF'
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF
sudo crictl info
```

가입 전 RuntimeReady=true·NetworkReady=false는 가능하다. CNI 설치 후에도 false면 원인을 확인한다. 공유할 로그에 토큰·인증 정보가 들어 있지 않은지 확인한다.

## 완료 판정과 다음 범위

필요한 이유: 문서 작성·매니페스트 적용과 실제 자동 확장 성공은 다르다. 각 단계의 증거를 남겨 GPU·앱 배포의 출발점으로 사용한다.

| 항목 | 완료 증거 |
|---|---|
| 기본 노드 | init 성공·Node Ready·안정적인 사설 API·일반 앱 배치 허용 |
| CNI·CCM·DNS | 파드 정상·cloud 초기화 taint 제거 |
| IAM | CP·워커 역할 분리·인증/권한 오류 없음 |
| 자동 가입 | 새 워커가 SSM 조회 후 고유 신원으로 join |
| Karpenter | NodeClass/NodePool Ready·NodeClaim 준비 조건 정상 |
| CPU 확장 | 기본 노드에서 앱 실행 → 용량 부족 → 추가 EC2·파드 실행 |
| ECR | private pull 검증 또는 미검증 명시 |
| 축소 | 워커·NodeClaim·EC2 정리 |
| 재확장 기반 | 워커 0대에도 CP·Karpenter 정상 |

이후 범위는 앱·KEDA·관측성·ALB 또는 GPU 풀이다. Spot을 추가할 때는 service-linked role·interruption queue·권한·중단 동작을 준비한다. 이 가이드는 단일 CP PoC이며 HA·백업·외부 공개 구성을 완료하지 않는다.

## 문서 검증 범위

필요한 이유: 로컬 문법·설정 생성 검증과 실제 AWS 동작 검증을 구분해야 문서의 통과 기준을 배포 성공으로 오인하지 않는다. 실행 결과는 위 단계별 완료 증거로 남긴다.

- Bash 코드 블록 구문과 내장 Python 코드 구문 검사 통과.
- 공식 Karpenter 템플릿의 두 정책과 제공 JSON의 대응·치환·JSON 파싱 확인.
- YAML 템플릿의 maxPods·예약량·외부 cloud 설정·NodeClass 참조·공통 파드 selector·기본 노드 선호 연결 확인.
- 워커 userData 모의 검증: Karpenter 가입·수동 가입 성공, SSM 오류·잘못된 토큰·가입 실패에서 중단 및 임시 가입 파일 정리 확인.
- userData의 토큰 미출력과 16KB 제한 확인. 실제 생성 예시는 약 2.3KB다.
- 실제 AWS IAM 적용·EC2 생성·kubeadm 실행·Helm OCI 렌더링·클러스터 네트워크 검증은 이 작업 환경에서 수행하지 않았다. 사용 환경에서 각 단계의 통과 기준을 확인해야 한다.

## 공식 근거

필요한 이유: self-managed kubeadm에 맞춰 조합한 구성인 만큼 버전 변경 시 공식 문서·고정 소스와 다시 대조해야 한다. 변경 가능한 최신 문서와 고정 소스를 구분한다.

- [kubeadm v1beta4](https://kubernetes.io/docs/reference/config-api/kubeadm-config.v1beta4/)
- [kubeadm 클러스터 생성](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/)
- [컨트롤 플레인에 일반 파드 허용](https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/create-cluster-kubeadm/#control-plane-node-isolation)
- [IAM 콘솔 역할 생성](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-service.html)
- [EC2 인스턴스 프로파일·콘솔 자동 생성](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html)
- [기존 EC2에 역할 연결](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/attach-iam-role.html)
- [IAM 인라인 정책 입력](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_manage-attach-detach.html)
- [콘솔에서 SSM 파라미터 생성](https://docs.aws.amazon.com/systems-manager/latest/userguide/parameter-create-console.html)
- [EC2 태그 편집](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/Using_Tags.html)
- [public subnet·인터넷 게이트웨이·공인 IPv4 조건](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)
- [EC2 사설·공인 IP와 EIP](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-instance-addressing.html)
- [VPC CNI Pod 인터넷 통신·기본 SNAT](https://docs.aws.amazon.com/eks/latest/userguide/external-snat.html)
- [AWS cloud provider 준비·노드 이름](https://cloud-provider-aws.sigs.k8s.io/prerequisites/)
- [ECR credential provider](https://cloud-provider-aws.sigs.k8s.io/credential_provider/)
- [고정 VPC CNI 매니페스트](https://github.com/aws/amazon-vpc-cni-k8s/blob/e082bae7c03ddc28357b01595f3746b117ac08c7/config/master/aws-k8s-cni.yaml)
- [CNI README·호환성](https://github.com/aws/amazon-vpc-cni-k8s/blob/e082bae7c03ddc28357b01595f3746b117ac08c7/README.md)
- [고정 CCM chart](https://github.com/kubernetes/cloud-provider-aws/tree/1c494fbc3edacf55e2a1e06df21858c03ff9c457/charts/aws-cloud-controller-manager)
- [CCM 노드 정보 구현](https://github.com/kubernetes/cloud-provider-aws/blob/1c494fbc3edacf55e2a1e06df21858c03ff9c457/pkg/providers/v1/instances_v2.go)
- [Karpenter 호환성](https://karpenter.sh/docs/upgrading/compatibility/)
- [Karpenter controller 설정](https://karpenter.sh/docs/reference/settings/)
- [고정 Karpenter IAM 템플릿](https://github.com/aws/karpenter-provider-aws/blob/b2896451034b61e8641b8b891e4a0f10d905608f/website/content/en/docs/getting-started/getting-started-with-karpenter/cloudformation.yaml)
- [Karpenter NodeClass·Custom AMI](https://karpenter.sh/docs/concepts/nodeclasses/)
- [Karpenter NodePool·상한](https://karpenter.sh/docs/concepts/nodepools/)
- [다른 Kubernetes 배포판](https://karpenter.sh/docs/faq/#what-kubernetes-distributions-are-supported)
- [Helm 설치](https://helm.sh/docs/v3/intro/install/)
- [EC2 userData·16KB](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html)
- [crictl 설정](https://github.com/kubernetes-sigs/cri-tools/blob/master/docs/crictl.md)
