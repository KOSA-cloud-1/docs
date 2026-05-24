# ADR-002: 네트워크 분리 (pfSense + VLAN + AWS Edge)

- 상태: 채택(Accepted)

## Context

온프렘 워크로드를 인터넷에 직접 노출하지 않으면서 외부 사용자에게 서비스해야 한다.
내부(K8s 노드망/관리망)와 외부 진입(DMZ)을 분리하고, AWS↔온프렘을 안전하게 연결해야 한다.

## Decision

- **pfSense 방화벽 + VLAN 세그멘테이션**으로 경계를 나눈다.
  - DMZ(VLAN20, `172.17.32.0/24`): 외부 트래픽이 처음 닿는 온프렘 경계(HAProxy). gateway `172.17.32.1`.
  - K8s 노드망(`10.10.10.0/24`): 클러스터 내부 통신 전용(외부 비노출).
  - 관리망(`172.17.128.0/22`): 운영자 SSH/API/MetalLB.
- **공개 진입은 AWS Edge로 단일화**: Route53 + NLB(TLS 종료) + HAProxy EC2. 온프렘 공인 IP 없음.
- **AWS↔온프렘은 IPsec VPN**(StrongSwan ↔ ER605). AWS route에 온프렘 CIDR을 **DMZ로 한정**해 내부망 직접 라우팅을 막는다.
- **TLS는 NLB(443)에서 종료**, 이후 전 구간 평문 HTTP 80(온프렘 HAProxy도 80 전용).
- 이중화: AWS HAProxy=active/active(NLB), VPN=active/backup(EIP+Lambda), 온프렘 HAProxy=active/backup(VRRP).

## Consequences

- (+) 온프렘 미노출, 단일 도메인/인증서 관리, 계층별 장애 격리.
- (+) 외부 진입면이 AWS Edge로 좁혀져 공격면 축소.
- (−) 데이터 경로가 VPN을 통과 → VPN이 active/backup이라 페일오버 시 짧은 단절 가능(1분 주기 감지).
- (−) pfSense/VLAN/VPN 구성 복잡도 증가 → runbook/문서로 보완.
- 상세: [../architecture/network-design.md](../architecture/network-design.md), [../architecture/aws-integration.md](../architecture/aws-integration.md).
