# 하이브리드 아키텍처 (온프렘 + AWS)

## 1. 역할 분담

| 위치 | 역할 |
|---|---|
| **AWS (Edge)** | 공개 진입점(도메인·TLS·공인 IP), 외부→온프렘 연결(VPN) |
| **온프렘 (Proxmox)** | 실제 워크로드 — Kubernetes, 애플리케이션, DB(Galera), 스토리지(Ceph), 관측성 |

핵심 설계 의도: **공인 IP/도메인/TLS 같은 "인터넷 경계"는 AWS가 맡고, 컴퓨트·데이터는 온프렘에 둔다.**
온프렘 사설망을 인터넷에 직접 노출하지 않고 AWS Edge를 거쳐 들어오게 한다.

## 2. 두 사이트 연결: IPsec VPN

- **AWS측**: StrongSwan EC2 2대(`vpn-a`/`vpn-b`), 서비스 EIP 1개.
- **온프렘측**: ER605 라우터가 IPsec initiator로 AWS 서비스 EIP에 연결.
- **라우팅**: AWS route table에 온프렘 CIDR(`172.17.32.0/24`, `192.168.36.0/24`)을 active VPN 인스턴스 ENI로 향하게 추가.
- **이중화**: active/backup. EventBridge가 1분 주기로 Lambda를 실행해, 정상 인스턴스 중 우선순위(`vpn-a` 100 > `vpn-b` 90) 높은 쪽으로 **EIP와 온프렘 route를 이전**한다. (IPsec은 peer IP 고정 stateful 연결이라 단일 EIP + 페일오버 방식)

## 3. 데이터 경로 vs 관리 경로

- **데이터(웹) 경로**: 외부 → AWS NLB → AWS HAProxy → **VPN 터널** → 온프렘 HAProxy → ingress → 서비스.
  AWS HAProxy의 backend가 온프렘 사설 VIP(`172.17.32.20`)이므로, 이 hop은 VPN을 통과한다.
- **관리 경로**: 운영자는 관리망(`172.17.128.0/22`)으로 SSH/kubectl. AWS↔온프렘 내부망 직접 라우팅은 DMZ로 한정.

## 4. 전체 흐름

```text
[사용자] ─https─▶ Route53 ─▶ NLB(443 TLS종료) ─▶ AWS HAProxy ×2(active/active)
                                                        │
                                                  [IPsec VPN]  ← StrongSwan ×2(active/backup) ↔ ER605
                                                        ▼
                                  온프렘 HAProxy VIP(active/backup) ─▶ ingress-nginx(active/active)
                                                        ▼
                                  gateway ─▶ auth/employee ─▶ photo-service
                                                  │              │
                                              Galera(3)       Ceph RGW(S3)
```

## 5. 하이브리드에서 고려한 점

- **단일 진입 도메인**: `fhwang.cloud` 하나로 받아 AWS에서 TLS 종료 → 온프렘은 평문 80만 처리(인증서 관리 단순화).
- **온프렘 미노출**: 온프렘 공인 IP 없이 AWS Edge + VPN으로만 유입.
- **장애 격리**: AWS Edge HA(NLB/HAProxy/VPN)와 온프렘 HA(HAProxy/ingress/kube-vip/Galera)를 계층별로 구성. 계층별 방식은 [aws-integration.md](aws-integration.md) 및 `../operations/failure-scenarios.md` 참고.
- **비용/운영**: AWS는 t3.micro급 EC2(HAProxy/VPN) + NLB만 사용하는 경량 Edge. 무거운 컴퓨트/스토리지는 온프렘.
