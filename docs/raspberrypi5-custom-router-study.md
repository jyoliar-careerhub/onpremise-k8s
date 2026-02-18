# Raspberry Pi 5 커스텀 공유기 검토

## 개요

Raspberry Pi 5를 커스텀 공유기로 개조하여 자택 네트워크의 라우터 + Tailscale Subnet Router 역할을 겸하도록 하는 방안에 대한 검토.

## 1. 하드웨어 요구사항

### 문제: 이더넷 포트가 1개뿐

Pi 5에는 기본 Gigabit Ethernet 포트가 1개만 있음. 공유기 역할을 하려면 최소 2개(WAN + LAN) 필요.

### 해결 방안

| 방식 | 제품 예시 | 가격 | 속도 | 안정성 |
|------|-----------|------|------|--------|
| **PCIe HAT (권장)** | Pineboards HatNET! 2.5G | ~$25 | 2.5GbE | 높음 (USB 경유하지 않음) |
| PCIe HAT (듀얼) | ZDE ZP595 | ~$40 | 듀얼 2.5GbE | 높음 |
| PCIe HAT (NVMe 겸용) | Pineboards HatDrive! NET 1G | ~$35 | 1GbE + NVMe | 높음 |
| USB 3.0 어댑터 | RTL8156B 기반 | ~$15-25 | 2.5GbE | 보통 (USB 버스 리셋 위험) |

**권장**: PCIe HAT 방식. Pi 5의 PCIe Gen 2 대역폭(500 MB/s)은 Gigabit 라우팅(125 MB/s)에 충분하며, USB 스택을 우회하므로 안정성이 높음.

### 추가 고려사항

- **전원**: 공식 27W USB-C 전원 어댑터 권장 (Pi 5 + HAT + NVMe 시 15-20W 소모)
- **냉각**: 24/7 구동 시 능동 냉각 필수. 공식 Active Cooler 또는 Argon NEO 5 케이스
- **스토리지**: SD 카드는 24/7 라우터 운용에 부적합 (쓰기 수명 1-2년). NVMe SSD 부팅 권장

## 2. 소프트웨어 옵션

### A. OpenWrt

- **Pi 5 지원**: SNAPSHOT 빌드만 제공 (안정 릴리스 없음)
- 전용 라우터 OS, LuCI 웹 UI 포함
- **단점**: SNAPSHOT이라 업데이트 시 깨질 가능성, Pi 5 커뮤니티 소규모

### B. 일반 Linux (Debian/Ubuntu) + nftables (권장)

- Pi 5에서 안정적으로 동작 (공식 Raspberry Pi OS, Ubuntu 24.04 LTS)
- nftables로 NAT/방화벽, dnsmasq로 DHCP/DNS 구성
- Tailscale 공식 ARM64 패키지 바로 설치 가능
- **단점**: 웹 UI 없음 (Cockpit 등으로 보완 가능), OpenWrt보다 설정 수동

### C. 기타

- **OPNsense/pfSense**: ARM 미지원 → **사용 불가**
- **VyOS**: Pi 같은 단일 NIC 보드 미지원 계획
- **Alpine Linux**: 경량 대안이나 커뮤니티 제한적

**권장**: 일반 Linux. Tailscale 통합이 가장 쉽고, Pi 5에서 검증됨.

## 3. Tailscale Subnet Router 호환성

### Linux에서의 구성

```bash
# Tailscale 설치
curl -fsSL https://tailscale.com/install.sh | sh

# Subnet Router로 실행
tailscale up --advertise-routes=10.18.0.0/16 --snat-subnet-routes=false
```

### 필수 커널 파라미터

```
net.ipv4.ip_forward = 1              # 라우터 NAT + Tailscale 모두 필요
net.ipv6.conf.all.forwarding = 1     # IPv6 포워딩
net.ipv4.conf.all.rp_filter = 2      # loose 모드 (Tailscale 필수)
net.ipv4.conf.default.rp_filter = 2
```

- `ip_forward`는 라우터 기능과 Tailscale SR이 공통으로 필요 → 충돌 없음
- `rp_filter=2` (loose)를 설정하지 않으면 Tailscale SR 패킷이 조용히 드롭됨

### 알려진 이슈

- 다른 VPN(OpenVPN, WireGuard 클라이언트)과 동시 사용 시 라우팅 테이블 충돌 가능
- 특정 커널 버전에서 SR 문제 보고 → Ubuntu HWE 커널로 해결된 사례

**결론**: Pi 5 Linux에서 라우터 + Tailscale SR 겸용은 **기술적으로 문제 없음**.

## 4. 성능

| 항목 | Pi 5 | 일반 소비자 공유기 | N100 Mini PC |
|------|------|-------------------|-------------|
| CPU | Cortex-A76 4코어 2.4GHz | MIPS/Cortex-A7~A53 | x86 4코어 3.4GHz |
| NAT 처리 | 소프트웨어 (~940Mbps) | 하드웨어 ASIC (line rate) | 소프트웨어 (~2.5Gbps) |
| WireGuard 처리량 | ~700-900Mbps | ~100-300Mbps | ~Gigabit (40% CPU) |
| 소비 전력 | 5-12W | 5-15W | 15-25W |

- Pi 5는 소프트웨어 라우팅이지만 Gigabit 수준에서는 충분
- WireGuard/Tailscale 성능은 ARM crypto 확장 덕분에 소비자 공유기보다 우수
- 하드웨어 NAT 오프로드 부재가 멀티기가비트에서만 문제

## 5. 위험 요소

| 위험 | 영향 | 완화 방안 |
|------|------|-----------|
| Pi 5 고장 시 인터넷 끊김 | 높음 | 기존 공유기를 백업으로 보관 |
| USB 이더넷 불안정 | 중간 | PCIe HAT 사용으로 회피 |
| SD 카드 마모 | 높음 | NVMe SSD 부팅 사용 |
| 하드웨어 NAT 미지원 | 낮음 | Gigabit 수준에서는 문제 없음 |
| 과열로 인한 쓰로틀링 | 중간 | 능동 냉각 장착 |

**가장 큰 위험**: Pi 5 = 단일 장애점(SPOF). 고장 시 전체 네트워크 다운.

## 6. 대안 비교

| 방안 | 비용 | 복잡도 | 안정성 | 특징 |
|------|------|--------|--------|------|
| **Pi 5 풀 라우터** | ~$100-130 | 높음 | 보통 | 이더넷 HAT + NVMe + 냉각 필요 |
| **N100 Mini PC 라우터** | ~$120-180 | 중간 | 높음 | 듀얼 2.5GbE 내장, OPNsense 가능 |
| **GL.iNet Brume 2** | ~$65 | 매우 낮음 | 높음 | Tailscale 내장, VPN 게이트웨이 전용 |
| **기존 공유기 유지 + Pi 5는 SR만** | ~$60-80 | **매우 낮음** | **가장 높음** | 이더넷 1개로 충분, SPOF 없음 |

### 기존 공유기 + Pi 5 SR 전용 (최소 위험 방안)

```
인터넷 ─── 기존 공유기 (NAT/DHCP/DNS/방화벽) ─── LAN
                                                    │
                                    Pi 5 (Tailscale Subnet Router만)
                                    └ 단일 이더넷으로 충분
                                    └ 고장 시 인터넷은 유지, 원격 접속만 불가
```

- 추가 하드웨어(HAT, NVMe) 불필요
- 공유기 설정 변경 불필요
- Pi 5 고장 시에도 자택 인터넷은 정상 동작

## 7. 결론

Pi 5를 풀 라우터로 개조하는 것은 **기술적으로 가능**하지만:

- PCIe HAT + NVMe + 냉각 등 추가 하드웨어 비용 발생
- 소프트웨어 설정(NAT, DHCP, DNS, 방화벽) 복잡
- **단일 장애점** 문제 (고장 시 전체 네트워크 다운)

Tailscale Subnet Router만 필요하다면, **기존 공유기를 유지하고 Pi 5는 SR 전용으로 운용**하는 것이 가장 안전하고 간단함. 이 경우 이더넷 포트 1개로 충분하며, 추가 하드웨어가 불필요.

풀 라우터가 반드시 필요하다면 N100 Mini PC(듀얼 2.5GbE 내장)가 Pi 5보다 적합한 플랫폼.

## 부록: 라우터 전용 미니 PC

### 일반 미니 PC vs 라우터 전용 미니 PC

풀 라우터를 구축할 경우, 일반 미니 PC가 아닌 **라우터/방화벽 전용 미니 PC**(소프트 라우터, 방화벽 어플라이언스)라는 별도 카테고리가 존재한다.

| 항목 | 일반 미니 PC | 라우터 전용 미니 PC |
|------|-------------|-------------------|
| NIC | 1개 (HDMI, USB 등 풍부) | **2~6개** 2.5GbE (영상 출력 최소화) |
| 냉각 | 팬 있음 | **팬리스** (24/7 무소음 운용) |
| 크기 | 소형 데스크톱 | 더 작고 납작함 |
| 소비전력 | 15~25W | **6~15W** |
| 용도 | 범용 | OPNsense, pfSense, OpenWrt 등 |

### 권장 사양

- **CPU**: Intel N100 (4코어, 3.4GHz burst)
- **NIC**: 듀얼 이상 2.5GbE, **Intel i226-V** 칩셋 권장 (Realtek 대비 OPNsense/pfSense 호환성 우수)
- **RAM**: 8GB 이상
- **스토리지**: NVMe SSD 슬롯
- **냉각**: 팬리스

### 검색 키워드

`N100 firewall appliance fanless 2.5GbE` 또는 `N100 soft router`

### 대표 제조사

Topton, CWWK, Minisforum, Beelink 등이 이 카테고리 제품을 생산하며, Amazon이나 AliExpress에서 검색 가능.
