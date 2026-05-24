# 최종 발표 개요

발표 순서(21개 섹션)와 각 섹션 핵심 메시지·근거 문서를 매핑한다.

| # | 섹션 | 핵심 메시지 | 근거 / 자료 |
|---|---|---|---|
| 1 | 표지 | 프로젝트명 / 팀 / 날짜 | — |
| 2 | 팀원 소개 | 역할 분담 | `../schedules/role-assignment.md` |
| 3 | 프로젝트 배경 | 사내 AI 프로필 변환 수요 + 하이브리드 운영 과제 | `../requirements/project-scope.md` |
| 4 | 서비스 시나리오 | 업로드→AI 변환→저장. 트래픽 가변성 | `demo-scenario.md` |
| 5 | 운영 요구사항 | 비용/확장성/HA/보안/관측성 5축 | `../requirements/project-scope.md` |
| 6 | 전체 아키텍처 | 온프렘(워크로드) + AWS(Edge) 한 장 | `../architecture/overview.md` |
| 7 | 트래픽 흐름 | Route53→NLB→HAProxy→VPN→온프렘 HAProxy→ingress→pod | `../architecture/network-design.md` §3 |
| 8 | AWS 진입점 구성 | Route53/NLB(TLS종료)/HAProxy EC2×2 | `../architecture/aws-integration.md` |
| 9 | VPN / pfSense / VLAN 보안 경계 | pfSense+VLAN+DMZ, IPsec VPN, 온프렘 미노출 | `../decisions/ADR-002-network-separation.md` |
| 10 | On-Prem Proxmox / K8s 구성 | Proxmox VM + kubeadm(cp×3 kube-vip) + Calico + MetalLB | `../architecture/overview.md`, `../runbooks/` |
| 11 | Kubernetes 선택 이유 | 선언적 운영·자가복구·수평확장·GitOps 적합 | ADR-003 |
| 12 | Ceph 선택 이유 | 통합 스토리지(블록 RBD + 오브젝트 RGW), 온프렘 자체 운영 | ADR-004 |
| 13 | Ceph 구성 및 AWS 대응 비교 | RBD↔EBS, RGW S3↔S3. 온프렘 자체 vs 관리형 트레이드오프 | 비용 분석(#19)와 연계 |
| 14 | Galera DB 선택 이유 | 3-replica multi-master HA, RBD PVC | ADR-004 |
| 15 | CI/CD & GitOps | GitHub Actions(빌드)→k8s-manifests(overlay)→ArgoCD. dev/prod 분리 | `../operations/deployment-flow.md`, ADR-003/005 |
| 16 | Monitoring 구성 | Prometheus/Grafana/Loki/Alertmanager + HPA 메트릭 | `k8s-manifests/infra/monitoring/README.md` |
| 17 | 고가용성 설계 | 계층별 HA 표(active/active vs active/backup) | `../operations/failure-scenarios.md` §1 |
| 18 | 장애 대응 시나리오 | 실제 이력(ingress SPOF, calico, kube-vip, 루트앱 드리프트) | `../operations/failure-scenarios.md` §2 |
| 19 | 비용 분석 | 경량 AWS Edge + 온프렘 컴퓨트, on-demand dev로 절감 | 본 문서 아래 §비용 포인트 |
| 20 | 문제 해결 및 개선 방향 | 겪은 문제→해결, 향후(데이터 격리/Thanos/논리백업) | `../operations/`, ADR-005 |
| 21 | 결론 | 요구사항 5축을 어떻게 충족했나 요약 | — |

## 핵심 데모 (4번/17번/18번에서 활용)

부하 → HPA 스케일아웃 → Grafana 관측을 **prod에서** 라이브로. 상세: [demo-scenario.md](demo-scenario.md).

## 비용 포인트 (#19 보조 메모)

- AWS는 **t3.micro급 EC2(HAProxy×2, VPN×2) + NLB + Route53/ACM**만 사용하는 경량 Edge. 무거운 컴퓨트/스토리지/DB는 온프렘(고정비) → 클라우드 가변비 최소화.
- **on-demand dev**(평소 0 replica)로 비운영 환경 자원/비용 ~0.
- 관리형(EKS/RDS/관리형 S3) 대비: 운영 부담↑ 대신 **비용↓ + 온프렘 자산 활용**. (#13/#14 비교와 연결)
- 향후: Spot/스케줄 기반 축소, 모니터링 보존기간 조정으로 추가 절감 여지.

## 마무리 메시지(예시)

> "AI 이미지 변환은 요청량·처리비용이 가변적이라 metrics/logs 기반 관측과 계층적 HA가 필수다.
> 우리는 온프렘 컴퓨트 + AWS Edge를 GitOps로 묶어, 비용·확장성·가용성·보안·관측성을 모두 갖춘
> 재현 가능한 하이브리드 운영을 구성했다."
