# AL2023 CPU AMI 설치 및 확인 가이드

**Amazon Linux 2023·x86_64의 새 EC2 기준**으로, 아래 순서대로 설치하고 확인하면 됩니다. 아직 `kubeadm init`은 실행하지 않습니다.

Kubernetes 버전은 설명을 위해 **1.35 계열**로 잡았습니다. 이미 정한 버전이 있다면 저장소와 ECR provider의 minor 버전을 함께 맞춰야 합니다. 아래 명령은 EC2에서 실행할 안내이며, 해당 인스턴스에서 실행 검증한 결과는 아닙니다.

## 1. OS 확인 및 기본 도구 설치

먼저 환경을 확인합니다.

```bash
cat /etc/os-release
uname -m
```

기대값은 Amazon Linux `2023`, 아키텍처 `x86_64`입니다.

기본 패키지를 설치합니다.

```bash
sudo dnf upgrade -y

sudo dnf install -y \
  ca-certificates \
  cloud-init \
  awscli-2 \
  chrony \
  amazon-chrony-config
```

AL2023 기본 `curl-minimal`로 필요한 HTTP·HTTPS 요청을 처리할 수 있습니다. 전체 curl로 교체할 필요는 없습니다. [AWS curl 패키지 설명](https://docs.aws.amazon.com/linux/al2023/ug/curl-minimal.html)

```bash
command -v curl || sudo dnf install -y curl-minimal
```

설치 확인:

```bash
curl --version
aws --version
cloud-init --version
rpm -q ca-certificates chrony amazon-chrony-config
```

모두 버전이 출력되어야 합니다. AL2023의 AWS CLI 패키지 이름은 **`awscli-2`**입니다. [AWS CLI 안내](https://docs.aws.amazon.com/linux/al2023/ug/awscli2.html)

## 2. swap 비활성화

현재 사용 중인 swap을 끕니다.

```bash
sudo swapoff -a
```

재부팅 후에도 켜지지 않도록 `/etc/fstab`의 swap 항목을 주석 처리합니다. 원본은 `.bak`으로 보관합니다.

```bash
sudo sed -i.bak \
  '/^[^#].*[[:space:]]swap[[:space:]]/s/^/# /' \
  /etc/fstab
```

확인:

```bash
swapon --show
free -h
```

정상 기준:

- `swapon --show`: 출력 없음
- `free -h`: Swap 전체 용량 `0B`

이 절차는 문서의 t3.large처럼 메모리가 충분한 기본 AL2023 인스턴스를 전제로 합니다. 출력에 `zram`이 남아 있다면 해당 swap 생성 설정도 별도로 해제해야 합니다.

## 3. 커널 모듈·네트워크 설정

컨테이너 파일시스템과 네트워크에 사용할 모듈을 설정합니다.

```bash
sudo tee /etc/modules-load.d/kubernetes.conf >/dev/null <<'EOF'
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

IP forwarding과 bridge 트래픽 설정을 적용합니다.

```bash
sudo tee /etc/sysctl.d/99-kubernetes.conf >/dev/null <<'EOF'
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system
```

확인:

```bash
lsmod | grep -E 'overlay|br_netfilter'

sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables
sysctl net.bridge.bridge-nf-call-ip6tables
```

세 sysctl 값이 모두 `1`인지 확인합니다.

SELinux도 현재 상태를 확인합니다.

```bash
getenforce
```

AL2023 기본값은 `Permissive`입니다. 기존에 `Enforcing`으로 변경한 이미지라면 이 안내와 별도로 Kubernetes·CNI에 맞는 정책 구성이 필요합니다. [AL2023 SELinux 기본값](https://docs.aws.amazon.com/linux/al2023/ug/selinux-modes.html)

## 4. containerd 설치 및 cgroup·CRI 설정

설치합니다.

```bash
sudo dnf install -y containerd runc
```

다음 명령은 **새 EC2에서 containerd 기본 설정 파일을 만드는 과정**입니다. 기존 설정이 있는 서버에서는 덮어쓰기 전에 확인해야 합니다.

```bash
sudo mkdir -p /etc/containerd

containerd config default \
  | sudo tee /etc/containerd/config.toml >/dev/null

sudo sed -i \
  's/SystemdCgroup = false/SystemdCgroup = true/' \
  /etc/containerd/config.toml

sudo systemctl enable --now containerd
sudo systemctl restart containerd
```

기본 설정을 생성한 뒤 `SystemdCgroup`을 변경하면 containerd 1.x와 2.x의 설정 경로 차이를 직접 작성할 필요가 줄어듭니다. CRI가 비활성화 목록에 없어야 합니다. [Kubernetes containerd 설정](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

설치·실행 확인:

```bash
containerd --version
runc --version

systemctl is-enabled containerd
systemctl is-active containerd

sudo containerd config dump \
  | grep -E 'disabled_plugins|SystemdCgroup'

sudo ctr plugins ls | grep cri
```

정상 기준:

| 확인 항목 | 기대값 |
|---|---|
| 자동 시작 | `enabled` |
| 현재 실행 | `active` |
| cgroup | `SystemdCgroup = true` |
| 비활성화 목록 | CRI 관련 플러그인이 포함되지 않음 |
| CRI 플러그인 | 관련 항목 상태가 `ok` |

## 5. kubelet·kubeadm·kubectl 설치

같은 터미널에서 아래 명령을 순서대로 실행합니다.

먼저 사용할 minor 버전을 지정합니다.

```bash
K8S_MINOR="v1.35"
```

공식 RPM 저장소를 등록합니다.

```bash
sudo tee /etc/yum.repos.d/kubernetes.repo >/dev/null <<EOF
[kubernetes]
name=Kubernetes
baseurl=https://pkgs.k8s.io/core:/stable:/${K8S_MINOR}/rpm/
enabled=1
gpgcheck=1
gpgkey=https://pkgs.k8s.io/core:/stable:/${K8S_MINOR}/rpm/repodata/repomd.xml.key
EOF
```

해당 minor의 안정 패치 버전을 한 번 조회하고, 세 패키지를 같은 버전으로 설치합니다.

```bash
K8S_VERSION="$(curl -fsSL \
  "https://dl.k8s.io/release/stable-${K8S_MINOR#v}.txt")"

printf '설치할 Kubernetes 버전: %s\n' "$K8S_VERSION"

sudo dnf install -y \
  "kubelet-${K8S_VERSION#v}" \
  "kubeadm-${K8S_VERSION#v}" \
  "kubectl-${K8S_VERSION#v}" \
  cri-tools
```

일반 OS 업데이트에서 Kubernetes 패키지가 함께 올라가지 않도록 제외합니다.

```bash
printf '%s\n' \
  'exclude=kubelet kubeadm kubectl cri-tools kubernetes-cni' \
  | sudo tee -a /etc/yum.repos.d/kubernetes.repo >/dev/null

sudo systemctl enable kubelet
```

이는 Kubernetes 공식 RPM 저장소를 사용하는 구성입니다. [공식 설치 안내](https://v1-35.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)

확인:

```bash
kubelet --version
kubeadm version -o short
kubectl version --client
crictl --version

rpm -q kubelet kubeadm kubectl
systemctl is-enabled kubelet
```

세 Kubernetes 패키지의 버전이 일치하고, kubelet은 `enabled`여야 합니다.

**지금은 kubelet이 `active`일 필요는 없습니다.** `/var/lib/kubelet/config.yaml`과 가입 인증 정보는 이후 `init/join` 과정에서 만들어집니다.

kubeadm은 기본적으로 kubelet의 cgroup driver를 `systemd`로 설정합니다. 이후 `init/join` 때 별도로 다른 값으로 덮어쓰지 않으면 됩니다.

containerd의 CRI 응답도 확인합니다.

```bash
sudo crictl \
  --runtime-endpoint unix:///run/containerd/containerd.sock \
  info
```

출력에서 `RuntimeReady`가 `true`인지 확인합니다. **CNI 설치 전에는 `NetworkReady`가 `false`여도 예상된 상태**입니다.

## 6. ECR credential provider 설치

여기서는 **공식 릴리스 소스를 고정해 빌드하는 방법**을 사용합니다. Kubernetes 1.35 계열에 맞춰 provider는 `v1.35.2`를 예시로 사용합니다.

먼저 빌드 도구를 설치합니다. 이 도구들은 provider를 만드는 데 필요하며, 실행 시 필수 프로그램은 아닙니다.

```bash
sudo dnf install -y git make golang
```

공식 소스를 받아 빌드합니다.

```bash
ECR_BUILD_DIR="$(mktemp -d)"

git clone --depth 1 --branch v1.35.2 \
  https://github.com/kubernetes/cloud-provider-aws.git \
  "$ECR_BUILD_DIR/cloud-provider-aws"

(
  cd "$ECR_BUILD_DIR/cloud-provider-aws"

  GOTOOLCHAIN=auto make ecr-credential-provider

  sudo install -d -m 0755 \
    /usr/local/libexec/kubelet-credential-providers

  sudo install -m 0755 ecr-credential-provider \
    /usr/local/libexec/kubelet-credential-providers/ecr-credential-provider
)
```

해당 릴리스는 Go 1.25 이상을 요구합니다. `GOTOOLCHAIN=auto`는 필요한 Go toolchain을 다운로드할 수 있으므로 빌드 중 인터넷 연결이 필요합니다. [공식 빌드 정의](https://github.com/kubernetes/cloud-provider-aws/blob/v1.35.2/Makefile), [Go 버전 요구사항](https://github.com/kubernetes/cloud-provider-aws/blob/v1.35.2/go.mod)

바이너리 확인:

```bash
/usr/local/libexec/kubelet-credential-providers/ecr-credential-provider \
  --version
```

다음으로 provider 설정을 만듭니다.

```bash
sudo mkdir -p /etc/kubernetes

sudo tee /etc/kubernetes/ecr-credential-provider.yaml >/dev/null <<'EOF'
apiVersion: kubelet.config.k8s.io/v1
kind: CredentialProviderConfig
providers:
  - name: ecr-credential-provider
    matchImages:
      - "*.dkr.ecr.*.amazonaws.com"
      - "*.dkr.ecr.*.amazonaws.com.cn"
    defaultCacheDuration: "12h"
    apiVersion: credentialprovider.kubelet.k8s.io/v1
EOF
```

kubelet이 실행 파일과 설정을 찾도록 연결합니다. **새 EC2라 기존 추가 인수가 없다는 전제**입니다.

```bash
sudo tee /etc/sysconfig/kubelet >/dev/null <<'EOF'
KUBELET_EXTRA_ARGS="--image-credential-provider-bin-dir=/usr/local/libexec/kubelet-credential-providers --image-credential-provider-config=/etc/kubernetes/ecr-credential-provider.yaml"
EOF

sudo systemctl daemon-reload
```

RPM 방식의 kubelet은 `/etc/sysconfig/kubelet`을 읽습니다. 이후 userData에서 이 파일을 덮어쓰지 않도록 해야 합니다. [kubelet systemd 설정](https://github.com/kubernetes/release/blob/master/cmd/krel/templates/latest/kubeadm/10-kubeadm.conf)

연결 확인:

```bash
test -x \
  /usr/local/libexec/kubelet-credential-providers/ecr-credential-provider \
  && echo "ECR provider 실행 파일 확인"

cat /etc/kubernetes/ecr-credential-provider.yaml
cat /etc/sysconfig/kubelet

systemctl cat kubelet
```

`systemctl cat kubelet`에서 다음이 있는지 확인합니다.

```text
EnvironmentFile=-/etc/sysconfig/kubelet
ExecStart=... $KUBELET_EXTRA_ARGS
```

**이 단계는 설치·설정 확인입니다.** 실제 ECR 이미지 다운로드 성공 여부는 EC2 역할의 ECR 권한을 연결하고, `init/join` 후 private ECR 이미지를 사용하는 파드로 검증합니다. provider는 EC2 역할을 사용하므로 별도 ECR 로그인 스크립트는 필요하지 않습니다. [AWS provider 동작](https://cloud-provider-aws.sigs.k8s.io/credential_provider/)

## 7. AWS CLI·cloud-init 동작 확인

AWS CLI는 1번에서 설치했습니다.

```bash
aws --version
```

`aws-cli/2...`가 출력되어야 합니다.

EC2에 IAM 역할을 연결했다면 인증도 확인할 수 있습니다.

```bash
aws sts get-caller-identity
```

현재 EC2 역할의 ARN이 나오는지 확인합니다. 정적 액세스 키를 저장하는 `aws configure`는 필요하지 않습니다.

cloud-init 확인:

```bash
cloud-init --version
sudo cloud-init status --wait
```

현재 부팅의 처리가 끝났다면 `status: done`이 기대값입니다. 오류가 있다면 확인합니다.

```bash
sudo journalctl -u cloud-final --no-pager -n 50
```

AL2023의 cloud-init이 이후 새 워커의 userData를 실행합니다. 지금은 `cloud-init clean`을 실행하지 않고 설치·완료 상태만 확인합니다. [AL2023 cloud-init 안내](https://docs.aws.amazon.com/linux/al2023/ug/cloud-init.html)

## 8. 시간 동기화 활성화

1번에서 설치한 chrony를 활성화합니다.

```bash
sudo systemctl enable --now chronyd
```

확인:

```bash
systemctl is-enabled chronyd
systemctl is-active chronyd

chronyc sources -v
chronyc tracking
```

정상 기준:

- 서비스: `enabled`, `active`
- `sources`: 선택된 시간 소스에 `^*` 표시
- `tracking`: `Leap status: Normal`

기본 AL2023는 Amazon Time Sync를 사용하도록 구성되어 있습니다. 일반적인 IPv4 시간 소스는 `169.254.169.123`입니다. [AWS 시간 동기화 안내](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configure-ec2-ntp.html)

## 9. 재부팅 후 최종 확인

OS 업데이트와 영구 설정 반영을 확인하기 위해 재부팅합니다.

```bash
sudo reboot
```

다시 접속한 뒤 확인합니다.

```bash
swapon --show
sysctl net.ipv4.ip_forward

systemctl is-active containerd
systemctl is-enabled containerd
systemctl is-enabled kubelet
systemctl is-active chronyd

kubelet --version
kubeadm version -o short
kubectl version --client

sudo crictl \
  --runtime-endpoint unix:///run/containerd/containerd.sock \
  info
```

| 항목 | AMI 제작 전 통과 기준 |
|---|---|
| swap | 사용 중인 swap 없음 |
| 네트워크 설정 | IP forwarding 등 설정 유지 |
| containerd | 자동 시작·실행 정상, CRI `RuntimeReady: true` |
| Kubernetes | 세 도구 버전 일치, kubelet 자동 시작 설정 |
| ECR provider | 실행 파일·설정·kubelet 연결 확인 |
| AWS CLI | v2 실행 가능 |
| cloud-init | 현재 부팅 처리 완료 |
| 시간 동기화 | chronyd 실행·동기화 정상 |
| 클러스터 상태 | 아직 `kubeadm init/join` 미실행 |

여기까지가 **공통 설치·설정 검증 범위**입니다. AMI 캡처 직전에는 이미지 일반화와 비밀 정보 잔존 여부를 확인하고, AMI 생성 후 원래 EC2에서 `kubeadm init`을 진행하면 됩니다.
