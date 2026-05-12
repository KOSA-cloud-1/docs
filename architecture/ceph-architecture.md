# Ceph 스토리지 아키텍처

## 목적

본 문서는 Kubernetes 워크로드, MariaDB Galera Cluster, 서비스 이미지 파일 저장소에서 사용할 Ceph Storage 연동 구조를 정의한다.

제공된 Ceph 구성도 기반의 스토리지를 사용한다. Kubernetes Persistent Volume은 Ceph RBD를 사용하고, 서비스에서 사용하는 이미지 파일 저장소는 같은 Ceph Storage의 Ceph S3를 사용한다.

## 현재 확정 기준

| 항목 | 값 |
| --- | --- |
| 팀 사용 가능 Ceph Storage | 약 3.7TB |
| Ceph RBD Node | 10.10.10.11 |
| Kubernetes PV 연동 방식 | RBD |
| Object Storage | Ceph S3 |
| Storage Traffic Network | 10.10.10.0/24 |
| Kubernetes Persistent Volume | Ceph RBD 기반 |
| 서비스 이미지 파일 저장소 | Ceph S3 기반 |

## 전체 구조

```text
Kubernetes Workload
  |-----------------------------|
  |                             |
  v                             v
PersistentVolumeClaim           Service Image Storage
  |                             |
  v                             v
PersistentVolume                Ceph S3
  |                             |
  v                             |
Ceph RBD                        |
  |                             |
  |-------------+---------------|
                v
       Ceph Storage
       Ceph Node (10.10.10.11)
```

Ceph는 On-Premise Kubernetes에서 상태 저장 워크로드와 서비스 이미지 파일 저장을 처리하기 위한 Storage 계층이다. Application PVC와 MariaDB Galera PVC는 Ceph RBD를 사용하고, 서비스에서 업로드하거나 조회하는 이미지 파일은 Ceph S3를 사용한다.

## Proxmox 및 Kubernetes 관점

| 항목 | 값 |
| --- | --- |
| Proxmox Node | team11, team12, team13, team14 |
| Proxmox Management IP | 192.168.36.151 ~ 192.168.36.154 |
| Proxmox 10G Node IP | 10.10.10.31 ~ 10.10.10.34 |
| Kubernetes Node 10G 할당 가능 대역 | 10.10.10.50 ~ 10.10.10.99 |
| Kubernetes Node 10G 현재 사용 IP | 10.10.10.50 ~ 10.10.10.58 |
| Ceph RBD Node | 10.10.10.11 |
| Ceph S3 Endpoint | TBD |

192.168.36.x 대역은 Management Network로 사용한다. Ceph RBD 접근, Ceph S3 접근, Kubernetes Node의 10G 내부 통신은 10.10.10.x 대역에서 처리한다.

## Ceph Storage 용량

| 항목 | 값 |
| --- | --- |
| 팀 사용 가능 용량 | 약 3.7TB |
| 용도 | Kubernetes PV, MariaDB Galera PVC, Application PVC, 서비스 이미지 파일 저장 |
| Quota 정책 | TBD |
| PVC 크기 정책 | TBD |
| S3 Bucket 정책 | TBD |

약 3.7TB는 팀이 사용할 수 있는 Ceph Storage 용량이다. Ceph RBD와 Ceph S3는 같은 Ceph Storage 용량을 공유하므로 PVC와 이미지 파일 저장소의 Quota 정책을 함께 관리해야 한다. Ceph 내부 산정값은 현재 확인할 수 없으므로 운영 산정 기준으로 사용하지 않는다.

## Ceph RBD 연동

Kubernetes는 Ceph RBD를 통해 Block Storage 기반 Persistent Volume을 사용한다.

```text
Pod
  |
  v
PVC
  |
  v
PV
  |
  v
RBD Image
  |
  v
Ceph Node 10.10.10.11
```

| 항목 | 값 |
| --- | --- |
| Backend | Ceph RBD |
| Ceph Node | 10.10.10.11 |
| Kubernetes PV 방식 | RBD 기반 Block Volume |
| StorageClass 이름 | TBD |

현재 문서의 기준은 RBD 사용이다.

## Ceph S3 연동

서비스에서 사용하는 이미지 파일 저장소는 같은 Ceph Storage의 Ceph S3를 사용한다.

```text
Application Pod
  |
  v
S3 API
  |
  v
Ceph S3
  |
  v
Ceph Storage
```

| 항목 | 값 |
| --- | --- |
| Backend | Ceph S3 |
| 용도 | 서비스 이미지 파일 저장소 |
| S3 Endpoint | TBD |
| Bucket 이름 | TBD |
| Access Key 관리 | TBD |
| 이미지 보관 정책 | TBD |

Ceph S3는 애플리케이션에서 업로드하거나 조회하는 이미지 파일을 저장하는 용도이다. Container Image 저장은 GitOps 배포 흐름의 Container Registry가 담당하므로, Ceph S3의 이미지 저장소와 구분한다.

## 네트워크 구조

Ceph 접근은 10.10.10.0/24 대역을 사용한다.

```text
Kubernetes Node (10.10.10.50 ~ 10.10.10.58)
  |
  v
10G Internal Network (10.10.10.0/24)
  |
  v
Ceph RBD / S3
Ceph Node (10.10.10.11)
```

| 네트워크 | 용도 |
| --- | --- |
| 192.168.36.0/24 | Proxmox Management, pfSense VM WAN |
| 172.17.x.x | VLAN 기반 Service Network |
| 10.10.10.0/24 | Kubernetes Node 10G 내부 통신, Ceph RBD 접근, Ceph S3 접근 |

Ceph Storage Traffic은 Management Network와 분리한다. Kubernetes 외부 서비스 트래픽은 AWS NLB, EC2 HAProxy, Site-to-Site VPN, pfSense VM, Kubernetes Ingress Controller 경로를 따르며 Ceph RBD/S3 접근 경로와 분리한다.

## 데이터베이스 저장소 관점

MariaDB Galera Cluster는 고가용성 DB 구성을 담당하고, Persistent Storage는 Ceph RBD 기반 PV를 사용한다.

| 항목 | 값 |
| --- | --- |
| DB | MariaDB Galera Cluster |
| Storage Backend | Ceph RBD |
| Ceph Node | 10.10.10.11 |
| 배치 네트워크 | VLAN40 Private Network |
| DB별 PVC 크기 | TBD |
| Backup Storage | TBD |

## 장애 대응 관점

| 장애 유형 | 대응 관점 |
| --- | --- |
| Kubernetes Pod 장애 | Kubernetes 재시작 및 재스케줄링으로 복구 |
| Kubernetes Node 장애 | 다른 Worker Node로 Pod 재배치 |
| Ceph 접근 장애 | RBD 및 S3 접근 영향 발생, Ceph 운영 상태 확인 필요 |
| 10G Network 장애 | Kubernetes Node와 Ceph RBD/S3 간 Storage 접근 영향 발생 |

Ceph 내부 장애 대응 방식은 직접 구현하지 않았으므로 본 문서는 Kubernetes에서 Ceph RBD와 Ceph S3를 사용하는 관점의 영향 범위만 기록한다.

## 설계 의도

Ceph RBD와 Ceph S3는 On-Premise 중심 운영 전략에서 Kubernetes 상태 저장 워크로드와 서비스 이미지 파일 저장을 처리하기 위한 스토리지 계층이다.

팀에서 사용할 수 있는 약 3.7TB Ceph Storage를 Kubernetes PV, DB Persistent Storage, 서비스 이미지 파일 저장소에 사용한다. Kubernetes Node와 Ceph RBD/S3는 10.10.10.0/24 내부망을 통해 연결하여 Management Network와 서비스 진입 트래픽의 영향을 분리한다.

## 관련 문서

- [Architecture Overview](./overview.md)
- [Network Design](./network-design.md)
- [Kubernetes Node Spec](./kubernetes-node-spec.md)
- [Hybrid Architecture](./hybrid-architecture.md)
