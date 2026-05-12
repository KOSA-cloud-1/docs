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

| 장비 | 역할 |
| --- | --- |
| Router `192.168.3.250` | 외부 인터넷 연결 및 상위 라우팅 |
| Managed Switch | VLAN Trunk 및 VLAN30 Access Port 제공 |
| Unmanaged Switch | Proxmox Node 1G NIC Management Network 연결 |

### Managed Switch Port

| 포트 | 모드 | 용도 |
| --- | --- | --- |
| Port 1 | Trunk | VLAN 트래픽 전달 |
| Port 2 | Trunk | VLAN 트래픽 전달 |
| Port 3 | Trunk | VLAN 트래픽 전달 |
| Port 4 | Access | VLAN30 Development |
| Port 5 | Trunk | VLAN 트래픽 전달 |

## Proxmox 네트워크

| 항목 | 값 |
| --- | --- |
| Management Network | 192.168.36.0/24 |
| Proxmox Management IP | 192.168.36.151 ~ 192.168.36.154 |
| pfSense VM WAN | 192.168.36.150 |
| Proxmox Cluster / Ceph Network | 10.10.10.0/24 |
| Proxmox Cluster / Ceph IP | 10.10.10.31 ~ 10.10.10.34 |

192.168.36.x 대역은 Management Network로만 사용한다.

## pfSense 구성

pfSense는 물리 장비가 아니라 Proxmox Node 위에서 동작하는 VM이다.

| 인터페이스 | 값 | 역할 |
| --- | --- | --- |
| WAN | 192.168.36.150 | Management Network 연결 |
| LAN | 172.16.1.1 | 내부 LAN Gateway |
| OPT1 | 172.17.0.1 | VLAN10 Public Gateway |
| OPT2 | 172.17.32.1 | VLAN20 DMZ Gateway |
| OPT3 | 172.17.64.1 | VLAN30 Development Gateway |
| OPT4 | 172.17.128.1 | VLAN40 Private Gateway |

pfSense VM은 VLAN 간 라우팅, 방화벽 정책, AWS Site-to-Site VPN 종단을 담당한다.

## VLAN 구성

| VLAN | 이름 | 대역 | 용도 |
| --- | --- | --- | --- |
| VLAN10 | Public | 172.17.0.0/24 | Public Service Network |
| VLAN20 | DMZ | 172.17.32.0/24 | DMZ Service Network |
| VLAN30 | Development | 172.17.64.0/24 | 개발 및 테스트 Service Network |
| VLAN40 | Private | 172.17.128.0/22 | Kubernetes, DB, 내부 서비스 Network |

Kubernetes Ingress Controller는 VLAN40 Private Network에서 내부 서비스 라우팅을 담당한다.

## 10G Cluster / Ceph Network

| 항목 | 값 |
| --- | --- |
| Network | 10.10.10.0/24 |
| Proxmox 10G Node IP | 10.10.10.31 ~ 10.10.10.34 |
| Kubernetes VM 10G 할당 가능 대역 | 10.10.10.50 ~ 10.10.10.99 |
| Kubernetes VM 10G 현재 사용 IP | 10.10.10.50 ~ 10.10.10.58 |
| Ceph RBD Node | 10.10.10.11 |
| Ceph S3 Endpoint | TBD |
| 용도 | Proxmox Cluster 통신, Ceph RBD/S3 접근, Storage Traffic |

이 네트워크는 Management Network와 VLAN Service Network와 분리한다.

## Kubernetes 온프레미스 네트워크

| 항목 | 값 |
| --- | --- |
| VLAN40 Private (Node Network) | 172.17.128.0/22 |
| Kubernetes Service Network | 10.96.0.0/12 |
| Kubernetes Pod Network | 10.244.0.0/16 |
| MetalLB IP Pool | 172.17.131.200-172.17.131.250 |
| Ingress VIP | 172.17.131.200 |

## AWS 네트워크

| 항목 | 대역 |
| --- | --- |
| AWS VPC CIDR | 10.20.0.0/16 |
| Public Subnet 1 | 10.20.0.0/24 |
| Public Subnet 2 | 10.20.1.0/24 |
| Private Subnet 1 | 10.20.10.0/24 |
| Private Subnet 2 | 10.20.11.0/24 |
| VPN Tunnel Inside CIDR | TBD |

## 서비스 트래픽 흐름

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
```

## Firewall Policy

pfSense VM은 Top-Down 방식으로 룰을 평가한다. 따라서 차단 정책을 허용 정책보다 위에 배치한다.

### VPN / AWS 유입

| 순서 | 정책 | Action |
| --- | --- | --- |
| 1 | EC2 HAProxy -> Ingress VIP(172.17.131.200) : 80/443 | 허용 |

### VLAN10 Public(사용안함)

| 순서 | 정책 | Action |
| --- | --- | --- |
| 1 | Public -> Internet | 허용 |

### VLAN20 DMZ

| 순서 | 정책 | Action |
| --- | --- | --- |
| 1 | DMZ -> Private | 차단 |
| 2 | DMZ -> Dev | 차단 |
| 3 | DMZ -> Internet | 허용 |

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
| 상세 Tunnel 설정 | TBD |

Site-to-Site VPN은 AWS EC2 HAProxy에서 On-Premise VLAN40의 Kubernetes Ingress Controller로 트래픽을 전달하는 사설 경로이다.

## 보안 설계 원칙

- 외부 Client의 직접 진입 지점은 AWS NLB로 제한한다.
- AWS와 On-Premise 간 통신은 Site-to-Site VPN을 사용한다.
- pfSense VM에서 VLAN 간 라우팅과 방화벽 정책을 중앙 관리한다.
- Kubernetes 내부 서비스 접근은 VLAN40과 Ingress Controller 중심으로 제한한다.
- Management Network, VLAN Service Network, 10G Cluster / Ceph Network를 분리한다.

## 관련 문서

- [Architecture Overview](./overview.md)
- [Hybrid Architecture](./hybrid-architecture.md)
- [Ceph Storage Architecture](./ceph-architecture.md)
- [Kubernetes Node Spec](./kubernetes-node-spec.md)
- [Docs README](../README.md)
