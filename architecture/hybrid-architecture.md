# 하이브리드 아키텍처

## 목적

본 문서는 On-Premise와 AWS를 연결한 Hybrid Infrastructure의 구성, 트래픽 흐름, 확장 전략, 보안 설계, 장애 대응 관점을 정의한다.

Hybrid Architecture의 목적은 다음과 같다.

| 목적 | 설명 |
| --- | --- |
| On-Premise 우선 운영 | 기본 워크로드를 On-Premise에서 처리한다. |
| Cloud 비용 최적화 | 상시 Cloud 리소스 사용을 줄인다. |
| AWS 기반 확장 | 필요 시 AWS EKS를 활용한다. |
| 사설 연결 | AWS Site-to-Site VPN으로 AWS와 On-Premise를 연결한다. |

## 전체 Hybrid 구성

```text
Client
  |
  v
AWS ALB
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
Kubernetes Ingress Controller
  |
  v
Kubernetes Service
  |
  v
Application Pod
```

Hybrid 구조는 AWS를 외부 Entry Point로 사용하고, On-Premise를 기본 Application 실행 환경으로 사용한다. EKS는 필요 시 확장 대상으로 활용한다.

## On-Premise 영역

On-Premise 영역은 기본 워크로드 실행, 데이터 저장, 데이터베이스 운영, 내부 서비스 라우팅을 담당한다.

| 구성 요소 | 역할 |
| --- | --- |
| Proxmox | VM 기반 인프라 제공 |
| Kubernetes | Application Pod 실행 |
| Kubernetes Ingress Controller | VLAN40 Private Network 내부 서비스 라우팅 |
| Ceph | Kubernetes Persistent Volume 및 서비스 이미지 파일 저장소 제공 |
| MariaDB Galera Cluster | 고가용성 DB 구성 |
| Proxmox 상의 pfSense VM | VLAN Gateway, 방화벽, AWS Site-to-Site VPN 종단 |

| 네트워크 | 대역 | 용도 |
| --- | --- | --- |
| Management Network | 192.168.36.0/24 | Proxmox Management 및 pfSense VM WAN |
| VLAN40 Private | 172.17.128.0/22 | Kubernetes, DB, 내부 서비스 |
| Proxmox Cluster / Ceph Network | 10.10.10.0/24 | Proxmox Cluster 및 Ceph 통신 |

pfSense는 물리 장비가 아니라 Proxmox 위에서 동작하는 VM이다. pfSense VM은 WAN을 Management Network에 연결하고, LAN/VLAN 인터페이스를 Proxmox VLAN Trunk Bridge에 연결한다.

### On-Premise 리소스 요약

| 항목 | 값 |
| --- | --- |
| Proxmox Node 수 | 4대 |
| Node당 CPU | 24 core |
| Node당 RAM | 32GB |
| 전체 물리 CPU | 96 core |
| 전체 물리 RAM | 128GB |
| Kubernetes VM | Control Plane 3대, Worker 6대 |
| Kubernetes VM 할당량 | 32 vCPU, 24GB RAM, 180GB Disk |
| 팀 사용 가능 Ceph Storage | 약 3.7TB |
| Ceph RBD Node | 10.10.10.11 |
| 서비스 이미지 파일 저장소 | Ceph S3 |

## AWS 영역

AWS 영역은 외부 Client 진입점, On-Premise 전달 계층, 선택적 확장 환경을 담당한다.

| 구성 요소 | 역할 |
| --- | --- |
| AWS ALB | 외부 Client 요청 수신 |
| EC2 (HAProxy) | ALB 트래픽을 Site-to-Site VPN 경로로 전달 |
| Site-to-Site VPN | AWS와 On-Premise 간 사설 연결 |
| EKS | 필요 시 Application 확장 실행 환경 |

AWS VPC, Subnet, Routing Table, Security Group의 상세 값은 TBD이다.

## 기본 트래픽 흐름

기본 트래픽은 On-Premise Kubernetes에서 처리한다.

```text
Client
  |
  v
AWS ALB
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
VLAN40 (Private)
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

1. Client는 AWS ALB로 접근한다.
2. ALB는 EC2에 구성된 HAProxy로 트래픽을 전달한다.
3. EC2 HAProxy는 Site-to-Site VPN으로 On-Premise에 트래픽을 전달한다.
4. On-Premise VPN 종단은 Proxmox 상의 pfSense VM이 담당한다.
5. pfSense VM은 VLAN40 Private Network로 트래픽을 라우팅한다.
6. Kubernetes Ingress Controller가 Kubernetes Service로 요청을 전달한다.
7. Kubernetes Service가 Application Pod로 요청을 전달한다.

## 확장 트래픽 흐름

On-Premise 리소스가 부족하거나 특정 워크로드 확장이 필요한 경우 AWS EKS를 선택적으로 활용한다.

```text
Client
  |
  v
AWS ALB
  |
  v
EC2 (HAProxy)
  |-----------------------------|
  |                             |
  v                             v
On-Premise Kubernetes           AWS EKS
```

| 항목 | 내용 |
| --- | --- |
| 기본 처리 | On-Premise Kubernetes |
| 확장 처리 | AWS EKS |
| 확장 기준 | TBD |
| Traffic Split 정책 | TBD |
| EKS 배포 Manifest | `k8s-manifest` 기준, 상세 TBD |

## AWS Entry Point 설계

AWS Entry Point는 ALB와 EC2 HAProxy로 구성한다.

```text
Client -> AWS ALB -> EC2 (HAProxy)
```

| 구성 요소 | 설계 역할 |
| --- | --- |
| AWS ALB | 외부 Client 요청을 수신하는 단일 Entry Point |
| EC2 (HAProxy) | ALB에서 받은 트래픽을 On-Premise 또는 EKS 경로로 전달 |

외부 Client는 On-Premise에 직접 접근하지 않고 AWS ALB를 통해 접근한다.

## VPN 설계

```text
AWS VPC
  |
  v
Site-to-Site VPN
  |
  v
Proxmox 상의 pfSense VM
  |
  v
VLAN40 Private Network
```

| 항목 | 값 |
| --- | --- |
| VPN 방식 | AWS Site-to-Site VPN |
| AWS 측 | AWS VPC |
| On-Premise 측 종단 | Proxmox 상의 pfSense VM |
| 목적 | AWS와 On-Premise 간 사설 통신 |
| 상세 Tunnel 설정 | TBD |

pfSense VM은 AWS Site-to-Site VPN 종단으로 동작하며, VPN으로 들어온 서비스 트래픽을 VLAN40 Private Network의 Kubernetes Ingress Controller로 전달한다.

## On-Premise 우선 전략

| 영역 | 기본 위치 | 비고 |
| --- | --- | --- |
| Application | On-Premise Kubernetes | 기본 워크로드 처리 |
| Storage | On-Premise Ceph RBD / Ceph S3 | 약 3.7TB, Kubernetes Persistent Volume 및 서비스 이미지 파일 저장소 |
| Database | On-Premise MariaDB Galera Cluster | 고가용성 DB |
| Network Gateway | Proxmox 상의 pfSense VM | VLAN 라우팅, 방화벽, VPN 종단 |
| 확장 실행 환경 | AWS EKS | 필요 시 사용 |

이 전략은 지속적으로 발생하는 기본 워크로드를 On-Premise에서 처리하여 Cloud 비용을 줄이고, 순간적인 확장 요구에 대해서만 AWS 리소스를 사용하는 방향이다.

## 장애 및 Failover 관점

| 장애 영역 | 대응 관점 |
| --- | --- |
| Application Pod 장애 | Kubernetes Replica 및 재시작 정책으로 복구 |
| Kubernetes Node 장애 | 다른 Node로 Pod 재배치 |
| Storage 장애 | Ceph 복제 구조로 데이터 가용성 확보 |
| DB 장애 | MariaDB Galera Cluster 기반 고가용성 확보 |
| On-Premise 리소스 부족 | AWS EKS로 확장 검토 |
| Site-to-Site VPN 장애 | AWS에서 On-Premise 서비스 접근 영향 발생 |
| AWS ALB 또는 EC2 HAProxy 장애 | 외부 진입 경로 영향 발생 |

세부 Failover 자동화 정책, Health Check 기준, RTO/RPO는 TBD이다.

## 보안 설계

| 영역 | 설계 방향 |
| --- | --- |
| 외부 접근 | AWS ALB로 진입 지점 제한 |
| AWS-On-Premise 연결 | Site-to-Site VPN 기반 사설 통신 |
| On-Premise Gateway | Proxmox 상의 pfSense VM에서 라우팅 및 방화벽 적용 |
| 내부 서비스 | VLAN40 Private Network와 Kubernetes Ingress Controller 중심 접근 |
| 네트워크 분리 | Management, VLAN Service, Proxmox Cluster / Ceph Network 분리 |

pfSense VM은 VLAN 간 라우팅과 방화벽 정책을 적용한다. Firewall Policy는 Top-Down 방식으로 평가되므로 차단 정책을 허용 정책보다 위에 배치한다.

## 설계 의도

Hybrid Architecture는 On-Premise의 고정 인프라를 최대한 활용하면서 AWS의 외부 Entry Point와 확장성을 결합하기 위한 설계이다.

외부 요청은 AWS ALB로 단일화하고, AWS EC2 HAProxy와 Site-to-Site VPN을 통해 On-Premise VLAN40 Private Network의 Kubernetes Ingress Controller까지 전달한다. 이를 통해 On-Premise 내부 서비스는 Private Network 중심으로 유지된다.

스토리지는 Ceph RBD와 Ceph S3, 데이터베이스는 MariaDB Galera Cluster, 애플리케이션 실행은 Kubernetes를 사용하여 각 계층에서 고가용성 전략을 확보한다.

## 관련 문서

- [Architecture Overview](./overview.md)
- [Network Design](./network-design.md)
- [Ceph Storage Architecture](./ceph-architecture.md)
- [Kubernetes Node Spec](./kubernetes-node-spec.md)
- [Docs README](../README.md)
