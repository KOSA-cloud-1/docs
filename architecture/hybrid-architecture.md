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
Kubernetes Ingress Controller
  |
  v
Kubernetes Service
  |
  v
Application Pod
```

Hybrid 구조는 AWS를 외부 Entry Point로 사용하고, On-Premise를 기본 Application 실행 환경으로 사용한다. EKS는 필요 시 확장 대상으로 활용한다.

## AWS 영역

| 구성 요소 | 역할 |
| --- | --- |
| AWS NLB | 외부 Client 요청을 받는 L4 Entry Point |
| EC2 (HAProxy) | NLB 트래픽을 On-Premise 또는 EKS 경로로 전달 |
| Site-to-Site VPN | AWS와 On-Premise 사설 연결 |
| EKS | 필요 시 확장 실행 환경 |

NLB는 외부 진입점을 단순화하고, 상세 라우팅은 EC2 HAProxy와 Kubernetes Ingress Controller가 담당한다.

## On-Premise 연계

| 구성 요소 | 역할 |
| --- | --- |
| Proxmox 상의 pfSense VM | VPN 종단, VLAN Gateway, 방화벽 |
| VLAN40 Private Network | Kubernetes, DB, 내부 서비스 배치 |
| Kubernetes Ingress Controller | On-Premise 내부 서비스 라우팅 |

pfSense는 물리 장비가 아니라 Proxmox 위에서 동작하는 VM이다.

## 기본 트래픽 흐름

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
Kubernetes Ingress Controller
  |
  v
Kubernetes Service
  |
  v
Application Pod
```

## 확장 트래픽 흐름

```text
Client
  |
  v
AWS NLB
  |
  v
EC2 (HAProxy)
  |-----------------------------|
  |                             |
  v                             v
On-Premise Kubernetes           AWS EKS
```

기본 트래픽은 On-Premise에서 처리한다. 리소스 부족 또는 특정 워크로드 확장이 필요한 경우 EKS로 일부 트래픽을 분산한다. 확장 기준과 Traffic Split 정책은 TBD이다.

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
| On-Premise 측 종단 | Proxmox 상의 pfSense VM |
| 상세 Tunnel 설정 | TBD |

## 장애 및 Failover 관점

| 장애 영역 | 대응 관점 |
| --- | --- |
| Application Pod 장애 | Kubernetes Replica 및 재시작 정책 |
| Kubernetes Node 장애 | 다른 Node로 Pod 재배치 |
| Site-to-Site VPN 장애 | AWS에서 On-Premise 서비스 접근 영향 |
| AWS NLB 또는 EC2 HAProxy 장애 | 외부 진입 경로 영향 |
| On-Premise 리소스 부족 | AWS EKS 확장 검토 |

세부 Health Check 기준과 RTO/RPO는 TBD이다.

## 보안 설계

| 영역 | 설계 방향 |
| --- | --- |
| 외부 접근 | AWS NLB로 진입 지점 제한 |
| AWS-On-Premise 연결 | Site-to-Site VPN 기반 사설 통신 |
| On-Premise Gateway | pfSense VM에서 라우팅 및 방화벽 적용 |
| 내부 서비스 | VLAN40과 Kubernetes Ingress Controller 중심 접근 |

## 관련 문서

- [Architecture Overview](./overview.md)
- [Network Design](./network-design.md)
- [Ceph Storage Architecture](./ceph-architecture.md)
- [Kubernetes Node Spec](./kubernetes-node-spec.md)
- [Docs README](../README.md)
