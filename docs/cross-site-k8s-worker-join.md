# Cross-site K8s Worker Join: AWS EC2 → 홈랩 클러스터

## 개요

홈랩 K8s 클러스터(10.18.0.0/16)에 AWS EC2 워커 노드(10.0.10.100)를 Tailscale Subnet Router를 통해 join하는 과정에서 발견된 문제와 해결 방법을 정리한다.

```
홈랩 (10.18.0.0/16)                          AWS VPC (10.0.0.0/16)
┌─────────────────────────────┐              ┌─────────────────────────────┐
│ CP: Pi-0/1/2 (10.18.1.10-12)│              │ Tailscale SR (public subnet)│
│ Worker: Pi-s0/s1, notebook  │  Tailscale   │                             │
│                             │ ◄══════════► │ K8s Worker (10.0.10.100)    │
│ VIP: 10.18.1.1 (kube-vip)  │  WireGuard   │   (private subnet)          │
│ Pod CIDR: 10.244.0.0/16    │              │   Pod CIDR: 10.244.6.0/24   │
└─────────────────────────────┘              └─────────────────────────────┘
  Tailscale SR: Pi-0/1/2 (HA, 3대)
  advertise: 10.18.0.0/16
```

## Join 절차

### 1. 인벤토리 등록

`kubernetes/inventory/baremetal`의 `[new-k8s-workers]` 그룹에 AWS 워커 추가:

```ini
[new-k8s-workers]
aws-ec2-worker00 ansible_host=10.0.10.100 ansible_user=ubuntu
```

### 2. Playbook 실행

```bash
cd kubernetes
ansible-playbook -i inventory/baremetal playbooks/add-new-workers-all.yaml
```

적용되는 역할: `repos` → `ntpd` → `crio` → `ufw` → `kubeadm` → `kubeadm-join-node`

## 발견된 문제: Flannel VXLAN Pod-to-Pod 통신 실패

### 증상

- 노드 join 및 Ready 상태 전환: 성공
- LoadBalancer 서비스 (kube-vip): 동작함 (homelab에서 AWS Pod 접근 가능)
- **Pod-to-Pod VXLAN 양방향 통신: 실패** (100% packet loss)
- Longhorn DaemonSet Pod들 CrashLoopBackOff

### 원인 분석: Tailscale HA Primary SR과 라우터 게이트웨이 불일치

#### 패킷 추적 결과

```
ubuntu-notebook → 10.0.10.100 ping

① ubuntu-notebook (10.18.3.100/24)
   ip route get 10.0.10.100 → via 10.18.0.1 (공유기)         ✓ 패킷 출발

② 공유기 (10.18.0.1)
   정적 경로: 10.0.0.0/16 → 10.18.1.1 (kube-vip VIP)
   L2 ARP → VIP 보유 노드로 전달                              ✓ 라우팅

③ raspberrypi-0 (VIP 10.18.1.1 보유)
   커널 포워딩 → tailscale0 (ip rule 5270, table 52)          ✓ tailscale0 진입
   Tailscale 데몬: 포워딩된 패킷을 WireGuard 터널로 전달하지 않음  ✗ 드랍!
```

#### 근본 원인: Non-primary SR은 포워딩된 트래픽을 터널로 전달하지 않음

Tailscale HA Subnet Router 구성에서 동일 서브넷(10.18.0.0/16)을 3대가 광고할 때, **Primary로 선택된 노드만** 물리 네트워크 → Tailscale 터널 방향의 포워딩된 트래픽(transit traffic)을 전달한다.

```
Tailscale HA 상태:
  raspberrypi-0: advertise 10.18.0.0/16, PrimaryRoutes: N/A     ← VIP 보유
  raspberrypi-1: advertise 10.18.0.0/16, PrimaryRoutes: N/A
  raspberrypi-2: advertise 10.18.0.0/16, PrimaryRoutes: 10.18.0.0/16  ← Primary

문제: VIP 보유 노드 ≠ Tailscale Primary SR
  → 공유기가 VIP로 보낸 패킷이 non-primary 노드에 도착
  → non-primary 노드의 Tailscale 데몬이 transit 패킷 드랍
```

| 구분 | 로컬 발생 트래픽 | 포워딩된 트래픽 (transit) |
|------|-----------------|------------------------|
| Primary SR | ✓ 전달 | ✓ 전달 |
| Non-primary SR | ✓ 전달 | ✗ 드랍 |

#### 검증 방법

```bash
# 1. 현재 Tailscale Primary 확인
tailscale status --json | python3 -c "
import json,sys
d=json.load(sys.stdin)
s=d.get('Self',{})
print(f\"Host: {s.get('HostName')}, PrimaryRoutes: {s.get('PrimaryRoutes','N/A')}\")"

# 2. 어느 노드가 VIP를 보유하는지 확인
for ip in 10.18.1.10 10.18.1.11 10.18.1.12; do
  ssh pi@$ip "ip addr show eth1 | grep 10.18.1.1/" 2>/dev/null && echo "VIP on $ip"
done
```

### 해결: 공유기 정적 경로를 Primary SR IP로 지정

```
변경 전: 10.0.0.0/16 → 10.18.1.1 (kube-vip VIP, 어느 노드인지 불확정)
변경 후: 10.0.0.0/16 → 10.18.1.12 (현재 Tailscale Primary SR)
```

변경 즉시 **노드 간 통신, Pod-to-Pod VXLAN 양방향 모두 성공**.

### 미해결 과제: Primary SR 변동 시 라우팅 깨짐

Tailscale Primary SR은 노드 재시작, 네트워크 장애 등으로 다른 노드로 이동할 수 있다. 공유기 정적 경로가 고정 IP이므로, Primary가 이동하면 cross-site 통신이 다시 끊어진다.

가능한 대안:
1. **공유기 대신 각 K8s 노드에 직접 정적 경로 추가**: 라우터 우회, Ansible로 자동화 가능
2. **Tailscale Primary 고정**: Tailscale ACL로 특정 노드를 Primary로 고정 (admin console에서 설정)
3. **모든 K8s 노드에 Tailscale 설치**: 각 노드가 직접 accept-routes로 AWS 경로를 수신 (리소스 오버헤드)
4. **keepalived/VRRP로 Primary 추적**: Primary SR 변경 시 VIP도 함께 이동하는 스크립트

## 기타 발견 사항

### kube-vip LoadBalancer IP 중복 할당

kube-vip이 동일 VIP(`10.18.100.2`)를 여러 서비스에 할당할 수 있다. 기존 서비스(Nginx Gateway Fabric)와 새 서비스가 같은 IP를 받아 트래픽이 잘못 라우팅된다.

```bash
# 확인
kubectl get svc --all-namespaces --field-selector spec.type=LoadBalancer

# 해결: 특정 IP 지정
spec:
  type: LoadBalancer
  loadBalancerIP: "10.18.100.3"
```

### Flannel VXLAN MTU

Cross-site 통신은 VXLAN(50B) + WireGuard(60B) 이중 캡슐화로 MTU 오버헤드가 크다. Flannel ConfigMap에 MTU를 설정하고 flannel.1 인터페이스 재생성이 필요하다.

```json
"Backend": {
  "Type": "vxlan",
  "MTU": 1390
}
```

flannel.1 인터페이스 재생성 시 Flannel이 자동 계산한 MTU(1340)가 적용될 수 있다 (더 보수적).

### 서브넷 분리 주의

홈랩 네트워크가 VLAN/서브넷 분리되어 있을 경우 (`10.18.1.0/24`, `10.18.2.0/24`, `10.18.3.0/24` 등), 워커 노드에서 SR 노드로의 직접 정적 경로 추가가 불가능하다 (`Nexthop has invalid gateway`). 반드시 공유기를 경유해야 한다.
