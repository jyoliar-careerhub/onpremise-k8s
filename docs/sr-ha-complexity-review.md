# Tailscale Subnet Router HA: 복잡도 리뷰 및 제거 제안

## 요약

Tailscale SR의 SPOF를 제거하기 위해 CP 노드 3대에 HA SR을 구성했으나, 이로 인해 발생한 문제들이 HA의 이점을 크게 초과한다. 이 문서는 SR HA로 인해 발생한 이슈들을 정리하고, HA를 걷어내야 하는 근거를 제시한다.

## SR HA로 인해 발생한 이슈 목록

### 1. Non-primary SR의 transit 트래픽 드랍

**문제**: 동일 서브넷을 3대가 광고하면 Tailscale이 Primary를 1대만 선출한다. Non-primary SR은 물리 네트워크에서 들어온 포워딩 트래픽(transit)을 Tailscale 터널로 전달하지 않고 드랍한다.

**영향**: 공유기 정적 경로가 non-primary SR을 가리키면 cross-site 통신 전면 두절.

```
공유기 → 10.18.1.1 (VIP, raspberrypi-0 보유) → Tailscale 터널 전달 시도
  → raspberrypi-0이 non-primary → transit 패킷 드랍 → 통신 실패
```

**우회**: 정적 경로를 현재 Primary SR IP로 수동 변경.

- 참고: `docs/cross-site-k8s-worker-join.md`

---

### 2. `--accept-routes` 활성화 시 SSH 끊김

**문제**: cross-site 통신을 위해 SR에 `--accept-routes`를 켜면, Tailscale이 table 52에 자기 서브넷 경로(`10.18.0.0/16 dev tailscale0`)를 설치한다. 이 경로가 main 테이블의 로컬 LAN 경로(`10.18.1.0/24 dev eth1`)보다 우선 조회되어 **모든 로컬 트래픽(SSH 응답 포함)이 tailscale0으로 빠진다.**

**영향**: SR 노드에 대한 SSH 연결 즉시 끊김.

**우회**: 보호 ip rule 추가 (`ip rule add to 10.18.0.0/16 lookup main priority 5269`).

- 참고: `docs/cross-site-routing-accept-routes.md`

---

### 3. SNAT=false 시 anti-spoofing에 의한 응답 드랍

**문제**: SNAT을 비활성화하면 대상 노드가 원본 소스 IP를 보존하여 자신의 tailscale0으로 직접 응답한다. 그러나 Tailscale 클라이언트는 서브넷 대역 패킷을 **Primary SR에서만 수신 허용**하므로, non-primary SR 노드가 보낸 응답은 anti-spoofing 정책에 의해 드랍된다.

**영향**: Primary SR 자기 자신에게만 접근 가능, 나머지 모든 노드 접근 불가.

**우회**: SNAT=true로 전환. 소스 IP 보존 포기.

- 참고: `docs/tailscale-snat-routing-analysis.md`

---

### 4. Primary SR 변동 시 정적 경로 수동 갱신 필요

**문제**: Tailscale Primary SR은 노드 재시작, 네트워크 장애 등으로 다른 노드로 자동 이동한다. 공유기 정적 경로는 고정 IP이므로, Primary가 이동하면 cross-site 통신이 끊어진다.

**영향**: Primary 변동 시마다 수동 개입 필요. 자동화 불가 (공유기 API 없음).

**우회**: 없음. 근본적 한계.

---

### 5. 각 K8s 노드에 Tailscale 설치 시도 (ip rule + systemd 우회)

**문제**: SR Primary 의존성을 제거하기 위해 모든 K8s 노드에 Tailscale을 직접 설치하고 `--accept-routes`로 원격 사이트 경로를 수신하려 했다.

**추가 발생 이슈**:
- 이슈 2와 동일한 보호 ip rule 필요
- Raspberry Pi OS에 `networkd-dispatcher` 미설치 → ip rule 영속화 실패 → systemd oneshot 서비스로 전환 필요
- SR role, node role 두 벌의 Tailscale 역할 관리 필요

---

### 6. Flannel VXLAN `SO_BINDTODEVICE`로 Tailscale 우회 불가 (최종 실패)

**문제**: 각 노드에 Tailscale을 설치해도, Flannel VXLAN이 물리 인터페이스(ens5, eth1)에 `SO_BINDTODEVICE`로 바인딩되어 있어 **커널 라우팅 테이블(table 52, tailscale0)을 완전히 우회**한다. VXLAN 패킷은 물리 인터페이스를 통해 직접 전송되며, Tailscale 터널을 타지 않는다.

**영향**: 노드에 Tailscale을 설치하는 접근 자체가 Pod 네트워크에 무의미. ip rule, 보호 룰, systemd 서비스 등 모든 우회가 VXLAN 트래픽에 효과 없음.

**검증 결과**:
```
AWS 노드 tcpdump:
  ens5:       2 packets captured (VXLAN 패킷이 ens5로 나감)
  tailscale0: 0 packets captured (Tailscale 터널 미사용)
```

**우회**: 라우팅 테이블(VPC route, 공유기 정적 경로)을 복원하면 동작하지만, SR Primary 의존성이 부활.

---

## 복잡도 누적 다이어그램

```
SR SPOF 해결하자
  → HA SR 구성 (3대)
    → Primary만 transit 전달, non-primary 드랍 [이슈 1]
      → --accept-routes 켜자
        → SSH 끊김 [이슈 2]
          → 보호 ip rule 추가
            → SNAT=false이면 anti-spoofing 드랍 [이슈 3]
              → SNAT=true로 전환
                → Primary 변동 시 정적 경로 깨짐 [이슈 4]
                  → 각 노드에 Tailscale 직접 설치 [이슈 5]
                    → Pi OS networkd-dispatcher 없음 → systemd oneshot
                      → VXLAN SO_BINDTODEVICE로 Tailscale 우회 [이슈 6]
                        → ??? (더 이상 우회 불가)
```

하나의 문제를 해결할 때마다 새로운 문제가 발생하고, 각 우회 조치가 시스템 복잡도를 누적시켰다.

## SR 1대 구성으로의 전환 제안

### HA 제거 시 해소되는 것

| 이슈 | HA 제거 시 |
|------|-----------|
| Non-primary transit 드랍 | Primary 경합 없음 — 1대가 유일한 SR |
| accept-routes SSH 끊김 + 보호 ip rule | SR 1대만 accept-routes 필요, 나머지 노드 불필요 |
| SNAT anti-spoofing | Primary 불일치 자체가 없음 |
| Primary 변동 시 정적 경로 깨짐 | Primary 고정 — 정적 경로 불변 |
| 노드별 Tailscale 설치 | 불필요 |
| systemd oneshot ip rule 영속화 | 불필요 |
| VXLAN SO_BINDTODEVICE 문제 | 물리 경로로 VXLAN 전달 — 정상 동작 |

### SR 1대의 SPOF 영향 범위

SR이 죽었을 때 **영향받는 것**:
- Cross-site 통신 (홈랩 ↔ AWS)
- 외부에서 Tailscale 경유 클러스터 접근

SR이 죽어도 **영향 없는 것**:
- 같은 사이트 내 K8s 클러스터 동작
- 같은 사이트 내 Pod 통신
- 로컬 kubectl 접근

### 결론

SR SPOF의 실질적 영향은 cross-site 통신 일시 중단에 한정된다. 이를 위해 6건의 연쇄적 이슈와 우회 조치를 감수하는 것은 비용 대비 효과가 맞지 않는다. **SR 1대 구성으로 단순화하는 것을 권장한다.**
