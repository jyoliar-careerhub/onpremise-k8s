# K8s Service Public Endpoint 노출 설계

## 개요

자택 LAN(10.18.x.x)에 있는 K8s 클러스터의 Service를 인터넷에서 HTTP/HTTPS로 접근할 수 있도록 하는 방안.

## 전제 조건

- 노출 대상: **HTTP/HTTPS 트래픽만** (L7)
- ISP 공인 IP 보유 (포트포워딩 가능)
- 도메인 보유 (Cloudflare 외 DNS)
- 코드(IaC)로 구성 가능해야 함

## 검토한 방안

| 방안 | 방식 | 포트 노출 | DNS 이전 | 100% IaC |
|------|------|:-:|:-:|:-:|
| Cloudflare Tunnel | Outbound 터널 → CF Edge | 불필요 | Cloudflare 필수 | 가능 |
| **Ingress + cert-manager** | 포트포워딩 → LB IP | 80, 443 | **불필요** | 포트포워딩 제외 |

**채택: Ingress Controller + cert-manager + 포트포워딩**

Cloudflare Tunnel은 DNS를 Cloudflare로 이전해야 하고, 외부 서비스 의존도가 높아 제외.

## 아키텍처

### 트래픽 흐름

```
인터넷
  │
  ▼
공유기 (공인 IP, 포트포워딩)
  │  :80  → 10.18.100.x:80
  │  :443 → 10.18.100.x:443
  ▼
kube-vip Service LB (10.18.100.x)
  │
  ▼
Nginx Ingress Controller (Pod)
  │  Host/Path 기반 라우팅
  │  TLS 종료
  ▼
K8s Service → Pod
```

### 현재 kube-vip Service LB 구성

이미 구성되어 있음:

- `kube_vip_lb_enabled=true`
- `kube_vip_lb_range=10.18.100.2-10.18.100.254`
- Nginx Ingress Controller를 `type: LoadBalancer`로 배포하면 kube-vip가 이 범위에서 IP 할당

## 구성 요소

### 1. Nginx Ingress Controller

- K8s 클러스터 내 리버스 프록시
- `type: LoadBalancer` Service로 배포 → kube-vip가 External IP 할당
- Host/Path 기반으로 여러 서비스를 단일 IP:443으로 라우팅
- Helm chart로 배포

### 2. cert-manager

- Let's Encrypt 인증서 자동 발급/갱신
- HTTP-01 또는 DNS-01 challenge 지원
- Helm chart로 배포
- `ClusterIssuer` 리소스로 Let's Encrypt 연동 설정

### 3. DDNS

- 공인 IP가 변경될 수 있으므로 도메인 → 공인 IP 자동 매핑
- ddclient 또는 DNS 제공자의 API 활용

### 4. 공유기 포트포워딩 (수동)

- 80 → kube-vip LB IP:80
- 443 → kube-vip LB IP:443
- IaC 범위 밖 (공유기 관리 화면에서 설정)

## 비교: Cloudflare Tunnel vs 본 방안

| 항목 | Cloudflare Tunnel | Ingress + 포트포워딩 |
|------|:-:|:-:|
| 포트 노출 | 없음 | 80, 443 |
| TLS 인증서 | Cloudflare 자동 | cert-manager + Let's Encrypt |
| DDoS 보호 | 포함 | 없음 |
| DNS 이전 필요 | Cloudflare로 이전 | 불필요 |
| 공유기 설정 | 불필요 | 포트포워딩 필수 |
| 외부 의존 | Cloudflare | Let's Encrypt만 |
| 100% IaC | 가능 | 포트포워딩 제외 |

## 구현 로드맵

1. Nginx Ingress Controller Helm 배포 (Ansible 플레이북)
2. cert-manager Helm 배포 + ClusterIssuer 설정
3. 공유기 포트포워딩 설정 (수동)
4. DDNS 구성
5. 테스트용 Ingress 리소스 배포 → TLS 인증서 발급 확인
