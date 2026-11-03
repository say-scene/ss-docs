# AL2023 VPC CNI의 Pod IP 중복 등록·라우팅 규칙 삭제 방지

## 문제 상황

- **Pod IP 중복 등록:** VPC CNI의 Pod IP가 호스트 `ens5`에도 `/32`로 붙어 `ip route get <Pod IP>`가 `local … dev lo`로 나왔다. CoreDNS의 API·DNS 연결과 health probe가 실패했다.
- **추가 ENI에서 재발:** ens5에만 alias 마스크가 있어, 나중에 추가된 ens6·ens7은 보호되지 않았다. 이번에는 ens7의 중복 `/32`를 제거하고, 현재 ENI뿐 아니라 앞으로 추가될 ENI에도 마스크를 적용하도록 수정했다.
- **별개 장애인 CNI 규칙 누락:** 호스트 주소 중복이 없어도 Pod 목적지 `512`·보조 ENI Pod 출발지 `1536` 규칙이 누락되면 Argo CD에서 `argocd-repo-server`, `argocd-redis` 등의 DNS 조회가 타임아웃됐다.

사용자가 보고한 현재 복구 결과: ens5·ens6·ens7에는 각각 원래 DHCP 주소 `10.0.4.232/20`, `10.0.11.176/20`, `10.0.11.240/20`만 남았다. networkd restart 없이 reload·ens7 재적용으로 확인했고, 기존 라우팅 보존 옵션과 SNAT 설정은 유지했다. 아래 주소·NIC 이름은 이 호스트의 사례이며 새 AMI에 고정하지 않는다.

## 원인

`amazon-ec2-net-utils`가 EC2 ENI의 secondary IP를 다음 runtime drop-in으로 생성하면, systemd-networkd가 Pod IP를 호스트 주소로도 등록할 수 있다.

**runtime drop-in은 쉽게 말해 “실행 중에 자동으로 만들어지는 추가 설정 파일”이다.** `runtime`은 `/run`에 저장되어 재부팅하면 지워지고 필요할 때 다시 생성된다는 뜻이고, `drop-in`은 기본 설정 파일을 직접 고치지 않고 보충 설정을 추가하는 방식이다.

```text
/run/systemd/network/70-ens7.network.d/ec2net_alias.conf
Address=<Pod IP>/32
```

위 `Address=`는 **“이 IP를 ens7의 주소로도 등록하라”**는 뜻이다. Pod가 쓰는 IP를 호스트에도 등록하면서 중복이 발생한다. 아래에서 만드는 `/etc`의 마스크는 **“이 추가 설정을 무시하라”**는 표시로, `/run`에 파일이 다시 생겨도 적용되지 않게 한다.

[net-utils 2.7.7의 alias 생성 구현](https://github.com/amazonlinux/amazon-ec2-net-utils/blob/8b699892b1551d5615d17ab39ab52a3070bfe1f5/lib/lib.sh#L264-L289)에서 확인되는 동작이다.

**ens5 마스크만으로는 새 ENI를 보호하지 못한다.** 각 NIC에 대응하는 `/etc`의 같은 이름 drop-in을 `/dev/null`로 마스크해야 한다. `/etc` drop-in은 `/run`보다 우선한다. [systemd 설정 우선순위](https://www.freedesktop.org/software/systemd/man/252/systemd.network.html)

또한 networkd가 자신이 만들지 않은 CNI 경로·규칙을 정리하지 않도록 별도 보존 옵션이 필요하다. **주소 중복 방지와 라우팅 보존은 서로 다른 설정이다.** 실제 과거 규칙 삭제 주체는 로그로 확정하지 않았으므로, 모든 DNS 장애를 networkd 때문이라고 단정하지 않는다.

## 해결

### 이 문서의 목표: ENI가 하나인 AMI로도 모든 ens를 자동 보호

**이 문서는 AMI 제작용이다.** 제작용 EC2에 ens5 하나만 있어도 된다. AMI에서 만든 새 노드에는 **가드 설치·수동 실행·NIC별 마스크 생성용 userData가 필요 없어야 한다.** 기존 운영 노드 보완 절차는 1번으로 구분한다.

AMI에는 **가드 실행 파일 + 초기·갱신 서비스의 템플릿 drop-in + 경로 보존 설정**을 저장한다. ens5 마스크 하나만 저장하는 방식이 아니다. 새 노드에서 가드는 사람이 아니라 **기존 udev·systemd가 자동 실행**한다.

```text
제작용 EC2: ens5 하나
  └─ 가드 + 두 @.service.d 설정 + 경로 보존 설정을 설치하고 AMI 생성

그 AMI에서 노드 시작 / 나중에 ENI 추가
  └─ ens6 등 NIC add 이벤트
      └─ 기존 udev가 policy-routes@ens6.service 시작
          ├─ ExecStartPre: 가드 ens6 → ens6의 /etc alias 마스크 생성
          └─ ExecStart: vendor NIC 설정 실행

이후 주기적 IP 갱신
  └─ refresh-policy-routes@ens6.service
      ├─ ExecStartPre: 가드 ens6 → 마스크 재확인
      └─ ExecStart: vendor 갱신 실행
```

**`@.service.d`는 NIC 이름을 고정하지 않는 템플릿 설정**이다. 부팅 때부터 여러 ENI가 있는 경우와 나중에 추가되는 경우 모두 같은 진입점을 사용한다. 반대로 `policy-routes@ens5.service.d`처럼 ens5 전용으로 만들거나, 현재 NIC를 도는 반복문만 실행하면 미래 NIC 보호가 완성되지 않는다. 이 문서의 범위는 위 진입점을 사용하는 `ens숫자` NIC이며 다른 NIC 이름·패키지 경로는 2번에서 판정한다.

### 실행 경로부터 선택

대상은 **AL2023·IPv4·`ens숫자` 이름의 ENA NIC**이며 CPU·GPU에 공통 적용한다. 명령은 해당 EC2에 SSH 또는 Session Manager로 접속해 실행한다. `REPLACE_...`는 실제 값으로 바꾸며, 한 블록의 변수 선언과 명령은 함께 실행한다.

| 대상 | 실행 순서 |
| --- | --- |
| 기존 설정이 있는 현재 노드 | **1 → 2~6**. 이미 정상인 설정은 각 단계의 확인만 하고 보존한다. 7번 AMI 생성은 하지 않는다. 8번의 통신 확인은 수행하되 ENI 부하 시험은 별도 시험 워커에서 한다. |
| 새로운 AMI 제작용 EC2 | **2 → 3 → 4 → 5 → 6 → 7**. init/join 전 상태를 유지한다. 새 AMI로 만든 별도 시험 워커에서 **8 → 9**를 수행한다. |

**모든 단계에서 기본 IP·EIP·AWS ENI의 secondary IP 할당·VPC route table·SG·SNAT은 변경하지 않는다.** 호스트에 잘못 붙은 Pod `/32`만 6번에서 선택적으로 제거한다. 운영 노드의 설정 변경·재부팅에는 복구 접속 수단과 중단 시간이 필요하다.

### 1. 기존 노드 전용: 백업하고 유지·전환 대상을 결정

**이유:** 기존 설정을 모두 지우고 재설치하면 정상 DHCP·CNI 설정까지 잃을 수 있다. 현재 상태를 보존하고, 이전 문서 방식에서 옮겨야 할 파일만 구분한다. **새 제작용 EC2는 이 단계를 건너뛰고 2번으로 간다.**

#### 1-1. 복구 접속 수단과 백업 확보

현재 SSH 외에 실제 접속 가능한 Session Manager 등 복구 수단을 확인한다. 네트워크 자체가 끊기면 SSM도 실패할 수 있으므로, EC2 Serial Console은 해당 인스턴스·계정에서 사용 가능하고 OS 로그인도 준비된 경우에만 복구 수단으로 간주한다. 둘 다 없다면 **운영 노드의 전환·재부팅은 보류**하고 먼저 복구 수단을 준비한다. 단일 컨트롤 플레인을 재부팅하면 API와 서비스가 중단된다.

```bash
sudo bash <<'BACKUP'
set -euo pipefail
backup="$(mktemp -d /root/sayscene-network-before.XXXXXX)"
for path in \
  /etc/systemd/network \
  /etc/systemd/networkd.conf \
  /etc/systemd/networkd.conf.d \
  /etc/systemd/system/policy-routes@.service.d \
  /etc/systemd/system/refresh-policy-routes@.service.d \
  /usr/local/sbin/sayscene-cni-alias-guard; do
  if [[ -e "$path" || -L "$path" ]]; then
    cp -a --parents "$path" "$backup/"
  fi
done
ip -o -4 addr show > "$backup/addresses.txt"
ip -4 route show table all > "$backup/routes.txt"
ip -4 rule show > "$backup/rules.txt"
systemctl show systemd-networkd -p MainPID > "$backup/networkd-pid.txt"
printf '백업 완료: %s\n' "$backup"
BACKUP
```

**정상:** 마지막에 `/root/sayscene-network-before.XXXXXX` 형태의 실제 경로가 출력된다. 이 경로를 기록한다. **오류:** 출력이 없거나 복사 실패가 있으면 `sudo df -h /root`로 공간과 오류에 나온 파일을 확인해 해결한 뒤 다시 실행한다. 백업이 완료되기 전에는 다음 변경을 하지 않는다.

#### 1-2. 기존 파일·실제 적용 상태 확인

```bash
sudo ls -l /etc/systemd/network/
networkctl status ens5 --no-pager
sudo find /etc/systemd/network -name ec2net_alias.conf -exec ls -l {} \;
sudo ls -l /usr/local/sbin/sayscene-cni-alias-guard
systemctl cat policy-routes@.service refresh-policy-routes@.service
systemd-analyze cat-config systemd/networkd.conf
ip -br -4 addr
```

가드나 마스크가 없어 `No such file`이 나오는 것은 **설치가 필요한 상태**이지, 기존 설정을 전부 지워야 한다는 뜻이 아니다. 다음 표대로 진행한다.

| 확인 결과 | 이 노드에서 할 조치 |
| --- | --- |
| `/etc/.../ec2net_alias.conf → /dev/null` | **유지한다.** 3~4번에서 반복 실행 시 변경 시각이 유지되는지 확인한다. |
| ens5 마스크만 있고 다른 ens에는 없음 | ens5는 유지하고 **3~4번으로 누락 NIC와 미래 NIC 보호를 추가**한다. 아직 주소를 삭제하지 않는다. |
| 가드·두 서비스 훅이 이미 있고 모든 ens 마스크 정상 | 다시 만들지 않는다. **2~5번의 확인·판정 항목만 수행**하고 6번에서 현재 주소·통신을 확인한다. |
| `10-kubernetes-primary.network` 또는 `20-kubernetes-cni-eni.network`가 있음 | 이전 문서의 DHCP override / `Unmanaged=yes` 방식이다. `sudo cat /etc/systemd/network/파일명`으로 내용을 확인한다. **지금 삭제하지 않고 3~5번 보호 장치를 먼저 설치한 뒤 6-B에서 백업 이동·재부팅**한다. |
| 위 두 파일에 다른 용도의 설정도 섞임, 또는 다른 `.network`가 먼저 매칭됨 | 자동 이동 대상이 아니다. 실제 파일 경로를 `networkctl status <NIC>`의 `Network File`에서 확인한다. 해당 파일을 백업하고 필요한 DHCP·MTU·라우팅 설정을 vendor 구성과 비교해 별도로 이관한 뒤 6-B로 진행한다. **이관 전에는 운영 노드를 전환하지 않는다.** |
| alias 위치에 `/dev/null`이 아닌 일반 파일·다른 링크가 있음 | 덮어쓰지 않는다. **3번의 마스크 충돌 처리**에 따라 내용을 확인·백업 이동한 뒤 가드를 다시 실행한다. |
| 보존 옵션이 둘 다 `no` | 기존 파일을 유지한다. 하나라도 `yes`이거나 옵션이 없으면 **5번에서 수정 후 6-B에서 재부팅**한다. |

### 2. 공통: 패키지·NIC·vendor 실행 경로 확인

**이유:** 서비스 이름이나 alias 생성 경로가 다른 환경에 가드를 붙여도 효과가 없다. 새 노드는 여기서 시작하고, 기존 노드는 1번의 백업 후 진행한다.

```bash
rpm -q amazon-ec2-net-utils
ip -br -4 addr
readlink -f /sys/class/net/ens5/device/driver
systemctl cat policy-routes@.service refresh-policy-routes@.service
sudo cat /usr/lib/udev/rules.d/99-vpc-policy-routes.rules
```

| 정상 기준 | 다르면 할 조치 |
| --- | --- |
| `amazon-ec2-net-utils` 설치됨. 이 장애에서 확인한 버전은 `2.7.7` | 미설치 새 EC2는 `sudo dnf install -y amazon-ec2-net-utils` 후 다시 확인한다. 기존 운영 노드의 설치·업그레이드는 1번 백업과 유지보수 계획 후 한다. 설치 실패 시 저장소·인터넷 연결부터 해결한다. |
| 기본 NIC가 ens5이고 driver 경로 끝이 `/ena` | 이름·드라이버가 다르면 이 가드를 그대로 적용하지 않는다. 콘솔 ENI의 MAC과 `ip link`를 대조해 실제 이름을 확인한다. `eth0`, `enp...` 등은 이 가드의 대상이 아니므로 해당 환경용 이름 조건을 검토해야 한다. |
| 초기 서비스는 `setup-policy-routes %i start`, 갱신 서비스는 `%i refresh` 실행 | 서비스가 없거나 다른 명령이면 `rpm -ql amazon-ec2-net-utils`로 설치 파일을 찾고 `rpm -V amazon-ec2-net-utils`로 vendor 파일 변경 여부를 확인한다. 변경된 파일은 백업본과 비교해 원래 패키지 구성으로 복구하거나 해당 버전에 맞는 훅을 검토한다. **없는 서비스 이름으로 계속 진행하지 않는다.** |
| udev NIC add 경로가 `policy-routes@$name.service`를 시작 | 경로가 다르면 실제 호출 서비스에 대한 검증 전까지 진행하지 않는다. 직접 vendor 스크립트를 실행하는 구조는 아래 두 훅만으로 보호되지 않는다. |

v2.7.7의 [udev 규칙](https://github.com/amazonlinux/amazon-ec2-net-utils/blob/8b699892b1551d5615d17ab39ab52a3070bfe1f5/udev/99-vpc-policy-routes.rules), [초기 서비스](https://github.com/amazonlinux/amazon-ec2-net-utils/blob/8b699892b1551d5615d17ab39ab52a3070bfe1f5/systemd/system/policy-routes%40.service), [갱신 서비스](https://github.com/amazonlinux/amazon-ec2-net-utils/blob/8b699892b1551d5615d17ab39ab52a3070bfe1f5/systemd/system/refresh-policy-routes%40.service)를 기준으로 한다. 다른 버전도 세 경로가 같은지 확인한다. vendor 스크립트·`lib.sh`·udev 규칙을 이 문서에서 수정하지 않는다.

### 3. 공통: alias 마스크 가드 설치 또는 기존 가드 보존

**이유:** NIC 이름을 받아 마스크를 만드는 공통 실행 파일이 있어야 ens5뿐 아니라 앞으로 추가되는 ens도 보호된다. 가드는 Pod IP·DHCP·SNAT을 변경하지 않는다.

#### 3-1. 설치 후보 파일 작성

아래는 EC2에 직접 파일을 작성하는 명령이다. **실제 실행 중인 기존 가드는 아직 덮어쓰지 않는다.**

```bash
sudo bash <<'WRITE_CANDIDATE'
set -euo pipefail
cat > /root/sayscene-cni-alias-guard.new <<'GUARD'
#!/bin/bash
set -euo pipefail
nic="${1:?NIC 이름이 필요합니다}"
[[ "$nic" =~ ^ens[0-9]+$ ]] || exit 0
mask="/etc/systemd/network/70-${nic}.network.d/ec2net_alias.conf"
if [[ -L "$mask" && "$(readlink -- "$mask")" == /dev/null ]]; then
  exit 0
fi
if [[ -e "$mask" || -L "$mask" ]]; then
  echo "다른 사용자 설정을 덮어쓰지 않습니다: $mask" >&2
  exit 1
fi
mkdir -p -- "${mask%/*}"
if ! ln -s -- /dev/null "$mask"; then
  [[ -L "$mask" && "$(readlink -- "$mask")" == /dev/null ]] || exit 1
fi
GUARD
bash -n /root/sayscene-cni-alias-guard.new
WRITE_CANDIDATE
```

**정상:** 오류 출력 없이 종료한다. **실패:** 디스크·권한 또는 `bash -n`이 지적한 구문을 고치고 다시 작성한다. 후보 검사가 통과하기 전에는 실행 파일로 설치하지 않는다.

#### 3-2. 기존 유무에 따라 설치·유지 선택

```bash
sudo ls -l /usr/local/sbin/sayscene-cni-alias-guard
# 기존 파일이 있을 때만 비교한다. 차이가 있으면 diff의 종료 코드 1은 정상이다.
sudo diff -u /usr/local/sbin/sayscene-cni-alias-guard \
  /root/sayscene-cni-alias-guard.new
```

- **기존 파일이 없음:** 다음 설치 명령을 실행한다.
- **차이가 없음:** 내용은 유지한다. 실행 권한만 없으면 `sudo chmod 0755 /usr/local/sbin/sayscene-cni-alias-guard`를 실행한다.
- **차이가 있음:** 이번에 복구한 가드가 같은 기능을 수행한다면 유지하고 3-3을 검증한다. 다른 기능을 유지할 필요가 없고 이 후보로 통일하려면 **기존 노드는 1번 백업이 완료됐는지 확인한 뒤** 다음 설치 명령으로 교체한다. 기존 가드에 필요한 추가 기능이 있으면 먼저 후보에 반영·검증하며, 무조건 덮어쓰지 않는다.

```bash
# 새 설치 또는 위 비교 후 교체하기로 판단한 경우에만 실행한다.
sudo install -D -m 0755 /root/sayscene-cni-alias-guard.new \
  /usr/local/sbin/sayscene-cni-alias-guard
```

#### 3-3. 제작 시 이미 존재하는 NIC만 1회 보호·검증

**이 반복문은 제작용 EC2의 현재 NIC를 위한 1회 초기화다. 새 노드마다 실행하라는 뜻이 아니다.** 아직 없는 ens6·ens7 등의 마스크는 4번 서비스 템플릿이 자동 생성한다. 미래 NIC 보호는 4번까지 설치해야 완성된다.

```bash
sudo bash <<'MASK_CURRENT'
set -euo pipefail
for path in /sys/class/net/ens*; do
  [[ -e "$path" ]] || continue
  nic="${path##*/}"
  mask="/etc/systemd/network/70-${nic}.network.d/ec2net_alias.conf"
  /usr/local/sbin/sayscene-cni-alias-guard "$nic"
  test "$(readlink -- "$mask")" = /dev/null
  before="$(stat -c '%y' "$mask")"
  /usr/local/sbin/sayscene-cni-alias-guard "$nic"
  test "$(stat -c '%y' "$mask")" = "$before"
  printf 'PASS %s: /dev/null 마스크, 반복 실행 시 변경 시각 유지\n' "$nic"
done
MASK_CURRENT
```

**정상:** 현재 존재하는 ens5·ens6·ens7 등에 각각 `PASS`가 나온다. 기존 ens5 마스크는 교체되지 않는다. **실패:**

| 오류 | 조치 |
| --- | --- |
| `Permission denied` | `sudo ls -l /usr/local/sbin/sayscene-cni-alias-guard`로 실행 권한을 확인해 `0755`로 맞춘다. 파일 내용·인터프리터도 확인한다. |
| `다른 사용자 설정을 덮어쓰지 않습니다` | 오류에 나온 **정확한 파일**을 `sudo ls -l 파일경로`, `sudo cat 파일경로`로 확인한다. 필요한 사용자 설정이면 이 방식으로 덮어쓰지 말고 별도 이관한다. 예전 alias 전용 파일로 확인되면 아래처럼 백업 이동 후 3-3을 재실행한다. |
| 두 번째 실행에서 변경 시각이 달라짐 | 기존 가드가 매번 링크를 재생성하는지 확인한다. 후보와 비교해 정상 마스크면 즉시 종료하도록 고친 뒤 재검증한다. |
| `PASS`가 하나도 없음 | `ls /sys/class/net`으로 실제 NIC 이름을 확인한다. 다른 이름이면 2번 적용 대상 판정으로 돌아간다. |

```bash
# 오류에 나온 파일로 변경한다. ens7은 예시다.
# /dev/null 마스크가 이미 정상인 경우에는 실행하지 않는다.
sudo bash <<'RETIRE_CONFLICT'
set -euo pipefail
file=/etc/systemd/network/70-ens7.network.d/ec2net_alias.conf
backup="$(mktemp -d /root/sayscene-alias-conflict.XXXXXX)"
mv -- "$file" "$backup/"
printf '충돌 파일 보관: %s\n' "$backup"
RETIRE_CONFLICT
```

이 시점에는 **마스크만 준비한 것**이다. 기존에 호스트에 붙은 `/32`가 남아 있어도 6번 전에는 주소를 일괄 삭제하지 않는다.

### 4. 공통: 새 ENI 초기 설정·주기적 갱신에 가드 연결

**이유:** 기존 NIC에 마스크를 만들어도 미래 NIC에는 파일이 없다. **AMI 제작 시 반드시** 두 서비스의 템플릿에 연결해야, 이 AMI에서 시작한 모든 노드가 NIC별 수동 실행 없이 보호된다. 아래 경로는 `@ens5.service.d`가 아니라 **`@.service.d`**다.

```bash
sudo bash <<'INSTALL_HOOKS'
set -euo pipefail
test -x /usr/local/sbin/sayscene-cni-alias-guard
tmp="$(mktemp)"
trap 'rm -f "$tmp"' EXIT
cat > "$tmp" <<'DROPIN'
[Service]
ExecStartPre=/usr/local/sbin/sayscene-cni-alias-guard %I
DROPIN
for unit in policy-routes refresh-policy-routes; do
  file="/etc/systemd/system/${unit}@.service.d/10-sayscene-cni-alias-guard.conf"
  if [[ -e "$file" || -L "$file" ]]; then
    cmp -s "$tmp" "$file" || {
      echo "내용이 달라 변경하지 않습니다: $file" >&2
      exit 1
    }
  else
    install -D -m 0644 "$tmp" "$file"
  fi
done
systemctl daemon-reload
INSTALL_HOOKS

systemctl cat policy-routes@.service refresh-policy-routes@.service
systemctl show policy-routes@ens5.service refresh-policy-routes@ens5.service \
  -p ExecStartPre -p ExecStart
```

**정상:** 두 서비스 모두 실제 `ExecStartPre`에 가드와 `ens5` 인자가 보이고, vendor `ExecStart`도 남아 있다. `%I`는 `ens5` 같은 NIC 이름으로 치환된다. `daemon-reload`는 서비스 정의를 읽는 작업이며 networkd restart가 아니다.

아직 없는 NIC에도 템플릿이 적용되는지 **설정 조회만으로** 확인한다. 다음 명령은 ens99를 만들거나 서비스를 실행하지 않는다.

```bash
systemctl show policy-routes@ens99.service refresh-policy-routes@ens99.service \
  -p Id -p LoadState -p ExecStartPre -p ExecStart
```

**정상:** 두 unit 모두 `LoadState=loaded`이고 가드 인자가 `ens99`로 표시된다. ens5에서만 가드가 보이고 ens99에서는 안 보이면 **NIC 전용 설정만 한 것이므로 AMI 생성 금지**다. 위 템플릿 경로로 설치하고 `daemon-reload`한 뒤 다시 확인한다. 이 조회는 템플릿 구성 검사이며 실제 ENI add·부팅 검증은 8번에서 별도로 한다.

**예상과 다를 때:**

- 내용 불일치로 중단하면 오류에 나온 파일을 `sudo cat 파일경로`로 확인한다. 이미 같은 가드만 호출하는 동등한 훅이면 유지하고 `sudo systemctl daemon-reload` 후 실제 `ExecStartPre`를 확인한다. 잘못된 훅이면 1번 백업 후 `sudoedit 파일경로`로 위 `[Service]` 두 줄에 맞추고 블록을 다시 실행한다. 다른 필요한 사전 작업은 삭제하지 않는다.
- 파일에는 가드가 있는데 실제 `ExecStartPre`에 없으면 `systemctl cat policy-routes@ens5.service refresh-policy-routes@ens5.service`로 **인스턴스 전용 drop-in**까지 확인한다. 뒤에서 `ExecStartPre=`로 목록을 초기화하는 설정이 있다면 그 설정을 백업·수정하여 가드가 실제 목록에 남게 한 뒤 `daemon-reload`한다.
- `ExecStartPre=-/usr/local/...`처럼 앞에 `-`가 있으면 실패 무시이므로 제거한다. 정상 가드 실패 시 vendor 작업도 중단되는 것이 의도다. 오류 원인을 고친 뒤 **시험 워커에서** 해당 `refresh-policy-routes@NIC.service`를 다시 시작해 확인한다.

마스크는 `/run/.../ec2net_alias.conf` **생성 자체를 막지 않는다.** `/etc`의 동명 `/dev/null` 마스크가 적용을 막는다. 따라서 `/run`에 `Address=`가 남아 있다는 이유로 vendor 파일을 지우지 않는다. [ExecStartPre 동작](https://www.freedesktop.org/software/systemd/man/252/systemd.service.html), [템플릿 drop-in](https://www.freedesktop.org/software/systemd/man/252/systemd.unit.html)

### 5. 공통: CNI 경로·정책 라우팅 규칙 보존

**이유:** alias 마스크는 주소 중복만 막는다. networkd가 CNI의 경로·규칙을 정리하지 않도록 별도 옵션도 필요하다.

```bash
systemd-analyze cat-config systemd/networkd.conf
sudo grep -R -n -E '^[[:space:]]*ManageForeign(Routes|RoutingPolicyRules)[[:space:]]*=' \
  /etc/systemd/networkd.conf /etc/systemd/networkd.conf.d \
  /run/systemd/networkd.conf.d /usr/lib/systemd/networkd.conf.d
```

**정상:** 적용 대상 설정에서 두 옵션의 **최종 값**이 모두 `no`다. `cat-config`의 파일 구분과 순서를 같이 읽는다. `grep`에서 없는 디렉터리 오류가 나오는 것과 옵션 미설치는 구분한다.

| 결과 | 조치 |
| --- | --- |
| 두 값이 이미 `no`, 충돌 없음 | `80-vpc-cni.conf` 또는 기존 `90-kubernetes-cni.conf`를 **그대로 유지**한다. 같은 설정을 하나 더 만들 필요 없다. |
| 옵션 없음 | 아래 파일 생성 명령을 실행한다. |
| 어떤 파일에서 `yes`로 설정함 | `cat-config`로 실제 선택된 파일인지 확인한다. `/etc`의 사용자 파일이면 1번 백업 후 `sudoedit 해당파일`로 두 값을 `no`로 통일한다. `/usr/lib` vendor 파일은 수정하지 않고 아래 로컬 파일로 재정의한다. 더 뒤의 drop-in이 다시 `yes`로 덮으면 그 로컬 설정도 수정한다. |
| 새 파일이 이미 있어 생성 명령 중단 | `sudo cat /etc/systemd/networkd.conf.d/80-vpc-cni.conf`로 확인한다. 두 값이 `no`면 유지하고, 다르면 백업 후 `sudoedit`로 수정한다. |

```bash
# 옵션이 없거나 로컬 재정의가 필요한 경우에만 실행한다.
sudo bash <<'INSTALL_NETWORKD'
set -euo pipefail
file=/etc/systemd/networkd.conf.d/80-vpc-cni.conf
if [[ -e "$file" || -L "$file" ]]; then
  echo "기존 파일을 확인·수정하세요: $file" >&2
  exit 1
fi
install -d -m 0755 /etc/systemd/networkd.conf.d
cat > "$file" <<'NETWORKD'
[Network]
ManageForeignRoutes=no
ManageForeignRoutingPolicyRules=no
NETWORKD
chmod 0644 "$file"
INSTALL_NETWORKD

systemd-analyze cat-config systemd/networkd.conf
```

**종료 조건:** 최종 값이 둘 다 `no`여야 한다. `cat-config`는 디스크 설정 확인이지 실행 중 반영의 증거는 아니다. 기존부터 적용 중이면 유지하고, 이번에 값을 바꿨으면 **6-B의 계획된 재부팅**으로 적용한다. `networkctl reload`만으로 이 전역 옵션까지 적용됐다고 간주하지 않는다. [옵션 설명](https://www.freedesktop.org/software/systemd/man/252/networkd.conf.html)

### 6. 공통: 설정 적용·현재 주소 정리·접속 검증

**이유:** 파일 생성만으로 이미 등록된 주소가 사라지지는 않는다. 기존 설정 종류에 맞는 적용 방법을 선택해야 DHCP와 접속을 잃지 않는다.

#### 6-A. 기존 운영 노드: 옛 override 없음 + 보존 옵션은 이미 적용 중

이번에 마스크·훅만 추가했다면 다음처럼 적용한다. **이미 이번 장애 복구까지 완료했다면 reload/reconfigure/주소 삭제를 반복하지 말고 아래 공통 검증만 한다.**

```bash
systemctl show systemd-networkd -p MainPID
sudo networkctl reload
# 문제가 있었던 실제 NIC로 바꾼다. 다른 NIC까지 일괄 재적용하지 않는다.
sudo networkctl reconfigure REPLACE_NIC
systemctl show systemd-networkd -p MainPID
```

**정상:** 명령 오류가 없고 networkd PID가 전후 동일하며 기존 접속이 유지된다. reload/reconfigure도 네트워크 상태를 바꿀 수 있어 무중단을 보장하지 않는다. 오류·접속 이상이면 주소 삭제를 계속하지 말고 아래 복구 절차로 간다.

호스트에 남은 중복 주소는 먼저 대조한다.

```bash
kubectl get pods -A -o wide
ip -o -4 addr show
```

현재 **일반 Pod의 IP가 ens NIC에도 `/32`로 붙어 있는 경우만**, 실제 값으로 다음을 실행한다. API server·aws-node 같은 `hostNetwork` Pod의 노드 IP는 이 삭제 대상이 아니다.

```bash
sudo ip address del REPLACE_POD_IP/32 dev REPLACE_NIC
ip -4 route get REPLACE_POD_IP
```

**정상:** 같은 노드의 현재 일반 Pod IP 경로가 `local … dev lo`에서 `dev eni...`로 바뀐다. **주소가 이미 없으면** 삭제를 생략한다. **여전히 local이면** `ip -o -4 addr show`에서 다른 ens에도 중복 주소가 있는지 찾아 해당 중복만 처리한다. **주소 중복은 없는데 경로가 물리 ens로 향하면** CNI 규칙·Pod veth 경로 문제이므로 임의 경로를 추가하지 말고 8번의 통신 실패 분기로 간다.

#### 6-B. 새 제작용 EC2 / 옛 override 전환 / 보존 옵션 신규 적용

- **새 제작용 EC2:** 3~5번 완료 후 아래 `sudo reboot`만 실행한다.
- **기존 노드에서 보존 옵션만 변경:** 1번 복구 수단·유지보수 시간 확인 후 재부팅한다.
- **기존 노드에 이전 문서의 10/20 override가 있음:** 3번 마스크, 4번 실제 서비스 훅, 5번 보존 옵션이 모두 정상인 상태에서만 다음으로 옛 파일을 퇴역시킨다. 해당 두 파일이 **이전 가이드의 설정만 담고 있는지** 1번에서 확인했어야 한다. 다른 용도의 설정이 섞였으면 이 명령을 실행하지 말고 먼저 이관한다.

```bash
# 이전 가이드의 두 파일을 전환하는 기존 노드에서만 실행한다.
sudo bash <<'RETIRE_LEGACY'
set -euo pipefail
backup="$(mktemp -d /root/sayscene-retired-network.XXXXXX)"
for name in 10-kubernetes-primary.network 20-kubernetes-cni-eni.network; do
  file="/etc/systemd/network/$name"
  if [[ -e "$file" || -L "$file" ]]; then
    mv -- "$file" "$backup/"
  fi
done
printf '옛 override 보관: %s\n' "$backup"
RETIRE_LEGACY
```

**정상:** 해당 두 파일만 `/etc/systemd/network`에서 빠지고 백업 경로에 보관된다. vendor DHCP 설정과 alias 마스크는 남아 있어야 한다. 다른 우선 매칭 `.network`가 있으면 아직 전환 준비가 끝난 것이 아니다. 모든 설정이 준비된 뒤 실행한다.

```bash
sudo reboot
```

#### 6-C. 재접속 후 공통 검증과 실패 시 복구

```bash
networkctl status ens5 --no-pager
ip -br -4 addr
ip -4 route show default
resolvectl status ens5
getent ahostsv4 github.com
```

| 정상 기준 | 다르면 할 조치 |
| --- | --- |
| ens5는 vendor `70-ens5.network` 사용, 원래 기본 사설 IP 유지 | 아직 `10-*` 등이 선택되면 남은 우선 매칭 파일을 찾아 1번·6-B 전환을 마친다. DHCP 주소가 없으면 아래 백업 복구를 먼저 한다. |
| 각 ens에는 원래 DHCP 기본 주소가 있고 Pod `/32`는 없음 | 중복 `/32`만 남으면 6-A 대조·삭제를 수행한다. 재생성되면 해당 NIC의 마스크와 두 서비스 훅을 3~4번에서 재확인한다. |
| 기본 경로는 기존 primary ens5를 우선 사용, DNS 서버·이름 조회 정상 | 이전 백업의 주소·경로와 비교한다. 이번 수정 때문에 달라졌으면 백업 복구를 우선한다. **원인 확인 없이 SG·VPC route table·SNAT까지 바꾸지 않는다.** |
| SSH/SSM 접속 정상 | 복구 접속으로 로그인해 아래 절차를 수행한다. 접속 수단이 모두 실패하면 EC2 루트 EBS를 복구용 EC2에 연결해 `/etc` 파일을 복원하는 별도 장애 복구가 필요하다. 운영 EBS를 콘솔에서 무작정 분리하지 않는다. |

**옛 10/20 override를 옮긴 직후 문제가 생겼다면**, 복구 접속에서 6-B에 출력된 실제 백업 경로로 다음을 실행한다. 이후 유지보수 중 다시 전환 원인을 확인한다.

```bash
sudo bash <<'RESTORE_LEGACY'
set -euo pipefail
backup=/root/REPLACE_WITH_ACTUAL_sayscene-retired-network_DIRECTORY
test -d "$backup"
if [[ ! -e "$backup/10-kubernetes-primary.network" && ! -e "$backup/20-kubernetes-cni-eni.network" ]]; then
  echo "백업 파일이 없습니다. 경로를 확인하세요." >&2
  exit 1
fi
for name in 10-kubernetes-primary.network 20-kubernetes-cni-eni.network; do
  if [[ -e "$backup/$name" ]]; then
    cp -a -- "$backup/$name" /etc/systemd/network/
  fi
done
# 같은 set -e 블록 안에서 복원 성공 시에만 재부팅한다.
reboot
RESTORE_LEGACY
```

그 외 이번에 수정한 가드·서비스 훅·전역 옵션이 원인이면 1번 백업의 **해당 파일만** 복원한다. 예: `sudo cp -a /root/실제백업/etc/systemd/networkd.conf.d/실제파일 /etc/systemd/networkd.conf.d/`. 이번에 새로 만든 파일은 백업에 없으므로 해당 파일만 `/root`로 이동한다. 서비스 훅 복원 후 `sudo systemctl daemon-reload`, 네트워크 설정 전환은 계획된 재부팅 후 확인한다. 정상 alias 마스크까지 일괄 삭제하지 않는다.

### 7. 새 제작용 EC2 전용: 깨끗한 CPU·GPU AMI 생성

**이유:** 현재 노드의 `/etc`·`/usr/local/sbin` 변경은 다른 EC2에 자동 복제되지 않는다. Karpenter 워커도 보호 설정을 가진 이미지로 시작해야 한다. **기존 운영 노드는 이 단계를 실행하지 않는다.**

```bash
sudo bash <<'CHECK_CLEAN'
set -euo pipefail
for file in /etc/kubernetes/admin.conf /etc/kubernetes/kubelet.conf \
  /etc/kubernetes/bootstrap-kubelet.conf /etc/kubernetes/pki/ca.key; do
  if [[ -e "$file" ]]; then
    echo "AMI 생성 중단: 클러스터 초기화/가입 흔적 $file" >&2
    exit 1
  fi
done
test ! -d /var/lib/etcd/member
printf '기본 init/join 흔적 검사 통과. 수동 보안 검사도 수행하세요.\n'
CHECK_CLEAN
```

**정상:** 검사 통과 + init/join을 수행하지 않은 제작 이력이 확인된다. 이 검사는 모든 인증 정보 부재를 보증하지 않으므로 사용자 데이터의 비밀 값·SSM Agent의 기존 인스턴스 등록 정보·개인 자격 증명·토큰 파일이 AMI에 남지 않도록 별도로 확인한다. SSM Agent 프로그램 자체를 제거하라는 뜻은 아니다. **실패:** 해당 파일만 지워 이미지를 만들지 않는다. init/join 전 원본으로 새 제작용 EC2를 만들고 2~6번을 적용한다.

이미지에 포함되어야 할 항목:

| 항목 | 정상 상태 / 누락 시 조치 |
| --- | --- |
| `/usr/local/sbin/sayscene-cni-alias-guard` | 실행 가능. 없으면 3번 적용 |
| 초기·갱신 **템플릿** 두 개의 `10-sayscene-cni-alias-guard.conf` | 4번 ens5·ens99 조회 모두 실제 `ExecStartPre`에 각각의 NIC 인자 포함. 없으면 4번 적용 |
| 기존 이름의 networkd 보존 설정 파일 | 두 옵션 최종 값 `no`. 아니면 5번 적용·재부팅 |
| 현재 NIC의 `/etc/.../ec2net_alias.conf → /dev/null` | 3-3의 모든 현재 ens `PASS`. 미래 ens는 서비스 시작 시 생성됨 |

CPU/GPU 패키지 준비까지 끝나고 6번 재부팅 검증을 통과했으면 **AWS EC2 콘솔 → 제작용 인스턴스 선택 → 작업 → 이미지 및 템플릿 → 이미지 생성**을 실행한다. 이미지 생성의 재부팅은 기본값을 유지하고, AMI 상태가 **Available**이 될 때까지 기다린다. `Pending`이면 기다리고, `Failed`이면 콘솔 상태 메시지·관련 스냅샷 상태를 확인해 실패 원인을 해결한다. **init/join한 시험 워커를 다시 AMI로 캡처하지 않는다.**

### 8. 시험 워커: 실제 새 ENI·refresh·Pod 통신 검증

**이유:** 재부팅 후 ens5만 정상인 것으로는 이번 재발 원인을 검증하지 못한다. **새 AMI에서 시작한 노드가 사람의 보완 작업 없이** 부팅 시 여러 ENI·이후 추가 ENI·주기적 갱신 모두에서 보호되는지 확인해야 한다.

**합격 조건:** 시험 노드에는 3~5번 설치 명령·가드 직접 실행·수동 마스크 생성·가드용 userData를 추가하지 않는다. 부팅·ENI 추가 후에는 조회만 한다. 수동 실행으로 복구했다면 현장 복구일 뿐 **AMI 자동 보호 시험은 불합격**이며, 제작용 EC2를 수정해 새 AMI로 다시 시험한다. kubeadm 가입 userData는 별개이며 그대로 사용할 수 있다.

#### 8-1. 새 AMI로 시험 워커를 만들고 기준 상태 기록

새 AMI로 별도 시험 EC2를 시작하고 기존 가입 절차로 클러스터에 연결한다. Karpenter 설정을 운영에 반영하기 전 수동 시험 워커를 써도 된다. **운영 컨트롤 플레인에 시험용 Pod를 대량 배치하지 않는다.**

```bash
# kubectl 사용이 가능한 컨트롤 플레인에서 실행한다.
kubectl get nodes -o wide --show-labels
kubectl -n kube-system get daemonset aws-node
kubectl -n kube-system get pods -l k8s-app=aws-node -o wide
kubectl -n kube-system get service kube-dns
```

**정상:** 시험 노드가 `Ready`, 해당 노드의 aws-node가 `Running/Ready`다. **다르면:** Node `NotReady`는 `kubectl describe node 실제노드이름`, aws-node 오류는 해당 Pod의 `kubectl -n kube-system logs 실제Pod이름 -c aws-node --tail=100`으로 확인한다. 가입·IAM·CNI 오류부터 해결하며 이 상태에서 부하 시험을 시작하지 않는다. `kube-dns`의 ClusterIP도 기록한다.

시험 워커 EC2에서 `ip -br -4 addr`와 `ls /sys/class/net`을 기록한다. **현재 ens 개수**와 각 기본 주소가 기준이다.

**부팅 때부터 ENI가 여러 개인 시험 구성도 확인한다.** 같은 AMI에서 시작한 해당 시험 노드에 마스크를 수동 생성하지 않고, 다음 조회로 각 ens의 자동 마스크를 확인한다. 처음에는 ens5 하나만 있는 경우 이 검사는 ens5만 통과한 것이며, 다중 ENI 부팅 검증을 했다고 기록하지 않는다.

```bash
for path in /sys/class/net/ens*; do
  [ -e "$path" ] || continue
  nic="${path##*/}"
  sudo ls -l "/etc/systemd/network/70-${nic}.network.d/ec2net_alias.conf"
  systemctl show "policy-routes@${nic}.service" \
    -p Id -p ExecMainStartTimestamp -p Result -p ExecStartPre
done
```

**정상:** 현재 모든 ens에 `/dev/null` 마스크와 초기 서비스의 실제 실행 시각·성공 결과가 있다. 누락 시 가드를 직접 실행하지 말고 8-3의 실패 분기로 간다. 부팅 시 ENI 수가 달라도 AMI 파일을 수정할 필요는 없다.

#### 8-2. 시험 Pod 수를 조금씩 늘려 새 CNI ENI 확인

아래 namespace는 **이번 시험에서 새로 만든 것만 사용**한다. 이미 있으면 기존 리소스를 지우지 말고 다른 고유 namespace 이름으로 모든 명령·YAML을 함께 바꾼다. YAML의 `REPLACE_TEST_NODE_HOSTNAME`은 8-1의 시험 노드 `kubernetes.io/hostname` 라벨 값으로 바꾼다.

```bash
# namespace 생성이 성공했을 때만 적용한다. 기존 namespace는 수정하지 않는다.
kubectl create namespace cni-eni-check &&
kubectl apply -f - <<'PODS'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cni-eni-check
  namespace: cni-eni-check
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cni-eni-check
  template:
    metadata:
      labels:
        app: cni-eni-check
    spec:
      nodeSelector:
        kubernetes.io/hostname: REPLACE_TEST_NODE_HOSTNAME
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      automountServiceAccountToken: false
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        seccompProfile:
          type: RuntimeDefault
      containers:
        - name: check
          image: python:3.12-slim
          command: [python3, -c, "import time; time.sleep(86400)"]
          resources:
            requests:
              cpu: 1m
              memory: 16Mi
            limits:
              memory: 64Mi
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: [ALL]
PODS

kubectl -n cni-eni-check get pods -o wide
```

**정상:** 첫 1개 Pod가 시험 워커에서 Running이다. `Pending`이면 `kubectl -n cni-eni-check describe pod 실제Pod이름`의 Events에서 라벨·taint·리소스 부족을 해결한다. `ImagePullBackOff`는 노드의 이미지 registry 연결을 해결한다. 운영 노드에 배치되면 YAML의 라벨을 고쳐 시험 워커로 이동한 뒤 진행한다.

한 번에 5개 정도씩 늘린다. 매번 기존 Pod 포함 시험 노드 수가 `kubectl describe node 실제노드이름`의 `Allocatable: pods` 및 CPU·메모리를 넘지 않도록 한다.

```bash
kubectl -n cni-eni-check scale deployment cni-eni-check --replicas=5
kubectl -n cni-eni-check get pods -o wide
# 5개가 정상이고 추가 여유가 있을 때만 --replicas=10, 15 ... 로 증가한다.
```

동시에 **시험 워커 EC2에서** `ip -br -4 addr`로 새 ens가 생기는지 확인한다. CNI가 warm IP를 이미 많이 보유하면 바로 새 ENI가 생기지 않을 수 있다. **새 ens를 하나 확인하면 증설을 멈춘다.** ENI 한도·Pod 수 한도까지 도달해도 새 ENI가 생기지 않으면 이 시험은 **신규 ENI 검증 미완료**다. 운영 aws-node의 warm target을 임의로 바꾸지 말고, 여유가 있는 별도 시험 인스턴스·CNI 시험 구성을 준비해 재검증한다. 콘솔 수동 부착 ENI를 CNI가 관리한다고 가정하지 않는다.

#### 8-3. 새 ENI의 마스크·초기 서비스·갱신 서비스 판정

시험 워커에서 새 NIC 이름으로 실행한다. `ens8`은 예시이며 실제 이름으로 바꾼다.

```bash
sudo ls -l /etc/systemd/network/70-ens8.network.d/ec2net_alias.conf
systemctl show policy-routes@ens8.service refresh-policy-routes@ens8.service \
  -p Id -p ExecMainStartTimestamp -p ExecMainStatus -p Result -p ExecStartPre
systemctl status refresh-policy-routes@ens8.timer --no-pager
sudo journalctl -u policy-routes@ens8.service \
  -u refresh-policy-routes@ens8.service --since '10 minutes ago' --no-pager
ip -o -4 addr show dev ens8
```

**여기서는 서비스의 수동 start나 가드 직접 실행을 하지 않는다.** v2.7.7 timer는 대략 1분 주기이므로 2분 이상 기다린 뒤 같은 조회를 다시 실행한다. 새 ENI의 자동 초기 설정과 **timer에 의한 실제 갱신 실행 시각·로그**가 모두 있어야 한다. `Result=success`만 있고 실행 시각이 비어 있으면 미실행일 수 있으므로 합격이 아니다.

| 정상 기준 | 다르면 할 조치 |
| --- | --- |
| 새 NIC에도 `/dev/null` 마스크, 두 서비스 실제 가드 인자도 해당 NIC | 자동 보호 실패다. 4번 템플릿·인스턴스 drop-in과 2번 udev 경로를 확인해 **제작용 EC2를 수정하고 AMI를 다시 만든다.** 시험 노드의 수동 마스킹으로 합격 처리하지 않는다. |
| 초기·자동 갱신 모두 실행 시각 있음, `Result=success`, `ExecMainStatus=0` | journal의 가드 충돌 또는 vendor 오류를 확인한다. 가드 충돌 파일이 AMI에서 왔다면 제작용 EC2의 3번 충돌 처리를 수정한다. 해당 이미지로 운영 배포하지 않고 새 이미지로 재시험한다. |
| refresh timer 활성, 다음 자동 실행 후 시각이 갱신됨 | timer 비활성·미실행은 초기 서비스와 연결된 timer 정의·journal을 확인한다. vendor 구성 복구 후 새 AMI의 재부팅/ENI 추가로 자동 시작을 재검증한다. 수동 `start`로만 되는 상태는 불합격이다. |
| 갱신 후에도 ens에는 원래 기본 DHCP 주소만 있고 Pod `/32` 없음 | 실제 선택 `.network`와 alias 마스크 우선순위를 확인한다. 운영 중인 노드의 긴급 복구는 6-A를 쓸 수 있지만, AMI는 수정 후 별도 새 노드로 재시험한다. |

`inactive (dead)`인 정상 종료 oneshot 자체는 실패가 아니다. 실제 실행 시각·종료 결과와 timer의 다음 자동 실행으로 판단한다.

#### 8-4. 같은 노드 Pod 경로·DNS·Service·외부 HTTPS 검증

시험 Pod 하나의 IP를 `kubectl -n cni-eni-check get pods -o wide`에서 확인한다. **새 ENI가 가진 secondary IP 중 하나를 받은 Pod**도 시험해야 한다. EC2 콘솔 → 네트워크 인터페이스 → 새 ENI → IPv4 주소 목록과 Pod IP를 대조한다.

시험 워커 EC2에서:

```bash
ip -4 route get REPLACE_POD_IP
ip -o -4 addr show
ip -4 rule show
```

**정상:** 같은 노드 일반 Pod의 목적지 경로가 `dev eni...`이고 호스트 ens에 같은 IP가 없다. 현재 CNI 모드에 필요한 목적지 `512`·보조 ENI Pod 출발지 `1536` 규칙도 확인한다. 모든 노드의 규칙 개수가 같아야 하는 것은 아니다.

kubectl 터미널에서 실제 Pod 이름으로:

```bash
kubectl -n cni-eni-check exec -i REPLACE_POD_NAME -- python3 - <<'CHECK_NETWORK'
import socket
import urllib.request
for name in ('kubernetes.default.svc.cluster.local', 'github.com'):
    print(name, socket.gethostbyname(name))
with socket.create_connection(('kubernetes.default.svc.cluster.local', 443), timeout=5):
    print('API_SERVICE_TCP_OK')
with urllib.request.urlopen('https://github.com', timeout=10) as response:
    print('EXTERNAL_HTTPS_OK', response.status)
CHECK_NETWORK
```

**정상:** 두 이름의 IP, `API_SERVICE_TCP_OK`, `EXTERNAL_HTTPS_OK`가 출력된다. 기존 운영 노드의 현재 Pod 통신만 확인할 때도 같은 원리로 검사하되 Python 없는 이미지에 위 명령을 그대로 실행하지 않는다.

| 실패 | 조치 |
| --- | --- |
| `local … dev lo` | 새 AMI 시험은 불합격이다. 중복 주소·마스크 누락 증거를 기록하고 제작용 EC2의 3~4번을 수정한 뒤 새 AMI·새 시험 노드로 다시 검증한다. 6-A의 즉시 주소 제거는 기존 운영 노드의 긴급 복구에만 사용한다. |
| 호스트 중복은 없는데 같은 노드 Pod 경로가 ens로 향함 / CNI 규칙 누락 | 5번 최종 설정과 적용 여부를 확인한다. AMI의 보존 설정 결함이면 제작용 EC2에서 수정·재부팅 후 새 AMI·새 시험 노드로 재검증한다. 이미 사라진 규칙은 설정만으로 복원되지 않는다. 시험 Pod 재생성은 진단에 쓸 수 있지만 AMI 결함을 고친 것으로 간주하지 않는다. 기존 운영 노드 복구는 영향 workload·중단 시간을 확인해 수행하며 단일 컨트롤 플레인에 무작정 rollout/reboot하지 않는다. |
| DNS 실패 | `kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide`로 CoreDNS Ready·Pod IP를 확인한다. Pod의 `/etc/resolv.conf`를 `kubectl -n cni-eni-check exec 실제Pod -- cat /etc/resolv.conf`로 보고 kube-dns ClusterIP와 대조한다. CoreDNS 주소 경로·Service/CNI 규칙부터 확인하며 repo 인증 문제로 단정하지 않는다. |
| DNS는 되는데 API TCP 실패 | kube-proxy/Service 경로·정책을 확인한다. `kubectl -n kube-system get pods -l k8s-app=kube-proxy -o wide`와 해당 Pod 로그에서 오류를 확인한다. alias 마스크가 API 포트를 여는 설정은 아니다. |
| DNS·API는 되는데 외부 HTTPS 실패 | HTTP 응답 코드가 있으면 DNS/TCP/TLS 연결 이후 문제일 수 있다. timeout이면 시험 노드의 기존 인터넷 경로·Pod SNAT·정책을 별도로 확인한다. `CERTIFICATE_VERIFY_FAILED`는 인증서·CA·시간을 확인한다. **이 문서의 검증 실패를 이유로 SNAT·SG를 무조건 변경하지 않는다.** |

시험 Pod 재생성은 `kubectl -n cni-eni-check delete pod 실제시험Pod이름`으로 한다. Deployment가 다시 만든 Pod의 **새 IP**로 확인한다. 통과 후 이번에 만든 namespace만 `kubectl delete namespace cni-eni-check`로 정리한다. 생성 전부터 있던 namespace는 삭제하지 않는다.

### 9. 검증 완료 후 새 AMI를 운영 설정에 반영

**이유:** AMI를 만들기만 하면 Karpenter가 자동으로 새 이미지를 선택하는 것은 아니다. 반대로 이미지 참조를 바꾸면 기존 워커 교체가 시작될 수 있으므로 검증 후 반영해야 한다.

1. **Karpenter가 이미 있으면:** Git의 해당 CPU/GPU `EC2NodeClass.amiSelectorTerms`에서 사용할 AMI ID를 새 ID로 변경한다. 예시는 `amiSelectorTerms: [{id: ami-실제새ID}]`이며 기존 다른 NodeClass 설정은 유지한다. Argo CD의 해당 Application에서 Refresh → Diff로 변경 대상을 확인한 뒤 Sync한다. 다른 NodeClass까지 바뀌면 Sync하지 말고 Git 수정을 바로잡는다.
2. **아직 Karpenter 설치 전이면:** 이후 설치에 사용할 CPU/GPU AMI ID 기록만 새 값으로 갱신한다. `settings.env` 등에서도 관리한다면 함께 맞춘다. 이 문서를 위해 Karpenter를 먼저 설치할 필요는 없다.
3. `kubectl get ec2nodeclass`, `kubectl describe ec2nodeclass 실제이름`으로 준비 상태·선택된 AMI를 확인한다. Ready가 아니면 Conditions/Events에 나온 아키텍처·AMI 접근·보안 그룹·서브넷 등의 원인을 해결하고 새 노드 생성을 진행하지 않는다.
4. 새 워커 EC2의 콘솔 상세에서 실제 AMI ID가 맞는지 확인하고 8번 결과와 대조한다. 이전 AMI면 다른 selector·NodeClass를 참조하는 NodePool·실제 Git Sync 상태를 확인한다.
5. 기존 워커의 drift·교체는 NodePool 중단 예산·PDB·workload 상태를 확인해 계획한다. **기존 컨트롤 플레인은 새 AMI로 자동 수정되지 않는다.**

**완료 기준:** 기존 노드는 실제 추가 ENI/갱신에서도 중복이 재발하지 않고 Pod 통신이 정상이며, 새 AMI는 별도 시험 워커에서 같은 검증을 통과해야 한다. 패키지 업그레이드로 서비스·udev·alias 생성 경로가 바뀌면 2번부터 재검증한다. 이 문서의 현장 복구 결과는 사용자 제공 기록이며, 여기서 실제 EC2 변경·AMI 생성·ENI 시험을 대신 수행한 것은 아니다.
