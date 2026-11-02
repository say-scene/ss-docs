# 01. llm-v1 AMI로 m7i 노드 조인 실패 — 주 NIC 매칭이 `ens5`로 고정

> 상태: **해결** (`sayscene-worker-llm-v2`, ss-k8s `53a51c7`) · 발생: 2026-10-27

## 증상

llm Deployment를 replicas 1로 올리자 Karpenter가 `m7i.2xlarge`(NodeClaim `llm-dx22m`)를 띄웠으나 노드가 클러스터에 등록되지 않았다.

| 확인 | 결과 |
|---|---|
| `kubectl get nodeclaim` | `READY Unknown`, NODE 비어 있음 (4분 이상) |
| NodeClaim 이벤트 | `Launched: True`, 이후 `DisruptionBlocked ... does not have an associated node` 반복 |
| EC2 콘솔 | 인스턴스는 실행 중 |

`Launched: True`이므로 AMI·서브넷·SG·인스턴스 프로필은 정상이고, 문제는 인스턴스 내부(부팅·userData) 단계다.

## 진단

SSH 키를 AMI 정리 때 제거했으므로 EC2 콘솔의 **시스템 로그 가져오기**로 확인했다.

| 로그 | 의미 |
|---|---|
| `enp39s0 \| False \| . \|` (cloud-init Net device info) | 주 NIC가 down, IP 없음 |
| `FAILED ... policy-routes for enp39s0` | amazon-ec2-net-utils가 인터페이스 설정 실패 |
| `FAILED ... Wait for Network to be Configured` | systemd-networkd가 약 2분 대기 후 포기 |
| cloud-init 138초에 시작, IMDS `Network is unreachable` 반복 | 네트워크가 없어 userData(`kubeadm join`)까지 가지 못함 |

빌더 인스턴스(llm-v1 AMI를 t3.large로 기동)에서 networkd 설정을 확인했다.

```
/etc/systemd/network/10-kubernetes-primary.network   [Match] Name=ens5  Driver=ena   → DHCP
/etc/systemd/network/20-kubernetes-cni-eni.network   [Match] Driver=ena             → Unmanaged=yes
/etc/systemd/networkd.conf.d/90-kubernetes-cni.conf  ManageForeignRoutes=no, ManageForeignRoutingPolicyRules=no
```

## 원인

systemd-networkd는 `.network` 파일을 이름순으로 보고 **처음 일치하는 파일 하나**만 적용한다.

| 인스턴스 | 주 NIC 이름 | 적용된 파일 | 결과 |
|---|---|---|---|
| t3 계열 | `ens5` | `10-kubernetes-primary` | DHCP, 정상 |
| m7i.2xlarge | `enp39s0` | `20-kubernetes-cni-eni` | **Unmanaged → 네트워크 없음** |

이 설정은 [AL2023 VPC CNI 네트워크 재발 방지 가이드](../../al2023-vpc-cni-network-prevention-guide.md)에서 온 것으로, 가이드도 "대상: 기본 ENI가 ens5인 CPU 노드"로 범위를 한정하고 있다. v2 워커 AMI는 지금까지 t3.large에서만 쓰여 드러나지 않았고, 7세대 Nitro 인스턴스(`m7i`)에서 처음 발생했다.

## 해결

llm-v1 AMI를 t3.large 빌더로 띄워 주 NIC 매칭에 `enp39s0`을 **추가**했다(교체가 아님).

```bash
sudo sed -i 's/^Name=ens5$/Name=ens5 enp39s0/' /etc/systemd/network/10-kubernetes-primary.network
```

- `enp39s0`만 남기면 다음 AMI 재빌드 때 t3 빌더(`ens5`)가 네트워크를 잃는다.
- `Name=en*` 같은 와일드카드는 보조 ENI까지 DHCP로 잡아 VPC CNI 규칙 문제를 다시 만들 수 있어 쓰지 않았다.
- `20-kubernetes-cni-eni.network`, `90-kubernetes-cni.conf`는 유지했다.

이후 `ss-k8s/ami/cleanup.sh`로 정리하고 `sayscene-worker-llm-v2`(`ami-019b11c51aea1b31d`)를 만들어 `ec2nodeclass-llm.yaml`에 반영했다(ss-k8s `53a51c7`).

## 검증

| 단계 | 결과 |
|---|---|
| 빌더(t3) `networkctl reload` | `ens5` 잠시 `configuring` 후 `routable configured` |
| 빌더(t3) 재부팅 후 | `ens5 routable configured`, SSH 정상 |
| m7i 노드 기동 (`llm-gr892`) | NodeClaim 생성부터 Pod 스케줄까지 **1분 8초**, 노드 `ip-10-0-8-247` 등록 |

## 남은 일

- `enp39s0`이 모든 m7i 크기·다른 계열에서 같은 이름인지는 확인하지 않았다. llm NodePool의 인스턴스 타입을 바꾸면 다시 확인한다.
- general 워커 AMI(`ami-04e5c592ecea95a37`, v2)도 `ens5` 고정이다. general NodePool에 t3 외 계열(특히 7세대)을 추가하면 같은 장애가 난다.
- 재발 방지 가이드의 대상 범위 문구와 AMI 생성 절차에 "인스턴스 계열별 주 NIC 이름 확인"을 추가한다.
- llm-v1 AMI·스냅샷은 v2 동작이 확인됐으므로 등록 취소·삭제한다.
