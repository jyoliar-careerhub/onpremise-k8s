# 소프트 라우터 제품 조사

## 개요

풀 라우터 구축 시 Pi 5보다 적합한 플랫폼으로 권장되는 x86 기반 소프트 라우터(방화벽 어플라이언스) 제품 조사 결과.

관련 문서: [Raspberry Pi 5 커스텀 공유기 검토](raspberrypi5-custom-router-study.md)

## CPU 선택: N100 vs J5005

소프트 라우터 시장에서 주로 사용되는 두 CPU의 비교.

| 항목 | J5005 (Pentium Silver) | N100 (Alder Lake-N) |
|------|----------------------|---------------------|
| 아키텍처 | Gemini Lake (2017) | Alder Lake-N (2023) |
| 공정 | 14nm | 10nm (Intel 7) |
| 코어/스레드 | 4C/4T | 4C/4T |
| 최대 클럭 | 2.8GHz | 3.4GHz |
| TDP | 10W | 6W |
| RAM | DDR4 2400MHz | DDR5 4800MHz |
| 종합 성능 | 기준 | ~70% 더 빠름 |
| NAT/라우팅 | 2.5Gbps line rate 가능 | 2.5Gbps line rate 가능 |
| WireGuard | ~500-900Mbps | ~1Gbps+ |
| 가격대 | **저렴** (구형 재고) | 상대적으로 높음 |

**네트워크 속도는 NIC(i226-V 2.5GbE)가 결정**하므로, NAT/라우팅 처리량은 둘 다 동일하게 2.5Gbps line rate를 달성할 수 있다. CPU 성능 차이가 체감되는 것은 VPN 암호화 처리량과 Proxmox 가상화 같은 추가 부하에서뿐이다.

- **라우터 전용**: J5005로 충분. 가성비 우수
- **Proxmox 가상화 겸용**: N100 권장. 여유 성능 + 낮은 전력 소모

## 공통 스펙

대부분의 제품이 공유하는 기본 사양.

| 항목 | 내용 |
|------|------|
| NIC | Intel i226-V 2.5GbE |
| 보안 | AES-NI 하드웨어 가속 |
| 냉각 | 팬리스 (알루미늄 방열) |
| 지원 OS | OPNsense, pfSense, OpenWrt, Proxmox, Ubuntu/Debian 등 |

## 제품별 상세

### J5005 기반 (저가형)

#### 1. HUNSN RS34 (4포트, J5005)

| 항목 | 내용 |
|------|------|
| CPU | Intel Pentium Silver J5005 (4C/4T, 최대 2.8GHz, TDP 10W) |
| LAN | 4x i226-V 2.5GbE |
| RAM | DDR4 SO-DIMM x2 슬롯, 최대 16GB |
| 스토리지 | M.2 NVMe + 2.5" SATA |
| 디스플레이 | VGA + HDMI |
| 가격 | 베어본 ~$100-120 |
| 판매처 | [Amazon](https://www.amazon.com/Firewall-Appliance-Celeron-HUNSN-Barebone/dp/B09PHHXN9V), [HUNSN 공식](https://www.hunsn.com/product/pfsense-mini-pc/) |

**특징**: HUNSN은 소프트 라우터 전문 제조사로 커뮤니티 인지도가 높음. DDR4 SO-DIMM 2슬롯으로 RAM 확장 유연. pfSense/OPNsense 커뮤니티에서 검증된 모델.

#### 2. HAICOTECH X2B (4포트, J5005)

| 항목 | 내용 |
|------|------|
| CPU | Intel Pentium Silver J5005 (4C/4T, 최대 2.8GHz, TDP 10W) |
| LAN | 4x i226-V 2.5GbE |
| RAM | DDR4 SO-DIMM |
| 스토리지 | M.2 NVMe |
| 디스플레이 | HDMI 2.0 + DP |
| 가격 | 베어본 ~$90-110 |
| 판매처 | [Amazon](https://www.amazon.com/HAICOTECH-Firewall-Appliance-Computer-P-F-Sense/dp/B0G5Z649RB) |

**특징**: J5005 베어본 중 가장 저렴한 편. 단순 라우터/방화벽 전용으로 적합.

#### 3. UDPTCP (4포트, J5005, Wi-Fi 포함)

| 항목 | 내용 |
|------|------|
| CPU | Intel Pentium Silver J5005 (4C/4T, 최대 2.8GHz, TDP 10W) |
| LAN | 4x i226-V 2.5GbE |
| RAM | DDR4 SO-DIMM |
| 스토리지 | M.2 NVMe |
| 추가 기능 | **듀얼밴드 Wi-Fi 6**, SIM 카드 슬롯 |
| 가격 | 8GB+128GB SSD ~$130-150 |
| 판매처 | [Amazon (8GB+128GB)](https://www.amazon.com/Firewall-Appliance-Computer-Hardware-OPNsense/dp/B0FP4SV282), [Amazon (16GB+512GB)](https://www.amazon.com/UDPTCP-Fanless-Computer-Industry-Business/dp/B0FWXGD3FH) |

**특징**: Wi-Fi 6 내장으로 별도 AP 없이 무선 라우터로 사용 가능. SIM 슬롯으로 LTE 백업 회선도 가능. J5005 제품 중 가장 기능이 풍부.

### N100 기반 (고성능)

#### 4. CWWK X86 P5 (2포트, N100, 초소형)

| 항목 | 내용 |
|------|------|
| CPU | Intel N100 (4C/4T, 최대 3.4GHz, TDP 6W) |
| LAN | 2x i226-V 2.5GbE |
| RAM | DDR5 4800MHz SO-DIMM, 최대 32GB |
| 스토리지 | M.2 NVMe |
| 크기 | 90mm x 90mm (초소형) |
| 무게 | 750g |
| 소비전력 | ~6W |
| 가격 | 베어본 ~$149, 8GB+128GB SSD ~$200-220 |
| 판매처 | [CWWK 공식 스토어](https://cwwk.net/products/cwwk-x86-p5-super-mini-router-12th-gen-intel-n100-ddr5-4800mhz-firewall-pc-2x-i226-v-2-5g-lan-fanless-mini-pc) |

**특징**: 가장 작은 폼팩터. WAN+LAN 2포트면 충분한 단순 라우터/SR 구성에 최적.

### 5. CWWK F3 (4포트, N100)

| 항목 | 내용 |
|------|------|
| LAN | 4x i226-V 2.5GbE |
| RAM | DDR5 SO-DIMM, 최대 32GB |
| 스토리지 | M.2 NVMe |
| 가격 | 베어본 ~$180-220, 8GB+128GB ~$250 |
| 판매처 | [CWWK 공식 스토어](https://cwwkpc.com/products/cwwk-f5-mini-pc-n150-n355-upgraded-n100-n305-firewall-appliance-opnsense-mini-computer-with-6-port-i226-v-2-5gbe-lan-fanless-micro-pc-ddr5-usb-type-c-3-display) |

**특징**: OPNsense/pfSense 메인스트림 구성. 4포트로 WAN/LAN/DMZ/관리 분리 가능.

### 6. Topton 4-LAN (4포트, N100)

| 항목 | 내용 |
|------|------|
| LAN | 4x i226-V 2.5GbE |
| RAM | DDR5 4800MHz SO-DIMM, 최대 32GB |
| 스토리지 | M.2 NVMe + 2.5" SATA |
| 추가 I/O | HDMI, DP, RJ45 COM (시리얼 콘솔), SIM 슬롯 |
| 가격 | ~$150-200 (AliExpress) |
| 판매처 | [Topton 공식](https://www.toptonpc.com/product/topton-4x2-5g-lan-firewall-mini-pc-intel-n100-top-version-i226-v-fanless-router-pc-proxmox-vpn-server-pfsense-opnsense-aes-ni/) |

**특징**: 올 알루미늄 방열판 설계로 일반 제품 대비 온도 10~15도 낮다고 주장. AliExpress에서 가성비 좋은 가격. 시리얼 콘솔 포트 포함.

### 7. MOGINSOK (4포트, N100, 울트라 컴팩트)

| 항목 | 내용 |
|------|------|
| LAN | 4x i226-V 2.5GbE |
| RAM | LPDDR5 온보드, 8GB 또는 16GB (**교체 불가**) |
| 스토리지 | M.2 NVMe PCIe 3.0 |
| 소비전력 | TDP 6W |
| 가격 | ~$180-220 |
| 판매처 | [MOGINSOK 공식](https://www.moginsok.com/products/intel-n100), [Amazon](https://www.amazon.com/MOGINSOK-Firewall-Appliance-Computer-OPNsense/dp/B0CY8FLFVR) |

**특징**: LPDDR5 온보드라 RAM 교체 불가하지만 울트라 컴팩트 폼팩터. 구매 시 RAM 용량을 확정해야 함.

### 8. SJRC 6-LAN (6포트, N100)

| 항목 | 내용 |
|------|------|
| LAN | **6x** i226-V 2.5GbE |
| RAM | DDR5 SO-DIMM, 최대 32GB |
| 스토리지 | 2x M.2 NVMe + 1x 2.5" SATA |
| 디스플레이 | 2x HDMI 2.1 + 1x DP 1.4 (4K 트리플) |
| 가격 | 8GB+256GB SSD 구성 ~$250-300 |
| 판매처 | [Amazon](https://www.amazon.com/SJRC-Computer-Firewall-Appliance-6Lan-N100/dp/B0CXM3148B) |

**특징**: 6포트로 VLAN 세분화, DMZ, 별도 관리 네트워크 구성에 유리. 스토리지 슬롯 3개로 Proxmox 가상화 겸용도 가능.

### 9. goodtico (4포트, N100, DDR4 저가형)

| 항목 | 내용 |
|------|------|
| LAN | 4x i226-V 2.5GbE |
| RAM | DDR**4** 8GB (**DDR5가 아님에 주의**) |
| 스토리지 | 256GB SSD |
| 가격 | ~$150-180 |
| 판매처 | [Amazon](https://www.amazon.com/Fanless-Firewall-Appliance-Industrial-Computer/dp/B0DGG4GMQH) |

**특징**: DDR4 모델이라 가격이 저렴. 라우터 용도에서 DDR4와 DDR5의 성능 차이는 무시할 수준.

## 요약 비교

### J5005 기반 (저가형)

| # | 제품 | 포트 | CPU | RAM 타입 | 예상 가격 (8GB 기준) | 특징 |
|:-:|------|:----:|-----|----------|:-------------------:|------|
| 1 | HUNSN RS34 | 4 | J5005 | DDR4 x2 | 베어본 ~$100-120 | 커뮤니티 검증, RAM 슬롯 2개 |
| 2 | HAICOTECH X2B | 4 | J5005 | DDR4 | 베어본 ~$90-110 | 최저가 베어본 |
| 3 | UDPTCP | 4 | J5005 | DDR4 | ~$130-150 | **Wi-Fi 6 + SIM 슬롯** 내장 |

### N100 기반 (고성능)

| # | 제품 | 포트 | CPU | RAM 타입 | 예상 가격 (8GB 기준) | 특징 |
|:-:|------|:----:|-----|----------|:-------------------:|------|
| 4 | CWWK X86 P5 | 2 | N100 | DDR5 | ~$200 | 90x90mm 초소형 |
| 5 | CWWK F3 | 4 | N100 | DDR5 | ~$250 | 표준 방화벽/라우터 |
| 6 | Topton 4-LAN | 4 | N100 | DDR5 | ~$150-200 | 가성비, 시리얼 콘솔 |
| 7 | MOGINSOK | 4 | N100 | LPDDR5 (온보드) | ~$180-220 | 울트라 컴팩트, RAM 교체 불가 |
| 8 | SJRC 6-LAN | 6 | N100 | DDR5 | ~$250-300 | 6포트, 스토리지 3슬롯 |
| 9 | goodtico | 4 | N100 | DDR4 | ~$150-180 | N100이지만 DDR4로 저가 |

가격은 판매처(Amazon, AliExpress, 공식 스토어)와 시점에 따라 변동된다.

## 선택 가이드

- **최저가 라우터**: HAICOTECH X2B (J5005 베어본) — ~$90-110
- **Wi-Fi AP 겸용**: UDPTCP (J5005) — Wi-Fi 6 + SIM 슬롯 내장
- **단순 라우터 + Tailscale SR**: HUNSN RS34 (J5005) 또는 CWWK X86 P5 (N100 2포트)
- **OPNsense/pfSense 표준 구성**: Topton 4-LAN (N100) 또는 CWWK F3 (N100) — 4포트, 가성비 우수
- **가상화(Proxmox) 겸용**: SJRC 6-LAN (N100) — 6포트 + 스토리지 슬롯 3개 + CPU 여유
- **N100 최저가**: goodtico (DDR4) — 라우터 용도에서 DDR4/DDR5 차이 미미

## 구매 링크 모음

### J5005 제품 (Amazon)

| 제품 | 구성 | 링크 |
|------|------|------|
| HUNSN RS34 | 베어본 | https://www.amazon.com/Firewall-Appliance-Celeron-HUNSN-Barebone/dp/B09PHHXN9V |
| HAICOTECH X2B | 베어본 | https://www.amazon.com/HAICOTECH-Firewall-Appliance-Computer-P-F-Sense/dp/B0G5Z649RB |
| UDPTCP (Wi-Fi 6) | 8GB+128GB SSD | https://www.amazon.com/Firewall-Appliance-Computer-Hardware-OPNsense/dp/B0FP4SV282 |
| UDPTCP (Wi-Fi 6) | 16GB+512GB SSD | https://www.amazon.com/UDPTCP-Fanless-Computer-Industry-Business/dp/B0FWXGD3FH |

### N100 제품 (Amazon)

| 제품 | 구성 | 링크 |
|------|------|------|
| N100 4포트 | 8GB+128GB SSD | https://www.amazon.com/Firewall-Appliance-Fanless-Network-Computer/dp/B0CNDBLKPY |
| CWWK 4포트 | 베어본 | https://www.amazon.com/CWWK-PC-N100-Fanless-Barebone/dp/B0C1TVXQ93 |
| goodtico 4포트 | 8GB+256GB SSD | https://www.amazon.com/Fanless-Firewall-Appliance-Industrial-Computer/dp/B0DGG4GMQH |
| SJRC 6포트 | 8GB+256GB SSD | https://www.amazon.com/SJRC-Computer-Firewall-Appliance-6Lan-N100/dp/B0CXM3148B |

### 제조사 공식 스토어

| 제조사 | 링크 |
|--------|------|
| HUNSN (J5005/N100 라인업) | https://www.hunsn.com/product/pfsense-mini-pc/ |
| CWWK (X86 P5, 2포트, 베어본 $149~) | https://cwwk.net/products/cwwk-x86-p5-super-mini-router-12th-gen-intel-n100-ddr5-4800mhz-firewall-pc-2x-i226-v-2-5g-lan-fanless-mini-pc |
| CWWK 전체 라인업 | https://cwwkpc.com/collections/mini-pc |
| MOGINSOK (N100) | https://www.moginsok.com/products/intel-n100 |
| Topton (N100 4포트) | https://www.toptonpc.com/product/topton-4x2-5g-lan-firewall-mini-pc-intel-n100-top-version-i226-v-fanless-router-pc-proxmox-vpn-server-pfsense-opnsense-aes-ni/ |
| Topton (J5005 4포트) | https://www.toptonpc.com/product/firewall-appliance-upgraded-version-intel-n150-j5005-n6210-fanless-mini-pc-4xi226-v-2-5g-solid-mini-computer-pfsense-opnsense/ |

### 검색 키워드

위 링크가 품절이거나 가격이 변동된 경우, 아래 키워드로 검색:

- J5005: `J5005 firewall appliance 4 port i226-V fanless`
- N100: `N100 firewall appliance 4 port i226-V 8GB`
- AliExpress: `N100 soft router 4 i226 fanless` 또는 `J5005 soft router 4 i226`

## 부록: Proxmox 가상화란

### Proxmox VE란

Proxmox VE(Virtual Environment)는 Debian Linux 기반의 **오픈소스 Type-1 하이퍼바이저**다. 물리 서버 한 대에 여러 가상 머신(VM)과 컨테이너(LXC)를 동시에 실행할 수 있게 해주는 가상화 플랫폼이다.

```
┌─────────────────────────────────────────┐
│         Proxmox VE 웹 관리 UI           │
├─────────────────────────────────────────┤
│  VM 1          │  VM 2       │  LXC 1   │
│  OPNsense      │  Ubuntu     │  Pi-hole  │
│  (방화벽)       │  (서버)     │  (DNS)    │
├─────────────────────────────────────────┤
│  KVM 하이퍼바이저 + LXC 컨테이너 런타임  │
├─────────────────────────────────────────┤
│  Debian Linux (호스트 OS)               │
├─────────────────────────────────────────┤
│  하드웨어 (N100 미니 PC)                │
│  NIC: i226-V x4 → VM에 개별 할당 가능   │
└─────────────────────────────────────────┘
```

### 왜 소프트 라우터에서 Proxmox를 쓰는가

소프트 라우터에 OPNsense를 **직접 설치**(bare metal)하면 그 장비는 라우터 **하나의 역할만** 수행한다. Proxmox를 설치하면 하나의 장비에서 **여러 서비스를 동시에** 실행할 수 있다.

```
bare metal 설치:                Proxmox 가상화:
┌──────────────┐               ┌──────────────────────────┐
│  OPNsense    │               │  OPNsense VM (방화벽)    │
│  (이것만 실행)│               │  Pi-hole LXC (광고 차단)  │
│              │               │  Tailscale LXC (VPN)     │
│              │               │  Home Assistant VM (IoT) │
└──────────────┘               └──────────────────────────┘
  장비 1대 = 역할 1개             장비 1대 = 역할 여러 개
```

### 소프트 라우터에서의 Proxmox 활용 예시

| VM/LXC | 역할 | 할당 리소스 |
|--------|------|------------|
| OPNsense VM | 방화벽, NAT, DHCP | 2 vCPU, 2GB RAM, NIC 2개 (WAN+LAN) |
| Pi-hole LXC | DNS 광고 차단 | 1 vCPU, 256MB RAM |
| Tailscale LXC | Subnet Router | 1 vCPU, 256MB RAM |
| Home Assistant VM | IoT 관리 | 1 vCPU, 1GB RAM |

**NIC 패스스루(PCI Passthrough)**: i226-V NIC 포트를 개별 VM에 직접 할당할 수 있다. 예를 들어 포트 1(WAN)과 포트 2(LAN)를 OPNsense VM에 할당하고, 포트 3을 별도 VM에 할당하는 식이다.

### Proxmox의 주요 장점

| 장점 | 설명 |
|------|------|
| **무료** | 오픈소스, 기능 제한 없음, 라이선스 비용 없음 |
| **스냅샷/백업** | OPNsense 설정 변경 전에 스냅샷 → 문제 시 즉시 롤백 |
| **웹 UI** | 브라우저에서 모든 VM/LXC 관리, 콘솔 접속 가능 |
| **KVM + LXC** | 무거운 서비스는 VM, 가벼운 서비스는 LXC 컨테이너로 분리 |
| **라이브 마이그레이션** | 클러스터 구성 시 VM을 다른 노드로 무중단 이동 |

### Proxmox를 쓰면 안 되는 경우

| 상황 | 이유 |
|------|------|
| 라우터 **전용**으로만 사용 | 가상화 오버헤드만 추가됨. OPNsense bare metal이 단순 |
| RAM 8GB 미만 | Proxmox 호스트 + OPNsense VM + 기타 서비스에 최소 8GB 필요 |
| J5005 CPU | 가상화 오버헤드로 라우팅 성능 저하 가능. N100 이상 권장 |
| 최대한 안정적이어야 하는 경우 | 가상화 레이어가 추가되면 장애 포인트 증가 |

### J5005 vs N100: Proxmox 적합성

| 항목 | J5005 | N100 |
|------|-------|------|
| OPNsense bare metal | 충분 | 충분 |
| Proxmox + OPNsense VM 1개 | 가능하지만 여유 없음 | **여유 있음** |
| Proxmox + VM/LXC 다수 | **비권장** | 가능 (3~4개) |

**결론**: Proxmox 가상화를 계획한다면 N100, 라우터 전용이라면 J5005가 가성비 면에서 합리적이다.

## 참고

- [ServeTheHome: Fanless Intel N100 Firewall Appliance Review](https://www.servethehome.com/fanless-intel-n100-firewall-and-virtualization-appliance-review/)
- [Proxmox에서 OPNsense 가상화 가이드](https://homenetworkguy.com/how-to/virtualize-opnsense-on-proxmox-as-your-primary-router/)
- [XDA: OPNsense on Proxmox](https://www.xda-developers.com/opnsense-proxmox-best-way-run-network/)
- 본 문서의 가격은 2026년 2월 기준 조사 결과이며, 실제 구매 시 각 판매처에서 최신 가격을 확인할 것
