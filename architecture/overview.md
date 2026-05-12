# 하이브리드 인프라 전체 아키텍처

## 목적

본 문서는 On-Premise와 AWS를 연결한 Hybrid Infrastructure의 전체 구조와 주요 설계 방향을 정의한다.

프로젝트의 주요 목표는 다음과 같다.

| 목표 | 설명 |
| --- | --- |
| Cost Optimization | 기본 워크로드를 On-Premise에서 처리하여 Cloud 비용을 최소화한다. |
| Performance Scalability | 필요 시 AWS EKS를 활용하여 확장한다. |
| High Availability | Kubernetes, Ceph, MariaDB Galera 기반으로 장애 대응 구조를 구성한다. |

## 전체 구성 개요

본 프로젝트는 On-Premise를 기본 실행 환경으로 사용하고, AWS를 외부 진입점과 선택적 확장 환경으로 사용하는 Hybrid Infrastructure이다.

```text
Client
  |
  v
AWS NLB
  |
  v
EC2 (HAProxy)
  |
  v
Site-to-Site VPN
  |
  v
Proxmox 상의 pfSense VM
  |
  v
VLAN40 Private Network
  |
  v
On-Prem HAProxy
  |
  v
Kubernetes Ingress Controller
  |
  v
Kubernetes Service
  |
  v
Application Pod
```

Repository는 다음 영역으로 구성된다.

| 디렉토리 | 역할 |
| --- | --- |
| `app` | 애플리케이션 소스 코드 |
| `docs` | 설계, 구축, 운영 문서 |
| `k8s-manifest` | Kubernetes 배포 Manifest |
| `infra-aws` | AWS 인프라 구성 코드 |
| `infra-proxmox` | Proxmox 기반 On-Premise 인프라 구성 코드 |

## 주요 구성 요소

### On-Premise

| 구성 요소 | 역할 |
| --- | --- |
| Proxmox | VM 기반 가상화 인프라 |
| Kubernetes | Application Pod 실행 및 서비스 오케스트레이션 |
| On-Prem HAProxy | VPN 유입 트래픽을 Ingress로 전달 |
| Kubernetes Ingress Controller | On-Premise 내부 서비스 라우팅 |
| Ceph | RBD 기반 PV 및 S3 기반 서비스 이미지 파일 저장소 |
| MariaDB Galera Cluster | 고가용성 데이터베이스 |
| Proxmox 상의 pfSense VM | VLAN Gateway, 방화벽, AWS Site-to-Site VPN 종단 |

### AWS

| 구성 요소 | 역할 |
| --- | --- |
| NLB | 외부 Client 요청 수신 지점 |
| EC2 (HAProxy) | NLB 트래픽을 On-Premise 또는 EKS 경로로 전달 |
| Site-to-Site VPN | AWS와 On-Premise 간 사설 연결 |
| EKS | 필요 시 확장 가능한 Kubernetes 실행 환경 |

### Deployment

| 구성 요소 | 역할 |
| --- | --- |
| GitHub Actions | Container Image Build 및 배포 자동화 트리거 |
| Container Registry | 빌드된 Container Image 저장 |
| ArgoCD | GitOps 기반 Kubernetes 배포 동기화 |

## 인프라 리소스 요약

### Proxmox 물리 리소스

| 항목 | 값 |
| --- | --- |
| Proxmox Node 수 | 4대 |
| Node당 CPU | 24 core |
| Node당 RAM | 32GB |
| 전체 CPU | 96 core |
| 전체 RAM | 128GB |

### Kubernetes VM 리소스

| 구분 | 노드 수 | vCPU 합계 | Memory 합계 | Disk 합계 |
| --- | ---: | ---: | ---: | ---: |
| Control Plane | 3 | 8 vCPU | 12GB | 60GB |
| Worker | 6 | 24 vCPU | 12GB | 120GB |
| Total | 9 | 32 vCPU | 24GB | 180GB |

각 Kubernetes VM의 Disk는 20GB이며, VM별 상세 배치는 [Kubernetes Node Spec](./kubernetes-node-spec.md)에 정리한다.

### Ceph Storage

| 항목 | 값 |
| --- | --- |
| 팀 사용 가능 Ceph Storage | 약 3.7TB |
| Ceph RBD Node | 10.10.10.11 |
| Kubernetes PV 연동 방식 | RBD |
| Object Storage | Ceph S3 |
| 서비스 이미지 파일 저장소 | Ceph S3 |

## 네트워크 구조 요약

On-Premise 네트워크는 관리, 서비스, 스토리지 트래픽을 분리한다.

| 네트워크 | 대역 | 용도 |
| --- | --- | --- |
| Management Network | 192.168.36.0/24 | Proxmox Management 및 pfSense VM WAN |
| VLAN10 Public | 172.17.0.0/24 | Public Service Network |
| VLAN20 DMZ | 172.17.32.0/24 | DMZ Service Network |
| VLAN30 Development | 172.17.64.0/24 | Development Service Network |
| VLAN40 Private | 172.17.128.0/22 | Kubernetes, DB, 내부 서비스 Network |
| Proxmox Cluster / Ceph Network | 10.10.10.0/24 | Proxmox Cluster 통신 및 Ceph Storage Traffic |

서비스 트래픽은 AWS NLB와 AWS HAProxy EC2를 거쳐 Site-to-Site VPN으로 On-Premise에 전달된다. On-Premise 내부에서는 pfSense VM 이후 On-Prem HAProxy가 Ingress로 트래픽을 전달하고, Ingress가 서비스 라우팅을 담당한다.

## 트래픽 흐름

### 기본 서비스 트래픽

```text
Client
  |
  v
AWS NLB
  |
  v
AWS HAProxy EC2 x2
  |
  v
Site-to-Site VPN
  |
  v
Proxmox 상의 pfSense VM
  |
  v
VLAN40 Private Network
  |
  v
On-Prem HAProxy
  |
  v
Ingress
  |
  v
Kubernetes Service
  |
  v
Application Pod
```

### 흐름 설명

1. 외부 Client는 AWS NLB로 접근한다.
2. NLB는 EC2 2대에 구성된 HAProxy로 트래픽을 전달한다.
3. AWS HAProxy는 Site-to-Site VPN을 통해 On-Premise로 트래픽을 전달한다.
4. On-Premise VPN 종단은 Proxmox 상의 pfSense VM이 담당한다.
5. pfSense VM은 트래픽을 VLAN40 Private Network로 라우팅한다.
6. On-Prem HAProxy는 트래픽을 Ingress로 전달한다.
7. Ingress가 내부 서비스 라우팅을 수행한다.
8. Kubernetes Service를 통해 Application Pod로 요청이 전달된다.

## 스케일링 전략

기본 워크로드는 On-Premise Kubernetes Cluster에서 처리한다. On-Premise 리소스가 부족하거나 특정 서비스의 처리량 확장이 필요한 경우 AWS EKS를 선택적 확장 영역으로 활용한다.

| 구분 | 기본 처리 위치 | 확장 위치 |
| --- | --- | --- |
| Application | On-Premise Kubernetes | AWS EKS |
| Storage | On-Premise Ceph | TBD |
| Database | On-Premise MariaDB Galera Cluster | TBD |

확장 기준과 Traffic Split 정책은 [Hybrid Architecture](./hybrid-architecture.md)에서 관리한다.

## 스토리지 구조

Ceph는 Kubernetes Persistent Volume과 서비스 이미지 파일 저장소의 기반 스토리지로 사용한다.

| 항목 | 내용 |
| --- | --- |
| Storage Backend | Ceph RBD, Ceph S3 |
| 팀 사용 가능 용량 | 약 3.7TB |
| Ceph RBD Node | 10.10.10.11 |
| Kubernetes PV 연동 | Ceph RBD |
| 서비스 이미지 파일 저장소 | Ceph S3 |
| Storage Traffic | 10.10.10.0/24 Proxmox Cluster / Ceph Network |
| 주요 용도 | Application 상태 데이터, DB Persistent Storage, 서비스 이미지 파일 저장 |

Ceph RBD와 Ceph S3는 같은 Ceph Storage를 사용한다. 팀 사용 가능 용량은 약 3.7TB이다. 상세 구조는 [Ceph Storage Architecture](./ceph-architecture.md)에서 관리한다.

## 데이터베이스 구조

데이터베이스는 MariaDB Galera Cluster 기반으로 구성한다.

| 항목 | 내용 |
| --- | --- |
| DB | MariaDB Galera Cluster |
| 고가용성 방식 | Galera Cluster |
| Persistent Storage | Ceph RBD 기반 Kubernetes Persistent Volume |
| 배치 위치 | VLAN40 Private Network |

세부 노드 수, DB Endpoint, 백업 정책은 TBD

## 배포 구조(GitOps)

배포는 GitHub Actions, Container Registry, ArgoCD를 조합한 GitOps 흐름을 따른다.

```text
app
  |
  v
GitHub Actions
  |
  v
Container Registry
  |
  v
k8s-manifest
  |
  v
ArgoCD
  |
  v
Kubernetes Cluster
```

| 단계 | 설명 |
| --- | --- |
| Build | GitHub Actions가 애플리케이션을 빌드한다. |
| Push | 빌드된 이미지를 Container Registry에 저장한다. |
| Manifest Update | Kubernetes Manifest 변경 사항을 관리한다. |
| Sync | ArgoCD가 Git 상태를 기준으로 Kubernetes Cluster에 반영한다. |

## 모니터링

모니터링 구성은 Prometheus와 Grafana를 기준으로 한다.

| 구성 요소 | 역할 |
| --- | --- |
| Prometheus | Metric 수집 |
| Grafana | Dashboard 시각화 |
| Alert 정책 | TBD |
| Log 수집 | TBD |

## 고가용성 전략

| 영역 | 전략 |
| --- | --- |
| Application | Kubernetes Replica 및 Service 기반 장애 대응 |
| Storage | Ceph 분산 스토리지 |
| Database | MariaDB Galera Cluster |
| Network | pfSense VM 기반 Gateway 및 방화벽 정책 |
| Deployment | ArgoCD 기반 선언적 배포 상태 복구 |
| Cloud 확장 | 필요 시 AWS EKS 활용 |

## 설계 의도

본 아키텍처는 On-Premise를 기본 실행 환경으로 유지하여 Cloud 비용을 최소화하면서, AWS를 외부 진입점과 확장 환경으로 활용하기 위한 구조이다.

On-Premise 내부 서비스 라우팅은 Ingress 중심으로 구성한다. pfSense VM은 VLAN Gateway, 방화벽, VPN 종단을 담당하고, Ingress는 Application Service 라우팅을 담당한다.

Management Network, VLAN Service Network, Proxmox 10G Cluster / Ceph Network를 분리하여 관리 트래픽, 서비스 트래픽, 스토리지 트래픽이 서로 영향을 주지 않도록 설계한다.

## 관련 문서

- [Network Design](./network-design.md)
- [Hybrid Architecture](./hybrid-architecture.md)
- [Ceph Storage Architecture](./ceph-architecture.md)
- [Kubernetes Node Spec](./kubernetes-node-spec.md)
- [Docs README](../README.md)
