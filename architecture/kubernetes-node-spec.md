# Kubernetes 노드 스펙

## 목적

본 문서는 On-Premise Kubernetes Cluster를 구성하는 VM 노드의 목표 스펙과 배치 기준을 정의한다.

이 문서는 Kubernetes 노드 설계 기준이며, 구현 코드는 이 문서의 스펙을 따르는 형태로 관리한다.

## Proxmox 물리 노드 스펙

Proxmox는 총 4대로 구성한다.

| Proxmox Node | Management IP | CPU | RAM |
| --- | --- | --- | --- |
| team11 | 192.168.36.151 | 24 core | 32GB |
| team12 | 192.168.36.152 | 24 core | 32GB |
| team13 | 192.168.36.153 | 24 core | 32GB |
| team14 | 192.168.36.154 | 24 core | 32GB |

| 항목 | 총합 |
| --- | --- |
| Physical CPU | 96 core |
| Physical RAM | 128GB |

## Kubernetes VM 전체 요약

| 구분 | 노드 수 | vCPU 합계 | Memory 합계 | Disk 합계 |
| --- | ---: | ---: | ---: | ---: |
| Control Plane | 3 | 8 vCPU | 12GB | 60GB |
| Worker | 6 | 24 vCPU | 12GB | 120GB |
| Total | 9 | 32 vCPU | 24GB | 180GB |

모든 Kubernetes VM의 OS Disk는 20GB로 구성한다.

## Kubernetes VM 상세 스펙

| 이름 | 역할 | VM ID | Proxmox Node | IP | vCPU | Memory | Disk |
| --- | --- | ---: | --- | --- | ---: | ---: | ---: |
| cp1 | Control Plane | 1001 | team12 | 10.10.10.50 | 2 | 4GB | 20GB |
| cp2 | Control Plane | 1002 | team13 | 10.10.10.52 | 4 | 4GB | 20GB |
| cp3 | Control Plane | 1003 | team14 | 10.10.10.54 | 2 | 4GB | 20GB |
| worker1 | Worker | 1011 | team12 | 10.10.10.51 | 4 | 2GB | 20GB |
| worker2 | Worker | 1012 | team13 | 10.10.10.53 | 4 | 2GB | 20GB |
| worker3 | Worker | 1013 | team14 | 10.10.10.55 | 4 | 2GB | 20GB |
| worker4 | Worker | 1014 | team11 | 10.10.10.56 | 4 | 2GB | 20GB |
| worker5 | Worker | 1015 | team11 | 10.10.10.57 | 4 | 2GB | 20GB |
| worker6 | Worker | 1016 | team11 | 10.10.10.58 | 4 | 2GB | 20GB |

## Proxmox Node별 VM 배치

| Proxmox Node | 배치된 Kubernetes VM | vCPU 합계 | Memory 합계 | Disk 합계 |
| --- | --- | ---: | ---: | ---: |
| team11 | worker4, worker5, worker6 | 12 vCPU | 6GB | 60GB |
| team12 | cp1, worker1 | 6 vCPU | 6GB | 40GB |
| team13 | cp2, worker2 | 8 vCPU | 6GB | 40GB |
| team14 | cp3, worker3 | 6 vCPU | 6GB | 40GB |

단순 합산 기준으로 Kubernetes VM은 전체 Proxmox 물리 CPU 96 core 중 32 vCPU, 전체 RAM 128GB 중 24GB를 할당받는다. CPU Overcommit 정책과 Proxmox/Ceph/pfSense VM을 포함한 전체 리소스 예약 정책은 TBD이다.

## VM 공통 설정

| 항목 | 값 |
| --- | --- |
| Clone Source Node | team14 |
| Template VM ID | 8000 |
| Clone 방식 | Full Clone |
| QEMU Guest Agent | Enabled |
| CPU Type | host |
| Memory 방식 | Dedicated |
| OS Type | Linux |
| SCSI Hardware | virtio-scsi-pci |
| Boot Disk | scsi0 |
| Disk Datastore | local-lvm |
| Disk Format | raw |
| Disk Size | 20GB |

## VM 네트워크 설정

| Interface | Bridge | Model | VLAN | IP 설정 | 용도 |
| --- | --- | --- | --- | --- | --- |
| net0 | vmbr0 | virtio | VLAN40 | DHCP | VLAN40 Private / 1G |
| net1 | vmbr1 | virtio | 없음 | Static `10.10.10.x/24` | 10G 내부망 / Storage 접근 |

Kubernetes VM의 Static IP는 `10.10.10.50`부터 `10.10.10.58`까지 사용한다. 이 IP는 Kubernetes Service CIDR 또는 Pod CIDR이 아니라 VM Node의 10G 내부망 IP이다.

| 항목 | 값 |
| --- | --- |
| Kubernetes VM 10G IP Range | 10.10.10.50 ~ 10.10.10.58 |
| Ceph RBD Node | 10.10.10.11 |
| Ceph S3 Endpoint | TBD |
| Kubernetes Service CIDR | TBD |
| Kubernetes Pod CIDR | TBD |
| Control Plane Endpoint | TBD |
| Ingress Controller Endpoint | TBD |

## Cloud-Init 설정

Cloud-Init은 VM 초기화와 QEMU Guest Agent 실행에 사용한다.

| 항목 | 값 |
| --- | --- |
| Package Update | `apt update` |
| Guest Agent | `systemctl start qemu-guest-agent` |
| SSH Key 주입 | 사용 |

## 설계 의도

Kubernetes Control Plane은 3대로 구성하여 기본적인 Control Plane 고가용성 구조를 확보한다. Worker Node는 6대로 구성하여 Application Pod를 분산 배치한다.

Kubernetes VM의 10G Static IP는 내부 통신, Ceph RBD 접근, Ceph S3 접근을 위한 노드 IP로 사용한다. 외부 서비스 노출은 AWS ALB, EC2 HAProxy, Site-to-Site VPN, pfSense VM, Kubernetes Ingress Controller 경로를 따른다.

## 관련 문서

- [Architecture Overview](./overview.md)
- [Network Design](./network-design.md)
- [Ceph Storage Architecture](./ceph-architecture.md)
- [Hybrid Architecture](./hybrid-architecture.md)
