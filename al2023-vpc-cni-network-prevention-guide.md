# AL2023 VPC CNI의 Pod IP 중복 등록 방지

## 문제 상황

### 1. Pod IP 중복 등록으로 CoreDNS 장애

CoreDNS가 API·DNS에 연결하지 못하고, health probe 실패로 CrashLoopBackOff가 반복됐다.

```text
노드 IP: 10.0.4.232
CoreDNS Pod IP: 10.0.1.107
문제 경로: local 10.0.1.107 dev lo
```

Pod IP가 호스트 ens5에도 붙어 있었다. 해당 호스트 주소만 제거하자 경로가 `dev eni...`로 바뀌고, 같은 Pod의 health·ready가 모두 OK가 됐다.

### 2. CNI 라우팅 규칙 누락으로 Argo CD DNS 장애

CoreDNS가 Running·Ready인 상태에서도 일부 Argo CD Pod에서 DNS 조회가 타임아웃됐다. `argocd-repo-server`와 `argocd-redis`를 찾지 못해 manifest 생성·Redis 연결이 실패했고, 일부 Pod를 재시작해도 다른 Pod의 오류는 남았다.

```text
lookup argocd-repo-server on 10.96.0.10:53: dial udp 10.96.0.10:53: i/o timeout
lookup argocd-redis: i/o timeout
```

당시 호스트에 Pod IP 중복은 없었지만, `ip -4 rule show`에서 기존 Pod의 `512` 목적지 규칙과 보조 ENI Pod의 `1536` 출발지 규칙이 누락돼 있었다. 라우팅 보존 설정 적용·노드 재부팅 후 CNI 규칙과 `argocd-repo-server` DNS 조회가 복구됐다.

## 원인

AL2023의 `amazon-ec2-net-utils 2.7.7`이 EC2 보조 IP를 호스트 주소로 자동 등록했다.

```text
/run/systemd/network/70-ens5.network.d/ec2net_alias.conf
Address=10.0.1.107/32
```

VPC CNI가 Pod에 할당한 IP를 OS도 자기 주소로 사용하면서, health 요청이 Pod 대신 호스트로 전달됐다. 또한 networkd가 CNI의 경로·규칙을 지울 수 있어 관리 역할을 분리해야 한다. [주소 자동 등록 구현](https://github.com/amazonlinux/amazon-ec2-net-utils/blob/v2.7.7/lib/lib.sh#L246)

## 해결

### 1. 현재 장애 복구

필요한 이유: 이미 호스트에 붙은 Pod IP는 예방 설정만 작성해도 즉시 제거되지 않는다.

`kubectl get pods -n kube-system -o wide`와 `ip -o -4 addr show`로 **현재 Pod IP와 호스트의 중복 주소**를 확인한다. 다음은 `10.0.8.58/32`가 실제로 ens5에 중복 등록된 경우의 예시다.

```bash
sudo ip address del 10.0.8.58/32 dev ens5
ip -4 route get 10.0.8.58
```

기본 IP `10.0.4.232`는 유지한다. AWS 콘솔에서 ENI의 IP 할당을 해제하는 작업이 아니다. 주소 수동 삭제와 기존 ens5 alias 마스크는 임시 대응이며, 새 Pod·ENI까지 보호하지 못한다.

#### Troubleshooting: 주소 중복과 라우팅 규칙 누락 구분

두 장애는 모두 OS 네트워크 관리와 VPC CNI의 간섭과 관련되지만, 확인 방법과 예방 설정은 다르다.

| 구분 | 확인할 증상·근거 | 예방 설정 |
| --- | --- | --- |
| Pod IP의 호스트 중복 등록 | `ip -o -4 addr show`에 현재 Pod IP가 ens5 주소로 나오고, `ip -4 route get <Pod IP>`가 `local … dev lo`로 나옴. 당시 `ec2net_alias.conf`의 `Address=`에서도 확인됨 | `10-kubernetes-primary.network`로 기존 ens5 alias 설정이 적용되지 않게 함 |
| CNI 라우팅 규칙 누락 | 호스트에 Pod IP 중복은 없지만 `ip -4 rule show`에서 현재 Pod 목적지의 `512 … lookup main` 규칙이 누락됨. 보조 ENI의 Pod는 출발지 `1536` 규칙도 확인 | `ManageForeignRoutes=no`, `ManageForeignRoutingPolicyRules=no`로 CNI의 경로·규칙을 보존 |

이번 Argo CD DNS 장애에서는 CNI 규칙 누락이 확인됐고, 설정 적용·재부팅 후 규칙과 DNS 조회가 복구됐다. networkd의 외부 규칙 정리가 유력한 원인이지만, 실제 삭제 주체는 로그로 확정하지 않았다. 규칙의 역할은 [VPC CNI 구현](https://github.com/aws/amazon-vpc-cni-k8s/blob/v1.21.1/cmd/routed-eni-cni-plugin/driver/driver.go), 보존 옵션은 [systemd 문서](https://www.freedesktop.org/software/systemd/man/252/networkd.conf.html)를 참고한다.

**라우팅 보존 옵션 두 개만으로 Pod IP 중복 등록까지 예방되지는 않는다.** 두 종류의 예방 설정이 모두 필요하다. 또한 AWS 콘솔의 ENI Secondary private IP 목록에 Pod IP가 보이는 것은 정상이며, 이를 할당 해제하면 안 된다.

설정 파일은 이미 사라진 규칙을 즉시 복원하지 않는다. 현재 노드는 보존 설정을 적용한 뒤 재부팅하여 Pod 네트워크가 다시 구성되면, **변경된 현재 Pod IP** 기준으로 경로·규칙과 DNS 조회를 재확인한다. 단일 컨트롤 플레인 겸 워커이므로 재부팅 중에는 서비스가 중단된다. 아래 AMI 제작용 명령 전체를 현재 클러스터에 실행하지 않는다.

### 2. AMI에 재발 방지 설정

필요한 이유: ens5의 호스트 기본 IP·게이트웨이·DNS는 OS가 관리하고, Pod용 보조 IP·추가 ENI·경로는 CNI가 관리하도록 분리해야 한다.

**AMI에 CNI가 없어도 적용할 수 있다.** 아래 파일은 AL2023의 `systemd-networkd` 설정이다. AMI에는 이 OS 설정만 저장하고, CNI는 클러스터 초기화 후 DaemonSet으로 설치한다. 이후 CNI가 추가 ENI·Pod IP를 만들 때 OS가 중복으로 관리하지 않게 하는 설정이다.

**대상: IPv4·ENA 드라이버·기본 ENI가 ens5인 CPU 노드.** 다른 인터페이스 이름에는 그대로 적용하지 않는다. IP·MAC은 이미지에 고정하지 않는다.

**ens5를 DHCP로 관리해도 괜찮다.** `DHCP=ipv4`는 호스트 주소·기본 경로·DNS를 EC2의 DHCP에서 받는다는 뜻이다. ens5는 계속 노드·API·SSM·ECR·인터넷 통신에 사용되며, DHCP 트래픽 전용으로 제한되지 않는다. EC2의 기본 사설 IP도 인스턴스에 할당된 주소를 그대로 받는다.

VPC CNI는 **ens5의 Pod용 보조 IP도 계속 사용**할 수 있다. AWS ENI의 IP 할당은 유지하고 OS에 동일 주소를 추가하지 않는 것이 목적이다. `DHCP=ipv4` 하나가 중복 등록을 막는 것이 아니라, 새 10번 설정이 기존 70번 alias 설정보다 먼저 선택되는 점이 핵심이다. [VPC CNI의 ENI 사용](https://github.com/aws/amazon-vpc-cni-k8s/blob/v1.21.1/README.md#eni-allocation)

| 설정 파일                                                 | 역할                                                                        |
| ----------------------------------------------------- | ------------------------------------------------------------------------- |
| `/etc/systemd/network/10-kubernetes-primary.network`  | ens5의 호스트 기본 IP·경로·DNS를 DHCP로 관리. Pod 보조 IP는 OS 주소로 중복 등록하지 않음 |
| `/etc/systemd/network/20-kubernetes-cni-eni.network`  | 나머지 ENA ENI를 `Unmanaged=yes`로 두어 CNI에 맡김                                  |
| `/etc/systemd/networkd.conf.d/90-kubernetes-cni.conf` | `ManageForeignRoutes=no`, `ManageForeignRoutingPolicyRules=no`로 CNI 경로 보존 |

systemd는 먼저 일치하는 .network 파일을 사용하므로, ens5에는 10번 설정이 적용되고 기존 70번 alias 설정은 적용되지 않는다. [선택 규칙](https://www.freedesktop.org/software/systemd/man/252/systemd.network.html), [경로 보존 옵션](https://www.freedesktop.org/software/systemd/man/252/networkd.conf.html)

**init/join 이전의 제작용 EC2에 SSH 또는 Session Manager로 접속**하고, 아래 명령을 터미널에 직접 붙여 넣는다. `.sh` 파일을 만들거나 다운로드할 필요 없다. 사전 검사가 성공하면 `tee`로 설정 파일 세 개를 생성한다. 기존 클러스터 흔적·잘못된 NIC·기존 설정이 있으면 파일 생성 전에 중단한다.

```bash
sudo bash <<'SETUP'
set -euo pipefail

test ! -e /etc/kubernetes/admin.conf
test ! -e /etc/kubernetes/kubelet.conf
test ! -e /etc/kubernetes/pki/ca.key
test ! -d /var/lib/etcd/member

IMDS_TOKEN="$(curl -fsS --retry 5 --connect-timeout 3 --max-time 15 \
  -X PUT -H 'X-aws-ec2-metadata-token-ttl-seconds: 300' \
  http://169.254.169.254/latest/api/token)"
PRIMARY_MAC="$(curl -fsS --retry 5 --connect-timeout 3 --max-time 15 \
  -H "X-aws-ec2-metadata-token: $IMDS_TOKEN" \
  http://169.254.169.254/latest/meta-data/mac)"

test -r /sys/class/net/ens5/address
test "$(cat /sys/class/net/ens5/address)" = "$PRIMARY_MAC"
test "$(basename "$(readlink -f /sys/class/net/ens5/device/driver)")" = ena

for file in \
  /etc/systemd/network/10-kubernetes-primary.network \
  /etc/systemd/network/20-kubernetes-cni-eni.network \
  /etc/systemd/networkd.conf.d/90-kubernetes-cni.conf; do
  if [[ -e "$file" || -L "$file" ]]; then
    echo "기존 설정을 먼저 확인하세요: $file"
    exit 1
  fi
done

mkdir -p /etc/systemd/network /etc/systemd/networkd.conf.d

tee /etc/systemd/network/10-kubernetes-primary.network >/dev/null <<'PRIMARY'
[Match]
Name=ens5
Driver=ena

[Link]
MTUBytes=9001

[Network]
DHCP=ipv4
LLMNR=no
DNSDefaultRoute=yes

[DHCPv4]
UseHostname=no
UseDNS=yes
UseNTP=yes
UseDomains=yes
UseRoutes=yes
RouteMetric=512
PRIMARY

tee /etc/systemd/network/20-kubernetes-cni-eni.network >/dev/null <<'SECONDARY'
[Match]
Driver=ena

[Link]
Unmanaged=yes
SECONDARY

tee /etc/systemd/networkd.conf.d/90-kubernetes-cni.conf >/dev/null <<'NETWORKD'
[Network]
ManageForeignRoutes=no
ManageForeignRoutingPolicyRules=no
NETWORKD

chmod 0644 \
  /etc/systemd/network/10-kubernetes-primary.network \
  /etc/systemd/network/20-kubernetes-cni-eni.network \
  /etc/systemd/networkd.conf.d/90-kubernetes-cni.conf
echo "설정 파일 세 개 생성 완료. 다음 단계에서 재부팅 후 확인하세요."
SETUP
```

### 3. 검증 후 적용

필요한 이유: 파일 작성만으로 재부팅·새 ENI에서의 정상 동작이 입증되지는 않는다.

1. **기존 init 이전 AMI**로 제작용 EC2를 시작한다. 사용자 데이터는 비워 두고 접속해 위 파일 생성 명령을 실행한다. 이미 설정이 있는 경우 자동 덮어쓰기하지 않고 내용을 먼저 확인한다.
2. 아래 재부팅 명령을 실행한 뒤 재접속해 검증한다. ens5는 10번 설정을 사용하고, DHCP·기본 경로·DNS·SSH/Session Manager 접속이 정상이어야 한다.
3. 전체 패키지 설치까지 끝낸 뒤 **EC2 콘솔 → 작업 → 이미지 및 템플릿 → 이미지 생성**으로 새 AMI를 만든다. 기존 컨트롤 플레인을 워커 AMI로 캡처하지 않는다.
4. 새 AMI의 시험 워커에서 Pod·추가 ENI·재부팅을 검증한다. 추가 ENI는 unmanaged, Pod 경로는 `dev eni...`여야 하며 Pod IP가 호스트 ENI에 붙으면 안 된다.
5. 검증 후 NodeClass의 `amiSelectorTerms.id`와 `settings.env`의 `CPU_AMI_ID`를 갱신한다. 기존 Karpenter 워커의 drift·교체 상태도 확인한다.

```bash
sudo reboot
```

재접속 후 확인한다.

```bash
networkctl status ens5 --no-pager
ip -br -4 addr
ip -4 route show default
resolvectl status ens5
systemd-analyze cat-config systemd/networkd.conf
```

**현재 기본 노드는 별도 유지보수가 필요하다.** 새 AMI로 자동 수정되지 않으며, 위 명령도 기존 클러스터에서는 실행을 차단한다. 설정 파일 작성 중에도 자동 reload로 연결이 끊길 수 있으므로, 활성 설정 확인·백업·복구 접속 수단·중단 시간을 확보한 뒤 전환한다.

로컬에서는 명령 구문·설정 내용·사전 검사 유지를 검증했다. 실제 EC2의 새 AMI·추가 ENI·재부팅 검증은 위 절차에서 별도로 수행한다.
