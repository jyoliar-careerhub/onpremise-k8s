# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 언어 설정

이 프로젝트의 모든 마크다운(.md) 파일은 한국어로 작성해야 합니다.

## 프로젝트 개요

Ubuntu 24.04 LTS 환경에서 다중 데이터센터를 지원하는 고가용성(HA) Kubernetes 멀티마스터 클러스터와 Tailscale VPN을 자동 구축하는 Ansible 기반 IaC(Infrastructure as Code) 프로젝트입니다.

## 워크스페이스 구조

프로젝트는 두 개의 독립된 Ansible 워크스페이스로 구성됩니다:

```
kubernetes/   # K8s 클러스터 구축 (roles, playbooks, inventory, ansible.cfg)
vpn/          # Tailscale Subnet Router 설정 (roles, playbooks, inventory, ansible.cfg)
```

각 워크스페이스는 자체 `ansible.cfg`를 포함하며, 해당 디렉토리에서 Ansible 명령을 실행해야 합니다.

## 세션 시작 절차

### 워크스페이스 및 환경 선택

- 세션 시작 시 작업 대상 워크스페이스(`kubernetes` 또는 `vpn`)를 확인합니다.
- 각 워크스페이스의 `inventory/` 경로에서 인벤토리 파일을 확인하여 작업 환경을 질문합니다.
- 인벤토리 파일명이 곧 환경명입니다 (예: `kubernetes/inventory/baremetal` → 환경명: `baremetal`).
- 환경이 하나뿐일 경우 질문을 생략하고 해당 환경을 사용합니다.

### kubeconfig 파일 관리

- kubeconfig 파일은 `.kube/{환경명}.config` 경로에 위치합니다 (예: `.kube/baremetal.config`).
- 쿠버네티스 구축 직후 또는 kubeconfig 파일이 존재하지 않는 경우:
  - 해당 환경의 쿠버네티스 클러스터에서 `/etc/kubernetes/admin.conf` 파일을 `.kube/{환경명}.config`로 복제합니다.

## ⚠️ 필수 행동 규칙

**작업 실패/오류 발생 시:**
1. 원인 파악까지만 진행
2. **절대 임의로 조치하지 않음** (cleanup, rollback, 재시도 등 금지)
3. 상황을 보고하고 사용자 지시 대기

이 규칙을 어기면 안 됩니다.

## 신규 K8s 클러스터 구축 전 필수 작업

서버를 새로 생성한 후 K8s 클러스터를 구축할 때는 먼저 다음 스크립트를 실행하세요:

```bash
./remove_known_hosts.sh
```

이 스크립트는:
- SSH Agent에 키를 로드합니다 (Agent Forwarding 용)
- 이전 서버의 SSH 호스트 키를 known_hosts에서 제거합니다

## 신규 노드 추가 시 필수 부트스트랩 (NOPASSWD sudo)

신규 노드는 `ansible_user`에 passwordless sudo(NOPASSWD)가 설정되어 있어야 모든 playbook이 무인(`become: yes`)으로 실행됩니다. 기존 클러스터 노드에는 이미 적용되어 있으나(`/etc/sudoers.d/010_<user>-nopasswd`), **새로 추가하는 노드에는 빠져 있어** 그대로 playbook을 돌리면 첫 태스크(Gathering Facts)에서 `Missing sudo password`로 실패합니다.

`ansible.cfg`에는 `become_ask_pass`가 설정되어 있지 않으므로(무인 운영 전제), 신규 노드는 **다른 작업보다 먼저** 부트스트랩 playbook을 1회 실행해 NOPASSWD를 설치해야 합니다. 이 단계만 `-K`(`--ask-become-pass`)가 필요하며, 이후 단계는 모두 무인입니다.

```bash
cd kubernetes

# 신규 노드를 인벤토리 [new-k8s-workers] 그룹에 등록한 뒤, 최초 1회만 -K 로 실행
ansible-playbook -i inventory/baremetal playbooks/bootstrap-node.yaml --limit <신규노드> -K

# 이후 Pi 사전작업 / 워커 추가는 -K 없이 무인 진행
ansible-playbook -i inventory/baremetal playbooks/raspi-all.yaml --limit <신규노드>   # Pi 전용
ansible-playbook -i inventory/baremetal playbooks/add-new-workers-all.yaml
```

- `bootstrap-node.yaml` → `bootstrap-sudo` role: `copy ... validate: 'visudo -cf %s'`로 임시파일을 검증한 뒤에만 `/etc/sudoers.d/010_<ansible_user>-nopasswd`를 설치하므로 sudoers 문법 오류로 인한 sudo 잠금 사고를 방지합니다.
- 노드에서 수동 설정도 가능합니다: `echo '<user> ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/010_<user>-nopasswd && sudo chmod 0440 /etc/sudoers.d/010_<user>-nopasswd && sudo visudo -c`

## cleanup-all-vms 실행 시 주의

`cleanup-all-vms.yaml` playbook은 모든 노드의 Kubernetes 데이터를 삭제하는 파괴적 작업입니다. 실행 전 **반드시 사용자에게 확인을 받고** 허가가 있을 때만 실행하세요.

## K8s 클러스터 구축 시 주의

`k8s-all.yaml`을 한 번에 실행하지 마세요. 해당 파일을 읽고 포함된 playbook들을 **순차적으로 개별 실행**하세요. 각 단계 완료 후 결과를 확인하고 다음으로 진행합니다.

## 주요 명령어

### Kubernetes 워크스페이스

```bash
cd kubernetes

# 전체 클러스터 설치 (비권장 - 순차 개별 실행 권장)
ansible-playbook -i inventory/mycluster playbooks/k8s-all.yaml

# 연결 테스트
ansible -m ping all -i inventory/mycluster

# 개별 플레이북 실행 (예시)
ansible-playbook -i inventory/mycluster playbooks/crio.yaml
ansible-playbook -i inventory/mycluster playbooks/smoke-test.yaml

# 클러스터 초기화 (재설치 시)
ansible-playbook -i inventory/mycluster playbooks/cleanup-all-vms.yaml

# 시크릿 암호화 키 생성
head -c 32 /dev/urandom | base64
```

### VPN 워크스페이스

```bash
cd vpn

# Tailscale Subnet Router 배포
ansible-playbook -i inventory/baremetal playbooks/tailscale-subnet-router.yaml

# 연결 테스트
ansible -m ping all -i inventory/baremetal
```

## 클러스터 접근

특정 환경의 클러스터에 접근할 때:

```bash
kubectl <동사> [리소스] [-n <namespace>] --kubeconfig .kube/{환경명}.config
```

**중요**: `kubectl` 바로 뒤에 동사(get, apply, delete 등)를 먼저 위치시키고, 옵션들(`-n`, `--kubeconfig` 등)은 뒤에 배치합니다.

```bash
# 예시
kubectl get nodes --kubeconfig .kube/{환경명}.config
kubectl get pods -n kube-system --kubeconfig .kube/{환경명}.config
kubectl apply -f manifest.yaml -n default --kubeconfig .kube/{환경명}.config
```

## 아키텍처

### Kubernetes 설치 흐름 (k8s-all.yaml)

1. **저장소 설정**: repos.yaml (apt 저장소 구성)
2. **시간 동기화**: ntpd.yaml (chrony 사용)
3. **컨테이너 런타임**: crio.yaml (CRI-O 설치)
4. **Kubernetes 설치**: kubeadm.yaml
5. **방화벽 설정**: setup-ufw.yaml (ufw 사용)
6. **HA etcd 클러스터**: ha-etcd.yaml (CFSSL 인증서 기반)
7. **로드밸런서**: setup-haproxy.yaml (API 서버 HA)
8. **컨트롤 플레인**: kubernetes-masters.yaml → kubernetes-add-workers.yaml
9. **재구성 및 보안**: config-k8s-for-haproxy.yaml → encrypting-secrets.yaml
10. **검증**: smoke-test.yaml

### Kubernetes 인벤토리 구조

```
[dc{N}-k8s-masters]     # 각 데이터센터의 마스터 노드
[dc{N}-k8s-workers-vm]  # 각 데이터센터의 VM 워커 노드
[dc{N}-k8s-workers-bm]  # 각 데이터센터의 베어메탈 워커 노드
[k8s-masters:children]  # 전체 마스터 노드 그룹
[k8s-workers:children]  # 전체 워커 노드 그룹
[new-k8s-masters]       # 새로 추가할 마스터 노드
[new-k8s-workers]       # 새로 추가할 워커 노드
```

### VPN 인벤토리 구조

```
[subnet-routers]        # Tailscale Subnet Router 노드 (CP 노드 3대, HA 구성)
```

### 주요 설정 변수 (kubernetes/inventory)

- `crio_version`: CRI-O 버전 (예: 1.35)
- `kubernetes_version`: Kubernetes 버전 (예: 1.35)
- `kubelet_version`, `kubeadm_version`, `kubectl_version`: 컴포넌트 버전
- `pod_network_cidr`: Pod 네트워크 CIDR (기본: 10.244.0.0/16, Flannel용)
- `virtual_ip`: Keepalived용 VIP
- `setup`: `public` 또는 `private` (프라이빗 레지스트리 사용 여부)
- `encoded_secret`: apiserver Secret 암호화 키 **목록**(`[{name, secret}]`, Base64) — 인벤토리 평문이 아니라 `inventory/group_vars/all/vault.yml` 에 있고 파일 전체가 ansible-vault 로 암호화되어 있다. vault password 는 `kubernetes/.vault_pass`(gitignore, `ansible.cfg` 의 `vault_password_file`). 첫 키가 쓰기 키 — 로테이션은 ops-workflow#73

### Ansible Vault password 의 위치

`kubernetes/.vault_pass` 는 gitignore 대상이라 clone · worktree 에는 없다. 값을 찾을 때는 ticket-engineer 하네스의 참조 저장소 경로 `workspace/ref/onpremise-k8s/kubernetes/.vault_pass` 를 본다 — 거기 보관본이 있다. 작업 worktree 의 `kubernetes/` 에 같은 이름으로 복사하면 `ansible.cfg` 가 집어 쓴다. 값은 대화에 출력하지 않는다.

### 주요 설정 변수 (vpn/inventory)

- `tailscale_auth_key`: Tailscale 인증 키
- `tailscale_advertise_routes`: Subnet Router가 광고할 CIDR
- `tailscale_gateway`: 게이트웨이 IP

### 역할(Roles) 구조

Ansible 역할은 각 워크스페이스의 `roles/` 디렉토리에 위치하며, 표준 Ansible 구조를 따릅니다:

- `tasks/main.yaml`: 태스크 정의
- `templates/`: Jinja2 템플릿 (선택)
- `handlers/`: 핸들러 정의 (선택)
- `files/`: 정적 파일 (선택)

### Kubernetes 주요 역할

- `repos`: apt 저장소 설정 (CRI-O, Kubernetes)
- `crio`: CRI-O 컨테이너 런타임 설치
- `ufw`: Ubuntu 방화벽 설정
- `kubeadm`: Kubernetes 컴포넌트 설치
- `ntpd`: chrony 시간 동기화
- `etcd-ha`: HA etcd 클러스터 구성
- `haproxy`: API 서버 로드밸런싱
- `keepalived`: VIP 관리

### VPN 주요 역할

- `tailscale`: Tailscale 설치 및 Subnet Router 구성 (SNAT=true)
- `tailscale-health-check`: SR health check 스크립트 및 systemd 서비스
