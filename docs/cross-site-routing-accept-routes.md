# Cross-site 라우팅: `--accept-routes`와 HA 충돌 해결

## 배경

Tailscale Subnet Router를 통해 두 사이트(AWS / 홈랩)를 연결하는 구성에서, 양방향 통신이 되지 않는 문제가 발생했다.

```
AWS VPC (10.0.0.0/16)                    홈랩 (10.18.0.0/16)
┌──────────────────────┐                 ┌──────────────────────┐
│ 10.0.1.100  (SR)     │   Tailscale     │ 10.18.1.10 (SR, Pi-0)│
│ 10.0.10.100 (EC2)    │ ◄═══════════► │ 10.18.1.11 (SR, Pi-1)│
│                      │   WireGuard     │ 10.18.1.12 (SR, Pi-2)│
└──────────────────────┘                 └──────────────────────┘
  advertise: 10.0.0.0/16                   advertise: 10.18.0.0/16
                                           (HA: 3대 모두 동일 서브넷 광고)
```

## 문제 1: 왜 cross-site 패킷이 전달되지 않는가

### 증상

수동으로 `ip route add` 라우트를 추가했음에도 ping이 실패했다.

```bash
# AWS SR에서
$ ip route show
10.18.0.0/16 dev tailscale0 scope link   # 수동 추가된 라우트

$ ping 10.18.1.12
100% packet loss
```

### 원인: WireGuard AllowedIPs

Tailscale은 내부적으로 userspace WireGuard를 사용한다. 각 피어에는 **AllowedIPs**가 설정되어 있으며, tailscale0(TUN 디바이스)에 들어온 패킷은 목적지가 어떤 피어의 AllowedIPs에 매칭될 때만 해당 피어로 전달된다.

```
[--accept-routes=false 일 때]

AWS SR의 WireGuard 설정:
  피어 raspberrypi-2:
    AllowedIPs: 100.121.41.59/32    ← Tailscale IP만 포함
                                     10.18.0.0/16 미포함!

패킷 흐름:
  커널: 10.18.1.12 → ip route 매칭 → tailscale0에 전달   ✓
  WireGuard: dst=10.18.1.12 → AllowedIPs 검색 → 매칭 없음 → DROP  ✗
```

`ip route add`는 커널 레벨에서 패킷을 tailscale0까지 보내는 역할만 한다. 실제 피어 간 전달은 WireGuard의 AllowedIPs가 결정하며, `--accept-routes=false`이면 **피어의 서브넷이 AllowedIPs에 포함되지 않는다.**

### 검증

```bash
# Tailscale LocalAPI로 10.18.1.12가 어느 피어에 매핑되는지 조회
$ tailscale debug localapi v0/whois?addr=10.18.1.12:0
404 Not Found: "no match for IP:port"    ← WireGuard가 이 IP를 모름

# 반면 Tailscale IP는 정상
$ tailscale debug localapi v0/whois?addr=100.121.41.59:0
→ raspberrypi-2 (정상 매칭)
```

### 해결

`--accept-routes`를 활성화하면 피어의 서브넷이 WireGuard AllowedIPs에 포함된다.

```
[--accept-routes=true 일 때]

AWS SR의 WireGuard 설정:
  피어 raspberrypi-2:
    AllowedIPs: 100.121.41.59/32
                10.18.0.0/16         ← 서브넷 포함!

패킷 흐름:
  커널: 10.18.1.12 → tailscale0에 전달   ✓
  WireGuard: dst=10.18.1.12 → 10.18.0.0/16 매칭 → raspberrypi-2로 전달  ✓
```

## 문제 2: 왜 `--accept-routes`가 HA SR의 SSH를 끊는가

### Tailscale의 라우팅 정책 (ip rule)

Tailscale은 자체 라우팅 테이블(table 52)과 정책 룰(ip rule)을 관리한다.

```
$ ip rule list
0:      from all lookup local
5210:   from all fwmark 0x80000/0xff0000 lookup main     ← Tailscale 자체 패킷
5230:   from all fwmark 0x80000/0xff0000 lookup default
5250:   from all fwmark 0x80000/0xff0000 unreachable
5270:   from all lookup 52                                ← ★ main보다 먼저 조회
32766:  from all lookup main                              ← 커널 라우트 테이블
32767:  from all lookup default
```

핵심: **rule 5270(table 52)이 rule 32766(main)보다 우선순위가 높다.**

### `--accept-routes` 활성화 시 변화

`--accept-routes`를 켜면 Tailscale이 table 52에 피어의 서브넷 라우트를 추가한다.

```
[--accept-routes=false]
table 52:
  100.65.227.53 dev tailscale0       ← 피어 Tailscale IP들만
  100.92.251.108 dev tailscale0
  100.98.246.31 dev tailscale0
  ...

[--accept-routes=true]
table 52:
  100.65.227.53 dev tailscale0
  100.92.251.108 dev tailscale0
  100.98.246.31 dev tailscale0
  10.18.0.0/16 dev tailscale0        ← ★ 추가됨 (primary SR의 서브넷)
  10.0.0.0/16 dev tailscale0         ← ★ 추가됨 (AWS SR의 서브넷)
  ...
```

### HA SR에서 문제 발생 메커니즘

홈랩의 3대 Pi는 모두 10.18.0.0/16을 advertise하지만, Primary는 raspberrypi-2 한 대뿐이다.

raspberrypi-0(비주)에서 `--accept-routes`를 켜면:

```
raspberrypi-0의 패킷 라우팅 (예: SSH 응답):

  dst=10.18.x.x (LAN 트래픽)

  ① rule 5270: lookup table 52
     → 10.18.0.0/16 dev tailscale0 매칭!
     → tailscale0으로 전송 (primary SR인 raspberrypi-2 경유)     ← ✗ 잘못됨

  ② rule 32766: lookup main                                      ← 도달 못함
     → 10.18.1.0/24 dev eth1 (로컬 LAN)
```

table 52의 10.18.0.0/16이 main의 10.18.1.0/24보다 **먼저 조회**되기 때문에, 로컬 LAN 트래픽(SSH 응답 포함)이 tailscale0으로 빠진다. 결과적으로 SSH 연결이 끊어진다.

```
[정상 상태: --accept-routes=false]

SSH 클라이언트 → eth1 → raspberrypi-0 → SSH 응답 → eth1 → 클라이언트  ✓
                                                      ↑
                                         main 테이블: 10.18.1.0/24 dev eth1

[비정상: --accept-routes=true, 보호 룰 없음]

SSH 클라이언트 → eth1 → raspberrypi-0 → SSH 응답 → tailscale0 → raspberrypi-2 → ???  ✗
                                                      ↑
                                         table 52: 10.18.0.0/16 dev tailscale0
                                         (main보다 먼저 매칭)
```

### 왜 AWS SR에서는 문제가 없는가

AWS SR은 10.0.0.0/16을 advertise하며, **같은 서브넷을 advertise하는 다른 SR이 없다.** 따라서 `--accept-routes`를 켜도 table 52에 10.0.0.0/16이 추가되지 않는다 (자기 자신이 유일한 SR이므로).

## 해결: 보호 ip rule

table 52보다 높은 우선순위로, 자기 서브넷 트래픽은 main 테이블을 먼저 조회하도록 ip rule을 추가한다.

```
$ ip rule add to 10.18.0.0/16 lookup main priority 5269

결과:
5269:   from all to 10.18.0.0/16 lookup main    ← ★ 보호 룰 (신규)
5270:   from all lookup 52
32766:  from all lookup main
```

### 보호 룰 적용 후 패킷 흐름

```
[로컬 LAN 트래픽: dst=10.18.1.11]

  ① rule 5269: to 10.18.0.0/16 → lookup main
     → main 테이블: 10.18.1.0/24 dev eth1 매칭
     → eth1로 전송  ✓ (정상)

[Cross-site 트래픽: dst=10.0.1.100]

  ① rule 5269: to 10.18.0.0/16 → 매칭 안 됨 (dst가 10.0.x.x)
  ② rule 5270: lookup table 52
     → 10.0.0.0/16 dev tailscale0 매칭
     → tailscale0으로 전송  ✓ (정상)
```

로컬 트래픽과 cross-site 트래픽이 올바르게 분리된다.

### Ansible 역할 반영

`vpn/roles/tailscale/tasks/main.yaml`에 다음이 추가되었다:

```yaml
# --accept-routes 활성화 전에 보호 룰 먼저 추가
- name: Add protective ip rule to prevent accept-routes from overriding local subnet
  command: ip rule add to {{ tailscale_advertise_routes }} lookup main priority 5269

# 재부팅 시에도 보호 룰 유지
- name: Persist protective ip rule across reboots
  copy:
    dest: /etc/networkd-dispatcher/routable.d/49-protect-local-route.sh
    mode: '0755'
    content: |
      #!/bin/bash
      ip rule add to {{ tailscale_advertise_routes }} lookup main priority 5269 2>/dev/null || true

# --accept-routes 포함된 Tailscale 설정
- name: Apply Tailscale subnet router configuration
  command: >
    tailscale set
    --advertise-routes={{ tailscale_advertise_routes }}
    --accept-routes
    --snat-subnet-routes=true
    --hostname={{ inventory_hostname }}
```

순서가 중요하다: **보호 룰 → accept-routes**. 역순이면 SSH가 끊어진다.

## 전체 패킷 흐름 요약

### 홈랩 → AWS (Pi-0에서 10.0.1.100으로 ping)

```
Pi-0 (10.18.1.10)
  │
  │ dst=10.0.1.100
  │ rule 5269: to 10.18.0.0/16? → No (dst는 10.0.x.x)
  │ rule 5270: lookup 52 → 10.0.0.0/16 dev tailscale0
  │
  ▼
tailscale0 (WireGuard)
  │ AllowedIPs: AWS SR = [100.92.251.108/32, 10.0.0.0/16]
  │ dst=10.0.1.100 → 10.0.0.0/16 매칭 → AWS SR로 전송
  │
  ▼
AWS SR (10.0.1.100)
  │ SNAT: src를 10.0.1.100으로 변환 (--snat-subnet-routes=true)
  │ dst=10.0.1.100은 자기 자신 → 로컬 처리
  │
  ▼ ICMP reply
  │
tailscale0 → WireGuard → Pi-0  ✓
```

### AWS → 홈랩 (10.0.1.100에서 Pi-0으로 ping)

```
AWS SR (10.0.1.100)
  │
  │ dst=10.18.1.10
  │ ip route: 10.18.0.0/16 dev tailscale0
  │
  ▼
tailscale0 (WireGuard)
  │ AllowedIPs: Pi-2(Primary) = [100.121.41.59/32, 10.18.0.0/16]
  │ dst=10.18.1.10 → 10.18.0.0/16 매칭 → Pi-2로 전송
  │
  ▼
Pi-2 (10.18.1.12, Primary SR)
  │ SNAT: src를 10.18.1.12로 변환
  │ dst=10.18.1.10 → eth1로 포워딩 (같은 서브넷)
  │
  ▼
Pi-0 (10.18.1.10)
  │ ICMP reply: dst=10.18.1.12
  │ rule 5269: to 10.18.0.0/16 → lookup main
  │ main: 10.18.1.0/24 dev eth1 → eth1로 직접 전송  ✓
  │
  ▼
Pi-2 → un-SNAT → tailscale0 → AWS SR  ✓
```

## 핵심 교훈

| 개념 | 설명 |
|------|------|
| `ip route add ... dev tailscale0` | 커널이 패킷을 TUN 디바이스까지 보내는 역할만 함 |
| WireGuard AllowedIPs | TUN에 도착한 패킷을 실제로 어느 피어에게 보낼지 결정 |
| `--accept-routes` | 피어의 서브넷을 AllowedIPs에 포함시킴 + table 52에 시스템 라우트 설치 |
| table 52 vs main | Tailscale의 ip rule에 의해 table 52가 main보다 먼저 조회됨 |
| 보호 ip rule | 자기 서브넷 트래픽만 main 테이블로 우회시켜 로컬 LAN 보호 |
