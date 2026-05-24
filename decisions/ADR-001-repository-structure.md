# ADR-001: 리포지토리 구조 (멀티 레포 분리)

- 상태: 채택(Accepted)

## Context

AI 프로필 이미지 변환 서비스를 온프렘+AWS 하이브리드에 GitOps로 운영해야 한다.
애플리케이션 코드, Kubernetes 선언 상태, 온프렘 인프라, AWS 인프라, 문서가 한 곳에 섞이면
권한 분리·CI 트리거·소유권이 모호해진다.

## Decision

역할별 **멀티 레포**로 분리한다.

| repo | 책임 |
|---|---|
| `app` | 애플리케이션 소스 + 이미지 빌드 CI |
| `k8s-manifests` | Kubernetes 선언 상태 (ArgoCD가 보는 GitOps source) |
| `infra-proxmox` | 온프렘 Proxmox(Terraform) + kubeadm(Ansible) + 온프렘 HAProxy |
| `infra-aws` | AWS Edge(NLB/HAProxy/VPN) Terraform |
| `docs` | 설계·운영·의사결정 문서 |

- app CI는 이미지 빌드 후 `k8s-manifests`의 image 태그만 갱신(코드↔배포 관심사 분리).
- 모든 repo는 Git Flow(`main`/`dev`/`feature/*`/`release/*`/`hotfix/*`).

## Consequences

- (+) 관심사·권한·CI 트리거 분리. k8s-manifests만 ArgoCD가 신뢰.
- (+) 인프라(IaC)와 런타임 상태(매니페스트)가 섞이지 않음.
- (−) 변경이 두 repo에 걸치면(예: 앱 포트 변경) PR이 2개 필요 → 배포 흐름 문서로 보완([../operations/deployment-flow.md](../operations/deployment-flow.md)).
- (−) repo 간 버전 정합은 이미지 태그(커밋 SHA)로 추적.
