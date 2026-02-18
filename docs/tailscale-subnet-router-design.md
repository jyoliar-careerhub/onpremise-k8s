# Tailscale Subnet Router 방식의 K8s 클러스터 구축 검토

## 1. 개요

### 1.1 이전 보고서와의 관계

| 보고서 | 방식 | 핵심 문제 |
|--------|------|-----------|
| `tailscale-vpn-integration.md` | 노드마다 Tailscale 설치 | 치킨-에그 문제, `k8s_ip` 변수 도입 필요 |
| `tailscale-two-workflow-design.md` | 워크플로우 분리 (VPN + K8s) | 템플릿 8곳 수정, kube-vip 대체 필요 |
| **본 보고서** | **Subnet Router** | **K8s 노드에 Tailscale 미설치, 기존 코드 최소 변경** |

### 1.2 핵심 아이디어

Tailscale을 K8s 노드에 설치하지 않고, 각 사이트의 **게이트웨이 장치에만** 설치하여 사이트 간 라우팅을 제공한다.

- K8s 노드는 기존 LAN IP 하나만 사용
- `ansible_host`를 그대로 SSH와 K8s 네트워킹 모두에 사용
- 템플릿 수정, 워크플로우 분리, `k8s_ip` 변수 도입 **불필요**

---

## 2. Subnet Router 기본 개념

### 2.1 동작 원리

Subnet Router는 자신이 속한 서브넷을 tailnet(Tailscale 네트워크)에 **광고(advertise)**한다. tailnet의 다른 장치들은 이 광고된 서브넷의 IP로 직접 통신할 수 있다.

```
[tailnet 장치] ── WireGuard 터널 ──→ [Subnet Router] ──→ [서브넷 내 장치들]
   100.x.x.x                          10.18.1.1           10.18.1.10~12
                                    (tailnet + LAN)        (LAN only)
```

- Subnet Router만 Tailscale 클라이언트를 실행
- 서브넷 내 장치(K8s 노드)에는 Tailscale 설치 불필요
- IP Forwarding을 통해 패킷을 중계

### 2.2 Exit Node와의 차이

| | Subnet Router | Exit Node |
|---|---|---|
| 라우팅 범위 | 지정된 서브넷만 | 전체 인터넷 트래픽 |
| CLI 플래그 | `--advertise-routes=CIDR` | `--advertise-exit-node` |
| 사용 목적 | 내부 네트워크 접근 | 보안 인터넷 접속 |
| K8s 용도 | **적합** | 부적합 |

---

## 3. 네트워크 아키텍처

### 3.1 시나리오 A: 자택 단일 사이트

현재 `inventory/baremetal`의 모든 노드가 자택 LAN에 위치한 구성.

```
[Tailscale SaaS]
      │
      │ WireGuard
      │
[자택 공유기 / 전용 Pi]
  Tailscale Subnet Router
  --advertise-routes=10.18.0.0/16
      │
      ├── 10.18.1.0/24 (마스터)
      │   ├── Pi5-0 (10.18.1.10)
      │   ├── Pi5-1 (10.18.1.11)
      │   └── Pi5-2 (10.18.1.12)
      │
      ├── 10.18.2.0/24 (워커 - Pi)
      │   ├── Pi-s0 (10.18.2.10)
      │   └── Pi-s1 (10.18.2.11)
      │
      └── 10.18.3.0/24 (워커 - 노트북)
          └── ubuntu-notebook (10.18.3.100)

[외부]
  사용자 Mac (Tailscale 클라이언트)
  → tailscale up --accept-routes
  → 10.18.x.x 대역으로 K8s 노드 직접 접근
```

**이 시나리오에서는:**
- Subnet Router 하나만 필요 (자택 공유기 또는 전용 Pi)
- K8s 노드 간 통신은 기존 LAN (L2, 동일 네트워크)
- kube-vip ARP 모드 **그대로 사용 가능** (같은 L2)
- 외부에서 kubectl 접근 시 VIP(10.18.1.1) 또는 마스터 IP 직접 사용

### 3.2 시나리오 B: 자택 + AWS 멀티사이트

향후 AWS EC2 워커 노드를 추가하는 구성.

```
[자택 LAN: 10.18.0.0/16]                    [AWS VPC: 10.0.1.0/24]

  Pi5-0 (10.18.1.10) ─┐                      ┌─ EC2-w1 (10.0.1.10)
  Pi5-1 (10.18.1.11) ─┤                      ├─ EC2-w2 (10.0.1.11)
  Pi5-2 (10.18.1.12) ─┤                      │
  Pi-s0 (10.18.2.10) ─┤                      │
  Pi-s1 (10.18.2.11) ─┤                      │
  Notebook (10.18.3.100)                      │
           │                                  │
  [자택 Subnet Router]    ── Tailscale ──    [AWS Subnet Router]
  advertise 10.18.0.0/16                     advertise 10.0.1.0/24
  --snat-subnet-routes=false                 --snat-subnet-routes=false
           │                                  │
   자택 공유기 / Pi                      전용 EC2 (t3.micro)
```

**서브넷 겹침 확인:**
- 자택: `10.18.0.0/16`
- AWS: `10.0.0.0/16` (일반적 VPC CIDR)
- **겹치지 않음** → Subnet Router 방식 적용 가능

**라우팅 흐름 (Pi5 → EC2):**
```
Pi5 (10.18.1.10)
  → 패킷: dst=10.0.1.10
  → default gw = 자택 공유기 (10.18.x.1)
  → 공유기: 10.0.0.0/16은 Tailscale 터널로 전달
  → Tailscale WireGuard 터널
  → AWS Subnet Router
  → EC2 (10.0.1.10)
```

**라우팅 흐름 (EC2 → Pi5):**
```
EC2 (10.0.1.10)
  → 패킷: dst=10.18.1.10
  → VPC Route Table: 10.18.0.0/16 → AWS Subnet Router ENI
  → AWS Subnet Router
  → Tailscale WireGuard 터널
  → 자택 공유기/Subnet Router
  → Pi5 (10.18.1.10)
```

---

## 4. Subnet Router 설정 상세

### 4.1 공통 요구사항 (Linux)

**커널 파라미터:**

```bash
# /etc/sysctl.d/99-tailscale.conf

# IP Forwarding (필수)
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1

# Reverse Path Filter - loose mode (필수)
# Tailscale의 tun 인터페이스 패킷이 드롭되지 않도록
net.ipv4.conf.all.rp_filter = 2
net.ipv4.conf.default.rp_filter = 2
```

**Tailscale 설치 및 설정:**

```bash
# 1. Tailscale 설치
curl -fsSL https://tailscale.com/install.sh | sh

# 2. Subnet Router로 시작
sudo tailscale up \
  --authkey=tskey-auth-xxxxx \
  --advertise-routes=10.18.0.0/16 \
  --snat-subnet-routes=false \
  --advertise-tags=tag:subnet-router

# 3. (관리 콘솔 또는 autoApprovers로) Route 승인
```

### 4.2 자택 Subnet Router 구성 방안

세 가지 옵션을 검토한다.

#### 방안 A: 공유기에 직접 설치 (OpenWrt)

```
[공유기 (OpenWrt)]
  ├── Tailscale daemon
  ├── --advertise-routes=10.18.0.0/16
  └── 이미 K8s 노드들의 default gateway
```

| 장점 | 단점 |
|------|------|
| 추가 장치 불필요 | 공유기가 OpenWrt 지원 필수 |
| 이미 default gateway → 라우팅 자동 해결 | 공유기 CPU/메모리 부하 (암호화 처리) |
| 네트워크 경로 최단 | 공유기 교체/재부팅 시 K8s + VPN 동시 중단 |

사용자가 공유기를 커스텀할 수 있다고 했으므로, **가장 단순한 방안**.

공유기가 이미 모든 K8s 노드의 default gateway이므로, K8s 노드에 별도 라우팅 설정이 **전혀 필요 없다**.

#### 방안 B: 전용 Pi를 Subnet Router로 사용

```
[공유기]
  ├── route: 10.0.0.0/16 → Pi-SR (10.18.x.x)
  └── K8s 노드들

[Pi-SR (전용)]
  ├── Tailscale daemon
  ├── --advertise-routes=10.18.0.0/16
  └── IP Forwarding 활성화
```

| 장점 | 단점 |
|------|------|
| 공유기 부하 없음 | 추가 장치(Pi) 필요 |
| 전용 리소스 | 공유기에 정적 라우트 추가 필요 |
| 장애 범위 분리 | 추가 전력/공간 |

공유기에 `10.0.0.0/16 → Pi-SR` 정적 라우트를 추가해야 K8s 노드가 AWS 방향 트래픽을 올바르게 라우팅한다.

#### 방안 C: K8s 노드 중 하나가 겸임

```
[공유기]
  ├── route: 10.0.0.0/16 → Pi5-0 (10.18.1.10)
  └── K8s 노드들

[Pi5-0 (마스터 + Subnet Router)]
  ├── Tailscale daemon
  ├── K8s master 역할
  └── Subnet Router 역할
```

| 장점 | 단점 |
|------|------|
| 추가 장치 불필요 | K8s 노드에 Tailscale 설치됨 (이중 역할) |
| | 해당 노드 장애 시 K8s + VPN 동시 영향 |
| | K8s 노드의 리소스 공유 |

**비권장** — K8s 노드와 네트워크 인프라의 관심사가 혼재됨.

#### 권장: 방안 A (공유기) > 방안 B (전용 Pi)

공유기를 커스텀할 수 있으므로 방안 A가 가장 단순. 공유기 성능이 부족하면 방안 B로 전환.

### 4.3 AWS Subnet Router 구성

**전용 EC2 인스턴스:**

| 항목 | 설정 |
|------|------|
| AMI | Ubuntu 24.04 LTS |
| Instance Type | `t3.micro` (저트래픽) ~ `t3.small` |
| Subnet | Public subnet (Tailscale 제어 서버 접근 필요) |
| **Source/Dest Check** | **반드시 비활성화** |

```bash
# Source/Dest Check 비활성화 (AWS CLI)
aws ec2 modify-instance-attribute \
  --instance-id i-xxxxxxxxxxxx \
  --source-dest-check '{"Value": false}'
```

**Security Group:**

| 방향 | 포트 | 용도 |
|------|------|------|
| Outbound | UDP 41641 | WireGuard (direct) |
| Outbound | TCP/UDP 443 | DERP relay fallback |
| Outbound | TCP 80/443 | Tailscale 제어 서버 |
| Inbound | VPC CIDR 전체 | K8s 노드 → 자택으로의 트래픽 수신 |

**VPC Route Table:**

```
Destination        Target
10.18.0.0/16       eni-xxxx (AWS Subnet Router의 ENI)
```

이 라우트가 있어야 VPC 내 EC2 인스턴스들이 자택 LAN(10.18.x.x)에 도달 가능.

---

## 5. SNAT 설정

### 5.1 SNAT이란

Tailscale Subnet Router는 기본적으로 **SNAT(Source NAT)**을 적용한다. 서브넷 내 장치는 트래픽의 출발지가 subnet router의 IP로 보인다.

```
SNAT 활성화 (기본):
  Pi5 (10.18.1.10) → [Home SR] → tailnet → [AWS SR] → EC2
  EC2가 보는 source: AWS SR의 VPC IP (예: 10.0.1.99)

SNAT 비활성화:
  Pi5 (10.18.1.10) → [Home SR] → tailnet → [AWS SR] → EC2
  EC2가 보는 source: 10.18.1.10 (실제 IP)
```

### 5.2 K8s에서의 SNAT 선택

**K8s에는 SNAT 비활성화(`--snat-subnet-routes=false`) 권장:**

| 고려 항목 | SNAT 활성화 | SNAT 비활성화 |
|-----------|:-:|:-:|
| 라우팅 설정 간소화 | 부분적 | - |
| 실제 소스 IP 보존 | X | **O** |
| 디버깅/로깅 | 불투명 | **투명** |
| etcd peer URL | 동작 (TCP 추적) | **동작** |
| kubelet InternalIP | 동작 (TCP 추적) | **동작** |
| VPC Route Table 변경 | 필요 | 필요 |

SNAT 활성화 여부와 무관하게 **양방향 라우팅 설정은 필요**하므로, SNAT을 비활성화하여 투명한 네트워킹을 확보하는 것이 유리하다.

### 5.3 SNAT 비활성화 시 추가 요구사항

SNAT을 비활성화하면, 서브넷 내 장치들이 원격 서브넷으로의 **응답 경로(return route)**를 알아야 한다. 그러나:

- **자택**: K8s 노드의 default gateway가 공유기(= Subnet Router) → **추가 설정 불필요**
- **AWS**: VPC Route Table에 `10.18.0.0/16 → SR ENI` 추가 → EC2가 자동으로 SR을 통해 응답

따라서 양쪽 모두 이미 라우팅 설정으로 해결되므로, SNAT 비활성화에 따른 추가 작업은 없다.

---

## 6. Tailscale ACL 및 인증

### 6.1 ACL 정책 (Tailscale Admin Console)

```json
{
  "tagOwners": {
    "tag:subnet-router": ["autogroup:admin"]
  },
  "autoApprovers": {
    "routes": {
      "10.18.0.0/16": ["tag:subnet-router"],
      "10.0.0.0/16": ["tag:subnet-router"]
    }
  },
  "acls": [
    {
      "action": "accept",
      "src": ["tag:subnet-router"],
      "dst": ["tag:subnet-router:*"]
    },
    {
      "action": "accept",
      "src": ["autogroup:members"],
      "dst": ["10.18.0.0/16:*", "10.0.0.0/16:*"]
    }
  ]
}
```

`autoApprovers`를 설정하면 Subnet Router가 route를 광고할 때 **관리 콘솔에서 수동 승인할 필요가 없다**.

### 6.2 인증 방식

| 방식 | 유효기간 | 자동화 적합성 | 비고 |
|------|----------|:-:|------|
| Auth Key | 최대 90일 | 보통 | Reusable + Pre-approved 옵션 |
| OAuth Client | 만료 없음 | **높음** | API로 auth key 동적 생성 |
| 수동 로그인 | - | X | 브라우저 인증 필요 |

**프로덕션 권장**: OAuth Client로 auth key를 동적 생성하여 90일 만료 문제 해결.

Subnet Router는 소수(1~2대)이므로 수동 auth key로도 운영 가능 (90일마다 갱신).

### 6.3 클라이언트 측 설정

사용자의 Mac 등 tailnet 클라이언트에서 subnet route를 사용하려면:

```bash
# Linux 클라이언트 (기본 비활성화)
tailscale set --accept-routes=true

# macOS, Windows, iOS, Android는 기본 활성화
```

---

## 7. K8s 프로젝트에 미치는 영향

### 7.1 시나리오 A (단일 사이트): 영향 없음

모든 K8s 노드가 자택 LAN에 있으면:

| 항목 | 변경 필요 여부 |
|------|:-:|
| `ansible_host` | 그대로 (10.18.x.x) |
| 템플릿 (`ansible_host` 참조) | **수정 없음** |
| kube-vip | **그대로 사용** (같은 L2) |
| kubelet `--node-ip` | 불필요 |
| Flannel `--iface` | 불필요 |
| UFW | 불필요 |
| 워크플로우 | 기존 그대로 |

**Subnet Router 설정은 K8s Ansible 프로젝트 외부**에서 처리하면 된다 (공유기 설정, 또는 별도 Pi 설정).

### 7.2 시나리오 B (멀티사이트): 변경 필요

자택 + AWS 구성에서는 일부 변경이 필요하다.

#### 변경 대상

| # | 파일 | 변경 내용 | 이유 |
|---|------|-----------|------|
| 1 | `inventory/baremetal` | AWS 노드 추가, `infra_provider` 분기 | 새 사이트 노드 등록 |
| 2 | `roles/kubeadm-init-prep/templates/kubeadm-config.yaml` | certSANs에 AWS 노드 IP 추가 | API 서버 인증서에 원격 노드 포함 |
| 3 | UFW / Security Group | 원격 서브넷 대역 허용 | 사이트 간 방화벽 오픈 |

그러나:

| 항목 | 변경 필요 여부 |
|------|:-:|
| `ansible_host` → `k8s_ip` 분리 | **불필요** |
| 템플릿 8곳 수정 | **불필요** |
| kubelet `--node-ip` 추가 | 상황에 따라 (네트워크 인터페이스 2개 이상인 경우) |
| Flannel `--iface` 추가 | **불필요** (기본 인터페이스 사용) |

#### kube-vip 관련

| 마스터 배치 | kube-vip ARP | 대안 필요 여부 |
|-------------|:-:|:-:|
| 마스터 전부 자택 LAN | **동작** | 불필요 |
| 마스터가 자택 + AWS 분산 | 불가 | 로컬 HAProxy 필요 |

**마스터 노드를 한 사이트(자택)에 집중 배치하면 kube-vip를 그대로 사용할 수 있다.**
AWS에는 워커 노드만 배치하는 방식이 가장 단순하다.

---

## 8. 이전 방안과의 비교

### 8.1 세 가지 방안 종합 비교

| 평가 항목 | 노드마다 Tailscale | 워크플로우 분리 | **Subnet Router** |
|-----------|:-:|:-:|:-:|
| K8s 노드에 Tailscale 설치 | 전체 노드 | 전체 노드 | **Subnet Router만** |
| 노드 IP 개수 | 2개 | 2개 | **1개** |
| `ansible_host` 변경 | `k8s_ip` 분리 | `k8s_ip` 분리 | **변경 없음** |
| 템플릿 수정 | 8곳 | 8곳 | **0곳** (단일 사이트) |
| 치킨-에그 문제 | 있음 | host_vars로 해결 | **없음** |
| kube-vip 사용 가능 | 불가 | 불가 | **가능** (마스터 같은 사이트) |
| kubelet `--node-ip` | 필수 | 필수 | 불필요 |
| Flannel `--iface` | 필수 | 필수 | 불필요 |
| 워크플로우 변경 | 분리 필요 | 분리 필요 | **변경 없음** |
| 추가 인프라 | 없음 | 없음 | **Subnet Router 장치** |
| Ansible 자동화 범위 | Tailscale + K8s | Tailscale + K8s | **K8s만** (SR은 별도) |

### 8.2 핵심 차이

**노드마다 Tailscale** 방식은 K8s의 네트워크 레이어를 변경하는 것이고,
**Subnet Router** 방식은 K8s 아래의 인프라 레이어에서 네트워크를 해결하는 것이다.

```
노드마다 Tailscale:              Subnet Router:

┌──────────────────┐           ┌──────────────────┐
│   K8s 클러스터    │           │   K8s 클러스터    │
│  (Tailscale IP)  │           │   (기존 LAN IP)  │
├──────────────────┤           ├──────────────────┤
│  Tailscale Agent │           │     (변경 없음)   │
├──────────────────┤           ├──────────────────┤
│   OS / Network   │           │   OS / Network   │
└──────────────────┘           ├──────────────────┤
                               │  Subnet Router   │  ← K8s 외부
                               └──────────────────┘
```

Subnet Router 방식은 K8s에 대해 **투명(transparent)**하다. K8s는 VPN의 존재를 모른다.

---

## 9. Subnet Router HA (이중화)

### 9.1 Tailscale의 Failover 지원

동일한 서브넷을 광고하는 **복수의 Subnet Router**를 배포할 수 있다.

```
[자택]                              [AWS]
┌─────────────────┐               ┌─────────────────┐
│ SR-A (Primary)  │               │ SR-C (Primary)  │
│ advertise       │── tailnet ──  │ advertise       │
│ 10.18.0.0/16    │               │ 10.0.1.0/24     │
└─────────────────┘               └─────────────────┘
┌─────────────────┐               ┌─────────────────┐
│ SR-B (Standby)  │               │ SR-D (Standby)  │
│ advertise       │── tailnet ──  │ advertise       │
│ 10.18.0.0/16    │               │ 10.0.1.0/24     │
└─────────────────┘               └─────────────────┘
```

- 두 SR이 동일한 route를 광고하면 Tailscale이 자동으로 primary 선택
- Primary 오프라인 시 자동 failover
- 이 기능은 **Free 플랜 포함 모든 플랜**에서 사용 가능

### 9.2 알려진 제한

- SR이 온라인이지만 실제 라우팅 불가능한 상태에서 failover가 지연될 수 있음
- 어떤 SR이 primary가 될지 명시적으로 지정 불가 (Tailscale이 자동 결정)
- Active-Active 로드 밸런싱은 제한적 (Regional Routing으로 부분 해결)

---

## 10. 성능 및 제약

### 10.1 성능

| 항목 | 수치 |
|------|------|
| Tailscale 기본 MTU | 1280 (보수적 설정) |
| Subnet Router 경유 시 처리량 | 직접 LAN 대비 약 30~40% 수준 |
| 추가 레이턴시 | 사이트 간 네트워크 레이턴시 + WireGuard 처리 |

**K8s에서의 영향:**
- **사이트 내** 노드 간 통신: LAN 직접 통신이므로 **성능 영향 없음**
- **사이트 간** 통신 (자택↔AWS): 물리적 거리에 의한 레이턴시 + WireGuard 암호화 오버헤드 (이 오버헤드는 노드마다 Tailscale을 설치하는 방식에서도 동일하게 발생하며, Subnet Router 고유의 추가 비용은 SR 장치를 한 홉 더 경유하는 정도)
- kubectl 명령, API 접근: 소량 제어 트래픽이므로 영향 미미
- Pod 간 트래픽: CNI(Flannel)가 노드 IP 기반 VXLAN 캡슐화 → 사이트 간이면 SR 경유

### 10.2 프로토콜 제한

| 프로토콜 | Subnet Router 통과 | 비고 |
|----------|:-:|------|
| TCP | O | 모든 K8s 통신 |
| UDP | O | DNS, Flannel VXLAN |
| ARP | **X** (사이트 간) | L2 프로토콜, 터널 통과 불가 |
| Multicast/Broadcast | **X** | L3 터널 특성상 불가 |

ARP는 **같은 사이트 내에서는 정상 동작** (LAN 직접 통신). 사이트 간에만 불가.

### 10.3 기타 제약

| 제약 | 설명 | 영향 |
|------|------|------|
| 서브넷 겹침 불가 | 사이트 간 IP 대역 겹치면 라우팅 충돌 | 현재 구성 (10.18 vs 10.0) 에서는 문제 없음 |
| Auth key 만료 | 90일 (OAuth client로 우회 가능) | SR 재인증 필요 |
| Tailscale SaaS 의존 | 제어 플레인 장애 시 신규 연결 불가 | 기존 터널은 유지됨 |
| SR 단일 장애점 | SR 장애 시 사이트 간 통신 단절 | HA 구성으로 해결 |

---

## 11. Ansible 자동화 범위

### 11.1 K8s 프로젝트의 자동화 범위

Subnet Router 방식에서는 Tailscale 설정이 **K8s 프로젝트 외부**에 위치한다.

```
┌─────────────────────────────────────┐
│    이 프로젝트 (Ansible)            │
│                                     │
│  inventory/baremetal                │
│  playbooks/k8s-all.yaml            │
│  roles/*                           │
│                                     │
│  → K8s 구축에만 집중               │
│  → Tailscale 관련 코드 없음        │
└─────────────────────────────────────┘
         ↑ 의존 (네트워크 연결)
┌─────────────────────────────────────┐
│    Subnet Router 설정 (별도 관리)    │
│                                     │
│  - 공유기: OpenWrt에 Tailscale 설치 │
│  - 또는 전용 Pi 에 Tailscale 설치   │
│  - AWS: Terraform 또는 수동 EC2     │
│  - Tailscale Admin Console ACL      │
│                                     │
│  → 인프라 레벨 (K8s와 독립)        │
└─────────────────────────────────────┘
```

### 11.2 Subnet Router 자동화를 원할 경우

Subnet Router 설정도 자동화하고 싶다면, **별도의 playbook을 추가**할 수 있다:

```yaml
# playbooks/subnet-router.yaml (신규, 선택적)
- hosts: subnet_routers  # 인벤토리에 SR 장치 추가
  become: true
  tasks:
    - name: IP Forwarding 활성화
      sysctl:
        name: "{{ item }}"
        value: "1"
        sysctl_file: /etc/sysctl.d/99-tailscale.conf
      loop:
        - net.ipv4.ip_forward
        - net.ipv6.conf.all.forwarding

    - name: rp_filter loose mode
      sysctl:
        name: "{{ item }}"
        value: "2"
        sysctl_file: /etc/sysctl.d/99-tailscale.conf
      loop:
        - net.ipv4.conf.all.rp_filter
        - net.ipv4.conf.default.rp_filter

    - name: Tailscale 설치
      shell: curl -fsSL https://tailscale.com/install.sh | sh
      args:
        creates: /usr/bin/tailscale

    - name: Tailscale Subnet Router 시작
      command: >
        tailscale up
        --authkey={{ tailscale_authkey }}
        --advertise-routes={{ sr_advertise_routes }}
        --snat-subnet-routes=false
        --advertise-tags=tag:subnet-router
```

이 playbook은 K8s 구축 **전에 한 번만** 실행하면 된다. K8s playbook들과는 완전히 독립적이다.

---

## 12. 구현 로드맵

### 12.1 Phase 1: 단일 사이트 (자택만)

**목표**: 외부에서 자택 K8s 클러스터에 접근

1. 자택 공유기에 Tailscale 설치 + `--advertise-routes=10.18.0.0/16`
2. Tailscale Admin Console에서 route 승인
3. 사용자 Mac에서 `tailscale up --accept-routes`
4. `kubectl --kubeconfig .kube/baremetal.config` 접근 확인

**K8s 프로젝트 변경: 없음**

### 12.2 Phase 2: 멀티사이트 (자택 + AWS)

**목표**: AWS EC2 워커 노드를 자택 K8s 클러스터에 추가

1. AWS에 Subnet Router EC2 배포 (Source/Dest check 비활성화)
2. VPC Route Table에 `10.18.0.0/16 → SR ENI` 추가
3. AWS SR에서 `--advertise-routes=10.0.1.0/24`
4. 인벤토리에 AWS 워커 노드 추가
5. K8s 워커 노드 조인 (`kubernetes-add-workers.yaml` 실행)

**K8s 프로젝트 변경: 인벤토리에 노드 추가만**

---

## 13. 결론

### Subnet Router 방식의 핵심 가치

1. **K8s에 투명** — K8s는 VPN의 존재를 모르며, 기존 LAN IP로 통신
2. **프로젝트 변경 최소화** — 템플릿 수정 0건, 워크플로우 변경 없음
3. **관심사 분리** — 네트워크 인프라(Tailscale)와 애플리케이션 인프라(K8s) 독립
4. **점진적 확장** — 단일 사이트에서 시작, 필요시 멀티사이트로 확장
5. **kube-vip 유지 가능** — 마스터를 같은 사이트에 배치하면 ARP 그대로 사용

### 트레이드오프

| 장점 | 단점 |
|------|------|
| K8s 코드 변경 없음 | Subnet Router 장치(공유기/Pi/EC2) 필요 |
| 닭과 달걀 문제 없음 | 사이트 간 SR이 단일 장애점 (HA로 해결) |
| 기존 환경 호환 100% | Tailscale 자동화가 K8s 프로젝트 밖 |
| 성숙한 기술 (site-to-site VPN) | 사이트 간 성능 오버헤드 (~60%) |

### 권장 사항

**Phase 1부터 시작**하여 자택 공유기에 Tailscale Subnet Router를 설정하고, 외부에서 K8s 클러스터에 접근할 수 있는지 검증하는 것을 권장한다. 이 단계에서는 K8s 프로젝트에 어떤 변경도 필요하지 않다.

멀티사이트 확장이 필요해지면 Phase 2로 진행하되, 마스터 노드는 자택에 유지하여 kube-vip ARP 모드를 그대로 사용할 수 있도록 한다.
