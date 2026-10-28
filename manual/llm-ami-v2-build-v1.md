# LLM 워커 AMI(`sayscene-worker-llm-v2`) 빌드 매뉴얼

> 상태: **운영 중** · 작성일: 2026-10-28 · 실측일: 2026-10-27
> 관련: [이슈 01 — 주 NIC `ens5` 고정](../issue/20261027/01-llm-ami-ens5-primary-nic.md) · [이슈 02 — EBS lazy loading](../issue/20261027/02-ebs-lazy-loading-cold-start.md) · [AL2023 VPC CNI 재발 방지 가이드](../al2023-vpc-cni-network-prevention-guide.md)

---

## 0. 요약

- CPU vLLM(`Qwen/Qwen3-4B-Instruct-2507`)을 Karpenter `llm` NodePool(`m7i.2xlarge`)에서 돌리기 위한 **전용 워커 AMI**를 만든다.
- 기존 CPU 워커 AMI(v2, `ami-04e5c592ecea95a37`) 위에 **모델 가중치와 vLLM CPU 이미지를 미리 넣고**, m7i의 주 NIC 이름(`enp39s0`)을 networkd 설정에 추가한다.
- 결과 AMI: **`sayscene-worker-llm-v2` (`ami-019b11c51aea1b31d`)**. ss-k8s `platform/karpenter/ec2nodeclass-llm.yaml`이 참조한다.
- 실측: 새 노드에서 vLLM Ready까지 **13분 20초**(노드 준비 1분 8초), 같은 노드 재기동 **2분 1초**(§7).

| 버전 | AMI ID | 차이 | 결과 |
|---|---|---|---|
| `sayscene-worker-llm-v1` | `ami-0e8d33b291f90152d` | 가중치 + vLLM 이미지 | **m7i에서 노드 조인 실패** (주 NIC가 Unmanaged) |
| `sayscene-worker-llm-v2` | `ami-019b11c51aea1b31d` | v1 + 주 NIC 매칭에 `enp39s0` 추가 | 정상 |

> 실제로는 v1을 만든 뒤 v1을 다시 띄워 NIC 설정만 고쳐 v2를 만들었다. 이 문서는 **베이스 AMI에서 한 번에 v2를 만드는 순서**로 정리했다. 각 명령은 실제로 실행한 것과 같다.

---

## 1. 포함 내용

| 항목 | 값 |
|---|---|
| 베이스 AMI | `ami-04e5c592ecea95a37` (CPU 워커 v2: AL2023, kubeadm·kubelet·containerd·AWS CLI, networkd VPC CNI 설정) |
| 모델 | `Qwen/Qwen3-4B-Instruct-2507` @ revision `cdbee75f17c01a7cc42f958dc650907174af0554` |
| 모델 경로 | `/opt/models/Qwen3-4B-Instruct-2507` (root 소유, 7.6G) |
| vLLM 이미지 | `docker.io/vllm/vllm-openai-cpu:v0.32.0-x86_64` (containerd `k8s.io` 네임스페이스) |
| 이미지 digest | `sha256:827ef38187f9b52f2bb838695ef815ec5b65f4bf3e4c061b002b2b0a19cb0701` (압축 1.6GiB) |
| networkd 변경 | `/etc/systemd/network/10-kubernetes-primary.network`의 `Name=ens5` → `Name=ens5 enp39s0` |
| 루트 볼륨 | 40GiB gp3 (사용 약 19G) |

### 버전 선택 근거

- **모델**: `Qwen3-4B-Instruct-2507`은 thinking 모드가 없는 instruct 전용이라 TTFT가 `<think>` 토큰으로 왜곡되지 않는다. revision을 commit hash로 고정해 재빌드해도 같은 가중치가 들어간다.
- **vLLM**: vLLM에는 LTS가 없다. 2026-10-27 기준 최신 안정 `v0.32.0`(2026-10-18 배포). `nightly`·`latest`는 쓰지 않는다. 문제가 있으면 `v0.31.0-x86_64`로 내린다.

---

## 2. 사전 준비

| 항목 | 내용 |
|---|---|
| 권한 | EC2 인스턴스 생성·정지·종료, AMI 생성·등록 취소, 스냅샷 삭제 |
| 리전 | 서울 `ap-northeast-2` |
| 네트워크 | 인터넷에 나갈 수 있는 서브넷(퍼블릭 IP 또는 NAT). HF Hub·Docker Hub에서 받는다 |
| 키페어 | SSH 접속용. 정리 단계에서 `authorized_keys`를 지우므로 AMI에는 남지 않는다 |
| 정리 스크립트 | ss-k8s `ami/cleanup.sh` (§4-7에 전문 수록) |

---

## 3. 빌더 인스턴스 띄우기 (콘솔)

| 항목 | 값 | 이유 |
|---|---|---|
| AMI | `ami-04e5c592ecea95a37` | CPU 워커 베이스 |
| 인스턴스 타입 | `t3.large` | 다운로드만 하므로 m7i일 필요 없다. 주 NIC는 `ens5` |
| 루트 볼륨 | **40GiB gp3** | 가중치 7.6G + 이미지 압축 해제분. 이 크기가 AMI 스냅샷 크기가 된다 |
| userData | **비운다** | 클러스터에 조인하면 안 된다(§8-1) |
| 퍼블릭 IP | 사용 | SSH와 다운로드 |

> Karpenter가 띄운 노드로 AMI를 만들면 안 된다. 이미 `kubeadm join`이 끝나 `/etc/kubernetes/kubelet.conf`와 노드 인증서가 남고, 이 AMI로 뜬 새 노드는 userData가 "이미 가입한 노드"로 판단해 조인을 건너뛴다.

SSH 접속 후 `ec2-user`로 진행한다.

---

## 4. 빌드 절차 (빌더 인스턴스에서)

### 4-1. 다운로드 도구 설치

AL2023에는 pip가 기본으로 없다.

```bash
sudo dnf install -y python3-pip
python3 -m pip install --user -U huggingface_hub
```

설치된 `hf` CLI는 `~/.local/bin`에 있다. `hf: command not found`면 `~/.local/bin/hf`로 실행한다. (실측 당시 `huggingface_hub` 1.8.0)

### 4-2. 모델 revision 확인

```bash
python3 -c "from huggingface_hub import HfApi; print(HfApi().model_info('Qwen/Qwen3-4B-Instruct-2507').sha)"
```

실측 결과: `cdbee75f17c01a7cc42f958dc650907174af0554`. 같은 AMI를 재현하려면 이 값을 그대로 쓴다. 최신으로 갱신하려면 출력값을 쓰고 기록한다.

### 4-3. 가중치 다운로드

`/opt`는 root 소유라 폴더를 만들고 권한을 준다.

```bash
sudo mkdir -p /opt/models && sudo chown ec2-user:ec2-user /opt/models
```

```bash
REV=cdbee75f17c01a7cc42f958dc650907174af0554; hf download Qwen/Qwen3-4B-Instruct-2507 --revision "$REV" --local-dir /opt/models/Qwen3-4B-Instruct-2507
```

- `HF_TOKEN` 경고는 무시해도 된다. Qwen은 Apache 2.0이라 토큰 없이 받는다.
- 실측: 13개 파일, 8.06GB, 약 70초(약 135MB/s).

확인 후 소유자를 root로 되돌린다. Pod는 hostPath를 읽기 전용으로 마운트한다.

```bash
du -sh /opt/models/Qwen3-4B-Instruct-2507
```

```bash
sudo chown -R root:root /opt/models
```

실측: `7.6G` (GB/GiB 단위 차이로 8.06GB와 같은 크기).

### 4-4. vLLM CPU 이미지 미리 받기

베이스 AMI의 userData가 조인 직전에 containerd를 시작하므로, 빌더에서는 직접 시작한다.

```bash
sudo systemctl start containerd
```

```bash
sudo ctr -n k8s.io images pull docker.io/vllm/vllm-openai-cpu:v0.32.0-x86_64
```

```bash
sudo ctr -n k8s.io images ls | grep vllm-openai-cpu
```

- **`-n k8s.io`를 빠뜨리면 안 된다.** kubelet은 이 네임스페이스만 본다.
- 결과에 `io.cri-containerd.image=managed` 라벨과 digest `sha256:827ef381…0701`, `1.6 GiB`, `linux/amd64`가 보이면 정상.
- 실측: 76.3초.

### 4-5. 디스크 확인

```bash
df -h /
```

실측: `40G` 중 `19G` 사용, `22G` 남음.

### 4-6. 주 NIC 매칭에 `enp39s0` 추가

베이스 AMI의 networkd 설정은 다음과 같다.

```
10-kubernetes-primary.network   [Match] Name=ens5  Driver=ena   → DHCP (주 NIC)
20-kubernetes-cni-eni.network   [Match] Driver=ena             → Unmanaged=yes (보조 ENI는 VPC CNI에 맡김)
networkd.conf.d/90-kubernetes-cni.conf  ManageForeignRoutes=no, ManageForeignRoutingPolicyRules=no
```

networkd는 이름순으로 **처음 일치하는 파일 하나**만 적용한다. m7i의 주 NIC는 `enp39s0`이라 10번에 맞지 않고 20번(Unmanaged)에 걸려 네트워크가 없어진다([이슈 01](../issue/20261027/01-llm-ami-ens5-primary-nic.md)). 10번 매칭에 이름을 **추가**한다.

```bash
sudo grep -rn "" /etc/systemd/network/ /etc/systemd/networkd.conf.d/
```

```bash
sudo sed -i 's/^Name=ens5$/Name=ens5 enp39s0/' /etc/systemd/network/10-kubernetes-primary.network
```

```bash
sudo sed -n 1,3p /etc/systemd/network/10-kubernetes-primary.network
```

기대 출력:

```
[Match]
Name=ens5 enp39s0
Driver=ena
```

- `enp39s0`만 남기면 다음 재빌드 때 t3 빌더(`ens5`)가 네트워크를 잃는다.
- `Name=en*` 같은 와일드카드는 보조 ENI까지 DHCP로 잡아 VPC CNI 경로 문제를 다시 만들 수 있다.
- 20번 파일과 90번 설정은 건드리지 않는다.

설정을 다시 읽고 t3에서 깨지지 않았는지 확인한다.

```bash
sudo networkctl reload && networkctl list
```

- 직후에는 `ens5 degraded configuring`으로 보일 수 있다. DHCP를 다시 받는 중이다. 10~20초 뒤 다시 확인한다.
- `ens5 routable configured`이면 정상.

**재부팅 검증** (정리 전에 한다. 정리 후에는 SSH 키가 없어 재접속할 수 없다):

```bash
sudo reboot
```

1~2분 뒤 재접속해서 `networkctl list`가 `ens5 routable configured`인지 확인한다.

### 4-7. 정리

ss-k8s `ami/cleanup.sh`를 빌더의 `~/cleanup.sh`로 옮긴다(예: 로컬에서 `cat` 출력 후 빌더에서 `cat > ~/cleanup.sh`에 붙여넣고 `Ctrl+D`). 전문:

```bash
#!/bin/bash
# llm 워커 AMI 생성 직전 빌더 인스턴스 정리. 패키지는 건드리지 않는다.
# 사용: sudo bash cleanup.sh → 재접속하지 말고 콘솔에서 정지 후 AMI 생성.
set -euo pipefail

MODEL_DIR=/opt/models/Qwen3-4B-Instruct-2507
VLLM_IMAGE=docker.io/vllm/vllm-openai-cpu:v0.32.0-x86_64
USER_HOME=/home/ec2-user

[[ $EUID -eq 0 ]] || { echo "sudo로 실행하세요." >&2; exit 1; }

# 1. 사전 검사: 조인 흔적이 있으면 새 노드가 가입을 건너뛴다.
[[ ! -e /etc/kubernetes/kubelet.conf ]] || { echo "kubelet.conf 존재: 조인된 인스턴스입니다." >&2; exit 1; }
[[ -f "$MODEL_DIR/config.json" ]] || { echo "모델 파일이 없습니다: $MODEL_DIR" >&2; exit 1; }
ctr -n k8s.io images ls -q | grep -x "$VLLM_IMAGE" >/dev/null || { echo "vLLM 이미지가 없습니다: $VLLM_IMAGE" >&2; exit 1; }

# 2. 다운로드 캐시 (가중치 본체와 패키지는 유지)
rm -rf "$MODEL_DIR/.cache" "$USER_HOME/.cache/huggingface" "$USER_HOME/.cache/pip"
dnf clean all

# 3. 인스턴스 고유 정보: 새 인스턴스 첫 부팅 때 재생성된다.
cloud-init clean --logs
truncate -s 0 /etc/machine-id

# 4. 빌더 접속 키와 셸 기록
rm -f "$USER_HOME/.ssh/authorized_keys" /root/.ssh/authorized_keys
rm -f "$USER_HOME/.bash_history" /root/.bash_history

# 5. 로그와 임시 파일
journalctl --rotate
journalctl --vacuum-time=1s
find /var/log -type f -name '*.gz' -delete
find /tmp /var/tmp -mindepth 1 -delete

echo "== 남은 모델: $(du -sh "$MODEL_DIR" | cut -f1)"
echo "== vLLM 이미지: $VLLM_IMAGE"
echo "완료. 재접속하지 말고 콘솔에서 인스턴스를 정지한 뒤 AMI를 생성하세요."
```

실행:

```bash
sudo bash ~/cleanup.sh && history -c
```

| 정리 항목 | 이유 |
|---|---|
| `kubelet.conf` 검사 | 조인된 인스턴스면 아무것도 지우지 않고 멈춘다 |
| `$MODEL_DIR/.cache` | `hf download --local-dir`가 남기는 메타데이터. vLLM에 불필요 |
| `cloud-init clean` | 새 인스턴스에서 userData·SSH 호스트 키 생성이 처음부터 실행되게 한다 |
| `machine-id` 비우기 | 노드마다 다른 ID를 재생성. 같으면 DHCP·journald에서 충돌할 수 있다 |
| `authorized_keys` | 빌더 키로 모든 llm 노드에 접속되는 것을 막는다 |
| 패키지 | **지우지 않는다.** `python3-pip`·`hf` CLI가 남아도 노드 동작에 영향 없다 |

- `~/cleanup.sh` 파일 자체는 AMI에 남는다. 비밀값이 없고 재빌드 때 다시 쓰므로 남겨 둔다. 남기지 않으려면 `sudo bash ~/cleanup.sh && rm -f ~/cleanup.sh && history -c`.
- `완료.`가 나오면 **재접속하지 않는다.**

---

## 5. AMI 생성 (콘솔)

1. EC2 → 인스턴스 → 빌더 선택 → **인스턴스 상태 → 인스턴스 중지**.
2. **작업 → 이미지 및 템플릿 → 이미지 생성**:

| 항목 | 값 |
|---|---|
| 이미지 이름 | `sayscene-worker-llm-v2` |
| 이미지 설명 | `v1 + enp39s0 primary NIC match (m7i). Qwen3-4B-Instruct-2507@cdbee75, vllm-openai-cpu:v0.32.0-x86_64` |
| 재부팅 안 함 | 상관없음 (정지 상태) |
| 볼륨 | **40GiB 그대로**, gp3, 종료 시 삭제 체크, 암호화 기본값 |
| 태그 (선택) | `Name=sayscene-worker-llm-v2`, `Project=sayscene` |

3. EC2 → AMI에서 상태가 **`available`** 이 될 때까지 기다린다(40GiB 스냅샷, 보통 5~15분).
4. `available` 확인 후 빌더 **인스턴스 종료**. EC2 → 볼륨에 `available` 상태로 남은 40GiB 볼륨이 있으면 삭제한다.

> AMI 스냅샷 크기(40GiB)는 EC2NodeClass `volumeSize`보다 작거나 같아야 한다. llm은 60Gi라 문제없다. **general EC2NodeClass(20Gi)에는 이 AMI를 쓸 수 없다.**

---

## 6. 클러스터 반영

### 6-1. EC2NodeClass

ss-k8s `platform/karpenter/ec2nodeclass-llm.yaml`의 AMI를 바꾸고 main에 머지한다(실제: ss-k8s `53a51c7`).

```yaml
  amiSelectorTerms:
    - id: ami-019b11c51aea1b31d
```

**AMI가 `available`이 되기 전에 머지하지 않는다.** Karpenter가 AMI를 찾지 못해 EC2NodeClass가 Ready가 되지 않는다.

Argo CD `karpenter` Application Sync 후 확인:

```bash
kubectl get ec2nodeclass llm -o jsonpath='{.spec.amiSelectorTerms[0].id}{"\n"}'
kubectl get ec2nodeclass llm
```

### 6-2. 관련 설정값 (참고)

| 위치 | 값 |
|---|---|
| EC2NodeClass `llm` | 이 AMI, `volumeSize: 60Gi`, userData는 general과 같음(SSM에서 join 정보 조회 → `kubeadm join`) |
| NodePool `llm` | `m7i.2xlarge` on-demand, limits cpu 8 / memory 32Gi(노드 최대 1대), taint `sayscene.io/workload=llm:NoSchedule`, `WhenEmpty` 1m |
| Deployment `llm` (`apps/llm`) | 이미지 `vllm/vllm-openai-cpu:v0.32.0-x86_64` `IfNotPresent`, hostPath `/opt/models` 읽기 전용, args `/opt/models/Qwen3-4B-Instruct-2507 --served-model-name=qwen3-4b --dtype=bfloat16 --max-model-len=8192`, `VLLM_CPU_KVCACHE_SPACE=8`, `HF_HUB_OFFLINE=1`, cpu 6 / memory 24Gi(limit 28Gi), `Recreate`, startupProbe 최대 15분 |

- Deployment의 이미지 태그는 AMI에 넣은 태그와 **정확히 같아야** 노드에서 다시 받지 않는다.
- `--max-model-len`이 없으면 기본 컨텍스트(262k)가 KV cache 8GiB에 들어가지 않아 기동에 실패한다. 8192는 ss-be 리포트 요청의 `max_tokens: 4096`을 수용하기 위한 값이다.

### 6-3. 기동

replicas를 1로 올리면 Pending Pod를 보고 Karpenter가 m7i 노드를 띄운다.

```bash
kubectl get nodeclaim -w
```

| 열 | 의미 |
|---|---|
| `LAUNCHED=True` | EC2 생성 완료 |
| `REGISTERED=True` | 노드 조인 완료 (`kubectl get nodes`에 보임) |
| `INITIALIZED=True` | Ready, `karpenter.sh/unregistered` taint 제거 |

```bash
kubectl -n sayscene get pod -l app=llm -o wide -w
kubectl -n sayscene logs deploy/llm -f --tail=50
```

`Starting vLLM server on http://0.0.0.0:8000`이 나오고 Pod가 `1/1`이면 완료. 동작 확인:

```bash
kubectl -n sayscene exec deploy/llm -- curl -s localhost:8000/v1/models
kubectl -n sayscene exec deploy/llm -- curl -s localhost:8000/v1/chat/completions -H 'Content-Type: application/json' -d '{"model":"qwen3-4b","max_tokens":128,"messages":[{"role":"user","content":"Hi! I would like to order a latte, please."}]}'
```

> Argo CD UI에서만 replicas를 올렸다면 다음 Sync 때 Git 값(`0`)으로 돌아간다. 확인 후 Git도 맞춘다.

---

## 7. 실측: 노드 기동 시간 (2026-10-27)

시각은 Pod conditions(`lastTransitionTime`), NodeClaim `creationTimestamp`, vLLM 로그 기준.

```bash
kubectl get nodeclaim -l karpenter.sh/nodepool=llm -o jsonpath='{.items[0].metadata.creationTimestamp}{"\n"}'
kubectl -n sayscene get pod -l app=llm -o jsonpath='{.items[0].metadata.creationTimestamp}{"\tCreated\n"}{range .items[0].status.conditions[*]}{.lastTransitionTime}{"\t"}{.type}{"\n"}{end}' | sort
kubectl -n sayscene logs deploy/llm | grep -E "DP group leader|Loading weights took|Model loading took|Warming up|Starting vLLM server"
```

| 구간 | 첫 기동 (새 노드) | 재기동 (같은 노드, `rollout restart`) |
|---|---|---|
| Pod 생성 → NodeClaim 생성 | 1초 (10:22:00 → 10:22:01) | — |
| **노드 준비** (EC2 부팅·조인·Ready) | **1분 8초** (→ 10:23:09 PodScheduled) | — |
| Pod 네트워크 생성 | 17초 (CNI 재시도 1회 포함) | 2초 |
| 컨테이너 시작 → vLLM 엔진 시작 | 약 2분 33초 | 26초 |
| 가중치 로딩 (`Loading weights took`) | **322.8초** | **1.09초** |
| 워밍업·컴파일 | 3분 39초 | 1분 15초 |
| **PodScheduled → Ready** | **12분 11초** | **2분 1초** |
| **전체** | **13분 20초** (10:22:00 → 10:35:20) | 2분 1초 (10:40:00 → 10:42:01) |

해석:

- **노드 준비(1분 8초)는 빠르다.** AMI·조인 경로는 정상이다.
- **나머지 약 10분은 EBS 첫 읽기 비용이다.** AMI 스냅샷에서 만든 볼륨은 블록을 처음 읽을 때 S3에서 가져온다(lazy loading). 가중치뿐 아니라 이미지 레이어(Python·torch)와 워밍업 중 읽는 파일도 같은 볼륨이다. 첫 기동 가중치 읽기는 약 20MB/s로, 빌더의 HF 다운로드(약 135MB/s)보다 느렸다.
- **재기동 가중치 1.09초는 노드 페이지 캐시에서 읽은 값이다.** 재기동 2분 1초는 "같은 노드 + 메모리 캐시"의 최선값이다.
- 메모리: 재기동 기준 `Model loading took 9.24 GiB`(첫 기동 16.75GiB는 로딩 중 집계). KV cache 8GiB를 더해도 limit 28Gi 이내.
- CPU: `VLLM_CPU_OMP_THREADS_BIND=auto`가 물리 코어 4개 중 3개(`core ids=[4, 5, 6]`)를 추론에, 1개(`reserved_cpus=[7]`)를 예약했다.

개선 후보(미적용): 기동 시 S3·HF에서 가중치 받기, 이미지를 레지스트리에서 pull, Fast Snapshot Restore(AZ당 시간 과금), `--safetensors-load-strategy=prefetch`. 상세는 [이슈 02](../issue/20261027/02-ebs-lazy-loading-cold-start.md).

---

## 8. 트러블슈팅

### 8-1. 노드가 `REGISTERED`가 되지 않음

`kubectl get nodeclaim`에서 `READY Unknown`, NODE가 비어 있고 이벤트가 `Launched: True` → `DisruptionBlocked ... does not have an associated node`면 인스턴스 내부(부팅·userData) 문제다. SSH 키가 없으므로 **EC2 콘솔 → 인스턴스 → 작업 → 모니터링 및 문제 해결 → 시스템 로그 가져오기**로 본다. 15분 안에 등록되지 않으면 Karpenter가 인스턴스를 지우고 다시 띄우므로, 원인을 찾을 때까지 replicas를 0으로 내린다.

| 시스템 로그 | 원인 |
|---|---|
| `Net device info`에서 주 NIC가 `False`, `FAILED ... policy-routes for <nic>`, IMDS `Network is unreachable` | 주 NIC 이름이 `10-kubernetes-primary.network`에 없음(§4-6). v1에서 발생 |
| `이미 가입한 노드입니다` | AMI에 `kubelet.conf`가 남음. Karpenter 노드로 AMI를 만든 경우 |
| SSM `get-parameter` 실패 | 인스턴스 프로필 권한 또는 네트워크 |
| `kubeadm join` 타임아웃 | 서브넷에서 control plane `:6443` 연결 불가 |

### 8-2. `FailedCreatePodSandBox ... failed to setup network policy`

노드 Ready 직후 1회 발생. 노드의 network policy agent(`aws-node`의 `aws-eks-nodeagent`)가 준비되기 전 호출된 것으로, kubelet 재시도로 해결됐다. 반복되면 다음을 본다.

```bash
kubectl -n kube-system get pod -l k8s-app=aws-node -o wide | grep <노드명>
kubectl -n kube-system logs <aws-node Pod> -c aws-eks-nodeagent --tail=30
```

### 8-3. 기동이 15분을 넘김

첫 기동은 약 12분이 걸리고 startupProbe 한도는 15분이다. 가중치 로딩과 워밍업이 더 느려지면 Pod가 재시작된다(`RESTARTS 1`). 로그의 `Loading weights took`·`Warming up done` 시각을 보고 `failureThreshold`를 늘린다.

---

## 9. 다른 인스턴스 타입으로 바꿀 때

- **주 NIC 이름을 먼저 확인한다.** `enp39s0`은 m7i.2xlarge에서 확인한 값이다. 다른 크기·계열은 다를 수 있다. 새 타입으로 빌더(또는 테스트 인스턴스)를 띄워 `ip -o link`로 이름을 보고 §4-6의 `Name=` 목록에 추가해 재빌드한다.
- general 워커 AMI(`ami-04e5c592ecea95a37`)도 `ens5` 고정이다. general NodePool에 t3 외 계열(특히 7세대)을 추가하면 같은 장애가 난다.

---

## 10. 재빌드 (모델·vLLM 버전 변경)

1. §3처럼 **현재 llm AMI(`ami-019b11c51aea1b31d`)** 를 t3.large로 띄운다(NIC 설정·이미지가 이미 들어 있다).
2. 모델 변경: §4-1~4-3을 새 revision으로 실행하고 기존 모델 폴더는 지운다. vLLM 변경: §4-4를 새 태그로 실행하고 이전 이미지는 `sudo ctr -n k8s.io images rm <이미지>`로 지운다.
3. `cleanup.sh`의 `MODEL_DIR`·`VLLM_IMAGE`와 `apps/llm`의 args·image를 새 값으로 맞춘다.
4. §4-5 디스크 확인 → §4-7 정리 → §5 AMI 생성(`sayscene-worker-llm-v3`) → §6 반영.
5. 새 AMI로 노드 조인이 확인되면 이전 AMI를 **등록 취소**하고 연결된 스냅샷을 삭제한다(v1 `ami-0e8d33b291f90152d` 포함).
