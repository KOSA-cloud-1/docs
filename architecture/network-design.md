# 네트워크 설계

## 목적

본 문서는 On-Premise Hybrid Infrastructure의 네트워크 구조, VLAN 설계, Proxmox 네트워크, pfSense VM 구성, 방화벽 정책, AWS Site-to-Site VPN 연결 구조를 정의한다.

## 전체 네트워크 구조

On-Premise 네트워크는 Router, Managed Switch, Unmanaged Switch, Proxmox Node, Proxmox 상의 pfSense VM, VLAN 기반 Service Network로 구성된다.

```text
Internet
  |
  v
Router (192.168.3.250)
  |
  v
Managed Switch
  |-- Port 1: Trunk
  |-- Port 2: Trunk
  |-- Port 3: Trunk
  |-- Port 4: Access VLAN30 Development
  |-- Port 5: Trunk
  |
  v
Unmanaged Switch
  |
  v
Proxmox Node 1G NICs
  |
  v
Management Network (192.168.36.0/24)
```

Service Network와 Cluster / Ceph Network는 다음과 같이 분리한다.

```text
Proxmox Node
  |-- 1G NIC  -> Management Network (192.168.36.0/24)
  |-- 10G NIC -> Proxmox Cluster / Ceph Network (10.10.10.0/24)
  |
  v
Proxmox 상의 pfSense VM
  |-- WAN      -> Management Network (192.168.36.0/24)
  |-- LAN/VLAN -> Proxmox VLAN Trunk Bridge
                  |-- VLAN10 Public      (172.17.0.0/24)
                  |-- VLAN20 DMZ         (172.17.32.0/24)
                  |-- VLAN30 Development (172.17.64.0/24)
                  |-- VLAN40 Private     (172.17.128.0/22)
```

## 물리 네트워크 구성

### Router

| 항목 | 값 |
| --- | --- |
| Router IP | 192.168.3.250 |
| 역할 | 외부 인터넷 연결 및 상위 라우팅 |

### Managed Switch

Managed Switch는 VLAN Trunk 및 Access Port 구성을 담당한다.

| 포트 | 모드 | 용도 |
| --- | --- | --- |
| Port 1 | Trunk | VLAN 트래픽 전달 |
| Port 2 | Trunk | VLAN 트래픽 전달 |
| Port 3 | Trunk | VLAN 트래픽 전달 |
| Port 4 | Access | VLAN30 Development |
| Port 5 | Trunk | VLAN 트래픽 전달 |

### Unmanaged Switch

Unmanaged Switch는 VLAN 기능 없이 Management Network 전용으로 사용한다.

| 항목 | 내용 |
| --- | --- |
| VLAN 기능 | 없음 |
| 연결 대상 | Proxmox Node들의 1G NIC |
| 사용 네트워크 | Management Network |
| 대역 | 192.168.36.0/24 |

## Managed Switch / Unmanaged Switch 역할

| 구분 | 역할 |
| --- | --- |
| Managed Switch | VLAN Trunk, VLAN30 Access Port, VLAN 기반 Service Network 전달 |
| Unmanaged Switch | Proxmox Node 1G NIC Management Network 연결 |

Managed Switch는 VLAN 기반 Service Network를 전달하고, Unmanaged Switch는 Proxmox Management Network 연결만 담당한다.

## Proxmox 네트워크

Proxmox Node는 1G NIC와 10G NIC를 분리하여 사용한다.

| Proxmox Node | Management IP | CPU | RAM |
| --- | --- | --- | --- |
| team11 | 192.168.36.151 | 24 core | 32GB |
| team12 | 192.168.36.152 | 24 core | 32GB |
| team13 | 192.168.36.153 | 24 core | 32GB |
| team14 | 192.168.36.154 | 24 core | 32GB |

| 항목 | 값 |
| --- | --- |
| 1G NIC 용도 | Management Network |
| 10G NIC 용도 | Proxmox Cluster / Ceph Network |
| Management Network | 192.168.36.0/24 |
| Proxmox Management IP | 192.168.36.151 ~ 192.168.36.154 |
| pfSense VM WAN | 192.168.36.150 |
| Proxmox Cluster / Ceph Network | 10.10.10.0/24 |
| Proxmox Cluster / Ceph IP | 10.10.10.31 ~ 10.10.10.34 |

192.168.36.x 대역은 Proxmox Management 및 pfSense VM WAN 연결 용도로만 설명한다. Kubernetes Service Network나 Ceph Traffic 용도로 사용하지 않는다.

## pfSense 구성

pfSense는 물리 장비가 아니라 Proxmox Node 위에서 동작하는 VM이다.

pfSense VM은 WAN 인터페이스를 Management Network에 연결하고, LAN/VLAN 인터페이스를 Proxmox의 VLAN Trunk Bridge에 연결한다. pfSense VM은 VLAN10, VLAN20, VLAN30, VLAN40의 Gateway 역할을 수행한다.

| 인터페이스 | 값 | 역할 |
| --- | --- | --- |
| WAN | 192.168.36.150 | Management Network 연결 |
| LAN | 172.16.1.1 | 내부 LAN Gateway |
| OPT1 | VLAN10 Public Gateway | VLAN10 Gateway |
| OPT2 | VLAN20 DMZ Gateway | VLAN20 Gateway |
| OPT3 | VLAN30 Development Gateway | VLAN30 Gateway |
| OPT4 | VLAN40 Private Gateway | VLAN40 Gateway |

pfSense VM의 주요 역할은 다음과 같다.

- VLAN 간 라우팅
- 방화벽 정책 적용
- AWS Site-to-Site VPN 종단
- On-Premise 네트워크 중앙 Gateway

## VLAN 구성

172.17.x.x 대역은 VLAN 기반 Service Network로 사용한다.

| VLAN | 이름 | 대역 | 용도 |
| --- | --- | --- | --- |
| VLAN10 | Public | 172.17.0.0/24 | Public Service Network |
| VLAN20 | DMZ | 172.17.32.0/24 | DMZ Service Network |
| VLAN30 | Development | 172.17.64.0/24 | 개발 및 테스트 Service Network |
| VLAN40 | Private | 172.17.128.0/22 | Kubernetes, DB, 내부 서비스 Network |

Kubernetes Ingress Controller는 VLAN40 Private Network에서 내부 서비스 라우팅의 중심 역할을 수행한다.

## 10G Cluster / Ceph Network

10.10.10.x 대역은 Proxmox Cluster / Ceph Network로만 사용한다.

| 항목 | 값 |
| --- | --- |
| Network | 10.10.10.0/24 |
| Proxmox 10G Node IP | 10.10.10.31 ~ 10.10.10.34 |
| Kubernetes VM 10G IP | 10.10.10.50 ~ 10.10.10.99 |
| Ceph RBD Node | 10.10.10.11 |
| Ceph S3 Endpoint | TBD |
| 용도 | Proxmox Cluster / Ceph 통신 |
| 구조 | Spine-Leaf 기반 |
| 역할 | Ceph OSD 복제, Cluster 통신, Storage Traffic 분리 |

10G Cluster / Ceph Network는 Management Network, Kubernetes Service Network, VLAN Network와 분리한다. 이 네트워크는 Ceph 복제와 Proxmox Cluster 통신에 집중되며, Application Service Traffic 경로로 사용하지 않는다.

Kubernetes VM은 `vmbr1`에 Static IP `10.10.10.50 ~ 10.10.10.99`을 사용한다. 이 값은 Kubernetes Service CIDR 또는 Pod CIDR이 아니라 VM Node의 10G 내부망 IP이다.

## Kubernetes 네트워크

| 항목 | 값 |
| --- | --- |
| Kubernetes VM 수 | 9대 |
| Control Plane | 3대 |
| Worker | 6대 |
| Kubernetes VM vCPU 합계 | 32 vCPU |
| Kubernetes VM Memory 합계 | 24GB |
| Kubernetes VM Disk | VM당 20GB |
| Kubernetes 배치 네트워크 | VLAN40 Private Network |
| net0 | vmbr0, virtio, VLAN40, DHCP |
| net1 | vmbr1, virtio, Static 10.10.10.x/24 |
| Service 진입점 | Kubernetes Ingress Controller |
| Kubernetes Service Network 세부 대역 | TBD |
| Pod Network 세부 대역 | TBD |
| Persistent Volume | Ceph RBD 기반 PV |
| Object Storage | Ceph S3 기반 서비스 이미지 파일 저장소 |

외부에서 들어온 서비스 트래픽은 AWS ALB, EC2 HAProxy, Site-to-Site VPN, pfSense VM을 거쳐 VLAN40의 Kubernetes Ingress Controller로 전달된다.

## 트래픽 흐름

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

트래픽은 외부 진입 지점인 AWS ALB에서 시작하여 AWS EC2 HAProxy를 거친 뒤, Site-to-Site VPN을 통해 On-Premise로 들어온다. On-Premise VPN 종단은 pfSense VM이 담당하며, pfSense VM 이후 트래픽은 VLAN40 Private Network의 Kubernetes Ingress Controller로 전달된다.

## Firewall Policy

pfSense VM은 Top-Down 방식으로 룰을 평가한다. 따라서 차단 정책을 허용 정책보다 위에 배치한다.

### VLAN10 Public(사용안함)

| 순서 | 정책 | Action |
| --- | --- | --- |
| 1 | Public -> Internet | 허용 |

### VLAN20 DMZ

| 순서 | 정책 | Action |
| --- | --- | --- |
| 1 | AWS/HAProxy -> Ingress(Private) : 80/443 | 허용 |
| 2 | DMZ -> Private | 차단 |
| 3 | DMZ -> Dev | 차단 |
| 4 | DMZ -> Internet | 허용 |

### VLAN30 Development

| 순서 | 정책 | Action |
| --- | --- | --- |
| 1 | Dev -> Ingress(Private) : 80/443 | 허용 |
| 2 | Dev -> K8s API(Private) : 6443 | 허용 |
| 3 | Dev -> pfSense(172.17.64.1) : 443/80/22 | 허용 |
| 4 | Dev -> Private : 22 | 허용 |
| 5 | Dev -> Private | 차단 |
| 6 | Dev -> DMZ | 차단 |
| 7 | Dev -> Internet | 허용 |

### VLAN40 Private

| 순서 | 정책 | Action |
| --- | --- | --- |
| 1 | Private -> Public | 차단 |
| 2 | Private -> DMZ | 차단 |
| 3 | Private -> Dev | 차단 |
| 4 | Private -> DNS(53) | 허용 |
| 5 | Private -> Internet : 80/443 | 허용 |

## VPN 구성

| 항목 | 값 |
| --- | --- |
| VPN 방식 | AWS Site-to-Site VPN |
| AWS 측 | AWS VPC |
| On-Premise 측 종단 | Proxmox 상의 pfSense VM |
| 주요 목적 | AWS와 On-Premise 간 사설 통신 |
| 상세 Tunnel 설정 | TBD |

Site-to-Site VPN은 AWS EC2 HAProxy에서 On-Premise VLAN40 Private Network의 Kubernetes Ingress Controller로 트래픽을 전달하기 위한 사설 연결 경로이다.

## 보안 설계 원칙

- 외부 Client의 직접 진입 지점은 AWS ALB로 제한한다.
- AWS와 On-Premise 간 통신은 Site-to-Site VPN을 사용한다.
- pfSense VM에서 VLAN 간 라우팅과 방화벽 정책을 중앙 관리한다.
- DMZ, Development, Private Network 간 접근은 필요한 방향만 허용한다.
- Kubernetes 내부 서비스 접근은 VLAN40 Private Network와 Ingress Controller 중심으로 제한한다.
- Management Network, Service Network, Proxmox Cluster / Ceph Network를 분리한다.

## 설계 의도

본 네트워크 설계는 관리 트래픽, 서비스 트래픽, 스토리지 트래픽을 분리하여 장애 영향 범위를 줄이고 운영 안정성을 확보하기 위한 구조이다.

Proxmox 상의 pfSense VM은 VLAN Gateway, 방화벽, AWS Site-to-Site VPN 종단을 담당한다. Kubernetes 서비스 라우팅은 VLAN40 Private Network의 Kubernetes Ingress Controller를 중심으로 구성한다.

Ceph와 Proxmox Cluster 통신은 10G 전용 네트워크에서 처리하여 Storage Traffic이 Management Network나 Service Network에 영향을 주지 않도록 한다.

## 관련 문서

- [Architecture Overview](./overview.md)
- [Hybrid Architecture](./hybrid-architecture.md)
- [Ceph Storage Architecture](./ceph-architecture.md)
- [Kubernetes Node Spec](./kubernetes-node-spec.md)
- [Docs README](../README.md)
