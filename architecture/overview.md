# 아키텍처 개요

## 1. 프로젝트 목적

사내 임직원의 프로필 이미지를 업로드받아 **AI 스타일 프로필 이미지로 변환**하는 서비스를,
**온프레미스(Proxmox) + AWS** 하이브리드 인프라 위에 Kubernetes/GitOps로 안정적으로 운영·관측하는 것이 목표다.
핵심은 AI 모델 자체가 아니라 **이미지 처리 워크로드의 운영 관측성과 가용성**이다.

## 2. 전체 구성 (실제 구축 기준)

```text
                          [ 외부 사용자 ]
                                │  https://fhwang.cloud
                                ▼
        ┌──────────────────────  AWS (Edge)  ──────────────────────┐
        │  Route53 ─▶ NLB (443 TLS 종료/ACM, 80)                    │
        │              └▶ HAProxy EC2 × 2  (active/active)          │
        │                    └▶ StrongSwan VPN EC2 × 2 (active/backup)│
        └───────────────────────────┬──────────────────────────────┘
                                     │  IPsec 터널 (StrongSwan ↔ ER605)
        ┌────────────────────  On-Prem (Proxmox)  ─────────────────┐
        │  On-Prem HAProxy VIP 172.17.32.20 (keepalived, active/backup)│
        │     └▶ ingress-nginx VIP 172.17.128.240 (MetalLB, 2 replica)│
        │           └▶ gateway ─▶ auth / employee ─▶ photo-service   │
        │                              │                    │        │
        │                          Galera(MariaDB)      Ceph RGW(S3) │
        │  Kubernetes(kubeadm): cp×3(kube-vip API VIP) + worker×N    │
        │  Storage: Ceph (RBD=DB PVC, RGW=이미지 객체)               │
        └────────────────────────────────────────────────────────────┘
```

> AWS는 **Edge(공개 진입 + VPN)** 역할만 한다. **EKS는 사용하지 않으며**, Kubernetes는 온프렘 kubeadm 클러스터다.

## 3. 구성 요소

| 영역 | 구성 |
|---|---|
| **컴퓨트(온프렘)** | Proxmox VE 클러스터, kubeadm Kubernetes (control-plane ×3 + worker ×N) |
| **K8s 네트워킹** | Calico CNI, kube-vip(API VIP), MetalLB(L2, pool `172.17.128.240-250`), ingress-nginx |
| **스토리지** | Ceph — RBD(`ceph-csi-rbd`, Galera PVC) + RGW S3(`photo-service`/backup 이미지 객체) |
| **애플리케이션** | gateway(5000)·auth-server(5001)·employee-server(5002)·photo-service(5003)·frontend/nginx(80) |
| **데이터베이스** | MariaDB Galera 3-replica (multi-primary, `data` ns) |
| **AWS Edge** | Route53(`fhwang.cloud`) + NLB + HAProxy EC2 ×2 + StrongSwan VPN EC2 ×2 + ACM |
| **CI/CD** | GitHub Actions(이미지 빌드/push) → k8s-manifests(Kustomize) → ArgoCD(GitOps) |
| **관측성** | kube-prometheus-stack(Prometheus/Grafana/Alertmanager) + Loki/promtail + metrics-server |
| **시크릿** | External Secrets Operator ← AWS Secrets Manager(`prod/*`) |

## 4. 리포지토리 구성

| repo | 역할 |
|---|---|
| `app` | 애플리케이션 모노레포(서비스 5종) + 이미지 빌드 CI |
| `k8s-manifests` | Kubernetes 선언 상태(GitOps 기준 = ArgoCD source) |
| `infra-proxmox` | 온프렘 Proxmox VM(Terraform) + kubeadm 구성(Ansible) + 온프렘 HAProxy |
| `infra-aws` | AWS Edge(NLB/HAProxy/VPN) Terraform |
| `docs` | 설계·운영·의사결정 문서(본 저장소) |

## 5. 가용성(HA) 요약

| 계층 | 방식 |
|---|---|
| AWS HAProxy | active/active (NLB cross-zone 분산) |
| AWS VPN(StrongSwan) | active/backup (EIP + Lambda 1분 페일오버) |
| 온프렘 HAProxy | active/backup (keepalived VRRP 단일 VIP) |
| ingress-nginx | active/active (2 replica + topologySpread + PDB) |
| K8s control-plane | kube-vip API VIP + control-plane ×3 |
| MariaDB | Galera 3-replica multi-primary |

자세한 흐름·근거는 [hybrid-architecture.md](hybrid-architecture.md), [aws-integration.md](aws-integration.md),
[network-design.md](network-design.md), `../operations/`, `../decisions/` 참고.
