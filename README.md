# 노드 · 클러스터 하부 — onpremise-k8s

홈랩 노드와 kubeadm 클러스터 하부를 세우는 Ansible 저장소다. 클러스터 안은 jyoliar-k8s 가, 클라우드 쪽은 jyoliar-platform-terraform 이 맡는다.

## 디렉터리 구조

```
kubernetes/            # 클러스터 워크스페이스 — OS 준비 · CRI-O · kubeadm · kube-vip · Secret 암호화 · 스토리지 디스크 · 업그레이드
├── ansible.cfg
├── inventory/         # 환경별 인벤토리 — 파일명이 환경명
├── playbooks/         # 실행 단위 — 상황별 입구는 「반영 방법」
└── roles/
vpn/                   # 운영자 접근 워크스페이스 — Tailscale Subnet Router
├── ansible.cfg
├── inventory/
├── playbooks/
└── roles/
docs/                  # 설계 · 검토 문서
worklogs/              # 작업 일지
remove_known_hosts.sh  # 서버를 새로 만든 뒤 SSH known_hosts 정리
```

두 워크스페이스는 독립이다. 각자 `ansible.cfg` 를 갖고, Ansible 명령은 그 디렉터리 안에서 실행한다.

## 환경

- 환경은 인벤토리 파일이다 — `kubernetes/inventory/<env>` · `vpn/inventory/<env>`. 파일명이 환경명이고, 디렉터리의 파일 목록이 환경 목록이다.
- 노드 그룹(컨트롤 플레인 · 워커 · 새로 더할 노드 · Raspberry Pi 전용)과 버전 · 네트워크 · kube-vip · Secret 암호화 · OIDC 발급자 변수는 전부 인벤토리에 있다. 값은 인벤토리가 답한다.
- 구축이 끝난 클러스터의 admin kubeconfig(`/etc/kubernetes/admin.conf`)는 로컬 `.kube/<env>.config` 로 복제해 쓴다 — gitignore 대상이다.

## 반영 방법

워크스페이스 디렉터리에서 인벤토리를 지정해 플레이북을 돌린다.

```bash
cd kubernetes
ansible -m ping all -i inventory/<env>
ansible-playbook -i inventory/<env> playbooks/<플레이북>.yaml
```

실패하면 원인까지만 보고 멈춘다 — cleanup · 재시도는 사람의 결정이다.

**클러스터 워크스페이스(`kubernetes/`)** — 상황별로 보는 플레이북

| 상황 | 플레이북 | 비고 |
|---|---|---|
| 새 클러스터 | [`k8s-all.yaml`](kubernetes/playbooks/k8s-all.yaml) | 한 번에 돌리지 않는다. 파일을 열어 import 순서대로 하나씩 돌리고 단계마다 확인한다. 서버를 새로 만들었으면 먼저 [`remove_known_hosts.sh`](remove_known_hosts.sh) |
| 새 노드 | [`bootstrap-node.yaml`](kubernetes/playbooks/bootstrap-node.yaml) → [`raspi-all.yaml`](kubernetes/playbooks/raspi-all.yaml) → [`add-new-workers-all.yaml`](kubernetes/playbooks/add-new-workers-all.yaml) 또는 [`kubeadm-add-new-masters.yaml`](kubernetes/playbooks/kubeadm-add-new-masters.yaml) | 인벤토리의 새 노드 그룹에 넣은 뒤. 부트스트랩만 `-K`, 이후는 무인 |
| 패치 업그레이드 | [`kubeadm-upgrade.yaml`](kubernetes/playbooks/kubeadm-upgrade.yaml) | 목표 버전은 인벤토리의 버전 변수가 정한다 |
| kubeadm 설정 변경 | [`kubeadm-reconfigure.yaml`](kubernetes/playbooks/kubeadm-reconfigure.yaml) | 템플릿 [`kubeadm-config.yaml`](kubernetes/roles/kubeadm-init-prep/templates/kubeadm-config.yaml) 이 유일한 진실. OIDC 발급자를 바꿨으면 이어서 [`rollout-restart-all.yaml`](kubernetes/playbooks/rollout-restart-all.yaml) |
| Secret 암호화 | [`encrypting-secrets.yaml`](kubernetes/playbooks/encrypting-secrets.yaml) → 인벤토리 `apiserver_encryption_enabled` → `kubeadm-reconfigure.yaml` | 순서는 `k8s-all.yaml` 의 주석 |
| Longhorn 디스크 | [`longhorn-prep.yaml`](kubernetes/playbooks/longhorn-prep.yaml) | 스토리지 노드의 디스크 마운트 |
| 재설치 | [`cleanup-all-vms.yaml`](kubernetes/playbooks/cleanup-all-vms.yaml) | 모든 노드의 클러스터 데이터를 지운다. 사람의 확인 뒤에만 |

**운영자 접근 워크스페이스(`vpn/`)**

| 상황 | 플레이북 |
|---|---|
| Subnet Router 후보 노드 구성 | [`tailscale-sr-candidate.yaml`](vpn/playbooks/tailscale-sr-candidate.yaml) — 대상 그룹은 인벤토리 |
| 노드에서 Tailscale 제거 | [`cleanup-tailscale-workers.yaml`](vpn/playbooks/cleanup-tailscale-workers.yaml) |

## 다른 저장소와의 경계

받는 것 — 값의 합의 하나다. 코드나 산출물을 읽는 일은 없다:

| 넘기는 저장소 | 무엇 | 그쪽의 자리 | 이 저장소의 자리 |
|---|---|---|---|
| jyoliar-platform-terraform | OIDC 발급자 URL | `iam` 변수 `k8s_oidc_issuer_url` 이 먼저 정한다 | 인벤토리 `service_account_issuer` → kubeadm 템플릿. 같은 값이어야 IRSA 가 선다 |

`vpn/` 의 Tailscale 인증 키는 Tailscale 콘솔에서 오고, 이 셋 밖이다.

넘기는 것:

| 받는 저장소 | 무엇 | 이 저장소의 자리 | 그쪽의 자리 |
|---|---|---|---|
| jyoliar-k8s | 클러스터와 admin kubeconfig | 구축 뒤 `/etc/kubernetes/admin.conf` | `.kube/{env}.config` — 초기 구축 스크립트가 요구 |
| jyoliar-k8s | kube-vip 컨트롤 플레인 VIP · LoadBalancer IP 풀 | 인벤토리의 kube-vip 변수 · [`kubernetes/roles/kube-vip`](kubernetes/roles/kube-vip) | Gateway 의 LoadBalancer 주소 |
| jyoliar-k8s | Longhorn 용 디스크 | [`longhorn-prep.yaml`](kubernetes/playbooks/longhorn-prep.yaml) | `platform/overlays/{env}/longhorn` |
