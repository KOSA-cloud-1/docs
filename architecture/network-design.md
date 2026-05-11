# Network Design

## 1. 목적

본 문서는 On-Premise 환경의 네트워크 구조, VLAN 설계, 10G Cluster Network, pfSense 방화벽 정책을 정의한다.

---

## 2. 전체 네트워크 구조

```text
Internet
  |
  v
Router (192.168.3.250)
  |
  v
Managed Switch (L2)
  |-- Port 1: Trunk
  |-- Port 2: Trunk
  |-- Port 3: Trunk
  |-- Port 4: Access VLAN30 (Development)
  |-- Port 5: Trunk
  |
  v
Unmanaged Switch
  |
  v
Proxmox Cluster (Management: 192.168.36.x)
  |
  v
pfSense (WAN: 192.168.36.150)
  |
  v
VLAN Network (172.17.x.x)
```

---

## 3. 물리 네트워크 구성

### Router

* 외부 인터넷 연결
* Router IP: 192.168.3.250
* 내부 Management Network로 트래픽 전달

---

### Managed Switch

관리형 스위치는 VLAN 및 트래픽 분리를 담당한다.

| 포트     | 모드     | 용도                    |
| ------ | ------ | --------------------- |
| Port 1 | Trunk  | VLAN 트래픽 전달           |
| Port 2 | Trunk  | VLAN 트래픽 전달           |
| Port 3 | Trunk  | VLAN 트래픽 전달           |
| Port 4 | Access | VLAN30 Development 전용 |
| Port 5 | Trunk  | VLAN 트래픽 전달           |

특징:

* pfSense 및 VLAN 네트워크 트래픽 처리
* Trunk 포트를 통해 VLAN10~40 전달

---

### Unmanaged Switch

비관리형 스위치는 Proxmox 관리망 연결 용도로 사용한다.

* VLAN 기능 없음
* Proxmox 1G NIC 연결
* Management Network 전용

---

## 4. Proxmox 네트워크

| 항목                     | 내용                              |
| ---------------------- | ------------------------------- |
| Management Network     | 192.168.36.0/24                 |
| Management IP          | 192.168.36.151 ~ 192.168.36.154 |
| Cluster / Ceph Network | 10.10.10.0/24                   |
| Cluster IP             | 10.10.10.31 ~ 10.10.10.34       |
| 역할                     | VM 실행, Kubernetes Node, Ceph 연동 |

Proxmox Node는 NIC 2개를 사용한다.

* 1G NIC → Management Network
* 10G NIC → Ceph / Cluster Network

---

## 5. pfSense 구성

| 인터페이스 | IP             | 역할                             |
| ----- | -------------- | ------------------------------ |
| WAN   | 192.168.36.150 | Router 및 Management Network 연결 |
| LAN   | 172.16.1.1     | 내부 Gateway                     |
| OPT1  | VLAN10         | Public Gateway                 |
| OPT2  | VLAN20         | DMZ Gateway                    |
| OPT3  | VLAN30         | Development Gateway            |
| OPT4  | VLAN40         | Private Gateway                |

pfSense는 다음 역할을 수행한다.

* VLAN 간 라우팅
* 방화벽 정책 적용
* AWS와의 Site-to-Site VPN 종단
* On-Premise 네트워크의 중앙 Gateway

---

## 6. VLAN 구성

| VLAN   | 용도          | 대역              | 설명                     |
| ------ | ----------- | --------------- | ---------------------- |
| VLAN10 | Public      | 172.17.0.0/24   | 외부 공개 가능 영역            |
| VLAN20 | DMZ         | 172.17.32.0/24  | 외부 접근 서비스 영역           |
| VLAN30 | Development | 172.17.64.0/24  | 개발 및 테스트 환경            |
| VLAN40 | Private     | 172.17.128.0/22 | Kubernetes, DB, 내부 서비스 |

---

## 7. 10G Cluster / Ceph Network

Proxmox Cluster와 Ceph Storage는 별도의 10G 전용 네트워크를 사용한다.

| 항목      | 내용                                          |
| ------- | ------------------------------------------- |
| Network | 10.10.10.0/24                               |
| Node IP | 10.10.10.31 ~ 10.10.10.34                   |
| 용도      | Proxmox Cluster / Ceph 통신                   |
| 속도      | 10GbE                                       |
| 구조      | Spine-Leaf                                  |
| 역할      | Ceph OSD 복제, Cluster 통신, Storage Traffic 분리 |

특징:

* 서비스 네트워크와 완전히 분리
* Ceph 복제 트래픽 고부하 처리
* Proxmox Cluster 안정성 확보

---

## 8. Kubernetes 네트워크

* Kubernetes Node는 VLAN40 (Private)에 배치
* 외부 접근은 Ingress Controller를 통해 수행
* Pod Network는 CNI 기반 구성
* 상태 저장 서비스는 Ceph PVC 사용

---

## 9. 트래픽 흐름

### 전체 흐름

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
VLAN40 (Private)
  |
  v
Kubernetes Ingress
  |
  v
Service
  |
  v
Pod
```

---

## 10. Firewall Policy

### 정책 원칙

* 차단 정책을 허용 정책보다 상단 배치
* VLAN 간 접근 최소화
* 최소 권한 원칙 적용

---

### VLAN20 (DMZ)

* DMZ → Private 차단
* DMZ → Dev 차단
* DMZ → Internet 허용

---

### VLAN30 (Development)

* Dev → Private 차단
* Dev → DMZ 차단
* Dev → Internet 허용

---

### VLAN40 (Private)

* Private → Dev 차단
* Private → DMZ 차단
* Private → Internet 허용

---

## 11. VPN 구성

* AWS VPC ↔ pfSense IPSec VPN
* AWS ALB → HAProxy → VPN → On-Prem 전달
* VPN 트래픽은 Kubernetes Ingress로 전달

---

## 12. 보안 설계 원칙

* VLAN 기반 네트워크 분리
* Private Network 외부 직접 접근 차단
* DMZ / Dev → Private 접근 차단
* pfSense 중심 접근 통제
* 최소 권한 정책 적용

---

## 13. 설계 의도

* 네트워크 분리를 통한 보안 강화
* 10G 전용망 기반 Storage 성능 확보
* Proxmox / Ceph 안정성 확보
* Hybrid Cloud 연결 구조 지원
* 장애 발생 시 영향 최소화
