# Tailscale Subnet Router: SNAT 비활성화 시 비대칭 라우팅 문제 분석

## 현상

외부 네트워크(스마트폰 핫스팟)에서 Tailscale Subnet Router를 통해 자택 K8s 클러스터 접근 시:

| 대상 | 접근 | 이유 |
|------|------|------|
| raspberrypi-0 (10.18.1.10) — Primary SR | SSH 성공 | 패킷이 SR 자신에게 도착, 로컬 처리 |
| raspberrypi-1 (10.18.1.11) | SSH 실패 (timeout) | 응답 패킷이 Mac에 도달하지 못함 |
| raspberrypi-2 (10.18.1.12) | SSH 실패 (timeout) | 동일 |
| 워커 노드 (10.18.2.x) | SSH 실패 (timeout) | 동일 |

Tailscale IP 직접 접근은 모든 노드에서 정상:

```
$ nc -z -w 5 100.98.246.31 22   # rpi-1의 Tailscale IP
Connection succeeded!
```

## 환경

- SR 노드: raspberrypi-0, 1, 2 (CP 노드 3대, HA 구성)
- `--snat-subnet-routes=false` 설정
- `--advertise-routes=10.18.0.0/16`
- Mac 클라이언트: `--accept-routes` 활성화, Tailscale IP 100.126.181.100

## 근본 원인: Tailscale Anti-Spoofing

### 패킷 흐름 분석

**정상 경로 (rpi-0 자신에게 접근):**

```
Mac (100.126.181.100)
  → Tailscale 터널
  → rpi-0 (Primary SR, tailscale0)
  → 목적지가 자신 (10.18.1.10) → 로컬 처리
  → 응답: rpi-0 → tailscale0 → Mac ✓
```

**실패 경로 (rpi-1에 접근):**

```
Mac (100.126.181.100)
  → Tailscale 터널
  → rpi-0 (Primary SR, tailscale0)
  → eth1로 포워딩 (src=100.126.181.100, dst=10.18.1.11)
  → rpi-1 eth1에 도착, SSH 연결 수립
  → 응답: src=10.18.1.11, dst=100.126.181.100
  → rpi-1 라우팅: table 52 → 100.126.181.100 via tailscale0
  → rpi-1의 tailscale0 → Tailscale 터널 → Mac
  → Mac의 Tailscale: "10.18.1.11는 rpi-0(Primary SR)에서만 수신 허용" → DROP ✗
```

### 검증 결과

rpi-1에서 Mac으로의 ping 테스트:

```bash
# LAN IP를 소스로 사용 → Mac의 Tailscale이 drop
$ ping -I 10.18.1.11 100.126.181.100
2 packets transmitted, 0 received, 100% packet loss

# Tailscale IP를 소스로 사용 → 정상
$ ping 100.126.181.100
2 packets transmitted, 2 received, 0% packet loss
rtt min/avg/max/mdev = 28.725/39.285/49.845/10.560 ms
```

**결론**: Mac의 Tailscale 클라이언트는 서브넷 대역(10.18.0.0/16) 패킷을 **Primary SR(rpi-0)의 WireGuard 터널에서만 수신을 허용**한다. 다른 노드(rpi-1)가 자신의 tailscale0으로 보내는 서브넷 대역 패킷은 anti-spoofing 정책에 의해 drop된다.

### 왜 이런 설계인가

Tailscale은 각 노드가 전송할 수 있는 IP 범위를 엄격히 제한한다:
- 각 노드: 자신의 Tailscale IP (100.x.x.x)만 소스로 허용
- Subnet Router: 승인된 서브넷 대역도 소스로 허용
- **HA SR 환경**: 동일 서브넷을 광고하는 여러 SR 중 **Primary만** 클라이언트가 수신 허용

이는 네트워크 격리와 보안을 위한 의도된 동작이다.

## 진단 과정에서 확인한 노드 상태

### rpi-0 (Primary SR)

```
ip_forward: 1
rp_filter: 2 (loose)
FORWARD chain: policy DROP, ts-forward 체인으로 jump
ts-forward: tailscale0 → * ACCEPT (932 pkts), * → tailscale0 ACCEPT (0 pkts)
VIP (10.18.1.1): 없음 (kube-vip 리더 아님)
```

### rpi-1 (kube-vip 리더, VIP 보유)

```
ip_forward: 1
rp_filter: 2 (loose)
eth1: 10.18.1.11/24, 10.18.1.1/32 (VIP), 10.18.100.2/32 (Service LB)
FORWARD chain: policy DROP, ts-forward + KUBE-* + FLANNEL-FWD + ufw-* 체인
ts-forward: tailscale0 → * (0 pkts), * → tailscale0 ACCEPT (208 pkts)
table 52: 100.126.181.100 dev tailscale0 ✓
```

### 공유기 정적 라우트 시도 (효과 없음)

```
100.64.0.0/10 → 10.18.1.1 (VIP)
```

이 정적 라우트는 **워커 노드의 응답**을 SR로 보내는 데는 도움이 될 수 있지만, **CP 노드(SR 자체)의 응답**에는 효과가 없다. CP 노드는 자체 table 52에 tailscale0 라우트가 있어 공유기를 거치지 않고 직접 tailscale0으로 응답을 보내기 때문이다.

## 해결 방안

### 방안 1: SNAT 활성화 (권장)

```yaml
# roles/tailscale/tasks/main.yaml
- name: Configure Tailscale as subnet router
  command: >
    tailscale up
    --authkey={{ tailscale_auth_key }}
    --advertise-routes={{ tailscale_advertise_routes }}
    --hostname={{ inventory_hostname }}
  # --snat-subnet-routes=false 제거 (기본값 true 사용)
```

**동작 원리:**

```
Mac → Tailscale → rpi-0 (SR)
  → SNAT: src 100.126.181.100 → 10.18.1.10 (SR의 IP)
  → eth1 → rpi-1 (src=10.18.1.10, dst=10.18.1.11)
  → rpi-1 응답: dst=10.18.1.10 → LAN 직접 전달 (같은 서브넷)
  → rpi-0 수신 → un-SNAT → tailscale0 → Mac ✓
```

| 장점 | 단점 |
|------|------|
| 설정 간단 (기본값) | 대상 노드가 실제 클라이언트 IP를 볼 수 없음 |
| HA failover 시에도 정상 동작 | 향후 멀티사이트 K8s에서 소스 IP 보존 불가 |
| 공유기 정적 라우트 불필요 | |

### 방안 2: SNAT 비활성화 유지 + 비 SR 노드에서 라우팅 조정

CP 노드가 아닌 **워커 노드**에서는 table 52가 없으므로 공유기 정적 라우트로 해결 가능하지만, **CP 노드(SR) 자체**는 table 52가 있어 해결 불가.

CP 노드에 ip rule을 추가하여 100.64.0.0/10 트래픽을 tailscale0 대신 main 테이블로 보내는 방법:

```bash
# SR 노드에서 (자신이 Primary SR이 아닌 경우에만)
ip rule add to 100.64.0.0/10 lookup main priority 5260
```

| 장점 | 단점 |
|------|------|
| 소스 IP 보존 | Primary SR 변경 시 ip rule도 동적으로 변경 필요 |
| 향후 멀티사이트 호환 | 구현 복잡도 높음 |
| | HA failover 자동화 어려움 |

### 방안 3: 4번째 전용 SR 노드 추가

CP 노드에서 Tailscale을 제거하고, 별도 장치(Pi 5 등)에 단일 SR만 설치.

| 장점 | 단점 |
|------|------|
| Anti-spoofing 문제 없음 | SPOF (단일 장치) |
| 구조적으로 깔끔 | 추가 하드웨어 필요 |

## 권장사항

**현 시점: 방안 1 (SNAT 활성화) 권장**

- 현재 유일한 사용 사례는 외부에서 클러스터 원격 접속
- SNAT으로 인한 소스 IP 마스킹은 관리 목적 접근에서 문제 없음
- AWS 멀티사이트 추가 시점에 재검토 (그때는 전용 SR 노드 또는 다른 아키텍처 고려)

## 참고: `--snat-subnet-routes` 동작 정리

| 설정 | 외부→내부 패킷의 src IP | 응답 경로 |
|------|--------------------------|-----------|
| `true` (기본값) | SR의 LAN IP (예: 10.18.1.10) | LAN 직접 → SR → Tailscale → 클라이언트 |
| `false` | 원본 유지 (예: 100.126.181.100) | 대상 노드 → tailscale0 직접 → 클라이언트 (anti-spoofing으로 drop) |
