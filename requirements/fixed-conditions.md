# 고정 조건 (Fixed Conditions)

프로젝트 시작 시점에 **주어진(변경 불가에 가까운) 조건**.

## 인프라

- 온프렘 가상화: **Proxmox VE 클러스터**(노드망 `192.168.36.0/24`).
- Kubernetes: **kubeadm**(관리형 EKS 아님), control-plane ×3 + worker ×N.
- 스토리지: **Ceph**(mon/RGW `10.10.10.11`), RBD + RGW S3.
- 네트워크 경계: **pfSense** + VLAN(DMZ VLAN20 `172.17.32.0/24`).
- 사이트 간 연결: **TP-Link ER605** 라우터(IPsec initiator).

## 클라우드

- AWS 리전 `ap-northeast-2`, 기존 VPC `cloud-team1-vpc`(`10.1.0.0/16`).
- 도메인 `fhwang.cloud`(Route53), ACM 인증서(`ap-northeast-2`).
- 이미지 레지스트리: Docker Hub `kosa1team/*`.

## 운영/배포

- 배포: **ArgoCD GitOps**(source = `k8s-manifests` dev 브랜치) + GitHub Actions(빌드).
- 시크릿 원본: **AWS Secrets Manager**(`prod/*`) + External Secrets Operator.
- Git 전략: **Git Flow**(`main`/`dev`/`feature/*`/`release/*`/`hotfix/*`), squash 머지.

## 네트워크 고정값(요지)

| 영역 | 대역/값 |
|---|---|
| 관리망 | `172.17.128.0/22` (MetalLB pool `.240-.250`) |
| K8s 노드망 | `10.10.10.0/24` |
| DMZ(VLAN20) | `172.17.32.0/24` (HAProxy VIP `.20`) |

> 상세/최신값은 [../architecture/network-design.md](../architecture/network-design.md) 및 각 인프라 repo README 기준.
