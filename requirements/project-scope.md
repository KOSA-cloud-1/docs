# 프로젝트 범위

## 목표

사내 AI 프로필 이미지 변환 서비스를 **온프렘+AWS 하이브리드** 인프라 위에
**Kubernetes/GitOps**로 구축하고, **고가용성·관측성·보안·비용 효율**을 갖춰 운영한다.

핵심은 AI 모델 구현이 아니라 **워크로드를 하이브리드 인프라에서 안정적으로 운영·관측**하는 것.

## In Scope

- 온프렘 Proxmox 위 kubeadm Kubernetes 클러스터(HA control-plane) 구축
- Ceph 스토리지(RBD/RGW), MariaDB Galera, 애플리케이션 5종 배포
- AWS Edge(Route53/NLB/HAProxy/VPN)로 공개 진입 + 사이트 연결
- GitOps(ArgoCD) + CI(GitHub Actions) 배포 파이프라인, dev/prod 분리
- 관측성(Prometheus/Grafana/Loki) 및 HPA, 알림
- 시크릿 관리(External Secrets + AWS Secrets Manager)
- 백업(사진 객체 → AWS S3)

## Out of Scope

- AI 모델 자체의 학습/정확도 개선
- AWS EKS(관리형 K8s) 사용 — K8s는 온프렘 kubeadm
- 멀티 리전/멀티 클러스터 DR
- dev/prod **데이터** 완전 격리(현재 공유형, 데모 한정)

## 평가 관점(운영 요구사항)

| 항목 | 무엇으로 보여주나 |
|---|---|
| 비용 절감 | 경량 AWS Edge(EC2 t3.micro+NLB) + 온프렘 컴퓨트, on-demand dev |
| 확장성 | HPA(photo/gateway), Galera/ingress 수평 확장 |
| 고가용성 | 계층별 HA(NLB·VPN·HAProxy·ingress·kube-vip·Galera) |
| 보안 | pfSense/VLAN/DMZ, VPN, NLB TLS 종료, ESO 시크릿 |
| 관측성 | Prometheus/Grafana/Loki, 부하→HPA→대시보드 시연 |

## 제약

- 수행 기간 약 2주 → 핵심 위주, 문서는 명확성 우선.
- 자원 제한 → dev는 on-demand, 모니터링은 전용 노드(worker7).
