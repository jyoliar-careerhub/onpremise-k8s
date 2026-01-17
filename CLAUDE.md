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

## 작업 중 주의사항

작업 중 이슈 발생 시, 원인의 파악 단계까지만 진행하고 조치 단계 이전에는 사용자의 작업 진행 허용을 요청하십시오.

## 주요 명령어

```bash
# 전체 클러스터 설치
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
