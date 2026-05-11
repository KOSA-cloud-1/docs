# Hybrid Infrastructure Architecture Overview

## 1. 목적

본 문서는 On-Premise 환경과 AWS Cloud를 결합한 Hybrid Infrastructure의 전체 아키텍처를 정의한다.

본 프로젝트의 목표는 다음과 같다.

* Cost Optimization: 기본 워크로드는 On-Premise에서 처리하여 Cloud 비용 최소화
* Performance Scalability: 필요 시 AWS 환경을 활용한 확장
* High Availability: 장애 대응 가능한 구조 설계

---

## 2. 전체 구성 개요

본 시스템은 On-Premise와 AWS를 Site-to-Site VPN으로 연결한 Hybrid Infrastructure로 구성된다.

* On-Premise: 실제 서비스 실행 환경
* AWS: 외부 진입 지점 및 트래픽 제어 계층
* Kubernetes: 컨테이너 오케스트레이션
* Ceph: 분산 스토리지
* GitOps: ArgoCD 기반 배포

AWS는 외부 트래픽의 진입 지점 역할을 수행하며,
On-Premise 리소스가 포화 상태에 도달할 경우 확장 환경(EKS)으로 활용된다.

---

## 3. 주요 구성 요소

### 3.1 On-Premise

| 구성 요소                | 역할                         |
| -------------------- | -------------------------- |
| Proxmox Cluster | VM 기반 가상화 인프라 및 10G Cluster Network 구성 |
| Ceph Storage | 10G 전용망 기반 분산 스토리지 및 Persistent Volume 제공 |
| Kubernetes Cluster   | Application 실행             |
| pfSense              | VLAN 라우팅, 방화벽, VPN Gateway |
| MariaDB Galera       | 고가용성 DB                    |
| Prometheus / Grafana | 모니터링                       |

---

### 3.2 AWS Cloud

| 구성 요소            | 역할                      |
| ---------------- | ----------------------- |
| ALB              | 외부 트래픽 수신 및 Entry Point |
| EC2 (HAProxy)    | 트래픽 제어 및 On-Prem 전달     |
| Site-to-Site VPN | On-Premise 연결           |
| EKS (선택)         | 확장 환경                   |

AWS ALB는 외부 트래픽 진입 지점으로 동작하며, 필요 시 TLS termination을 수행한다.

---

## 4. 네트워크 구조 요약

On-Premise는 VLAN 기반으로 네트워크를 분리한다.

| VLAN   | 용도          | 대역              |
| ------ | ----------- | --------------- |
| VLAN10 | Public      | 172.17.0.0/24   |
| VLAN20 | DMZ         | 172.17.32.0/24  |
| VLAN30 | Development | 172.17.64.0/24  |
| VLAN40 | Private     | 172.17.128.0/22 |

* pfSense는 모든 VLAN의 Gateway 역할을 수행한다.
* AWS와 On-Premise는 VPN으로 연결된다.
* 추가로 Proxmox Cluster와 Ceph Storage 통신을 위해 10G 전용 네트워크(10.10.10.0/24)를 사용한다.
해당 네트워크는 서비스 트래픽과 분리하여 스토리지 복제 및 클러스터 통신 안정성을 확보한다.

Kubernetes Cluster는 Private Network(VLAN40)에 배치된다.

---

## 5. 트래픽 흐름

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
pfSense
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

### 흐름 설명

1. Client는 AWS ALB로 요청을 전달한다.
2. ALB는 EC2의 HAProxy로 트래픽을 전달한다.
3. HAProxy는 VPN을 통해 On-Premise로 전달한다.
4. pfSense는 VLAN 기반 내부 네트워크로 라우팅한다.
5. 트래픽은 Private Network(VLAN40)를 통해 Kubernetes Ingress로 전달된다.
6. Ingress Controller는 요청을 서비스로 라우팅한다.
7. Application Pod에서 요청을 처리한다.

확장 시 일부 트래픽은 AWS EKS로 분산될 수 있다.

---

## 6. 스케일링 전략

* 기본: On-Premise Kubernetes에서 처리
* 확장:

  * CPU / Memory 사용률 70~80% 이상 시
  * AWS EKS로 워크로드 확장 또는 트래픽 분산

---

## 7. 스토리지 구조

* Ceph 기반 분산 스토리지 사용
* Kubernetes Persistent Volume으로 연동

---

## 8. 데이터베이스 구조

MariaDB Galera Cluster 기반

* Multi-Master 구조
* Ceph 기반 Persistent Storage 사용

---

## 9. 배포 구조 (GitOps)

```text
app Repository
  |
  v
GitHub Actions
  |
  v
Container Registry
  |
  v
k8s-manifest Repository
  |
  v
ArgoCD
  |
  v
Kubernetes Cluster
```

Application Repository와 Manifest Repository를 분리하여 관리한다.

ArgoCD는 Repository 상태를 기준으로 클러스터를 지속적으로 동기화한다.

---

## 10. 모니터링

* Prometheus + Grafana 기반

---

## 11. 고가용성 전략

| 영역          | 방식                 |
| ----------- | ------------------ |
| Application | Kubernetes Replica |
| Storage     | Ceph               |
| Database    | MariaDB Galera     |
| Network     | pfSense            |
| Deployment  | ArgoCD             |

---

## 12. 설계 의도

* On-Premise 중심 비용 최적화
* AWS 기반 확장 구조
* Kubernetes 표준 운영
* GitOps 기반 자동화
* 장애 대응 가능 구조

---

## 13. 관련 문서

* architecture/network-design.md
* architecture/hybrid-architecture.md
* architecture/aws-integration.md
* runbooks/*
* operations/*
* decisions/*
