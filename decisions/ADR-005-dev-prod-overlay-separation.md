# ADR-005: dev/prod 환경 분리 (Kustomize overlay + on-demand dev)

- 상태: 채택(Accepted)

## Context

초기에는 단일 환경이었고 모든 배포 파이프라인이 `dev` 브랜치에 묶여 있어, `main` 릴리즈가
실제로는 아무것도 배포하지 않았다. dev/운영을 분리하되, 클러스터/노드 자원은 제한적이다.

## Decision

- `apps/`를 **Kustomize `base` + `overlays/{prod,dev}`** 로 재구성.
  - prod → 네임스페이스 `apps`(ArgoCD `kosa-apps`), HPA/PDB 유지.
  - dev → 네임스페이스 `apps-dev`(ArgoCD `kosa-apps-dev`).
- **dev는 on-demand**: 평소 replica 0(자원 ~0), 검증 시 `kubectl scale`로 기동. `ignoreDifferences(.spec.replicas)`로 ArgoCD가 되돌리지 않음. photo-service는 affinity 해제해 prod 노드(worker1/2) 경합 회피.
- **격리 수준 = 공유형**: dev가 운영 DB(Galera)·Ceph·AWS Secret(`prod/*`)을 공유(데모 속도 우선). 데이터 완전 분리는 추후 `dev/*` Secret + 별도 스키마로 승격 가능.
- **이미지 registry/태그 분리**: base는 논리 이미지명만, overlay `images[].newName`(=`DOCKERHUB_USERNAME`)·`newTag`(=SHA)를 CI가 주입.
- 잦은 롤아웃 누적 RS 정리: Deployment `revisionHistoryLimit: 3`.

## Consequences

- (+) "dev에서 검증 → main 릴리즈로 prod 승격" GitOps 흐름 확보(실증됨).
- (+) dev 평소 자원 점유 ~0(on-demand)로 제한된 클러스터에 부담 없음.
- (+) registry 계정 변경 시 시크릿만 바꾸면 됨(매니페스트 하드코딩 제거).
- (−) **공유형이라 dev 테스트가 운영 데이터에 영향 가능** — 데모 한정 트레이드오프. 실서비스 전 데이터 격리 필요.
- (−) 시연(HPA/부하/Grafana)은 HPA가 있는 **prod에서** 수행(dev는 HPA 없음·0 replica).
- 운영 흐름: [../operations/deployment-flow.md](../operations/deployment-flow.md), `k8s-manifests/apps/README.md`.
