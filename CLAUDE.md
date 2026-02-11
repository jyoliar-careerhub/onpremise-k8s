# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 언어 설정

이 프로젝트의 모든 마크다운(.md) 파일은 한국어로 작성해야 합니다.

## 프로젝트 개요

Ubuntu 24.04 LTS 환경에서 다중 데이터센터를 지원하는 고가용성(HA) Kubernetes 멀티마스터 클러스터를 자동 구축하는 Ansible 기반 IaC(Infrastructure as Code) 프로젝트입니다.

## 세션 시작 절차

### 작업 환경 선택

- 세션 시작 시 `inventory/` 경로의 인벤토리 파일을 확인하여 작업 환경을 질문합니다.
- 인벤토리 파일명이 곧 환경명입니다 (예: `inventory/aws-ec2` → 환경명: `aws-ec2`).
- 환경이 하나뿐일 경우 질문을 생략하고 해당 환경을 사용합니다.

### kubeconfig 파일 관리

- kubeconfig 파일은 `.kube/{환경명}.config` 경로에 위치합니다 (예: `.kube/aws-ec2.config`).
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

## K8s 클러스터 구축 시 주의

`k8s-all.yaml`을 한 번에 실행하지 마세요. 해당 파일을 읽고 포함된 playbook들을 **순차적으로 개별 실행**하세요. 각 단계 완료 후 결과를 확인하고 다음으로 진행합니다.

## 주요 명령어

```bash
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

### 설치 흐름 (k8s-all.yaml)

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

### 인벤토리 구조

```
[dc{N}-k8s-masters]     # 각 데이터센터의 마스터 노드
[dc{N}-k8s-workers-vm]  # 각 데이터센터의 VM 워커 노드
[dc{N}-k8s-workers-bm]  # 각 데이터센터의 베어메탈 워커 노드
[k8s-masters:children]  # 전체 마스터 노드 그룹
[k8s-workers:children]  # 전체 워커 노드 그룹
[new-k8s-masters]       # 새로 추가할 마스터 노드
[new-k8s-workers]       # 새로 추가할 워커 노드
```

### 주요 설정 변수 (inventory/mycluster)

- `crio_version`: CRI-O 버전 (예: 1.35)
- `kubernetes_version`: Kubernetes 버전 (예: 1.35)
- `kubelet_version`, `kubeadm_version`, `kubectl_version`: 컴포넌트 버전
- `pod_network_cidr`: Pod 네트워크 CIDR (기본: 10.244.0.0/16, Flannel용)
- `virtual_ip`: Keepalived용 VIP
- `setup`: `public` 또는 `private` (프라이빗 레지스트리 사용 여부)
- `encoded_secret`: 시크릿 암호화 키 (Base64)

### 역할(Roles) 구조

Ansible 역할이 `roles/` 디렉토리에 위치하며, 각 역할은 표준 Ansible 구조를 따릅니다:

- `tasks/main.yaml`: 태스크 정의
- `templates/`: Jinja2 템플릿 (선택)
- `handlers/`: 핸들러 정의 (선택)
- `files/`: 정적 파일 (선택)

### 주요 역할

- `repos`: apt 저장소 설정 (CRI-O, Kubernetes)
- `crio`: CRI-O 컨테이너 런타임 설치
- `ufw`: Ubuntu 방화벽 설정
- `kubeadm`: Kubernetes 컴포넌트 설치
- `ntpd`: chrony 시간 동기화
- `etcd-ha`: HA etcd 클러스터 구성
- `haproxy`: API 서버 로드밸런싱
- `keepalived`: VIP 관리
